# Part 08: Resources และ Linear Types

## สารบัญ
- [ทำความเข้าใจ Resources](#ทำความเข้าใจ-resources)
- [Linear Type System](#linear-type-system)
- [Abilities ทั้ง 4](#abilities-ทั้ง-4)
- [Resource Lifecycle](#resource-lifecycle)
- [Global Storage Operations](#global-storage-operations)
- [Resource Patterns](#resource-patterns)
- [ตัวอย่างโปรแกรม: Digital Asset](#ตัวอย่างโปรแกรม-digital-asset)

---

## ทำความเข้าใจ Resources

**Resource** คือ struct ที่มี `key` หรือ `store` ability และไม่มี `copy` ability

```move
module learning::resource_basics {
    
    // ❌ นี่ไม่ใช่ Resource (มี copy ability)
    struct SimpleData has copy, drop {
        value: u64,
    }
    
    // ✅ นี่คือ Resource (ไม่มี copy, มี key)
    struct Token has key {
        id: u64,
        amount: u64,
    }
    
    // ✅ Resource ที่ซับซ้อน
    struct Vault has key {
        balance: u64,
        locked_until: u64,
        owner: address,
    }
    
    // Properties ของ Resources:
    // 1. ไม่สามารถ copy โดยไม่ตั้งใจ
    // 2. ไม่สามารถ drop โดยไม่ตั้งใจ
    // 3. ต้อง "ใช้" อย่างถูกต้อง (move_to, return, หรือ destructure)
    // 4. เก็บได้ใน global storage (ถ้ามี key ability)
    
    public fun demonstrate_resource_rules() {
        let token = Token { id: 1, amount: 1000 };
        
        // ❌ ไม่สามารถ copy
        // let token2 = token;  // ERROR: token ถูก move ไม่ใช่ copy
        
        // ❌ ไม่สามารถ drop โดยไม่ตั้งใจ
        // { let _t = token; }  // ERROR: token ถูก drop โดยไม่ได้ destroy
        
        // ✅ ต้อง consume อย่างถูกต้อง
        let Token { id: _, amount: _ } = token;  // Destructure = destroy
        
        // หรือ move_to (ถ้ามี signer)
        // move_to(account, token);
    }
}
```

---

## Linear Type System

Linear types บังคับให้ใช้ value **ครั้งเดียวเท่านั้น** - ไม่ copy, ไม่ drop โดยไม่ตั้งใจ

```move
module learning::linear_types {
    
    // ทำไม Linear Types ถึงสำคัญสำหรับ Finance?
    
    // ปัญหาใน non-linear systems (เช่น Solidity):
    // 1. Double-spending: สร้าง token แล้ว copy ไปจ่ายหลายที่
    // 2. Token loss: token ถูก drop โดยไม่ตั้งใจ
    
    // ใน Move: Compiler ป้องกันสิ่งเหล่านี้
    
    struct Coin has key {
        value: u64,
    }
    
    // ตัวอย่างที่ SECURE: ไม่สามารถ double-spend
    public fun split_coin(coin: Coin, amount: u64): (Coin, Coin) {
        assert!(coin.value >= amount, 1);
        let remainder = coin.value - amount;
        
        // สร้าง 2 coins จาก 1 coin (total ยังคงเท่าเดิม)
        let coin1 = Coin { value: amount };
        let coin2 = Coin { value: remainder };
        
        // coin เดิมถูก destructure (destroyed)
        let Coin { value: _ } = coin;
        
        (coin1, coin2)  // return 2 coins ใหม่
    }
    
    // ✅ ปลอดภัย: total value preserved
    public fun merge_coins(coin1: Coin, coin2: Coin): Coin {
        let total = coin1.value + coin2.value;
        let Coin { value: _ } = coin1;
        let Coin { value: _ } = coin2;
        Coin { value: total }
    }
    
    // หลักฐานว่า Linear Types ป้องกัน bugs
    #[test]
    fun test_conservation_of_value() {
        let original_value = 1000u64;
        let coin = Coin { value: original_value };
        
        // Split
        let (coin_a, coin_b) = split_coin(coin, 300);
        assert!(coin_a.value == 300, 0);
        assert!(coin_b.value == 700, 1);
        assert!(coin_a.value + coin_b.value == original_value, 2);
        
        // Merge
        let merged = merge_coins(coin_a, coin_b);
        assert!(merged.value == original_value, 3);
        
        // Clean up
        let Coin { value: _ } = merged;
    }
}
```

---

## Abilities ทั้ง 4

```move
module learning::abilities {
    
    // ============================================
    // COPY - สามารถ copy ค่าได้
    // ============================================
    struct Copyable has copy, drop {
        value: u64,
    }
    
    public fun demo_copy() {
        let a = Copyable { value: 10 };
        let b = a;  // ✅ copy เพราะมี copy ability
        let c = a;  // ✅ a ยังใช้ได้หลัง copy
        assert!(a.value == b.value, 0);
        assert!(a.value == c.value, 1);
    }
    
    // ============================================
    // DROP - สามารถ discard/drop ค่าได้
    // ============================================
    struct Droppable has drop {
        value: u64,
    }
    
    public fun demo_drop() {
        let d = Droppable { value: 42 };
        // d ถูก drop อัตโนมัติตอนออกจาก scope
        // ไม่ต้อง explicit drop
    }
    
    // ถ้าไม่มี drop ต้อง explicit consume
    struct MustConsume {  // ไม่มี ability ใดเลย
        value: u64,
    }
    
    public fun demo_must_consume() {
        let mc = MustConsume { value: 99 };
        // mc ต้องถูก consume ก่อนออกจาก scope
        let MustConsume { value: _ } = mc;  // explicit consume
    }
    
    // ============================================
    // STORE - สามารถเก็บใน struct อื่นหรือ global storage
    // ============================================
    struct Storable has store {
        data: u64,
    }
    
    struct Container has key {
        inner: Storable,  // ✅ Storable สามารถเก็บใน Container ได้
    }
    
    // struct WithoutStore { data: u64 }  // ไม่มี store
    // struct BadContainer has key {
    //     inner: WithoutStore,  // ❌ ERROR: WithoutStore ไม่มี store ability
    // }
    
    // ============================================
    // KEY - สามารถเป็น top-level resource ใน global storage
    // ============================================
    struct KeyResource has key {
        data: u64,
    }
    
    struct NonKeyResource has store {
        data: u64,
    }
    
    public fun demo_key(account: &signer) {
        // KeyResource สามารถ move_to ได้
        move_to(account, KeyResource { data: 42 });
        
        // NonKeyResource ไม่สามารถ move_to ได้โดยตรง
        // move_to(account, NonKeyResource { data: 42 });  // ❌ ERROR
    }
    
    // ============================================
    // Combination ต่างๆ
    // ============================================
    
    // Primitive types: copy + drop
    // integers, bool - คัดลอกและทิ้งได้อัตโนมัติ
    
    // Data types: copy + drop + store  
    struct DataValue has copy, drop, store {
        value: u64,
        flag: bool,
    }
    
    // Resource in storage: key + store
    struct StoredResource has key, store {
        data: vector<u8>,
    }
    
    // Asset (no copy, no drop): 
    struct Asset has key {  // ไม่มี copy, ไม่มี drop
        id: u64,
        owner: address,
    }
    
    // Nested abilities
    struct Inner has store, copy, drop {
        value: u64,
    }
    
    struct Outer has key, store {
        // inner ต้องมี store ability เพื่อเก็บใน Outer
        inner: Inner,
        data: u64,
    }
}
```

### Ability Decision Tree

```
ต้องการ copy/clone ค่า?
    ├── YES → เพิ่ม `copy`
    └── NO  → ไม่ใส่ `copy` (Resource behavior)

ค่าสามารถถูกทิ้ง/ignore ได้?
    ├── YES → เพิ่ม `drop`
    └── NO  → ไม่ใส่ `drop` (ต้องถูก consumed)

ต้องการเก็บใน global storage หรือ nested struct?
    ├── YES → เพิ่ม `store`
    └── NO  → ไม่จำเป็น

ต้องการเป็น top-level resource ใน global storage?
    ├── YES → เพิ่ม `key`
    └── NO  → ไม่จำเป็น

Common patterns:
    Primitive-like:    has copy, drop
    Data object:       has copy, drop, store
    Nested data:       has store
    Nested resource:   has store (no copy, no drop)
    Global resource:   has key
    Global + nested:   has key, store
```

---

## Resource Lifecycle

```move
module learning::resource_lifecycle {
    use std::signer;
    
    struct Escrow has key {
        amount: u64,
        beneficiary: address,
        release_time: u64,
    }
    
    // Phase 1: CREATION
    public fun create_escrow(
        depositor: &signer,
        beneficiary: address,
        amount: u64,
        release_time: u64,
    ) {
        // สร้าง resource ใหม่
        let escrow = Escrow { amount, beneficiary, release_time };
        
        // Phase 2: STORAGE (move to global storage)
        move_to(depositor, escrow);
        // ตอนนี้ escrow อยู่ใน global storage ของ depositor's address
    }
    
    // Phase 3: ACCESS (borrow from global storage)
    #[view]
    public fun get_amount(addr: address): u64 acquires Escrow {
        // Immutable borrow
        borrow_global<Escrow>(addr).amount
    }
    
    // Phase 4: MODIFICATION
    public entry fun update_beneficiary(
        account: &signer,
        new_beneficiary: address,
    ) acquires Escrow {
        let addr = signer::address_of(account);
        // Mutable borrow
        let escrow = borrow_global_mut<Escrow>(addr);
        escrow.beneficiary = new_beneficiary;
    }
    
    // Phase 5: REMOVAL (move from global storage)
    public entry fun release_escrow(
        account: &signer,
        current_time: u64,
    ) acquires Escrow {
        let addr = signer::address_of(account);
        let escrow = borrow_global<Escrow>(addr);
        
        // ตรวจสอบเงื่อนไข
        assert!(current_time >= escrow.release_time, 1);
        
        // Phase 6: CONSUMPTION (move_from + destructure)
        let Escrow { amount, beneficiary, release_time: _ } = move_from<Escrow>(addr);
        
        // ทำงานกับ amount และ beneficiary
        // (ใน production: transfer amount ไปยัง beneficiary)
        let _ = amount;
        let _ = beneficiary;
    }
    
    // ============================================
    // Global Storage Operations Summary
    // ============================================
    
    // exists<T>(addr) → bool
    // - ตรวจสอบว่ามี resource T ที่ addr หรือไม่
    
    // borrow_global<T>(addr) → &T
    // - อ่านค่า resource (immutable reference)
    // - function ต้องประกาศ `acquires T`
    
    // borrow_global_mut<T>(addr) → &mut T
    // - แก้ไข resource (mutable reference)
    // - function ต้องประกาศ `acquires T`
    
    // move_to(signer, resource)
    // - เก็บ resource ลงใน signer's address
    // - resource ต้องมี key ability
    
    // move_from<T>(addr) → T
    // - เอา resource ออกจาก global storage
    // - function ต้องประกาศ `acquires T`
}
```

---

## Resource Patterns

### Pattern 1: Capability Pattern

```move
module learning::capability_pattern {
    use std::signer;
    
    // Capability = permission token
    struct MintCapability has key {
        max_per_tx: u64,
    }
    
    struct BurnCapability has key { }
    
    struct AdminCapability has key {
        admin: address,
    }
    
    // ให้ capabilities ตอน initialize
    public entry fun initialize(admin: &signer) {
        let addr = signer::address_of(admin);
        
        move_to(admin, AdminCapability { admin: addr });
        move_to(admin, MintCapability { max_per_tx: 1_000_000 });
        move_to(admin, BurnCapability {});
    }
    
    // ใช้ capability เป็น authorization
    public entry fun mint(
        minter: &signer,
        amount: u64,
        recipient: address,
    ) acquires MintCapability {
        let minter_addr = signer::address_of(minter);
        
        // ตรวจสอบว่ามี MintCapability
        assert!(exists<MintCapability>(minter_addr), 1);
        
        let cap = borrow_global<MintCapability>(minter_addr);
        assert!(amount <= cap.max_per_tx, 2);
        
        // ทำการ mint
        // ...
    }
    
    // Transfer capability
    public entry fun grant_mint_capability(
        granter: &signer,
        grantee: address,
        max_per_tx: u64,
    ) acquires AdminCapability, MintCapability {
        let granter_addr = signer::address_of(granter);
        
        // ต้องมี AdminCapability ถึง grant ได้
        assert!(exists<AdminCapability>(granter_addr), 1);
        let admin_cap = borrow_global<AdminCapability>(granter_addr);
        assert!(admin_cap.admin == granter_addr, 2);
        
        // Check ไม่ได้ grant ให้ตัวเอง
        assert!(grantee != granter_addr, 3);
        
        // ส่ง capability ไปให้ grantee
        // (ในการ implement จริง ต้องใช้ table หรือ resource account)
    }
}
```

### Pattern 2: Hot Potato Pattern

```move
module learning::hot_potato {
    
    // Hot Potato = struct ที่ไม่มี drop ability
    // ต้องถูก consumed ก่อนออกจาก call frame
    struct Receipt {
        amount: u64,
        payer: address,
    }
    
    // สร้าง receipt แล้วต้องถูก process ภายใน transaction
    public fun begin_payment(
        payer: address,
        amount: u64,
    ): Receipt {
        Receipt { amount, payer }
    }
    
    // ต้อง consume receipt ด้วยการ complete payment
    public fun complete_payment(
        receipt: Receipt,
    ): bool {
        let Receipt { amount, payer } = receipt;
        // ตรวจสอบว่า payment ถูกต้อง
        amount > 0  // simplified
    }
    
    // ตัวอย่างการใช้งาน Flash Loan
    struct FlashLoanReceipt {
        amount: u64,
        fee: u64,
        pool: address,
    }
    
    // ยืม tokens (ต้อง return ภายใน transaction เดียวกัน)
    public fun flash_loan(
        pool_addr: address,
        amount: u64,
    ): (u64, FlashLoanReceipt) {
        let fee = amount / 1000;  // 0.1% fee
        let receipt = FlashLoanReceipt { amount, fee, pool: pool_addr };
        (amount, receipt)  // return tokens + receipt
    }
    
    // คืน tokens (พร้อม fee) และ consume receipt
    public fun repay_flash_loan(
        receipt: FlashLoanReceipt,
        repay_amount: u64,
    ) {
        let FlashLoanReceipt { amount, fee, pool: _ } = receipt;
        assert!(repay_amount >= amount + fee, 1);
        // คืน tokens ไปยัง pool
    }
}
```

### Pattern 3: Resource Account Pattern (Aptos)

```move
module learning::resource_account_pattern {
    use std::signer;
    use aptos_framework::account;
    
    // Resource Account คือ account ที่ไม่มี private key
    // ใช้สำหรับ autonomous smart contracts
    
    struct ProtocolState has key {
        total_liquidity: u64,
        fee_collector: address,
    }
    
    // Initialize protocol ใน resource account
    public entry fun initialize_protocol(
        admin: &signer,
        seed: vector<u8>,
    ) {
        // สร้าง resource account
        let (resource_signer, cap) = account::create_resource_account(admin, seed);
        
        // เก็บ state ใน resource account
        move_to(&resource_signer, ProtocolState {
            total_liquidity: 0,
            fee_collector: signer::address_of(admin),
        });
        
        // เก็บ signer capability สำหรับใช้ภายหลัง
        // (ปกติจะเก็บใน admin account)
    }
}
```

---

## ตัวอย่างโปรแกรม: Digital Asset System

```move
module learning::digital_asset {
    use std::signer;
    use std::string::{Self, String};
    use std::vector;
    
    // ============================================
    // Types
    // ============================================
    
    struct AssetConfig has key {
        name: String,
        symbol: String,
        decimals: u8,
        max_supply: u64,
        total_supply: u64,
        is_paused: bool,
        admin: address,
        minters: vector<address>,
    }
    
    struct AssetBalance has key {
        amount: u64,
    }
    
    struct AllowanceRecord has key {
        // owner → spender → amount
        // ใน production ใช้ Table แต่นี่ใช้ simplified version
        spender: address,
        amount: u64,
    }
    
    // ============================================
    // Constants
    // ============================================
    
    const MAX_DECIMALS: u8 = 18;
    const E_NOT_ADMIN: u64 = 100;
    const E_NOT_MINTER: u64 = 101;
    const E_NOT_AUTHORIZED: u64 = 102;
    const E_PAUSED: u64 = 200;
    const E_ZERO_AMOUNT: u64 = 201;
    const E_INSUFFICIENT_BALANCE: u64 = 202;
    const E_EXCEEDS_ALLOWANCE: u64 = 203;
    const E_EXCEEDS_MAX_SUPPLY: u64 = 204;
    const E_ALREADY_INITIALIZED: u64 = 300;
    const E_NOT_INITIALIZED: u64 = 301;
    const E_INVALID_DECIMALS: u64 = 302;
    
    // ============================================
    // Initialization
    // ============================================
    
    public entry fun initialize(
        admin: &signer,
        name: vector<u8>,
        symbol: vector<u8>,
        decimals: u8,
        max_supply: u64,
    ) {
        let addr = signer::address_of(admin);
        assert!(!exists<AssetConfig>(addr), E_ALREADY_INITIALIZED);
        assert!(decimals <= MAX_DECIMALS, E_INVALID_DECIMALS);
        
        move_to(admin, AssetConfig {
            name: string::utf8(name),
            symbol: string::utf8(symbol),
            decimals,
            max_supply,
            total_supply: 0,
            is_paused: false,
            admin: addr,
            minters: vector[addr],  // admin is initial minter
        });
        
        // Give admin initial balance record
        move_to(admin, AssetBalance { amount: 0 });
    }
    
    // ============================================
    // Admin Functions
    // ============================================
    
    public entry fun pause(admin: &signer, config_addr: address) acquires AssetConfig {
        let addr = signer::address_of(admin);
        let config = borrow_global_mut<AssetConfig>(config_addr);
        assert!(config.admin == addr, E_NOT_ADMIN);
        config.is_paused = true;
    }
    
    public entry fun unpause(admin: &signer, config_addr: address) acquires AssetConfig {
        let addr = signer::address_of(admin);
        let config = borrow_global_mut<AssetConfig>(config_addr);
        assert!(config.admin == addr, E_NOT_ADMIN);
        config.is_paused = false;
    }
    
    public entry fun add_minter(
        admin: &signer,
        config_addr: address,
        new_minter: address,
    ) acquires AssetConfig {
        let addr = signer::address_of(admin);
        let config = borrow_global_mut<AssetConfig>(config_addr);
        assert!(config.admin == addr, E_NOT_ADMIN);
        assert!(!vector::contains(&config.minters, &new_minter), 0);
        vector::push_back(&mut config.minters, new_minter);
    }
    
    public entry fun remove_minter(
        admin: &signer,
        config_addr: address,
        minter: address,
    ) acquires AssetConfig {
        let addr = signer::address_of(admin);
        let config = borrow_global_mut<AssetConfig>(config_addr);
        assert!(config.admin == addr, E_NOT_ADMIN);
        
        let (found, index) = vector::index_of(&config.minters, &minter);
        assert!(found, 0);
        vector::remove(&mut config.minters, index);
    }
    
    // ============================================
    // Token Operations
    // ============================================
    
    public entry fun mint(
        minter: &signer,
        config_addr: address,
        recipient: address,
        amount: u64,
    ) acquires AssetConfig, AssetBalance {
        assert!(amount > 0, E_ZERO_AMOUNT);
        
        let minter_addr = signer::address_of(minter);
        let config = borrow_global_mut<AssetConfig>(config_addr);
        
        // Check pause
        assert!(!config.is_paused, E_PAUSED);
        
        // Check minter authorization
        assert!(vector::contains(&config.minters, &minter_addr), E_NOT_MINTER);
        
        // Check supply cap
        let new_supply = config.total_supply + amount;
        assert!(new_supply <= config.max_supply, E_EXCEEDS_MAX_SUPPLY);
        
        config.total_supply = new_supply;
        
        // Credit recipient
        credit_balance(recipient, amount);
    }
    
    public entry fun burn(
        burner: &signer,
        config_addr: address,
        amount: u64,
    ) acquires AssetConfig, AssetBalance {
        assert!(amount > 0, E_ZERO_AMOUNT);
        
        let burner_addr = signer::address_of(burner);
        let config = borrow_global_mut<AssetConfig>(config_addr);
        
        assert!(!config.is_paused, E_PAUSED);
        
        // Debit burner
        debit_balance(burner_addr, amount);
        
        config.total_supply = config.total_supply - amount;
    }
    
    public entry fun transfer(
        from: &signer,
        to: address,
        config_addr: address,
        amount: u64,
    ) acquires AssetConfig, AssetBalance {
        assert!(amount > 0, E_ZERO_AMOUNT);
        
        let from_addr = signer::address_of(from);
        assert!(from_addr != to, 0);  // no self transfer
        
        let config = borrow_global<AssetConfig>(config_addr);
        assert!(!config.is_paused, E_PAUSED);
        
        debit_balance(from_addr, amount);
        credit_balance(to, amount);
    }
    
    // ============================================
    // Internal Helpers
    // ============================================
    
    fun credit_balance(addr: address, amount: u64) acquires AssetBalance {
        if (exists<AssetBalance>(addr)) {
            let balance = borrow_global_mut<AssetBalance>(addr);
            balance.amount = balance.amount + amount;
        }
        // Note: ใน production ต้องจัดการกรณีที่ยังไม่มี balance record
    }
    
    fun debit_balance(addr: address, amount: u64) acquires AssetBalance {
        assert!(exists<AssetBalance>(addr), E_INSUFFICIENT_BALANCE);
        let balance = borrow_global_mut<AssetBalance>(addr);
        assert!(balance.amount >= amount, E_INSUFFICIENT_BALANCE);
        balance.amount = balance.amount - amount;
    }
    
    // ============================================
    // View Functions
    // ============================================
    
    #[view]
    public fun balance_of(addr: address): u64 acquires AssetBalance {
        if (exists<AssetBalance>(addr)) {
            borrow_global<AssetBalance>(addr).amount
        } else {
            0
        }
    }
    
    #[view]
    public fun total_supply(config_addr: address): u64 acquires AssetConfig {
        borrow_global<AssetConfig>(config_addr).total_supply
    }
    
    #[view]
    public fun max_supply(config_addr: address): u64 acquires AssetConfig {
        borrow_global<AssetConfig>(config_addr).max_supply
    }
    
    #[view]
    public fun is_paused(config_addr: address): bool acquires AssetConfig {
        borrow_global<AssetConfig>(config_addr).is_paused
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test(admin = @0x1, user = @0x2)]
    public fun test_full_lifecycle(
        admin: &signer,
        user: &signer,
    ) acquires AssetConfig, AssetBalance {
        let admin_addr = signer::address_of(admin);
        let user_addr = signer::address_of(user);
        
        // Initialize
        initialize(admin, b"Test Token", b"TST", 8, 1_000_000_000);
        
        // Setup user balance record
        move_to(user, AssetBalance { amount: 0 });
        
        // Mint
        mint(admin, admin_addr, admin_addr, 100_000_000);
        assert!(balance_of(admin_addr) == 100_000_000, 0);
        assert!(total_supply(admin_addr) == 100_000_000, 1);
        
        // Transfer
        transfer(admin, user_addr, admin_addr, 10_000_000);
        assert!(balance_of(admin_addr) == 90_000_000, 2);
        assert!(balance_of(user_addr) == 10_000_000, 3);
        
        // Burn
        burn(admin, admin_addr, 5_000_000);
        assert!(balance_of(admin_addr) == 85_000_000, 4);
        assert!(total_supply(admin_addr) == 95_000_000, 5);
        
        // Pause
        pause(admin, admin_addr);
        assert!(is_paused(admin_addr) == true, 6);
        
        // Unpause
        unpause(admin, admin_addr);
        assert!(is_paused(admin_addr) == false, 7);
    }
    
    #[test(admin = @0x1)]
    #[expected_failure(abort_code = E_EXCEEDS_MAX_SUPPLY)]
    public fun test_exceeds_max_supply(admin: &signer) acquires AssetConfig, AssetBalance {
        let admin_addr = signer::address_of(admin);
        initialize(admin, b"Test", b"TST", 8, 100);
        mint(admin, admin_addr, admin_addr, 101);  // ควร fail
    }
}
```

---

## สรุป

| แนวคิด | สิ่งที่เรียนรู้ |
|-------|--------------|
| Resources | struct ที่ไม่มี copy ability |
| Linear Types | ใช้ครั้งเดียว, ไม่ copy, ไม่ drop |
| Abilities | copy, drop, store, key |
| Lifecycle | create → store → access → modify → remove → consume |
| Global Ops | move_to, borrow_global, move_from, exists |
| Patterns | Capability, Hot Potato, Resource Account |

---

## แบบฝึกหัด

### Exercise 8.1: Abilities
เขียน structs ต่อไปนี้พร้อม abilities ที่เหมาะสม:
1. `Config` - settings ที่สามารถ copy ได้
2. `Ticket` - ตั๋วที่ใช้ครั้งเดียว
3. `Badge` - badge ที่เก็บใน wallet
4. `Escrow` - เงินที่ล็อคไว้

### Exercise 8.2: Resource Lifecycle
สร้าง `TimeLock` resource ที่:
1. Lock tokens จนถึงเวลาที่กำหนด
2. Unlock เมื่อถึงเวลา
3. ส่งต่อ lock ให้คนอื่นได้
4. Cancel lock ได้ (ก่อนหมดเวลา)

### Exercise 8.3: Hot Potato
สร้าง FlashLoan protocol:
1. Borrow function ที่ return (amount, Receipt)
2. Repay function ที่ consume Receipt
3. ถ้าไม่ repay ใน transaction เดียวกัน จะ fail อัตโนมัติ

---

**ก่อนหน้า**: [Part 07 - Modules](part-07-modules.md)  
**ต่อไป**: [Part 09 - References และ Borrowing →](part-09-references.md)
