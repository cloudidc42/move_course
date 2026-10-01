# Part 45: Liquid Staking Protocol

## สารบัญ
- [Liquid Staking คืออะไร](#liquid-staking-คืออะไร)
- [Architecture Overview](#architecture-overview)
- [Validator Management](#validator-management)
- [Exchange Rate & Yield Accrual](#exchange-rate--yield-accrual)
- [Delegation & Unbonding](#delegation--unbonding)
- [ตัวอย่าง: Complete lstAPT Protocol](#ตัวอย่าง-complete-lstapt-protocol)

---

## Liquid Staking คืออะไร

```
Traditional Staking:
  APT → Lock APT → Earn rewards → Wait 30 days to unlock
  Problem: APT is illiquid during staking

Liquid Staking:
  APT → Deposit → Receive lstAPT → Use lstAPT in DeFi
         ↓
  Protocol stakes APT with validators
  lstAPT appreciates vs APT as rewards accrue
  
  Exchange Rate: 1 lstAPT = (1 + accumulated yield) APT
  
Benefits:
  - Earn staking yield
  - Keep liquidity
  - Use in DeFi (collateral, liquidity pools)
  - No minimum lock period (instant via DEX)

Key Metrics:
  - APY: Annual Percentage Yield from staking
  - Exchange Rate: lstAPT/APT ratio
  - Unbonding Period: 30 days (Aptos)
  - Slashing Risk: Validator misbehavior

Protocols:
  - Lido (Ethereum) - largest liquid staking
  - Marinade (Solana) 
  - Similar can be built on Aptos/Sui
```

---

## Architecture Overview

```
Users ──deposit APT──→ lstAPT Contract ──delegate──→ Validator 1
                              ↑                  ──delegate──→ Validator 2
Users ──burn lstAPT──→ (get APT back)            ──delegate──→ Validator 3
                              ↓
                       Exchange Rate Oracle
                       (tracks APT per lstAPT)

Components:
  1. StakingPool: holds delegation state
  2. ValidatorSet: which validators + weights
  3. lstAPT Token: liquid receipt token
  4. WithdrawQueue: manages unbonding requests
  5. RewardAccumulator: tracks yield
  6. Fee Treasury: protocol fee (10% of yield)
```

---

## Validator Management

```move
module liquid_staking::validator_set {
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // Validator registry with scoring
    // ============================================
    
    struct ValidatorInfo has copy, drop, store {
        addr: address,
        delegated_amount: u64,     // APT delegated to this validator
        target_weight: u64,         // Basis points (total = 10000)
        performance_score: u64,     // 0-100, updated from validator stats
        commission_bps: u64,        // Validator commission
        active: bool,
    }
    
    struct ValidatorSet has key {
        admin: address,
        validators: SmartTable<address, ValidatorInfo>,
        validator_list: vector<address>,      // ordered list
        total_delegated: u64,
        max_validators: u64,
        min_validator_stake: u64,
    }
    
    const E_WEIGHT_OVERFLOW: u64 = 1;
    const E_VALIDATOR_EXISTS: u64 = 2;
    const E_VALIDATOR_NOT_FOUND: u64 = 3;
    const E_TOO_MANY_VALIDATORS: u64 = 4;
    
    public entry fun add_validator(
        admin: &signer,
        validator_set_addr: address,
        validator: address,
        target_weight: u64,
        commission_bps: u64,
    ) acquires ValidatorSet {
        let vs = borrow_global_mut<ValidatorSet>(validator_set_addr);
        assert!(std::signer::address_of(admin) == vs.admin, 0);
        assert!(
            std::vector::length(&vs.validator_list) < vs.max_validators,
            E_TOO_MANY_VALIDATORS
        );
        assert!(!smart_table::contains(&vs.validators, validator), E_VALIDATOR_EXISTS);
        
        // Verify new total weight <= 10000
        let current_total = total_weight(vs);
        assert!(current_total + target_weight <= 10000, E_WEIGHT_OVERFLOW);
        
        smart_table::add(&mut vs.validators, validator, ValidatorInfo {
            addr: validator,
            delegated_amount: 0,
            target_weight,
            performance_score: 100,
            commission_bps,
            active: true,
        });
        std::vector::push_back(&mut vs.validator_list, validator);
    }
    
    public entry fun update_validator_weight(
        admin: &signer,
        validator_set_addr: address,
        validator: address,
        new_weight: u64,
    ) acquires ValidatorSet {
        let vs = borrow_global_mut<ValidatorSet>(validator_set_addr);
        assert!(std::signer::address_of(admin) == vs.admin, 0);
        assert!(smart_table::contains(&vs.validators, validator), E_VALIDATOR_NOT_FOUND);
        
        let info = smart_table::borrow_mut(&mut vs.validators, validator);
        let old_weight = info.target_weight;
        let current_total = total_weight(vs);
        assert!(
            current_total - old_weight + new_weight <= 10000,
            E_WEIGHT_OVERFLOW
        );
        
        info.target_weight = new_weight;
    }
    
    // Rebalance: return list of validators and amounts to adjust
    public fun compute_rebalance(
        vs: &ValidatorSet,
        total_stake: u64,
    ): (vector<address>, vector<u64>, vector<u64>) {
        let n = std::vector::length(&vs.validator_list);
        let mut targets = std::vector::empty<u64>();
        let mut to_delegate = std::vector::empty<u64>();
        let mut to_undelegate = std::vector::empty<u64>();
        
        let mut i = 0u64;
        while (i < n) {
            let validator = *std::vector::borrow(&vs.validator_list, i);
            let info = smart_table::borrow(&vs.validators, validator);
            
            let target_amount = total_stake * info.target_weight / 10000;
            let current = info.delegated_amount;
            
            std::vector::push_back(&mut targets, target_amount);
            
            if (target_amount > current) {
                std::vector::push_back(&mut to_delegate, target_amount - current);
                std::vector::push_back(&mut to_undelegate, 0);
            } else {
                std::vector::push_back(&mut to_delegate, 0);
                std::vector::push_back(&mut to_undelegate, current - target_amount);
            };
            
            i = i + 1;
        };
        
        (vs.validator_list, to_delegate, to_undelegate)
    }
    
    fun total_weight(vs: &ValidatorSet): u64 {
        let n = std::vector::length(&vs.validator_list);
        let mut total = 0u64;
        let mut i = 0u64;
        while (i < n) {
            let v = *std::vector::borrow(&vs.validator_list, i);
            let info = smart_table::borrow(&vs.validators, v);
            total = total + info.target_weight;
            i = i + 1;
        };
        total
    }
}
```

---

## Exchange Rate & Yield Accrual

```move
module liquid_staking::exchange_rate {
    use aptos_framework::timestamp;
    
    // ============================================
    // Exchange rate: APT per lstAPT
    // Increases monotonically as rewards accrue
    // ============================================
    
    // Exchange rate stored as APT per 1e9 lstAPT
    // (precision: 9 decimal places)
    const PRECISION: u64 = 1_000_000_000;  // 1e9
    
    struct RateState has key {
        // APT per 1e9 lstAPT
        // Initial: 1_000_000_000 (1:1)
        // After 10% yield: 1_100_000_000 (1.1 APT per lstAPT)
        rate: u64,
        
        last_update: u64,
        
        // Track total APT and lstAPT for verification
        total_apt: u64,
        total_lst_apt: u64,
        
        // Accumulated rewards (not yet compounded)
        pending_rewards: u64,
        
        // Protocol fee (e.g., 10% of rewards)
        fee_bps: u64,
        fee_recipient: address,
    }
    
    // Convert APT → lstAPT at current rate
    public fun apt_to_lst(state: &RateState, apt_amount: u64): u64 {
        // lst = apt * PRECISION / rate
        (apt_amount as u128 * (PRECISION as u128) / (state.rate as u128)) as u64
    }
    
    // Convert lstAPT → APT at current rate
    public fun lst_to_apt(state: &RateState, lst_amount: u64): u64 {
        // apt = lst * rate / PRECISION
        (lst_amount as u128 * (state.rate as u128) / (PRECISION as u128)) as u64
    }
    
    // Accrue rewards: increases exchange rate
    public fun accrue_rewards(
        state: &mut RateState,
        new_rewards: u64,
    ): u64 {  // Returns fee amount
        // Calculate protocol fee
        let fee = new_rewards * state.fee_bps / 10_000;
        let net_rewards = new_rewards - fee;
        
        // New rate = (total_apt + net_rewards) * PRECISION / total_lst_apt
        if (state.total_lst_apt == 0) return fee;
        
        state.total_apt = state.total_apt + net_rewards;
        state.rate = ((state.total_apt as u128) * (PRECISION as u128) 
                     / (state.total_lst_apt as u128)) as u64;
        state.last_update = timestamp::now_seconds();
        
        fee
    }
    
    // Record deposit
    public fun record_deposit(
        state: &mut RateState,
        apt_amount: u64,
        lst_minted: u64,
    ) {
        state.total_apt = state.total_apt + apt_amount;
        state.total_lst_apt = state.total_lst_apt + lst_minted;
    }
    
    // Record withdrawal
    public fun record_withdrawal(
        state: &mut RateState,
        apt_amount: u64,
        lst_burned: u64,
    ) {
        state.total_apt = state.total_apt - apt_amount;
        state.total_lst_apt = state.total_lst_apt - lst_burned;
    }
    
    #[view]
    public fun current_rate(state_addr: address): u64 acquires RateState {
        borrow_global<RateState>(state_addr).rate
    }
    
    #[view]
    public fun total_staked(state_addr: address): u64 acquires RateState {
        borrow_global<RateState>(state_addr).total_apt
    }
}
```

---

## Delegation & Unbonding

```move
module liquid_staking::withdraw_queue {
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::timestamp;
    
    // ============================================
    // Withdrawal queue manages the 30-day unbonding
    // ============================================
    
    const UNBONDING_PERIOD: u64 = 30 * 86_400;  // 30 days in seconds
    
    struct WithdrawRequest has copy, drop, store {
        user: address,
        apt_amount: u64,         // APT to receive after unbonding
        unlock_time: u64,        // When can be claimed
        lst_burned: u64,         // lstAPT burned
        claimed: bool,
    }
    
    struct WithdrawQueue has key {
        requests: SmartTable<u64, WithdrawRequest>,
        next_request_id: u64,
        
        // Total APT in unbonding (not yet liquid)
        total_unbonding: u64,
        
        // Instant redemption pool (from protocol's liquid buffer)
        instant_pool: u64,
        instant_pool_target_bps: u64,  // Target % of TVL to keep liquid
    }
    
    const E_ALREADY_CLAIMED: u64 = 1;
    const E_NOT_UNLOCKED: u64 = 2;
    const E_NOT_OWNER: u64 = 3;
    
    // Create withdrawal request (burn lstAPT, queue for 30 days)
    public fun request_withdraw(
        queue: &mut WithdrawQueue,
        user: address,
        apt_amount: u64,
        lst_burned: u64,
    ): u64 {  // Returns request ID
        // Check if instant redemption possible
        if (queue.instant_pool >= apt_amount) {
            // Instant redemption from buffer
            queue.instant_pool = queue.instant_pool - apt_amount;
            // Signal instant claim (use request_id = 0 for instant)
            return 0
        };
        
        let request_id = queue.next_request_id;
        queue.next_request_id = request_id + 1;
        
        let unlock_time = timestamp::now_seconds() + UNBONDING_PERIOD;
        
        smart_table::add(&mut queue.requests, request_id, WithdrawRequest {
            user,
            apt_amount,
            unlock_time,
            lst_burned,
            claimed: false,
        });
        
        queue.total_unbonding = queue.total_unbonding + apt_amount;
        
        request_id
    }
    
    // Claim after unbonding period
    public fun claim_withdraw(
        queue: &mut WithdrawQueue,
        claimer: address,
        request_id: u64,
    ): u64 {  // Returns APT amount
        let request = smart_table::borrow_mut(&mut queue.requests, request_id);
        
        assert!(!request.claimed, E_ALREADY_CLAIMED);
        assert!(request.user == claimer, E_NOT_OWNER);
        assert!(
            timestamp::now_seconds() >= request.unlock_time,
            E_NOT_UNLOCKED
        );
        
        request.claimed = true;
        queue.total_unbonding = queue.total_unbonding - request.apt_amount;
        
        request.apt_amount
    }
    
    // Replenish instant pool from staking rewards
    public fun replenish_instant_pool(
        queue: &mut WithdrawQueue,
        amount: u64,
        total_tvl: u64,
    ) {
        let target = total_tvl * queue.instant_pool_target_bps / 10_000;
        if (queue.instant_pool < target) {
            let add = std::math64::min(amount, target - queue.instant_pool);
            queue.instant_pool = queue.instant_pool + add;
        };
    }
    
    #[view]
    public fun get_request(
        queue_addr: address,
        request_id: u64,
    ): WithdrawRequest acquires WithdrawQueue {
        *smart_table::borrow(
            &borrow_global<WithdrawQueue>(queue_addr).requests,
            request_id
        )
    }
    
    #[view]
    public fun can_instant_redeem(
        queue_addr: address,
        apt_amount: u64,
    ): bool acquires WithdrawQueue {
        borrow_global<WithdrawQueue>(queue_addr).instant_pool >= apt_amount
    }
}
```

---

## ตัวอย่าง: Complete lstAPT Protocol

```move
module liquid_staking::lst_apt {
    use std::signer;
    use aptos_framework::coin::{Self, Coin, MintCapability, BurnCapability};
    use aptos_framework::timestamp;
    use aptos_framework::event;
    use liquid_staking::exchange_rate;
    use liquid_staking::withdraw_queue;
    use liquid_staking::validator_set;
    
    // ============================================
    // lstAPT Token: liquid staking receipt
    // ============================================
    
    struct LstApt {}  // Coin type marker
    
    struct LstProtocol has key {
        admin: address,
        
        // Token capabilities
        mint_cap: MintCapability<LstApt>,
        burn_cap: BurnCapability<LstApt>,
        
        // Protocol state addresses
        rate_state_addr: address,
        queue_addr: address,
        validator_set_addr: address,
        
        // Protocol settings
        paused: bool,
        max_deposit: u64,       // Per-tx limit
        min_deposit: u64,
        total_deposited: u64,
        
        // Fee tracking
        accumulated_fees: u64,
    }
    
    #[event]
    struct Deposited has drop, store {
        user: address,
        apt_amount: u64,
        lst_minted: u64,
        rate: u64,
    }
    
    #[event]
    struct WithdrawRequested has drop, store {
        user: address,
        lst_burned: u64,
        apt_expected: u64,
        request_id: u64,
        unlock_time: u64,
    }
    
    #[event]
    struct RewardsAccrued has drop, store {
        rewards: u64,
        fee: u64,
        new_rate: u64,
    }
    
    const E_PAUSED: u64 = 1;
    const E_BELOW_MIN: u64 = 2;
    const E_ABOVE_MAX: u64 = 3;
    const E_ZERO_AMOUNT: u64 = 4;
    
    // ============================================
    // Deposit APT → Receive lstAPT
    // ============================================
    
    public entry fun deposit(
        user: &signer,
        protocol_addr: address,
        apt_amount: u64,
    ) acquires LstProtocol {
        let protocol = borrow_global_mut<LstProtocol>(protocol_addr);
        assert!(!protocol.paused, E_PAUSED);
        assert!(apt_amount >= protocol.min_deposit, E_BELOW_MIN);
        assert!(apt_amount <= protocol.max_deposit, E_ABOVE_MAX);
        
        let user_addr = signer::address_of(user);
        
        // Get current exchange rate
        let rate = exchange_rate::current_rate(protocol.rate_state_addr);
        
        // Calculate lstAPT to mint
        let rate_state = borrow_global<exchange_rate::RateState>(protocol.rate_state_addr);
        let lst_to_mint = exchange_rate::apt_to_lst(rate_state, apt_amount);
        assert!(lst_to_mint > 0, E_ZERO_AMOUNT);
        
        // Take APT from user
        let apt_coins = coin::withdraw<aptos_coin::AptosCoin>(user, apt_amount);
        // Deposit to protocol vault (simplified - real impl deposits to validator)
        coin::deposit<aptos_coin::AptosCoin>(protocol_addr, apt_coins);
        
        // Mint lstAPT to user
        let lst_coins = coin::mint<LstApt>(lst_to_mint, &protocol.mint_cap);
        coin::deposit<LstApt>(user_addr, lst_coins);
        
        // Update rate state
        let rate_state_mut = borrow_global_mut<exchange_rate::RateState>(protocol.rate_state_addr);
        exchange_rate::record_deposit(rate_state_mut, apt_amount, lst_to_mint);
        
        protocol.total_deposited = protocol.total_deposited + apt_amount;
        
        event::emit(Deposited {
            user: user_addr,
            apt_amount,
            lst_minted: lst_to_mint,
            rate,
        });
    }
    
    // ============================================
    // Request withdrawal (burn lstAPT → queue)
    // ============================================
    
    public entry fun request_withdraw(
        user: &signer,
        protocol_addr: address,
        lst_amount: u64,
    ) acquires LstProtocol {
        let protocol = borrow_global_mut<LstProtocol>(protocol_addr);
        assert!(!protocol.paused, E_PAUSED);
        
        let user_addr = signer::address_of(user);
        
        // Calculate APT amount
        let rate_state = borrow_global<exchange_rate::RateState>(protocol.rate_state_addr);
        let apt_amount = exchange_rate::lst_to_apt(rate_state, lst_amount);
        assert!(apt_amount > 0, E_ZERO_AMOUNT);
        
        // Burn lstAPT
        let lst_coins = coin::withdraw<LstApt>(user, lst_amount);
        coin::burn(lst_coins, &protocol.burn_cap);
        
        // Update rate state
        let rate_state_mut = borrow_global_mut<exchange_rate::RateState>(protocol.rate_state_addr);
        exchange_rate::record_withdrawal(rate_state_mut, apt_amount, lst_amount);
        
        // Create withdrawal request
        let queue = borrow_global_mut<withdraw_queue::WithdrawQueue>(protocol.queue_addr);
        let request_id = withdraw_queue::request_withdraw(
            queue,
            user_addr,
            apt_amount,
            lst_amount,
        );
        
        let unlock_time = timestamp::now_seconds() + 30 * 86_400;
        
        event::emit(WithdrawRequested {
            user: user_addr,
            lst_burned: lst_amount,
            apt_expected: apt_amount,
            request_id,
            unlock_time,
        });
    }
    
    // ============================================
    // Claim after unbonding period
    // ============================================
    
    public entry fun claim_withdraw(
        user: &signer,
        protocol_addr: address,
        request_id: u64,
    ) acquires LstProtocol {
        let protocol = borrow_global<LstProtocol>(protocol_addr);
        let user_addr = signer::address_of(user);
        
        let queue = borrow_global_mut<withdraw_queue::WithdrawQueue>(protocol.queue_addr);
        let apt_amount = withdraw_queue::claim_withdraw(queue, user_addr, request_id);
        
        // Send APT to user
        let apt_coins = coin::withdraw<aptos_coin::AptosCoin>(
            // protocol signer needed here in real impl
            &create_signer(protocol_addr),
            apt_amount
        );
        coin::deposit<aptos_coin::AptosCoin>(user_addr, apt_coins);
    }
    
    // ============================================
    // Operator: Accrue staking rewards
    // Called daily/weekly by operator bot
    // ============================================
    
    public entry fun accrue_rewards(
        operator: &signer,
        protocol_addr: address,
        new_rewards: u64,
    ) acquires LstProtocol {
        let protocol = borrow_global_mut<LstProtocol>(protocol_addr);
        
        let rate_state_mut = borrow_global_mut<exchange_rate::RateState>(protocol.rate_state_addr);
        let fee = exchange_rate::accrue_rewards(rate_state_mut, new_rewards);
        
        protocol.accumulated_fees = protocol.accumulated_fees + fee;
        
        let new_rate = exchange_rate::current_rate(protocol.rate_state_addr);
        
        event::emit(RewardsAccrued {
            rewards: new_rewards,
            fee,
            new_rate,
        });
    }
    
    // ============================================
    // View functions
    // ============================================
    
    #[view]
    public fun exchange_rate(protocol_addr: address): u64 acquires LstProtocol {
        let protocol = borrow_global<LstProtocol>(protocol_addr);
        exchange_rate::current_rate(protocol.rate_state_addr)
    }
    
    #[view]
    public fun apt_for_lst(protocol_addr: address, lst_amount: u64): u64 acquires LstProtocol {
        let protocol = borrow_global<LstProtocol>(protocol_addr);
        let rate_state = borrow_global<exchange_rate::RateState>(protocol.rate_state_addr);
        exchange_rate::lst_to_apt(rate_state, lst_amount)
    }
    
    #[view]
    public fun lst_for_apt(protocol_addr: address, apt_amount: u64): u64 acquires LstProtocol {
        let protocol = borrow_global<LstProtocol>(protocol_addr);
        let rate_state = borrow_global<exchange_rate::RateState>(protocol.rate_state_addr);
        exchange_rate::apt_to_lst(rate_state, apt_amount)
    }
    
    // Helper (would need framework support in real code)
    native fun create_signer(addr: address): signer;
}
```

---

## Sui Liquid Staking (SUI native)

```move
// sources/liquid_staking_sui.move
module liquid_staking::sui_lst {
    use sui::coin::{Self, Coin, TreasuryCap};
    use sui::balance::{Self, Balance};
    use sui::tx_context::{Self, TxContext};
    use sui::object::{Self, UID};
    use sui::transfer;
    use sui_system::sui_system::{Self, SuiSystemState};
    use sui_system::staking_pool::StakedSui;
    
    // ============================================
    // Sui native staking with liquid receipt token
    // ============================================
    
    struct LST_SUI {}  // OTW for coin
    
    struct LiquidStakingHub has key {
        id: UID,
        
        treasury_cap: TreasuryCap<LST_SUI>,
        
        // Staked SUI held by protocol
        staked_sui: vector<StakedSui>,
        
        // Total SUI value (including accrued rewards)
        total_sui_value: u64,
        
        // Total lstSUI supply
        total_lst_supply: u64,
        
        // Protocol fee (basis points of rewards)
        fee_bps: u64,
        fee_balance: Balance<sui::SUI>,
        
        // Preferred validator
        validator: address,
    }
    
    // Initialize the protocol
    fun init(otw: LST_SUI, ctx: &mut TxContext) {
        let (treasury_cap, metadata) = coin::create_currency(
            otw,
            9,
            b"lstSUI",
            b"Liquid Staked SUI",
            b"Liquid staking receipt for SUI",
            option::none(),
            ctx,
        );
        
        transfer::public_freeze_object(metadata);
        
        let hub = LiquidStakingHub {
            id: object::new(ctx),
            treasury_cap,
            staked_sui: vector::empty(),
            total_sui_value: 0,
            total_lst_supply: 0,
            fee_bps: 1000,  // 10%
            fee_balance: balance::zero(),
            validator: @0xVALIDATOR,
        };
        
        transfer::share_object(hub);
    }
    
    // Deposit SUI → lstSUI
    public entry fun deposit(
        hub: &mut LiquidStakingHub,
        system_state: &mut SuiSystemState,
        sui: Coin<sui::SUI>,
        ctx: &mut TxContext,
    ) {
        let sui_amount = coin::value(&sui);
        
        // Calculate lstSUI to mint (before staking, use current rate)
        let lst_amount = if (hub.total_lst_supply == 0) {
            sui_amount
        } else {
            (sui_amount as u128) * (hub.total_lst_supply as u128) 
                / (hub.total_sui_value as u128) as u64
        };
        
        // Stake SUI with validator
        let staked = sui_system::request_add_stake_non_entry(
            system_state,
            sui,
            hub.validator,
            ctx,
        );
        
        vector::push_back(&mut hub.staked_sui, staked);
        hub.total_sui_value = hub.total_sui_value + sui_amount;
        hub.total_lst_supply = hub.total_lst_supply + lst_amount;
        
        // Mint lstSUI
        let lst_coins = coin::mint(&mut hub.treasury_cap, lst_amount, ctx);
        transfer::public_transfer(lst_coins, tx_context::sender(ctx));
    }
    
    // Update total_sui_value from validator rewards
    // Called by protocol bot each epoch
    public entry fun sync_rewards(
        hub: &mut LiquidStakingHub,
        system_state: &mut SuiSystemState,
        ctx: &mut TxContext,
    ) {
        // In production: compute actual StakedSui value
        // This reads the current epoch rewards from the staking pool
        // For simplicity, we track via oracle/estimation
    }
    
    // Get current rate: SUI per lstSUI (in PRECISION = 1e9 units)
    public fun rate(hub: &LiquidStakingHub): u64 {
        if (hub.total_lst_supply == 0) return 1_000_000_000;
        ((hub.total_sui_value as u128) * 1_000_000_000 
            / (hub.total_lst_supply as u128)) as u64
    }
}
```

---

## Integration: lstAPT as Collateral

```move
module lending::lst_collateral {
    // ============================================
    // Use lstAPT as collateral in lending protocol
    // ============================================
    
    struct LstCollateralConfig has key {
        // Collateral parameters for lstAPT
        loan_to_value_bps: u64,      // 75% LTV
        liquidation_threshold_bps: u64,  // 80%
        liquidation_bonus_bps: u64,  // 5% bonus
        
        // Oracle for lstAPT/USD price
        // (uses exchange rate * APT/USD price)
        lst_protocol_addr: address,
        apt_usd_oracle: address,
    }
    
    // Get lstAPT value in USD
    public fun lst_apt_usd_value(
        config: &LstCollateralConfig,
        lst_amount: u64,
    ): u64 {
        // Get apt per lstAPT
        let apt_per_lst = liquid_staking::lst_apt::exchange_rate(
            config.lst_protocol_addr
        );
        
        // Convert to APT amount (rate is in PRECISION units)
        let apt_amount = (lst_amount as u128) * (apt_per_lst as u128) 
            / 1_000_000_000u128;
        
        // Multiply by APT/USD price from oracle
        // oracle::get_price(config.apt_usd_oracle) * apt_amount
        apt_amount as u64  // simplified
    }
    
    // Borrow against lstAPT collateral
    public entry fun borrow_against_lst(
        user: &signer,
        lst_amount: u64,
        borrow_amount: u64,
        config_addr: address,
    ) acquires LstCollateralConfig {
        let config = borrow_global<LstCollateralConfig>(config_addr);
        
        // Calculate max borrow
        let collateral_value = lst_apt_usd_value(config, lst_amount);
        let max_borrow = collateral_value * config.loan_to_value_bps / 10_000;
        
        assert!(borrow_amount <= max_borrow, 1);
        
        // Lock lstAPT as collateral
        // Mint stablecoin to user
    }
}
```

---

## สรุป Liquid Staking Design

```
Key Design Decisions:

1. Exchange Rate Precision
   - Use PRECISION = 1e9 for accuracy
   - Rate = total_APT / total_lstAPT * PRECISION
   - Monotonically increasing (only rewards, no slashing on Aptos)

2. Withdrawal Strategy
   - Instant redemption pool (5-10% of TVL)
   - Queue + 30-day unbonding for rest
   - Rebalance unbonding when possible

3. Validator Selection
   - Multiple validators for diversification
   - Weight by performance score
   - Regular rebalancing

4. Fee Model
   - 10% of staking rewards (industry standard)
   - Goes to protocol treasury/DAO

5. Security
   - Protocol upgrade is compatible (no breaking changes)
   - Emergency pause
   - Multisig admin

TVL Growth Strategy:
   lstAPT integration points:
   - AMM liquidity (lstAPT/APT, lstAPT/USDC)
   - Lending collateral
   - Yield strategies (autocompound rewards)
   - Leverage staking (borrow APT, stake, repeat)
```

---

**ก่อนหน้า**: [Part 44 - ZK Proofs ←](part-44-zkproofs.md)
**ต่อไป**: [Part 46 - Protocol Security Auditing →](part-46-security-auditing.md)
