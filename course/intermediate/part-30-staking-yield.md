# Part 30: Staking and Yield Farming

## สารบัญ
- [Staking Patterns](#staking-patterns)
- [Reward Distribution](#reward-distribution)
- [Yield Farming](#yield-farming)
- [Vote-Escrow (veToken)](#vote-escrow-vetoken)
- [ตัวอย่าง: Complete Staking Protocol](#ตัวอย่าง-complete-staking-protocol)

---

## Staking Patterns

```
Simple Staking:
  User deposits token → gets staking receipt
  Protocol pays rewards over time
  User withdraws → gets back tokens + rewards

Liquid Staking:
  User deposits APT → gets stAPT (liquid staking token)
  stAPT accumulates value as validators earn rewards
  User can use stAPT in DeFi while staking

Governance Staking:
  User locks token → gets voting power
  Longer lock = more voting power
  Reward = protocol fees + emissions
```

---

## Reward Distribution

```move
module staking::reward_math {
    // ============================================
    // Per-share accumulator pattern
    // (like Sushiswap MasterChef)
    // ============================================
    
    // Global: acc_reward_per_share increases over time
    // User: pending_reward = shares * acc - reward_debt
    // 
    // When user deposits:
    //   reward_debt = shares * acc_reward_per_share
    //
    // When user withdraws:
    //   pending = shares * acc - reward_debt
    
    const PRECISION: u128 = 1_000_000_000_000;  // 1e12
    
    struct Pool has copy, drop, store {
        total_staked: u64,
        acc_reward_per_share: u128,  // accumulated reward per share (scaled by PRECISION)
        last_reward_time: u64,
        reward_per_second: u64,      // rewards emitted per second
    }
    
    struct UserInfo has copy, drop, store {
        amount: u64,           // staked amount
        reward_debt: u128,     // reward already "paid" to user
        pending_rewards: u64,  // unclaimed rewards
    }
    
    // Update pool accumulator
    public fun update_pool(pool: &mut Pool, current_time: u64) {
        if (current_time <= pool.last_reward_time) return;
        if (pool.total_staked == 0) {
            pool.last_reward_time = current_time;
            return
        };
        
        let elapsed = current_time - pool.last_reward_time;
        let reward = (pool.reward_per_second as u128) * (elapsed as u128);
        
        // Distribute reward among all stakers proportionally
        pool.acc_reward_per_share = pool.acc_reward_per_share
            + reward * PRECISION / (pool.total_staked as u128);
        pool.last_reward_time = current_time;
    }
    
    // Calculate pending rewards for user
    public fun pending_reward(pool: &Pool, user: &UserInfo): u64 {
        let acc = pool.acc_reward_per_share;
        let pending = (user.amount as u128) * acc / PRECISION;
        
        if (pending < user.reward_debt) return user.pending_rewards;
        
        user.pending_rewards + (pending - user.reward_debt) as u64
    }
    
    // Deposit (harvest existing rewards first)
    public fun deposit(
        pool: &mut Pool,
        user: &mut UserInfo,
        amount: u64,
        current_time: u64,
    ): u64 {  // returns pending rewards to send to user
        update_pool(pool, current_time);
        
        // Harvest existing rewards
        let pending = pending_reward(pool, user);
        
        // Update user
        user.amount = user.amount + amount;
        user.reward_debt = (user.amount as u128) * pool.acc_reward_per_share / PRECISION;
        user.pending_rewards = 0;
        
        // Update pool
        pool.total_staked = pool.total_staked + amount;
        
        pending
    }
    
    // Withdraw (harvest + unstake)
    public fun withdraw(
        pool: &mut Pool,
        user: &mut UserInfo,
        amount: u64,
        current_time: u64,
    ): (u64, u64) {  // returns (withdrawn_amount, pending_rewards)
        assert!(user.amount >= amount, 1);
        update_pool(pool, current_time);
        
        let pending = pending_reward(pool, user);
        
        user.amount = user.amount - amount;
        user.reward_debt = (user.amount as u128) * pool.acc_reward_per_share / PRECISION;
        user.pending_rewards = 0;
        
        pool.total_staked = pool.total_staked - amount;
        
        (amount, pending)
    }
    
    // Harvest only (no withdraw)
    public fun harvest(
        pool: &mut Pool,
        user: &mut UserInfo,
        current_time: u64,
    ): u64 {
        update_pool(pool, current_time);
        
        let pending = pending_reward(pool, user);
        user.reward_debt = (user.amount as u128) * pool.acc_reward_per_share / PRECISION;
        user.pending_rewards = 0;
        
        pending
    }
}
```

---

## Yield Farming

```move
module farming::masterchef {
    use std::signer;
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::timestamp;
    use aptos_framework::event;
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // Types (similar to Sushiswap MasterChef)
    // ============================================
    
    struct FARM {}  // Farm token type
    
    struct PoolInfo has copy, drop, store {
        lp_token_addr: address,  // which LP token to stake
        alloc_point: u64,        // weight for rewards
        acc_farm_per_share: u128,
        last_reward_time: u64,
        total_staked: u64,
    }
    
    struct UserInfo has copy, drop, store {
        amount: u64,
        reward_debt: u128,
    }
    
    struct FarmingConfig has key {
        admin: address,
        farm_per_second: u64,
        total_alloc_point: u64,
        start_time: u64,
        end_time: u64,
        pools: vector<PoolInfo>,
        // pool_id -> user_addr -> UserInfo
    }
    
    struct UserPositions has key {
        positions: SmartTable<u64, UserInfo>,  // pool_id -> UserInfo
    }
    
    // ============================================
    // Events
    // ============================================
    
    #[event]
    struct Deposited has drop, store {
        user: address,
        pool_id: u64,
        amount: u64,
    }
    
    #[event]
    struct Withdrawn has drop, store {
        user: address,
        pool_id: u64,
        amount: u64,
        reward: u64,
    }
    
    #[event]
    struct Harvested has drop, store {
        user: address,
        pool_id: u64,
        reward: u64,
    }
    
    // ============================================
    // Errors
    // ============================================
    
    const E_INVALID_POOL: u64 = 1;
    const E_INSUFFICIENT_STAKE: u64 = 2;
    const E_NOT_ADMIN: u64 = 3;
    
    // ============================================
    // Initialize
    // ============================================
    
    public entry fun initialize(
        admin: &signer,
        farm_per_second: u64,
        start_time: u64,
        duration_seconds: u64,
    ) {
        let admin_addr = signer::address_of(admin);
        let now = timestamp::now_seconds();
        let actual_start = if (start_time < now) { now } else { start_time };
        
        move_to(admin, FarmingConfig {
            admin: admin_addr,
            farm_per_second,
            total_alloc_point: 0,
            start_time: actual_start,
            end_time: actual_start + duration_seconds,
            pools: std::vector::empty(),
        });
    }
    
    // ============================================
    // Add Pool
    // ============================================
    
    public entry fun add_pool(
        admin: &signer,
        config_addr: address,
        lp_token_addr: address,
        alloc_point: u64,
    ) acquires FarmingConfig {
        let config = borrow_global_mut<FarmingConfig>(config_addr);
        assert!(signer::address_of(admin) == config.admin, E_NOT_ADMIN);
        
        // Mass update all pools first
        update_all_pools(config);
        
        config.total_alloc_point = config.total_alloc_point + alloc_point;
        
        std::vector::push_back(&mut config.pools, PoolInfo {
            lp_token_addr,
            alloc_point,
            acc_farm_per_share: 0,
            last_reward_time: if (timestamp::now_seconds() < config.start_time) {
                config.start_time
            } else {
                timestamp::now_seconds()
            },
            total_staked: 0,
        });
    }
    
    // ============================================
    // Deposit
    // ============================================
    
    public entry fun deposit<LP>(
        user: &signer,
        config_addr: address,
        pool_id: u64,
        amount: u64,
    ) acquires FarmingConfig, UserPositions {
        let user_addr = signer::address_of(user);
        let config = borrow_global_mut<FarmingConfig>(config_addr);
        
        assert!(pool_id < std::vector::length(&config.pools), E_INVALID_POOL);
        
        let pool = std::vector::borrow_mut(&mut config.pools, pool_id);
        update_pool(pool, config.farm_per_second, config.total_alloc_point, config.end_time);
        
        // Get user position
        if (!exists<UserPositions>(user_addr)) {
            move_to(user, UserPositions { positions: smart_table::new() });
        };
        
        let positions = borrow_global_mut<UserPositions>(user_addr);
        let default_info = UserInfo { amount: 0, reward_debt: 0 };
        
        if (!smart_table::contains(&positions.positions, pool_id)) {
            smart_table::add(&mut positions.positions, pool_id, default_info);
        };
        
        let user_info = smart_table::borrow_mut(&mut positions.positions, pool_id);
        
        // Harvest pending rewards first
        if (user_info.amount > 0) {
            let pending = calc_pending(user_info, pool);
            if (pending > 0) {
                // Send FARM tokens to user (simplified)
                // In production: transfer from reward vault
            };
        };
        
        // Deposit LP tokens
        if (amount > 0) {
            coin::transfer<LP>(user, config_addr, amount);
            user_info.amount = user_info.amount + amount;
            pool.total_staked = pool.total_staked + amount;
        };
        
        // Update reward debt
        user_info.reward_debt = (user_info.amount as u128)
            * pool.acc_farm_per_share / 1_000_000_000_000u128;
        
        event::emit(Deposited { user: user_addr, pool_id, amount });
    }
    
    // ============================================
    // Withdraw
    // ============================================
    
    public entry fun withdraw<LP>(
        user: &signer,
        config_addr: address,
        pool_id: u64,
        amount: u64,
    ) acquires FarmingConfig, UserPositions {
        let user_addr = signer::address_of(user);
        let config = borrow_global_mut<FarmingConfig>(config_addr);
        
        let pool = std::vector::borrow_mut(&mut config.pools, pool_id);
        update_pool(pool, config.farm_per_second, config.total_alloc_point, config.end_time);
        
        let positions = borrow_global_mut<UserPositions>(user_addr);
        let user_info = smart_table::borrow_mut(&mut positions.positions, pool_id);
        
        assert!(user_info.amount >= amount, E_INSUFFICIENT_STAKE);
        
        let pending = calc_pending(user_info, pool);
        
        user_info.amount = user_info.amount - amount;
        pool.total_staked = pool.total_staked - amount;
        user_info.reward_debt = (user_info.amount as u128)
            * pool.acc_farm_per_share / 1_000_000_000_000u128;
        
        // Return LP to user (simplified: would use a signer cap for config_addr)
        
        event::emit(Withdrawn { user: user_addr, pool_id, amount, reward: pending });
    }
    
    // ============================================
    // View: Pending rewards
    // ============================================
    
    #[view]
    public fun pending_farm(
        config_addr: address,
        pool_id: u64,
        user_addr: address,
    ): u64 acquires FarmingConfig, UserPositions {
        let config = borrow_global<FarmingConfig>(config_addr);
        if (pool_id >= std::vector::length(&config.pools)) return 0;
        
        let pool = std::vector::borrow(&config.pools, pool_id);
        
        if (!exists<UserPositions>(user_addr)) return 0;
        let positions = borrow_global<UserPositions>(user_addr);
        if (!smart_table::contains(&positions.positions, pool_id)) return 0;
        
        let user_info = smart_table::borrow(&positions.positions, pool_id);
        
        // Simulate pool update
        let now = timestamp::now_seconds();
        let end = config.end_time;
        let time = if (now < end) { now } else { end };
        
        let mut acc = pool.acc_farm_per_share;
        if (time > pool.last_reward_time && pool.total_staked > 0) {
            let elapsed = time - pool.last_reward_time;
            let reward = (config.farm_per_second as u128)
                * (pool.alloc_point as u128)
                * (elapsed as u128)
                / (config.total_alloc_point as u128);
            acc = acc + reward * 1_000_000_000_000u128 / (pool.total_staked as u128);
        };
        
        let pending_shares = (user_info.amount as u128) * acc / 1_000_000_000_000u128;
        if (pending_shares <= user_info.reward_debt) return 0;
        (pending_shares - user_info.reward_debt) as u64
    }
    
    // ============================================
    // Helpers
    // ============================================
    
    fun update_pool(
        pool: &mut PoolInfo,
        farm_per_second: u64,
        total_alloc: u64,
        end_time: u64,
    ) {
        let now = timestamp::now_seconds();
        let time = if (now < end_time) { now } else { end_time };
        
        if (time <= pool.last_reward_time || pool.total_staked == 0) {
            pool.last_reward_time = time;
            return
        };
        
        let elapsed = time - pool.last_reward_time;
        let reward = (farm_per_second as u128)
            * (pool.alloc_point as u128)
            * (elapsed as u128)
            / (total_alloc as u128);
        
        pool.acc_farm_per_share = pool.acc_farm_per_share
            + reward * 1_000_000_000_000u128 / (pool.total_staked as u128);
        pool.last_reward_time = time;
    }
    
    fun update_all_pools(config: &mut FarmingConfig) {
        let i = 0;
        let len = std::vector::length(&config.pools);
        while (i < len) {
            let pool = std::vector::borrow_mut(&mut config.pools, i);
            update_pool(pool, config.farm_per_second, config.total_alloc_point, config.end_time);
            i = i + 1;
        };
    }
    
    fun calc_pending(user_info: &UserInfo, pool: &PoolInfo): u64 {
        let shares = (user_info.amount as u128) * pool.acc_farm_per_share / 1_000_000_000_000u128;
        if (shares <= user_info.reward_debt) return 0;
        (shares - user_info.reward_debt) as u64
    }
}
```

---

## Vote-Escrow (veToken)

```move
module governance::ve_token {
    use std::signer;
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::timestamp;
    use aptos_framework::event;
    
    // ============================================
    // veToken model (like veCRV)
    // ============================================
    
    // Lock token → get voting power
    // More lock time = more voting power
    // Max lock = 4 years, max multiplier = 4x
    // Voting power decays linearly over lock period
    
    struct GOV {}  // Governance token type
    
    const MAX_LOCK_SECONDS: u64 = 4 * 365 * 24 * 3600;  // 4 years
    const WEEK_SECONDS: u64 = 7 * 24 * 3600;
    
    struct Lock has key {
        amount: u64,
        locked_until: u64,
        created_at: u64,
    }
    
    struct VeConfig has key {
        admin: address,
        total_locked: u64,
        total_ve_supply: u64,
    }
    
    #[event]
    struct Locked has drop, store {
        user: address,
        amount: u64,
        locked_until: u64,
        voting_power: u64,
    }
    
    #[event]
    struct Unlocked has drop, store {
        user: address,
        amount: u64,
    }
    
    const E_ALREADY_LOCKED: u64 = 1;
    const E_LOCK_TOO_SHORT: u64 = 2;
    const E_LOCK_TOO_LONG: u64 = 3;
    const E_STILL_LOCKED: u64 = 4;
    const E_NO_LOCK: u64 = 5;
    
    // ============================================
    // Lock tokens
    // ============================================
    
    public entry fun lock(
        user: &signer,
        amount: u64,
        lock_duration_seconds: u64,
    ) {
        let user_addr = signer::address_of(user);
        assert!(!exists<Lock>(user_addr), E_ALREADY_LOCKED);
        
        // Round duration to nearest week
        let weeks = lock_duration_seconds / WEEK_SECONDS;
        assert!(weeks >= 1, E_LOCK_TOO_SHORT);
        
        let rounded_duration = weeks * WEEK_SECONDS;
        assert!(rounded_duration <= MAX_LOCK_SECONDS, E_LOCK_TOO_LONG);
        
        let now = timestamp::now_seconds();
        let locked_until = now + rounded_duration;
        
        // Transfer tokens from user
        coin::transfer<GOV>(user, user_addr, amount);  // lock in same account
        
        let voting_power = calculate_voting_power(amount, rounded_duration);
        
        move_to(user, Lock {
            amount,
            locked_until,
            created_at: now,
        });
        
        event::emit(Locked { user: user_addr, amount, locked_until, voting_power });
    }
    
    // ============================================
    // Extend lock
    // ============================================
    
    public entry fun extend_lock(
        user: &signer,
        additional_duration_seconds: u64,
    ) acquires Lock {
        let user_addr = signer::address_of(user);
        assert!(exists<Lock>(user_addr), E_NO_LOCK);
        
        let lock = borrow_global_mut<Lock>(user_addr);
        let now = timestamp::now_seconds();
        
        let current_remaining = if (lock.locked_until > now) { lock.locked_until - now } else { 0 };
        let new_duration = current_remaining + additional_duration_seconds;
        
        let weeks = new_duration / WEEK_SECONDS;
        let rounded = weeks * WEEK_SECONDS;
        assert!(rounded <= MAX_LOCK_SECONDS, E_LOCK_TOO_LONG);
        
        lock.locked_until = now + rounded;
    }
    
    // ============================================
    // Unlock (after lock expires)
    // ============================================
    
    public entry fun unlock(user: &signer) acquires Lock {
        let user_addr = signer::address_of(user);
        assert!(exists<Lock>(user_addr), E_NO_LOCK);
        
        let lock = borrow_global<Lock>(user_addr);
        let now = timestamp::now_seconds();
        assert!(now >= lock.locked_until, E_STILL_LOCKED);
        
        let Lock { amount, locked_until: _, created_at: _ } = move_from<Lock>(user_addr);
        
        event::emit(Unlocked { user: user_addr, amount });
    }
    
    // ============================================
    // View: voting power
    // ============================================
    
    #[view]
    public fun voting_power(user_addr: address): u64 acquires Lock {
        if (!exists<Lock>(user_addr)) return 0;
        
        let lock = borrow_global<Lock>(user_addr);
        let now = timestamp::now_seconds();
        
        if (now >= lock.locked_until) return 0;
        
        let remaining = lock.locked_until - now;
        calculate_voting_power(lock.amount, remaining)
    }
    
    #[view]
    public fun lock_info(user_addr: address): (u64, u64) acquires Lock {
        if (!exists<Lock>(user_addr)) return (0, 0);
        let lock = borrow_global<Lock>(user_addr);
        (lock.amount, lock.locked_until)
    }
    
    // ============================================
    // Helpers
    // ============================================
    
    // Voting power = amount * remaining_time / max_time
    fun calculate_voting_power(amount: u64, duration_seconds: u64): u64 {
        let capped = if (duration_seconds > MAX_LOCK_SECONDS) {
            MAX_LOCK_SECONDS
        } else {
            duration_seconds
        };
        (amount as u128 * capped as u128 / MAX_LOCK_SECONDS as u128) as u64
    }
}
```

---

## ตัวอย่าง: Complete Staking Protocol

```move
module staking::protocol {
    use std::signer;
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::timestamp;
    use aptos_framework::event;
    
    struct STAKE_TOKEN {}
    struct REWARD_TOKEN {}
    
    struct StakingPool has key {
        admin: address,
        total_staked: u64,
        acc_reward_per_share: u128,
        last_reward_time: u64,
        reward_per_second: u64,
        reward_balance: Coin<REWARD_TOKEN>,
        lock_duration: u64,
        early_exit_penalty_bps: u64,
    }
    
    struct UserStake has key {
        amount: u64,
        reward_debt: u128,
        stake_time: u64,
    }
    
    #[event]
    struct StakeEvent has drop, store { user: address, amount: u64 }
    #[event]
    struct UnstakeEvent has drop, store { user: address, amount: u64, penalty: u64, reward: u64 }
    #[event]
    struct ClaimEvent has drop, store { user: address, reward: u64 }
    
    const PRECISION: u128 = 1_000_000_000_000u128;
    const E_ZERO: u64 = 1;
    const E_INSUFFICIENT: u64 = 2;
    
    public entry fun initialize(
        admin: &signer,
        reward_per_second: u64,
        lock_duration: u64,
        early_exit_penalty_bps: u64,
        initial_rewards: u64,
    ) {
        let admin_addr = signer::address_of(admin);
        let now = timestamp::now_seconds();
        
        let reward_balance = coin::withdraw<REWARD_TOKEN>(admin, initial_rewards);
        
        move_to(admin, StakingPool {
            admin: admin_addr,
            total_staked: 0,
            acc_reward_per_share: 0,
            last_reward_time: now,
            reward_per_second,
            reward_balance,
            lock_duration,
            early_exit_penalty_bps,
        });
    }
    
    public entry fun stake(
        user: &signer,
        pool_addr: address,
        amount: u64,
    ) acquires StakingPool, UserStake {
        assert!(amount > 0, E_ZERO);
        let user_addr = signer::address_of(user);
        let pool = borrow_global_mut<StakingPool>(pool_addr);
        
        update_pool(pool);
        
        // Harvest pending if already staking
        if (exists<UserStake>(user_addr)) {
            let stake = borrow_global_mut<UserStake>(user_addr);
            let pending = calc_pending(stake, pool);
            if (pending > 0) {
                let reward = coin::extract(&mut pool.reward_balance, pending);
                coin::deposit<REWARD_TOKEN>(user_addr, reward);
                event::emit(ClaimEvent { user: user_addr, reward: pending });
            };
            stake.amount = stake.amount + amount;
            stake.reward_debt = (stake.amount as u128) * pool.acc_reward_per_share / PRECISION;
            stake.stake_time = timestamp::now_seconds();
        } else {
            move_to(user, UserStake {
                amount,
                reward_debt: (amount as u128) * pool.acc_reward_per_share / PRECISION,
                stake_time: timestamp::now_seconds(),
            });
        };
        
        coin::transfer<STAKE_TOKEN>(user, pool_addr, amount);
        pool.total_staked = pool.total_staked + amount;
        
        event::emit(StakeEvent { user: user_addr, amount });
    }
    
    public entry fun unstake(
        user: &signer,
        pool_addr: address,
        amount: u64,
    ) acquires StakingPool, UserStake {
        assert!(amount > 0, E_ZERO);
        let user_addr = signer::address_of(user);
        let pool = borrow_global_mut<StakingPool>(pool_addr);
        
        update_pool(pool);
        
        let stake = borrow_global_mut<UserStake>(user_addr);
        assert!(stake.amount >= amount, E_INSUFFICIENT);
        
        let pending = calc_pending(stake, pool);
        
        // Check for early exit penalty
        let now = timestamp::now_seconds();
        let penalty = if (now < stake.stake_time + pool.lock_duration) {
            amount * pool.early_exit_penalty_bps / 10_000
        } else {
            0
        };
        
        let return_amount = amount - penalty;
        
        stake.amount = stake.amount - amount;
        stake.reward_debt = (stake.amount as u128) * pool.acc_reward_per_share / PRECISION;
        pool.total_staked = pool.total_staked - amount;
        
        // Return staked tokens (minus penalty which stays in pool)
        // In production: transfer from pool using a signer cap
        
        // Send rewards
        if (pending > 0) {
            let reward = coin::extract(&mut pool.reward_balance, pending);
            coin::deposit<REWARD_TOKEN>(user_addr, reward);
        };
        
        event::emit(UnstakeEvent {
            user: user_addr,
            amount: return_amount,
            penalty,
            reward: pending,
        });
    }
    
    public entry fun claim_rewards(
        user: &signer,
        pool_addr: address,
    ) acquires StakingPool, UserStake {
        let user_addr = signer::address_of(user);
        let pool = borrow_global_mut<StakingPool>(pool_addr);
        
        update_pool(pool);
        
        let stake = borrow_global_mut<UserStake>(user_addr);
        let pending = calc_pending(stake, pool);
        
        assert!(pending > 0, E_ZERO);
        
        stake.reward_debt = (stake.amount as u128) * pool.acc_reward_per_share / PRECISION;
        
        let reward = coin::extract(&mut pool.reward_balance, pending);
        coin::deposit<REWARD_TOKEN>(user_addr, reward);
        
        event::emit(ClaimEvent { user: user_addr, reward: pending });
    }
    
    #[view]
    public fun pending_rewards(pool_addr: address, user_addr: address): u64 
    acquires StakingPool, UserStake {
        if (!exists<UserStake>(user_addr)) return 0;
        
        let pool = borrow_global<StakingPool>(pool_addr);
        let stake = borrow_global<UserStake>(user_addr);
        
        // Simulate update
        let now = timestamp::now_seconds();
        let mut acc = pool.acc_reward_per_share;
        if (now > pool.last_reward_time && pool.total_staked > 0) {
            let elapsed = now - pool.last_reward_time;
            let reward = (pool.reward_per_second as u128) * (elapsed as u128);
            acc = acc + reward * PRECISION / (pool.total_staked as u128);
        };
        
        let shares = (stake.amount as u128) * acc / PRECISION;
        if (shares <= stake.reward_debt) return 0;
        (shares - stake.reward_debt) as u64
    }
    
    fun update_pool(pool: &mut StakingPool) {
        let now = timestamp::now_seconds();
        if (now <= pool.last_reward_time || pool.total_staked == 0) {
            pool.last_reward_time = now;
            return
        };
        let elapsed = now - pool.last_reward_time;
        let reward = (pool.reward_per_second as u128) * (elapsed as u128);
        pool.acc_reward_per_share = pool.acc_reward_per_share
            + reward * PRECISION / (pool.total_staked as u128);
        pool.last_reward_time = now;
    }
    
    fun calc_pending(stake: &UserStake, pool: &StakingPool): u64 {
        let shares = (stake.amount as u128) * pool.acc_reward_per_share / PRECISION;
        if (shares <= stake.reward_debt) return 0;
        (shares - stake.reward_debt) as u64
    }
}
```

---

## สรุป Staking Patterns

| Pattern | Description | Use Case |
|---------|-------------|----------|
| Simple Staking | Lock → earn flat rate | Basic reward distribution |
| Per-share accumulator | Reward grows with time, claimed proportionally | MasterChef, Sushiswap |
| Liquid Staking | Get liquid token back | stAPT, stETH |
| veToken | Lock for voting power | Curve, governance |
| Time-weighted | Longer stake = more rewards | Loyalty programs |

---

**ก่อนหน้า**: [Part 29 - Lending Protocol ←](part-29-lending-protocol.md)
**ต่อไป**: [Part 31 - Advanced Patterns →](../advanced/part-31-advanced-patterns.md)
