# Part 09: References และ Borrowing

## สารบัญ
- [References คืออะไร?](#references-คืออะไร)
- [Immutable References](#immutable-references)
- [Mutable References](#mutable-references)
- [Borrow Rules](#borrow-rules)
- [Reference Patterns](#reference-patterns)
- [ตัวอย่างโปรแกรม](#ตัวอย่างโปรแกรม)

---

## References คืออะไร?

Reference คือ "pointer" ไปยัง value โดยไม่ consume value นั้น

```move
module learning::references_intro {
    
    struct Counter has key {
        count: u64,
    }
    
    // ============================================
    // 3 วิธีในการ "access" value ใน Move
    // ============================================
    
    // 1. By value (MOVE/CONSUME)
    fun consume_counter(counter: Counter): u64 {
        let count = counter.count;
        // counter ถูก destroy ที่นี่
        count
    }
    
    // 2. By immutable reference (BORROW)
    fun read_counter(counter: &Counter): u64 {
        // อ่านค่า, counter ยังอยู่
        counter.count
    }
    
    // 3. By mutable reference (BORROW MUT)
    fun increment_counter(counter: &mut Counter) {
        // แก้ไขค่า, counter ยังอยู่
        counter.count = counter.count + 1;
    }
    
    // ตัวอย่างการใช้งาน
    public fun demo_references() {
        let mut counter = Counter { count: 0 };
        
        // อ่านค่าโดยไม่ consume
        let val = read_counter(&counter);
        assert!(val == 0, 0);
        
        // แก้ไขค่า
        increment_counter(&mut counter);
        increment_counter(&mut counter);
        
        // อ่านอีกครั้ง
        let new_val = read_counter(&counter);
        assert!(new_val == 2, 1);
        
        // Consume ตอนสุดท้าย
        let final_val = consume_counter(counter);
        assert!(final_val == 2, 2);
    }
}
```

---

## Immutable References

```move
module learning::immutable_refs {
    use std::vector;
    
    struct Config has key {
        fee_bps: u64,
        max_supply: u64,
        name: vector<u8>,
    }
    
    // รับ immutable reference
    public fun get_fee(config: &Config): u64 {
        config.fee_bps  // อ่าน field โดยตรง
    }
    
    // Return reference (ระวัง lifetime!)
    public fun get_name_ref(config: &Config): &vector<u8> {
        &config.name  // return reference ไปยัง field
    }
    
    // หลายๆ immutable references พร้อมกันได้
    public fun compare_configs(a: &Config, b: &Config): bool {
        a.fee_bps == b.fee_bps
    }
    
    // Dereference operator *
    public fun demo_deref() {
        let x = 42u64;
        let x_ref = &x;           // สร้าง reference
        let x_val = *x_ref;       // dereference = copy ค่า (ต้องมี copy ability)
        assert!(x_val == 42, 0);
        assert!(x == x_val, 1);   // x ยังอยู่
    }
    
    // Reference ใน vector operations
    public fun sum_vector(v: &vector<u64>): u64 {
        let total = 0u64;
        let i = 0u64;
        let len = vector::length(v);
        
        while (i < len) {
            let element_ref = vector::borrow(v, i);  // &u64
            total = total + *element_ref;              // dereference
            i = i + 1;
        };
        
        total
    }
    
    // Chaining references
    struct Inner has copy, drop {
        value: u64,
    }
    
    struct Outer has drop {
        inner: Inner,
        data: u64,
    }
    
    public fun get_inner_value(outer: &Outer): u64 {
        outer.inner.value  // access nested field through reference
    }
    
    public fun get_inner_ref(outer: &Outer): &Inner {
        &outer.inner  // return reference to nested struct
    }
}
```

---

## Mutable References

```move
module learning::mutable_refs {
    use std::vector;
    
    struct Pool has key {
        reserve_a: u64,
        reserve_b: u64,
        fee_bps: u64,
        locked: bool,
    }
    
    // รับ mutable reference
    public fun add_liquidity(pool: &mut Pool, amount_a: u64, amount_b: u64) {
        pool.reserve_a = pool.reserve_a + amount_a;
        pool.reserve_b = pool.reserve_b + amount_b;
    }
    
    public fun remove_liquidity(pool: &mut Pool, amount_a: u64, amount_b: u64) {
        assert!(pool.reserve_a >= amount_a, 1);
        assert!(pool.reserve_b >= amount_b, 2);
        pool.reserve_a = pool.reserve_a - amount_a;
        pool.reserve_b = pool.reserve_b - amount_b;
    }
    
    // แก้ไข through nested reference
    struct Nested {
        data: vector<u64>,
    }
    
    struct Container has key {
        inner: Nested,
        count: u64,
    }
    
    public fun push_to_nested(container: &mut Container, value: u64) {
        vector::push_back(&mut container.inner.data, value);
        container.count = container.count + 1;
    }
    
    // Dereference assignment
    public fun reset_value(x: &mut u64, new_val: u64) {
        *x = new_val;  // assign through mutable reference
    }
    
    public fun increment(x: &mut u64) {
        *x = *x + 1;
    }
    
    // Vector mutable operations
    public fun double_all_values(v: &mut vector<u64>) {
        let i = 0u64;
        let len = vector::length(v);
        
        while (i < len) {
            let element = vector::borrow_mut(v, i);
            *element = *element * 2;
            i = i + 1;
        };
    }
    
    // Swap elements
    public fun swap_elements(v: &mut vector<u64>, i: u64, j: u64) {
        let len = vector::length(v);
        assert!(i < len && j < len, 0);
        
        let val_i = *vector::borrow(v, i);
        let val_j = *vector::borrow(v, j);
        
        *vector::borrow_mut(v, i) = val_j;
        *vector::borrow_mut(v, j) = val_i;
    }
    
    #[test]
    fun test_mutable_refs() {
        let mut v = vector[5u64, 3u64, 1u64, 4u64, 2u64];
        
        double_all_values(&mut v);
        assert!(*vector::borrow(&v, 0) == 10, 0);
        assert!(*vector::borrow(&v, 1) == 6, 1);
        
        swap_elements(&mut v, 0, 4);
        assert!(*vector::borrow(&v, 0) == 4, 2);  // 2*2 = 4
        assert!(*vector::borrow(&v, 4) == 10, 3);
    }
}
```

---

## Borrow Rules

Move มี Borrow Checker ที่บังคับ rules เหล่านี้:

```move
module learning::borrow_rules {
    use std::vector;
    
    // Rule 1: ไม่สามารถมี mutable borrow ร่วมกับ borrow อื่น
    public fun rule_1_demo() {
        let mut v = vector[1u64, 2u64, 3u64];
        
        // ✅ หลาย immutable borrows พร้อมกันได้
        let a = vector::borrow(&v, 0);
        let b = vector::borrow(&v, 1);
        let sum = *a + *b;
        // a, b สิ้นสุด borrow ที่นี่
        
        // ✅ mutable borrow หลังจาก immutable borrows สิ้นสุด
        vector::push_back(&mut v, 4);
        
        // ❌ ไม่สามารถ mix mutable และ immutable borrows
        // let c = vector::borrow(&v, 0);    // immutable borrow
        // vector::push_back(&mut v, 4);     // ERROR: mutable borrow ไม่ได้ขณะมี immutable borrow
    }
    
    // Rule 2: Borrow ต้อง outlive โดย value ที่ borrow
    public fun rule_2_demo(): u64 {
        let result;
        {
            let x = 42u64;
            let x_ref = &x;  // borrow x
            result = *x_ref; // copy ค่าออกมา
            // x_ref (และ borrow) สิ้นสุดที่นี่
        }
        // x ถูก drop แล้ว แต่ result ยังมีค่า copy ของมัน
        result  // ✅ 42
    }
    
    // Rule 3: ไม่สามารถ return dangling reference
    // ❌ ไม่สามารถทำแบบนี้:
    // public fun bad_return(): &u64 {
    //     let x = 42u64;
    //     &x  // ERROR: x จะถูก drop ตอน return
    // }
    
    // ✅ แต่สามารถ return reference จาก parameter ได้
    public fun good_return<'a>(x: &'a u64): &'a u64 {
        // ใน Move ไม่มี explicit lifetime แต่ concept เดียวกัน
        x
    }
    
    // Rule 4: ใน global storage - borrow ต้องถูก "acquires"
    struct Resource has key { value: u64 }
    
    // ❌ ลืม acquires
    // public fun bad_borrow(addr: address): u64 {
    //     borrow_global<Resource>(addr).value  // ERROR
    // }
    
    // ✅ ต้องระบุ acquires
    public fun good_borrow(addr: address): u64 acquires Resource {
        borrow_global<Resource>(addr).value
    }
    
    // ถ้าฟังก์ชันเรียก function อื่นที่ acquires ก็ต้อง acquires ด้วย
    public fun indirect_borrow(addr: address): u64 acquires Resource {
        good_borrow(addr)  // ฟังก์ชันนี้ก็ต้อง acquires Resource
    }
}
```

---

## Reference Patterns

### Pattern 1: View Functions

```move
module learning::view_patterns {
    use std::vector;
    
    struct UserProfile has key {
        name: vector<u8>,
        level: u64,
        items: vector<u64>,
    }
    
    // Query ทีละ field
    #[view]
    public fun get_name(addr: address): vector<u8> acquires UserProfile {
        *&borrow_global<UserProfile>(addr).name  // copy ออกมา
    }
    
    #[view]
    public fun get_level(addr: address): u64 acquires UserProfile {
        borrow_global<UserProfile>(addr).level
    }
    
    // Query aggregate info
    #[view]
    public fun get_item_count(addr: address): u64 acquires UserProfile {
        vector::length(&borrow_global<UserProfile>(addr).items)
    }
    
    // Check existence
    #[view]
    public fun has_profile(addr: address): bool {
        exists<UserProfile>(addr)
    }
    
    // Compare (ต้องใช้ references เพื่อไม่ consume)
    public fun is_higher_level(addr_a: address, addr_b: address): bool acquires UserProfile {
        let level_a = get_level(addr_a);
        let level_b = get_level(addr_b);
        level_a > level_b
    }
}
```

### Pattern 2: Mutable Operations with Validation

```move
module learning::safe_operations {
    use std::signer;
    
    struct Treasury has key {
        balance: u64,
        spending_limit_daily: u64,
        spent_today: u64,
        admin: address,
    }
    
    // Safe withdraw ที่ validate ก่อน modify
    public entry fun withdraw(
        admin: &signer,
        treasury_addr: address,
        amount: u64,
    ) acquires Treasury {
        let addr = signer::address_of(admin);
        
        // ✅ อ่านก่อน validate
        {
            let treasury = borrow_global<Treasury>(treasury_addr);
            assert!(treasury.admin == addr, 1);
            assert!(amount > 0, 2);
            assert!(amount <= treasury.balance, 3);
            assert!(treasury.spent_today + amount <= treasury.spending_limit_daily, 4);
        }  // borrow สิ้นสุดที่นี่
        
        // ✅ modify หลังจาก validate ผ่าน
        let treasury = borrow_global_mut<Treasury>(treasury_addr);
        treasury.balance = treasury.balance - amount;
        treasury.spent_today = treasury.spent_today + amount;
    }
    
    // Pattern: Read-Validate-Modify
    public entry fun complex_update(
        account: &signer,
        treasury_addr: address,
        new_limit: u64,
    ) acquires Treasury {
        let addr = signer::address_of(account);
        
        // Phase 1: Read
        let current_balance = borrow_global<Treasury>(treasury_addr).balance;
        let current_admin = borrow_global<Treasury>(treasury_addr).admin;
        
        // Phase 2: Validate
        assert!(current_admin == addr, 1);
        assert!(new_limit >= current_balance / 10, 2);  // limit >= 10% of balance
        
        // Phase 3: Modify
        let treasury = borrow_global_mut<Treasury>(treasury_addr);
        treasury.spending_limit_daily = new_limit;
    }
}
```

### Pattern 3: Nested Borrow

```move
module learning::nested_borrow {
    use std::vector;
    
    struct Position {
        size: u64,
        entry_price: u64,
        is_long: bool,
    }
    
    struct Portfolio has key {
        positions: vector<Position>,
        total_value: u64,
    }
    
    // Borrow nested element
    public fun get_position_size(
        portfolio: &Portfolio,
        index: u64,
    ): u64 {
        // borrow ตัว outer แล้ว access ตัว inner
        vector::borrow(&portfolio.positions, index).size
    }
    
    // Mutable nested access
    public fun update_position_size(
        portfolio: &mut Portfolio,
        index: u64,
        new_size: u64,
    ) {
        let position = vector::borrow_mut(&mut portfolio.positions, index);
        position.size = new_size;
    }
    
    // Cannot have overlapping borrows
    public fun compute_pnl(portfolio: &Portfolio, current_price: u64): u64 {
        let total_pnl = 0u64;
        let i = 0u64;
        let len = vector::length(&portfolio.positions);
        
        while (i < len) {
            let pos = vector::borrow(&portfolio.positions, i);
            let pnl = if (pos.is_long) {
                if (current_price > pos.entry_price) {
                    pos.size * (current_price - pos.entry_price) / pos.entry_price
                } else { 0 }
            } else {
                if (current_price < pos.entry_price) {
                    pos.size * (pos.entry_price - current_price) / pos.entry_price
                } else { 0 }
            };
            total_pnl = total_pnl + pnl;
            i = i + 1;
        };
        
        total_pnl
    }
    
    #[test]
    fun test_nested_borrow() {
        let positions = vector[
            Position { size: 100, entry_price: 1000, is_long: true },
            Position { size: 50, entry_price: 2000, is_long: false },
        ];
        
        let mut portfolio = Portfolio {
            positions,
            total_value: 150,
        };
        
        assert!(get_position_size(&portfolio, 0) == 100, 0);
        
        update_position_size(&mut portfolio, 0, 200);
        assert!(get_position_size(&portfolio, 0) == 200, 1);
        
        // PnL: long pos: price went up 10% -> pnl = 200 * 100 / 1000 = 20
        let pnl = compute_pnl(&portfolio, 1100);
        assert!(pnl == 20, 2);
    }
}
```

---

## ตัวอย่างโปรแกรม: Lending Protocol with References

```move
module learning::lending {
    use std::signer;
    use std::vector;
    
    // ============================================
    // Types
    // ============================================
    
    struct Market has key {
        total_deposits: u64,
        total_borrows: u64,
        reserve_factor_bps: u64,  // ส่วนแบ่งสำหรับ protocol
        collateral_factor_bps: u64,  // LTV ratio
        interest_rate_bps: u64,  // annual rate
        last_updated: u64,
    }
    
    struct UserPosition has key {
        deposited: u64,
        borrowed: u64,
        collateral_locked: u64,
    }
    
    // ============================================
    // Constants
    // ============================================
    
    const PRECISION: u64 = 10_000;
    const E_INSUFFICIENT_COLLATERAL: u64 = 1;
    const E_INSUFFICIENT_LIQUIDITY: u64 = 2;
    const E_ZERO_AMOUNT: u64 = 3;
    const E_NOT_INITIALIZED: u64 = 4;
    
    // ============================================
    // View functions (immutable references)
    // ============================================
    
    #[view]
    public fun get_utilization_rate(market_addr: address): u64 acquires Market {
        let market = borrow_global<Market>(market_addr);
        if (market.total_deposits == 0) { return 0 };
        market.total_borrows * PRECISION / market.total_deposits
    }
    
    #[view]
    public fun get_supply_rate(market_addr: address): u64 acquires Market {
        let market = borrow_global<Market>(market_addr);
        let utilization = get_utilization_rate(market_addr);
        
        // Supply rate = borrow rate * utilization * (1 - reserve_factor)
        let borrow_rate = market.interest_rate_bps;
        borrow_rate * utilization / PRECISION 
            * (PRECISION - market.reserve_factor_bps) / PRECISION
    }
    
    #[view]
    public fun get_health_factor(
        user_addr: address,
        market_addr: address,
        current_price: u64,  // collateral price in USD
    ): u64 acquires UserPosition, Market {
        let position = borrow_global<UserPosition>(user_addr);
        let market = borrow_global<Market>(market_addr);
        
        if (position.borrowed == 0) { return PRECISION * 100 };  // healthy
        
        let collateral_value = position.collateral_locked * current_price / PRECISION;
        let max_borrow = collateral_value * market.collateral_factor_bps / PRECISION;
        
        max_borrow * PRECISION / position.borrowed
    }
    
    #[view]
    public fun is_healthy(
        user_addr: address,
        market_addr: address,
        current_price: u64,
    ): bool acquires UserPosition, Market {
        get_health_factor(user_addr, market_addr, current_price) >= PRECISION
    }
    
    // ============================================
    // State-changing functions (mutable references)
    // ============================================
    
    public entry fun deposit(
        user: &signer,
        market_addr: address,
        amount: u64,
    ) acquires Market, UserPosition {
        assert!(amount > 0, E_ZERO_AMOUNT);
        
        let user_addr = signer::address_of(user);
        
        // Update market
        {
            let market = borrow_global_mut<Market>(market_addr);
            market.total_deposits = market.total_deposits + amount;
        };
        
        // Update user position
        if (exists<UserPosition>(user_addr)) {
            let position = borrow_global_mut<UserPosition>(user_addr);
            position.deposited = position.deposited + amount;
        } else {
            move_to(user, UserPosition {
                deposited: amount,
                borrowed: 0,
                collateral_locked: 0,
            });
        }
    }
    
    public entry fun borrow_tokens(
        user: &signer,
        market_addr: address,
        amount: u64,
        collateral_price: u64,
    ) acquires Market, UserPosition {
        assert!(amount > 0, E_ZERO_AMOUNT);
        
        let user_addr = signer::address_of(user);
        assert!(exists<UserPosition>(user_addr), E_NOT_INITIALIZED);
        
        // Check ว่ามี liquidity เพียงพอ
        {
            let market = borrow_global<Market>(market_addr);
            let available = market.total_deposits - market.total_borrows;
            assert!(available >= amount, E_INSUFFICIENT_LIQUIDITY);
        };
        
        // Check collateral sufficiency
        {
            let position = borrow_global<UserPosition>(user_addr);
            let market = borrow_global<Market>(market_addr);
            
            let collateral_value = position.deposited * collateral_price / PRECISION;
            let max_borrow = collateral_value * market.collateral_factor_bps / PRECISION;
            let current_plus_new = position.borrowed + amount;
            
            assert!(current_plus_new <= max_borrow, E_INSUFFICIENT_COLLATERAL);
        };
        
        // Execute borrow
        {
            let market = borrow_global_mut<Market>(market_addr);
            market.total_borrows = market.total_borrows + amount;
        };
        
        {
            let position = borrow_global_mut<UserPosition>(user_addr);
            position.borrowed = position.borrowed + amount;
            position.collateral_locked = position.collateral_locked + amount;
        }
    }
    
    public entry fun repay(
        user: &signer,
        market_addr: address,
        amount: u64,
    ) acquires Market, UserPosition {
        assert!(amount > 0, E_ZERO_AMOUNT);
        
        let user_addr = signer::address_of(user);
        assert!(exists<UserPosition>(user_addr), E_NOT_INITIALIZED);
        
        // แก้ไข user position
        let actual_repay;
        {
            let position = borrow_global_mut<UserPosition>(user_addr);
            actual_repay = if (amount > position.borrowed) {
                position.borrowed  // repay ทั้งหมดที่เป็นหนี้
            } else {
                amount
            };
            
            position.borrowed = position.borrowed - actual_repay;
            position.collateral_locked = if (position.collateral_locked >= actual_repay) {
                position.collateral_locked - actual_repay
            } else { 0 };
        };
        
        // แก้ไข market
        {
            let market = borrow_global_mut<Market>(market_addr);
            market.total_borrows = market.total_borrows - actual_repay;
        }
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test(admin = @0x1, user = @0x2)]
    public fun test_lending_lifecycle(
        admin: &signer,
        user: &signer,
    ) acquires Market, UserPosition {
        let market_addr = signer::address_of(admin);
        let user_addr = signer::address_of(user);
        
        // Setup market
        move_to(admin, Market {
            total_deposits: 0,
            total_borrows: 0,
            reserve_factor_bps: 1000,    // 10%
            collateral_factor_bps: 7500,  // 75% LTV
            interest_rate_bps: 800,       // 8% APR
            last_updated: 0,
        });
        
        // Deposit
        deposit(user, market_addr, 10_000);
        assert!(borrow_global<Market>(market_addr).total_deposits == 10_000, 0);
        assert!(borrow_global<UserPosition>(user_addr).deposited == 10_000, 1);
        
        // Borrow (price = 1.0 = PRECISION)
        let price = PRECISION;
        borrow_tokens(user, market_addr, 7_000, price);  // 70% LTV (under 75% limit)
        
        assert!(borrow_global<Market>(market_addr).total_borrows == 7_000, 2);
        assert!(borrow_global<UserPosition>(user_addr).borrowed == 7_000, 3);
        assert!(is_healthy(user_addr, market_addr, price) == true, 4);
        
        // Repay
        repay(user, market_addr, 7_000);
        assert!(borrow_global<UserPosition>(user_addr).borrowed == 0, 5);
    }
}
```

---

## สรุป

| แนวคิด | Syntax | การใช้งาน |
|-------|--------|---------|
| Immutable Reference | `&T` | อ่านค่าโดยไม่ consume |
| Mutable Reference | `&mut T` | แก้ไขค่าโดยไม่ consume |
| Dereference | `*ref` | ได้ค่าจาก reference |
| Borrow | `&value` | สร้าง immutable reference |
| Borrow Mut | `&mut value` | สร้าง mutable reference |
| Global Borrow | `borrow_global<T>(addr)` | อ่าน global resource |
| Global Borrow Mut | `borrow_global_mut<T>(addr)` | แก้ไข global resource |

---

## แบบฝึกหัด

### Exercise 9.1: Reference Practice
เขียนฟังก์ชันที่รับ `&vector<u64>` และ return:
1. ค่าสูงสุด
2. ค่าต่ำสุด
3. ค่า median
4. Standard deviation (ประมาณ)

### Exercise 9.2: Mutable Reference
เขียน in-place sorting algorithms:
1. Bubble sort (`&mut vector<u64>`)
2. Insertion sort (`&mut vector<u64>`)

### Exercise 9.3: Lending Enhancement
เพิ่ม features ให้ Lending Protocol:
1. `liquidate(liquidator: &signer, borrower: address, ...)` - ถ้า health factor < 1
2. `withdraw_deposit(user: &signer, amount: u64, ...)` - ถอน deposits
3. View: `get_apy(market_addr: address): u64` - คำนวณ APY

---

**ก่อนหน้า**: [Part 08 - Resources](part-08-resources.md)  
**ต่อไป**: [Part 10 - Structs →](part-10-structs.md)
