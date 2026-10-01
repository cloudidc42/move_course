# Part 07: Modules และ Packages

## สารบัญ
- [Module Basics](#module-basics)
- [Package Structure](#package-structure)
- [Import และ Use](#import-และ-use)
- [Visibility](#visibility)
- [Friend Modules](#friend-modules)
- [Module Dependencies](#module-dependencies)
- [ตัวอย่าง: Multi-module Protocol](#ตัวอย่าง-multi-module-protocol)

---

## Module Basics

```move
// Module declaration syntax:
// module <address>::<module_name> { ... }

module 0xCAFE::my_module {
    // Module code here
}

// หรือใช้ named address (กำหนดใน Move.toml)
module my_package::token {
    // Token implementation
}
```

### Module Components

```move
module learning::module_anatomy {
    // 1. Imports
    use std::signer;
    use std::vector;
    
    // 2. Friend declarations
    friend learning::friend_module;
    
    // 3. Constants
    const MAX_VALUE: u64 = 1000;
    const E_ERROR: u64 = 1;
    
    // 4. Struct definitions
    struct MyResource has key, store {
        value: u64,
    }
    
    struct MyData has copy, drop, store {
        data: vector<u8>,
    }
    
    // 5. Private functions (internal helpers)
    fun internal_compute(x: u64): u64 {
        x * 2
    }
    
    // 6. Public functions (external API)
    public fun external_api(x: u64): u64 {
        internal_compute(x)
    }
    
    // 7. Friend functions
    public(friend) fun friend_only(): u64 {
        42
    }
    
    // 8. Entry functions (transaction endpoints)
    public entry fun transaction_endpoint(account: &signer) {
        // ...
    }
    
    // 9. View functions (queries)
    #[view]
    public fun query(addr: address): u64 {
        0
    }
    
    // 10. Test functions
    #[test]
    fun unit_test() {
        assert!(external_api(5) == 10, 0);
    }
    
    // 11. Test helper
    #[test_only]
    public fun test_helper(): u64 {
        999
    }
}
```

---

## Package Structure

### Move.toml Configuration

```toml
# Move.toml - Package configuration

[package]
name = "defi_protocol"
version = "1.0.0"
authors = ["Dev Team <dev@example.com>"]
license = "MIT"

[addresses]
# Named addresses ที่ใช้ใน code
defi_protocol = "0xCAFE"
admin = "0x1"

[dev-addresses]
# Addresses สำหรับ testing เท่านั้น
defi_protocol = "0xBEEF"

[dependencies]
# Dependencies จาก git
AptosFramework = { 
    git = "https://github.com/aptos-labs/aptos-core.git",
    rev = "mainnet",
    subdir = "aptos-move/framework/aptos-framework"
}

# Local dependencies
MathLib = { local = "../math_lib" }

[dev-dependencies]
# Dependencies สำหรับ testing เท่านั้น
AptosFrameworkTest = {
    git = "https://github.com/aptos-labs/aptos-core.git",
    rev = "mainnet",
    subdir = "aptos-move/framework/aptos-framework"
}
```

### Recommended Project Structure

```
defi_protocol/
├── Move.toml
├── README.md
├── sources/
│   ├── core/
│   │   ├── math.move          # Math utilities
│   │   ├── errors.move        # Error codes
│   │   └── events.move        # Event definitions
│   ├── token/
│   │   ├── token.move         # Token implementation
│   │   ├── treasury.move      # Treasury management
│   │   └── vesting.move       # Vesting schedule
│   ├── defi/
│   │   ├── pool.move          # Liquidity pool
│   │   ├── swap.move          # Swap logic
│   │   └── oracle.move        # Price oracle
│   └── governance/
│       ├── voting.move        # Voting mechanism
│       └── proposal.move      # Proposal management
├── tests/
│   ├── token_tests.move
│   ├── pool_tests.move
│   └── integration_tests.move
└── scripts/
    ├── initialize.move        # Deployment scripts
    └── migrate.move           # Migration scripts
```

---

## Import และ Use

```move
module learning::imports {
    // ============================================
    // Basic imports
    // ============================================
    
    // Import ทั้ง module
    use std::signer;
    use std::vector;
    use std::string;
    
    // Import specific items
    use std::string::{String, utf8};
    use std::vector::{push_back, pop_back, length};
    
    // Import ด้วย alias
    use std::string::{Self as str, String as Str};
    
    // Import จาก Aptos framework
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::aptos_coin::AptosCoin;
    use aptos_framework::account;
    use aptos_framework::timestamp;
    
    // ============================================
    // Using imported items
    // ============================================
    
    struct TokenInfo has store, drop {
        name: String,
        symbol: String,
        decimals: u8,
        total_supply: u64,
    }
    
    public fun create_token_info(
        name: vector<u8>,
        symbol: vector<u8>,
        decimals: u8,
        total_supply: u64,
    ): TokenInfo {
        TokenInfo {
            name: string::utf8(name),    // ใช้ module prefix
            symbol: utf8(symbol),         // ใช้ imported function
            decimals,
            total_supply,
        }
    }
    
    // ============================================
    // Namespace resolution
    // ============================================
    
    // ถ้า import ชนกัน ใช้ full path
    public fun resolve_conflict() {
        // สมมติมี 2 modules ที่มี create function
        // let a = module_a::create();
        // let b = module_b::create();
    }
}
```

---

## Visibility

```move
// ============================================
// Module ที่เปิดเผย API ต่างๆ
// ============================================
module learning::library {
    
    // ============================================
    // PRIVATE (default) - ใช้ได้แค่ภายใน module
    // ============================================
    
    fun private_compute(x: u64, y: u64): u64 {
        // ใช้ได้แค่ใน module นี้
        x * y + x + y
    }
    
    struct InternalState {
        counter: u64,
    }
    
    // ============================================
    // PUBLIC - ใช้ได้จากทุก module
    // ============================================
    
    public fun add(a: u64, b: u64): u64 {
        a + b
    }
    
    public fun multiply(a: u64, b: u64): u64 {
        private_compute(a, b)  // เรียก private function ได้
    }
    
    // Public struct (แต่ fields ไม่ public โดย default)
    public struct Config has drop {
        fee_bps: u64,    // private field
        max_supply: u64, // private field
    }
    
    // Public constructor
    public fun create_config(fee_bps: u64, max_supply: u64): Config {
        Config { fee_bps, max_supply }
    }
    
    // Public getter (field access control)
    public fun get_fee_bps(config: &Config): u64 {
        config.fee_bps
    }
    
    // ============================================
    // PUBLIC ENTRY - เรียกได้จาก transaction
    // ============================================
    
    public entry fun initialize(account: &signer) {
        // Setup initial state
    }
    
    // ============================================
    // VIEW - Pure reads
    // ============================================
    
    #[view]
    public fun get_info(): u64 {
        0
    }
}

// ============================================
// External module ที่ใช้ library
// ============================================
module learning::consumer {
    use learning::library;
    
    public fun use_library(): u64 {
        let sum = library::add(10, 20);       // ✅ public
        let product = library::multiply(3, 4); // ✅ public
        // library::private_compute(1, 2);     // ❌ ERROR: private
        
        let config = library::create_config(30, 1000);
        let fee = library::get_fee_bps(&config);  // ✅ public getter
        // config.fee_bps  // ❌ ERROR: private field
        
        sum + product + fee
    }
}
```

---

## Friend Modules

```move
// ============================================
// Module ที่ declare friends
// ============================================
module learning::core_module {
    
    // Declare ว่า module ใดบ้างที่เป็น friend
    friend learning::trusted_module;
    friend learning::another_trusted;
    
    // Friend-only function
    public(friend) fun internal_operation(): u64 {
        // เฉพาะ trusted_module และ another_trusted เท่านั้นที่เรียกได้
        secret_value()
    }
    
    fun secret_value(): u64 {
        42
    }
    
    // ตัวอย่าง: access control ที่ละเอียดขึ้น
    public(friend) fun privileged_mint(amount: u64): u64 {
        // Mint tokens - เฉพาะ friends เท่านั้น
        amount
    }
}

// ============================================
// Trusted module ที่ใช้ friend functions
// ============================================
module learning::trusted_module {
    use learning::core_module;
    
    public fun use_friend(): u64 {
        core_module::internal_operation()  // ✅ เพราะเป็น friend
    }
}

// ============================================
// Untrusted module ที่ไม่สามารถใช้ friend functions
// ============================================
module learning::untrusted_module {
    use learning::core_module;
    
    public fun try_to_use_friend() {
        // core_module::internal_operation()  // ❌ ERROR: not a friend
    }
}
```

### Use Case: Token with Controlled Mint

```move
module defi::token {
    friend defi::minter;    // เฉพาะ minter module เท่านั้นที่ mint ได้
    friend defi::bridge;    // bridge สามารถ mint ด้วย
    
    struct TokenCap has key {
        total_minted: u64,
        max_supply: u64,
    }
    
    // เฉพาะ friends เท่านั้นที่เรียกได้
    public(friend) fun mint_internal(amount: u64): u64 {
        // ...
        amount
    }
    
    // Public burn - ทุกคนสามารถ burn ได้
    public fun burn(amount: u64) {
        // ...
    }
}

module defi::minter {
    use defi::token;
    
    public entry fun mint_tokens(admin: &signer, amount: u64) {
        // ตรวจสอบ authorization
        let minted = token::mint_internal(amount);  // ✅ เพราะเป็น friend
        // ...
    }
}
```

---

## Module Dependencies

```move
// ============================================
// Layer 1: Core utilities (ไม่ depend บน module อื่น)
// ============================================
module protocol::math {
    
    public fun sqrt(n: u128): u128 {
        if (n == 0) return 0;
        let mut x = n;
        let mut y = (x + 1) / 2;
        while (y < x) {
            x = y;
            y = (x + n / x) / 2;
        };
        x
    }
    
    public fun mul_div(a: u64, b: u64, c: u64): u64 {
        assert!(c > 0, 1);
        ((a as u128) * (b as u128) / (c as u128)) as u64
    }
    
    public fun safe_add(a: u64, b: u64): u64 {
        assert!(a <= 18446744073709551615u64 - b, 1);
        a + b
    }
}

// ============================================
// Layer 2: Core protocol (depend บน math)
// ============================================
module protocol::pool {
    use protocol::math;
    
    const E_ZERO_LIQUIDITY: u64 = 1;
    
    struct Pool has key {
        reserve_a: u64,
        reserve_b: u64,
        total_supply: u64,
    }
    
    public fun get_amount_out(
        amount_in: u64,
        reserve_in: u64,
        reserve_out: u64,
    ): u64 {
        let numerator = math::mul_div(amount_in, reserve_out, 1);
        let denominator = math::safe_add(reserve_in, amount_in);
        numerator / denominator
    }
    
    public fun get_liquidity(
        amount_a: u64,
        amount_b: u64,
        total_supply: u64,
        reserve_a: u64,
        reserve_b: u64,
    ): u64 {
        if (total_supply == 0) {
            let product = (amount_a as u128) * (amount_b as u128);
            math::sqrt(product) as u64
        } else {
            let liq_a = math::mul_div(amount_a, total_supply, reserve_a);
            let liq_b = math::mul_div(amount_b, total_supply, reserve_b);
            if (liq_a < liq_b) { liq_a } else { liq_b }
        }
    }
}

// ============================================
// Layer 3: Router (depend บน pool)
// ============================================
module protocol::router {
    use protocol::pool;
    use protocol::math;
    
    public fun multi_hop_swap(
        amount_in: u64,
        reserve_pairs: &vector<(u64, u64)>,  // (reserve_in, reserve_out) pairs
    ): u64 {
        let current_amount = amount_in;
        let i = 0u64;
        let len = vector::length(reserve_pairs);
        
        while (i < len) {
            let (reserve_in, reserve_out) = *vector::borrow(reserve_pairs, i);
            current_amount = pool::get_amount_out(
                current_amount,
                reserve_in,
                reserve_out,
            );
            i = i + 1;
        };
        
        current_amount
    }
}
```

---

## ตัวอย่าง: Multi-module Protocol

### แผนภาพ Architecture

```
protocol/
├── core/
│   ├── errors.move      ← shared error codes
│   ├── events.move      ← event definitions
│   └── math.move        ← math utilities
├── token/
│   ├── base_token.move  ← token standard
│   └── lp_token.move    ← LP token
├── pool/
│   ├── pool_core.move   ← pool logic
│   └── pool_math.move   ← AMM math
└── governance/
    └── governor.move    ← governance

Dependencies:
  governance → pool, token
  pool → token, math
  token → math
  math → (none)
```

```move
// ============================================
// errors.move - Shared error codes
// ============================================
module protocol::errors {
    // Auth
    const E_NOT_AUTHORIZED: u64 = 1000;
    const E_NOT_ADMIN: u64 = 1001;
    
    // Input validation
    const E_ZERO_AMOUNT: u64 = 2000;
    const E_INVALID_ADDRESS: u64 = 2001;
    const E_SLIPPAGE_EXCEEDED: u64 = 2002;
    const E_DEADLINE_EXCEEDED: u64 = 2003;
    
    // State
    const E_PAUSED: u64 = 3000;
    const E_NOT_INITIALIZED: u64 = 3001;
    const E_ALREADY_INITIALIZED: u64 = 3002;
    const E_INSUFFICIENT_LIQUIDITY: u64 = 3003;
    
    // Getters (ถ้าต้องการ access จาก modules อื่น)
    public fun not_authorized(): u64 { E_NOT_AUTHORIZED }
    public fun zero_amount(): u64 { E_ZERO_AMOUNT }
    public fun paused(): u64 { E_PAUSED }
    public fun insufficient_liquidity(): u64 { E_INSUFFICIENT_LIQUIDITY }
}

// ============================================
// events.move - Event definitions
// ============================================
module protocol::events {
    use std::string::String;
    
    // Token events
    struct TokenMinted has drop, store {
        recipient: address,
        amount: u64,
        total_supply: u64,
    }
    
    struct TokenBurned has drop, store {
        sender: address,
        amount: u64,
        total_supply: u64,
    }
    
    struct TokenTransferred has drop, store {
        from: address,
        to: address,
        amount: u64,
    }
    
    // Pool events
    struct LiquidityAdded has drop, store {
        provider: address,
        amount_a: u64,
        amount_b: u64,
        shares_minted: u64,
    }
    
    struct SwapExecuted has drop, store {
        user: address,
        token_in: String,
        token_out: String,
        amount_in: u64,
        amount_out: u64,
        fee_amount: u64,
    }
}

// ============================================
// math.move - Mathematical utilities
// ============================================
module protocol::math {
    
    const MAX_U64: u64 = 18446744073709551615;
    const MAX_U128: u128 = 340282366920938463463374607431768211455;
    
    // Safe arithmetic
    public fun safe_add(a: u64, b: u64): u64 {
        assert!(a <= MAX_U64 - b, 1);
        a + b
    }
    
    public fun safe_mul(a: u64, b: u64): u64 {
        if (a == 0 || b == 0) return 0;
        let result = (a as u128) * (b as u128);
        assert!(result <= MAX_U64 as u128, 1);
        result as u64
    }
    
    // Precise division with scaling
    public fun mul_div(a: u64, b: u64, c: u64): u64 {
        assert!(c > 0, 1);
        let result = (a as u128) * (b as u128) / (c as u128);
        assert!(result <= MAX_U64 as u128, 1);
        result as u64
    }
    
    // Integer square root
    public fun sqrt(n: u128): u128 {
        if (n == 0) return 0;
        let mut x = n;
        let mut y = (x + 1) / 2;
        while (y < x) {
            x = y;
            y = (x + n / x) / 2;
        };
        x
    }
    
    // Power function
    public fun pow(base: u64, exp: u64): u64 {
        if (exp == 0) return 1;
        let mut result = 1u64;
        let mut b = base;
        let mut e = exp;
        
        while (e > 0) {
            if (e % 2 == 1) {
                result = safe_mul(result, b);
            };
            b = safe_mul(b, b);
            e = e / 2;
        };
        
        result
    }
    
    // Percentage calculation (with 4 decimal precision)
    public fun percentage(value: u64, bps: u64): u64 {
        mul_div(value, bps, 10000)
    }
    
    #[test]
    fun test_math() {
        assert!(safe_add(100, 200) == 300, 0);
        assert!(safe_mul(100, 200) == 20000, 1);
        assert!(mul_div(100, 3, 10) == 30, 2);
        assert!(sqrt(100u128) == 10, 3);
        assert!(sqrt(144u128) == 12, 4);
        assert!(pow(2, 10) == 1024, 5);
        assert!(percentage(1000, 30) == 3, 6);  // 0.30% of 1000 = 3
    }
}

// ============================================
// base_token.move - Token standard
// ============================================
module protocol::base_token {
    use std::signer;
    use std::string::{Self, String};
    use protocol::math;
    use protocol::errors;
    
    friend protocol::lp_token;
    friend protocol::pool_core;
    
    struct TokenConfig has key {
        name: String,
        symbol: String,
        decimals: u8,
        max_supply: u64,
        total_supply: u64,
        is_paused: bool,
        admin: address,
    }
    
    struct Balance has key {
        amount: u64,
    }
    
    // Initialize token
    public entry fun initialize(
        admin: &signer,
        name: vector<u8>,
        symbol: vector<u8>,
        decimals: u8,
        max_supply: u64,
    ) {
        let addr = signer::address_of(admin);
        assert!(!exists<TokenConfig>(addr), errors::already_initialized());
        
        move_to(admin, TokenConfig {
            name: string::utf8(name),
            symbol: string::utf8(symbol),
            decimals,
            max_supply,
            total_supply: 0,
            is_paused: false,
            admin: addr,
        });
    }
    
    // Internal mint (friends only)
    public(friend) fun mint_to(
        config_addr: address,
        recipient: address,
        amount: u64,
    ) acquires TokenConfig, Balance {
        assert!(amount > 0, errors::zero_amount());
        
        let config = borrow_global_mut<TokenConfig>(config_addr);
        assert!(!config.is_paused, errors::paused());
        
        let new_supply = math::safe_add(config.total_supply, amount);
        assert!(new_supply <= config.max_supply, 1);
        
        config.total_supply = new_supply;
        
        // Add to recipient balance
        if (exists<Balance>(recipient)) {
            let balance = borrow_global_mut<Balance>(recipient);
            balance.amount = math::safe_add(balance.amount, amount);
        } else {
            // สร้าง balance ใหม่ - ต้องการ signer ของ recipient
            // ใน production ใช้ resource_account หรือ global table
        }
    }
    
    // Public transfer
    public entry fun transfer(
        from: &signer,
        to: address,
        amount: u64,
        config_addr: address,
    ) acquires TokenConfig, Balance {
        let from_addr = signer::address_of(from);
        assert!(amount > 0, errors::zero_amount());
        
        let config = borrow_global<TokenConfig>(config_addr);
        assert!(!config.is_paused, errors::paused());
        
        // Deduct from sender
        let from_balance = borrow_global_mut<Balance>(from_addr);
        assert!(from_balance.amount >= amount, 1);
        from_balance.amount = from_balance.amount - amount;
        
        // Add to recipient
        if (exists<Balance>(to)) {
            let to_balance = borrow_global_mut<Balance>(to);
            to_balance.amount = math::safe_add(to_balance.amount, amount);
        }
    }
    
    // View functions
    #[view]
    public fun total_supply(config_addr: address): u64 acquires TokenConfig {
        borrow_global<TokenConfig>(config_addr).total_supply
    }
    
    #[view]
    public fun balance_of(addr: address): u64 acquires Balance {
        if (exists<Balance>(addr)) {
            borrow_global<Balance>(addr).amount
        } else {
            0
        }
    }
    
    #[view]
    public fun is_paused(config_addr: address): bool acquires TokenConfig {
        borrow_global<TokenConfig>(config_addr).is_paused
    }
}
```

---

## Module Best Practices

### 1. Circular Dependencies หลีกเลี่ยง

```
❌ ไม่ดี:
  module_a → module_b → module_a  (circular!)

✅ ดี:
  common utilities (no dependencies)
       ↑
  core modules (depend on utilities)
       ↑
  feature modules (depend on core)
       ↑
  entry points (depend on features)
```

### 2. Module Size

```
✅ แนะนำ:
- แต่ละ module มีความรับผิดชอบเดียว (Single Responsibility)
- ไม่เกิน ~500 บรรทัดต่อ module
- ถ้าใหญ่เกิน แยกเป็น sub-modules

❌ หลีกเลี่ยง:
- God module (module เดียวทำทุกอย่าง)
- Micro modules (module เล็กเกินไปจนไม่มี logic)
```

### 3. Public API Design

```move
// ✅ ดี: Public API ที่ชัดเจน
module protocol::pool {
    // --- Configuration ---
    public entry fun initialize(...) { }
    public entry fun update_config(...) { }
    
    // --- Core operations ---
    public entry fun add_liquidity(...) { }
    public entry fun remove_liquidity(...) { }
    public entry fun swap(...) { }
    
    // --- Queries ---
    #[view] public fun get_price(...) { }
    #[view] public fun get_reserves(...) { }
    #[view] public fun get_tvl(...) { }
    
    // --- Internal (private) ---
    fun calculate_output(...) { }
    fun update_reserves(...) { }
}
```

---

## สรุป

| แนวคิด | สิ่งที่เรียนรู้ |
|-------|--------------|
| Module | `module address::name { }` |
| Package | `Move.toml`, directory structure |
| Imports | `use`, aliases, `Self` |
| Visibility | private, public, public(friend), entry |
| Friends | `friend module::name` |
| Dependencies | layered architecture |

---

## แบบฝึกหัด

### Exercise 7.1: Module Organization
แยก Counter contract ออกเป็น 3 modules:
1. `counter_errors` - error codes
2. `counter_core` - core logic
3. `counter_ui` - entry functions

### Exercise 7.2: Friend Pattern
สร้าง `vault` module ที่:
- เฉพาะ `strategy` module เท่านั้นที่ withdraw ได้
- ทุกคน deposit ได้
- ทุกคน query balance ได้

### Exercise 7.3: Package Design
ออกแบบ package structure สำหรับ NFT marketplace:
- NFT token
- Marketplace listing
- Auction
- Royalty management

---

**ก่อนหน้า**: [Part 06 - Control Flow](part-06-control-flow.md)  
**ต่อไป**: [Part 08 - Resources และ Linear Types →](part-08-resources.md)
