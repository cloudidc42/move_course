# Part 91: Yield Aggregator Strategies

## สารบัญ
- [Yield Aggregator Architecture](#yield-aggregator-architecture)
- [Strategy Interface](#strategy-interface)
- [Vault Contract](#vault-contract)
- [Auto-Compound Engine](#auto-compound-engine)
- [Strategy Router](#strategy-router)
- [Risk-Adjusted Allocation](#risk-adjusted-allocation)

---

## Yield Aggregator Architecture

```
YIELD AGGREGATOR SYSTEM

             User Deposits
                  │
                  ▼
         ┌─────────────────┐
         │    Vault (yToken)│   ← ERC-4626 style
         │  price_per_share │   ← Grows as yield accrues
         └────────┬────────┘
                  │ allocates funds
                  ▼
         ┌─────────────────┐
         │  Strategy Router │   ← Picks best strategies
         └────┬────┬────┬──┘
              │    │    │
              ▼    ▼    ▼
         ┌─────┐ ┌────┐ ┌──────┐
         │Strat│ │Strat│ │Strat │
         │ A   │ │ B  │ │  C   │
         │DEX  │ │Lend│ │Farm  │
         │LP   │ │ing │ │Yield │
         └─────┘ └────┘ └──────┘
              │    │    │
              └────┴────┘
                   │ auto-compound
                   ▼
              Rewards collected,
              swapped to base asset,
              redeposited

KEY METRICS
  TVL: Total Value Locked (in base asset)
  APY: Annual Percentage Yield (compounded)
  APR: Annual Percentage Rate (simple)
  APY = (1 + APR/n)^n - 1 where n = compounds per year
  
  Compound daily: APY = (1 + APR/365)^365 - 1
  Compound every block: APY ≈ e^APR - 1 (continuous)

VAULT SHARE MATH
  price_per_share = total_assets / total_shares
  shares_to_mint = deposit / price_per_share
  assets_to_return = shares * price_per_share
  
  Example:
    Initial: 1000 USDC, 1000 shares → price = 1.0
    After yield: 1100 USDC, 1000 shares → price = 1.1
    New deposit: 100 USDC → mint 100/1.1 = 90.9 shares
    Withdraw 100 shares: get 100 * 1.1 = 110 USDC
```

---

## Strategy Interface

```move
// ============================================
// STRATEGY INTERFACE: All strategies implement this
// ============================================

module protocol::strategy_interface {
    
    /// Every strategy must expose these capabilities
    /// Implemented via convention (Move has no interfaces)
    
    /// Returns current APR in basis points (annualized)
    /// Used by router to allocate funds optimally
    // #[view]
    // public fun get_current_apr(strategy_addr: address): u64
    
    /// Returns total assets managed by this strategy
    // #[view]
    // public fun total_assets(strategy_addr: address): u64
    
    /// Deposit assets into strategy, returns shares/receipt
    // public fun deposit(amount: u64): u64
    
    /// Withdraw assets from strategy
    // public fun withdraw(shares: u64): u64
    
    /// Harvest rewards and compound
    // public fun harvest(): u64  // Returns newly compounded amount
    
    /// Emergency withdraw (skip compound, just get funds back)
    // public fun emergency_withdraw(): u64

    /// Strategy health (0 = healthy, non-zero = problem code)
    // #[view]
    // public fun health_check(): u64
}

// ============================================
// CONCRETE STRATEGY: LP Farming
// Deposits into AMM pool, earns fees + farm rewards
// ============================================

module protocol::lp_farm_strategy {
    use aptos_framework::timestamp;
    
    const PRECISION: u128 = 1_000_000_000;
    
    struct StrategyState has key {
        // Underlying pool reference
        pool_addr: address,
        farm_addr: address,
        
        // Asset tracking
        total_lp_tokens: u64,
        total_shares: u64,
        
        // Reward tracking
        reward_token_addr: address,
        last_harvest: u64,
        
        // Performance
        lifetime_yield: u128,
        harvest_count: u64,
        
        // Risk limits
        max_tvl: u64,
        min_apr_bps: u64,     // Minimum acceptable APR
    }
    
    /// Deposit base token, get strategy shares
    public fun deposit(
        state: &mut StrategyState,
        amount: u64,
    ): u64 {
        assert!(amount > 0, 1);
        
        let total_before = total_assets_internal(state);
        
        // Add liquidity to pool → get LP tokens
        // let lp_received = pool::add_single_side(state.pool_addr, amount);
        let lp_received = amount * 990 / 1000;  // Simplified: ~1% slippage
        
        // Stake LP in farm
        // farm::stake(state.farm_addr, lp_received);
        
        // Calculate shares to mint
        let shares = if (state.total_shares == 0) {
            lp_received  // 1:1 for first deposit
        } else {
            (lp_received as u128) * (state.total_shares as u128) / (total_before as u128)
        };
        
        state.total_lp_tokens = state.total_lp_tokens + lp_received;
        state.total_shares = state.total_shares + (shares as u64);
        
        (shares as u64)
    }
    
    /// Withdraw: burn shares, receive assets
    public fun withdraw(
        state: &mut StrategyState,
        shares: u64,
    ): u64 {
        assert!(shares > 0 && shares <= state.total_shares, 2);
        
        // Calculate LP to withdraw (proportional)
        let lp_to_withdraw = (shares as u128) * (state.total_lp_tokens as u128)
            / (state.total_shares as u128);
        
        // Unstake from farm
        // farm::unstake(state.farm_addr, lp_to_withdraw as u64);
        
        // Remove liquidity from pool → get base token
        // let base_received = pool::remove_single_side(state.pool_addr, lp_to_withdraw as u64);
        let base_received = (lp_to_withdraw as u64) * 990 / 1000;  // Simplified
        
        state.total_lp_tokens = state.total_lp_tokens - (lp_to_withdraw as u64);
        state.total_shares = state.total_shares - shares;
        
        base_received
    }
    
    /// Harvest farm rewards and compound
    /// Returns extra assets added to TVL
    public fun harvest(state: &mut StrategyState): u64 {
        let now = timestamp::now_microseconds();
        
        // Claim farm rewards
        // let reward_amount = farm::claim_rewards(state.farm_addr);
        let reward_amount = 1000u64;  // Placeholder
        
        if (reward_amount == 0) return 0;
        
        // Swap reward token → base token
        // let base_amount = dex::swap(state.reward_token_addr, BASE_TOKEN, reward_amount);
        let base_amount = reward_amount * 95 / 100;  // Simplified: 5% slippage
        
        // Add back to pool (compound)
        // let new_lp = pool::add_single_side(state.pool_addr, base_amount);
        let new_lp = base_amount * 990 / 1000;
        
        // Stake new LP
        // farm::stake(state.farm_addr, new_lp);
        
        // Update accounting (new LP increases everyone's share value)
        state.total_lp_tokens = state.total_lp_tokens + new_lp;
        state.last_harvest = now;
        state.harvest_count = state.harvest_count + 1;
        state.lifetime_yield = state.lifetime_yield + (base_amount as u128);
        
        base_amount
    }
    
    fun total_assets_internal(state: &StrategyState): u64 {
        // LP tokens → underlying value
        // = lp_tokens * (reserve_base / total_lp) [from pool]
        // Simplified: 1:1 approximation
        state.total_lp_tokens
    }
    
    #[view]
    public fun total_assets(state: &StrategyState): u64 {
        total_assets_internal(state)
    }
    
    #[view]
    public fun get_apr_bps(state: &StrategyState): u64 {
        // Estimate from last harvest: annualize
        let now = timestamp::now_microseconds();
        let elapsed = now - state.last_harvest;
        
        if (elapsed == 0 || state.total_lp_tokens == 0) return 0;
        
        // Use lifetime_yield / total time to estimate rate
        // APR = (yield / elapsed) * secs_per_year
        let secs_per_year = 365u128 * 24 * 3600 * 1_000_000;
        let yield_rate = state.lifetime_yield * secs_per_year 
            / (state.total_lp_tokens as u128)
            / (elapsed as u128);
        
        (yield_rate * 10_000 / PRECISION) as u64
    }
    
    #[view]
    public fun price_per_share(state: &StrategyState): u128 {
        if (state.total_shares == 0) return PRECISION;
        (state.total_lp_tokens as u128) * PRECISION / (state.total_shares as u128)
    }
}
```

---

## Vault Contract

```move
// ============================================
// YIELD VAULT: ERC-4626 style vault for Move
// ============================================

module protocol::yield_vault {
    use aptos_framework::timestamp;
    
    const PRECISION: u128 = 1_000_000_000_000;  // 1e12
    const MANAGEMENT_FEE_BPS: u64 = 200;        // 2% management fee
    const PERFORMANCE_FEE_BPS: u64 = 1000;      // 10% performance fee
    
    struct Vault has key {
        // Token accounting
        total_assets: u64,
        total_shares: u64,
        
        // Fee accounting
        management_fee_bps: u64,
        performance_fee_bps: u64,
        fee_recipient: address,
        last_fee_timestamp: u64,
        
        // High watermark for performance fees
        high_watermark: u128,  // Price-per-share at last fee collection
        
        // Strategy allocation (basis points, must sum to 10000)
        strategy_addresses: vector<address>,
        strategy_allocations: vector<u64>,
        
        // Vault metadata
        name: vector<u8>,
        symbol: vector<u8>,
        paused: bool,
        
        // Stats
        total_deposited: u128,
        total_withdrawn: u128,
        deposit_count: u64,
    }
    
    /// Deposit base tokens, receive vault shares
    public entry fun deposit(
        user: &signer,
        vault_addr: address,
        amount: u64,
    ) {
        assert!(amount > 0, 1);
        let vault = borrow_global_mut<Vault>(vault_addr);
        assert!(!vault.paused, 2);
        
        // Collect management fees first (based on elapsed time)
        collect_management_fee(vault);
        
        // Calculate shares to mint
        let shares = if (vault.total_shares == 0) {
            (amount as u128) * PRECISION  // Bootstrap at 1:1
        } else {
            (amount as u128) * (vault.total_shares as u128) / (vault.total_assets as u128)
        };
        
        // Transfer user's tokens to vault (simplified)
        // coin::transfer<BaseToken>(user, vault_addr, amount);
        
        // Allocate to strategies
        deploy_to_strategies(vault, amount);
        
        vault.total_assets = vault.total_assets + amount;
        vault.total_shares = vault.total_shares + (shares / PRECISION) as u64;
        vault.total_deposited = vault.total_deposited + (amount as u128);
        vault.deposit_count = vault.deposit_count + 1;
        
        let _ = user;
    }
    
    /// Withdraw: burn shares, receive base tokens
    public entry fun withdraw(
        user: &signer,
        vault_addr: address,
        shares: u64,
    ) {
        let vault = borrow_global_mut<Vault>(vault_addr);
        assert!(shares > 0 && shares <= vault.total_shares, 3);
        
        // Collect fees first
        collect_management_fee(vault);
        
        // Calculate assets to return
        let assets = (shares as u128) * (vault.total_assets as u128) / (vault.total_shares as u128);
        
        // Withdraw from strategies
        withdraw_from_strategies(vault, assets as u64);
        
        vault.total_shares = vault.total_shares - shares;
        vault.total_assets = vault.total_assets - (assets as u64);
        vault.total_withdrawn = vault.total_withdrawn + assets;
        
        // Transfer assets back to user
        // coin::transfer<BaseToken>(vault_signer, user_addr, assets as u64);
        let _ = user;
    }
    
    /// Harvest all strategies and compound
    public entry fun harvest_all(keeper: &signer, vault_addr: address) {
        let vault = borrow_global_mut<Vault>(vault_addr);
        
        let new_assets = 0u64;
        let n = vector::length(&vault.strategy_addresses);
        let i = 0;
        
        while (i < n) {
            // let strategy_addr = *vector::borrow(&vault.strategy_addresses, i);
            // let gained = strategy::harvest(strategy_addr);
            let gained = 0u64;  // Placeholder
            new_assets = new_assets + gained;
            i = i + 1;
        };
        
        if (new_assets > 0) {
            // Collect performance fee on gains
            let perf_fee = (new_assets as u128) 
                * (vault.performance_fee_bps as u128) / 10_000;
            
            vault.total_assets = vault.total_assets + new_assets - (perf_fee as u64);
            
            // Mint fee shares for fee recipient
            let fee_shares = (perf_fee * (vault.total_shares as u128)) 
                / (vault.total_assets as u128);
            vault.total_shares = vault.total_shares + (fee_shares as u64);
        };
        
        let _ = keeper;
    }
    
    fun collect_management_fee(vault: &mut Vault) {
        let now = timestamp::now_microseconds();
        let elapsed = now - vault.last_fee_timestamp;
        
        if (elapsed == 0) return;
        
        // Management fee: annualized, charged pro-rata
        let secs_per_year = 365u128 * 24 * 3600 * 1_000_000;
        let fee_amount = (vault.total_assets as u128)
            * (vault.management_fee_bps as u128)
            * (elapsed as u128)
            / 10_000
            / secs_per_year;
        
        if (fee_amount > 0) {
            // Mint shares for fee (dilutes existing holders)
            let fee_shares = fee_amount * (vault.total_shares as u128) 
                / (vault.total_assets as u128);
            vault.total_shares = vault.total_shares + (fee_shares as u64);
        };
        
        vault.last_fee_timestamp = now;
    }
    
    fun deploy_to_strategies(vault: &mut Vault, amount: u64) {
        let n = vector::length(&vault.strategy_allocations);
        let i = 0;
        while (i < n) {
            let alloc_bps = *vector::borrow(&vault.strategy_allocations, i);
            let strategy_amount = (amount as u128) * (alloc_bps as u128) / 10_000;
            // let strategy_addr = *vector::borrow(&vault.strategy_addresses, i);
            // strategy::deposit(strategy_addr, strategy_amount as u64);
            let _ = strategy_amount;
            i = i + 1;
        };
    }
    
    fun withdraw_from_strategies(vault: &mut Vault, amount: u64) {
        // Withdraw proportionally from each strategy
        let n = vector::length(&vault.strategy_addresses);
        let remaining = amount;
        let i = 0;
        while (i < n && remaining > 0) {
            let alloc_bps = *vector::borrow(&vault.strategy_allocations, i);
            let from_strategy = (amount as u128) * (alloc_bps as u128) / 10_000;
            // let strategy_addr = *vector::borrow(&vault.strategy_addresses, i);
            // strategy::withdraw(strategy_addr, from_strategy as u64);
            let withdrew = std::u64::min(from_strategy as u64, remaining);
            remaining = remaining - withdrew;
            i = i + 1;
        };
    }
    
    #[view]
    public fun price_per_share(vault: &Vault): u128 {
        if (vault.total_shares == 0) return PRECISION;
        (vault.total_assets as u128) * PRECISION / (vault.total_shares as u128)
    }
    
    #[view]
    public fun preview_deposit(vault: &Vault, amount: u64): u64 {
        if (vault.total_shares == 0) return amount;
        ((amount as u128) * (vault.total_shares as u128) / (vault.total_assets as u128)) as u64
    }
    
    #[view]
    public fun preview_withdraw(vault: &Vault, shares: u64): u64 {
        ((shares as u128) * (vault.total_assets as u128) / (vault.total_shares as u128)) as u64
    }
}
```

---

## Auto-Compound Engine

```typescript
// ============================================
// AUTO-COMPOUND KEEPER BOT
// Monitors yield, compounds when profitable
// ============================================

import { AptosClient, AptosAccount } from 'aptos';

interface StrategyInfo {
  address: string;
  name: string;
  currentAprBps: number;
  pendingRewards: bigint;
  totalAssets: bigint;
  lastHarvest: number;
}

class AutoCompoundKeeper {
  private client: AptosClient;
  private keeper: AptosAccount;
  private vaultAddress: string;
  
  // Cost to harvest (in gas × gas_price)
  private HARVEST_GAS_COST = 5000n;  // octa
  private MIN_PROFIT_RATIO = 10;     // Must earn 10× gas cost to be worth it
  
  constructor(nodeUrl: string, keeperKey: string, vaultAddress: string) {
    this.client = new AptosClient(nodeUrl);
    this.keeper = new AptosAccount(Buffer.from(keeperKey, 'hex'));
    this.vaultAddress = vaultAddress;
  }
  
  async getStrategyInfo(strategyAddr: string): Promise<StrategyInfo> {
    const resource = await this.client.getAccountResource(
      strategyAddr,
      `${this.vaultAddress}::lp_farm_strategy::StrategyState`,
    );
    const data = resource.data as any;
    
    return {
      address: strategyAddr,
      name: 'LP Farm',
      currentAprBps: parseInt(data.last_apr_bps),
      pendingRewards: BigInt(data.pending_rewards || 0),
      totalAssets: BigInt(data.total_lp_tokens),
      lastHarvest: parseInt(data.last_harvest),
    };
  }
  
  isProfitableToHarvest(info: StrategyInfo): boolean {
    const estimatedRewardValue = info.pendingRewards;
    const minValueNeeded = this.HARVEST_GAS_COST * BigInt(this.MIN_PROFIT_RATIO);
    return estimatedRewardValue >= minValueNeeded;
  }
  
  async harvestStrategy(strategyAddr: string): Promise<boolean> {
    try {
      const payload = {
        function: `${this.vaultAddress}::yield_vault::harvest_all`,
        type_arguments: [],
        arguments: [this.vaultAddress],
      };
      
      const txn = await this.client.generateTransaction(
        this.keeper.address().toString(),
        payload,
        { max_gas_amount: '100000' },
      );
      
      const signed = await this.client.signTransaction(this.keeper, txn);
      const result = await this.client.submitTransaction(signed);
      await this.client.waitForTransaction(result.hash);
      
      console.log(`Harvested ${strategyAddr}: ${result.hash}`);
      return true;
    } catch (error) {
      console.error(`Harvest failed for ${strategyAddr}:`, error);
      return false;
    }
  }
  
  async runKeeperLoop(): Promise<void> {
    console.log('Auto-compound keeper started');
    
    while (true) {
      try {
        // Get all strategies from vault
        const strategies = ['0xSTRATEGY_A', '0xSTRATEGY_B'];
        
        for (const stratAddr of strategies) {
          const info = await this.getStrategyInfo(stratAddr);
          
          // Check time-based condition (harvest at most once per hour)
          const hoursSinceLastHarvest = (Date.now() / 1000 - info.lastHarvest / 1e6) / 3600;
          if (hoursSinceLastHarvest < 1) continue;
          
          // Check profitability
          if (!this.isProfitableToHarvest(info)) {
            console.log(`${info.name}: Not profitable yet (${info.pendingRewards} pending)`);
            continue;
          }
          
          console.log(`Harvesting ${info.name}: ${info.pendingRewards} reward tokens`);
          await this.harvestStrategy(stratAddr);
        }
      } catch (error) {
        console.error('Keeper loop error:', error);
      }
      
      // Wait 15 minutes before next check
      await new Promise(r => setTimeout(r, 15 * 60 * 1000));
    }
  }
}
```

---

## Risk-Adjusted Allocation

```typescript
// ============================================
// RISK-ADJUSTED STRATEGY ALLOCATION
// Maximize risk-adjusted yield (Sharpe-like)
// ============================================

interface StrategyData {
  address: string;
  aprBps: number;         // Current APR in bps
  tvlCap: bigint;         // Maximum TVL this strategy accepts
  currentTvl: bigint;     // Current TVL
  volatility: number;     // APR volatility (30d std dev)
  auditScore: number;     // 0-100 (100 = fully audited)
  ageMonths: number;      // How old is the strategy
}

class AllocationOptimizer {
  private riskFreeRateBps = 400;  // 4% risk-free rate (T-bills)
  
  // Risk-adjusted score: (APR - risk_free) / risk
  riskAdjustedScore(strategy: StrategyData): number {
    const excessReturn = strategy.aprBps - this.riskFreeRateBps;
    
    // Risk factors:
    const volatilityRisk = strategy.volatility;
    const auditRisk = (100 - strategy.auditScore) / 10;   // 0-10
    const ageRisk = Math.max(0, 12 - strategy.ageMonths) / 12;  // Higher if younger
    
    const totalRisk = Math.sqrt(
      volatilityRisk ** 2 + auditRisk ** 2 + ageRisk ** 2
    );
    
    if (totalRisk === 0) return 0;
    return excessReturn / totalRisk;
  }
  
  // Allocate based on risk-adjusted scores
  computeAllocations(
    strategies: StrategyData[],
    totalCapital: bigint,
  ): Map<string, bigint> {
    const scores = strategies.map(s => ({
      ...s,
      score: Math.max(0, this.riskAdjustedScore(s)),
    }));
    
    const totalScore = scores.reduce((sum, s) => sum + s.score, 0);
    const allocations = new Map<string, bigint>();
    
    if (totalScore === 0) {
      // Equal-weight fallback
      const perStrategy = totalCapital / BigInt(strategies.length);
      strategies.forEach(s => allocations.set(s.address, perStrategy));
      return allocations;
    }
    
    // Allocate proportional to risk-adjusted score
    let allocated = 0n;
    scores.forEach((s, i) => {
      const rawAllocation = (totalCapital * BigInt(Math.round(s.score * 1e6))) 
        / BigInt(Math.round(totalScore * 1e6));
      
      // Respect TVL cap
      const available = s.tvlCap - s.currentTvl;
      const alloc = rawAllocation > available ? available : rawAllocation;
      
      allocations.set(s.address, alloc);
      allocated += alloc;
    });
    
    // Put remainder in highest-scoring uncapped strategy
    const remainder = totalCapital - allocated;
    if (remainder > 0n && scores.length > 0) {
      const best = scores.sort((a, b) => b.score - a.score)[0];
      const current = allocations.get(best.address) || 0n;
      allocations.set(best.address, current + remainder);
    }
    
    return allocations;
  }
  
  // Convert allocations to basis points (for on-chain storage)
  toBasisPoints(
    allocations: Map<string, bigint>,
    totalCapital: bigint,
  ): Map<string, number> {
    const bps = new Map<string, number>();
    let totalBps = 0;
    
    allocations.forEach((amount, addr) => {
      const bp = Number((amount * 10000n) / totalCapital);
      bps.set(addr, bp);
      totalBps += bp;
    });
    
    // Ensure sums to exactly 10000 (adjust last strategy)
    if (totalBps !== 10000 && bps.size > 0) {
      const lastAddr = Array.from(bps.keys()).pop()!;
      bps.set(lastAddr, bps.get(lastAddr)! + (10000 - totalBps));
    }
    
    return bps;
  }
}

// Example usage
async function rebalanceVault() {
  const optimizer = new AllocationOptimizer();
  
  const strategies: StrategyData[] = [
    {
      address: '0xSTRAT_A',
      aprBps: 1200,      // 12% APR
      tvlCap: 1_000_000_000_000n,
      currentTvl: 500_000_000_000n,
      volatility: 200,   // 2% std dev
      auditScore: 95,
      ageMonths: 18,
    },
    {
      address: '0xSTRAT_B',
      aprBps: 2500,      // 25% APR (riskier)
      tvlCap: 500_000_000_000n,
      currentTvl: 100_000_000_000n,
      volatility: 800,   // 8% std dev
      auditScore: 70,
      ageMonths: 3,
    },
  ];
  
  const totalCapital = 100_000_000n;  // 100 USDC (6 decimals)
  const allocations = optimizer.computeAllocations(strategies, totalCapital);
  const bps = optimizer.toBasisPoints(allocations, totalCapital);
  
  console.log('Allocation Result:');
  bps.forEach((bp, addr) => {
    console.log(`  ${addr}: ${bp} bps (${(bp/100).toFixed(1)}%)`);
  });
}
```

---

## สรุป Yield Aggregator

```
YIELD AGGREGATOR KEY CONCEPTS

1. VAULT SHARE MATH
   shares_minted = deposit * total_shares / total_assets
   assets_returned = shares * total_assets / total_shares
   price_per_share grows monotonically as yield accrues

2. FEE STRUCTURE
   Management Fee: % of AUM per year (dilute shares over time)
   Performance Fee: % of profits above high watermark
   Both collected as newly minted vault shares

3. STRATEGY SELECTION
   APR > operating costs = profitable to harvest
   Risk-adjusted APR (Sharpe) > simple APR for selection
   TVL caps prevent concentration risk

4. COMPOUND FREQUENCY IMPACT
   APY = (1 + APR/n)^n - 1
   Daily:   n=365, 10% APR → 10.52% APY
   Hourly:  n=8760,          → 10.52% (diminishing returns)
   Insight: After daily compounding, more frequent = minimal gain

5. KEEPER ECONOMICS
   Harvest when: reward_value > gas_cost × min_ratio
   Gas estimation: simulate first, then decide
   Keeper incentive: 1-5% of harvest value
   Competition: MEV bots also harvest → need to be first

6. SECURITY
   Strategy isolation: each strategy in separate module
   TVL caps: limit blast radius of any single strategy failure
   Emergency withdraw: skip compound, just get funds out
   Pause mechanism: halt deposits during incidents
```

---

**ก่อนหน้า**: [Part 90 - Protocol Architecture ←](part-90-protocol-architecture.md)
**ต่อไป**: [Part 92 - NFT Marketplace Protocol →](part-92-nft-marketplace.md)
