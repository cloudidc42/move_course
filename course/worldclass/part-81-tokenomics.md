# Part 81: Advanced Tokenomics Design

## สารบัญ
- [Tokenomics Framework](#tokenomics-framework)
- [Emission Schedules](#emission-schedules)
- [veToken Model (Vote-Escrow)](#vetoken-model-vote-escrow)
- [Protocol Revenue Distribution](#protocol-revenue-distribution)
- [Tokenomics Simulation](#tokenomics-simulation)

---

## Tokenomics Framework

```
TOKENOMICS = Token + Economics
How tokens create value and incentivize behavior

CORE QUESTIONS
  1. Why does the token have value?
     - Governance rights
     - Fee sharing (cash flow)
     - Utility (needed to use protocol)
     - Scarcity + demand
     
  2. Who gets tokens and when?
     - Team: vest over 4 years
     - Investors: vest over 2-3 years
     - Community: distributed via incentives
     - Treasury: governed by DAO
     
  3. How is supply managed?
     - Inflation: new tokens for incentives
     - Deflation: burn fees/buybacks
     - Equilibrium: emissions vs burns
     
  4. What prevents inflation spirals?
     - Utility creates demand as supply grows
     - Lock mechanisms (veToken) reduce circulating supply
     - Buyback and burn with protocol revenue

COMMON MODELS

  Governance-only (no cash flow):
    Token → Voting rights
    Value: speculation on future value capture
    Risk: "governance token has no value" narrative
    
  Fee-sharing (cash flow):
    Token stakers receive % of protocol fees
    Value: dividend-like cash flow
    Risk: regulatory classification as security
    
  veToken (Vote-escrow):
    Lock token → get veToken → voting power + fee boost
    Value: locked supply reduces float, fee revenue
    Examples: Curve (veCRV), Velodrome (veVELO), Thena
    
  Burn:
    % of fees used to buy and burn token
    Value: deflationary pressure
    Examples: BNB quarterly burns

DISTRIBUTION BENCHMARKS (DeFi average)
  Team:         15-20%
  Investors:    15-25%
  Community:    30-50%
  Treasury:     20-30%
  Ecosystem:    5-15%
  
  Total: always 100%
```

---

## Emission Schedules

```move
// ============================================
// EMISSION SCHEDULE IMPLEMENTATIONS
// ============================================

module tokenomics::emissions {
    use aptos_framework::timestamp;
    
    // ============================================
    // LINEAR DECAY EMISSION
    // Common for DeFi: high early incentives, decaying to sustainable level
    // ============================================
    
    struct EmissionConfig has key {
        start_time: u64,
        initial_rate: u64,      // tokens per second (scaled by 1e8)
        min_rate: u64,          // Floor emission rate
        decay_period: u64,      // Seconds for full decay to min_rate
        last_update: u64,
        total_emitted: u64,
    }
    
    // Get current emission rate
    public fun get_current_rate(config: &EmissionConfig): u64 {
        let now = timestamp::now_seconds();
        let elapsed = now - config.start_time;
        
        if (elapsed >= config.decay_period) {
            return config.min_rate
        };
        
        // Linear decay: rate = initial - (initial - min) * elapsed / decay_period
        let decay_range = config.initial_rate - config.min_rate;
        let decay_amount = (decay_range as u128) * (elapsed as u128) / (config.decay_period as u128);
        
        config.initial_rate - (decay_amount as u64)
    }
    
    // Get total tokens emitted up to now
    public fun get_total_emitted(config: &EmissionConfig): u64 {
        let now = timestamp::now_seconds();
        let elapsed = now - config.start_time;
        
        if (elapsed >= config.decay_period) {
            // Full decay period + flat emission after
            let during_decay = calculate_area_trapezoid(
                config.initial_rate,
                config.min_rate,
                config.decay_period,
            );
            let after_decay = config.min_rate * (elapsed - config.decay_period);
            during_decay + after_decay
        } else {
            let current_rate = get_current_rate(config);
            calculate_area_trapezoid(config.initial_rate, current_rate, elapsed)
        }
    }
    
    // Area under linear decay curve (trapezoid)
    // = (a + b) / 2 * width
    fun calculate_area_trapezoid(a: u64, b: u64, width: u64): u64 {
        ((a as u128 + b as u128) / 2 * width as u128) as u64
    }
    
    // ============================================
    // HALVING SCHEDULE
    // Like Bitcoin: emission halves every epoch
    // ============================================
    
    struct HalvingConfig has key {
        start_time: u64,
        epoch_duration: u64,    // Duration of each epoch (seconds)
        initial_epoch_emission: u64,  // Total tokens in epoch 0
        min_epoch_emission: u64,      // Floor (never below this)
    }
    
    public fun get_epoch(config: &HalvingConfig): u64 {
        let elapsed = timestamp::now_seconds() - config.start_time;
        elapsed / config.epoch_duration
    }
    
    public fun get_epoch_emission(config: &HalvingConfig, epoch: u64): u64 {
        if (epoch >= 64) return config.min_epoch_emission;  // Guard against shift overflow
        
        let emission = config.initial_epoch_emission >> epoch;  // Halve each epoch
        if (emission < config.min_epoch_emission) {
            config.min_epoch_emission
        } else {
            emission
        }
    }
    
    public fun get_rate_per_second(config: &HalvingConfig): u64 {
        let epoch = get_epoch(config);
        let epoch_emission = get_epoch_emission(config, epoch);
        epoch_emission / config.epoch_duration
    }
    
    // ============================================
    // BONDING CURVE EMISSION
    // Emit more when TVL is high (incentivize growth)
    // ============================================
    
    struct DynamicEmission has key {
        base_rate: u64,         // Base emission per second
        tvl_multiplier: u64,    // bps: how much TVL affects rate
        max_multiplier: u64,    // Cap at 3x base rate
        current_tvl: u64,       // Updated by oracle
        target_tvl: u64,        // TVL at which base_rate applies
    }
    
    public fun get_dynamic_rate(config: &DynamicEmission): u64 {
        if (config.current_tvl == 0 || config.target_tvl == 0) {
            return config.base_rate
        };
        
        // Rate scales linearly with TVL ratio
        let tvl_ratio_bps = (config.current_tvl as u128) * 10_000 / (config.target_tvl as u128);
        
        // Multiplier = 1 + (tvl_ratio - 1) * tvl_multiplier / 10_000
        // Capped at max_multiplier
        let multiplier = if (tvl_ratio_bps > 10_000) {
            let excess = tvl_ratio_bps - 10_000;
            let add = excess * (config.tvl_multiplier as u128) / 10_000;
            let m = 10_000u128 + add;
            if (m > config.max_multiplier as u128) {
                config.max_multiplier as u128
            } else {
                m
            }
        } else {
            10_000u128  // At or below target: base rate
        };
        
        ((config.base_rate as u128) * multiplier / 10_000) as u64
    }
}
```

---

## veToken Model (Vote-Escrow)

```move
// ============================================
// veToken: Curve's famous model
// Lock TOKEN → receive veTOKEN
// Longer lock = more veToken = more voting power + fee share
// ============================================

module tokenomics::ve_token {
    use aptos_framework::timestamp;
    use aptos_framework::coin::{Self, Coin};
    
    // Protocol token
    struct PROTO has key {}
    
    // Lock position
    struct Lock has key {
        amount: u64,            // Tokens locked
        lock_end: u64,          // When lock expires (Unix seconds)
        owner: address,
    }
    
    struct VeConfig has key {
        max_lock_duration: u64,     // 4 years = 126_144_000 seconds
        min_lock_duration: u64,     // 1 week = 604_800 seconds
        total_locked: u64,
        total_ve_supply: u64,
    }
    
    const MAX_LOCK_SECS: u64 = 126_144_000;  // 4 years
    
    // Create a new lock position
    // Longer lock → more ve power
    public fun create_lock(
        user: &signer,
        tokens: Coin<PROTO>,
        lock_duration: u64,
        config_addr: address,
    ) acquires VeConfig {
        let config = borrow_global_mut<VeConfig>(config_addr);
        
        assert!(lock_duration >= config.min_lock_duration, 1);
        assert!(lock_duration <= config.max_lock_duration, 2);
        
        let amount = coin::value(&tokens);
        assert!(amount > 0, 3);
        
        // Deposit tokens
        let user_addr = std::signer::address_of(user);
        coin::deposit(config_addr, tokens);  // Store in protocol
        
        let lock_end = timestamp::now_seconds() + lock_duration;
        
        // ve power = amount * remaining_duration / max_duration
        let ve_power = calculate_ve_power(amount, lock_duration);
        
        config.total_locked = config.total_locked + amount;
        config.total_ve_supply = config.total_ve_supply + ve_power;
        
        move_to(user, Lock {
            amount,
            lock_end,
            owner: user_addr,
        });
        
        aptos_framework::event::emit(LockCreated {
            user: user_addr,
            amount,
            lock_end,
            ve_power,
        });
    }
    
    // Calculate current ve power (decreases linearly as lock expires)
    public fun get_ve_power(lock: &Lock): u64 {
        let now = timestamp::now_seconds();
        if (now >= lock.lock_end) {
            return 0  // Lock expired, no power
        };
        
        let remaining = lock.lock_end - now;
        calculate_ve_power(lock.amount, remaining)
    }
    
    fun calculate_ve_power(amount: u64, duration: u64): u64 {
        // Linear: 1 token locked 4 years = 1 veToken
        //         1 token locked 1 year = 0.25 veToken
        (amount as u128 * duration as u128 / MAX_LOCK_SECS as u128) as u64
    }
    
    // Extend lock duration
    public fun extend_lock(
        user: &signer,
        new_lock_end: u64,
        config_addr: address,
    ) acquires Lock, VeConfig {
        let user_addr = std::signer::address_of(user);
        let lock = borrow_global_mut<Lock>(user_addr);
        
        assert!(lock.owner == user_addr, 1);
        assert!(new_lock_end > lock.lock_end, 2);
        assert!(new_lock_end <= timestamp::now_seconds() + MAX_LOCK_SECS, 3);
        
        let old_ve = get_ve_power(lock);
        lock.lock_end = new_lock_end;
        let new_ve = get_ve_power(lock);
        
        let config = borrow_global_mut<VeConfig>(config_addr);
        config.total_ve_supply = config.total_ve_supply - old_ve + new_ve;
    }
    
    // Unlock expired lock
    public fun unlock(
        user: &signer,
        config_addr: address,
    ) acquires Lock, VeConfig {
        let user_addr = std::signer::address_of(user);
        let Lock { amount, lock_end, owner } = move_from<Lock>(user_addr);
        
        assert!(owner == user_addr, 1);
        assert!(timestamp::now_seconds() >= lock_end, 2);  // Must be expired
        
        let config = borrow_global_mut<VeConfig>(config_addr);
        config.total_locked = config.total_locked - amount;
        
        // Return tokens to user
        let tokens = coin::withdraw<PROTO>(config_addr, amount);
        coin::deposit(user_addr, tokens);
        
        aptos_framework::event::emit(LockReleased { user: user_addr, amount });
    }
    
    // ============================================
    // VOTING POWER: Used in governance
    // ============================================
    
    public fun get_voting_power(user: address): u64 acquires Lock {
        if (!exists<Lock>(user)) return 0;
        let lock = borrow_global<Lock>(user);
        get_ve_power(lock)
    }
    
    // Get vote fraction (user_ve / total_ve)
    public fun get_vote_fraction_bps(user: address, config_addr: address): u64 acquires Lock, VeConfig {
        let user_ve = get_voting_power(user);
        let config = borrow_global<VeConfig>(config_addr);
        
        if (config.total_ve_supply == 0) return 0;
        (user_ve as u128 * 10_000 / config.total_ve_supply as u128) as u64
    }
    
    // ============================================
    // FEE DISTRIBUTION: Proportional to ve power
    // ============================================
    
    struct FeeEpoch has key {
        epoch: u64,
        total_fees: u64,
        distributed: u64,
        start_time: u64,
        end_time: u64,
        user_checkpoints: aptos_std::table::Table<address, u64>,  // user → ve at snapshot
    }
    
    // Claim fee share for an epoch
    public fun claim_fees(
        user: &signer,
        epoch_addr: address,
        config_addr: address,
    ) acquires FeeEpoch, Lock, VeConfig {
        let user_addr = std::signer::address_of(user);
        let epoch = borrow_global_mut<FeeEpoch>(epoch_addr);
        let config = borrow_global<VeConfig>(config_addr);
        
        // Get user's ve power at epoch snapshot
        let user_ve = if (aptos_std::table::contains(&epoch.user_checkpoints, user_addr)) {
            *aptos_std::table::borrow(&epoch.user_checkpoints, user_addr)
        } else {
            get_voting_power(user_addr)  // Current power if not checkpointed
        };
        
        assert!(user_ve > 0, 1);
        
        // User's share = user_ve / total_ve * total_fees
        let user_fee = (epoch.total_fees as u128) * (user_ve as u128) / (config.total_ve_supply as u128);
        let user_fee = user_fee as u64;
        
        // Mark as distributed
        epoch.distributed = epoch.distributed + user_fee;
        
        // Transfer fee to user
        // coin::transfer<FeeToken>(config_addr, user_addr, user_fee);
    }
    
    #[event] struct LockCreated has drop, store { user: address, amount: u64, lock_end: u64, ve_power: u64 }
    #[event] struct LockReleased has drop, store { user: address, amount: u64 }
}
```

---

## Protocol Revenue Distribution

```move
// ============================================
// FEE DISTRIBUTION ENGINE
// Split protocol fees among multiple stakeholders
// ============================================

module tokenomics::fee_distributor {
    use aptos_framework::coin::{Self, Coin};
    
    struct FeeAllocation has store {
        recipient: address,
        share_bps: u64,     // Must sum to 10_000 (100%)
        description: std::string::String,
    }
    
    struct DistributorConfig has key {
        admin: address,
        allocations: vector<FeeAllocation>,
        total_distributed: u64,
    }
    
    // Standard DeFi fee split:
    // LP providers: 70%, stakers: 20%, treasury: 10%
    public fun initialize(
        admin: &signer,
        lp_pool_addr: address,
        staking_addr: address,
        treasury_addr: address,
    ) {
        let mut allocations = std::vector::empty<FeeAllocation>();
        
        std::vector::push_back(&mut allocations, FeeAllocation {
            recipient: lp_pool_addr,
            share_bps: 7_000,  // 70%
            description: std::string::utf8(b"LP Fee Rebate"),
        });
        
        std::vector::push_back(&mut allocations, FeeAllocation {
            recipient: staking_addr,
            share_bps: 2_000,  // 20%
            description: std::string::utf8(b"Staker Reward"),
        });
        
        std::vector::push_back(&mut allocations, FeeAllocation {
            recipient: treasury_addr,
            share_bps: 1_000,  // 10%
            description: std::string::utf8(b"Treasury"),
        });
        
        move_to(admin, DistributorConfig {
            admin: std::signer::address_of(admin),
            allocations,
            total_distributed: 0,
        });
    }
    
    // Distribute accumulated fees
    public fun distribute<CoinType>(
        fees: Coin<CoinType>,
        config_addr: address,
    ) acquires DistributorConfig {
        let config = borrow_global_mut<DistributorConfig>(config_addr);
        let total = coin::value(&fees);
        
        let n = std::vector::length(&config.allocations);
        let mut distributed_so_far = 0u64;
        
        let mut i = 0u64;
        while (i < n) {
            let allocation = std::vector::borrow(&config.allocations, i);
            
            let amount = if (i == n - 1) {
                // Last recipient gets remainder (handles rounding)
                total - distributed_so_far
            } else {
                (total as u128 * allocation.share_bps as u128 / 10_000) as u64
            };
            
            // Split and deposit
            // coin::deposit(allocation.recipient, coin::extract(&mut fees, amount));
            distributed_so_far = distributed_so_far + amount;
            
            i = i + 1;
        };
        
        config.total_distributed = config.total_distributed + total;
        
        // Cleanup: fees should be empty now
        coin::destroy_zero(fees);
    }
    
    // Update allocation (governance)
    public fun update_allocation(
        admin: &signer,
        new_allocations: vector<FeeAllocation>,
        config_addr: address,
    ) acquires DistributorConfig {
        let config = borrow_global_mut<DistributorConfig>(config_addr);
        assert!(std::signer::address_of(admin) == config.admin, 1);
        
        // Verify allocations sum to 10_000
        let total_bps = 0u64;
        let n = std::vector::length(&new_allocations);
        let mut i = 0u64;
        while (i < n) {
            let alloc = std::vector::borrow(&new_allocations, i);
            total_bps = total_bps + alloc.share_bps;
            i = i + 1;
        };
        assert!(total_bps == 10_000, 2);  // Must sum to 100%
        
        config.allocations = new_allocations;
    }
}
```

---

## Tokenomics Simulation

```typescript
// ============================================
// TOKENOMICS SIMULATION
// Model supply/demand dynamics over time
// ============================================

interface TokenomicsParams {
  totalSupply: bigint;
  initialCirculating: bigint;
  
  // Emissions
  emissionPerYear: bigint;
  halvingPeriodYears: number;
  emissionDurationYears: number;
  
  // Burns
  feeRevenueDailyUSD: bigint;
  tokenPriceUSD: bigint;  // scaled by 1e6
  burnPercentOfFees: number;  // 0-100
  
  // Lock mechanics
  lockIncentiveAPR: number;  // Expected APR from locking
  expectedLockRate: number;  // % of supply that will be locked
}

class TokenomicsModel {
  params: TokenomicsParams;
  
  constructor(params: TokenomicsParams) {
    this.params = params;
  }
  
  // Simulate supply over time
  simulateSupply(years: number): Array<{
    year: number;
    totalSupply: bigint;
    circulating: bigint;
    locked: bigint;
    emitted: bigint;
    burned: bigint;
    inflationRate: number;
  }> {
    const results = [];
    let totalSupply = this.params.totalSupply;
    let emitted = 0n;
    let burned = 0n;
    
    for (let year = 0; year <= years; year++) {
      // Emissions with halving
      const halvingsCompleted = Math.floor(year / this.params.halvingPeriodYears);
      const currentYearlyEmission = this.params.emissionPerYear >> BigInt(halvingsCompleted);
      const yearEmission = year < this.params.emissionDurationYears ? currentYearlyEmission : 0n;
      
      // Burns: daily fees * 365 / token price * burn%
      const yearlyFeesUSD = this.params.feeRevenueDailyUSD * 365n;
      const yearBurn = yearlyFeesUSD * BigInt(this.params.burnPercentOfFees) / 
        (this.params.tokenPriceUSD * 100n / 1_000_000n);
      
      totalSupply = totalSupply + yearEmission - yearBurn;
      emitted += yearEmission;
      burned += yearBurn;
      
      const locked = totalSupply * BigInt(Math.floor(this.params.expectedLockRate * 100)) / 100n;
      const circulating = totalSupply - locked;
      
      const inflationRate = year > 0 && results.length > 0 
        ? Number(totalSupply - results[results.length - 1].totalSupply) / Number(results[results.length - 1].totalSupply) * 100
        : 0;
      
      results.push({
        year,
        totalSupply,
        circulating,
        locked,
        emitted,
        burned,
        inflationRate,
      });
    }
    
    return results;
  }
  
  // Calculate FDV and market cap at different supply stages
  calculateValuation(
    targetRevenueMul: number,  // P/E equivalent for crypto
    revenueShare: number,       // % of fees going to token holders
  ): {
    annualRevenue: bigint;
    revenueToHolders: bigint;
    fairValuePerToken: bigint;
    fdv: bigint;
  } {
    const annualRevenue = this.params.feeRevenueDailyUSD * 365n;
    const revenueToHolders = annualRevenue * BigInt(Math.floor(revenueShare * 100)) / 100n;
    
    // Fair value = (revenue to holders * P/E multiple) / total supply
    const fairFdv = revenueToHolders * BigInt(Math.floor(targetRevenueMul));
    const fairValuePerToken = fairFdv * 1_000_000n / this.params.totalSupply;
    
    return {
      annualRevenue,
      revenueToHolders,
      fairValuePerToken,  // in USD * 1e6
      fdv: fairFdv,
    };
  }
  
  // Print formatted report
  printReport(): void {
    console.log('\n====== TOKENOMICS REPORT ======\n');
    
    const simulation = this.simulateSupply(10);
    
    console.log('Year | Total Supply | Circulating | Inflation | Emitted | Burned');
    console.log('-----|-------------|-------------|-----------|---------|-------');
    
    for (const row of simulation) {
      const sup = (Number(row.totalSupply) / 1e6).toFixed(1) + 'M';
      const circ = (Number(row.circulating) / 1e6).toFixed(1) + 'M';
      const inf = row.inflationRate.toFixed(1) + '%';
      const em = (Number(row.emitted) / 1e6).toFixed(1) + 'M';
      const bu = (Number(row.burned) / 1e6).toFixed(1) + 'M';
      
      console.log(`${row.year.toString().padStart(4)} | ${sup.padStart(11)} | ${circ.padStart(11)} | ${inf.padStart(9)} | ${em.padStart(7)} | ${bu}`);
    }
    
    const valuation = this.calculateValuation(20, 0.20);
    console.log('\n====== VALUATION MODEL ======');
    console.log(`Annual Protocol Revenue: $${(Number(valuation.annualRevenue) / 1e6).toFixed(1)}M`);
    console.log(`Revenue to Token Holders (20%): $${(Number(valuation.revenueToHolders) / 1e6).toFixed(1)}M`);
    console.log(`Fair Value/Token (20x P/E): $${(Number(valuation.fairValuePerToken) / 1e6).toFixed(4)}`);
    console.log(`Fair FDV: $${(Number(valuation.fdv) / 1e6).toFixed(1)}M`);
  }
}

// Example: Run tokenomics model for a DEX
const model = new TokenomicsModel({
  totalSupply: 1_000_000_000n * 1_000_000n,  // 1B tokens (6 decimals)
  initialCirculating: 100_000_000n * 1_000_000n,  // 10% initial
  
  emissionPerYear: 100_000_000n * 1_000_000n,  // 100M/year
  halvingPeriodYears: 2,
  emissionDurationYears: 8,
  
  feeRevenueDailyUSD: 100_000_000_000n,  // $100K/day in micro USD
  tokenPriceUSD: 1_000_000n,  // $1.00
  burnPercentOfFees: 20,
  
  lockIncentiveAPR: 15,
  expectedLockRate: 0.40,  // 40% of supply locked
});

model.printReport();
```

---

## สรุป Advanced Tokenomics

```
TOKENOMICS DESIGN PRINCIPLES

1. TOKEN MUST HAVE UTILITY
   Bad: "governance token" with nothing to govern
   Good: Required for protocol access, fee discounts, boosted rewards

2. ALIGN INCENTIVES LONG-TERM
   Bad: High early APY → dump pressure
   Good: veToken locks → rewards only for committed holders

3. SUSTAINABLE EMISSIONS
   Bad: Infinite inflation to attract liquidity
   Good: Emissions < protocol revenue = deflationary net

4. FAIR DISTRIBUTION
   Bad: 50% to VCs/team, 50% to community
   Good: Community-first distribution, gradual team vesting

5. TRANSPARENT SUPPLY SCHEDULE
   Bad: Discretionary minting (admin can print tokens)
   Good: On-chain emission schedule, immutable after launch

COMMON TOKENOMICS MISTAKES
   ✗ Ponzinomics: APR funded by new users, not protocol revenue
   ✗ VC dump: Large unlocks with no lockup period
   ✗ Inflation spiral: Emissions > value created
   ✗ Governance theater: Token has no real governance power
   ✗ Complexity theater: veToken model without real fee revenue
   
veToken Checklist:
   ✅ Real protocol revenue to distribute
   ✅ Lock period meaningful (1-4 years)
   ✅ Lock cannot be "unlocked early" (no workarounds)
   ✅ Voting weight decreases linearly over time
   ✅ Gauge voting for liquidity direction
   ✅ Bribe market for protocols competing for emissions
```

---

**ก่อนหน้า**: [Part 80 - Protocol Operations ←](part-80-protocol-ops.md)
**ต่อไป**: [Part 82 - MEV & Order Flow Optimization →](part-82-mev-orderflow.md)
