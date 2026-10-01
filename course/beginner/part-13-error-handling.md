# Part 13: Error Handling

## สารบัญ
- [Error Codes และ Constants](#error-codes-และ-constants)
- [assert! Macro](#assert-macro)
- [abort Statement](#abort-statement)
- [Error Naming Conventions](#error-naming-conventions)
- [Error Categories](#error-categories)
- [ตัวอย่างโปรแกรม: DeFi Error Handling](#ตัวอย่างโปรแกรม-defi-error-handling)

---

## Error Codes และ Constants

```move
module learning::error_codes {
    
    // ============================================
    // Basic Error Constants
    // ============================================
    
    // ใช้ prefix E_ สำหรับ errors
    // ใช้ UPPER_SNAKE_CASE
    
    const E_NOT_AUTHORIZED: u64 = 1;
    const E_INSUFFICIENT_BALANCE: u64 = 2;
    const E_ZERO_AMOUNT: u64 = 3;
    const E_PAUSED: u64 = 4;
    const E_ALREADY_EXISTS: u64 = 5;
    const E_NOT_FOUND: u64 = 6;
    const E_EXPIRED: u64 = 7;
    const E_INVALID_ARGUMENT: u64 = 8;
    const E_ARITHMETIC_OVERFLOW: u64 = 9;
    
    // ============================================
    // Categorized Error Codes (แบบ namespace)
    // ============================================
    
    // Auth errors: 100-199
    const E_UNAUTHORIZED: u64 = 100;
    const E_WRONG_SIGNATURE: u64 = 101;
    const E_SESSION_EXPIRED: u64 = 102;
    const E_BANNED: u64 = 103;
    
    // Token errors: 200-299
    const E_TOKEN_OVERFLOW: u64 = 200;
    const E_TOKEN_UNDERFLOW: u64 = 201;
    const E_TOKEN_NOT_REGISTERED: u64 = 202;
    const E_TOKEN_PAUSED: u64 = 203;
    
    // Pool errors: 300-399
    const E_POOL_NOT_INITIALIZED: u64 = 300;
    const E_POOL_FULL: u64 = 301;
    const E_SLIPPAGE_TOO_HIGH: u64 = 302;
    const E_INSUFFICIENT_LIQUIDITY: u64 = 303;
    
    // Governance errors: 400-499
    const E_PROPOSAL_NOT_FOUND: u64 = 400;
    const E_VOTING_CLOSED: u64 = 401;
    const E_ALREADY_VOTED: u64 = 402;
    const E_INSUFFICIENT_VOTING_POWER: u64 = 403;
}
```

---

## assert! Macro

```move
module learning::assert_usage {
    use std::signer;
    
    const E_ZERO_AMOUNT: u64 = 1;
    const E_OVERFLOW: u64 = 2;
    const E_NOT_AUTHORIZED: u64 = 3;
    const E_INVALID_STATE: u64 = 4;
    
    // ============================================
    // Basic assert!
    // ============================================
    
    public fun transfer(amount: u64, balance: u64): u64 {
        // assert!(condition, error_code)
        assert!(amount > 0, E_ZERO_AMOUNT);
        assert!(balance >= amount, 100);  // inline error code
        
        balance - amount
    }
    
    // ============================================
    // Multiple asserts
    // ============================================
    
    public fun validate_transfer(
        from: &signer,
        to: address,
        amount: u64,
        max_amount: u64,
    ) {
        let from_addr = signer::address_of(from);
        
        assert!(from_addr != to, E_INVALID_STATE);   // can't self-transfer
        assert!(amount > 0, E_ZERO_AMOUNT);           // positive amount
        assert!(amount <= max_amount, E_OVERFLOW);    // within limits
        // ... more validations
    }
    
    // ============================================
    // assert! with complex conditions
    // ============================================
    
    public fun validate_range(value: u64, min: u64, max: u64) {
        assert!(value >= min && value <= max, E_INVALID_STATE);
    }
    
    // Complex boolean
    public fun validate_access(is_admin: bool, is_minter: bool, is_paused: bool) {
        assert!(is_admin || is_minter, E_NOT_AUTHORIZED);
        assert!(!is_paused, E_INVALID_STATE);
    }
    
    // ============================================
    // Helper assertion functions
    // ============================================
    
    public fun assert_not_zero(amount: u64) {
        assert!(amount > 0, E_ZERO_AMOUNT);
    }
    
    public fun assert_is_admin(caller: address, admin: address) {
        assert!(caller == admin, E_NOT_AUTHORIZED);
    }
    
    public fun safe_add(a: u64, b: u64): u64 {
        // ป้องกัน overflow ด้วย assert
        let max_u64: u64 = 18446744073709551615u64;
        assert!(a <= max_u64 - b, E_OVERFLOW);
        a + b
    }
    
    public fun safe_sub(a: u64, b: u64): u64 {
        assert!(a >= b, E_OVERFLOW);
        a - b
    }
    
    public fun safe_mul(a: u64, b: u64): u64 {
        if (b == 0) return 0;
        let max_u64: u64 = 18446744073709551615u64;
        assert!(a <= max_u64 / b, E_OVERFLOW);
        a * b
    }
}
```

---

## abort Statement

```move
module learning::abort_usage {
    
    const E_INVALID: u64 = 1;
    
    // ============================================
    // Direct abort
    // ============================================
    
    public fun must_be_positive(value: i64): u64 {
        if (value <= 0) {
            abort E_INVALID
        };
        // Convert safely
        (value as u64)
    }
    
    // ============================================
    // abort in match-like structures
    // ============================================
    
    const STATE_ACTIVE: u8 = 1;
    const STATE_PAUSED: u8 = 2;
    const STATE_CLOSED: u8 = 3;
    
    const E_UNKNOWN_STATE: u64 = 10;
    const E_PAUSED: u64 = 11;
    const E_CLOSED: u64 = 12;
    
    public fun check_state(state: u8) {
        if (state == STATE_ACTIVE) {
            // proceed
        } else if (state == STATE_PAUSED) {
            abort E_PAUSED
        } else if (state == STATE_CLOSED) {
            abort E_CLOSED
        } else {
            abort E_UNKNOWN_STATE
        }
    }
    
    // ============================================
    // abort vs assert! equivalence
    // ============================================
    
    // These are equivalent:
    public fun check_with_assert(condition: bool) {
        assert!(condition, E_INVALID);
    }
    
    public fun check_with_abort(condition: bool) {
        if (!condition) {
            abort E_INVALID
        }
    }
    
    // assert! is syntactic sugar for: if (!cond) abort code;
    
    // ============================================
    // Error propagation patterns
    // ============================================
    
    public fun outer_operation(value: u64) {
        // Errors from inner calls propagate automatically
        inner_operation(value);
        // ถ้า inner_operation abort, outer_operation ก็ abort ด้วย
    }
    
    public fun inner_operation(value: u64) {
        assert!(value > 0, E_INVALID);
        // ... more logic
    }
    
    // ============================================
    // Conditional abort patterns
    // ============================================
    
    public fun conditional_abort(
        value: u64,
        should_check: bool,
        limit: u64,
    ) {
        if (should_check) {
            assert!(value <= limit, E_INVALID);
        }
        // continue if check passes or check is skipped
    }
}
```

---

## Error Naming Conventions

```move
module learning::error_conventions {
    
    // ============================================
    // Convention 1: Module-specific prefix
    // ============================================
    
    // Module: token
    const TOKEN_E_NOT_REGISTERED: u64 = 1;
    const TOKEN_E_INSUFFICIENT_SUPPLY: u64 = 2;
    
    // Module: pool
    const POOL_E_NOT_INITIALIZED: u64 = 1;
    const POOL_E_ZERO_LIQUIDITY: u64 = 2;
    
    // ============================================
    // Convention 2: Simple E_ prefix (most common)
    // ============================================
    
    const E_NOT_AUTHORIZED: u64 = 1;
    const E_INVALID_INPUT: u64 = 2;
    
    // ============================================
    // Convention 3: aptos_std::error module
    // ============================================
    
    use aptos_std::error;
    
    // aptos_std::error provides categorized error codes
    // error::invalid_argument(code)  → category 0x1
    // error::out_of_range(code)      → category 0x2
    // error::invalid_state(code)     → category 0x3
    // error::unauthenticated(code)   → category 0x4
    // error::permission_denied(code) → category 0x5
    // error::not_found(code)         → category 0x6
    // error::aborted(code)           → category 0x7
    // error::already_exists(code)    → category 0x8
    // error::resource_exhausted(code) → category 0xb
    
    const ENOT_AUTHORIZED: u64 = 1;
    const EINVALID_AMOUNT: u64 = 2;
    const EINSUFFICIENT_BALANCE: u64 = 3;
    const ETOKEN_NOT_FOUND: u64 = 4;
    
    public fun transfer_with_aptos_errors(
        caller: address,
        admin: address,
        amount: u64,
        balance: u64,
    ) {
        assert!(caller == admin,
            error::permission_denied(ENOT_AUTHORIZED));
        assert!(amount > 0,
            error::invalid_argument(EINVALID_AMOUNT));
        assert!(balance >= amount,
            error::invalid_state(EINSUFFICIENT_BALANCE));
    }
}
```

---

## Error Categories

```move
module learning::error_categories {
    
    // ============================================
    // Input Validation Errors
    // ============================================
    
    const E_ZERO_AMOUNT: u64 = 1;
    const E_INVALID_ADDRESS: u64 = 2;
    const E_OUT_OF_RANGE: u64 = 3;
    const E_EMPTY_INPUT: u64 = 4;
    
    public fun validate_inputs(
        amount: u64,
        recipient: address,
        slippage_bps: u64,
    ) {
        assert!(amount > 0, E_ZERO_AMOUNT);
        assert!(recipient != @0x0, E_INVALID_ADDRESS);
        assert!(slippage_bps <= 10_000, E_OUT_OF_RANGE);  // max 100%
    }
    
    // ============================================
    // State Errors
    // ============================================
    
    const E_PAUSED: u64 = 10;
    const E_ALREADY_INITIALIZED: u64 = 11;
    const E_NOT_INITIALIZED: u64 = 12;
    const E_WRONG_STATE: u64 = 13;
    
    struct Protocol has key {
        is_paused: bool,
        version: u64,
    }
    
    public fun check_protocol_state(addr: address) acquires Protocol {
        assert!(exists<Protocol>(addr), E_NOT_INITIALIZED);
        let protocol = borrow_global<Protocol>(addr);
        assert!(!protocol.is_paused, E_PAUSED);
    }
    
    // ============================================
    // Authorization Errors
    // ============================================
    
    const E_NOT_OWNER: u64 = 20;
    const E_NOT_ADMIN: u64 = 21;
    const E_NOT_MINTER: u64 = 22;
    const E_INSUFFICIENT_ROLE: u64 = 23;
    
    struct Roles has key {
        owner: address,
        admin: address,
        minter: address,
    }
    
    public fun check_owner_auth(caller: address, roles_addr: address) acquires Roles {
        let roles = borrow_global<Roles>(roles_addr);
        assert!(caller == roles.owner, E_NOT_OWNER);
    }
    
    // ============================================
    // Resource Errors
    // ============================================
    
    const E_INSUFFICIENT_BALANCE: u64 = 30;
    const E_INSUFFICIENT_LIQUIDITY: u64 = 31;
    const E_POOL_EMPTY: u64 = 32;
    const E_QUOTA_EXCEEDED: u64 = 33;
    
    // ============================================
    // Math Errors
    // ============================================
    
    const E_DIVISION_BY_ZERO: u64 = 40;
    const E_ARITHMETIC_OVERFLOW: u64 = 41;
    const E_ARITHMETIC_UNDERFLOW: u64 = 42;
    
    public fun safe_div(a: u64, b: u64): u64 {
        assert!(b != 0, E_DIVISION_BY_ZERO);
        a / b
    }
    
    // ============================================
    // Timeout/Expiry Errors
    // ============================================
    
    const E_EXPIRED: u64 = 50;
    const E_TOO_EARLY: u64 = 51;
    const E_DEADLINE_PASSED: u64 = 52;
    
    public fun check_deadline(current_time: u64, deadline: u64) {
        assert!(current_time <= deadline, E_DEADLINE_PASSED);
    }
    
    public fun check_start_time(current_time: u64, start_time: u64) {
        assert!(current_time >= start_time, E_TOO_EARLY);
    }
}
```

---

## ตัวอย่างโปรแกรม: DeFi Error Handling

```move
module learning::defi_errors {
    use std::signer;
    use aptos_std::error;
    
    // ============================================
    // Error Codes - DeFi Protocol
    // ============================================
    
    // Input validation (1-19)
    const EZERO_AMOUNT: u64 = 1;
    const EINVALID_SLIPPAGE: u64 = 2;
    const EINVALID_DEADLINE: u64 = 3;
    const EINVALID_TOKEN: u64 = 4;
    const ESELF_TRANSFER: u64 = 5;
    
    // State errors (20-39)
    const EPROTOCOL_PAUSED: u64 = 20;
    const EPOOL_NOT_INITIALIZED: u64 = 21;
    const EPOOL_ALREADY_EXISTS: u64 = 22;
    const EINSUFFICIENT_LIQUIDITY: u64 = 23;
    
    // Auth errors (40-59)
    const ENOT_ADMIN: u64 = 40;
    const ENOT_LIQUIDITY_PROVIDER: u64 = 41;
    const EUNAUTHORIZED_CALLER: u64 = 42;
    
    // Math errors (60-79)
    const EOVERFLOW: u64 = 60;
    const EUNDERFLOW: u64 = 61;
    const EDIVISION_BY_ZERO: u64 = 62;
    const ESLIPPAGE_EXCEEDED: u64 = 63;
    
    // Resource errors (80-99)
    const EINSUFFICIENT_BALANCE: u64 = 80;
    const EINSUFFICIENT_ALLOWANCE: u64 = 81;
    const EACCOUNT_NOT_REGISTERED: u64 = 82;
    
    // Deadline errors (100-119)
    const EDEADLINE_PASSED: u64 = 100;
    const ETOO_EARLY: u64 = 101;
    const ELOCKING_PERIOD: u64 = 102;
    
    // ============================================
    // Protocol State
    // ============================================
    
    struct ProtocolConfig has key {
        admin: address,
        is_paused: bool,
        fee_bps: u64,
        max_slippage_bps: u64,
    }
    
    struct LiquidityPool has key {
        reserve_a: u64,
        reserve_b: u64,
        lp_total_supply: u64,
        fee_bps: u64,
        is_active: bool,
    }
    
    struct UserAccount has key {
        balance_a: u64,
        balance_b: u64,
        lp_balance: u64,
        last_action_time: u64,
    }
    
    // ============================================
    // Validation Helpers
    // ============================================
    
    fun validate_not_paused(config: &ProtocolConfig) {
        assert!(!config.is_paused, error::invalid_state(EPROTOCOL_PAUSED));
    }
    
    fun validate_admin(caller: address, config: &ProtocolConfig) {
        assert!(caller == config.admin, error::permission_denied(ENOT_ADMIN));
    }
    
    fun validate_amount(amount: u64) {
        assert!(amount > 0, error::invalid_argument(EZERO_AMOUNT));
    }
    
    fun validate_slippage(slippage_bps: u64) {
        assert!(slippage_bps <= 10_000, error::invalid_argument(EINVALID_SLIPPAGE));
    }
    
    fun validate_deadline(current_time: u64, deadline: u64) {
        assert!(current_time <= deadline, error::invalid_state(EDEADLINE_PASSED));
    }
    
    fun validate_pool(pool: &LiquidityPool) {
        assert!(pool.is_active, error::invalid_state(EPOOL_NOT_INITIALIZED));
        assert!(pool.reserve_a > 0 && pool.reserve_b > 0, error::invalid_state(EINSUFFICIENT_LIQUIDITY));
    }
    
    // ============================================
    // Safe Math
    // ============================================
    
    fun safe_add(a: u64, b: u64): u64 {
        let max: u64 = 18446744073709551615u64;
        assert!(a <= max - b, error::out_of_range(EOVERFLOW));
        a + b
    }
    
    fun safe_sub(a: u64, b: u64): u64 {
        assert!(a >= b, error::out_of_range(EUNDERFLOW));
        a - b
    }
    
    fun safe_mul(a: u64, b: u64): u64 {
        if (b == 0) return 0;
        let max: u64 = 18446744073709551615u64;
        assert!(a <= max / b, error::out_of_range(EOVERFLOW));
        a * b
    }
    
    fun safe_div(a: u64, b: u64): u64 {
        assert!(b != 0, error::invalid_argument(EDIVISION_BY_ZERO));
        a / b
    }
    
    // ============================================
    // Main Protocol Functions
    // ============================================
    
    public entry fun swap_exact_in(
        user: &signer,
        config_addr: address,
        pool_addr: address,
        amount_in: u64,
        min_amount_out: u64,
        deadline: u64,
        current_time: u64,
    ) acquires ProtocolConfig, LiquidityPool, UserAccount {
        let user_addr = signer::address_of(user);
        
        // 1. Input validation
        validate_amount(amount_in);
        assert!(min_amount_out > 0, error::invalid_argument(EZERO_AMOUNT));
        validate_deadline(current_time, deadline);
        
        // 2. Protocol state check
        assert!(exists<ProtocolConfig>(config_addr), error::not_found(EPOOL_NOT_INITIALIZED));
        let config = borrow_global<ProtocolConfig>(config_addr);
        validate_not_paused(config);
        
        // 3. Pool state check
        assert!(exists<LiquidityPool>(pool_addr), error::not_found(EPOOL_NOT_INITIALIZED));
        let pool = borrow_global_mut<LiquidityPool>(pool_addr);
        validate_pool(pool);
        
        // 4. User account check
        assert!(exists<UserAccount>(user_addr), error::not_found(EACCOUNT_NOT_REGISTERED));
        let user_account = borrow_global_mut<UserAccount>(user_addr);
        assert!(user_account.balance_a >= amount_in, error::invalid_state(EINSUFFICIENT_BALANCE));
        
        // 5. Calculate output (x * y = k formula)
        let fee_amount = safe_mul(amount_in, pool.fee_bps) / 10_000;
        let amount_in_after_fee = safe_sub(amount_in, fee_amount);
        
        let numerator = safe_mul(amount_in_after_fee, pool.reserve_b);
        let denominator = safe_add(pool.reserve_a, amount_in_after_fee);
        let amount_out = safe_div(numerator, denominator);
        
        // 6. Slippage check
        assert!(amount_out >= min_amount_out, error::invalid_state(ESLIPPAGE_EXCEEDED));
        
        // 7. Execute swap
        user_account.balance_a = safe_sub(user_account.balance_a, amount_in);
        user_account.balance_b = safe_add(user_account.balance_b, amount_out);
        
        pool.reserve_a = safe_add(pool.reserve_a, amount_in);
        pool.reserve_b = safe_sub(pool.reserve_b, amount_out);
    }
    
    public entry fun add_liquidity(
        user: &signer,
        pool_addr: address,
        config_addr: address,
        amount_a: u64,
        amount_b: u64,
        min_lp: u64,
        deadline: u64,
        current_time: u64,
    ) acquires ProtocolConfig, LiquidityPool, UserAccount {
        let user_addr = signer::address_of(user);
        
        // Input validation
        validate_amount(amount_a);
        validate_amount(amount_b);
        validate_deadline(current_time, deadline);
        
        // State checks
        let config = borrow_global<ProtocolConfig>(config_addr);
        validate_not_paused(config);
        
        assert!(exists<LiquidityPool>(pool_addr), error::not_found(EPOOL_NOT_INITIALIZED));
        let pool = borrow_global_mut<LiquidityPool>(pool_addr);
        
        assert!(exists<UserAccount>(user_addr), error::not_found(EACCOUNT_NOT_REGISTERED));
        let user_account = borrow_global_mut<UserAccount>(user_addr);
        
        // Balance check
        assert!(user_account.balance_a >= amount_a, error::invalid_state(EINSUFFICIENT_BALANCE));
        assert!(user_account.balance_b >= amount_b, error::invalid_state(EINSUFFICIENT_BALANCE));
        
        // Calculate LP tokens
        let lp_minted = if (pool.lp_total_supply == 0) {
            // Initial liquidity: geometric mean
            // Simplified: sqrt(amount_a * amount_b)
            amount_a  // simplified
        } else {
            // Proportional to existing pool
            let lp_from_a = safe_mul(amount_a, pool.lp_total_supply) / pool.reserve_a;
            let lp_from_b = safe_mul(amount_b, pool.lp_total_supply) / pool.reserve_b;
            if (lp_from_a < lp_from_b) { lp_from_a } else { lp_from_b }
        };
        
        assert!(lp_minted >= min_lp, error::invalid_state(ESLIPPAGE_EXCEEDED));
        
        // Execute
        user_account.balance_a = safe_sub(user_account.balance_a, amount_a);
        user_account.balance_b = safe_sub(user_account.balance_b, amount_b);
        user_account.lp_balance = safe_add(user_account.lp_balance, lp_minted);
        
        pool.reserve_a = safe_add(pool.reserve_a, amount_a);
        pool.reserve_b = safe_add(pool.reserve_b, amount_b);
        pool.lp_total_supply = safe_add(pool.lp_total_supply, lp_minted);
    }
    
    // ============================================
    // Admin Functions with proper error handling
    // ============================================
    
    public entry fun pause_protocol(
        admin: &signer,
        config_addr: address,
    ) acquires ProtocolConfig {
        let admin_addr = signer::address_of(admin);
        
        assert!(exists<ProtocolConfig>(config_addr), error::not_found(EPOOL_NOT_INITIALIZED));
        let config = borrow_global_mut<ProtocolConfig>(config_addr);
        
        validate_admin(admin_addr, config);
        assert!(!config.is_paused, error::invalid_state(EPROTOCOL_PAUSED));
        
        config.is_paused = true;
    }
    
    public entry fun update_fee(
        admin: &signer,
        config_addr: address,
        new_fee_bps: u64,
    ) acquires ProtocolConfig {
        let admin_addr = signer::address_of(admin);
        
        // Fee can't exceed 10% (1000 bps)
        assert!(new_fee_bps <= 1_000, error::invalid_argument(EINVALID_SLIPPAGE));
        
        let config = borrow_global_mut<ProtocolConfig>(config_addr);
        validate_admin(admin_addr, config);
        validate_not_paused(config);
        
        config.fee_bps = new_fee_bps;
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test]
    #[expected_failure(abort_code = 0x10001, location = Self)]
    public fun test_swap_zero_amount() acquires ProtocolConfig, LiquidityPool, UserAccount {
        // This should fail with EZERO_AMOUNT
        // error::invalid_argument(EZERO_AMOUNT) = 0x10001
        validate_amount(0);
    }
    
    #[test]
    public fun test_safe_math() {
        assert!(safe_add(100, 200) == 300, 0);
        assert!(safe_sub(300, 100) == 200, 1);
        assert!(safe_mul(10, 20) == 200, 2);
        assert!(safe_div(100, 4) == 25, 3);
    }
    
    #[test]
    #[expected_failure(abort_code = 0x20003, location = Self)]
    public fun test_division_by_zero() {
        safe_div(100, 0);
    }
}
```

---

## สรุป Error Handling

| Pattern | ใช้เมื่อ |
|---------|---------|
| `assert!(cond, code)` | ส่วนใหญ่ใช้นี้ |
| `abort code` | ใช้ใน if/else chains |
| `error::category(code)` | ต้องการ categorized errors (Aptos) |
| Helper functions | มี validation logic ซ้ำหลายที่ |

---

## แบบฝึกหัด

### Exercise 13.1: Error Catalog
สร้าง error catalog สำหรับ Lending Protocol ที่มี:
- Borrow errors
- Repay errors
- Liquidation errors
- Health factor errors

### Exercise 13.2: Safe Math Library
สร้าง math library ที่มี:
- safe_add, safe_sub, safe_mul, safe_div
- percentage calculation
- basis points conversion

---

**ต่อไป**: [Part 14 - Testing →](part-14-testing.md)
