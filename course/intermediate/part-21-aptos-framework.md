# Part 21: Aptos Framework Deep Dive

## สารบัญ
- [Aptos Framework Overview](#aptos-framework-overview)
- [aptos_framework::account](#aptos_frameworkaccount)
- [aptos_framework::timestamp](#aptos_frameworktimestamp)
- [aptos_framework::coin](#aptos_frameworkcoin)
- [aptos_framework::aptos_coin](#aptos_frameworkaptos_coin)
- [ตัวอย่างโปรแกรม: Subscription Service](#ตัวอย่างโปรแกรม-subscription-service)

---

## Aptos Framework Overview

```
aptos_framework/
├── account.move          - Account management
├── coin.move             - Fungible tokens (legacy)
├── fungible_asset.move   - New fungible asset standard
├── primary_fungible_store.move
├── timestamp.move        - Block timestamps
├── event.move            - Event emission
├── table.move            - Key-value storage
├── object.move           - Aptos Object model
├── code.move             - Code publishing
├── transaction_context.move
└── aptos_coin.move       - APT token
```

---

## aptos_framework::account

```move
module learning::account_usage {
    use std::signer;
    use aptos_framework::account;
    use aptos_framework::event::{Self, EventHandle};
    
    // ============================================
    // Account basics
    // ============================================
    
    // Account = an address with resources stored at it
    // Every address can potentially hold resources
    // "Account" in Aptos = has AccountResource struct
    
    // ============================================
    // Getting signer
    // ============================================
    
    // In transactions: signer is provided automatically from private key signature
    // In tests: signer is created via #[test(addr = @0x1)]
    
    public entry fun with_signer(user: &signer) {
        let addr = signer::address_of(user);
        // user is proof that the transaction was signed by addr
        let _ = addr;
    }
    
    // ============================================
    // create_signer_with_capability (for resource accounts)
    // ============================================
    
    // Resource accounts can create signers programmatically
    // Useful for: modules that need to hold assets, escrow, etc.
    
    struct EscrowSigner has key {
        cap: account::SignerCapability,
    }
    
    public entry fun create_escrow(creator: &signer, seed: vector<u8>) {
        let (escrow_signer, cap) = account::create_resource_account(creator, seed);
        let _ = escrow_signer;  // could initialize resources here
        
        move_to(creator, EscrowSigner { cap });
    }
    
    public fun use_escrow_signer(creator_addr: address) acquires EscrowSigner {
        let escrow = borrow_global<EscrowSigner>(creator_addr);
        let _signer = account::create_signer_with_capability(&escrow.cap);
        // Now _signer can be used to move_to/move_from the resource account
    }
    
    // ============================================
    // EventHandles
    // ============================================
    
    struct CustomEvent has drop, store {
        value: u64,
    }
    
    struct MyEvents has key {
        events: EventHandle<CustomEvent>,
    }
    
    public fun setup_events(account: &signer) {
        move_to(account, MyEvents {
            // account::new_event_handle creates EventHandle
            events: account::new_event_handle<CustomEvent>(account),
        });
    }
    
    public fun emit_custom_event(
        account_addr: address,
        value: u64,
    ) acquires MyEvents {
        let my_events = borrow_global_mut<MyEvents>(account_addr);
        event::emit_event(&mut my_events.events, CustomEvent { value });
    }
    
    // ============================================
    // Account existence
    // ============================================
    
    public fun account_exists(addr: address): bool {
        account::exists_at(addr)
    }
    
    public fun get_sequence_number(addr: address): u64 {
        account::get_sequence_number(addr)
    }
    
    // ============================================
    // Resource accounts pattern
    // ============================================
    
    struct ProtocolAccount has key {
        signer_cap: account::SignerCapability,
        treasury_addr: address,
    }
    
    public entry fun initialize_protocol(deployer: &signer) {
        // Create resource account for protocol treasury
        let seed = b"protocol_treasury";
        let (treasury_signer, cap) = account::create_resource_account(deployer, seed);
        
        let treasury_addr = signer::address_of(&treasury_signer);
        
        // Store capability in deployer's account
        move_to(deployer, ProtocolAccount {
            signer_cap: cap,
            treasury_addr,
        });
    }
    
    public fun deposit_to_treasury(
        deployer_addr: address,
        amount: u64,
    ) acquires ProtocolAccount {
        let protocol = borrow_global<ProtocolAccount>(deployer_addr);
        let treasury_signer = account::create_signer_with_capability(&protocol.signer_cap);
        
        // Use treasury_signer to manage treasury resources
        let _ = (amount, treasury_signer);
    }
}
```

---

## aptos_framework::timestamp

```move
module learning::timestamp_usage {
    use aptos_framework::timestamp;
    
    // ============================================
    // Getting current time
    // ============================================
    
    // timestamp::now_seconds() - current block timestamp in seconds
    // timestamp::now_microseconds() - current block timestamp in microseconds
    
    public fun get_current_time(): (u64, u64) {
        let seconds = timestamp::now_seconds();
        let micros = timestamp::now_microseconds();
        (seconds, micros)
    }
    
    // ============================================
    // Time-based logic
    // ============================================
    
    struct TimedLock has key {
        unlock_time: u64,  // unix timestamp in seconds
        owner: address,
        amount: u64,
    }
    
    const E_STILL_LOCKED: u64 = 1;
    const E_INVALID_DURATION: u64 = 2;
    
    public entry fun lock_funds(
        user: &signer,
        amount: u64,
        lock_duration_seconds: u64,
    ) {
        assert!(lock_duration_seconds > 0, E_INVALID_DURATION);
        assert!(lock_duration_seconds <= 365 * 24 * 3600, E_INVALID_DURATION);  // max 1 year
        
        let now = timestamp::now_seconds();
        let unlock_time = now + lock_duration_seconds;
        
        use std::signer;
        let owner = signer::address_of(user);
        
        move_to(user, TimedLock {
            unlock_time,
            owner,
            amount,
        });
    }
    
    public entry fun unlock_funds(user: &signer) acquires TimedLock {
        use std::signer;
        let addr = signer::address_of(user);
        
        let lock = borrow_global<TimedLock>(addr);
        let now = timestamp::now_seconds();
        
        assert!(now >= lock.unlock_time, E_STILL_LOCKED);
        
        let TimedLock { unlock_time: _, owner: _, amount: _ } = move_from<TimedLock>(addr);
    }
    
    // ============================================
    // Vesting schedule
    // ============================================
    
    struct VestingSchedule has key {
        total_amount: u64,
        claimed: u64,
        start_time: u64,
        cliff_seconds: u64,
        vesting_duration_seconds: u64,
        beneficiary: address,
    }
    
    public fun vested_amount(schedule: &VestingSchedule): u64 {
        let now = timestamp::now_seconds();
        
        // Before cliff: nothing vested
        if (now < schedule.start_time + schedule.cliff_seconds) {
            return 0
        };
        
        // After full vesting: everything vested
        let end_time = schedule.start_time + schedule.vesting_duration_seconds;
        if (now >= end_time) {
            return schedule.total_amount
        };
        
        // Linear vesting
        let elapsed = now - schedule.start_time;
        schedule.total_amount * elapsed / schedule.vesting_duration_seconds
    }
    
    public fun claimable_amount(schedule: &VestingSchedule): u64 {
        let vested = vested_amount(schedule);
        if (vested <= schedule.claimed) { 0 }
        else { vested - schedule.claimed }
    }
    
    // ============================================
    // Expiry patterns
    // ============================================
    
    struct Offer has key, drop {
        price: u64,
        expires_at: u64,
    }
    
    const E_OFFER_EXPIRED: u64 = 3;
    
    public fun validate_offer(offer: &Offer) {
        let now = timestamp::now_seconds();
        assert!(now <= offer.expires_at, E_OFFER_EXPIRED);
    }
    
    public fun is_offer_valid(offer: &Offer): bool {
        let now = timestamp::now_seconds();
        now <= offer.expires_at
    }
    
    // ============================================
    // Cooldown pattern
    // ============================================
    
    struct UserStats has key {
        last_claim: u64,
        cooldown_seconds: u64,
    }
    
    const E_COOLDOWN: u64 = 4;
    
    public fun can_claim(stats: &UserStats): bool {
        let now = timestamp::now_seconds();
        now >= stats.last_claim + stats.cooldown_seconds
    }
    
    public fun claim(stats: &mut UserStats) {
        assert!(can_claim(stats), E_COOLDOWN);
        stats.last_claim = timestamp::now_seconds();
    }
}
```

---

## aptos_framework::coin

```move
module learning::coin_usage {
    use std::signer;
    use aptos_framework::coin::{Self, Coin, MintCapability, BurnCapability, FreezeCapability};
    use aptos_framework::account;
    
    // ============================================
    // Coin Standard (Legacy but still widely used)
    // ============================================
    
    // Coin<CoinType> is a resource representing fungible tokens
    // - Has store ability (can be stored)
    // - Does NOT have copy/drop (asset semantics)
    
    // Define your token type
    struct MyToken {}
    
    // Coin capabilities stored by token creator
    struct MyCoinCaps has key {
        mint_cap: MintCapability<MyToken>,
        burn_cap: BurnCapability<MyToken>,
        freeze_cap: FreezeCapability<MyToken>,
    }
    
    // ============================================
    // Initialize a coin
    // ============================================
    
    public entry fun initialize_my_token(creator: &signer) {
        let (burn_cap, freeze_cap, mint_cap) = coin::initialize<MyToken>(
            creator,
            b"My Token",       // name
            b"MTK",            // symbol
            8,                 // decimals
            true,              // monitor supply
        );
        
        move_to(creator, MyCoinCaps { mint_cap, burn_cap, freeze_cap });
    }
    
    // ============================================
    // Mint coins
    // ============================================
    
    public entry fun mint_tokens(
        creator: &signer,
        to: address,
        amount: u64,
    ) acquires MyCoinCaps {
        let creator_addr = signer::address_of(creator);
        let caps = borrow_global<MyCoinCaps>(creator_addr);
        
        let minted = coin::mint<MyToken>(amount, &caps.mint_cap);
        
        // Deposit to recipient's coin store
        coin::deposit<MyToken>(to, minted);
    }
    
    // ============================================
    // Transfer coins
    // ============================================
    
    public entry fun transfer_tokens(
        from: &signer,
        to: address,
        amount: u64,
    ) {
        coin::transfer<MyToken>(from, to, amount);
    }
    
    // ============================================
    // Burn coins
    // ============================================
    
    public entry fun burn_tokens(
        from: &signer,
        creator_addr: address,
        amount: u64,
    ) acquires MyCoinCaps {
        let caps = borrow_global<MyCoinCaps>(creator_addr);
        let coins = coin::withdraw<MyToken>(from, amount);
        coin::burn<MyToken>(coins, &caps.burn_cap);
    }
    
    // ============================================
    // Register to receive coins
    // ============================================
    
    public entry fun register_for_token(user: &signer) {
        coin::register<MyToken>(user);
    }
    
    // ============================================
    // Query balance
    // ============================================
    
    #[view]
    public fun token_balance(addr: address): u64 {
        if (!coin::is_account_registered<MyToken>(addr)) { return 0 };
        coin::balance<MyToken>(addr)
    }
    
    // ============================================
    // Working with Coin<T> directly
    // ============================================
    
    public fun split_coin(
        coin: Coin<MyToken>,
        amount: u64,
    ): (Coin<MyToken>, Coin<MyToken>) {
        let extracted = coin::extract(&mut coin, amount);
        (coin, extracted)
    }
    
    public fun merge_coins(
        base: &mut Coin<MyToken>,
        other: Coin<MyToken>,
    ) {
        coin::merge(base, other);
    }
    
    public fun coin_value(coin: &Coin<MyToken>): u64 {
        coin::value(coin)
    }
    
    // ============================================
    // Coin in protocol functions
    // ============================================
    
    struct Protocol has key {
        treasury: Coin<MyToken>,
        fee_bps: u64,
    }
    
    public entry fun initialize_protocol(
        admin: &signer,
        fee_bps: u64,
    ) {
        move_to(admin, Protocol {
            treasury: coin::zero<MyToken>(),  // empty coin
            fee_bps,
        });
    }
    
    public entry fun protocol_swap(
        user: &signer,
        protocol_addr: address,
        amount: u64,
    ) acquires Protocol {
        let protocol = borrow_global_mut<Protocol>(protocol_addr);
        
        // Collect fee
        let fee = amount * protocol.fee_bps / 10_000;
        let user_coins = coin::withdraw<MyToken>(user, fee);
        coin::merge(&mut protocol.treasury, user_coins);
    }
}
```

---

## aptos_framework::aptos_coin

```move
module learning::apt_usage {
    use std::signer;
    use aptos_framework::coin;
    use aptos_framework::aptos_coin::AptosCoin;
    
    // ============================================
    // Using APT (native token)
    // ============================================
    
    // AptosCoin is the native token of Aptos
    // Coin<AptosCoin> = APT coins
    // 1 APT = 10^8 octas (smallest unit)
    
    const APT_DECIMALS: u64 = 100_000_000;  // 1e8 octas = 1 APT
    
    public fun apt_balance(addr: address): u64 {
        coin::balance<AptosCoin>(addr)
    }
    
    public fun apt_balance_in_apt(addr: address): u64 {
        coin::balance<AptosCoin>(addr) / APT_DECIMALS
    }
    
    // Transfer APT
    public entry fun transfer_apt(
        from: &signer,
        to: address,
        amount_in_apt: u64,  // amount in APT (not octas)
    ) {
        let octas = amount_in_apt * APT_DECIMALS;
        coin::transfer<AptosCoin>(from, to, octas);
    }
    
    // ============================================
    // Fee collection in APT
    // ============================================
    
    struct FeeVault has key {
        collected_fees: u64,  // in octas
    }
    
    public entry fun collect_fee_apt(
        user: &signer,
        vault_addr: address,
        fee_octas: u64,
    ) acquires FeeVault {
        // Transfer APT from user to vault
        coin::transfer<AptosCoin>(user, vault_addr, fee_octas);
        
        let vault = borrow_global_mut<FeeVault>(vault_addr);
        vault.collected_fees = vault.collected_fees + fee_octas;
    }
}
```

---

## ตัวอย่างโปรแกรม: Subscription Service

```move
module learning::subscription_service {
    use std::signer;
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::aptos_coin::AptosCoin;
    use aptos_framework::timestamp;
    use aptos_framework::event;
    
    // ============================================
    // Types
    // ============================================
    
    struct SubscriptionPlan has copy, drop, store {
        id: u8,
        name: vector<u8>,
        price_per_month: u64,  // in APT octas
        max_users: u64,
        features: u64,  // bitmask
    }
    
    struct UserSubscription has key {
        plan_id: u8,
        expires_at: u64,
        auto_renew: bool,
        owner: address,
    }
    
    struct ServiceConfig has key {
        admin: address,
        treasury: address,
        plans: vector<SubscriptionPlan>,
        total_subscribers: u64,
        is_active: bool,
    }
    
    // ============================================
    // Events
    // ============================================
    
    #[event]
    struct SubscribedEvent has drop, store {
        user: address,
        plan_id: u8,
        expires_at: u64,
        amount_paid: u64,
    }
    
    #[event]
    struct RenewedEvent has drop, store {
        user: address,
        plan_id: u8,
        new_expires_at: u64,
        amount_paid: u64,
    }
    
    #[event]
    struct CancelledEvent has drop, store {
        user: address,
        plan_id: u8,
        refund_amount: u64,
    }
    
    // ============================================
    // Error codes
    // ============================================
    
    const E_NOT_ACTIVE: u64 = 1;
    const E_PLAN_NOT_FOUND: u64 = 2;
    const E_ALREADY_SUBSCRIBED: u64 = 3;
    const E_NOT_SUBSCRIBED: u64 = 4;
    const E_EXPIRED: u64 = 5;
    const E_NOT_ADMIN: u64 = 6;
    const E_INSUFFICIENT_PAYMENT: u64 = 7;
    
    // ============================================
    // Constants
    // ============================================
    
    const SECONDS_PER_MONTH: u64 = 30 * 24 * 3600;
    const APT_DECIMALS: u64 = 100_000_000;
    
    // ============================================
    // Initialize
    // ============================================
    
    public entry fun initialize(
        admin: &signer,
        treasury: address,
    ) {
        let addr = signer::address_of(admin);
        
        // Create default plans
        let plans = vector[
            SubscriptionPlan {
                id: 1,
                name: b"Basic",
                price_per_month: APT_DECIMALS / 2,  // 0.5 APT/month
                max_users: 5,
                features: 0b001,  // feature bit 0
            },
            SubscriptionPlan {
                id: 2,
                name: b"Professional",
                price_per_month: APT_DECIMALS * 2,  // 2 APT/month
                max_users: 50,
                features: 0b011,  // features 0 and 1
            },
            SubscriptionPlan {
                id: 3,
                name: b"Enterprise",
                price_per_month: APT_DECIMALS * 10,  // 10 APT/month
                max_users: 1000,
                features: 0b111,  // all features
            },
        ];
        
        move_to(admin, ServiceConfig {
            admin: addr,
            treasury,
            plans,
            total_subscribers: 0,
            is_active: true,
        });
    }
    
    // ============================================
    // Subscribe
    // ============================================
    
    public entry fun subscribe(
        user: &signer,
        service_addr: address,
        plan_id: u8,
        months: u64,
    ) acquires ServiceConfig, UserSubscription {
        let user_addr = signer::address_of(user);
        let config = borrow_global_mut<ServiceConfig>(service_addr);
        
        assert!(config.is_active, E_NOT_ACTIVE);
        assert!(!exists<UserSubscription>(user_addr), E_ALREADY_SUBSCRIBED);
        
        // Find plan
        let plan = find_plan(&config.plans, plan_id);
        assert!(plan.id != 0, E_PLAN_NOT_FOUND);
        
        let total_cost = plan.price_per_month * months;
        let now = timestamp::now_seconds();
        let expires_at = now + months * SECONDS_PER_MONTH;
        
        // Collect payment
        let payment = coin::withdraw<AptosCoin>(user, total_cost);
        coin::deposit<AptosCoin>(config.treasury, payment);
        
        // Create subscription
        move_to(user, UserSubscription {
            plan_id,
            expires_at,
            auto_renew: true,
            owner: user_addr,
        });
        
        config.total_subscribers = config.total_subscribers + 1;
        
        event::emit(SubscribedEvent {
            user: user_addr,
            plan_id,
            expires_at,
            amount_paid: total_cost,
        });
    }
    
    // ============================================
    // Renew Subscription
    // ============================================
    
    public entry fun renew(
        user: &signer,
        service_addr: address,
        months: u64,
    ) acquires ServiceConfig, UserSubscription {
        let user_addr = signer::address_of(user);
        let config = borrow_global<ServiceConfig>(service_addr);
        
        assert!(config.is_active, E_NOT_ACTIVE);
        assert!(exists<UserSubscription>(user_addr), E_NOT_SUBSCRIBED);
        
        let sub = borrow_global_mut<UserSubscription>(user_addr);
        let plan = find_plan(&config.plans, sub.plan_id);
        
        let total_cost = plan.price_per_month * months;
        
        // Collect payment
        let payment = coin::withdraw<AptosCoin>(user, total_cost);
        coin::deposit<AptosCoin>(config.treasury, payment);
        
        // Extend subscription
        let now = timestamp::now_seconds();
        let base_time = if (sub.expires_at > now) { sub.expires_at } else { now };
        sub.expires_at = base_time + months * SECONDS_PER_MONTH;
        
        event::emit(RenewedEvent {
            user: user_addr,
            plan_id: sub.plan_id,
            new_expires_at: sub.expires_at,
            amount_paid: total_cost,
        });
    }
    
    // ============================================
    // Cancel Subscription
    // ============================================
    
    public entry fun cancel(
        user: &signer,
        service_addr: address,
    ) acquires ServiceConfig, UserSubscription {
        let user_addr = signer::address_of(user);
        let config = borrow_global_mut<ServiceConfig>(service_addr);
        
        assert!(exists<UserSubscription>(user_addr), E_NOT_SUBSCRIBED);
        
        let sub = borrow_global<UserSubscription>(user_addr);
        let plan_id = sub.plan_id;
        
        // Calculate refund (remaining time)
        let now = timestamp::now_seconds();
        let refund = if (sub.expires_at > now) {
            let plan = find_plan(&config.plans, sub.plan_id);
            let remaining_seconds = sub.expires_at - now;
            let remaining_months = remaining_seconds / SECONDS_PER_MONTH;
            remaining_months * plan.price_per_month
        } else {
            0
        };
        
        // Remove subscription
        let UserSubscription { plan_id: _, expires_at: _, auto_renew: _, owner: _ } = 
            move_from<UserSubscription>(user_addr);
        
        config.total_subscribers = config.total_subscribers - 1;
        
        // Issue refund if any
        // Note: in production this would use a treasury withdrawal mechanism
        
        event::emit(CancelledEvent {
            user: user_addr,
            plan_id,
            refund_amount: refund,
        });
    }
    
    // ============================================
    // View Functions
    // ============================================
    
    #[view]
    public fun is_subscribed(user_addr: address): bool {
        if (!exists<UserSubscription>(user_addr)) return false;
        true
    }
    
    #[view]
    public fun is_subscription_active(user_addr: address): bool acquires UserSubscription {
        if (!exists<UserSubscription>(user_addr)) return false;
        let sub = borrow_global<UserSubscription>(user_addr);
        let now = timestamp::now_seconds();
        now < sub.expires_at
    }
    
    #[view]
    public fun subscription_details(user_addr: address): (u8, u64, bool) acquires UserSubscription {
        assert!(exists<UserSubscription>(user_addr), E_NOT_SUBSCRIBED);
        let sub = borrow_global<UserSubscription>(user_addr);
        (sub.plan_id, sub.expires_at, sub.auto_renew)
    }
    
    #[view]
    public fun has_feature(user_addr: address, feature_bit: u64): bool acquires UserSubscription, ServiceConfig {
        if (!is_subscription_active(user_addr)) return false;
        
        // This simplified version doesn't take service_addr - in production it would
        false  // simplified
    }
    
    #[view]
    public fun total_subscribers(service_addr: address): u64 acquires ServiceConfig {
        borrow_global<ServiceConfig>(service_addr).total_subscribers
    }
    
    // ============================================
    // Helpers
    // ============================================
    
    fun find_plan(plans: &vector<SubscriptionPlan>, plan_id: u8): SubscriptionPlan {
        use std::vector;
        
        let i = 0u64;
        while (i < vector::length(plans)) {
            let plan = *vector::borrow(plans, i);
            if (plan.id == plan_id) return plan;
            i = i + 1;
        };
        
        // Return empty plan (id == 0 means not found)
        SubscriptionPlan {
            id: 0, name: b"", price_per_month: 0, max_users: 0, features: 0
        }
    }
}
```

---

## สรุป Aptos Framework

| Module | Key Types/Functions |
|--------|-------------------|
| `account` | `SignerCapability`, `create_resource_account`, `new_event_handle` |
| `timestamp` | `now_seconds()`, `now_microseconds()` |
| `coin` | `Coin<T>`, `mint`, `burn`, `transfer`, `withdraw`, `deposit` |
| `aptos_coin` | `AptosCoin` type |
| `event` | `emit_event`, `emit` (new style) |

---

**ต่อไป**: [Part 22 - Fungible Assets →](part-22-fungible-assets.md)
