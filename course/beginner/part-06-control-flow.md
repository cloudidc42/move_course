# Part 06: Control Flow (if/while/loop)

## สารบัญ
- [If Expressions](#if-expressions)
- [Loop](#loop)
- [While Loop](#while-loop)
- [For Loop Pattern](#for-loop-pattern)
- [Break และ Continue](#break-และ-continue)
- [Abort และ Assert](#abort-และ-assert)
- [Pattern Matching](#pattern-matching)
- [ตัวอย่างโปรแกรม: State Machine](#ตัวอย่างโปรแกรม-state-machine)

---

## If Expressions

ใน Move `if` เป็น **expression** ที่มีค่า ไม่ใช่แค่ statement

```move
module learning::if_expressions {
    
    public fun basic_if() {
        let x = 10u64;
        
        // if statement
        if (x > 5) {
            // ทำงานเมื่อ condition เป็น true
        };
        
        // if-else statement  
        if (x > 5) {
            // true branch
        } else {
            // false branch
        };
    }
    
    public fun if_as_expression(): u64 {
        let x = 10u64;
        
        // if เป็น expression - return ค่า
        let result = if (x > 5) { 100u64 } else { 0u64 };
        
        // ใช้ใน return
        if (x % 2 == 0) { x / 2 } else { x * 3 + 1 }
    }
    
    // if-else if chain
    public fun classify_number(n: u64): u8 {
        if (n == 0) {
            0u8  // zero
        } else if (n < 10) {
            1u8  // single digit
        } else if (n < 100) {
            2u8  // double digit
        } else if (n < 1000) {
            3u8  // triple digit
        } else {
            4u8  // large
        }
    }
    
    // Nested if
    public fun nested_conditions(a: u64, b: u64, c: u64): u64 {
        if (a > b) {
            if (a > c) {
                a  // a เป็นค่ามากสุด
            } else {
                c  // c เป็นค่ามากสุด
            }
        } else {
            if (b > c) {
                b  // b เป็นค่ามากสุด
            } else {
                c  // c เป็นค่ามากสุด
            }
        }
    }
    
    // if expression ใน argument
    public fun conditional_fee(amount: u64, is_vip: bool): u64 {
        let fee_bps = if (is_vip) { 10u64 } else { 30u64 };
        amount * fee_bps / 10_000
    }
    
    // Complex conditions
    public fun is_valid_transaction(
        amount: u64,
        sender_balance: u64,
        is_paused: bool,
        daily_limit: u64,
        used_today: u64,
    ): bool {
        !is_paused
            && amount > 0
            && amount <= sender_balance
            && used_today + amount <= daily_limit
    }
    
    #[test]
    fun test_if_expressions() {
        assert!(if_as_expression() == 5, 0);  // 10/2 = 5
        assert!(classify_number(0) == 0, 1);
        assert!(classify_number(5) == 1, 2);
        assert!(classify_number(42) == 2, 3);
        assert!(classify_number(999) == 3, 4);
        assert!(classify_number(1000) == 4, 5);
        assert!(nested_conditions(5, 3, 4) == 5, 6);
    }
}
```

---

## Loop

`loop` คือ infinite loop ที่ต้องมี `break` เพื่อออก

```move
module learning::loops {
    
    // Basic loop
    public fun count_down(from: u64): vector<u64> {
        let result = vector::empty<u64>();
        let mut i = from;
        
        loop {
            vector::push_back(&mut result, i);
            if (i == 0) break;
            i = i - 1;
        };
        
        result
    }
    
    // Loop ที่ return ค่า
    public fun find_first_even(v: &vector<u64>): (u64, bool) {
        let i = 0u64;
        let len = vector::length(v);
        
        loop {
            if (i >= len) break;
            
            let val = *vector::borrow(v, i);
            if (val % 2 == 0) {
                return (val, true)  // พบค่าคู่
            };
            
            i = i + 1;
        };
        
        (0, false)  // ไม่พบ
    }
    
    // Loop กับ accumulator
    public fun factorial(n: u64): u64 {
        assert!(n <= 20, 0);  // ป้องกัน overflow สำหรับ u64
        
        let result = 1u64;
        let i = 1u64;
        
        loop {
            if (i > n) break;
            result = result * i;
            i = i + 1;
        };
        
        result
    }
    
    // Nested loops
    public fun multiplication_table(size: u64): vector<u64> {
        let table = vector::empty<u64>();
        let i = 1u64;
        
        loop {
            if (i > size) break;
            
            let j = 1u64;
            loop {
                if (j > size) break;
                vector::push_back(&mut table, i * j);
                j = j + 1;
            };
            
            i = i + 1;
        };
        
        table
    }
    
    #[test]
    fun test_loops() {
        assert!(factorial(0) == 1, 0);
        assert!(factorial(5) == 120, 1);
        assert!(factorial(10) == 3628800, 2);
        
        let (val, found) = find_first_even(&vector[1u64, 3u64, 4u64, 6u64]);
        assert!(found == true, 3);
        assert!(val == 4, 4);
        
        let (_, not_found) = find_first_even(&vector[1u64, 3u64, 5u64]);
        assert!(not_found == false, 5);
    }
}

use std::vector;
```

---

## While Loop

`while` คือ conditional loop ที่ตรวจสอบ condition ก่อนทำงาน

```move
module learning::while_loops {
    use std::vector;
    
    // Basic while
    public fun sum_to(n: u64): u64 {
        let total = 0u64;
        let i = 1u64;
        
        while (i <= n) {
            total = total + i;
            i = i + 1;
        };
        
        total
    }
    
    // While กับ condition ที่ซับซ้อน
    public fun collatz_sequence(n: u64): vector<u64> {
        assert!(n > 0, 0);
        
        let sequence = vector::empty<u64>();
        let current = n;
        
        vector::push_back(&mut sequence, current);
        
        while (current != 1) {
            current = if (current % 2 == 0) {
                current / 2
            } else {
                current * 3 + 1
            };
            vector::push_back(&mut sequence, current);
        };
        
        sequence
    }
    
    // Binary search
    public fun binary_search(sorted_v: &vector<u64>, target: u64): (u64, bool) {
        if (vector::is_empty(sorted_v)) return (0, false);
        
        let mut left = 0u64;
        let mut right = vector::length(sorted_v) - 1;
        
        while (left <= right) {
            let mid = left + (right - left) / 2;
            let mid_val = *vector::borrow(sorted_v, mid);
            
            if (mid_val == target) {
                return (mid, true)
            } else if (mid_val < target) {
                if (mid == right) break;  // ป้องกัน overflow
                left = mid + 1;
            } else {
                if (mid == 0) break;      // ป้องกัน underflow
                right = mid - 1;
            };
        };
        
        (0, false)
    }
    
    // GCD ด้วย Euclidean algorithm
    public fun gcd(mut a: u64, mut b: u64): u64 {
        while (b != 0) {
            let temp = b;
            b = a % b;
            a = temp;
        };
        a
    }
    
    // LCM
    public fun lcm(a: u64, b: u64): u64 {
        a / gcd(a, b) * b
    }
    
    // ตรวจสอบ prime number
    public fun is_prime(n: u64): bool {
        if (n < 2) return false;
        if (n == 2) return true;
        if (n % 2 == 0) return false;
        
        let i = 3u64;
        while (i * i <= n) {
            if (n % i == 0) return false;
            i = i + 2;
        };
        
        true
    }
    
    // หา prime numbers ใน range
    public fun primes_in_range(from: u64, to: u64): vector<u64> {
        let primes = vector::empty<u64>();
        let i = from;
        
        while (i <= to) {
            if (is_prime(i)) {
                vector::push_back(&mut primes, i);
            };
            i = i + 1;
        };
        
        primes
    }
    
    #[test]
    fun test_while_loops() {
        // Sum 1 to 100 = 5050
        assert!(sum_to(100) == 5050, 0);
        
        // Collatz from 6: 6,3,10,5,16,8,4,2,1
        let seq = collatz_sequence(6);
        assert!(vector::length(&seq) == 9, 1);
        assert!(*vector::borrow(&seq, 0) == 6, 2);
        assert!(*vector::borrow(&seq, 8) == 1, 3);
        
        // Binary search
        let sorted = vector[1u64, 3u64, 5u64, 7u64, 9u64];
        let (idx, found) = binary_search(&sorted, 5);
        assert!(found == true, 4);
        assert!(idx == 2, 5);
        
        let (_, not_found) = binary_search(&sorted, 4);
        assert!(not_found == false, 6);
        
        // GCD
        assert!(gcd(12, 8) == 4, 7);
        assert!(gcd(100, 75) == 25, 8);
        
        // LCM
        assert!(lcm(4, 6) == 12, 9);
        
        // Primes
        assert!(is_prime(2) == true, 10);
        assert!(is_prime(3) == true, 11);
        assert!(is_prime(4) == false, 12);
        assert!(is_prime(17) == true, 13);
        assert!(is_prime(100) == false, 14);
        
        let primes = primes_in_range(1, 20);
        assert!(vector::length(&primes) == 8, 15);  // 2,3,5,7,11,13,17,19
    }
}
```

---

## For Loop Pattern

Move ไม่มี `for` loop แบบ range-based แต่เราสามารถ simulate ได้

```move
module learning::for_patterns {
    use std::vector;
    
    // Pattern สำหรับ iterate vector (แทน for each)
    public fun iterate_vector(v: &vector<u64>) {
        let i = 0u64;
        let len = vector::length(v);
        
        while (i < len) {
            let element = *vector::borrow(v, i);
            // ทำงานกับ element
            i = i + 1;
        };
    }
    
    // iterate พร้อม index
    public fun enumerate_vector(v: &vector<u64>): vector<u64> {
        let result = vector::empty<u64>();
        let i = 0u64;
        let len = vector::length(v);
        
        while (i < len) {
            let val = *vector::borrow(v, i);
            // คำนวณ index-value pair (index * 1000 + value เป็น example)
            vector::push_back(&mut result, i * 1000 + val);
            i = i + 1;
        };
        
        result
    }
    
    // Range iteration
    public fun range_sum(start: u64, end: u64, step: u64): u64 {
        assert!(step > 0, 0);
        
        let total = 0u64;
        let i = start;
        
        while (i < end) {
            total = total + i;
            i = i + step;
        };
        
        total
    }
    
    // Zip two vectors
    public fun zip_with_sum(a: &vector<u64>, b: &vector<u64>): vector<u64> {
        let len_a = vector::length(a);
        let len_b = vector::length(b);
        let len = if (len_a < len_b) { len_a } else { len_b };
        
        let result = vector::empty<u64>();
        let i = 0u64;
        
        while (i < len) {
            let sum = *vector::borrow(a, i) + *vector::borrow(b, i);
            vector::push_back(&mut result, sum);
            i = i + 1;
        };
        
        result
    }
    
    // Reduce/fold
    public fun fold(v: &vector<u64>, initial: u64, operation: u8): u64 {
        // operation: 0=sum, 1=product, 2=max, 3=min
        let result = initial;
        let i = 0u64;
        let len = vector::length(v);
        
        while (i < len) {
            let val = *vector::borrow(v, i);
            result = if (operation == 0) {
                result + val
            } else if (operation == 1) {
                result * val
            } else if (operation == 2) {
                if (val > result) { val } else { result }
            } else {
                if (val < result) { val } else { result }
            };
            i = i + 1;
        };
        
        result
    }
    
    // Sliding window
    public fun sliding_window_avg(v: &vector<u64>, window_size: u64): vector<u64> {
        let len = vector::length(v);
        let result = vector::empty<u64>();
        
        if (len < window_size) return result;
        
        let i = 0u64;
        while (i + window_size <= len) {
            // คำนวณ average ของ window
            let window_sum = 0u64;
            let j = i;
            while (j < i + window_size) {
                window_sum = window_sum + *vector::borrow(v, j);
                j = j + 1;
            };
            vector::push_back(&mut result, window_sum / window_size);
            i = i + 1;
        };
        
        result
    }
    
    #[test]
    fun test_for_patterns() {
        // range sum: 0+2+4+6+8 = 20
        assert!(range_sum(0, 10, 2) == 20, 0);
        
        // zip sum
        let a = vector[1u64, 2u64, 3u64];
        let b = vector[4u64, 5u64, 6u64];
        let zipped = zip_with_sum(&a, &b);
        assert!(*vector::borrow(&zipped, 0) == 5, 1);
        assert!(*vector::borrow(&zipped, 1) == 7, 2);
        assert!(*vector::borrow(&zipped, 2) == 9, 3);
        
        // fold sum
        let v = vector[1u64, 2u64, 3u64, 4u64, 5u64];
        assert!(fold(&v, 0, 0) == 15, 4);  // sum
        assert!(fold(&v, 1, 1) == 120, 5); // product
        
        // sliding window
        let data = vector[1u64, 2u64, 3u64, 4u64, 5u64];
        let avgs = sliding_window_avg(&data, 3);
        assert!(vector::length(&avgs) == 3, 6);
        assert!(*vector::borrow(&avgs, 0) == 2, 7);  // (1+2+3)/3
        assert!(*vector::borrow(&avgs, 1) == 3, 8);  // (2+3+4)/3
        assert!(*vector::borrow(&avgs, 2) == 4, 9);  // (3+4+5)/3
    }
}
```

---

## Break และ Continue

```move
module learning::break_continue {
    use std::vector;
    
    // break - ออกจาก loop
    public fun find_index(v: &vector<u64>, target: u64): u64 {
        let i = 0u64;
        let len = vector::length(v);
        let found_at = len;  // sentinel value (not found)
        
        while (i < len) {
            if (*vector::borrow(v, i) == target) {
                found_at = i;
                break  // พบแล้ว ออกจาก loop
            };
            i = i + 1;
        };
        
        found_at
    }
    
    // continue - ข้าม iteration ปัจจุบัน
    public fun sum_even_numbers(v: &vector<u64>): u64 {
        let total = 0u64;
        let i = 0u64;
        let len = vector::length(v);
        
        while (i < len) {
            let val = *vector::borrow(v, i);
            i = i + 1;
            
            if (val % 2 != 0) {
                continue  // ข้ามเลขคี่
            };
            
            total = total + val;
        };
        
        total
    }
    
    // Loop labels (nested loops)
    public fun find_in_matrix(
        matrix: &vector<vector<u64>>,
        target: u64,
    ): (u64, u64, bool) {
        let rows = vector::length(matrix);
        let row = 0u64;
        let mut found_row = 0u64;
        let mut found_col = 0u64;
        let mut found = false;
        
        'outer: while (row < rows) {
            let row_vec = vector::borrow(matrix, row);
            let cols = vector::length(row_vec);
            let col = 0u64;
            
            while (col < cols) {
                if (*vector::borrow(row_vec, col) == target) {
                    found_row = row;
                    found_col = col;
                    found = true;
                    break 'outer  // ออกจาก outer loop (Aptos specific)
                };
                col = col + 1;  // ปัญหา: col เป็น immutable
            };
            
            row = row + 1;
        };
        
        (found_row, found_col, found)
    }
    
    // Early termination pattern
    public fun all_positive(v: &vector<u64>): bool {
        let i = 0u64;
        let len = vector::length(v);
        
        while (i < len) {
            if (*vector::borrow(v, i) == 0) {
                return false  // early return แทน break
            };
            i = i + 1;
        };
        
        true
    }
    
    public fun any_zero(v: &vector<u64>): bool {
        let i = 0u64;
        let len = vector::length(v);
        
        while (i < len) {
            if (*vector::borrow(v, i) == 0) {
                return true
            };
            i = i + 1;
        };
        
        false
    }
    
    #[test]
    fun test_break_continue() {
        let v = vector[1u64, 3u64, 5u64, 4u64, 7u64, 9u64];
        
        let idx = find_index(&v, 4);
        assert!(idx == 3, 0);
        
        let not_found_idx = find_index(&v, 100);
        assert!(not_found_idx == vector::length(&v), 1);
        
        let even_sum = sum_even_numbers(&vector[1u64, 2u64, 3u64, 4u64, 5u64, 6u64]);
        assert!(even_sum == 12, 2);  // 2 + 4 + 6
        
        assert!(all_positive(&vector[1u64, 2u64, 3u64]) == true, 3);
        assert!(all_positive(&vector[1u64, 0u64, 3u64]) == false, 4);
    }
}
```

---

## Abort และ Assert

```move
module learning::abort_assert {
    
    // Error codes ตาม category
    // Auth errors
    const E_NOT_AUTHORIZED: u64 = 100;
    const E_NOT_OWNER: u64 = 101;
    
    // Input validation errors
    const E_ZERO_AMOUNT: u64 = 200;
    const E_INVALID_AMOUNT: u64 = 201;
    const E_OVERFLOW: u64 = 202;
    
    // State errors  
    const E_NOT_INITIALIZED: u64 = 300;
    const E_ALREADY_EXISTS: u64 = 301;
    const E_PAUSED: u64 = 302;
    
    // assert! macro - abort ถ้า condition เป็น false
    public fun withdraw(
        amount: u64,
        balance: u64,
        is_paused: bool,
    ): u64 {
        assert!(!is_paused, E_PAUSED);
        assert!(amount > 0, E_ZERO_AMOUNT);
        assert!(amount <= balance, E_INVALID_AMOUNT);
        
        // ทำงานจริง
        balance - amount
    }
    
    // abort - ออกทันทีด้วย error code
    public fun require_admin(caller: address) {
        if (caller != @0x1) {
            abort E_NOT_AUTHORIZED
        }
    }
    
    // Custom error message patterns
    // Move ไม่รองรับ string errors แต่สามารถ encode ข้อมูลใน error code ได้
    
    // Encode: module_id * 10000 + error_code
    // Module 1 errors: 10100, 10101, 10200, ...
    // Module 2 errors: 20100, 20101, 20200, ...
    
    const MODULE_ID: u64 = 1;
    
    fun make_error(category: u64, code: u64): u64 {
        MODULE_ID * 1_000_000 + category * 1000 + code
    }
    
    // Assertion helpers
    fun assert_positive(value: u64, error_code: u64) {
        assert!(value > 0, error_code);
    }
    
    fun assert_in_range(value: u64, min: u64, max: u64, error_code: u64) {
        assert!(value >= min && value <= max, error_code);
    }
    
    fun assert_not_zero_address(addr: address, error_code: u64) {
        assert!(addr != @0x0, error_code);
    }
    
    // ตัวอย่างการใช้ assert ในชีวิตจริง
    public fun transfer_tokens(
        from: address,
        to: address,
        amount: u64,
        from_balance: u64,
        max_transfer: u64,
    ): u64 {
        // Validate inputs
        assert_not_zero_address(to, E_INVALID_AMOUNT);
        assert_positive(amount, E_ZERO_AMOUNT);
        assert_in_range(amount, 1, max_transfer, E_INVALID_AMOUNT);
        
        // Check balance
        assert!(from_balance >= amount, E_INVALID_AMOUNT);
        
        // Check not self-transfer
        assert!(from != to, E_INVALID_AMOUNT);
        
        // Return new balance
        from_balance - amount
    }
    
    // Test: expected failures
    #[test]
    #[expected_failure(abort_code = E_PAUSED)]
    fun test_withdraw_when_paused() {
        withdraw(100, 1000, true);  // ควร abort
    }
    
    #[test]
    #[expected_failure(abort_code = E_ZERO_AMOUNT)]
    fun test_withdraw_zero() {
        withdraw(0, 1000, false);  // ควร abort
    }
    
    #[test]
    fun test_withdraw_success() {
        let new_balance = withdraw(100, 1000, false);
        assert!(new_balance == 900, 0);
    }
}
```

---

## Pattern Matching

Move ไม่มี `match` expression แบบ Rust แต่สามารถ simulate ได้

```move
module learning::pattern_matching {
    
    // Simulate match ด้วย if-else
    const STATUS_PENDING: u8 = 0;
    const STATUS_ACTIVE: u8 = 1;
    const STATUS_COMPLETED: u8 = 2;
    const STATUS_CANCELLED: u8 = 3;
    
    public fun handle_status(status: u8): vector<u8> {
        if (status == STATUS_PENDING) {
            b"Order is pending"
        } else if (status == STATUS_ACTIVE) {
            b"Order is active"
        } else if (status == STATUS_COMPLETED) {
            b"Order completed successfully"
        } else if (status == STATUS_CANCELLED) {
            b"Order was cancelled"
        } else {
            abort 1  // unknown status
        }
    }
    
    // State machine pattern
    const STATE_IDLE: u8 = 0;
    const STATE_RUNNING: u8 = 1;
    const STATE_PAUSED: u8 = 2;
    const STATE_STOPPED: u8 = 3;
    
    struct Machine has key {
        state: u8,
        count: u64,
    }
    
    // Valid transitions
    public fun can_transition(from: u8, to: u8): bool {
        if (from == STATE_IDLE) {
            to == STATE_RUNNING
        } else if (from == STATE_RUNNING) {
            to == STATE_PAUSED || to == STATE_STOPPED
        } else if (from == STATE_PAUSED) {
            to == STATE_RUNNING || to == STATE_STOPPED
        } else {
            false  // STOPPED -> ไปไหนไม่ได้
        }
    }
    
    public fun transition(
        machine: &mut Machine,
        new_state: u8,
    ) {
        assert!(can_transition(machine.state, new_state), 1);
        machine.state = new_state;
    }
    
    // Enum-like pattern ด้วย struct
    struct Result {
        success: bool,
        value: u64,
        error_code: u64,
    }
    
    public fun make_success(value: u64): Result {
        Result { success: true, value, error_code: 0 }
    }
    
    public fun make_error_result(error_code: u64): Result {
        Result { success: false, value: 0, error_code }
    }
    
    public fun unwrap_result(r: Result): u64 {
        assert!(r.success, r.error_code);
        r.value
    }
    
    public fun unwrap_or(r: Result, default: u64): u64 {
        if (r.success) { r.value } else { default }
    }
    
    #[test]
    fun test_pattern_matching() {
        let status_msg = handle_status(STATUS_ACTIVE);
        assert!(status_msg == b"Order is active", 0);
        
        assert!(can_transition(STATE_IDLE, STATE_RUNNING) == true, 1);
        assert!(can_transition(STATE_IDLE, STATE_STOPPED) == false, 2);
        assert!(can_transition(STATE_STOPPED, STATE_RUNNING) == false, 3);
        
        let success = make_success(42);
        assert!(unwrap_result(success) == 42, 4);
        
        let error = make_error_result(100);
        assert!(unwrap_or(error, 99) == 99, 5);
    }
    
    #[test]
    #[expected_failure]
    fun test_invalid_transition() {
        let mut machine = Machine { state: STATE_STOPPED, count: 0 };
        transition(&mut machine, STATE_RUNNING);  // ควร fail
    }
}
```

---

## ตัวอย่างโปรแกรม: State Machine สำหรับ Order System

```move
module learning::order_system {
    use std::signer;
    use std::vector;
    
    // Order states
    const STATE_CREATED: u8 = 1;
    const STATE_CONFIRMED: u8 = 2;
    const STATE_PROCESSING: u8 = 3;
    const STATE_SHIPPED: u8 = 4;
    const STATE_DELIVERED: u8 = 5;
    const STATE_CANCELLED: u8 = 6;
    const STATE_REFUNDED: u8 = 7;
    
    // Error codes
    const E_INVALID_TRANSITION: u64 = 1;
    const E_ORDER_NOT_FOUND: u64 = 2;
    const E_NOT_AUTHORIZED: u64 = 3;
    const E_ZERO_AMOUNT: u64 = 4;
    
    struct Order has store, drop {
        id: u64,
        buyer: address,
        seller: address,
        amount: u64,
        state: u8,
        created_at: u64,
    }
    
    struct OrderBook has key {
        orders: vector<Order>,
        next_id: u64,
    }
    
    // Initialize
    public entry fun initialize(admin: &signer) {
        let addr = signer::address_of(admin);
        assert!(!exists<OrderBook>(addr), 0);
        
        move_to(admin, OrderBook {
            orders: vector::empty<Order>(),
            next_id: 1,
        });
    }
    
    // Create order
    public entry fun create_order(
        buyer: &signer,
        admin_addr: address,
        seller: address,
        amount: u64,
        timestamp: u64,
    ) acquires OrderBook {
        assert!(amount > 0, E_ZERO_AMOUNT);
        assert!(seller != @0x0, E_NOT_AUTHORIZED);
        
        let buyer_addr = signer::address_of(buyer);
        let book = borrow_global_mut<OrderBook>(admin_addr);
        
        let order = Order {
            id: book.next_id,
            buyer: buyer_addr,
            seller,
            amount,
            state: STATE_CREATED,
            created_at: timestamp,
        };
        
        vector::push_back(&mut book.orders, order);
        book.next_id = book.next_id + 1;
    }
    
    // Transition state
    public entry fun transition_order(
        caller: &signer,
        admin_addr: address,
        order_id: u64,
        new_state: u8,
    ) acquires OrderBook {
        let caller_addr = signer::address_of(caller);
        let book = borrow_global_mut<OrderBook>(admin_addr);
        
        let order = find_order_mut(&mut book.orders, order_id);
        
        // Check authorization
        let current_state = order.state;
        check_transition_auth(caller_addr, order, current_state, new_state);
        
        // Validate transition
        assert!(is_valid_transition(current_state, new_state), E_INVALID_TRANSITION);
        
        // Apply transition
        order.state = new_state;
    }
    
    // Valid state transitions
    fun is_valid_transition(from: u8, to: u8): bool {
        if (from == STATE_CREATED) {
            to == STATE_CONFIRMED || to == STATE_CANCELLED
        } else if (from == STATE_CONFIRMED) {
            to == STATE_PROCESSING || to == STATE_CANCELLED
        } else if (from == STATE_PROCESSING) {
            to == STATE_SHIPPED || to == STATE_CANCELLED
        } else if (from == STATE_SHIPPED) {
            to == STATE_DELIVERED
        } else if (from == STATE_CANCELLED) {
            to == STATE_REFUNDED
        } else {
            false
        }
    }
    
    // Check who can make which transition
    fun check_transition_auth(
        caller: address,
        order: &Order,
        from: u8,
        to: u8,
    ) {
        // Buyer actions: confirm, cancel (before shipping), mark delivered
        // Seller actions: process, ship
        // Anyone can refund after cancel
        
        let is_buyer = caller == order.buyer;
        let is_seller = caller == order.seller;
        
        if (to == STATE_CONFIRMED) {
            assert!(is_buyer, E_NOT_AUTHORIZED);
        } else if (to == STATE_PROCESSING || to == STATE_SHIPPED) {
            assert!(is_seller, E_NOT_AUTHORIZED);
        } else if (to == STATE_DELIVERED) {
            assert!(is_buyer, E_NOT_AUTHORIZED);
        } else if (to == STATE_CANCELLED) {
            // Either can cancel before shipping
            assert!(is_buyer || is_seller, E_NOT_AUTHORIZED);
            assert!(from != STATE_SHIPPED, E_INVALID_TRANSITION);
        }
    }
    
    // Find order by ID
    fun find_order_mut(orders: &mut vector<Order>, order_id: u64): &mut Order {
        let i = 0u64;
        let len = vector::length(orders);
        
        while (i < len) {
            if (vector::borrow(orders, i).id == order_id) {
                return vector::borrow_mut(orders, i)
            };
            i = i + 1;
        };
        
        abort E_ORDER_NOT_FOUND
    }
    
    // View: Get order state
    #[view]
    public fun get_order_state(
        admin_addr: address,
        order_id: u64,
    ): u8 acquires OrderBook {
        let book = borrow_global<OrderBook>(admin_addr);
        let i = 0u64;
        let len = vector::length(&book.orders);
        
        while (i < len) {
            let order = vector::borrow(&book.orders, i);
            if (order.id == order_id) {
                return order.state
            };
            i = i + 1;
        };
        
        abort E_ORDER_NOT_FOUND
    }
    
    #[view]
    public fun get_order_count(admin_addr: address): u64 acquires OrderBook {
        vector::length(&borrow_global<OrderBook>(admin_addr).orders)
    }
    
    // Tests
    #[test(admin = @0x1, buyer = @0x2, seller = @0x3)]
    public fun test_order_lifecycle(
        admin: &signer,
        buyer: &signer,
        seller: &signer,
    ) acquires OrderBook {
        let admin_addr = signer::address_of(admin);
        let buyer_addr = signer::address_of(buyer);
        let seller_addr = signer::address_of(seller);
        
        // Initialize
        initialize(admin);
        assert!(get_order_count(admin_addr) == 0, 0);
        
        // Create order
        create_order(buyer, admin_addr, seller_addr, 1000, 1000000);
        assert!(get_order_count(admin_addr) == 1, 1);
        assert!(get_order_state(admin_addr, 1) == STATE_CREATED, 2);
        
        // Buyer confirms
        transition_order(buyer, admin_addr, 1, STATE_CONFIRMED);
        assert!(get_order_state(admin_addr, 1) == STATE_CONFIRMED, 3);
        
        // Seller processes
        transition_order(seller, admin_addr, 1, STATE_PROCESSING);
        
        // Seller ships
        transition_order(seller, admin_addr, 1, STATE_SHIPPED);
        
        // Buyer marks delivered
        transition_order(buyer, admin_addr, 1, STATE_DELIVERED);
        assert!(get_order_state(admin_addr, 1) == STATE_DELIVERED, 4);
    }
}
```

---

## สรุป

| Control Flow | การใช้งาน |
|-------------|---------|
| `if/else` | conditional branching, ใช้เป็น expression ได้ |
| `loop` | infinite loop, ต้องมี break |
| `while` | conditional loop |
| `break` | ออกจาก loop |
| `continue` | ข้าม iteration ปัจจุบัน |
| `return` | return จาก function |
| `abort` | หยุดการทำงานด้วย error code |
| `assert!` | ตรวจสอบ condition, abort ถ้า false |

---

## แบบฝึกหัด

### Exercise 6.1: Control Flow
เขียนฟังก์ชัน `fizzbuzz(n: u64): vector<u64>` ที่:
- ถ้าหาร 15 ลงตัว: เพิ่ม 0
- ถ้าหาร 3 ลงตัว: เพิ่ม 3
- ถ้าหาร 5 ลงตัว: เพิ่ม 5
- อื่นๆ: เพิ่มตัวเลขนั้น

### Exercise 6.2: State Machine
สร้าง state machine สำหรับ vending machine:
- States: IDLE, ACCEPTING_MONEY, DISPENSING, OUT_OF_STOCK
- Transitions ที่ valid
- Error handling

### Exercise 6.3: Algorithms
Implement algorithms ต่อไปนี้ด้วย loop:
1. Bubble Sort
2. Selection Sort
3. Insertion Sort

### Exercise 6.4: Pattern Matching
สร้าง simple interpreter สำหรับ arithmetic expressions:
- `ADD`, `SUB`, `MUL`, `DIV` operations
- Stack-based evaluation

---

**ก่อนหน้า**: [Part 05 - Functions](part-05-functions.md)  
**ต่อไป**: [Part 07 - Modules และ Packages →](part-07-modules.md)
