# Part 11: Abilities (copy, drop, store, key)

## สารบัญ
- [Overview ของ Abilities](#overview-ของ-abilities)
- [Copy Ability](#copy-ability)
- [Drop Ability](#drop-ability)
- [Store Ability](#store-ability)
- [Key Ability](#key-ability)
- [Ability Constraints](#ability-constraints)
- [Ability Checking](#ability-checking)
- [ตัวอย่างโปรแกรม: Token Protocol](#ตัวอย่างโปรแกรม-token-protocol)

---

## Overview ของ Abilities

Abilities คือ "permissions" ที่กำหนดว่า type ทำอะไรได้บ้าง

```
4 Abilities ใน Move:

copy  → สามารถ copy ค่าได้
drop  → สามารถ discard ค่าได้ (end of scope)
store → สามารถเก็บใน struct อื่นหรือ global storage ได้
key   → สามารถเป็น top-level value ใน global storage ได้

กฎสำคัญ:
- struct มี ability ได้ก็ต่อเมื่อ fields ทั้งหมดมี ability นั้น
- key ต้องการ store ก่อน
- Primitives มี copy + drop โดย default
```

---

## Copy Ability

```move
module learning::copy_ability {
    
    // ============================================
    // Types ที่มี copy ability
    // ============================================
    
    // Primitives - ทุก primitive มี copy โดย default
    public fun primitive_copy_demo() {
        let x: u64 = 42;
        let y = x;      // copy
        let z = x;      // copy อีกครั้ง - x ยังใช้ได้
        assert!(x == y && y == z, 0);
    }
    
    // Struct ที่มี copy
    struct Price has copy, drop, store {
        value: u64,
        decimals: u8,
        symbol: vector<u8>,
    }
    
    public fun struct_copy_demo() {
        let p1 = Price { value: 1000, decimals: 8, symbol: b"BTC" };
        let p2 = p1;   // copy
        let p3 = p1;   // copy อีกครั้ง - p1 ยังใช้ได้
        assert!(p1.value == p2.value, 0);
        assert!(p1.value == p3.value, 1);
    }
    
    // ============================================
    // Types ที่ ไม่มี copy ability
    // ============================================
    
    struct Ticket {  // no copy!
        id: u64,
        event: vector<u8>,
    }
    
    public fun ticket_no_copy_demo() {
        let t1 = Ticket { id: 1, event: b"Concert" };
        let t2 = t1;   // MOVE (ไม่ใช่ copy)
        // t1 ไม่สามารถใช้ได้อีก!
        // let t3 = t1;  // ❌ ERROR: t1 was moved
        
        // t2 ต้องถูก consumed ก่อนออกจาก scope
        let Ticket { id: _, event: _ } = t2;
    }
    
    // ============================================
    // Copy Constraints
    // ============================================
    
    // struct ที่มี copy ต้องมี fields ที่มี copy ทั้งหมด
    struct CopyableInner has copy, drop {
        value: u64,
    }
    
    struct CopyableOuter has copy, drop {
        inner: CopyableInner,  // ✅ CopyableInner มี copy
        count: u64,            // ✅ u64 มี copy
    }
    
    // struct GoodForCopy has copy { inner: NonCopy }  // ❌ ERROR
    
    // ============================================
    // When to use copy
    // ============================================
    
    // ✅ ใช้ copy สำหรับ:
    // - ค่าที่เป็น data (ไม่ใช่ asset)
    // - Configurations ที่ต้องส่งหลายที่
    // - Small value types
    // - Metadata ที่ read-only
    
    struct TokenMetadata has copy, drop, store {
        name: vector<u8>,
        symbol: vector<u8>,
        decimals: u8,
    }
    
    struct PoolConfig has copy, drop, store {
        fee_bps: u64,
        max_slippage_bps: u64,
        is_active: bool,
    }
    
    // ❌ ไม่ควรใช้ copy สำหรับ:
    // - Assets (tokens, NFTs)
    // - Capabilities
    // - Anything that should have unique ownership
}
```

---

## Drop Ability

```move
module learning::drop_ability {
    use std::vector;
    
    // ============================================
    // Types ที่มี drop ability
    // ============================================
    
    struct TempResult has drop {
        value: u64,
        is_valid: bool,
    }
    
    public fun drop_demo() {
        {
            let result = TempResult { value: 42, is_valid: true };
            // result ถูก drop อัตโนมัติตอนออกจาก block
        }
        // ไม่ต้อง explicit drop
    }
    
    // ============================================
    // Types ที่ ไม่มี drop ability
    // ============================================
    
    struct Asset {  // no drop!
        id: u64,
        amount: u64,
    }
    
    public fun must_consume_demo() {
        let asset = Asset { id: 1, amount: 1000 };
        
        // asset ต้องถูก consumed ก่อนออกจาก scope
        // ❌ ERROR ถ้าออกจาก function โดยไม่ consume:
        // "the value was not consumed or moved"
        
        // ✅ ต้อง consume explicitly:
        let Asset { id: _, amount: _ } = asset;
    }
    
    // ============================================
    // Hot Potato Pattern
    // ============================================
    
    // Struct ที่ไม่มีทั้ง drop และ copy = Hot Potato
    struct HotPotato {
        data: u64,
    }
    
    public fun create_hot_potato(): HotPotato {
        HotPotato { data: 42 }
    }
    
    // ต้อง consume hot potato ด้วย function นี้เท่านั้น
    public fun resolve_hot_potato(potato: HotPotato): u64 {
        let HotPotato { data } = potato;
        data
    }
    
    // ใช้ใน transaction:
    // let potato = create_hot_potato();
    // let result = resolve_hot_potato(potato);  // ✅ consumed!
    
    // ============================================
    // Drop ใน vector operations
    // ============================================
    
    struct Item has drop, store {
        name: vector<u8>,
        value: u64,
    }
    
    public fun discard_items(items: vector<Item>) {
        // vector ถูก drop อัตโนมัติ (ถ้า Item มี drop ability)
        // ไม่ต้อง explicit drop elements
    }
    
    // ============================================
    // When to use drop
    // ============================================
    
    // ✅ ใช้ drop สำหรับ:
    // - ผลลัพธ์ชั่วคราว
    // - Events (ต้องมี drop)
    // - Data ที่ไม่ต้องการ preserve
    
    struct TransferEvent has drop, store {
        from: address,
        to: address,
        amount: u64,
        timestamp: u64,
    }
    
    // ❌ ไม่ใช้ drop สำหรับ:
    // - Assets ที่มีค่า
    // - Capabilities
    // - ทุกอย่างที่ต้องการ ensure "always used"
}
```

---

## Store Ability

```move
module learning::store_ability {
    use std::vector;
    use aptos_framework::table::{Self, Table};
    
    // ============================================
    // Store ช่วยให้เก็บใน containers ได้
    // ============================================
    
    // ✅ มี store - สามารถเก็บใน struct อื่น
    struct StorableData has store, copy, drop {
        value: u64,
    }
    
    // ❌ ไม่มี store - ไม่สามารถเก็บใน struct อื่น
    struct NonStorable {
        value: u64,
    }
    
    struct Container has key {
        data: StorableData,   // ✅ StorableData มี store
        // bad: NonStorable  // ❌ ERROR: NonStorable ไม่มี store
    }
    
    // ============================================
    // Store ใน Table (Aptos)
    // ============================================
    
    struct TokenBalance has store {
        amount: u64,
        last_updated: u64,
    }
    
    struct TokenLedger has key {
        // Table<key_type, value_type> - ต้องการ store ใน value_type
        balances: Table<address, TokenBalance>,
        total: u64,
    }
    
    // ============================================
    // Nested Store Requirements
    // ============================================
    
    struct Level3 has store, copy, drop {
        data: u64,
    }
    
    struct Level2 has store, copy, drop {
        inner: Level3,  // Level3 ต้องมี store
    }
    
    struct Level1 has store {
        inner: Level2,  // Level2 ต้องมี store
    }
    
    struct TopLevel has key {
        data: Level1,   // Level1 ต้องมี store
    }
    
    // ============================================
    // Store + Generics
    // ============================================
    
    struct Wrapper<T: store> has key {
        value: T,
    }
    
    // T ต้องมี store เพื่อเก็บใน Wrapper
    public fun create_wrapper<T: store>(value: T): Wrapper<T> {
        Wrapper { value }
    }
    
    // ============================================
    // Common Store Types
    // ============================================
    
    struct UserData has key, store {
        profile: ProfileData,
        settings: SettingsData,
        stats: StatsData,
    }
    
    struct ProfileData has store, copy, drop {
        name: vector<u8>,
        avatar_uri: vector<u8>,
        created_at: u64,
    }
    
    struct SettingsData has store, copy, drop {
        notifications_enabled: bool,
        language: u8,
        theme: u8,
    }
    
    struct StatsData has store, copy, drop {
        total_transactions: u64,
        total_volume: u64,
        last_active: u64,
    }
}
```

---

## Key Ability

```move
module learning::key_ability {
    use std::signer;
    
    // ============================================
    // Key = สามารถเป็น top-level resource
    // ============================================
    
    // ✅ มี key - สามารถ move_to, borrow_global, move_from
    struct UserAccount has key {
        balance: u64,
        owner: address,
    }
    
    // ❌ ไม่มี key - ไม่สามารถ move_to โดยตรง
    struct HelperData has store {
        value: u64,
    }
    
    // ============================================
    // Key Operations
    // ============================================
    
    public fun store_account(account: &signer, initial_balance: u64) {
        let addr = signer::address_of(account);
        
        // move_to ต้องการ:
        // 1. signer ของ address ที่จะเก็บ
        // 2. struct ที่มี key ability
        move_to(account, UserAccount {
            balance: initial_balance,
            owner: addr,
        });
    }
    
    public fun read_balance(addr: address): u64 acquires UserAccount {
        // borrow_global ต้องการ:
        // 1. address ที่เก็บ resource
        // 2. struct ที่มี key ability
        borrow_global<UserAccount>(addr).balance
    }
    
    public fun update_balance(account: &signer, new_balance: u64) acquires UserAccount {
        let addr = signer::address_of(account);
        borrow_global_mut<UserAccount>(addr).balance = new_balance;
    }
    
    public fun remove_account(account: &signer) acquires UserAccount {
        let addr = signer::address_of(account);
        let UserAccount { balance: _, owner: _ } = move_from<UserAccount>(addr);
        // Resource destroyed
    }
    
    public fun account_exists(addr: address): bool {
        exists<UserAccount>(addr)
    }
    
    // ============================================
    // Key + Store
    // ============================================
    
    // struct ที่มีทั้ง key และ store
    // สามารถ:
    // - เป็น top-level resource (key)
    // - เก็บใน struct อื่นหรือ Table (store)
    
    struct Token has key, store {
        id: u64,
        amount: u64,
    }
    
    struct Vault has key {
        tokens: vector<Token>,  // Token มี store จึงเก็บได้
        total: u64,
    }
    
    // ============================================
    // One Resource Per Address Per Type
    // ============================================
    
    // กฎสำคัญ: แต่ละ address เก็บ resource แต่ละ type ได้แค่ ONE ตัว!
    
    public fun demo_single_resource(account: &signer) {
        let addr = signer::address_of(account);
        
        if (!exists<UserAccount>(addr)) {
            move_to(account, UserAccount { balance: 0, owner: addr });
        };
        
        // ❌ ไม่สามารถ move_to ซ้ำได้:
        // move_to(account, UserAccount { ... });  // ERROR: already exists
    }
}
```

---

## Ability Constraints

```move
module learning::ability_constraints {
    use std::vector;
    
    // ============================================
    // Generic constraints
    // ============================================
    
    // T ต้องมี copy ability
    public fun clone_value<T: copy>(value: T): (T, T) {
        (value, value)  // copy T สองครั้ง
    }
    
    // T ต้องมี drop ability
    public fun discard<T: drop>(value: T) {
        // value ถูก drop อัตโนมัติ
    }
    
    // T ต้องมี store ability
    public fun store_in_vector<T: store>(value: T): vector<T> {
        vector[value]  // vector<T> ต้องการ T: store
    }
    
    // Multiple constraints
    public fun process<T: copy + drop + store>(value: T): T {
        let copy1 = value;  // copy เพราะ T: copy
        // copy1 จะถูก drop ตอนออก scope เพราะ T: drop
        value  // return original
    }
    
    // ============================================
    // Ability inheritance
    // ============================================
    
    // ถ้า struct มี ability A, ทุก field ต้องมี ability A ด้วย
    
    struct Inner has copy, drop, store {
        x: u64,
    }
    
    // ✅ สามารถมี copy เพราะ Inner มี copy
    struct Outer has copy, drop, store {
        inner: Inner,
        value: u64,
    }
    
    // ❌ ไม่สามารถมี copy เพราะ NonCopy ไม่มี copy
    struct NonCopy {
        value: u64,
    }
    
    // struct BadOuter has copy { nc: NonCopy }  // ERROR!
    
    // ============================================
    // Checking abilities at runtime (Aptos)
    // ============================================
    
    // ใน Aptos Move มี type_info ที่สามารถ check ได้
    // แต่ส่วนมากใช้ compile-time checks ผ่าน generics
    
    // ตัวอย่าง: ฟังก์ชันที่ต้องการ copy + drop
    public fun safe_copy_and_drop<T: copy + drop>(
        values: &vector<T>
    ): vector<T> {
        let result = vector::empty<T>();
        let i = 0u64;
        let len = vector::length(values);
        
        while (i < len) {
            let val = *vector::borrow(values, i);  // copy เพราะ T: copy
            vector::push_back(&mut result, val);
            i = i + 1;
        };
        
        result
    }
}
```

---

## Ability Checking

```move
module learning::ability_check_patterns {
    
    // ============================================
    // Design: choosing right abilities
    // ============================================
    
    // 1. Financial Assets: ไม่มี copy, ไม่มี drop
    struct FinancialAsset has key, store {
        id: u64,
        amount: u64,
        owner: address,
    }
    // ✅ ป้องกัน double-spend, ป้องกันการสูญหาย
    
    // 2. Data Object: copy + drop + store
    struct DataObject has copy, drop, store {
        value: u64,
        metadata: vector<u8>,
    }
    // ✅ ใช้ง่าย, ส่งต่อได้ง่าย
    
    // 3. Capability: ไม่มี copy, มี drop (สามารถ revoke)
    struct MintCapability has key {
        authorized: bool,
    }
    // ✅ ไม่สามารถ duplicate, สามารถ revoke ได้
    
    // 4. Soulbound Token: key เท่านั้น (ไม่สามารถ transfer)
    struct SoulboundToken has key {
        id: u64,
        achievement: vector<u8>,
    }
    // ✅ tied ถาวรกับ account
    
    // 5. Event: drop + store
    struct TransferEvent has drop, store {
        from: address,
        to: address,
        amount: u64,
    }
    // ✅ events ต้องมี drop
    
    // ============================================
    // Ability Checklist
    // ============================================
    
    /*
    ถามตัวเองก่อนกำหนด abilities:
    
    1. ค่านี้สามารถ copy/duplicate ได้หรือไม่?
       → YES: เพิ่ม copy
       → NO: อย่าใส่ copy (เช่น assets, caps)
    
    2. ถ้า "ลืม" ค่านี้ไปจะเป็นไรหรือไม่?
       → ไม่เป็นไร: เพิ่ม drop
       → เป็นปัญหา (เช่น asset หาย): อย่าใส่ drop
    
    3. ต้องการเก็บค่านี้ใน struct อื่นหรือ Table หรือไม่?
       → YES: เพิ่ม store
       → NO: อาจไม่จำเป็น
    
    4. ต้องการเก็บค่านี้เป็น top-level ใน account หรือไม่?
       → YES: เพิ่ม key (และ store ด้วย ถ้าต้องการ nest)
       → NO: ไม่จำเป็น
    */
}
```

---

## ตัวอย่างโปรแกรม: Token Protocol

```move
module learning::token_protocol {
    use std::signer;
    use std::vector;
    use std::string::{Self, String};
    
    // ============================================
    // Token Types (different ability combinations)
    // ============================================
    
    // 1. Main Token (Asset) - no copy, no drop
    struct Token has key, store {
        amount: u64,
    }
    
    // 2. Token Metadata - copy + drop + store
    struct TokenMetadata has copy, drop, store {
        name: String,
        symbol: String,
        decimals: u8,
        total_supply: u64,
        icon_uri: String,
    }
    
    // 3. Mint Capability - no copy (singleton)
    struct MintCapability has key {
        max_mint_per_tx: u64,
    }
    
    // 4. Burn Capability - no copy, has drop
    struct BurnCapability has key, drop {
        active: bool,
    }
    
    // 5. Transfer Event - drop + store (for event handling)
    struct TransferEvent has drop, store {
        from: address,
        to: address,
        amount: u64,
        timestamp: u64,
    }
    
    // 6. Protocol Config - stored globally
    struct ProtocolConfig has key {
        metadata: TokenMetadata,  // has store
        admin: address,
        is_paused: bool,
        transfer_fee_bps: u64,
    }
    
    // ============================================
    // Error codes
    // ============================================
    
    const E_PAUSED: u64 = 1;
    const E_ZERO_AMOUNT: u64 = 2;
    const E_INSUFFICIENT_BALANCE: u64 = 3;
    const E_NOT_AUTHORIZED: u64 = 4;
    const E_ALREADY_INITIALIZED: u64 = 5;
    
    // ============================================
    // Initialization
    // ============================================
    
    public entry fun initialize_protocol(
        admin: &signer,
        name: vector<u8>,
        symbol: vector<u8>,
        decimals: u8,
        max_supply: u64,
        transfer_fee_bps: u64,
    ) {
        let addr = signer::address_of(admin);
        assert!(!exists<ProtocolConfig>(addr), E_ALREADY_INITIALIZED);
        
        let metadata = TokenMetadata {
            name: string::utf8(name),
            symbol: string::utf8(symbol),
            decimals,
            total_supply: 0,
            icon_uri: string::utf8(b""),
        };
        
        move_to(admin, ProtocolConfig {
            metadata,           // store metadata (has store)
            admin: addr,
            is_paused: false,
            transfer_fee_bps,
        });
        
        // Grant capabilities to admin
        move_to(admin, MintCapability { max_mint_per_tx: max_supply });
        move_to(admin, BurnCapability { active: true });
    }
    
    // ============================================
    // Mint (requires MintCapability)
    // ============================================
    
    public entry fun mint(
        minter: &signer,
        config_addr: address,
        amount: u64,
    ) acquires ProtocolConfig, MintCapability {
        assert!(amount > 0, E_ZERO_AMOUNT);
        
        let minter_addr = signer::address_of(minter);
        
        // Check mint cap
        let cap = borrow_global<MintCapability>(minter_addr);
        assert!(amount <= cap.max_mint_per_tx, E_NOT_AUTHORIZED);
        
        let config = borrow_global_mut<ProtocolConfig>(config_addr);
        assert!(!config.is_paused, E_PAUSED);
        
        config.metadata.total_supply = config.metadata.total_supply + amount;
        
        // เพิ่ม token ให้ minter
        if (exists<Token>(minter_addr)) {
            let token = borrow_global_mut<Token>(minter_addr);
            token.amount = token.amount + amount;
        } else {
            move_to(minter, Token { amount });
        }
    }
    
    // ============================================
    // Transfer
    // ============================================
    
    public entry fun transfer(
        from: &signer,
        to: address,
        config_addr: address,
        amount: u64,
    ) acquires ProtocolConfig, Token {
        assert!(amount > 0, E_ZERO_AMOUNT);
        
        let from_addr = signer::address_of(from);
        assert!(from_addr != to, 0);
        
        // Check protocol state
        let config = borrow_global<ProtocolConfig>(config_addr);
        assert!(!config.is_paused, E_PAUSED);
        
        // Calculate fee
        let fee = amount * config.transfer_fee_bps / 10_000;
        let net_amount = amount - fee;
        
        // Debit from sender
        let from_token = borrow_global_mut<Token>(from_addr);
        assert!(from_token.amount >= amount, E_INSUFFICIENT_BALANCE);
        from_token.amount = from_token.amount - amount;
        
        // Credit to recipient (simplified - ใน production handle account creation)
        if (exists<Token>(to)) {
            let to_token = borrow_global_mut<Token>(to);
            to_token.amount = to_token.amount + net_amount;
        }
        
        // Fee goes to admin (simplified)
        let admin = config.admin;
        if (fee > 0 && exists<Token>(admin)) {
            let admin_token = borrow_global_mut<Token>(admin);
            admin_token.amount = admin_token.amount + fee;
        }
    }
    
    // ============================================
    // View Functions
    // ============================================
    
    #[view]
    public fun balance_of(addr: address): u64 acquires Token {
        if (!exists<Token>(addr)) { return 0 };
        borrow_global<Token>(addr).amount
    }
    
    #[view]
    public fun get_metadata(config_addr: address): TokenMetadata acquires ProtocolConfig {
        borrow_global<ProtocolConfig>(config_addr).metadata
    }
    
    #[view]
    public fun total_supply(config_addr: address): u64 acquires ProtocolConfig {
        borrow_global<ProtocolConfig>(config_addr).metadata.total_supply
    }
    
    // ============================================
    // Admin Functions
    // ============================================
    
    public entry fun pause(admin: &signer, config_addr: address) acquires ProtocolConfig {
        let addr = signer::address_of(admin);
        let config = borrow_global_mut<ProtocolConfig>(config_addr);
        assert!(config.admin == addr, E_NOT_AUTHORIZED);
        config.is_paused = true;
    }
    
    public entry fun revoke_burn_capability(
        admin: &signer,
    ) acquires BurnCapability {
        let addr = signer::address_of(admin);
        assert!(exists<BurnCapability>(addr), E_NOT_AUTHORIZED);
        
        // BurnCapability มี drop ability - ดังนั้น move_from จะ drop มัน
        let BurnCapability { active: _ } = move_from<BurnCapability>(addr);
        // Capability ถูก destroyed
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test(admin = @0x1, user = @0x2)]
    public fun test_token_protocol(
        admin: &signer,
        user: &signer,
    ) acquires ProtocolConfig, MintCapability, Token {
        let admin_addr = signer::address_of(admin);
        let user_addr = signer::address_of(user);
        
        // Initialize
        initialize_protocol(admin, b"TestToken", b"TST", 8, 1_000_000, 100);
        
        // Mint to admin
        mint(admin, admin_addr, 10_000);
        assert!(balance_of(admin_addr) == 10_000, 0);
        assert!(total_supply(admin_addr) == 10_000, 1);
        
        // Setup user account
        move_to(user, Token { amount: 0 });
        
        // Transfer 1000 (fee 100 = 1%)
        transfer(admin, user_addr, admin_addr, 1_000);
        assert!(balance_of(user_addr) == 990, 2);  // 1000 - 1% fee
        assert!(balance_of(admin_addr) == 9_010, 3);  // 10000 - 1000 + 10 (fee)
        
        // Metadata check
        let metadata = get_metadata(admin_addr);
        assert!(metadata.total_supply == 10_000, 4);
    }
}
```

---

## สรุป

| Ability | ความหมาย | ตัวอย่าง |
|---------|----------|---------|
| `copy` | สามารถ copy ค่าได้ | integers, metadata, configs |
| `drop` | สามารถทิ้งค่าได้ | events, temp results |
| `store` | เก็บใน struct/table ได้ | ทุกอย่างที่ต้องการ nest |
| `key` | เป็น global resource ได้ | accounts, tokens, caps |

---

## แบบฝึกหัด

### Exercise 11.1: Ability Design
กำหนด abilities ที่เหมาะสมสำหรับ:
1. `Loan` - สัญญากู้ยืม
2. `Receipt` - ใบเสร็จการซื้อขาย
3. `Governance Vote` - การโหวต DAO
4. `Staking Position` - ตำแหน่งการ stake

### Exercise 11.2: Implementing Soulbound
สร้าง Soulbound Token ที่:
- ไม่สามารถ transfer ได้
- แต่มีข้อมูล metadata
- Admin สามารถ revoke ได้

---

**ต่อไป**: [Part 12 - Generics →](part-12-generics.md)
