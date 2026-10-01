# Part 04: Variables และ Mutability

## สารบัญ
- [การประกาศ Variables](#การประกาศ-variables)
- [Mutability](#mutability)
- [Variable Binding](#variable-binding)
- [Destructuring](#destructuring)
- [Constants](#constants)
- [Global vs Local Variables](#global-vs-local-variables)
- [Move Semantics](#move-semantics)
- [ตัวอย่างโปรแกรม](#ตัวอย่างโปรแกรม)
- [แบบฝึกหัด](#แบบฝึกหัด)

---

## การประกาศ Variables

ใน Move ใช้ keyword `let` ในการประกาศ variable

```move
module learning::variables {
    
    public fun variable_declaration() {
        // การประกาศพื้นฐาน
        let x = 10;          // type inference: u64
        let y: u64 = 20;     // explicit type annotation
        let z: u8 = 42u8;    // type annotation + typed literal
        
        // ประกาศหลายตัวพร้อมกัน (tuple destructuring)
        let (a, b, c) = (1u64, 2u64, 3u64);
        // a = 1, b = 2, c = 3
        
        // ประกาศแบบมี type annotation ในแต่ละตัว
        let (p, q): (u64, bool) = (100, true);
        
        // Unused variable (เริ่มต้นด้วย _ เพื่อไม่ให้ compiler warning)
        let _unused = 42;
    }
    
    // ตัวอย่างการใช้ตัวแปรใน context จริง
    public fun token_calculation(): u64 {
        let total_supply: u64 = 1_000_000_000;  // 1 billion tokens
        let team_allocation_bps: u64 = 2000;     // 20% ใน basis points
        let investor_allocation_bps: u64 = 3000; // 30%
        let community_allocation_bps: u64 = 5000; // 50%
        
        let team_tokens = total_supply * team_allocation_bps / 10_000;
        let investor_tokens = total_supply * investor_allocation_bps / 10_000;
        let community_tokens = total_supply * community_allocation_bps / 10_000;
        
        // ตรวจสอบว่าบวกกันแล้วได้ total_supply
        assert!(
            team_tokens + investor_tokens + community_tokens == total_supply,
            0
        );
        
        community_tokens
    }
}
```

---

## Mutability

Move แยกแยะ mutable และ immutable variables อย่างชัดเจน

```move
module learning::mutability {
    
    public fun immutable_demo() {
        let x = 10u64;
        // x = 20;  // ❌ ERROR: x is not mutable
        
        // ต้อง declare ใหม่ (shadowing)
        let x = x + 10;  // ✅ shadow x ด้วยค่าใหม่
        // ตอนนี้ x = 20
    }
    
    public fun mutable_demo() {
        let mut x = 10u64;  // ประกาศเป็น mutable
        x = 20;             // ✅ แก้ไขได้
        x = x + 5;          // ✅ 25
        x += 1;             // ✅ compound assignment (25 + 1 = 26)
        x -= 2;             // ✅ 24
        x *= 2;             // ✅ 48
        x /= 4;             // ✅ 12
        x %= 5;             // ✅ 2
    }
    
    public fun mutable_vector_demo() {
        let mut v = vector[1u64, 2u64, 3u64];
        
        // เพิ่ม element
        vector::push_back(&mut v, 4);
        
        // แก้ไข element
        *vector::borrow_mut(&mut v, 0) = 100;
        
        // ลบ element สุดท้าย
        let last = vector::pop_back(&mut v);
        
        // v = [100, 2, 3], last = 4
    }
    
    // Mutable references
    public fun increment_value(x: &mut u64) {
        *x = *x + 1;
    }
    
    public fun demo_mutable_ref() {
        let mut x = 10u64;
        increment_value(&mut x);  // x = 11
        increment_value(&mut x);  // x = 12
    }
}

use std::vector;
```

### ทำไม Immutable by Default?

```
ข้อดีของ Immutable by Default:
1. ✅ ลด bugs จาก unintended mutation
2. ✅ Code ง่ายต่อการ reason about
3. ✅ Thread safety (ไม่มี data race)
4. ✅ Compiler สามารถ optimize ได้ดีกว่า
5. ✅ ชัดเจนว่าอะไร mutable อะไรไม่ mutable
```

---

## Variable Binding

```move
module learning::binding {
    
    struct Point {
        x: u64,
        y: u64,
    }
    
    struct Rectangle {
        top_left: Point,
        bottom_right: Point,
    }
    
    public fun binding_demo() {
        // Simple binding
        let n = 42u64;
        
        // Pattern binding with struct
        let Point { x, y } = Point { x: 10, y: 20 };
        // x = 10, y = 20
        
        // ใช้ชื่อต่างกัน
        let Point { x: px, y: py } = Point { x: 5, y: 15 };
        // px = 5, py = 15
        
        // Nested struct binding
        let rect = Rectangle {
            top_left: Point { x: 0, y: 10 },
            bottom_right: Point { x: 100, y: 0 },
        };
        
        let Rectangle {
            top_left: Point { x: x1, y: y1 },
            bottom_right: Point { x: x2, y: y2 },
        } = rect;
        // x1=0, y1=10, x2=100, y2=0
    }
    
    // Tuple binding
    public fun tuple_binding(): u64 {
        // ฟังก์ชันที่ return หลายค่า
        let (min, max) = find_min_max(vector[5u64, 2u64, 8u64, 1u64, 9u64]);
        max - min  // 9 - 1 = 8
    }
    
    fun find_min_max(v: vector<u64>): (u64, u64) {
        assert!(!vector::is_empty(&v), 0);
        
        let min = *vector::borrow(&v, 0);
        let max = *vector::borrow(&v, 0);
        let i = 1u64;
        let len = vector::length(&v);
        
        while (i < len) {
            let val = *vector::borrow(&v, i);
            if (val < min) { min = val; };
            if (val > max) { max = val; };
            i = i + 1;
        };
        
        (min, max)
    }
    
    // Wildcards in binding
    public fun wildcard_binding() {
        let (first, _, third) = (1u64, 2u64, 3u64);
        // first = 1, third = 3, 2 ถูกทิ้ง
        
        let Point { x, .. } = Point { x: 10, y: 20 };
        // x = 10, y ถูกทิ้ง (ใช้ .. สำหรับ struct fields)
        // (ไม่รองรับทุก compiler version)
    }
}

use std::vector;
```

---

## Destructuring

```move
module learning::destructuring {
    use std::vector;
    
    struct Token {
        id: u64,
        owner: address,
        amount: u64,
        is_locked: bool,
    }
    
    // Destructuring ใน let binding
    public fun destructure_token(token: Token): (u64, address, u64) {
        let Token { id, owner, amount, is_locked: _ } = token;
        (id, owner, amount)
    }
    
    // Destructuring ใน function parameters
    // (Move ไม่ support โดยตรง แต่สามารถ destructure ในตัวฟังก์ชัน)
    public fun get_token_info(token: &Token): (u64, address) {
        (token.id, token.owner)
    }
    
    // Destructuring กับ vector
    public fun first_and_rest(v: &vector<u64>): (u64, u64) {
        assert!(vector::length(v) >= 2, 0);
        
        let first = *vector::borrow(v, 0);
        let second = *vector::borrow(v, 1);
        (first, second)
    }
    
    // Mutable destructuring
    public fun update_token(token: &mut Token, new_amount: u64) {
        token.amount = new_amount;
    }
    
    // Complex destructuring example
    struct Position {
        x: u64,
        y: u64,
        z: u64,
    }
    
    public fun calculate_distance_3d(p1: &Position, p2: &Position): u64 {
        let Position { x: x1, y: y1, z: z1 } = *p1;
        let Position { x: x2, y: y2, z: z2 } = *p2;
        
        // คำนวณ Manhattan distance (ใช้ integer เพราะ Move ไม่มี float)
        let dx = if (x1 > x2) { x1 - x2 } else { x2 - x1 };
        let dy = if (y1 > y2) { y1 - y2 } else { y2 - y1 };
        let dz = if (z1 > z2) { z1 - z2 } else { z2 - z1 };
        
        dx + dy + dz
    }
}
```

---

## Constants

```move
module learning::constants {
    
    // Constants ใน module level
    // ต้องเป็น primitive types เท่านั้น (u8, u64, bool, address, vector<u8>)
    
    // Numeric constants
    const MAX_SUPPLY: u64 = 1_000_000_000;
    const MIN_STAKE: u64 = 100_000_000;  // 1 APT
    const FEE_DENOMINATOR: u64 = 10_000;  // basis points
    const PROTOCOL_FEE_BPS: u64 = 30;    // 0.30%
    
    // Boolean constants
    const IS_PAUSED: bool = false;
    const REQUIRE_KYC: bool = false;
    
    // Address constants
    const TREASURY: address = @0x1;
    const ZERO_ADDRESS: address = @0x0;
    
    // Byte string constants
    const MODULE_NAME: vector<u8> = b"DeFi Protocol v1";
    
    // Error code constants (convention: E_ prefix)
    const E_NOT_AUTHORIZED: u64 = 1;
    const E_INSUFFICIENT_BALANCE: u64 = 2;
    const E_ZERO_AMOUNT: u64 = 3;
    const E_OVERFLOW: u64 = 4;
    const E_PAUSED: u64 = 5;
    
    // ใช้ constants
    public fun calculate_fee(amount: u64): u64 {
        assert!(amount > 0, E_ZERO_AMOUNT);
        amount * PROTOCOL_FEE_BPS / FEE_DENOMINATOR
    }
    
    public fun validate_stake(amount: u64) {
        assert!(amount >= MIN_STAKE, E_INSUFFICIENT_BALANCE);
        assert!(amount <= MAX_SUPPLY, E_OVERFLOW);
    }
    
    #[test]
    fun test_fee_calculation() {
        let fee = calculate_fee(1_000_000);
        assert!(fee == 30, 0);  // 1,000,000 * 30 / 10,000 = 3,000
        
        let big_fee = calculate_fee(1_000_000_000);
        assert!(big_fee == 3_000_000, 1);  // 0.30% of 1B
    }
}
```

### Best Practices สำหรับ Constants

```move
module learning::constant_patterns {
    
    // ✅ ดี: ใช้ชื่อที่ชัดเจนและ descriptive
    const MAX_TOKENS_PER_ACCOUNT: u64 = 1000;
    const MINIMUM_DEPOSIT_AMOUNT: u64 = 100_000;
    const LOCK_PERIOD_SECONDS: u64 = 86400;  // 1 day
    
    // ✅ ดี: Group related constants
    // Fee constants
    const LP_FEE_BPS: u64 = 25;          // 0.25%
    const PROTOCOL_FEE_BPS: u64 = 5;     // 0.05%
    const TOTAL_FEE_BPS: u64 = 30;       // 0.30%
    
    // Time constants (in seconds)
    const SECONDS_PER_MINUTE: u64 = 60;
    const SECONDS_PER_HOUR: u64 = 3600;
    const SECONDS_PER_DAY: u64 = 86400;
    const SECONDS_PER_WEEK: u64 = 604800;
    
    // Decimal precision
    const PRECISION_6: u64 = 1_000_000;         // 6 decimals (USDC-like)
    const PRECISION_8: u64 = 100_000_000;        // 8 decimals (BTC-like)
    const PRECISION_9: u64 = 1_000_000_000;      // 9 decimals (SUI-like)
    
    // Status codes
    const STATUS_ACTIVE: u8 = 1;
    const STATUS_PAUSED: u8 = 2;
    const STATUS_CLOSED: u8 = 3;
    
    // ✅ ดี: Error codes เรียงตาม category
    // Auth errors: 100-199
    const E_NOT_OWNER: u64 = 100;
    const E_NOT_ADMIN: u64 = 101;
    const E_NOT_AUTHORIZED: u64 = 102;
    
    // Input errors: 200-299
    const E_ZERO_AMOUNT: u64 = 200;
    const E_INVALID_ADDRESS: u64 = 201;
    const E_INVALID_DEADLINE: u64 = 202;
    
    // State errors: 300-399
    const E_CONTRACT_PAUSED: u64 = 300;
    const E_INSUFFICIENT_BALANCE: u64 = 301;
    const E_POSITION_NOT_FOUND: u64 = 302;
}
```

---

## Global vs Local Variables

```move
module learning::storage_types {
    use std::signer;
    
    // Global Storage - เก็บใน Blockchain state
    // ใช้ move_to, borrow_global, borrow_global_mut, exists, move_from
    
    struct GlobalConfig has key {
        fee_bps: u64,
        max_supply: u64,
        is_paused: bool,
        admin: address,
    }
    
    // Local Variables - อยู่แค่ใน transaction/function call
    // ถูกสร้างและทำลายใน function scope
    
    // Initialize global storage
    public entry fun initialize(admin: &signer) {
        let admin_addr = signer::address_of(admin);
        
        // move_to: เก็บ Resource ไว้ใน account's global storage
        move_to(admin, GlobalConfig {
            fee_bps: 30,
            max_supply: 1_000_000_000,
            is_paused: false,
            admin: admin_addr,
        });
    }
    
    // Read from global storage
    #[view]
    public fun get_fee_bps(config_addr: address): u64 acquires GlobalConfig {
        // borrow_global: อ่านค่าจาก global storage (immutable)
        borrow_global<GlobalConfig>(config_addr).fee_bps
    }
    
    // Write to global storage
    public entry fun update_fee(
        admin: &signer,
        new_fee_bps: u64
    ) acquires GlobalConfig {
        let admin_addr = signer::address_of(admin);
        
        // borrow_global_mut: อ่านและเขียนค่า (mutable)
        let config = borrow_global_mut<GlobalConfig>(admin_addr);
        
        // ตรวจสอบ permissions
        assert!(config.admin == admin_addr, 100);
        assert!(new_fee_bps <= 1000, 200);  // max 10% fee
        
        config.fee_bps = new_fee_bps;
    }
    
    // ตรวจสอบว่า resource มีอยู่
    #[view]
    public fun is_initialized(addr: address): bool {
        exists<GlobalConfig>(addr)
    }
    
    // Remove from global storage
    public entry fun destroy_config(admin: &signer) acquires GlobalConfig {
        let admin_addr = signer::address_of(admin);
        
        // move_from: เอา Resource ออกจาก global storage
        let GlobalConfig {
            fee_bps: _,
            max_supply: _,
            is_paused: _,
            admin: _,
        } = move_from<GlobalConfig>(admin_addr);
        // Resource ถูก destroy ที่นี่
    }
}
```

---

## Move Semantics

Move ใช้ **Move Semantics** - ค่าถูก "move" ไม่ใช่ "copy"

```move
module learning::move_semantics {
    use std::signer;
    
    struct Token has key {
        id: u64,
        value: u64,
    }
    
    // Copy vs Move
    public fun copy_vs_move_demo() {
        // Primitive types ที่มี 'copy' ability - ถูก copy อัตโนมัติ
        let x = 10u64;
        let y = x;  // x ถูก copy ไปยัง y
        // ทั้ง x และ y ยังสามารถใช้ได้
        let z = x + y;  // ✅ x = 10, y = 10, z = 20
        
        // Struct ที่ไม่มี copy ability - ถูก move
        let token = Token { id: 1, value: 100 };
        let token2 = token;  // token ถูก MOVE ไปยัง token2
        // token ไม่สามารถใช้ได้อีก!
        // let id = token.id;  // ❌ ERROR: token was moved
        let id = token2.id;   // ✅ ใช้ token2 แทน
    }
    
    // การส่ง Resource ระหว่างฟังก์ชัน
    public fun transfer_token(token: Token, recipient: address) {
        // token ถูก move เข้ามาใน function นี้
        // owner เดิมไม่สามารถใช้ token ได้แล้ว
        
        // ทำอะไรสักอย่างกับ token...
        
        // token ต้องถูก "ใช้" (move_to หรือ return หรือ destroy)
        // ถ้าไม่ทำ compiler จะ error!
        
        // ตัวอย่าง: ทิ้ง token (ต้องมี drop ability ถึงจะทำได้)
        // let Token { id: _, value: _ } = token;  // explicit drop
    }
    
    // Consuming a Resource
    public fun destroy_token(token: Token): u64 {
        let Token { id: _, value } = token;  // destructure และ consume
        value  // return ค่า
    }
    
    // Borrowing (ไม่ move)
    public fun read_token_value(token: &Token): u64 {
        token.value  // อ่านค่าโดยไม่ consume
    }
    
    public fun update_token_value(token: &mut Token, new_value: u64) {
        token.value = new_value;  // แก้ไขโดยไม่ consume
    }
    
    // ตัวอย่างการจัดการ Resources อย่างถูกต้อง
    public entry fun create_and_store_token(account: &signer) {
        let token = Token { id: 1, value: 1000 };
        // ต้องทำอะไรกับ token:
        // 1. move_to (เก็บใน account)
        move_to(account, token);
        // ✅ token ถูก move ไปยัง account's storage
    }
    
    public entry fun use_token(
        account: &signer
    ) acquires Token {
        let addr = signer::address_of(account);
        
        // อ่านค่าโดยไม่ consume
        let value = borrow_global<Token>(addr).value;
        
        // หรือ consume
        // let token = move_from<Token>(addr);
        // let final_value = destroy_token(token);
    }
}
```

---

## ตัวอย่างโปรแกรม: Inventory System

```move
module learning::inventory {
    use std::signer;
    use std::vector;
    use std::string::{Self, String};
    
    // Constants
    const MAX_ITEMS: u64 = 100;
    const E_INVENTORY_FULL: u64 = 1;
    const E_ITEM_NOT_FOUND: u64 = 2;
    const E_INVALID_QUANTITY: u64 = 3;
    const E_NOT_OWNER: u64 = 4;
    
    // Item struct
    struct Item has store, copy, drop {
        id: u64,
        name: String,
        quantity: u64,
        unit_price: u64,
    }
    
    // Inventory resource
    struct Inventory has key {
        items: vector<Item>,
        next_id: u64,
        owner: address,
    }
    
    // Events
    struct ItemAdded has drop, store {
        item_id: u64,
        name: String,
        quantity: u64,
    }
    
    // Initialize inventory
    public entry fun initialize(account: &signer) {
        let addr = signer::address_of(account);
        assert!(!exists<Inventory>(addr), 0);
        
        move_to(account, Inventory {
            items: vector::empty<Item>(),
            next_id: 1,
            owner: addr,
        });
    }
    
    // Add item
    public entry fun add_item(
        account: &signer,
        name: vector<u8>,
        quantity: u64,
        unit_price: u64,
    ) acquires Inventory {
        let addr = signer::address_of(account);
        let inventory = borrow_global_mut<Inventory>(addr);
        
        // Validate
        assert!(inventory.owner == addr, E_NOT_OWNER);
        assert!(quantity > 0, E_INVALID_QUANTITY);
        assert!(
            vector::length(&inventory.items) < MAX_ITEMS,
            E_INVENTORY_FULL
        );
        
        // Create new item
        let item = Item {
            id: inventory.next_id,
            name: string::utf8(name),
            quantity,
            unit_price,
        };
        
        // Update state
        vector::push_back(&mut inventory.items, item);
        inventory.next_id = inventory.next_id + 1;
    }
    
    // Update quantity
    public entry fun update_quantity(
        account: &signer,
        item_id: u64,
        new_quantity: u64,
    ) acquires Inventory {
        let addr = signer::address_of(account);
        let inventory = borrow_global_mut<Inventory>(addr);
        
        assert!(inventory.owner == addr, E_NOT_OWNER);
        
        let i = find_item_index(&inventory.items, item_id);
        let item = vector::borrow_mut(&mut inventory.items, i);
        item.quantity = new_quantity;
    }
    
    // Remove item
    public entry fun remove_item(
        account: &signer,
        item_id: u64,
    ) acquires Inventory {
        let addr = signer::address_of(account);
        let inventory = borrow_global_mut<Inventory>(addr);
        
        assert!(inventory.owner == addr, E_NOT_OWNER);
        
        let i = find_item_index(&inventory.items, item_id);
        vector::remove(&mut inventory.items, i);
    }
    
    // Helper: find item index by id
    fun find_item_index(items: &vector<Item>, item_id: u64): u64 {
        let i = 0u64;
        let len = vector::length(items);
        
        while (i < len) {
            if (vector::borrow(items, i).id == item_id) {
                return i
            };
            i = i + 1;
        };
        
        abort E_ITEM_NOT_FOUND
    }
    
    // View: Get item count
    #[view]
    public fun get_item_count(addr: address): u64 acquires Inventory {
        vector::length(&borrow_global<Inventory>(addr).items)
    }
    
    // View: Get total value
    #[view]
    public fun get_total_value(addr: address): u64 acquires Inventory {
        let inventory = borrow_global<Inventory>(addr);
        let total = 0u64;
        let i = 0u64;
        let len = vector::length(&inventory.items);
        
        while (i < len) {
            let item = vector::borrow(&inventory.items, i);
            total = total + item.quantity * item.unit_price;
            i = i + 1;
        };
        
        total
    }
    
    // Tests
    #[test(account = @0xCAFE)]
    public fun test_inventory(account: &signer) acquires Inventory {
        initialize(account);
        
        let addr = signer::address_of(account);
        assert!(get_item_count(addr) == 0, 0);
        
        add_item(account, b"Widget", 100, 500);
        add_item(account, b"Gadget", 50, 1000);
        
        assert!(get_item_count(addr) == 2, 1);
        assert!(get_total_value(addr) == 100*500 + 50*1000, 2);
        
        update_quantity(account, 1, 200);  // update Widget
        assert!(get_total_value(addr) == 200*500 + 50*1000, 3);
        
        remove_item(account, 2);  // remove Gadget
        assert!(get_item_count(addr) == 1, 4);
    }
}
```

---

## สรุป

| แนวคิด | สิ่งที่เรียนรู้ |
|-------|--------------|
| Variables | `let`, type inference, explicit types |
| Mutability | `let mut`, compound assignment |
| Destructuring | tuple, struct pattern binding |
| Constants | `const`, naming conventions |
| Global Storage | `move_to`, `borrow_global`, `move_from` |
| Move Semantics | copy vs move, borrowing |

---

## แบบฝึกหัด

### Exercise 4.1: Mutability Practice
สร้างฟังก์ชัน `rotate_vector(v: &mut vector<u64>, n: u64)` ที่ rotate elements ไปทางซ้าย n ครั้ง

### Exercise 4.2: Constants Design
ออกแบบ constants สำหรับ token contract ที่มี:
- Total supply: 21 million
- Decimal places: 8
- Initial price: $0.001
- Fee: 0.25%
- Max wallet: 1% of supply

### Exercise 4.3: Inventory Enhancement
เพิ่ม features ให้ Inventory System:
1. `search_by_name(addr: address, name: vector<u8>): vector<u64>` - ค้นหา items ด้วยชื่อ
2. `get_low_stock_items(addr: address, threshold: u64): vector<u64>` - หา items ที่สต็อกน้อย
3. `apply_discount(account: &signer, item_id: u64, discount_bps: u64)` - ลดราคา

### Exercise 4.4: Move Semantics
เขียน unit tests เพื่อแสดงความเข้าใจ Move Semantics:
1. Test ที่แสดงว่า primitives ถูก copy
2. Test ที่แสดงว่า Resources ถูก move
3. Test ที่แสดงการใช้ references อย่างถูกต้อง

---

**ก่อนหน้า**: [Part 03 - พื้นฐาน Syntax และ Types](part-03-syntax-types.md)  
**ต่อไป**: [Part 05 - Functions และ Parameters →](part-05-functions.md)
