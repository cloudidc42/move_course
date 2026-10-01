# Part 03: พื้นฐาน Syntax และ Types

## สารบัญ
- [Primitive Types](#primitive-types)
- [Integer Types ในรายละเอียด](#integer-types-ในรายละเอียด)
- [Boolean Type](#boolean-type)
- [Address Type](#address-type)
- [Vector Type](#vector-type)
- [Type Casting](#type-casting)
- [Literals](#literals)
- [Expressions และ Statements](#expressions-และ-statements)
- [Scope และ Shadowing](#scope-และ-shadowing)
- [Comments](#comments)
- [แบบฝึกหัด](#แบบฝึกหัด)

---

## Primitive Types

Move มี primitive types หลัก 4 ประเภท:

```
Primitive Types
├── Integers: u8, u16, u32, u64, u128, u256
├── Boolean: bool
├── Address: address
└── Vector: vector<T>
```

---

## Integer Types ในรายละเอียด

### ช่วงของแต่ละ Integer Type

```move
module learning::integer_types {
    
    public fun show_integer_ranges() {
        // u8: 0 ถึง 255 (2^8 - 1)
        let max_u8: u8 = 255u8;
        let min_u8: u8 = 0u8;
        
        // u16: 0 ถึง 65,535 (2^16 - 1)
        let max_u16: u16 = 65535u16;
        
        // u32: 0 ถึง 4,294,967,295 (2^32 - 1)
        let max_u32: u32 = 4294967295u32;
        
        // u64: 0 ถึง 18,446,744,073,709,551,615 (2^64 - 1)
        let max_u64: u64 = 18446744073709551615u64;
        
        // u128: 0 ถึง 340,282,366,920,938,463,463,374,607,431,768,211,455
        let max_u128: u128 = 340282366920938463463374607431768211455u128;
        
        // u256: ใหญ่มาก (2^256 - 1)
        let max_u256: u256 = 0u256;  // ตัวอย่างแค่ 0
        
        // ใช้ _ เป็น separator เพื่ออ่านง่าย
        let million: u64 = 1_000_000;
        let billion: u64 = 1_000_000_000;
    }
    
    // ตัวอย่าง arithmetic operations
    public fun arithmetic_demo(): u64 {
        let a: u64 = 100;
        let b: u64 = 30;
        
        let sum = a + b;        // 130
        let diff = a - b;       // 70
        let product = a * b;    // 3000
        let quotient = a / b;   // 3 (integer division)
        let remainder = a % b;  // 10
        
        // Bitwise operations
        let and = a & b;        // bitwise AND
        let or = a | b;         // bitwise OR
        let xor = a ^ b;        // bitwise XOR
        let left_shift = a << 2; // shift left 2 bits (คูณ 4)
        let right_shift = a >> 2; // shift right 2 bits (หาร 4)
        
        sum
    }
}
```

### Overflow Behavior

```move
module learning::overflow_demo {
    
    // Move จะ ABORT (ไม่ใช่ wrap around) เมื่อเกิด overflow
    public fun overflow_example() {
        let max_u8: u8 = 255u8;
        // let overflow = max_u8 + 1u8;  // ❌ จะ ABORT ณ runtime!
        
        // วิธีที่ถูก: ตรวจสอบก่อน
        let value: u8 = 200u8;
        let add_amount: u8 = 50u8;
        
        // ตรวจสอบว่า overflow หรือไม่
        assert!((value as u64) + (add_amount as u64) <= 255u64, 0);
        let result = value + add_amount;
    }
    
    // Safer addition ที่ไม่ overflow
    public fun safe_add_u64(a: u64, b: u64): u64 {
        let max = 18446744073709551615u64;
        assert!(a <= max - b, 1);  // ตรวจสอบก่อน
        a + b
    }
    
    // Wrapping arithmetic (ใช้ manual implementation)
    public fun wrapping_add_u8(a: u8, b: u8): u8 {
        let result = (a as u64) + (b as u64);
        (result % 256u64) as u8
    }
}
```

### Integer Comparison

```move
module learning::comparisons {
    
    public fun comparison_demo(a: u64, b: u64): bool {
        let equal = a == b;
        let not_equal = a != b;
        let less = a < b;
        let less_or_equal = a <= b;
        let greater = a > b;
        let greater_or_equal = a >= b;
        
        equal
    }
    
    // ตัวอย่าง: หาค่ามากสุดและน้อยสุด
    public fun max(a: u64, b: u64): u64 {
        if (a >= b) { a } else { b }
    }
    
    public fun min(a: u64, b: u64): u64 {
        if (a <= b) { a } else { b }
    }
    
    // Clamp: จำกัดค่าให้อยู่ใน range
    public fun clamp(value: u64, min_val: u64, max_val: u64): u64 {
        if (value < min_val) { min_val }
        else if (value > max_val) { max_val }
        else { value }
    }
}
```

---

## Boolean Type

```move
module learning::boolean_demo {
    
    public fun boolean_basics() {
        // ประกาศ boolean
        let is_active: bool = true;
        let is_done: bool = false;
        
        // Logical operations
        let and_result = is_active && is_done;   // false
        let or_result = is_active || is_done;    // true
        let not_result = !is_active;              // false
        
        // Short-circuit evaluation (เหมือนภาษาอื่น)
        // && และ || ประเมินจากซ้ายไปขวา และหยุดทันทีที่รู้ผล
    }
    
    // ตัวอย่าง: การใช้ boolean ในเงื่อนไข
    public fun check_eligibility(age: u64, has_id: bool): bool {
        age >= 18 && has_id
    }
    
    // Boolean expressions ที่ซับซ้อน
    public fun complex_condition(
        a: u64, 
        b: u64, 
        c: u64
    ): bool {
        // หา 3 ค่าว่ามีอย่างน้อย 2 ค่าที่มากกว่า 10
        let a_big = a > 10;
        let b_big = b > 10;
        let c_big = c > 10;
        
        (a_big && b_big) || (a_big && c_big) || (b_big && c_big)
    }
    
    // DeMorgan's Laws
    public fun demorgan_demo(p: bool, q: bool) {
        // NOT (P AND Q) = (NOT P) OR (NOT Q)
        let law1_left = !(p && q);
        let law1_right = (!p) || (!q);
        assert!(law1_left == law1_right, 0);
        
        // NOT (P OR Q) = (NOT P) AND (NOT Q)
        let law2_left = !(p || q);
        let law2_right = (!p) && (!q);
        assert!(law2_left == law2_right, 0);
    }
}
```

---

## Address Type

```move
module learning::address_demo {
    use std::signer;
    
    public fun address_basics() {
        // Literal addresses
        let addr1: address = @0x1;       // Aptos/Move standard library
        let addr2: address = @0xCAFE;    // hex address
        let addr3: address = @0x000000000000000000000000000000000000000000000000000000000000CAFE;
        
        // Named addresses (ต้องกำหนดใน Move.toml)
        // let named_addr: address = @my_address;
        
        // Address เป็น 32 bytes (256 bits) ใน Aptos/Sui
    }
    
    // ใช้ address ในฟังก์ชัน
    public fun get_signer_address(account: &signer): address {
        signer::address_of(account)
    }
    
    // ตรวจสอบ address
    public fun is_same_address(a: address, b: address): bool {
        a == b
    }
    
    // ตรวจสอบว่า address มี resource หรือไม่
    struct MyResource has key { value: u64 }
    
    public fun has_resource(addr: address): bool {
        exists<MyResource>(addr)
    }
}
```

---

## Vector Type

Vector คือ dynamic array ใน Move ที่สามารถเก็บ elements ประเภทเดียวกัน

```move
module learning::vector_demo {
    use std::vector;
    
    public fun vector_basics() {
        // สร้าง vector ว่าง
        let v1: vector<u64> = vector::empty<u64>();
        
        // สร้าง vector พร้อม elements
        let v2: vector<u64> = vector[1, 2, 3, 4, 5];
        
        // Vector ของ bytes (string)
        let bytes: vector<u8> = b"Hello, Move!";
        
        // Vector ของ bool
        let flags: vector<bool> = vector[true, false, true];
        
        // Vector ของ vector (nested)
        let matrix: vector<vector<u64>> = vector[
            vector[1, 2, 3],
            vector[4, 5, 6],
            vector[7, 8, 9],
        ];
    }
    
    // Operations บน Vector
    public fun vector_operations(v: &mut vector<u64>) {
        // เพิ่ม element ต่อท้าย
        vector::push_back(v, 100);
        
        // ลบ element สุดท้าย
        let last = vector::pop_back(v);
        
        // ความยาว
        let len = vector::length(v);
        
        // ดู element (read)
        if (len > 0) {
            let first = vector::borrow(v, 0);  // &u64
            let first_value = *first;           // u64
            
            // แก้ไข element
            let first_mut = vector::borrow_mut(v, 0);  // &mut u64
            *first_mut = 999;
        }
        
        // เพิ่ม element ที่ตำแหน่งใดก็ได้ (O(n))
        vector::insert(v, 0, 999);  // ใส่ 999 ที่ index 0
        
        // ลบ element ที่ตำแหน่งใดก็ได้ (O(n))
        if (vector::length(v) > 0) {
            vector::remove(v, 0);   // ลบ element ที่ index 0
        };
        
        // ลบ element แบบ swap (O(1) - swap กับ last แล้ว pop)
        if (vector::length(v) > 1) {
            vector::swap_remove(v, 0);
        };
    }
    
    // ตรวจสอบ vector
    public fun vector_checks(v: &vector<u64>) {
        let is_empty = vector::is_empty(v);
        let len = vector::length(v);
        
        // ค้นหา element
        let (found, index) = vector::index_of(v, &42u64);
        
        // มี element หรือไม่
        let contains = vector::contains(v, &42u64);
    }
    
    // Operations ขั้นสูง
    public fun vector_advanced() {
        let mut v = vector[3, 1, 4, 1, 5, 9, 2, 6];
        
        // Reverse
        vector::reverse(&mut v);
        // v = [6, 2, 9, 5, 1, 4, 1, 3]
        
        // Append (ต่อ vector 2 เข้ากัน)
        let extra = vector[7, 8];
        vector::append(&mut v, extra);
        // v = [6, 2, 9, 5, 1, 4, 1, 3, 7, 8]
    }
    
    // สร้าง vector helper functions
    public fun sum(v: &vector<u64>): u64 {
        let total = 0u64;
        let i = 0u64;
        let len = vector::length(v);
        while (i < len) {
            total = total + *vector::borrow(v, i);
            i = i + 1;
        };
        total
    }
    
    public fun average(v: &vector<u64>): u64 {
        let len = vector::length(v);
        assert!(len > 0, 0);
        sum(v) / len
    }
    
    // ค้นหาค่ามากสุดใน vector
    public fun max_value(v: &vector<u64>): u64 {
        let len = vector::length(v);
        assert!(len > 0, 0);
        
        let max = *vector::borrow(v, 0);
        let i = 1u64;
        while (i < len) {
            let val = *vector::borrow(v, i);
            if (val > max) {
                max = val;
            };
            i = i + 1;
        };
        max
    }
    
    // กรอง elements (filter)
    public fun filter_greater_than(v: &vector<u64>, threshold: u64): vector<u64> {
        let result = vector::empty<u64>();
        let i = 0u64;
        let len = vector::length(v);
        while (i < len) {
            let val = *vector::borrow(v, i);
            if (val > threshold) {
                vector::push_back(&mut result, val);
            };
            i = i + 1;
        };
        result
    }
    
    // Map (แปลงแต่ละ element)
    public fun map_double(v: &vector<u64>): vector<u64> {
        let result = vector::empty<u64>();
        let i = 0u64;
        let len = vector::length(v);
        while (i < len) {
            let val = *vector::borrow(v, i);
            vector::push_back(&mut result, val * 2);
            i = i + 1;
        };
        result
    }
}
```

---

## Type Casting

```move
module learning::type_casting {
    
    public fun casting_demo() {
        // Casting เล็กไปใหญ่ (ปลอดภัย)
        let small: u8 = 100u8;
        let medium: u32 = small as u32;  // 100u32
        let large: u64 = small as u64;   // 100u64
        let xlarge: u128 = small as u128; // 100u128
        
        // Casting ใหญ่ไปเล็ก (truncation - อาจสูญเสียข้อมูล!)
        let big: u64 = 1000u64;
        let truncated: u8 = big as u8;   // 1000 mod 256 = 232
        
        // ❌ ระวัง! ค่า 1000 จะถูก truncate เป็น 232
        // 1000 in binary: 0000001111101000
        // เก็บแค่ 8 bits: 11101000 = 232
        
        // วิธีที่ถูก: ตรวจสอบก่อน cast
        let safe_big: u64 = 200u64;
        assert!(safe_big <= 255u64, 0);
        let safe_small: u8 = safe_big as u8;  // 200u8
    }
    
    // Casting function ที่ปลอดภัย
    public fun safe_cast_to_u8(value: u64): u8 {
        assert!(value <= 255u64, 0);
        value as u8
    }
    
    public fun safe_cast_to_u32(value: u64): u32 {
        assert!(value <= 4294967295u64, 0);
        value as u32
    }
    
    // ใช้ casting ในการคำนวณ
    public fun multiply_without_overflow(a: u64, b: u64): u64 {
        // Cast ขึ้นไปเป็น u128 เพื่อป้องกัน overflow
        let a_128 = a as u128;
        let b_128 = b as u128;
        let result_128 = a_128 * b_128;
        
        // ตรวจสอบว่า result ไม่เกิน u64
        assert!(result_128 <= 18446744073709551615u128, 0);
        result_128 as u64
    }
}
```

---

## Literals

```move
module learning::literals {
    
    public fun literal_demo() {
        // Integer literals
        let decimal: u64 = 1234567890;
        let hex: u64 = 0xDEADBEEF;          // hexadecimal
        let with_separator: u64 = 1_000_000; // underscore separator
        let typed: u8 = 42u8;                // explicit type suffix
        
        // Byte string literals (vector<u8>)
        let hello: vector<u8> = b"Hello, World!";
        let emoji_note: vector<u8> = b"Move";  // ASCII only
        
        // Hex string literals
        let hex_bytes: vector<u8> = x"DEADBEEF";  // 4 bytes
        
        // Boolean literals
        let yes: bool = true;
        let no: bool = false;
        
        // Address literals
        let framework: address = @0x1;
        let custom: address = @0xCAFEBABE;
    }
    
    // ตัวอย่างการใช้ literals จริงๆ
    public fun token_constants() {
        // Decimal places (ส่วนใหญ่ใช้ 8 decimal places เหมือน Bitcoin)
        let one_apt: u64 = 100_000_000;  // 1 APT = 10^8 Octas
        let one_sui: u64 = 1_000_000_000; // 1 SUI = 10^9 MIST
        
        // Fee calculation
        let fee_bps: u64 = 30;  // 0.30% = 30 basis points
        let amount: u64 = 1_000_000_000;
        let fee = amount * fee_bps / 10_000;  // 3,000,000 MIST
    }
}
```

---

## Expressions และ Statements

```move
module learning::expressions {
    
    public fun expression_demo(): u64 {
        // ใน Move ทุกอย่างเป็น expression ที่มีค่า
        
        // Block expression - ค่าสุดท้ายคือ return value
        let result = {
            let x = 10;
            let y = 20;
            x + y  // ไม่มี semicolon = return value ของ block
        };
        // result = 30
        
        // If expression
        let x = 5u64;
        let absolute = if (x >= 0) { x } else { 0 - x };
        // ใน Move u64 ไม่มี negative ต้องระวัง!
        
        // If-else เป็น expression
        let grade = if (x >= 90) {
            b"A"
        } else if (x >= 80) {
            b"B"
        } else if (x >= 70) {
            b"C"
        } else {
            b"F"
        };
        
        result
    }
    
    // Statements (จบด้วย semicolon)
    public fun statement_demo() {
        // let statement
        let x = 10;  // statement
        
        // expression statement (ค่าถูก discard)
        // 5 + 3;  // ❌ จะ warning เพราะ discard value
        
        // ฟังก์ชันที่ return () (unit) ถือเป็น statement
        let v = vector::empty<u64>();  
        vector::push_back(&mut v, 1);  // statement - side effect
    }
    
    // Return values
    public fun returns_value(): u64 {
        // ค่าสุดท้ายในฟังก์ชัน = return value
        let x = 10;
        let y = 20;
        x + y  // 30 (ไม่มี semicolon)
    }
    
    // Early return
    public fun early_return(x: u64): u64 {
        if (x == 0) {
            return 0  // return ก่อน
        };
        // ถ้ามาถึงตรงนี้ x != 0
        x * 2
    }
    
    // Abort expression
    public fun must_be_positive(x: u64): u64 {
        if (x == 0) {
            abort 1  // abort ด้วย error code 1
        };
        x
    }
}

// ต้อง import vector
use std::vector;
```

---

## Scope และ Shadowing

```move
module learning::scope_demo {
    
    public fun scope_example(): u64 {
        let x = 10u64;  // outer scope
        
        {
            let x = 20u64;  // inner scope (shadowing)
            // ที่นี่ x = 20
            let y = x + 5;  // y = 25
            // y ถูก drop ตอนออกจาก block นี้
        };
        
        // กลับมา outer scope: x = 10
        x  // return 10
    }
    
    // ตัวอย่าง: ใช้ scope เพื่อจัดการ lifetime
    public fun scope_for_borrowing(v: &mut vector<u64>): u64 {
        let result;
        {
            let first = vector::borrow(v, 0);  // borrow เริ่มต้น
            result = *first;                    // copy ค่าออกมา
            // borrow สิ้นสุดตอนออกจาก block
        };
        
        // ตอนนี้ borrow จบแล้ว สามารถ push ได้
        vector::push_back(v, result);
        result
    }
    
    // Nested scope
    public fun nested_scope(): u64 {
        let outer = 1u64;
        
        let middle = {
            let inner = outer + 10;  // inner สามารถ access outer
            inner + 5               // middle = 16
        };
        
        outer + middle  // 1 + 16 = 17
    }
    
    // Variable shadowing
    public fun shadowing_demo(): u64 {
        let x = 5u64;
        let x = x * 2;   // shadow x ด้วยค่าใหม่ 10
        let x = x + 3;   // shadow อีกครั้ง เป็น 13
        x  // return 13
    }
}

use std::vector;
```

---

## Comments

```move
module learning::comments {
    
    // Single line comment ด้วย //
    
    /* 
       Multi-line comment
       ด้วย /* ... */
    */
    
    // Documentation comment (ใช้ ///  3 slash)
    /// ฟังก์ชันนี้ทำการบวกตัวเลขสองตัว
    /// 
    /// # Arguments
    /// * `a` - ตัวเลขแรก
    /// * `b` - ตัวเลขที่สอง
    ///
    /// # Returns
    /// ผลบวกของ a และ b
    public fun add(a: u64, b: u64): u64 {
        a + b  // บวกกัน
    }
    
    // Comment ใน code
    public fun complex_logic(n: u64): u64 {
        // ตรวจสอบกรณี edge case ก่อน
        if (n == 0) {
            return 0
        };
        
        /* 
         * ใช้ algorithm Collatz sequence
         * ถ้าเลขคู่ หาร 2
         * ถ้าเลขคี่ คูณ 3 แล้วบวก 1
         */
        if (n % 2 == 0) {
            n / 2  // เลขคู่
        } else {
            n * 3 + 1  // เลขคี่
        }
    }
}
```

---

## ตัวอย่างโปรแกรมสมบูรณ์: Statistics Calculator

```move
module learning::statistics {
    use std::vector;
    
    const E_EMPTY_VECTOR: u64 = 1;
    const E_INVALID_PERCENTILE: u64 = 2;
    
    /// คำนวณผลรวม
    public fun sum(data: &vector<u64>): u64 {
        let total = 0u64;
        let i = 0u64;
        let len = vector::length(data);
        
        assert!(len > 0, E_EMPTY_VECTOR);
        
        while (i < len) {
            total = total + *vector::borrow(data, i);
            i = i + 1;
        };
        total
    }
    
    /// คำนวณค่าเฉลี่ย (mean)
    public fun mean(data: &vector<u64>): u64 {
        let len = vector::length(data);
        assert!(len > 0, E_EMPTY_VECTOR);
        sum(data) / len
    }
    
    /// หาค่าน้อยสุด
    public fun min_value(data: &vector<u64>): u64 {
        let len = vector::length(data);
        assert!(len > 0, E_EMPTY_VECTOR);
        
        let min = *vector::borrow(data, 0);
        let i = 1u64;
        
        while (i < len) {
            let val = *vector::borrow(data, i);
            if (val < min) { min = val; };
            i = i + 1;
        };
        min
    }
    
    /// หาค่ามากสุด
    public fun max_value(data: &vector<u64>): u64 {
        let len = vector::length(data);
        assert!(len > 0, E_EMPTY_VECTOR);
        
        let max = *vector::borrow(data, 0);
        let i = 1u64;
        
        while (i < len) {
            let val = *vector::borrow(data, i);
            if (val > max) { max = val; };
            i = i + 1;
        };
        max
    }
    
    /// คำนวณ range (max - min)
    public fun range(data: &vector<u64>): u64 {
        max_value(data) - min_value(data)
    }
    
    /// Sort vector (Bubble sort - simple but O(n^2))
    public fun sort(data: &mut vector<u64>) {
        let len = vector::length(data);
        let i = 0u64;
        
        while (i < len) {
            let j = 0u64;
            while (j < len - 1 - i) {
                let a = *vector::borrow(data, j);
                let b = *vector::borrow(data, j + 1);
                if (a > b) {
                    // Swap
                    *vector::borrow_mut(data, j) = b;
                    *vector::borrow_mut(data, j + 1) = a;
                };
                j = j + 1;
            };
            i = i + 1;
        };
    }
    
    /// หา median (ค่ากลาง)
    public fun median(data: &vector<u64>): u64 {
        let len = vector::length(data);
        assert!(len > 0, E_EMPTY_VECTOR);
        
        // Copy และ sort
        let sorted = *data;
        sort(&mut sorted);
        
        if (len % 2 == 1) {
            // จำนวนคี่: เอาตรงกลาง
            *vector::borrow(&sorted, len / 2)
        } else {
            // จำนวนคู่: เฉลี่ย 2 ค่ากลาง
            let mid1 = *vector::borrow(&sorted, len / 2 - 1);
            let mid2 = *vector::borrow(&sorted, len / 2);
            (mid1 + mid2) / 2
        }
    }
    
    /// นับจำนวน elements ที่ตรงกับ value
    public fun count_occurrences(data: &vector<u64>, value: u64): u64 {
        let count = 0u64;
        let i = 0u64;
        let len = vector::length(data);
        
        while (i < len) {
            if (*vector::borrow(data, i) == value) {
                count = count + 1;
            };
            i = i + 1;
        };
        count
    }
    
    // Tests
    #[test]
    fun test_statistics() {
        let data = vector[5u64, 3u64, 8u64, 1u64, 9u64, 2u64, 7u64, 4u64, 6u64];
        
        assert!(sum(&data) == 45, 0);
        assert!(mean(&data) == 5, 1);
        assert!(min_value(&data) == 1, 2);
        assert!(max_value(&data) == 9, 3);
        assert!(range(&data) == 8, 4);
        assert!(median(&data) == 5, 5);
    }
    
    #[test]
    fun test_sort() {
        let data = vector[5u64, 3u64, 1u64, 4u64, 2u64];
        sort(&mut data);
        
        assert!(*vector::borrow(&data, 0) == 1, 0);
        assert!(*vector::borrow(&data, 1) == 2, 1);
        assert!(*vector::borrow(&data, 2) == 3, 2);
        assert!(*vector::borrow(&data, 3) == 4, 3);
        assert!(*vector::borrow(&data, 4) == 5, 4);
    }
}
```

---

## สรุป

ในบทนี้เราเรียนรู้:

| หัวข้อ | สิ่งที่เรียนรู้ |
|-------|--------------|
| Integer Types | u8, u16, u32, u64, u128, u256 และการใช้งาน |
| Boolean | true/false, logical operators |
| Address | @0x1, @named_address |
| Vector | Dynamic array, operations ต่างๆ |
| Type Casting | `as` keyword, safe casting |
| Literals | decimal, hex, byte strings |
| Expressions | Block expressions, if expressions |
| Scope | Nested scope, shadowing |

---

## แบบฝึกหัด

### Exercise 3.1: Integer Operations
เขียนฟังก์ชัน `gcd(a: u64, b: u64): u64` ที่หา Greatest Common Divisor ด้วย Euclidean Algorithm

### Exercise 3.2: Vector Operations
เขียนฟังก์ชันต่อไปนี้:
1. `flatten(v: &vector<vector<u64>>): vector<u64>` - รวม nested vectors
2. `unique(v: &vector<u64>): vector<u64>` - ลบ duplicates
3. `zip_sum(a: &vector<u64>, b: &vector<u64>): vector<u64>` - บวก elements ที่ตำแหน่งเดียวกัน

### Exercise 3.3: Type Safety
เขียน `safe_divide(a: u64, b: u64): (u64, bool)` ที่ return (result, success) โดยไม่ abort

### Exercise 3.4: Statistics
เพิ่มฟังก์ชัน `mode(data: &vector<u64>): u64` ที่หาค่าที่พบบ่อยสุด (ค่าที่ซ้ำมากสุด)

---

**ก่อนหน้า**: [Part 02 - การติดตั้ง Development Environment](part-02-setup.md)  
**ต่อไป**: [Part 04 - Variables และ Mutability →](part-04-variables.md)
