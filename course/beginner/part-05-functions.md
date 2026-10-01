# Part 05: Functions และ Parameters

## สารบัญ
- [Function Declarations](#function-declarations)
- [Parameters และ Return Values](#parameters-และ-return-values)
- [Function Visibility](#function-visibility)
- [Entry Functions](#entry-functions)
- [Generic Functions](#generic-functions)
- [Inline Functions](#inline-functions)
- [Higher-order Patterns](#higher-order-patterns)
- [Best Practices](#best-practices)
- [ตัวอย่างโปรแกรม](#ตัวอย่างโปรแกรม)

---

## Function Declarations

```move
module learning::function_basics {
    
    // ฟังก์ชันพื้นฐาน
    fun basic_function() {
        // ไม่ต้อง return อะไร
    }
    
    // ฟังก์ชันที่ return ค่า
    fun return_value(): u64 {
        42
    }
    
    // ฟังก์ชันที่มี parameters
    fun with_params(a: u64, b: u64): u64 {
        a + b
    }
    
    // ฟังก์ชันที่ return หลายค่า (tuple)
    fun multiple_returns(n: u64): (u64, bool) {
        (n * 2, n > 10)
    }
    
    // ฟังก์ชันแบบ public
    public fun public_function(x: u64): u64 {
        x * 3
    }
    
    // ฟังก์ชัน entry (เรียกจาก transaction)
    public entry fun transaction_function(account: &signer) {
        // ทำงานบน blockchain
    }
    
    // ฟังก์ชัน view (อ่านข้อมูลเท่านั้น)
    #[view]
    public fun query_function(addr: address): u64 {
        0  // placeholder
    }
}
```

---

## Parameters และ Return Values

```move
module learning::parameters {
    use std::signer;
    use std::vector;
    
    // Parameter passing modes
    
    // 1. By value (consume/move)
    fun consume_vector(v: vector<u64>): u64 {
        // v ถูก move เข้ามา, caller ไม่สามารถใช้ v ได้แล้ว
        let sum = 0u64;
        let i = 0u64;
        let len = vector::length(&v);
        while (i < len) {
            sum = sum + *vector::borrow(&v, i);
            i = i + 1;
        };
        // v ถูก drop ตอนออกจากฟังก์ชัน
        sum
    }
    
    // 2. By immutable reference
    fun borrow_vector(v: &vector<u64>): u64 {
        // v ถูก borrow (ไม่ consume)
        // caller ยังสามารถใช้ v ได้หลังจากนี้
        let len = vector::length(v);
        if (len == 0) { 0 } else { *vector::borrow(v, 0) }
    }
    
    // 3. By mutable reference
    fun modify_vector(v: &mut vector<u64>, value: u64) {
        // v ถูก borrow แบบ mutable
        vector::push_back(v, value);
    }
    
    // ตัวอย่างการใช้
    public fun parameter_demo() {
        let mut v = vector[1u64, 2u64, 3u64];
        
        let first = borrow_vector(&v);       // immutable borrow - v ยังใช้ได้
        modify_vector(&mut v, 4);            // mutable borrow - v ยังใช้ได้
        let sum = consume_vector(v);         // consume - v ไม่สามารถใช้ได้แล้ว
        // let _ = borrow_vector(&v);        // ❌ ERROR: v was moved
    }
    
    // Multiple parameters
    public fun transfer(
        sender: &signer,
        recipient: address,
        amount: u64,
        memo: vector<u8>,
    ) {
        // ...
    }
    
    // Return multiple values
    public fun div_mod(a: u64, b: u64): (u64, u64) {
        assert!(b != 0, 0);
        (a / b, a % b)
    }
    
    // ใช้ return values
    public fun use_div_mod() {
        let (quotient, remainder) = div_mod(17, 5);
        // quotient = 3, remainder = 2
        
        // ทิ้งค่าที่ไม่ต้องการ
        let (q, _) = div_mod(100, 7);
    }
    
    // Optional-like return (ใช้ tuple)
    public fun safe_div(a: u64, b: u64): (u64, bool) {
        if (b == 0) {
            (0, false)  // ล้มเหลว
        } else {
            (a / b, true)  // สำเร็จ
        }
    }
    
    // Signer parameter
    public fun get_caller_address(caller: &signer): address {
        signer::address_of(caller)
    }
}
```

---

## Function Visibility

```move
module learning::visibility_demo {
    use std::signer;
    
    // 1. private (default) - ใช้ได้แค่ภายใน module นี้
    fun private_helper(x: u64): u64 {
        x * 2
    }
    
    // 2. public - ใช้ได้จากทุก module
    public fun public_compute(x: u64): u64 {
        private_helper(x) + 1
    }
    
    // 3. public(friend) - ใช้ได้จาก module ที่ declare เป็น friend
    friend learning::friend_module;
    
    public(friend) fun friend_only_function(): u64 {
        42
    }
    
    // 4. public entry - เรียกได้จาก transaction
    public entry fun entry_function(account: &signer) {
        // เรียกได้จาก:
        // - User transaction
        // - Script
        // ไม่ได้รับ return value
    }
    
    // View function - pure read
    #[view]
    public fun view_function(addr: address): u64 {
        0
    }
}

// Friend module - สามารถเรียก public(friend) ได้
module learning::friend_module {
    use learning::visibility_demo;
    
    public fun use_friend_function(): u64 {
        visibility_demo::friend_only_function()
    }
}
```

### Visibility Summary

```
Visibility     | Same Module | Friend Module | Any Module | Transaction
─────────────────────────────────────────────────────────────────────────
private        |     ✅      |      ❌       |     ❌     |     ❌
public(friend) |     ✅      |      ✅       |     ❌     |     ❌
public         |     ✅      |      ✅       |     ✅     |     ❌
public entry   |     ✅      |      ✅       |     ✅     |     ✅
```

---

## Entry Functions

Entry functions คือ "doorway" เข้าสู่ blockchain

```move
module learning::entry_functions {
    use std::signer;
    
    struct UserProfile has key {
        name: vector<u8>,
        level: u64,
        experience: u64,
    }
    
    // Entry function: สร้าง profile ใหม่
    public entry fun create_profile(
        account: &signer,
        name: vector<u8>,
    ) {
        let addr = signer::address_of(account);
        assert!(!exists<UserProfile>(addr), 1);
        
        move_to(account, UserProfile {
            name,
            level: 1,
            experience: 0,
        });
    }
    
    // Entry function: อัปเดต experience
    public entry fun gain_experience(
        account: &signer,
        amount: u64,
    ) acquires UserProfile {
        let addr = signer::address_of(account);
        let profile = borrow_global_mut<UserProfile>(addr);
        
        profile.experience = profile.experience + amount;
        
        // Level up system
        let new_level = calculate_level(profile.experience);
        if (new_level > profile.level) {
            profile.level = new_level;
        };
    }
    
    // Entry function สำหรับ admin
    public entry fun set_level(
        admin: &signer,
        target: address,
        new_level: u64,
    ) acquires UserProfile {
        // ตรวจสอบว่าเป็น admin
        assert!(signer::address_of(admin) == @0x1, 100);
        
        let profile = borrow_global_mut<UserProfile>(target);
        profile.level = new_level;
    }
    
    fun calculate_level(experience: u64): u64 {
        // Level formula: level = sqrt(experience / 100) + 1
        // ใช้ approximation เพราะ Move ไม่มี sqrt
        if (experience < 100) { 1 }
        else if (experience < 400) { 2 }
        else if (experience < 900) { 3 }
        else if (experience < 1600) { 4 }
        else if (experience < 2500) { 5 }
        else { 6 + (experience - 2500) / 1000 }
    }
    
    #[view]
    public fun get_level(addr: address): u64 acquires UserProfile {
        borrow_global<UserProfile>(addr).level
    }
    
    #[view]
    public fun get_experience(addr: address): u64 acquires UserProfile {
        borrow_global<UserProfile>(addr).experience
    }
}
```

---

## Generic Functions

```move
module learning::generics {
    use std::vector;
    
    // Generic function ที่ทำงานกับ type ใดก็ได้ที่มี drop ability
    public fun first_element<T: copy>(v: &vector<T>): T {
        assert!(!vector::is_empty(v), 0);
        *vector::borrow(v, 0)
    }
    
    // Generic swap
    public fun swap<T>(a: &mut T, b: &mut T) {
        // ใน Move ไม่สามารถ swap แบบตรงๆ ได้เหมือน Rust
        // ต้องใช้วิธีอื่น
        // (limitation ของ Move's borrow checker)
    }
    
    // Generic filter
    public fun filter_by_value<T: copy + drop>(
        v: &vector<T>,
        predicate: &vector<bool>,  // mask
    ): vector<T> {
        assert!(vector::length(v) == vector::length(predicate), 0);
        
        let result = vector::empty<T>();
        let i = 0u64;
        let len = vector::length(v);
        
        while (i < len) {
            if (*vector::borrow(predicate, i)) {
                vector::push_back(&mut result, *vector::borrow(v, i));
            };
            i = i + 1;
        };
        
        result
    }
    
    // Generic map (transform)
    // Note: Move ไม่มี closures แต่สามารถ simulate ได้ด้วย patterns
    
    // Generic container
    struct Box<T> has key, store {
        value: T,
    }
    
    public fun create_box<T: store>(value: T): Box<T> {
        Box { value }
    }
    
    public fun unbox<T: store>(box: Box<T>): T {
        let Box { value } = box;
        value
    }
    
    public fun peek<T: store>(box: &Box<T>): &T {
        &box.value
    }
    
    // Generic pair
    struct Pair<A, B> has copy, drop, store {
        first: A,
        second: B,
    }
    
    public fun make_pair<A: copy + drop + store, B: copy + drop + store>(
        first: A,
        second: B,
    ): Pair<A, B> {
        Pair { first, second }
    }
    
    public fun get_first<A: copy + drop + store, B: copy + drop + store>(
        pair: &Pair<A, B>
    ): A {
        pair.first
    }
    
    public fun swap_pair<A: copy + drop + store, B: copy + drop + store>(
        pair: Pair<A, B>
    ): Pair<B, A> {
        Pair {
            first: pair.second,
            second: pair.first,
        }
    }
    
    // ตัวอย่างการใช้ generics
    #[test]
    public fun test_generics() {
        let v = vector[1u64, 2u64, 3u64];
        assert!(first_element(&v) == 1, 0);
        
        let box = create_box(42u64);
        let val = unbox(box);
        assert!(val == 42, 1);
        
        let pair = make_pair(10u64, true);
        assert!(get_first(&pair) == 10, 2);
        
        let swapped = swap_pair(pair);
        assert!(swapped.first == true, 3);
        assert!(swapped.second == 10, 4);
    }
}
```

---

## Inline Functions

```move
module learning::inline_functions {
    
    // Inline function - compiler จะ inline code ณ call site
    // ประโยชน์: ลด overhead ของ function call, ดีสำหรับ hot paths
    inline fun is_even(n: u64): bool {
        n % 2 == 0
    }
    
    inline fun square(n: u64): u64 {
        n * n
    }
    
    // Inline ที่ใช้บ่อย
    inline fun min(a: u64, b: u64): u64 {
        if (a < b) { a } else { b }
    }
    
    inline fun max(a: u64, b: u64): u64 {
        if (a > b) { a } else { b }
    }
    
    inline fun abs_diff(a: u64, b: u64): u64 {
        if (a > b) { a - b } else { b - a }
    }
    
    // ใช้ inline functions
    public fun process_numbers(v: &vector<u64>): u64 {
        let sum_of_squares = 0u64;
        let i = 0u64;
        let len = vector::length(v);
        
        while (i < len) {
            let val = *vector::borrow(v, i);
            if (is_even(val)) {
                sum_of_squares = sum_of_squares + square(val);
            };
            i = i + 1;
        };
        
        sum_of_squares
    }
    
    // Inline function ที่รับ lambda-like behavior (Aptos specific)
    // ใน Aptos Move รองรับ inline functions ที่รับ function parameters
    inline fun for_each<T>(
        v: &vector<T>,
        f: |&T|,  // function parameter (Aptos extension)
    ) {
        let i = 0;
        let len = vector::length(v);
        while (i < len) {
            f(vector::borrow(v, i));
            i = i + 1;
        }
    }
    
    // ใช้ inline ที่รับ function
    #[test]
    public fun test_inline_with_lambda() {
        let v = vector[1u64, 2u64, 3u64, 4u64, 5u64];
        let sum = 0u64;
        
        for_each(&v, |x| {
            sum = sum + *x;
        });
        
        assert!(sum == 15, 0);
    }
}

use std::vector;
```

---

## Higher-order Patterns

Move ไม่มี first-class functions เหมือน functional languages แต่เราสามารถ simulate ได้

```move
module learning::higher_order {
    use std::vector;
    
    // Pattern 1: Strategy pattern ด้วย enum-like approach
    const STRATEGY_SUM: u8 = 0;
    const STRATEGY_PRODUCT: u8 = 1;
    const STRATEGY_MAX: u8 = 2;
    
    public fun reduce(v: &vector<u64>, strategy: u8): u64 {
        let len = vector::length(v);
        assert!(len > 0, 0);
        
        let result = *vector::borrow(v, 0);
        let i = 1u64;
        
        while (i < len) {
            let val = *vector::borrow(v, i);
            result = if (strategy == STRATEGY_SUM) {
                result + val
            } else if (strategy == STRATEGY_PRODUCT) {
                result * val
            } else {  // STRATEGY_MAX
                if (val > result) { val } else { result }
            };
            i = i + 1;
        };
        
        result
    }
    
    // Pattern 2: Builder pattern
    struct QueryBuilder {
        filters: vector<u64>,    // filter values
        limit: u64,
        offset: u64,
        ascending: bool,
    }
    
    public fun new_query(): QueryBuilder {
        QueryBuilder {
            filters: vector::empty<u64>(),
            limit: 10,
            offset: 0,
            ascending: true,
        }
    }
    
    public fun with_filter(mut builder: QueryBuilder, filter: u64): QueryBuilder {
        vector::push_back(&mut builder.filters, filter);
        builder
    }
    
    public fun with_limit(mut builder: QueryBuilder, limit: u64): QueryBuilder {
        builder.limit = limit;
        builder
    }
    
    public fun descending(mut builder: QueryBuilder): QueryBuilder {
        builder.ascending = false;
        builder
    }
    
    // ใช้ Builder pattern
    public fun example_query(): QueryBuilder {
        let query = new_query();
        let query = with_filter(query, 100);
        let query = with_filter(query, 200);
        let query = with_limit(query, 50);
        let query = descending(query);
        query
    }
    
    // Pattern 3: Visitor pattern
    struct DataProcessor {
        multiply_factor: u64,
        add_offset: u64,
    }
    
    public fun process_all(
        data: &vector<u64>,
        processor: &DataProcessor,
    ): vector<u64> {
        let result = vector::empty<u64>();
        let i = 0u64;
        let len = vector::length(data);
        
        while (i < len) {
            let val = *vector::borrow(data, i);
            let processed = val * processor.multiply_factor + processor.add_offset;
            vector::push_back(&mut result, processed);
            i = i + 1;
        };
        
        result
    }
}
```

---

## Best Practices

### 1. ฟังก์ชันควรทำแค่สิ่งเดียว (Single Responsibility)

```move
// ❌ ไม่ดี: ทำหลายอย่างในฟังก์ชันเดียว
public entry fun bad_create_and_transfer(
    account: &signer,
    recipient: address,
    amount: u64,
    name: vector<u8>,
) {
    // สร้าง token
    // ตรวจสอบ KYC
    // คำนวณ fee
    // transfer
    // emit event
    // อัปเดต stats
    // ...ทำหลายอย่างเกินไป
}

// ✅ ดี: แยกฟังก์ชัน
public entry fun create_token(account: &signer, name: vector<u8>) { ... }
public entry fun transfer_token(sender: &signer, recipient: address, amount: u64) { ... }
fun calculate_fee(amount: u64): u64 { ... }
fun emit_transfer_event(from: address, to: address, amount: u64) { ... }
```

### 2. Naming Conventions

```move
// ✅ ชื่อฟังก์ชัน: snake_case
public fun calculate_total_fee(amount: u64): u64 { ... }

// ✅ Entry functions: verb + noun
public entry fun create_pool() { ... }
public entry fun add_liquidity() { ... }
public entry fun remove_liquidity() { ... }
public entry fun swap_tokens() { ... }

// ✅ View functions: get_ prefix
#[view]
public fun get_pool_balance() { ... }
#[view]
public fun get_user_position() { ... }

// ✅ Helper functions: descriptive
fun is_valid_amount(amount: u64): bool { ... }
fun compute_price_impact(amount_in: u64, reserve: u64): u64 { ... }
```

### 3. Error Handling

```move
// ✅ ดี: ตรวจสอบทุก precondition ก่อนทำงาน
public entry fun safe_transfer(
    sender: &signer,
    recipient: address,
    amount: u64,
) acquires Balance {
    // Validate inputs ก่อน
    assert!(amount > 0, E_ZERO_AMOUNT);
    assert!(recipient != @0x0, E_INVALID_ADDRESS);
    
    let sender_addr = signer::address_of(sender);
    assert!(sender_addr != recipient, E_SELF_TRANSFER);
    
    let balance = borrow_global<Balance>(sender_addr);
    assert!(balance.amount >= amount, E_INSUFFICIENT_BALANCE);
    
    // ทำงานจริงหลังผ่าน validation ทั้งหมด
    do_transfer(sender_addr, recipient, amount);
}
```

### 4. Documentation

```move
/// คำนวณ output amount สำหรับ swap
/// 
/// ใช้ constant product formula: x * y = k
/// output = (amount_in * fee_adjusted * reserve_out) / 
///          (reserve_in * 10000 + amount_in * fee_adjusted)
///
/// # Arguments
/// * `amount_in` - จำนวน token ที่ต้องการ swap (denominated ใน smallest unit)
/// * `reserve_in` - จำนวน token ที่มีใน pool ฝั่ง input
/// * `reserve_out` - จำนวน token ที่มีใน pool ฝั่ง output
/// * `fee_bps` - fee ใน basis points (e.g., 30 = 0.30%)
///
/// # Returns
/// จำนวน output token ที่ได้รับ
///
/// # Aborts
/// * `E_ZERO_RESERVES` - ถ้า reserve ใดเป็น 0
/// * `E_ZERO_AMOUNT` - ถ้า amount_in เป็น 0
public fun get_amount_out(
    amount_in: u64,
    reserve_in: u64,
    reserve_out: u64,
    fee_bps: u64,
): u64 {
    assert!(amount_in > 0, E_ZERO_AMOUNT);
    assert!(reserve_in > 0 && reserve_out > 0, E_ZERO_RESERVES);
    
    let fee_adjusted = 10000 - fee_bps;
    let amount_in_with_fee = amount_in * fee_adjusted;
    let numerator = amount_in_with_fee * reserve_out;
    let denominator = reserve_in * 10000 + amount_in_with_fee;
    
    numerator / denominator
}
```

---

## ตัวอย่างโปรแกรม: AMM (Automated Market Maker) Core Logic

```move
module learning::amm_core {
    
    // Constants
    const FEE_DENOMINATOR: u64 = 10_000;
    const MIN_LIQUIDITY: u64 = 1_000;
    
    // Errors
    const E_ZERO_AMOUNT: u64 = 1;
    const E_ZERO_RESERVES: u64 = 2;
    const E_SLIPPAGE: u64 = 3;
    const E_INSUFFICIENT_LIQUIDITY: u64 = 4;
    const E_OVERFLOW: u64 = 5;
    
    // คำนวณ output ด้วย constant product formula
    public fun get_amount_out(
        amount_in: u64,
        reserve_in: u64,
        reserve_out: u64,
        fee_bps: u64,
    ): u64 {
        assert!(amount_in > 0, E_ZERO_AMOUNT);
        assert!(reserve_in > 0 && reserve_out > 0, E_ZERO_RESERVES);
        
        // คำนวณด้วย u128 เพื่อป้องกัน overflow
        let amount_in_u128 = amount_in as u128;
        let reserve_in_u128 = reserve_in as u128;
        let reserve_out_u128 = reserve_out as u128;
        let fee_adjusted_u128 = (FEE_DENOMINATOR - fee_bps) as u128;
        let fee_denom_u128 = FEE_DENOMINATOR as u128;
        
        let amount_in_with_fee = amount_in_u128 * fee_adjusted_u128;
        let numerator = amount_in_with_fee * reserve_out_u128;
        let denominator = reserve_in_u128 * fee_denom_u128 + amount_in_with_fee;
        
        assert!(denominator > 0, E_ZERO_RESERVES);
        (numerator / denominator) as u64
    }
    
    // คำนวณ input ที่ต้องใช้เพื่อได้ output ตามต้องการ
    public fun get_amount_in(
        amount_out: u64,
        reserve_in: u64,
        reserve_out: u64,
        fee_bps: u64,
    ): u64 {
        assert!(amount_out > 0, E_ZERO_AMOUNT);
        assert!(reserve_in > 0 && reserve_out > 0, E_ZERO_RESERVES);
        assert!(amount_out < reserve_out, E_INSUFFICIENT_LIQUIDITY);
        
        let amount_out_u128 = amount_out as u128;
        let reserve_in_u128 = reserve_in as u128;
        let reserve_out_u128 = reserve_out as u128;
        let fee_adjusted_u128 = (FEE_DENOMINATOR - fee_bps) as u128;
        let fee_denom_u128 = FEE_DENOMINATOR as u128;
        
        let numerator = reserve_in_u128 * amount_out_u128 * fee_denom_u128;
        let denominator = (reserve_out_u128 - amount_out_u128) * fee_adjusted_u128;
        
        ((numerator / denominator) + 1) as u64  // round up
    }
    
    // คำนวณ liquidity shares เมื่อ add liquidity
    public fun calculate_liquidity(
        amount_a: u64,
        amount_b: u64,
        reserve_a: u64,
        reserve_b: u64,
        total_supply: u64,
    ): u64 {
        if (total_supply == 0) {
            // Initial liquidity: geometric mean - MIN_LIQUIDITY
            let product = (amount_a as u128) * (amount_b as u128);
            let sqrt_product = sqrt_u128(product);
            assert!(sqrt_product > MIN_LIQUIDITY as u128, E_INSUFFICIENT_LIQUIDITY);
            (sqrt_product - MIN_LIQUIDITY as u128) as u64
        } else {
            // Proportional liquidity
            let liquidity_a = (amount_a as u128) * (total_supply as u128) / (reserve_a as u128);
            let liquidity_b = (amount_b as u128) * (total_supply as u128) / (reserve_b as u128);
            
            // ใช้ค่าน้อยสุด
            let min_liq = if (liquidity_a < liquidity_b) { liquidity_a } else { liquidity_b };
            min_liq as u64
        }
    }
    
    // คำนวณ optimal amounts สำหรับ add liquidity
    public fun quote(
        amount_a: u64,
        reserve_a: u64,
        reserve_b: u64,
    ): u64 {
        assert!(amount_a > 0, E_ZERO_AMOUNT);
        assert!(reserve_a > 0 && reserve_b > 0, E_ZERO_RESERVES);
        
        ((amount_a as u128) * (reserve_b as u128) / (reserve_a as u128)) as u64
    }
    
    // คำนวณ price impact
    public fun get_price_impact_bps(
        amount_in: u64,
        reserve_in: u64,
    ): u64 {
        // Price impact = amount_in / (reserve_in + amount_in) * 10000
        ((amount_in as u128) * 10000u128 / ((reserve_in + amount_in) as u128)) as u64
    }
    
    // Sqrt function (integer)
    fun sqrt_u128(n: u128): u128 {
        if (n == 0) return 0;
        
        let mut x = n;
        let mut y = (x + 1) / 2;
        
        while (y < x) {
            x = y;
            y = (x + n / x) / 2;
        };
        
        x
    }
    
    // Tests
    #[test]
    fun test_get_amount_out() {
        // Swap 1000 in, reserves 10000/10000, fee 0.30%
        let out = get_amount_out(1000, 10000, 10000, 30);
        // Expected: ~906 (constant product with fee)
        assert!(out > 900 && out < 910, 0);
        
        // ไม่มี fee
        let out_no_fee = get_amount_out(1000, 10000, 10000, 0);
        // Expected: ~909
        assert!(out_no_fee > 905 && out_no_fee < 915, 1);
        
        // out_no_fee ควรมากกว่า out (เพราะไม่มี fee)
        assert!(out_no_fee > out, 2);
    }
    
    #[test]
    fun test_get_amount_in() {
        // ต้องการ output 900, reserves 10000/10000, fee 0.30%
        let input_needed = get_amount_in(900, 10000, 10000, 30);
        // เอา input ที่ได้มา swap จะต้องได้ output >= 900
        let actual_out = get_amount_out(input_needed, 10000, 10000, 30);
        assert!(actual_out >= 900, 0);
    }
    
    #[test]
    fun test_quote() {
        // ถ้าใส่ 1000 token A, ต้องใส่กี่ token B?
        // reserves: A=5000, B=10000 (ratio 1:2)
        let amount_b = quote(1000, 5000, 10000);
        assert!(amount_b == 2000, 0);  // ratio 1:2
    }
    
    #[test]
    fun test_sqrt() {
        assert!(sqrt_u128(0) == 0, 0);
        assert!(sqrt_u128(1) == 1, 1);
        assert!(sqrt_u128(4) == 2, 2);
        assert!(sqrt_u128(9) == 3, 3);
        assert!(sqrt_u128(100) == 10, 4);
        assert!(sqrt_u128(10000) == 100, 5);
    }
}
```

---

## สรุป

| แนวคิด | สิ่งที่เรียนรู้ |
|-------|--------------|
| Declarations | `fun`, `public fun`, `public entry fun` |
| Parameters | by value, `&T`, `&mut T` |
| Return Values | single, multiple (tuple) |
| Visibility | private, public, public(friend), public entry |
| Generics | `<T: ability>` constraints |
| Inline | `inline fun` สำหรับ performance |
| Best Practices | naming, documentation, error handling |

---

## แบบฝึกหัด

### Exercise 5.1: Function Design
ออกแบบและเขียน function signatures สำหรับ:
1. Token minting
2. Token burning  
3. Token transfer
4. Staking
5. Unstaking

### Exercise 5.2: Generic Functions
เขียน generic functions:
1. `contains<T: drop>(v: &vector<T>, item: &T): bool` - ด้วย equality check
2. `partition<T: drop + copy>(v: &vector<T>, predicate_values: &vector<bool>): (vector<T>, vector<T>)`

### Exercise 5.3: AMM Enhancement
เพิ่มฟังก์ชันใน AMM core:
1. `get_spot_price(reserve_a: u64, reserve_b: u64): u64` - คำนวณ spot price (scaled)
2. `get_tvl(reserve_a: u64, reserve_b: u64, price_a: u64, price_b: u64): u64`
3. `calculate_impermanent_loss_bps(initial_price_ratio: u64, current_price_ratio: u64): u64`

---

**ก่อนหน้า**: [Part 04 - Variables และ Mutability](part-04-variables.md)  
**ต่อไป**: [Part 06 - Control Flow →](part-06-control-flow.md)
