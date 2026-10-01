# Part 50: Professional Project Architecture

## สารบัญ
- [Monorepo Structure](#monorepo-structure)
- [Module Design Principles](#module-design-principles)
- [Dependency Management](#dependency-management)
- [Testing Strategy](#testing-strategy)
- [Frontend Integration](#frontend-integration)
- [ตัวอย่าง: Full-Stack DeFi Project](#ตัวอย่าง-full-stack-defi-project)

---

## Monorepo Structure

```
myprotocol/
├── contracts/               # Move contracts
│   ├── core/                # Core module
│   │   ├── Move.toml
│   │   └── sources/
│   │       ├── config.move
│   │       ├── events.move
│   │       └── errors.move
│   ├── token/               # Protocol token
│   │   ├── Move.toml
│   │   └── sources/
│   │       ├── token.move
│   │       └── vesting.move
│   ├── amm/                 # AMM module
│   │   ├── Move.toml
│   │   └── sources/
│   │       ├── pool.move
│   │       ├── math.move
│   │       └── fee.move
│   ├── lending/             # Lending module
│   │   ├── Move.toml
│   │   └── sources/
│   │       ├── market.move
│   │       ├── interest.move
│   │       └── liquidation.move
│   └── governance/          # Governance
│       ├── Move.toml
│       └── sources/
│           ├── dao.move
│           └── timelock.move
│
├── sdk/                     # TypeScript SDK
│   ├── src/
│   │   ├── client.ts        # AptosClient wrapper
│   │   ├── amm/             # AMM interactions
│   │   ├── lending/         # Lending interactions
│   │   └── types/           # TypeScript types
│   ├── package.json
│   └── tsconfig.json
│
├── frontend/                # React/Next.js app
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   └── hooks/
│   └── package.json
│
├── scripts/                 # Deployment & admin scripts
│   ├── deploy/
│   │   ├── 01-core.ts
│   │   ├── 02-token.ts
│   │   └── 03-amm.ts
│   └── admin/
│       ├── set-fee.ts
│       └── emergency-pause.ts
│
├── tests/                   # Integration tests (off-chain)
│   ├── amm.test.ts
│   └── lending.test.ts
│
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
│
└── docs/
    ├── architecture.md
    └── security.md
```

---

## Module Design Principles

```move
// ============================================
// PRINCIPLE 1: Single Responsibility
// Each module does ONE thing well
// ============================================

// WRONG: One god module
module bad::everything {
    // AMM logic
    // Lending logic
    // Governance logic
    // Token logic
    // All mixed together!
}

// RIGHT: Separate concerns
module good::amm { /* only AMM */ }
module good::lending { /* only lending */ }
module good::governance { /* only governance */ }
module good::token { /* only token */ }
```

```move
// ============================================
// PRINCIPLE 2: Layered Architecture
// ============================================

// Layer 1: Primitives (no dependencies)
module proto::math {
    public fun mul_div(a: u64, b: u64, c: u64): u64 {
        ((a as u128) * (b as u128) / (c as u128)) as u64
    }
    
    public fun sqrt(x: u64): u64 {
        if (x == 0) return 0;
        let mut z = x;
        let mut y = (x + 1) / 2;
        while (y < z) {
            z = y;
            y = (y + x / y) / 2;
        };
        z
    }
    
    public fun min(a: u64, b: u64): u64 {
        if (a < b) a else b
    }
    
    public fun max(a: u64, b: u64): u64 {
        if (a > b) a else b
    }
}

// Layer 2: Core types (depends on primitives)
module proto::core {
    use proto::math;
    
    // Define core data structures
    struct ProtocolConfig has key {
        admin: address,
        paused: bool,
        fee_treasury: address,
    }
    
    // Module errors (centralized)
    const E_NOT_ADMIN: u64 = 1;
    const E_PAUSED: u64 = 2;
    
    public fun assert_not_paused(config: &ProtocolConfig) {
        assert!(!config.paused, E_PAUSED);
    }
    
    public fun assert_admin(config: &ProtocolConfig, caller: address) {
        assert!(caller == config.admin, E_NOT_ADMIN);
    }
}

// Layer 3: Domain modules (depends on core)
module proto::amm {
    use proto::core;
    use proto::math;
    
    // Uses core config checks
    // Uses math primitives
    // Implements AMM-specific logic
}

// Layer 4: Composition modules (depends on domain)
module proto::leverage_farming {
    use proto::amm;
    use proto::lending;
    
    // Composes amm + lending for leveraged farming
}
```

```move
// ============================================
// PRINCIPLE 3: Explicit Error Handling
// ============================================

module proto::errors {
    // Convention: module_id * 1000 + local_code
    
    // Core errors (0xxx)
    const E_NOT_ADMIN: u64 = 1;
    const E_PAUSED: u64 = 2;
    const E_REENTRANT: u64 = 3;
    const E_NOT_INITIALIZED: u64 = 4;
    
    // AMM errors (1xxx)
    const E_AMM_INSUFFICIENT_LIQUIDITY: u64 = 1001;
    const E_AMM_SLIPPAGE: u64 = 1002;
    const E_AMM_DEADLINE: u64 = 1003;
    const E_AMM_ZERO_AMOUNT: u64 = 1004;
    
    // Lending errors (2xxx)
    const E_LENDING_HEALTH_FACTOR: u64 = 2001;
    const E_LENDING_MAX_BORROW: u64 = 2002;
    const E_LENDING_ZERO_COLLATERAL: u64 = 2003;
    
    // Governance errors (3xxx)
    const E_GOV_PROPOSAL_NOT_ACTIVE: u64 = 3001;
    const E_GOV_ALREADY_VOTED: u64 = 3002;
    const E_GOV_INSUFFICIENT_QUORUM: u64 = 3003;
    
    public fun not_admin(): u64 { E_NOT_ADMIN }
    public fun paused(): u64 { E_PAUSED }
    // etc.
}
```

---

## Dependency Management

```toml
# contracts/amm/Move.toml
[package]
name = "ProtoAMM"
version = "1.0.0"

[addresses]
proto_amm = "_"          # Set at compile time
proto_core = "_"         # Sibling package

[dependencies]
AptosFramework = { git = "...", rev = "mainnet" }
AptosStdlib = { git = "...", rev = "mainnet" }

# Local dependency on core module
ProtoCore = { local = "../core" }
ProtoMath = { local = "../math" }

# External dependency (e.g., oracle)
Pyth = { git = "https://github.com/pyth-network/...", rev = "main" }
```

```
Dependency Rules:
  1. No circular dependencies (A→B→A is illegal)
  2. Dependency tree should be a DAG (directed acyclic graph)
  3. Core modules should have minimal dependencies
  4. External dependencies pinned to specific revision
  5. Test-only dependencies use [dev-dependencies]

Dependency Graph:
  framework
      ↑
    math
      ↑
    core ← config, errors, events
      ↑
  ┌──────┬──────┐
 amm  lending governance
      ↑
  composition
  (leverage farming, etc.)
```

---

## Testing Strategy

```move
// ============================================
// Test Organization
// ============================================

module proto::amm_tests {
    #[test_only]
    use std::signer;
    #[test_only]
    use aptos_framework::account;
    #[test_only]
    use aptos_framework::timestamp;
    
    // ============================================
    // Test setup helpers
    // ============================================
    
    #[test_only]
    struct TestEnv {
        admin: signer,
        alice: signer,
        bob: signer,
    }
    
    #[test_only]
    fun setup(): TestEnv {
        let framework = account::create_account_for_test(@0x1);
        timestamp::set_time_has_started_for_testing(&framework);
        
        TestEnv {
            admin: account::create_account_for_test(@0xADMIN),
            alice: account::create_account_for_test(@0xALICE),
            bob: account::create_account_for_test(@0xBOB),
        }
    }
    
    // ============================================
    // Unit tests: single function
    // ============================================
    
    #[test]
    fun test_compute_swap_out_basic() {
        // Given: 1000 X, 1000 Y reserves, swap 100 X
        let out = proto::amm::compute_out(1000, 1000, 100, 30);
        // Expected: ~90.66 Y (constant product with 0.3% fee)
        assert!(out >= 90 && out <= 91, 1);
    }
    
    #[test]
    fun test_compute_swap_respects_fee() {
        // Without fee: out = 1000 * 100 / (1000 + 100) = 90.9
        let out_no_fee = proto::amm::compute_out(1000, 1000, 100, 0);
        // With fee: should get less
        let out_with_fee = proto::amm::compute_out(1000, 1000, 100, 30);
        assert!(out_with_fee < out_no_fee, 1);
    }
    
    #[test]
    #[expected_failure(abort_code = 1001)]  // E_AMM_INSUFFICIENT_LIQUIDITY
    fun test_swap_fails_with_zero_reserve() {
        proto::amm::compute_out(0, 1000, 100, 30);  // Should abort
    }
    
    // ============================================
    // Integration tests: full workflow
    // ============================================
    
    #[test]
    fun test_add_swap_remove_liquidity() {
        let env = setup();
        let admin_addr = signer::address_of(&env.admin);
        
        // Step 1: Create pool
        proto::amm::create_pool<TokenA, TokenB>(&env.admin);
        
        // Step 2: Add liquidity
        proto::amm::add_liquidity<TokenA, TokenB>(
            &env.alice,
            admin_addr,
            1000,
            1000,
            990,  // min_x
            990,  // min_y
        );
        
        // Step 3: Verify LP tokens received
        let lp_balance = coin::balance<LpToken<TokenA, TokenB>>(
            signer::address_of(&env.alice)
        );
        assert!(lp_balance > 0, 1);
        
        // Step 4: Swap
        proto::amm::swap_x_for_y<TokenA, TokenB>(
            &env.bob,
            admin_addr,
            100,
            90,   // min_out
        );
        
        // Step 5: Remove liquidity
        proto::amm::remove_liquidity<TokenA, TokenB>(
            &env.alice,
            admin_addr,
            lp_balance,
        );
        
        // Verify: Alice got back >= her initial deposit
        // (plus her share of fees)
        let a_balance = coin::balance<TokenA>(signer::address_of(&env.alice));
        assert!(a_balance >= 1000, 2);
    }
    
    // ============================================
    // Property-based tests: invariant checking
    // ============================================
    
    #[test]
    fun test_k_invariant_preserved() {
        let env = setup();
        let admin_addr = signer::address_of(&env.admin);
        
        // Create pool with initial liquidity
        // ...
        
        // Get initial k
        let (r_x, r_y) = proto::amm::get_reserves(admin_addr);
        let k_before = (r_x as u128) * (r_y as u128);
        
        // Perform swap
        // ...
        
        // k should not decrease (fees cause slight increase)
        let (r_x2, r_y2) = proto::amm::get_reserves(admin_addr);
        let k_after = (r_x2 as u128) * (r_y2 as u128);
        assert!(k_after >= k_before, 1);
    }
    
    #[test_only]
    struct TokenA {}
    #[test_only]
    struct TokenB {}
    #[test_only]
    struct LpToken<phantom X, phantom Y> {}
}
```

---

## Frontend Integration

```typescript
// sdk/src/client.ts
import { Aptos, AptosConfig, Network } from "@aptos-labs/ts-sdk";

const PROTOCOL_ADDR = process.env.NEXT_PUBLIC_PROTOCOL_ADDR!;

export const aptosClient = new Aptos(
  new AptosConfig({ network: Network.MAINNET })
);

// ============================================
// AMM SDK
// ============================================

export interface PoolInfo {
  reserveX: bigint;
  reserveY: bigint;
  lpSupply: bigint;
  feeBps: number;
}

export async function getPoolInfo(poolAddr: string): Promise<PoolInfo> {
  const resource = await aptosClient.getAccountResource({
    accountAddress: poolAddr,
    resourceType: `${PROTOCOL_ADDR}::amm::Pool`,
  });
  
  return {
    reserveX: BigInt(resource.reserve_x),
    reserveY: BigInt(resource.reserve_y),
    lpSupply: BigInt(resource.lp_supply),
    feeBps: Number(resource.fee_bps),
  };
}

export function computeSwapOut(
  reserveIn: bigint,
  reserveOut: bigint,
  amountIn: bigint,
  feeBps: number,
): bigint {
  const feeFactor = BigInt(10000 - feeBps);
  const amountInWithFee = amountIn * feeFactor;
  const numerator = reserveOut * amountInWithFee;
  const denominator = reserveIn * 10000n + amountInWithFee;
  return numerator / denominator;
}

export async function buildSwapTransaction(
  userAddr: string,
  poolAddr: string,
  amountIn: bigint,
  minOut: bigint,
  deadline: number,
): Promise<any> {
  return {
    function: `${PROTOCOL_ADDR}::amm::swap_x_for_y`,
    functionArguments: [
      poolAddr,
      amountIn.toString(),
      minOut.toString(),
      deadline.toString(),
    ],
  };
}

// ============================================
// React Hooks
// ============================================

// hooks/usePool.ts
import { useQuery } from "@tanstack/react-query";

export function usePoolInfo(poolAddr: string | undefined) {
  return useQuery({
    queryKey: ["pool", poolAddr],
    queryFn: () => getPoolInfo(poolAddr!),
    enabled: !!poolAddr,
    staleTime: 5_000,  // Refresh every 5 seconds
    refetchInterval: 5_000,
  });
}

export function useSwapQuote(
  pool: PoolInfo | undefined,
  amountIn: string,
) {
  const amountInBig = BigInt(amountIn || "0");
  
  if (!pool || amountInBig === 0n) {
    return { amountOut: 0n, priceImpact: 0 };
  }
  
  const amountOut = computeSwapOut(
    pool.reserveX,
    pool.reserveY,
    amountInBig,
    pool.feeBps,
  );
  
  // Price impact = (spot - execution) / spot
  const spotPrice = Number(pool.reserveY) / Number(pool.reserveX);
  const executionPrice = Number(amountOut) / Number(amountInBig);
  const priceImpact = Math.abs(1 - executionPrice / spotPrice) * 100;
  
  return { amountOut, priceImpact };
}
```

---

## ตัวอย่าง: Full-Stack DeFi Project

```move
// contracts/amm/sources/pool.move
// Production-grade AMM pool

module proto_amm::pool {
    use std::signer;
    use aptos_framework::coin::{Self, Coin, MintCapability, BurnCapability};
    use aptos_framework::timestamp;
    use aptos_framework::event;
    use proto_core::config;
    use proto_math::math;
    
    // ============================================
    // Types
    // ============================================
    
    struct LPToken<phantom X, phantom Y> {}
    
    struct Pool<phantom X, phantom Y> has key {
        reserve_x: u64,
        reserve_y: u64,
        lp_supply: u64,
        fee_bps: u64,
        
        // TWAP tracking
        last_price_cumulative_x: u128,
        last_price_cumulative_y: u128,
        last_block_timestamp: u64,
        
        // LP token capabilities
        lp_mint_cap: MintCapability<LPToken<X, Y>>,
        lp_burn_cap: BurnCapability<LPToken<X, Y>>,
        
        // Protocol fee tracking
        protocol_fee_x: u64,
        protocol_fee_y: u64,
    }
    
    // ============================================
    // Events
    // ============================================
    
    #[event]
    struct PoolCreated has drop, store {
        pool_addr: address,
        fee_bps: u64,
    }
    
    #[event]
    struct LiquidityAdded has drop, store {
        provider: address,
        pool_addr: address,
        amount_x: u64,
        amount_y: u64,
        lp_minted: u64,
    }
    
    #[event]
    struct Swapped has drop, store {
        user: address,
        pool_addr: address,
        amount_in: u64,
        amount_out: u64,
        x_to_y: bool,
    }
    
    // ============================================
    // Core Functions
    // ============================================
    
    public entry fun create_pool<X, Y>(
        creator: &signer,
        fee_bps: u64,
    ) {
        assert!(!exists<Pool<X, Y>>(signer::address_of(creator)), 1);
        assert!(fee_bps <= 100, 2);  // Max 1%
        
        let (lp_burn_cap, lp_freeze_cap, lp_mint_cap) = coin::initialize<LPToken<X, Y>>(
            creator,
            std::string::utf8(b"LP Token"),
            std::string::utf8(b"LP"),
            8,
            true,
        );
        coin::destroy_freeze_cap(lp_freeze_cap);
        
        move_to(creator, Pool<X, Y> {
            reserve_x: 0,
            reserve_y: 0,
            lp_supply: 0,
            fee_bps,
            last_price_cumulative_x: 0,
            last_price_cumulative_y: 0,
            last_block_timestamp: timestamp::now_seconds(),
            lp_mint_cap,
            lp_burn_cap,
            protocol_fee_x: 0,
            protocol_fee_y: 0,
        });
        
        event::emit(PoolCreated {
            pool_addr: signer::address_of(creator),
            fee_bps,
        });
    }
    
    public entry fun add_liquidity<X, Y>(
        provider: &signer,
        pool_addr: address,
        amount_x: u64,
        amount_y: u64,
        min_lp: u64,
    ) acquires Pool {
        let pool = borrow_global_mut<Pool<X, Y>>(pool_addr);
        
        let (actual_x, actual_y) = if (pool.lp_supply == 0) {
            // Initial: use full amounts
            (amount_x, amount_y)
        } else {
            // Maintain ratio
            let optimal_y = amount_x * pool.reserve_y / pool.reserve_x;
            if (optimal_y <= amount_y) {
                (amount_x, optimal_y)
            } else {
                let optimal_x = amount_y * pool.reserve_x / pool.reserve_y;
                (optimal_x, amount_y)
            }
        };
        
        let lp_minted = if (pool.lp_supply == 0) {
            math::sqrt(actual_x * actual_y)  // Geometric mean
        } else {
            math::min(
                actual_x * pool.lp_supply / pool.reserve_x,
                actual_y * pool.lp_supply / pool.reserve_y,
            )
        };
        assert!(lp_minted >= min_lp, 3);
        
        // Transfer tokens to pool
        let coin_x = coin::withdraw<X>(provider, actual_x);
        let coin_y = coin::withdraw<Y>(provider, actual_y);
        coin::deposit<X>(pool_addr, coin_x);
        coin::deposit<Y>(pool_addr, coin_y);
        
        // Mint LP tokens
        let lp_coins = coin::mint<LPToken<X, Y>>(lp_minted, &pool.lp_mint_cap);
        coin::deposit<LPToken<X, Y>>(signer::address_of(provider), lp_coins);
        
        update_reserves_and_twap(pool, actual_x, actual_y);
        pool.lp_supply = pool.lp_supply + lp_minted;
        
        event::emit(LiquidityAdded {
            provider: signer::address_of(provider),
            pool_addr,
            amount_x: actual_x,
            amount_y: actual_y,
            lp_minted,
        });
    }
    
    public entry fun swap_x_for_y<X, Y>(
        user: &signer,
        pool_addr: address,
        amount_in: u64,
        min_out: u64,
        deadline: u64,
    ) acquires Pool {
        assert!(timestamp::now_seconds() <= deadline, 4);
        
        let pool = borrow_global_mut<Pool<X, Y>>(pool_addr);
        
        let amount_out = compute_out(
            pool.reserve_x,
            pool.reserve_y,
            amount_in,
            pool.fee_bps,
        );
        assert!(amount_out >= min_out, 5);
        
        // Verify k invariant after swap
        let new_k = (pool.reserve_x + amount_in) as u128 
                  * (pool.reserve_y - amount_out) as u128;
        let old_k = pool.reserve_x as u128 * pool.reserve_y as u128;
        assert!(new_k >= old_k, 6);
        
        // Transfer
        let coin_in = coin::withdraw<X>(user, amount_in);
        coin::deposit<X>(pool_addr, coin_in);
        let coin_out = coin::withdraw<Y>(
            &create_signer(pool_addr),
            amount_out
        );
        coin::deposit<Y>(signer::address_of(user), coin_out);
        
        update_reserves_and_twap(pool, amount_in, 0u64);
        pool.reserve_y = pool.reserve_y - amount_out;
        
        event::emit(Swapped {
            user: signer::address_of(user),
            pool_addr,
            amount_in,
            amount_out,
            x_to_y: true,
        });
    }
    
    fun compute_out(reserve_in: u64, reserve_out: u64, amount_in: u64, fee_bps: u64): u64 {
        let fee_factor = 10_000 - fee_bps;
        let in_with_fee = (amount_in as u128) * (fee_factor as u128);
        let numerator = (reserve_out as u128) * in_with_fee;
        let denominator = (reserve_in as u128) * 10_000u128 + in_with_fee;
        (numerator / denominator) as u64
    }
    
    fun update_reserves_and_twap<X, Y>(
        pool: &mut Pool<X, Y>,
        new_x_delta: u64,
        _new_y_delta: u64,
    ) {
        let now = timestamp::now_seconds();
        let elapsed = now - pool.last_block_timestamp;
        
        if (elapsed > 0 && pool.reserve_x > 0 && pool.reserve_y > 0) {
            // Price_X_in_Y = reserve_y / reserve_x * PRECISION
            pool.last_price_cumulative_x = pool.last_price_cumulative_x
                + ((pool.reserve_y as u128) * 1_000_000 / (pool.reserve_x as u128))
                  * (elapsed as u128);
                  
            pool.last_price_cumulative_y = pool.last_price_cumulative_y
                + ((pool.reserve_x as u128) * 1_000_000 / (pool.reserve_y as u128))
                  * (elapsed as u128);
        };
        
        pool.reserve_x = pool.reserve_x + new_x_delta;
        pool.last_block_timestamp = now;
    }
    
    // ============================================
    // View functions
    // ============================================
    
    #[view]
    public fun get_reserves<X, Y>(pool_addr: address): (u64, u64) acquires Pool {
        let pool = borrow_global<Pool<X, Y>>(pool_addr);
        (pool.reserve_x, pool.reserve_y)
    }
    
    #[view]
    public fun quote_swap_x_for_y<X, Y>(
        pool_addr: address,
        amount_in: u64,
    ): u64 acquires Pool {
        let pool = borrow_global<Pool<X, Y>>(pool_addr);
        compute_out(pool.reserve_x, pool.reserve_y, amount_in, pool.fee_bps)
    }
    
    native fun create_signer(addr: address): signer;
}
```

---

## สรุป Professional Architecture

```
Architecture Checklist:

MODULE STRUCTURE:
  ✓ Single-responsibility modules
  ✓ Layered architecture (primitives → core → domain → composition)
  ✓ Centralized error codes
  ✓ Events for all state changes
  ✓ Separation of view and mutation

TESTING:
  ✓ Unit tests: every function, including error cases
  ✓ Integration tests: full workflows
  ✓ Property tests: invariants
  ✓ Gas benchmarks: performance monitoring
  ✓ >90% code coverage

FRONTEND:
  ✓ SDK wraps all contract interactions
  ✓ React hooks for data fetching
  ✓ TWAP quotes (not spot) for display
  ✓ Slippage + deadline UI controls
  ✓ Error handling + retry logic

DEPLOYMENT:
  ✓ Staged deployment scripts
  ✓ CI/CD with tests before deploy
  ✓ Testnet → mainnet flow
  ✓ Admin key management
  ✓ Monitoring from day 1

SECURITY:
  ✓ Security audit before mainnet
  ✓ Bug bounty program
  ✓ Emergency pause mechanism
  ✓ Multi-sig admin
  ✓ Timelock for parameter changes
```

---

**ก่อนหน้า**: [Part 49 - Sui Object Runtime ←](part-49-sui-object-runtime.md)
**ต่อไป**: [Part 51 - World-class: MEV Strategies →](../worldclass/part-51-mev-strategies.md)
