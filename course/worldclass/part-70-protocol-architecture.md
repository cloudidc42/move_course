# Part 70: Real-World Protocol Architecture

## สารบัญ
- [Full DeFi Protocol Stack](#full-defi-protocol-stack)
- [Module Organization Patterns](#module-organization-patterns)
- [Upgrade Patterns](#upgrade-patterns)
- [Data Architecture](#data-architecture)
- [Integration Patterns](#integration-patterns)
- [Production Checklist](#production-checklist)

---

## Full DeFi Protocol Stack

```
Production Protocol Architecture (e.g., a DEX):

Layer 1: CORE (immutable or rarely changed)
  ├── core/math.move        (SafeMath, fixed-point)
  ├── core/pool.move        (AMM logic, swap formula)
  └── core/fee.move         (fee calculation)

Layer 2: ASSETS (token management)
  ├── assets/lp_token.move  (LP token minting/burning)
  ├── assets/wrapped.move   (wrapped token support)
  └── assets/registry.move  (token registry)

Layer 3: PERIPHERY (user-facing, upgradeable)
  ├── periphery/router.move      (multi-hop routing)
  ├── periphery/aggregator.move  (split routing)
  └── periphery/zap.move         (one-click liquidity)

Layer 4: GOVERNANCE (DAO control)
  ├── governance/governor.move  (proposal + voting)
  ├── governance/timelock.move  (delayed execution)
  └── governance/treasury.move  (fund management)

Layer 5: INFRASTRUCTURE (cross-cutting concerns)
  ├── infra/oracle.move         (price feeds)
  ├── infra/emergency.move      (pause + guardian)
  └── infra/events.move         (event definitions)

Layer 6: SDK (off-chain TypeScript)
  ├── sdk/client.ts              (API client)
  ├── sdk/router.ts              (routing engine)
  └── sdk/types.ts               (type definitions)
  
Module Dependencies (no circular dependencies!):
  Core ← Assets ← Periphery ← Governance
  Core ← Infrastructure ← everything
```

---

## Module Organization Patterns

```move
// ============================================
// PATTERN: ADMIN CAP (Object-based authority)
// Better than checking address == admin
// Cap can be transferred, shared, or burned
// ============================================

module protocol::admin {
    use aptos_framework::object::{Self, Object, ConstructorRef};
    
    struct AdminCap has key {
        // Empty struct - possession = authority
    }
    
    struct OperatorCap has key {
        // Limited authority
        can_pause: bool,
        can_set_fees: bool,
    }
    
    // Create admin cap (only once at deployment)
    public fun create_admin_cap(deployer: &signer): address {
        let constructor_ref = object::create_object(std::signer::address_of(deployer));
        let admin_signer = object::generate_signer(&constructor_ref);
        move_to(&admin_signer, AdminCap {});
        object::address_from_constructor_ref(&constructor_ref)
    }
    
    // Delegate operator
    public fun create_operator(
        admin: &signer,
        admin_cap_addr: address,
        operator_addr: address,
        can_pause: bool,
        can_set_fees: bool,
    ) acquires AdminCap {
        // Verify caller owns admin cap
        assert!(exists<AdminCap>(admin_cap_addr), 1);
        // In practice: verify admin has ownership of admin_cap object
        
        let constructor_ref = object::create_object(operator_addr);
        let op_signer = object::generate_signer(&constructor_ref);
        move_to(&op_signer, OperatorCap { can_pause, can_set_fees });
    }
}

// ============================================
// PATTERN: CONFIGURATION RESOURCE
// Centralize all protocol parameters
// ============================================

module protocol::config {
    friend protocol::pool;
    friend protocol::router;
    friend protocol::governance;
    
    struct ProtocolConfig has key {
        // Fee params
        default_fee_bps: u64,        // 30 = 0.3%
        protocol_fee_share_bps: u64, // Share going to treasury
        
        // Risk params
        max_pool_utilization: u64,   // e.g., 90%
        min_liquidity: u64,          // Minimum LP value
        
        // Oracle params
        oracle_freshness_threshold: u64,  // Max staleness in seconds
        twap_period: u64,                 // TWAP window
        
        // Limits
        max_swap_size: u64,
        daily_volume_limit: u64,
        
        // Emergency
        is_paused: bool,
        pause_guardian: address,
    }
    
    // Governance can update params (after timelock)
    public(friend) fun set_default_fee(config_addr: address, new_fee: u64) acquires ProtocolConfig {
        assert!(new_fee <= 1000, 1); // Max 10%
        borrow_global_mut<ProtocolConfig>(config_addr).default_fee_bps = new_fee;
    }
    
    // Read-only accessors for other modules
    public fun default_fee(config_addr: address): u64 acquires ProtocolConfig {
        borrow_global<ProtocolConfig>(config_addr).default_fee_bps
    }
    
    public fun is_paused(config_addr: address): bool acquires ProtocolConfig {
        borrow_global<ProtocolConfig>(config_addr).is_paused
    }
}

// ============================================
// PATTERN: EVENT BUS (centralized event logging)
// All protocol events in one place for indexers
// ============================================

module protocol::events {
    #[event]
    struct PoolCreated has drop, store {
        pool_id: u64,
        token_x: std::string::String,
        token_y: std::string::String,
        fee_bps: u64,
        creator: address,
        timestamp: u64,
    }
    
    #[event]
    struct LiquidityAdded has drop, store {
        pool_id: u64,
        provider: address,
        amount_x: u64,
        amount_y: u64,
        lp_tokens: u64,
        timestamp: u64,
    }
    
    #[event]
    struct Swap has drop, store {
        pool_id: u64,
        trader: address,
        token_in: std::string::String,
        token_out: std::string::String,
        amount_in: u64,
        amount_out: u64,
        fee: u64,
        timestamp: u64,
    }
    
    #[event]
    struct ParameterUpdated has drop, store {
        param_name: std::string::String,
        old_value: u64,
        new_value: u64,
        updated_by: address,
        timestamp: u64,
    }
    
    public fun emit_swap(
        pool_id: u64,
        trader: address,
        token_in: std::string::String,
        token_out: std::string::String,
        amount_in: u64,
        amount_out: u64,
        fee: u64,
    ) {
        aptos_framework::event::emit(Swap {
            pool_id,
            trader,
            token_in,
            token_out,
            amount_in,
            amount_out,
            fee,
            timestamp: aptos_framework::timestamp::now_microseconds(),
        });
    }
}
```

---

## Upgrade Patterns

```move
// ============================================
// APTOS UPGRADE STRATEGY
//
// Move modules can be upgraded if compiled with
// compatible bytecode changes:
//   Compatible: Add new functions, add new structs
//   Incompatible: Remove functions, change signatures
//                 Change struct fields
//
// Strategy: Store data in Resources, not in module
// ============================================

module protocol::upgradeable_pool {
    // V1 data structure
    struct PoolV1 has key {
        reserve_x: u64,
        reserve_y: u64,
        fee_bps: u64,
        // ... other fields
    }
    
    // V2: Added fields (compatible upgrade)
    // struct PoolV2 has key {
    //     reserve_x: u64,
    //     reserve_y: u64,
    //     fee_bps: u64,
    //     last_k: u128,        // NEW: invariant cache
    //     volume_24h: u64,     // NEW: analytics
    // }
    
    // MIGRATION FUNCTION (called after upgrade)
    // public entry fun migrate_v1_to_v2(
    //     admin: &signer,
    //     pool_addr: address,
    // ) acquires PoolV1 {
    //     let PoolV1 { reserve_x, reserve_y, fee_bps } = move_from<PoolV1>(pool_addr);
    //     let k = (reserve_x as u128) * (reserve_y as u128);
    //     move_to(admin, PoolV2 {
    //         reserve_x, reserve_y, fee_bps,
    //         last_k: k,
    //         volume_24h: 0,
    //     });
    // }
    
    // PROXY PATTERN (immutable core, upgradeable periphery)
    // Core logic is immutable (never upgraded)
    // Periphery (router, aggregator) can be upgraded
    // Users interact only with periphery
    
    struct ProxyConfig has key {
        implementation_version: u64,
        router_addr: address,
        aggregator_addr: address,
        current_config_addr: address,
    }
}

// ============================================
// SUI UPGRADE STRATEGY
// Sui has package versioning built-in
// Different approach: package ID changes on upgrade
// ============================================

// In Sui Move:
// module my_protocol::pool {
//     const VERSION: u64 = 2;
//     
//     struct PoolCap has key {
//         id: UID,
//         version: u64,  // Must match VERSION
//     }
//     
//     // Check version before each operation
//     fun check_version(cap: &PoolCap) {
//         assert!(cap.version == VERSION, EWrongVersion);
//     }
//     
//     // Migration function to update version
//     entry fun migrate(cap: &mut PoolCap, _: &AdminCap) {
//         assert!(cap.version == VERSION - 1, ENotUpgradeable);
//         cap.version = VERSION;
//     }
// }
```

---

## Data Architecture

```move
module protocol::data_architecture {
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // CHOOSING DATA STRUCTURES
    //
    // vector:       Simple list, O(n) search
    // SmartTable:   Hash map, O(1) amortized
    // Table:        O(1) but no iteration
    // OrderedMap:   Sorted, O(log n)
    // ============================================
    
    // BAD: Using vector for O(n) lookups (gas: O(n))
    struct BadRegistry {
        users: vector<address>,
        balances: vector<u64>,
    }
    
    // GOOD: Using SmartTable for O(1) lookups
    struct GoodRegistry has key {
        balances: SmartTable<address, u64>,
    }
    
    // PATTERN: Separation of hot and cold data
    // Hot data: frequently read/written (in SmartTable)
    // Cold data: historical/audit (in events, not on-chain)
    
    struct HotData has key {
        // Frequently accessed
        current_balances: SmartTable<address, u64>,
        pool_reserves: SmartTable<u64, PoolReserves>,
        active_positions: SmartTable<u64, Position>,
    }
    
    // Historical data → EVENTS (not on-chain storage)
    // #[event] struct SwapHistory { ... }
    // This saves gas: events are much cheaper than storage
    
    struct PoolReserves has store, copy {
        x: u64,
        y: u64,
        last_update: u64,
    }
    
    struct Position has store, drop {
        owner: address,
        pool_id: u64,
        size: u64,
        entry_price: u64,
    }
    
    // PATTERN: Pagination-friendly data
    // For large datasets that need iteration
    struct PaginatedList<T: store> has key {
        items: vector<T>,
        total_count: u64,
        page_size: u64,
    }
    
    public fun get_page<T: store>(
        list: &PaginatedList<T>,
        page: u64,
    ): vector<T> {
        let start = page * list.page_size;
        let end = std::u64::min(start + list.page_size, std::vector::length(&list.items));
        
        if (start >= end) return vector::empty();
        
        let mut result = vector::empty<T>();
        let mut i = start;
        // Note: Can't return references, need copy ability
        i // placeholder
    }
    
    // PATTERN: Versioned resources (for migrations)
    struct VersionedConfig has key {
        version: u64,
        data: vector<u8>,  // BCS-encoded config, version-specific
    }
}
```

---

## Integration Patterns

```typescript
// TypeScript SDK for protocol integration

import { Aptos, AptosConfig, Network, Account } from "@aptos-labs/ts-sdk";

// ============================================
// PROTOCOL SDK STRUCTURE
// ============================================

class ProtocolSDK {
  private aptos: Aptos;
  private moduleAddress: string;

  constructor(network: Network, moduleAddress: string) {
    this.aptos = new Aptos(new AptosConfig({ network }));
    this.moduleAddress = moduleAddress;
  }

  // ============================================
  // READ FUNCTIONS (View)
  // ============================================

  async getPoolInfo(poolId: number): Promise<PoolInfo> {
    const result = await this.aptos.view({
      payload: {
        function: `${this.moduleAddress}::pool::get_pool_info`,
        typeArguments: [],
        functionArguments: [poolId],
      },
    });
    
    return {
      id: poolId,
      reserveX: BigInt(result[0] as string),
      reserveY: BigInt(result[1] as string),
      feeBps: result[2] as number,
      lpTokenSupply: BigInt(result[3] as string),
    };
  }

  async getSwapOutput(
    poolId: number,
    tokenIn: 'X' | 'Y',
    amountIn: bigint,
  ): Promise<bigint> {
    const result = await this.aptos.view({
      payload: {
        function: `${this.moduleAddress}::pool::get_amount_out`,
        typeArguments: [],
        functionArguments: [poolId, tokenIn === 'X', amountIn.toString()],
      },
    });
    return BigInt(result[0] as string);
  }

  // ============================================
  // WRITE FUNCTIONS (Entry)
  // ============================================

  async swap(
    account: Account,
    poolId: number,
    tokenIn: string,
    tokenOut: string,
    amountIn: bigint,
    minAmountOut: bigint,
    slippagePercent: number = 0.5,
  ) {
    const expectedOut = await this.getSwapOutput(poolId, 'X', amountIn);
    const minOut = expectedOut * BigInt(Math.floor((100 - slippagePercent) * 100)) / 10000n;
    
    const txn = await this.aptos.transaction.build.simple({
      sender: account.accountAddress,
      data: {
        function: `${this.moduleAddress}::router::swap_exact_input`,
        typeArguments: [tokenIn, tokenOut],
        functionArguments: [
          poolId,
          amountIn,
          minOut,
          Math.floor(Date.now() / 1000) + 60, // 60 second deadline
        ],
      },
    });

    const signed = this.aptos.transaction.sign({ signer: account, transaction: txn });
    const submitted = await this.aptos.transaction.submit.simple({
      transaction: txn,
      senderAuthenticator: signed,
    });
    
    return this.aptos.waitForTransaction({ transactionHash: submitted.hash });
  }

  async addLiquidity(
    account: Account,
    poolId: number,
    amountX: bigint,
    amountY: bigint,
    slippagePercent: number = 0.5,
  ) {
    const minAmountX = amountX * BigInt(Math.floor((100 - slippagePercent) * 100)) / 10000n;
    const minAmountY = amountY * BigInt(Math.floor((100 - slippagePercent) * 100)) / 10000n;

    const txn = await this.aptos.transaction.build.simple({
      sender: account.accountAddress,
      data: {
        function: `${this.moduleAddress}::pool::add_liquidity`,
        typeArguments: [],
        functionArguments: [poolId, amountX, amountY, minAmountX, minAmountY],
      },
    });

    const signed = this.aptos.transaction.sign({ signer: account, transaction: txn });
    return this.aptos.transaction.submit.simple({
      transaction: txn,
      senderAuthenticator: signed,
    });
  }

  // ============================================
  // EVENT STREAMING (Real-time)
  // ============================================

  async subscribeToSwaps(
    poolId: number,
    callback: (event: SwapEvent) => void,
  ) {
    // Poll for events (websocket in production)
    setInterval(async () => {
      const events = await this.aptos.getEvents({
        options: {
          where: {
            account_address: { _eq: this.moduleAddress },
            indexed_type: { _eq: `${this.moduleAddress}::events::Swap` },
          },
        },
      });
      
      for (const event of events) {
        if (event.data.pool_id === poolId) {
          callback(event.data as SwapEvent);
        }
      }
    }, 1000);
  }
}

interface PoolInfo {
  id: number;
  reserveX: bigint;
  reserveY: bigint;
  feeBps: number;
  lpTokenSupply: bigint;
}

interface SwapEvent {
  pool_id: number;
  trader: string;
  amount_in: string;
  amount_out: string;
  timestamp: string;
}
```

---

## Production Checklist

```
PRE-LAUNCH PROTOCOL CHECKLIST

SECURITY
  [ ] 3+ independent audits from reputable firms
  [ ] Bug bounty program active (Immunefi, HackerOne)
  [ ] Move Prover specs for core invariants
  [ ] Fuzz testing on math functions (10M+ iterations)
  [ ] Economic attack simulation (flash loan, oracle manip)
  [ ] Admin key is hardware multisig (Ledger 3/5)
  [ ] Guardian key separate from admin
  [ ] Timelock on all parameter changes (48h min)
  [ ] Emergency pause tested end-to-end
  
SMART CONTRACTS
  [ ] All modules verified on explorer
  [ ] Move.toml pinned to specific commit hashes
  [ ] Named addresses use actual deployed addresses
  [ ] No debug/test code in production
  [ ] Gas limits tested on mainnet-like conditions
  [ ] Upgrade policy documented and tested
  
TESTING
  [ ] Unit tests > 90% coverage
  [ ] Integration tests covering all user flows
  [ ] Testnet running 2+ weeks without issues
  [ ] Load testing (high transaction volume)
  [ ] Chaos testing (node failures, network partitions)
  
INFRASTRUCTURE
  [ ] Multiple RPC endpoints (failover)
  [ ] Monitoring and alerting (PagerDuty/OpsGenie)
  [ ] 24/7 on-call rotation
  [ ] Incident response runbook documented
  [ ] Backup admin keys in cold storage
  [ ] Regular security reviews of off-chain systems
  
FRONTEND/SDK
  [ ] Multiple CDN endpoints
  [ ] Read-only fallback mode (if backend down)
  [ ] Transaction simulation before submission
  [ ] Clear error messages with recovery steps
  [ ] Mobile responsive
  [ ] Accessibility (WCAG 2.1 AA)
  
LEGAL/COMPLIANCE
  [ ] Legal review in target jurisdictions
  [ ] Terms of Service documented
  [ ] Privacy policy (GDPR if EU users)
  [ ] Token not classified as security (legal opinion)
  [ ] Treasury structure (foundation vs DAO)
  
LAUNCH STRATEGY
  [ ] Soft launch: Invite-only, low TVL cap
  [ ] Bug bounty starts before public launch
  [ ] Liquidity bootstrapping plan
  [ ] Marketing coordinated with launch
  [ ] Community channels live (Discord, Telegram)
  [ ] Response team ready for incident comms
  
POST-LAUNCH
  [ ] TVL monitoring with alerts
  [ ] Transaction anomaly detection
  [ ] Regular security reviews (quarterly)
  [ ] Governance decentralization roadmap
  [ ] Upgrade governance on-chain (not admin key)
```

---

**ก่อนหน้า**: [Part 69 - Multi-Chain Tokens ←](part-69-multichain-tokens.md)
**ต่อไป**: [Part 71 - Automated Market Maker Design →](part-71-amm-design.md)
