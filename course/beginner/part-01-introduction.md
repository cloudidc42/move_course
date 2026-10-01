# Part 01: แนะนำ Move Language และประวัติ

## สารบัญ
- [Move คืออะไร?](#move-คืออะไร)
- [ประวัติและที่มา](#ประวัติและที่มา)
- [ทำไมต้อง Move?](#ทำไมต้อง-move)
- [Move vs ภาษาอื่น](#move-vs-ภาษาอื่น)
- [Ecosystem ของ Move](#ecosystem-ของ-move)
- [แนวคิดหลักของ Move](#แนวคิดหลักของ-move)
- [โปรแกรมแรก: Hello, Move!](#โปรแกรมแรก-hello-move)
- [สรุป](#สรุป)
- [แบบฝึกหัด](#แบบฝึกหัด)

---

## Move คืออะไร?

**Move** คือภาษาโปรแกรมที่ออกแบบมาเพื่อ **Blockchain** โดยเฉพาะ สร้างโดยทีมวิศวกรจาก **Meta (Facebook)** ในปี 2019 ภายในโครงการ Libra (ต่อมาเปลี่ยนชื่อเป็น Diem)

### คุณสมบัติสำคัญ:

```
Move Language
├── 🔒 ปลอดภัยสูง (Safety-first)
├── ⚡ ประสิทธิภาพสูง (High Performance)  
├── 🧩 Expressive (แสดงออกได้ชัดเจน)
├── 🔍 Verifiable (ตรวจสอบได้)
└── 🌐 Interoperable (เชื่อมต่อได้)
```

Move ถูกออกแบบให้แก้ปัญหาหลักของ Smart Contract ที่พบบ่อยใน Ethereum (Solidity):

| ปัญหาใน Solidity | วิธีที่ Move แก้ |
|------------------|-----------------|
| Reentrancy attacks | Linear type system ป้องกัน double-spend |
| Integer overflow | Built-in overflow checking |
| Unchecked external calls | Type-safe external calls |
| Assets ถูกสร้างซ้ำ | Resource types (ไม่สามารถ copy หรือ drop ได้) |

---

## ประวัติและที่มา

### Timeline

```
2019 ─── Meta สร้าง Move สำหรับโครงการ Libra
          ├── เป้าหมาย: สร้าง Global Digital Currency
          └── ต้องการภาษาที่ปลอดภัยสูงสำหรับ Financial Assets

2020 ─── Libra เปลี่ยนชื่อเป็น Diem
          └── Move ยังคงพัฒนาต่อเนื่อง

2022 ─── โครงการ Diem ปิดตัว
          ├── ทีมวิศวกรออกไปสร้าง Aptos และ Sui
          └── Move กลายเป็น Open Source

2022 ─── Aptos Mainnet เปิดตัว (ตุลาคม 2022)
          └── ใช้ Move เป็นภาษาหลัก

2023 ─── Sui Mainnet เปิดตัว (พฤษภาคม 2023)
          └── ใช้ Move (Sui Move) เป็นภาษาหลัก

2024-2026 ─── Move Ecosystem เติบโตอย่างรวดเร็ว
              ├── Rooch Network
              ├── Movement Labs
              └── อื่นๆ อีกมากมาย
```

### ผู้สร้าง

Move ถูกสร้างโดยทีมวิศวกรที่มีความเชี่ยวชาญจาก:
- **Sam Blackshear** - หัวหน้าทีมออกแบบภาษา
- **Evan Cheng** - ต่อมาเป็น CEO ของ Mysten Labs (Sui)
- **Mo Shaikh และ Avery Ching** - ต่อมาเป็น CEO/CTO ของ Aptos

---

## ทำไมต้อง Move?

### 1. Resource Safety (ความปลอดภัยของ Resources)

ใน Move **Resources** คือสิ่งของที่มีค่า (เช่น tokens, NFTs) ที่:
- **ไม่สามารถ copy** ได้ (ป้องกันการสร้างเงินปลอม)
- **ไม่สามารถ drop/delete** โดยไม่ตั้งใจ (ป้องกันการสูญหาย)
- **ต้องถูกจัดการ** อย่างถูกต้องเสมอ

```move
// ตัวอย่าง: Resource ใน Move
struct Coin has key {
    value: u64  // ค่าของเหรียญ
}

// ❌ สิ่งนี้จะ ERROR - ไม่สามารถ copy Coin ได้
// let coin2 = coin1;  // ERROR!

// ✅ ต้องใช้ฟังก์ชันที่กำหนดไว้เท่านั้น
fun transfer_coin(coin: Coin, recipient: address) {
    // จัดการ coin อย่างถูกต้อง
}
```

### 2. Formal Verification

Move รองรับ **Move Prover** - เครื่องมือที่สามารถ:
- พิสูจน์ทางคณิตศาสตร์ว่า code ถูกต้อง
- ค้นหา bugs ก่อน deploy
- รับประกัน invariants ของ protocol

### 3. Bytecode Verification

ทุก Move bytecode จะถูก **verify** ก่อน execute:
- ตรวจสอบ type safety
- ตรวจสอบ memory safety
- ตรวจสอบ resource linearity

### 4. Module System

Move มีระบบ Module ที่ชัดเจน:
- แต่ละ module มี namespace ของตัวเอง
- Visibility control (public, private, friend)
- Package management

---

## Move vs ภาษาอื่น

### Move vs Solidity (Ethereum)

| Feature | Move | Solidity |
|---------|------|----------|
| Resource Safety | ✅ Built-in (Linear Types) | ❌ ต้องระวังเอง |
| Reentrancy | ✅ ป้องกันโดย type system | ❌ ต้องใช้ ReentrancyGuard |
| Integer Overflow | ✅ Abort by default | ⚠️ SafeMath ใน Solidity < 0.8 |
| Formal Verification | ✅ Move Prover | ⚠️ Certora, Halmos |
| Gas Model | ✅ Predictable | ⚠️ Variable |
| Learning Curve | ⚠️ Moderate | ✅ Easier |
| Ecosystem Size | ⚠️ Growing | ✅ Largest |

### Move vs Rust

| Feature | Move | Rust |
|---------|------|------|
| Target | Blockchain/Smart Contracts | Systems Programming |
| Memory Model | Global Storage + Resources | Ownership + Borrowing |
| Safety | Resource Linearity | Memory Safety |
| Complexity | Medium | High |
| Blockchain Integration | ✅ Native | ❌ ต้องใช้ Framework |

### Move vs Go

| Feature | Move | Go |
|---------|------|-----|
| Target | Blockchain | General Purpose |
| Type System | Strong + Resource Types | Strong |
| Concurrency | Transaction-based | Goroutines |
| Performance | High (Bytecode) | High (Compiled) |

---

## Ecosystem ของ Move

### Blockchains ที่ใช้ Move

```
Move Ecosystem
├── Aptos (aptos.dev)
│   ├── Move (Core)
│   ├── AptosBFT Consensus
│   └── Block-STM (Parallel Execution)
│
├── Sui (sui.io)
│   ├── Sui Move (Move + Object Model)
│   ├── Narwhal & Bullshark Consensus
│   └── Object-centric Storage
│
├── Rooch Network
│   ├── Move + Bitcoin
│   └── L2 สำหรับ Bitcoin
│
├── Movement Labs
│   ├── Move บน EVM
│   └── Ethereum Compatibility
│
└── อื่นๆ ที่กำลังพัฒนา
    ├── Linera
    └── Initia
```

### Tools และ Infrastructure

```
Development Tools
├── Aptos CLI - Command Line Interface สำหรับ Aptos
├── Sui CLI - Command Line Interface สำหรับ Sui
├── Move Analyzer - Language Server Protocol
├── Move Prover - Formal Verification Tool
├── Move Sandbox - Local Testing Environment
└── VS Code Extension - IDE Support
```

### Libraries และ Frameworks

```
Libraries
├── Aptos Framework (stdlib)
│   ├── aptos_framework
│   ├── aptos_token
│   └── aptos_std
│
├── Sui Framework (stdlib)
│   ├── sui::object
│   ├── sui::transfer
│   └── sui::coin
│
└── Third-party Libraries
    ├── Thala Labs
    ├── Liquidswap
    └── อื่นๆ
```

---

## แนวคิดหลักของ Move

### 1. Linear Type System

ใน Move ทุก "value" สามารถใช้ได้ **ครั้งเดียวเท่านั้น** (unless มี `copy` ability)

```
Linear Type System
├── เหมือน "ตัวต่อ" - แต่ละชิ้นใช้ได้ครั้งเดียว
├── ไม่สามารถ "คัดลอก" โดยไม่ตั้งใจ
└── Compiler ตรวจสอบให้อัตโนมัติ
```

### 2. Abilities (ความสามารถ)

แต่ละ Type ใน Move มี **abilities** ที่กำหนดสิ่งที่ทำได้:

| Ability | ความหมาย | ตัวอย่าง |
|---------|----------|---------|
| `copy` | สามารถ copy ค่าได้ | integers, booleans |
| `drop` | สามารถ drop/delete ได้ | integers, booleans |
| `store` | สามารถเก็บใน global storage | ค่าที่ต้องการ persist |
| `key` | สามารถเป็น key ใน global storage | Resources หลัก |

### 3. Global Storage

Move ใช้ **Global Storage** ที่มีโครงสร้างแบบ:

```
Global Storage: addr => Type => Resource
├── 0x1 => Coin => {value: 1000}
├── 0x2 => Token => {id: 42, owner: 0x1}
└── 0x3 => NFT => {uri: "...", creator: 0x1}
```

### 4. Module System

```move
// Module คือ unit ของ code organization
module my_address::my_module {
    // ประกาศ type, function, constants ที่นี่
    
    struct MyStruct has key {
        value: u64
    }
    
    public fun create_struct(value: u64): MyStruct {
        MyStruct { value }
    }
}
```

---

## โปรแกรมแรก: Hello, Move!

แม้ Move จะไม่มี `print` แบบตรงๆ เหมือนภาษาทั่วไป แต่เราสามารถสร้าง module แรกได้:

### ตัวอย่างที่ 1: Module พื้นฐาน

```move
// ไฟล์: hello_move.move
module hello_move::greeter {
    
    // ประกาศ Struct
    struct Greeting has key, drop {
        message: vector<u8>  // String ใน Move คือ vector<u8>
    }
    
    // ฟังก์ชัน public สำหรับสร้าง Greeting
    public fun create_greeting(message: vector<u8>): Greeting {
        Greeting { message }
    }
    
    // ฟังก์ชันสำหรับ get message
    public fun get_message(greeting: &Greeting): &vector<u8> {
        &greeting.message
    }
    
    // ฟังก์ชัน entry สำหรับ transaction
    public entry fun greet(account: &signer) {
        let greeting = create_greeting(b"Hello, Move!");
        // เก็บ greeting ไว้ใน account
        move_to(account, greeting);
    }
}
```

### ตัวอย่างที่ 2: Simple Calculator

```move
module hello_move::calculator {
    
    // ค่าคงที่
    const MAX_VALUE: u64 = 1000000;
    
    // Error codes
    const E_OVERFLOW: u64 = 1;
    const E_DIVISION_BY_ZERO: u64 = 2;
    
    // บวก
    public fun add(a: u64, b: u64): u64 {
        assert!(a + b <= MAX_VALUE, E_OVERFLOW);
        a + b
    }
    
    // ลบ
    public fun subtract(a: u64, b: u64): u64 {
        assert!(a >= b, 0);  // ป้องกัน underflow
        a - b
    }
    
    // คูณ
    public fun multiply(a: u64, b: u64): u64 {
        if (a == 0 || b == 0) return 0;
        let result = a * b;
        assert!(result / a == b, E_OVERFLOW);  // ตรวจสอบ overflow
        result
    }
    
    // หาร
    public fun divide(a: u64, b: u64): u64 {
        assert!(b != 0, E_DIVISION_BY_ZERO);
        a / b
    }
    
    // คำนวณ modulo
    public fun modulo(a: u64, b: u64): u64 {
        assert!(b != 0, E_DIVISION_BY_ZERO);
        a % b
    }
    
    // ยกกำลัง (power)
    public fun power(base: u64, exp: u64): u64 {
        if (exp == 0) return 1;
        let result = 1u64;
        let i = 0u64;
        while (i < exp) {
            result = multiply(result, base);
            i = i + 1;
        };
        result
    }
}
```

### ตัวอย่างที่ 3: Counter (Stateful)

```move
module hello_move::counter {
    use std::signer;
    
    // Resource ที่เก็บ state
    struct Counter has key {
        value: u64
    }
    
    // Error codes
    const E_COUNTER_NOT_FOUND: u64 = 1;
    const E_COUNTER_ALREADY_EXISTS: u64 = 2;
    
    // สร้าง Counter ใหม่
    public entry fun initialize(account: &signer) {
        let addr = signer::address_of(account);
        assert!(!exists<Counter>(addr), E_COUNTER_ALREADY_EXISTS);
        move_to(account, Counter { value: 0 });
    }
    
    // เพิ่มค่า Counter
    public entry fun increment(account: &signer) acquires Counter {
        let addr = signer::address_of(account);
        assert!(exists<Counter>(addr), E_COUNTER_NOT_FOUND);
        let counter = borrow_global_mut<Counter>(addr);
        counter.value = counter.value + 1;
    }
    
    // ลดค่า Counter
    public entry fun decrement(account: &signer) acquires Counter {
        let addr = signer::address_of(account);
        assert!(exists<Counter>(addr), E_COUNTER_NOT_FOUND);
        let counter = borrow_global_mut<Counter>(addr);
        assert!(counter.value > 0, 0);
        counter.value = counter.value - 1;
    }
    
    // Reset Counter
    public entry fun reset(account: &signer) acquires Counter {
        let addr = signer::address_of(account);
        assert!(exists<Counter>(addr), E_COUNTER_NOT_FOUND);
        let counter = borrow_global_mut<Counter>(addr);
        counter.value = 0;
    }
    
    // อ่านค่า Counter
    #[view]
    public fun get_value(addr: address): u64 acquires Counter {
        assert!(exists<Counter>(addr), E_COUNTER_NOT_FOUND);
        borrow_global<Counter>(addr).value
    }
    
    // ตรวจสอบว่า Counter มีอยู่หรือไม่
    #[view]
    public fun exists_counter(addr: address): bool {
        exists<Counter>(addr)
    }
}
```

---

## ทำความเข้าใจ Syntax พื้นฐาน

### การประกาศ Module

```move
module <address>::<module_name> {
    // code ของ module
}
```

- `<address>` - address ที่ deploy module (เช่น `0x1`, `my_addr`)
- `<module_name>` - ชื่อ module

### การ Import

```move
use std::signer;          // import signer module จาก standard library
use aptos_framework::coin; // import coin module จาก Aptos framework
use 0x1::string;          // import โดยใช้ address ตรงๆ
```

### ประเภทข้อมูลพื้นฐาน

```move
// Integer types
let a: u8 = 255;          // 0 ถึง 255
let b: u16 = 65535;       // 0 ถึง 65,535
let c: u32 = 4294967295;  // 0 ถึง 4,294,967,295
let d: u64 = 18446744073709551615u64; // ใหญ่มาก
let e: u128 = 340282366920938463463374607431768211455u128; // ใหญ่กว่า
let f: u256 = 0u256;      // ใหญ่ที่สุด

// Boolean
let is_active: bool = true;
let is_done: bool = false;

// Address
let addr: address = @0x1;
let my_addr: address = @my_address;

// Vector (Array)
let numbers: vector<u64> = vector[1, 2, 3, 4, 5];
let bytes: vector<u8> = b"Hello"; // byte string

// Tuple (ไม่มีใน Move - ใช้ Struct แทน)
```

### ฟังก์ชัน

```move
// ฟังก์ชัน private (default)
fun private_function(x: u64): u64 {
    x * 2
}

// ฟังก์ชัน public
public fun public_function(x: u64): u64 {
    x + 1
}

// ฟังก์ชัน entry (เรียกจาก transaction ได้)
public entry fun entry_function(account: &signer) {
    // ทำงานบน blockchain
}

// ฟังก์ชัน friend (เรียกได้จาก module ที่กำหนด)
public(friend) fun friend_function(): u64 {
    42
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Move คืออะไร** - ภาษาโปรแกรมสำหรับ Blockchain ที่ปลอดภัยสูง
2. **ประวัติ** - สร้างโดย Meta, ใช้ใน Aptos และ Sui
3. **ข้อดีของ Move** - Resource safety, Formal verification, Type safety
4. **Ecosystem** - Aptos, Sui, Rooch, Movement Labs
5. **แนวคิดหลัก** - Linear types, Abilities, Global storage, Modules
6. **โปรแกรมแรก** - Module พื้นฐาน, Calculator, Counter

ใน Part ถัดไปเราจะ**ติดตั้ง Development Environment** เพื่อเริ่มเขียน Move จริงๆ

---

## แบบฝึกหัด

### Exercise 1.1: ทำความเข้าใจ Concepts
ตอบคำถามต่อไปนี้:
1. Move แตกต่างจาก Solidity อย่างไรในแง่ Resource Safety?
2. Abilities 4 ตัวของ Move คืออะไรบ้าง?
3. ทำไม Move ถึงไม่อนุญาตให้ copy Resource โดยตรง?

### Exercise 1.2: อ่าน Code
อ่าน Counter module และตอบ:
1. `acquires Counter` คืออะไร?
2. `borrow_global_mut` แตกต่างจาก `borrow_global` อย่างไร?
3. ทำไมต้องมี `assert!` ก่อนทำงาน?

### Exercise 1.3: แก้ไข Code
แก้ไข Calculator module เพื่อเพิ่มฟังก์ชัน:
1. `absolute_difference(a: u64, b: u64): u64` - คำนวณค่าต่างสัมบูรณ์
2. `average(a: u64, b: u64): u64` - คำนวณค่าเฉลี่ย
3. `min(a: u64, b: u64): u64` - หาค่าน้อยสุด
4. `max(a: u64, b: u64): u64` - หาค่ามากสุด

### Exercise 1.4: สร้าง Module ใหม่
สร้าง module `temperature_converter` ที่มีฟังก์ชัน:
1. `celsius_to_fahrenheit(c: u64): u64`
2. `fahrenheit_to_celsius(f: u64): u64`
3. `celsius_to_kelvin(c: u64): u64`

**Hint**: ในการจัดการ negative numbers ใน Move อาจต้องใช้ trick พิเศษ!

---

## Resources เพิ่มเติม

- [Move Language Documentation](https://move-language.github.io/move/)
- [Aptos Move Book](https://aptos.dev/en/build/smart-contracts/book)
- [Sui Move Documentation](https://docs.sui.io/concepts/sui-move-concepts)
- [Move Examples on GitHub](https://github.com/move-language/move)

---

**ต่อไป**: [Part 02 - การติดตั้ง Development Environment →](part-02-setup.md)
