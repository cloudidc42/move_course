# Part 43: Token Economics Engineering

## สารบัญ
- [Tokenomics Design Patterns](#tokenomics-design-patterns)
- [Emission Schedules](#emission-schedules)
- [Bonding Curves](#bonding-curves)
- [Vote-Escrow & Gauge System](#vote-escrow--gauge-system)
- [Protocol Revenue Distribution](#protocol-revenue-distribution)
- [ตัวอย่าง: Full Tokenomics System](#ตัวอย่าง-full-tokenomics-system)

---

## Tokenomics Design Patterns

```
Token Value Sources:
  1. Utility (pay fees, access features)
  2. Governance (vote on protocol decisions)
  3. Revenue share (receive protocol income)
  4. Collateral (borrow against)
  5. Staking (earn yield)
  
Emission Models:
  1. Fixed supply (Bitcoin-style): hard cap, decreasing inflation
  2. Inflationary (Ethereum-style): annual % emission
  3. Deflationary: buy-and-burn mechanism
  4. Elastic supply: algorithmic supply adjustment
  
Sink Mechanisms (reduce supply):
  - Fee burning
  - Lockups (reduce circulating supply)
  - Slashing (penalties)
  - Buybacks
```

---

## Emission Schedules

```move
module tokenomics::emission {
    use std::signer;
    use aptos_framework::coin::{Self, MintCapability};
    use aptos_framework::timestamp;
    
    // ============================================
    // Halving emission schedule (Bitcoin-style)
    // ============================================
    
    struct EmissionConfig has key {
        // Emission
        initial_rate: u64,       // tokens per second at start
        current_rate: u64,       // current emission rate
        halving_interval: u64,   // seconds between halvings
        next_halving: u64,       // timestamp of next halving
        halving_count: u64,      // number of halvings done
        max_halvings: u64,       // max halvings (e.g., 10)
        
        // Tracking
        total_emitted: u64,
        max_supply: u64,
        
        // Distribution
        last_emission_time: u64,
    }
    
    struct MyToken has key {}
    
    struct TokenCaps has key {
        mint_cap: MintCapability<MyToken>,
    }
    
    const PRECISION: u64 = 1_000_000_000;  // 1e9 for rate calculations
    
    public entry fun initialize(
        admin: &signer,
        initial_rate: u64,    // tokens per second (e.g., 100 = 100 tokens/sec)
        halving_interval: u64, // e.g., 126144000 = 4 years
        max_halvings: u64,
        max_supply: u64,
    ) {
        let now = timestamp::now_seconds();
        move_to(admin, EmissionConfig {
            initial_rate,
            current_rate: initial_rate,
            halving_interval,
            next_halving: now + halving_interval,
            halving_count: 0,
            max_halvings,
            total_emitted: 0,
            max_supply,
            last_emission_time: now,
        });
    }
    
    // ============================================
    // Mint due emissions
    // ============================================
    
    public entry fun mint_emissions(
        config_addr: address,
        recipient: address,
    ) acquires EmissionConfig, TokenCaps {
        let config = borrow_global_mut<EmissionConfig>(config_addr);
        let now = timestamp::now_seconds();
        
        // Process halvings
        while (
            now >= config.next_halving && 
            config.halving_count < config.max_halvings
        ) {
            config.current_rate = config.current_rate / 2;
            config.halving_count = config.halving_count + 1;
            config.next_halving = config.next_halving + config.halving_interval;
        };
        
        // Calculate emissions since last mint
        let elapsed = now - config.last_emission_time;
        let to_mint = config.current_rate * elapsed;
        
        // Check supply cap
        let actual_mint = if (config.total_emitted + to_mint > config.max_supply) {
            config.max_supply - config.total_emitted
        } else {
            to_mint
        };
        
        if (actual_mint == 0) return;
        
        config.total_emitted = config.total_emitted + actual_mint;
        config.last_emission_time = now;
        
        // Mint tokens
        let caps = borrow_global<TokenCaps>(config_addr);
        let minted = coin::mint(actual_mint, &caps.mint_cap);
        coin::deposit(recipient, minted);
    }
    
    // ============================================
    // Vested emission (unlock linearly)
    // ============================================
    
    struct VestedAllocation has key {
        recipient: address,
        total_amount: u64,
        claimed: u64,
        start_time: u64,
        duration: u64,
        cliff: u64,  // seconds before any vesting
    }
    
    public fun claimable_amount(alloc: &VestedAllocation): u64 {
        let now = timestamp::now_seconds();
        
        if (now < alloc.start_time + alloc.cliff) return 0;
        
        let vested_time = if (now > alloc.start_time + alloc.duration) {
            alloc.duration
        } else {
            now - alloc.start_time
        };
        
        let vested = alloc.total_amount * vested_time / alloc.duration;
        if (vested > alloc.claimed) { vested - alloc.claimed } else { 0 }
    }
    
    public entry fun claim_vested(
        recipient: &signer,
        alloc_addr: address,
        caps_addr: address,
    ) acquires VestedAllocation, TokenCaps {
        let alloc = borrow_global_mut<VestedAllocation>(alloc_addr);
        assert!(signer::address_of(recipient) == alloc.recipient, 1);
        
        let amount = claimable_amount(alloc);
        assert!(amount > 0, 2);
        
        alloc.claimed = alloc.claimed + amount;
        
        let caps = borrow_global<TokenCaps>(caps_addr);
        let minted = coin::mint(amount, &caps.mint_cap);
        coin::deposit(alloc.recipient, minted);
    }
    
    // ============================================
    // Scheduled distribution (team, investors, etc.)
    // ============================================
    
    struct DistributionSchedule has key {
        allocations: aptos_std::smart_table::SmartTable<address, VestedAllocation>,
        total_allocated: u64,
    }
}
```

---

## Bonding Curves

```move
module tokenomics::bonding_curve {
    use aptos_framework::coin::{Self, Coin, MintCapability, BurnCapability};
    
    // ============================================
    // Bonding Curve: price determined by supply
    // price = f(supply)
    // ============================================
    
    struct BondingCurve has key {
        // Linear curve: price = base_price + supply * slope
        base_price: u64,    // price at 0 supply (scaled by PRECISION)
        slope: u64,         // price increase per token (scaled by PRECISION)
        
        // Current state
        total_supply: u64,
        reserve: u64,       // collateral backing
        
        // Precision
        precision: u64,     // 1e18
    }
    
    const PRECISION: u64 = 1_000_000_000_000_000_000;  // 1e18
    
    // ============================================
    // Buy tokens (price increases as supply grows)
    // ============================================
    
    // Price function: P(s) = base + s * slope / PRECISION
    public fun buy_price(curve: &BondingCurve, amount: u64): u64 {
        // Integral from supply to supply+amount of P(s) ds
        // = base*amount + slope*(s^2 - s1^2)/2 / PRECISION
        let s1 = curve.total_supply;
        let s2 = s1 + amount;
        
        let base_cost = curve.base_price * amount / PRECISION;
        let slope_cost = curve.slope * (s2 * s2 - s1 * s1) / (2 * PRECISION * PRECISION);
        
        base_cost + slope_cost
    }
    
    // Sell price (refund from reserve)
    public fun sell_price(curve: &BondingCurve, amount: u64): u64 {
        assert!(amount <= curve.total_supply, 1);
        let s1 = curve.total_supply - amount;
        let s2 = curve.total_supply;
        
        let base_refund = curve.base_price * amount / PRECISION;
        let slope_refund = curve.slope * (s2 * s2 - s1 * s1) / (2 * PRECISION * PRECISION);
        
        // Apply spread (e.g., 2% buy-sell spread)
        (base_refund + slope_refund) * 98 / 100
    }
    
    public entry fun buy<Reserve, Token>(
        buyer: &signer,
        curve_addr: address,
        reserve_coin: Coin<Reserve>,
        min_tokens: u64,
    ) acquires BondingCurve {
        // ...
    }
    
    // ============================================
    // Logarithmic bonding curve (Bancor-style)
    // price = price_0 * (supply / reserve_ratio)^(1/reserve_ratio - 1)
    // ============================================
    
    // Simplified: approximate with piecewise linear
    public fun bancor_price(
        reserve: u64,
        supply: u64,
        reserve_ratio: u64,  // bps (e.g., 5000 = 50%)
    ): u64 {
        // P = reserve / (supply * reserve_ratio / 10000)
        if (supply == 0) return reserve;
        reserve * 10_000 / (supply * reserve_ratio / 10_000)
    }
    
    // ============================================
    // Sigmoid bonding curve (for fairer distribution)
    // price stays low for a while, then shoots up
    // ============================================
    
    // Approximation: piece-wise linear sigmoid
    pub fun sigmoid_price(supply: u64, max_supply: u64): u64 {
        let ratio = supply * 100 / max_supply;  // 0-100
        
        if (ratio < 20) {
            // Flat bottom (0-20%)
            1_000_000  // $0.001
        } else if (ratio < 80) {
            // Linear middle (20-80%)
            1_000_000 + (ratio - 20) * 15_000_000 / 60  // $0.001 to $0.151
        } else {
            // Steep top (80-100%)
            151_000_000 + (ratio - 80) * 4_000_000_000 / 20  // $0.151 to $4.151
        }
    }
}
```

---

## Vote-Escrow & Gauge System

```move
module tokenomics::ve_gauge {
    use std::signer;
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::timestamp;
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // veToken (vote-escrow) + Gauge weight system
    // Inspired by Curve Finance
    // ============================================
    
    struct VESystem has key {
        admin: address,
        ve_token_supply: u64,
        epoch_duration: u64,  // 1 week
        max_lock_time: u64,   // 4 years
        
        // Gauge weights
        gauges: SmartTable<address, GaugeInfo>,
        total_weight: u64,
        
        // Emissions directed to gauges
        weekly_emission: u64,
    }
    
    struct GaugeInfo has copy, drop, store {
        pool_addr: address,
        weight: u64,         // votes allocated to this gauge
        relative_weight: u64, // weight / total * 10000
        emission_rate: u64,   // tokens/sec based on weight
    }
    
    struct VEPosition has key {
        owner: address,
        locked_amount: u64,
        lock_end: u64,      // timestamp when lock expires
        voting_power: u64,  // cached voting power (for gas efficiency)
    }
    
    struct GaugeVotes has key {
        voter: address,
        votes: SmartTable<address, u64>,  // gauge -> vote weight (bps)
        total_votes: u64,
        last_vote_time: u64,
        vote_cooldown: u64,
    }
    
    const MAX_LOCK: u64 = 126144000;  // 4 years in seconds
    const WEEK: u64 = 604800;
    
    // ============================================
    // Lock tokens to get veTokens
    // ============================================
    
    public entry fun lock<Token>(
        user: &signer,
        system_addr: address,
        tokens: Coin<Token>,
        lock_duration: u64,
    ) acquires VESystem {
        let user_addr = signer::address_of(user);
        let amount = coin::value(&tokens);
        let now = timestamp::now_seconds();
        
        // Round lock end to nearest week
        let lock_end = (now + lock_duration) / WEEK * WEEK;
        
        assert!(lock_end > now, 1);
        assert!(lock_duration <= MAX_LOCK, 2);
        
        // Voting power = amount * remaining_time / max_time
        let voting_power = amount * (lock_end - now) / MAX_LOCK;
        
        // Destroy or store the coins (veToken is non-transferable)
        // In production: lock in vault, burn original, mint veToken
        coin::destroy_zero(coin::extract(&mut tokens, 0));  // placeholder
        coin::deposit(system_addr, tokens);  // lock in system
        
        let system = borrow_global_mut<VESystem>(system_addr);
        system.ve_token_supply = system.ve_token_supply + voting_power;
        
        move_to(user, VEPosition {
            owner: user_addr,
            locked_amount: amount,
            lock_end,
            voting_power,
        });
    }
    
    // ============================================
    // Vote for gauge weights
    // ============================================
    
    public entry fun vote(
        user: &signer,
        system_addr: address,
        gauge_weights: vector<address>,
        weights_bps: vector<u64>,  // must sum to 10000
    ) acquires VESystem, VEPosition, GaugeVotes {
        let user_addr = signer::address_of(user);
        let position = borrow_global<VEPosition>(user_addr);
        
        // Check lock still active
        assert!(timestamp::now_seconds() < position.lock_end, 3);
        
        // Check vote cooldown (can't change votes too often)
        if (exists<GaugeVotes>(user_addr)) {
            let gv = borrow_global<GaugeVotes>(user_addr);
            assert!(
                timestamp::now_seconds() >= gv.last_vote_time + gv.vote_cooldown,
                4
            );
        };
        
        // Validate weights sum to 10000
        let n = std::vector::length(&gauge_weights);
        assert!(n == std::vector::length(&weights_bps), 5);
        
        let mut total = 0u64;
        let mut i = 0u64;
        while (i < n) {
            total = total + *std::vector::borrow(&weights_bps, i);
            i = i + 1;
        };
        assert!(total == 10_000, 6);
        
        // Apply votes to gauges
        let system = borrow_global_mut<VESystem>(system_addr);
        let voting_power = position.voting_power;
        
        i = 0;
        while (i < n) {
            let gauge = *std::vector::borrow(&gauge_weights, i);
            let weight_bps = *std::vector::borrow(&weights_bps, i);
            let vote_amount = voting_power * weight_bps / 10_000;
            
            if (smart_table::contains(&system.gauges, gauge)) {
                let gauge_info = smart_table::borrow_mut(&mut system.gauges, gauge);
                gauge_info.weight = gauge_info.weight + vote_amount;
                system.total_weight = system.total_weight + vote_amount;
            };
            i = i + 1;
        };
    }
    
    // ============================================
    // Calculate gauge emissions for epoch
    // ============================================
    
    public entry fun checkpoint_gauge_weights(
        system_addr: address,
    ) acquires VESystem {
        let system = borrow_global_mut<VESystem>(system_addr);
        
        if (system.total_weight == 0) return;
        
        // Update relative weights and emission rates
        // In production: iterate over gauges with SmartTable::for_each_ref
        // Here: simplified single gauge update
        let weekly = system.weekly_emission;
        let total = system.total_weight;
        
        // For each gauge: emission = (weight / total) * weekly
        // emission_rate_per_sec = emission / WEEK
    }
    
    // ============================================
    // Unlock expired position
    // ============================================
    
    public entry fun unlock<Token>(
        user: &signer,
        system_addr: address,
    ) acquires VESystem, VEPosition {
        let user_addr = signer::address_of(user);
        let position = borrow_global<VEPosition>(user_addr);
        
        assert!(timestamp::now_seconds() >= position.lock_end, 7);
        
        let VEPosition { owner, locked_amount, lock_end: _, voting_power } = 
            move_from<VEPosition>(user_addr);
        
        let system = borrow_global_mut<VESystem>(system_addr);
        system.ve_token_supply = system.ve_token_supply - voting_power;
        
        // Return locked tokens
        // In production: withdraw from vault
    }
    
    #[view]
    public fun voting_power_at(user: address, at_time: u64): u64 acquires VEPosition {
        if (!exists<VEPosition>(user)) return 0;
        let pos = borrow_global<VEPosition>(user);
        if (at_time >= pos.lock_end) return 0;
        pos.locked_amount * (pos.lock_end - at_time) / MAX_LOCK
    }
}
```

---

## Protocol Revenue Distribution

```move
module tokenomics::revenue {
    use std::signer;
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::timestamp;
    
    // ============================================
    // Protocol Revenue → Stakeholder Distribution
    // ============================================
    
    struct RevenueDistributor has key {
        admin: address,
        total_collected: u64,
        
        // Distribution ratios (must sum to 10000)
        treasury_bps: u64,      // e.g., 3000 = 30%
        stakers_bps: u64,       // e.g., 4000 = 40%
        veToken_holders_bps: u64, // e.g., 2000 = 20%
        buyback_bps: u64,       // e.g., 1000 = 10%
        
        // Accumulators
        staker_acc_per_share: u128,
        epoch: u64,
        
        // Addresses
        treasury: address,
        staking_pool: address,
        ve_system: address,
        market_addr: address,   // for buyback
    }
    
    struct StakerInfo has key {
        staked: u64,
        reward_debt: u128,
        unclaimed_rewards: u64,
    }
    
    const PRECISION: u128 = 1_000_000_000_000;  // 1e12
    
    // ============================================
    // Collect and distribute protocol fees
    // ============================================
    
    public entry fun distribute_fees<Fee>(
        distributor_addr: address,
        fees: Coin<Fee>,
    ) acquires RevenueDistributor {
        let total = coin::value(&fees);
        let dist = borrow_global_mut<RevenueDistributor>(distributor_addr);
        
        // Calculate shares
        let treasury_share = total * dist.treasury_bps / 10_000;
        let staker_share = total * dist.stakers_bps / 10_000;
        let ve_share = total * dist.veToken_holders_bps / 10_000;
        let buyback_share = total * dist.buyback_bps / 10_000;
        
        dist.total_collected = dist.total_collected + total;
        
        // Update staker accumulator
        // (simplified: assume total_staked is tracked separately)
        let total_staked = 1_000_000u64;  // placeholder
        if (total_staked > 0 && staker_share > 0) {
            dist.staker_acc_per_share = dist.staker_acc_per_share + 
                (staker_share as u128) * PRECISION / (total_staked as u128);
        };
        
        // In production: extract and transfer each share
        // coin::split + coin::deposit to each destination
        
        // Destroy fees (simplified - in reality distribute them)
        // coin::destroy_for_testing(fees); // test only
    }
    
    // ============================================
    // Staker claim rewards
    // ============================================
    
    public entry fun claim_staker_rewards(
        user: &signer,
        distributor_addr: address,
        staker_addr: address,
    ) acquires RevenueDistributor, StakerInfo {
        let dist = borrow_global<RevenueDistributor>(distributor_addr);
        let staker = borrow_global_mut<StakerInfo>(staker_addr);
        assert!(signer::address_of(user) == staker_addr, 1);
        
        let pending = (staker.staked as u128) 
            * dist.staker_acc_per_share 
            / PRECISION;
        let claimable = (pending - staker.reward_debt) as u64 + staker.unclaimed_rewards;
        
        assert!(claimable > 0, 2);
        staker.reward_debt = pending;
        staker.unclaimed_rewards = 0;
        
        // Transfer claimable to user
    }
    
    // ============================================
    // Buyback and burn mechanism
    // ============================================
    
    public entry fun execute_buyback<Fee, GovernanceToken>(
        keeper: &signer,
        distributor_addr: address,
        buyback_amount: u64,
    ) acquires RevenueDistributor {
        // 1. Use protocol fees to buy GovernanceToken from market
        // let gov_tokens = market::buy<Fee, GovernanceToken>(buyback_amount);
        
        // 2. Burn the bought tokens
        // coin::burn(gov_tokens, &burn_cap);
        
        // This reduces total supply → increases scarcity → higher price
    }
    
    // ============================================
    // Revenue sharing for veToken holders
    // Similar to Curve's 3CRV distribution
    // ============================================
    
    struct VERevenueEpoch has key {
        epoch: u64,
        revenue: u64,
        total_ve_supply: u64,
        start_time: u64,
        end_time: u64,
        claims: aptos_std::smart_table::SmartTable<address, bool>,
    }
    
    public entry fun claim_ve_revenue(
        user: &signer,
        epoch_addr: address,
        ve_position_addr: address,
    ) acquires VERevenueEpoch {
        use aptos_std::smart_table;
        let user_addr = signer::address_of(user);
        let epoch = borrow_global_mut<VERevenueEpoch>(epoch_addr);
        
        assert!(!smart_table::contains(&epoch.claims, user_addr), 1);
        
        // Get user's ve balance at epoch start
        let user_ve = tokenomics::ve_gauge::voting_power_at(user_addr, epoch.start_time);
        assert!(user_ve > 0, 2);
        
        let claimable = epoch.revenue * user_ve / epoch.total_ve_supply;
        smart_table::add(&mut epoch.claims, user_addr, true);
        
        // Transfer claimable revenue to user
    }
    
    #[view]
    public fun claimable_revenue(
        user: address,
        epoch_addr: address,
    ): u64 acquires VERevenueEpoch {
        use aptos_std::smart_table;
        let epoch = borrow_global<VERevenueEpoch>(epoch_addr);
        if (smart_table::contains(&epoch.claims, user)) return 0;
        
        let user_ve = tokenomics::ve_gauge::voting_power_at(user, epoch.start_time);
        if (user_ve == 0 || epoch.total_ve_supply == 0) return 0;
        
        epoch.revenue * user_ve / epoch.total_ve_supply
    }
}
```

---

## ตัวอย่าง: Full Tokenomics System

```move
module tokenomics::full_system {
    // ============================================
    // Complete tokenomics flow
    // ============================================
    
    // Token Allocation (1 billion total supply):
    // - Community (40%): emitted over 4 years with halving
    // - Team (20%): 1 year cliff + 3 year vesting
    // - Investors (15%): 6 month cliff + 2 year vesting
    // - Treasury (15%): controlled by governance
    // - Liquidity (10%): immediate, for initial DEX offering
    
    // Revenue Flow:
    // Protocol Fees → RevenueDistributor
    //   40% → Token Stakers (proportional to stake)
    //   30% → veToken Holders (proportional to lock)
    //   20% → Treasury (DAO controlled)
    //   10% → Buyback & Burn
    
    // Incentive Flow:
    // Emissions → Gauge System
    //   Gauges direct emissions to pools
    //   veToken holders vote on gauge weights
    //   Most popular pools get most rewards
    
    // Supply Dynamics:
    // Inflation: ~5% year 1, halves every 4 years
    // Deflation: Buyback burns + fee burns
    // Net: slightly inflationary early, deflationary long term
    
    struct TokenomicsParams {
        total_supply: u64,
        initial_price: u64,
        initial_market_cap: u64,
        initial_fdv: u64,
        unlock_schedule: vector<UnlockEvent>,
    }
    
    struct UnlockEvent has copy, drop, store {
        timestamp: u64,
        amount: u64,
        category: vector<u8>,
        recipient: address,
    }
    
    // Simulate 5-year token economics
    public fun simulate_5y(
        initial_price: u64,
        annual_fee_revenue: u64,
        staking_rate: u64,  // % of supply staked
    ): (u64, u64, u64) {
        // Returns: (price_5y, market_cap_5y, holder_yield_5y)
        
        // Year 1: 5% inflation, 10% fee yield on staked
        // Year 2: after halving, 2.5% inflation
        // Year 3-5: diminishing inflation
        
        // Simplified model
        let price_5y = initial_price * 5;  // 5x price assumption
        let mcap_5y = price_5y * 1_000_000_000;  // 1B tokens
        let yield_5y = annual_fee_revenue * 5 * staking_rate / 100;
        
        (price_5y, mcap_5y, yield_5y)
    }
}
```

---

## สรุป Tokenomics Engineering

```
Component           | Key Metrics              | Implementation
--------------------|--------------------------|------------------
Emission            | Inflation rate, halvings | EmissionConfig
Vesting             | Cliff, duration          | VestedAllocation
Bonding Curve       | Price discovery          | Mathematical curve
veToken             | Lock time, voting power  | VEPosition
Gauges              | Emission direction       | GaugeInfo + weights
Revenue             | Distribution ratios      | RevenueDistributor
Buyback             | Deflation mechanism      | Market purchase + burn

Sustainability Check:
  ✓ Emission < Protocol Revenue (inflation < yield)
  ✓ Sufficient liquidity for token trading
  ✓ Governance decentralized over time
  ✓ Aligned incentives for all stakeholders
  ✓ Emergency mechanisms for crisis
```

---

**ก่อนหน้า**: [Part 42 - Composability ←](part-42-composability.md)
**ต่อไป**: [Part 44 - ZK Proofs Integration →](part-44-zkproofs.md)
