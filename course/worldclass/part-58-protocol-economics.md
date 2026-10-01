# Part 58: DeFi Protocol Economics

## สารบัญ
- [Protocol Economics Framework](#protocol-economics-framework)
- [Fee Architecture](#fee-architecture)
- [Revenue Sharing Models](#revenue-sharing-models)
- [Tokenomics Integration](#tokenomics-integration)
- [Treasury Management](#treasury-management)
- [Incentive Alignment](#incentive-alignment)

---

## Protocol Economics Framework

```
DeFi Protocol Economics:

Revenue Sources:
  1. Trading fees (primary): 0.01% - 1% per swap
  2. Borrowing interest spread: borrow rate - deposit rate
  3. Liquidation bonus: 5-15% of liquidated position
  4. Withdrawal fees: 0.01% for large withdrawals
  5. Protocol-owned liquidity income
  6. Cross-chain bridge fees

Cost Structure:
  - Protocol team salaries
  - Security audits: $100k - $500k
  - Bug bounty: 10% of TVL at risk
  - Marketing & BD
  - Liquidity mining emissions
  - Infrastructure

Sustainability Check:
  Revenue > Emissions + Operating costs = Sustainable
  Revenue < Emissions = Ponzi (unsustainable)

Key Metrics:
  TVL (Total Value Locked): liquidity depth
  Volume: actual usage
  Revenue: fee income
  P/S ratio: market cap / annual revenue (like P/E for DeFi)
  Protocol-owned liquidity (POL): % of TVL owned by protocol

Healthy DeFi Protocol:
  P/S ratio < 20x (vs stock market ~25x PE)
  Revenue growing
  POL > 30% (resilient to LP withdrawal)
  Diversified revenue streams
```

---

## Fee Architecture

```move
module economics::fee_system {
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::coin;
    
    // ============================================
    // Multi-tiered fee system
    // ============================================
    
    struct FeeConfig has key {
        // Dynamic fees by volume tier
        // Tier 0: < 1M daily volume = 0.30%
        // Tier 1: 1M-10M = 0.25%
        // Tier 2: 10M-100M = 0.20%
        // Tier 3: > 100M = 0.15%
        volume_tiers: vector<VolumeTier>,
        
        // Fee distribution
        lp_share_bps: u64,        // e.g., 6700 = 67% to LPs
        protocol_share_bps: u64,  // e.g., 1650 = 16.5% to protocol
        insurance_share_bps: u64, // e.g., 500 = 5% to insurance
        staker_share_bps: u64,    // e.g., 1150 = 11.5% to xTOKEN stakers
        
        // Accumulated fees per token
        protocol_fees: SmartTable<std::string::String, u64>,
        insurance_fees: SmartTable<std::string::String, u64>,
        staker_fees: SmartTable<std::string::String, u64>,
        
        // 24h volume tracking (for tier calculation)
        daily_volume: u64,
        day_start: u64,
        
        // Admin
        governance: address,
    }
    
    struct VolumeTier has store, copy, drop {
        min_volume: u64,
        fee_bps: u64,
    }
    
    // Get current fee tier based on 24h volume
    public fun get_current_fee_bps(config: &FeeConfig): u64 {
        let now = aptos_framework::timestamp::now_seconds();
        
        // Reset if new day
        let daily_volume = if (now >= config.day_start + 86400) {
            0u64  // New day, start fresh
        } else {
            config.daily_volume
        };
        
        // Find applicable tier
        let len = std::vector::length(&config.volume_tiers);
        let mut applicable_fee = 30u64;  // Default 0.30%
        
        let mut i = 0u64;
        while (i < len) {
            let tier = std::vector::borrow(&config.volume_tiers, i);
            if (daily_volume >= tier.min_volume) {
                applicable_fee = tier.fee_bps;
            };
            i = i + 1;
        };
        
        applicable_fee
    }
    
    // Collect and distribute fees
    public fun collect_and_distribute<Token>(
        config: &mut FeeConfig,
        fee_amount: u64,
        pool_lp_supply: u64,
    ) {
        let lp_amount = fee_amount * config.lp_share_bps / 10_000;
        let protocol_amount = fee_amount * config.protocol_share_bps / 10_000;
        let insurance_amount = fee_amount * config.insurance_share_bps / 10_000;
        let staker_amount = fee_amount - lp_amount - protocol_amount - insurance_amount;
        
        // LP fees stay in pool (auto-compounding via k increase)
        
        // Accumulate protocol/insurance/staker fees
        let token_key = get_token_key<Token>();
        
        upsert_fees(&mut config.protocol_fees, token_key, protocol_amount);
        upsert_fees(&mut config.insurance_fees, token_key, insurance_amount);
        upsert_fees(&mut config.staker_fees, token_key, staker_amount);
        
        // Update volume tracking
        let now = aptos_framework::timestamp::now_seconds();
        if (now >= config.day_start + 86400) {
            config.daily_volume = fee_amount;
            config.day_start = now;
        } else {
            config.daily_volume = config.daily_volume + fee_amount;
        };
    }
    
    // ============================================
    // Dynamic fee adjustment via governance
    // ============================================
    
    public entry fun update_fee_tiers(
        governance: &signer,
        config_addr: address,
        new_tiers: vector<VolumeTier>,
    ) acquires FeeConfig {
        let config = borrow_global_mut<FeeConfig>(config_addr);
        assert!(std::signer::address_of(governance) == config.governance, 1);
        config.volume_tiers = new_tiers;
    }
    
    public entry fun update_fee_distribution(
        governance: &signer,
        config_addr: address,
        lp_bps: u64,
        protocol_bps: u64,
        insurance_bps: u64,
        staker_bps: u64,
    ) acquires FeeConfig {
        let config = borrow_global_mut<FeeConfig>(config_addr);
        assert!(std::signer::address_of(governance) == config.governance, 1);
        assert!(lp_bps + protocol_bps + insurance_bps + staker_bps == 10_000, 2);
        
        config.lp_share_bps = lp_bps;
        config.protocol_share_bps = protocol_bps;
        config.insurance_share_bps = insurance_bps;
        config.staker_share_bps = staker_bps;
    }
    
    fun upsert_fees(table: &mut SmartTable<std::string::String, u64>, key: std::string::String, amount: u64) {
        if (smart_table::contains(table, key)) {
            let current = smart_table::borrow_mut(table, key);
            *current = *current + amount;
        } else {
            smart_table::add(table, key, amount);
        }
    }
    
    fun get_token_key<Token>(): std::string::String {
        std::string::utf8(b"token")  // Simplified
    }
}
```

---

## Revenue Sharing Models

```move
module economics::revenue_sharing {
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::coin;
    use aptos_framework::aptos_coin::AptosCoin;
    
    // ============================================
    // xTOKEN Revenue Sharing (Staking for yield)
    // ============================================
    
    // Users stake TOKEN → receive xTOKEN
    // Protocol revenue distributed to xTOKEN holders
    // xTOKEN appreciates vs TOKEN over time
    
    struct StakingPool has key {
        // xTOKEN: TOKEN exchange rate
        // rate = total_token_in_pool / total_xtoken_supply
        total_token_staked: u64,
        total_xtoken_supply: u64,
        
        // Accumulated rewards per xToken (for efficient distribution)
        // Using reward-per-share approach
        reward_per_xtoken_cumulative: u128,  // Q64 fixed point
        
        // Individual staker tracking
        stakers: SmartTable<address, StakerInfo>,
        
        // Revenue streams (which tokens are distributed)
        // All protocol revenue converted to TOKEN or distributed in kind
        pending_rewards: SmartTable<std::string::String, u64>,  // token → amount
    }
    
    struct StakerInfo has store, copy, drop {
        xtoken_balance: u64,
        reward_per_xtoken_at_entry: u128,
        accrued_rewards: u64,
    }
    
    // Exchange rate: TOKEN per xTOKEN
    public fun get_exchange_rate(pool: &StakingPool): u64 {
        if (pool.total_xtoken_supply == 0) return 1_000_000;  // 1:1 initial
        
        // rate = total_staked * 1e6 / total_xtoken
        (pool.total_token_staked as u128) * 1_000_000 / (pool.total_xtoken_supply as u128)
            as u64
    }
    
    // Stake TOKEN → receive xTOKEN
    public entry fun stake(
        user: &signer,
        pool_addr: address,
        token_amount: u64,
    ) acquires StakingPool {
        let pool = borrow_global_mut<StakingPool>(pool_addr);
        let user_addr = std::signer::address_of(user);
        
        // Calculate xTOKEN to mint
        let xtoken_amount = if (pool.total_xtoken_supply == 0) {
            token_amount  // First stake: 1:1
        } else {
            (token_amount as u128) * (pool.total_xtoken_supply as u128)
                / (pool.total_token_staked as u128) as u64
        };
        
        // Pull tokens
        let payment = coin::withdraw<AptosCoin>(user, token_amount);
        coin::deposit(pool_addr, payment);
        
        pool.total_token_staked = pool.total_token_staked + token_amount;
        pool.total_xtoken_supply = pool.total_xtoken_supply + xtoken_amount;
        
        // Update staker info
        let staker = if (smart_table::contains(&pool.stakers, user_addr)) {
            smart_table::borrow_mut(&mut pool.stakers, user_addr)
        } else {
            smart_table::add(&mut pool.stakers, user_addr, StakerInfo {
                xtoken_balance: 0,
                reward_per_xtoken_at_entry: pool.reward_per_xtoken_cumulative,
                accrued_rewards: 0,
            });
            smart_table::borrow_mut(&mut pool.stakers, user_addr)
        };
        
        // Claim pending rewards before adding more xTokens
        let pending = calculate_pending(
            staker.xtoken_balance,
            pool.reward_per_xtoken_cumulative,
            staker.reward_per_xtoken_at_entry,
        );
        staker.accrued_rewards = staker.accrued_rewards + pending;
        staker.reward_per_xtoken_at_entry = pool.reward_per_xtoken_cumulative;
        staker.xtoken_balance = staker.xtoken_balance + xtoken_amount;
    }
    
    // Unstake xTOKEN → receive TOKEN (with accumulated yield)
    public entry fun unstake(
        user: &signer,
        pool_addr: address,
        xtoken_amount: u64,
    ) acquires StakingPool {
        let pool = borrow_global_mut<StakingPool>(pool_addr);
        let user_addr = std::signer::address_of(user);
        
        let staker = smart_table::borrow_mut(&mut pool.stakers, user_addr);
        assert!(staker.xtoken_balance >= xtoken_amount, 1);
        
        // Claim all pending rewards
        let pending = calculate_pending(
            staker.xtoken_balance,
            pool.reward_per_xtoken_cumulative,
            staker.reward_per_xtoken_at_entry,
        );
        let total_rewards = staker.accrued_rewards + pending;
        staker.accrued_rewards = 0;
        staker.reward_per_xtoken_at_entry = pool.reward_per_xtoken_cumulative;
        staker.xtoken_balance = staker.xtoken_balance - xtoken_amount;
        
        // Calculate TOKEN to return (uses exchange rate)
        let token_amount = (xtoken_amount as u128) * (pool.total_token_staked as u128)
            / (pool.total_xtoken_supply as u128) as u64;
        
        pool.total_token_staked = pool.total_token_staked - token_amount;
        pool.total_xtoken_supply = pool.total_xtoken_supply - xtoken_amount;
        
        // Transfer TOKEN back to user
        // coin::transfer<TOKEN>(pool_signer, user_addr, token_amount + total_rewards);
    }
    
    // Distribute revenue to stakers (called by protocol)
    public entry fun distribute_revenue(
        protocol: &signer,
        pool_addr: address,
        revenue_amount: u64,
    ) acquires StakingPool {
        let pool = borrow_global_mut<StakingPool>(pool_addr);
        
        if (pool.total_xtoken_supply == 0) return;
        
        // Add to total staked (auto-compounding)
        pool.total_token_staked = pool.total_token_staked + revenue_amount;
        
        // Update reward per xToken
        let reward_increment = (revenue_amount as u128) * (1u128 << 64)
            / (pool.total_xtoken_supply as u128);
        pool.reward_per_xtoken_cumulative = pool.reward_per_xtoken_cumulative + reward_increment;
    }
    
    fun calculate_pending(xtoken: u64, cumulative: u128, at_entry: u128): u64 {
        let diff = cumulative - at_entry;
        ((xtoken as u128) * diff / (1u128 << 64)) as u64
    }
}
```

---

## Tokenomics Integration

```move
module economics::tokenomics {
    use aptos_framework::coin;
    
    // ============================================
    // Integrated Tokenomics: Emission + Burn + Lock
    // ============================================
    
    struct TokenEconomy has key {
        // Total supply tracking
        total_supply: u64,
        circulating_supply: u64,
        
        // Emission schedule (halving every 2 years)
        epoch: u64,               // Current epoch (1 epoch = 1 week)
        epoch_start: u64,         // When current epoch started
        epoch_duration: u64,      // Duration in seconds
        base_emission: u64,       // Tokens per epoch (epoch 0)
        halving_period: u64,      // Epochs between halvings (= 2 years / 1 week = ~104)
        
        // Burned supply
        total_burned: u64,
        
        // Locked supply (not circulating)
        locked_supply: u64,
        
        // Allocation breakdown
        treasury_allocation_bps: u64,
        team_allocation_bps: u64,
        liquidity_mining_bps: u64,
        ecosystem_bps: u64,
    }
    
    // Get current epoch emission (with halvings)
    public fun get_epoch_emission(economy: &TokenEconomy): u64 {
        let halvings = economy.epoch / economy.halving_period;
        
        // Emission halves every halving_period epochs
        // After N halvings: emission = base / 2^N
        let mut emission = economy.base_emission;
        let mut i = 0u64;
        while (i < halvings && emission > 0) {
            emission = emission / 2;
            i = i + 1;
        };
        emission
    }
    
    // Deflationary mechanics: burn tokens
    // Called when:
    //   - Users pay protocol fees (burn % of fees)
    //   - NFTs are destroyed
    //   - Governance decides buyback
    public fun burn_tokens(
        economy: &mut TokenEconomy,
        amount: u64,
        burn_cap: &coin::BurnCapability<ProtocolToken>,
    ) {
        economy.total_burned = economy.total_burned + amount;
        economy.total_supply = economy.total_supply - amount;
        economy.circulating_supply = economy.circulating_supply - amount;
        
        // Actual burn
        // coin::burn(coins, burn_cap);
    }
    
    // Lock tokens (vest, reduce circulating supply)
    public fun lock_tokens(economy: &mut TokenEconomy, amount: u64) {
        economy.locked_supply = economy.locked_supply + amount;
        economy.circulating_supply = economy.circulating_supply - amount;
    }
    
    // Unlock tokens (vesting complete)
    public fun unlock_tokens(economy: &mut TokenEconomy, amount: u64) {
        economy.locked_supply = economy.locked_supply - amount;
        economy.circulating_supply = economy.circulating_supply + amount;
    }
    
    // Effective supply = circulating - staked - locked
    public fun get_effective_supply(economy: &TokenEconomy): u64 {
        economy.circulating_supply  // staked counted separately
    }
    
    struct ProtocolToken has store {}
}
```

---

## Treasury Management

```move
module economics::treasury {
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::coin;
    
    // ============================================
    // Protocol Treasury with Multi-sig Governance
    // ============================================
    
    struct Treasury has key {
        // Balances of each token
        balances: SmartTable<std::string::String, u64>,
        
        // Pending proposals
        proposals: SmartTable<u64, TreasuryProposal>,
        next_proposal_id: u64,
        
        // Multi-sig: N of M
        signers: vector<address>,
        threshold: u64,  // Minimum signatures needed
        
        // Time-lock on large transfers
        timelock_threshold: u64,  // Above this amount needs timelock
        timelock_duration: u64,   // 48 hours
    }
    
    struct TreasuryProposal has store {
        proposer: address,
        description: std::string::String,
        
        // Transfer details
        token_type: std::string::String,
        amount: u64,
        recipient: address,
        
        // Approval tracking
        approvals: vector<address>,
        executed: bool,
        created_at: u64,
        executable_at: u64,  // Timelock expires
        expires_at: u64,
    }
    
    // Create treasury transfer proposal
    public entry fun propose_transfer(
        proposer: &signer,
        treasury_addr: address,
        token_type: std::string::String,
        amount: u64,
        recipient: address,
        description: std::string::String,
    ) acquires Treasury {
        let treasury = borrow_global_mut<Treasury>(treasury_addr);
        let proposer_addr = std::signer::address_of(proposer);
        
        assert!(is_signer(treasury, proposer_addr), 1);
        
        let now = aptos_framework::timestamp::now_seconds();
        
        // Large amounts require timelock
        let executable_at = if (amount >= treasury.timelock_threshold) {
            now + treasury.timelock_duration
        } else {
            now  // Execute immediately after threshold approvals
        };
        
        let proposal_id = treasury.next_proposal_id;
        treasury.next_proposal_id = proposal_id + 1;
        
        smart_table::add(&mut treasury.proposals, proposal_id, TreasuryProposal {
            proposer: proposer_addr,
            description,
            token_type,
            amount,
            recipient,
            approvals: vector[proposer_addr],  // Proposer auto-approves
            executed: false,
            created_at: now,
            executable_at,
            expires_at: now + 7 * 24 * 3600,  // 7-day expiry
        });
    }
    
    // Approve a proposal
    public entry fun approve_proposal(
        signer_account: &signer,
        treasury_addr: address,
        proposal_id: u64,
    ) acquires Treasury {
        let treasury = borrow_global_mut<Treasury>(treasury_addr);
        let approver = std::signer::address_of(signer_account);
        
        assert!(is_signer(treasury, approver), 1);
        
        let proposal = smart_table::borrow_mut(&mut treasury.proposals, proposal_id);
        assert!(!proposal.executed, 2);
        assert!(!has_approved(proposal, approver), 3);
        
        std::vector::push_back(&mut proposal.approvals, approver);
    }
    
    // Execute approved proposal (after timelock)
    public entry fun execute_proposal(
        executor: &signer,
        treasury_addr: address,
        proposal_id: u64,
    ) acquires Treasury {
        let treasury = borrow_global_mut<Treasury>(treasury_addr);
        let now = aptos_framework::timestamp::now_seconds();
        
        let proposal = smart_table::borrow_mut(&mut treasury.proposals, proposal_id);
        
        assert!(!proposal.executed, 1);
        assert!(now >= proposal.executable_at, 2);
        assert!(now < proposal.expires_at, 3);
        assert!(
            std::vector::length(&proposal.approvals) >= treasury.threshold,
            4
        );
        
        proposal.executed = true;
        
        let amount = proposal.amount;
        let recipient = proposal.recipient;
        
        // Transfer from treasury to recipient
        // coin::transfer<Token>(treasury_signer, recipient, amount);
    }
    
    // Emergency: cancel proposal (requires threshold)
    public entry fun cancel_proposal(
        canceller: &signer,
        treasury_addr: address,
        proposal_id: u64,
    ) acquires Treasury {
        let treasury = borrow_global_mut<Treasury>(treasury_addr);
        let canceller_addr = std::signer::address_of(canceller);
        
        assert!(is_signer(treasury, canceller_addr), 1);
        
        smart_table::remove(&mut treasury.proposals, proposal_id);
    }
    
    fun is_signer(treasury: &Treasury, addr: address): bool {
        std::vector::contains(&treasury.signers, &addr)
    }
    
    fun has_approved(proposal: &TreasuryProposal, addr: address): bool {
        std::vector::contains(&proposal.approvals, &addr)
    }
}
```

---

## Incentive Alignment

```move
module economics::incentives {
    
    // ============================================
    // Incentive Design: Aligning all stakeholders
    // ============================================
    
    // Stakeholders and their incentives:
    // 1. Protocol team: long-term success (equity/tokens + vesting)
    // 2. LPs: yield from fees (protect capital)
    // 3. Traders: deep liquidity, low fees
    // 4. Token holders: value appreciation
    // 5. Protocol: sustainable growth
    
    struct VestingSchedule has key {
        beneficiary: address,
        total_amount: u64,
        claimed: u64,
        
        start_time: u64,
        cliff_duration: u64,    // No tokens before cliff
        total_duration: u64,    // Full vest by this time
        
        revokable: bool,         // Can governance revoke?
        revoked: bool,
    }
    
    // Linear vesting with cliff
    public fun get_vested_amount(
        schedule: &VestingSchedule,
    ): u64 {
        let now = aptos_framework::timestamp::now_seconds();
        
        if (schedule.revoked) return schedule.claimed;
        
        if (now < schedule.start_time + schedule.cliff_duration) {
            // Before cliff: nothing vested
            0
        } else if (now >= schedule.start_time + schedule.total_duration) {
            // Past full vesting: everything
            schedule.total_amount
        } else {
            // Linear vest after cliff
            let elapsed = now - schedule.start_time;
            schedule.total_amount * elapsed / schedule.total_duration
        }
    }
    
    public entry fun claim_vested(
        beneficiary: &signer,
        schedule_addr: address,
    ) acquires VestingSchedule {
        let schedule = borrow_global_mut<VestingSchedule>(schedule_addr);
        let claimant = std::signer::address_of(beneficiary);
        
        assert!(schedule.beneficiary == claimant, 1);
        
        let vested = get_vested_amount(schedule);
        let claimable = vested - schedule.claimed;
        assert!(claimable > 0, 2);
        
        schedule.claimed = schedule.claimed + claimable;
        
        // Transfer tokens to beneficiary
        // coin::transfer<ProtocolToken>(treasury, claimant, claimable);
    }
    
    // ============================================
    // Liquidity Mining: Reward LPs over time
    // ============================================
    
    struct LiquidityMiningConfig has key {
        // Emission per second (decreasing over time)
        emission_per_second: u64,
        
        // When mining ends
        end_time: u64,
        
        // Per-pool allocation (pool_addr → share_bps)
        pool_allocations: aptos_std::smart_table::SmartTable<address, u64>,
        
        // Total allocation bps (must sum to 10000)
        total_allocated_bps: u64,
        
        // Epoch tracking
        last_update_time: u64,
        accumulated_per_lp: u128,  // Global accumulator
    }
    
    // Adjust mining rewards via governance (with decay)
    public fun calculate_emission_at_time(
        config: &LiquidityMiningConfig,
        time: u64,
    ): u64 {
        if (time >= config.end_time) return 0;
        
        // Exponential decay: halve every 180 days
        let elapsed = time - config.last_update_time;
        let half_life = 180 * 24 * 3600;
        
        // emission(t) = base_emission * 0.5^(t/half_life)
        // Integer approximation: divide by 2 each half_life period
        let halvings = elapsed / half_life;
        let mut current_emission = config.emission_per_second;
        let mut i = 0u64;
        while (i < halvings && current_emission > 0) {
            current_emission = current_emission / 2;
            i = i + 1;
        };
        current_emission
    }
}
```

---

## สรุป Protocol Economics

```
Protocol Economics Principles:

1. SUSTAINABLE REVENUE
   Revenue > Emissions + Costs = Healthy
   Revenue < Emissions = Unsustainable (avoid!)
   
2. FEE STRUCTURE
   LPs: 50-70% (must attract liquidity)
   Stakers: 10-20% (reward governance participation)
   Protocol: 10-20% (fund development)
   Insurance: 5% (cover hacks)

3. VALUE ACCRUAL
   Protocol revenue → token value:
   a) Buy & burn (deflationary)
   b) Revenue share to stakers (yield)
   c) Protocol-owned liquidity
   
4. EMISSION SCHEDULE
   - Decreasing over time (prevent inflation)
   - Halving events (like Bitcoin)
   - End date is ideal (no permanent inflation)
   
5. TEAM INCENTIVES
   - 4-year vesting, 1-year cliff
   - Tokens worth more if protocol succeeds
   - Cannot dump early = aligned with users

6. TREASURY MANAGEMENT
   - Multi-sig (3/5 minimum)
   - Timelock for large transfers
   - Diversified (not 100% own token)
   - Target: 12+ months runway

Successful Protocol Examples:
  Uniswap: Fee switch (0.05% to governance)
  Aave: Safety Module (staked AAVE as insurance)
  Curve: veToken model (lock → boost + revenue)
  dYdX: Trading fee rebates + governance rewards
```

---

**ก่อนหน้า**: [Part 57 - Advanced Testing ←](part-57-advanced-testing.md)
**ต่อไป**: [Part 59 - Move on Sui Deep Dive →](part-59-sui-deep-dive.md)
