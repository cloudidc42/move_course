# Part 14: Testing

## สารบัญ
- [Unit Tests](#unit-tests)
- [Test Attributes](#test-attributes)
- [Test Helpers](#test-helpers)
- [Expected Failures](#expected-failures)
- [Test Coverage](#test-coverage)
- [ตัวอย่างโปรแกรม: Complete Test Suite](#ตัวอย่างโปรแกรม-complete-test-suite)

---

## Unit Tests

```move
module learning::basic_tests {
    
    // ============================================
    // Basic test structure
    // ============================================
    
    // #[test] marks a function as a test
    // Test functions must have no arguments (unless using #[test(addr = @0x1)])
    // Test functions must return nothing or ()
    
    #[test]
    public fun test_addition() {
        let result = add(2, 3);
        assert!(result == 5, 0);  // error code 0 if fails
    }
    
    #[test]
    public fun test_subtraction() {
        let result = subtract(10, 3);
        assert!(result == 7, 0);
    }
    
    fun add(a: u64, b: u64): u64 { a + b }
    fun subtract(a: u64, b: u64): u64 { a - b }
    
    // ============================================
    // Multiple assertions in one test
    // ============================================
    
    #[test]
    public fun test_arithmetic() {
        assert!(add(0, 0) == 0, 0);
        assert!(add(100, 200) == 300, 1);
        assert!(add(u64::MAX - 1, 1) == u64::MAX, 2);
        
        assert!(subtract(100, 100) == 0, 3);
        assert!(subtract(u64::MAX, u64::MAX) == 0, 4);
    }
    
    // ============================================
    // Test with complex types
    // ============================================
    
    use std::vector;
    use std::string;
    
    struct User has drop {
        name: vector<u8>,
        age: u64,
        active: bool,
    }
    
    fun create_user(name: vector<u8>, age: u64): User {
        User { name, age, active: true }
    }
    
    #[test]
    public fun test_create_user() {
        let user = create_user(b"Alice", 25);
        assert!(user.name == b"Alice", 0);
        assert!(user.age == 25, 1);
        assert!(user.active, 2);
    }
    
    // ============================================
    // Test isolation
    // ============================================
    
    // Each test runs in isolation
    // State from one test doesn't affect another
    // Global storage is reset between tests
    
    #[test]
    public fun test_first() {
        let x = 42u64;
        assert!(x == 42, 0);
    }
    
    #[test]
    public fun test_second() {
        let x = 100u64;  // fresh state, unaffected by test_first
        assert!(x == 100, 0);
    }
}
```

---

## Test Attributes

```move
module learning::test_attributes {
    use std::signer;
    
    // ============================================
    // #[test] - basic test
    // ============================================
    
    #[test]
    public fun simple_test() {
        assert!(1 + 1 == 2, 0);
    }
    
    // ============================================
    // #[test(account = @address)] - test with signer
    // ============================================
    
    struct Balance has key {
        amount: u64,
    }
    
    #[test(account = @0x1)]
    public fun test_with_signer(account: &signer) acquires Balance {
        let addr = signer::address_of(account);
        
        // Initialize balance
        move_to(account, Balance { amount: 100 });
        
        assert!(exists<Balance>(addr), 0);
        let balance = borrow_global<Balance>(addr);
        assert!(balance.amount == 100, 1);
    }
    
    // ============================================
    // #[test(user1 = @0x1, user2 = @0x2)] - multiple signers
    // ============================================
    
    #[test(user1 = @0x1, user2 = @0x2)]
    public fun test_with_multiple_signers(
        user1: &signer,
        user2: &signer,
    ) acquires Balance {
        let addr1 = signer::address_of(user1);
        let addr2 = signer::address_of(user2);
        
        assert!(addr1 != addr2, 0);
        
        move_to(user1, Balance { amount: 500 });
        move_to(user2, Balance { amount: 300 });
        
        let b1 = borrow_global<Balance>(addr1);
        let b2 = borrow_global<Balance>(addr2);
        
        assert!(b1.amount + b2.amount == 800, 1);
    }
    
    // ============================================
    // #[test_only] - only available in tests
    // ============================================
    
    #[test_only]
    public fun setup_test_balance(account: &signer, amount: u64) {
        move_to(account, Balance { amount });
    }
    
    #[test(account = @0x1)]
    public fun test_using_test_only(account: &signer) acquires Balance {
        setup_test_balance(account, 1000);
        let addr = signer::address_of(account);
        let b = borrow_global<Balance>(addr);
        assert!(b.amount == 1000, 0);
    }
    
    // ============================================
    // #[expected_failure] - test should fail
    // ============================================
    
    const E_ZERO: u64 = 1;
    
    fun must_be_positive(x: u64) {
        assert!(x > 0, E_ZERO);
    }
    
    #[test]
    #[expected_failure]
    public fun test_failure_simple() {
        must_be_positive(0);  // This should abort
    }
    
    #[test]
    #[expected_failure(abort_code = 1)]
    public fun test_failure_with_code() {
        must_be_positive(0);  // Must abort with code 1
    }
    
    #[test]
    #[expected_failure(abort_code = 1, location = Self)]
    public fun test_failure_with_location() {
        must_be_positive(0);  // Must abort with code 1 from this module
    }
}
```

---

## Test Helpers

```move
module learning::test_helpers {
    use std::signer;
    use std::vector;
    
    struct Token has key, drop {
        amount: u64,
    }
    
    struct Config has key, drop {
        admin: address,
        fee_bps: u64,
        is_paused: bool,
    }
    
    // ============================================
    // Test helper functions (marked #[test_only])
    // ============================================
    
    #[test_only]
    public fun setup_config(admin: &signer, fee_bps: u64) {
        let addr = signer::address_of(admin);
        move_to(admin, Config {
            admin: addr,
            fee_bps,
            is_paused: false,
        });
    }
    
    #[test_only]
    public fun setup_token(account: &signer, amount: u64) {
        move_to(account, Token { amount });
    }
    
    #[test_only]
    public fun get_token_amount(addr: address): u64 acquires Token {
        borrow_global<Token>(addr).amount
    }
    
    #[test_only]
    public fun assert_token_amount(addr: address, expected: u64) acquires Token {
        assert!(get_token_amount(addr) == expected, 0);
    }
    
    // ============================================
    // Scenario-based tests
    // ============================================
    
    #[test_only]
    fun run_standard_scenario(
        admin: &signer,
        user: &signer,
    ) acquires Token {
        let admin_addr = signer::address_of(admin);
        let user_addr = signer::address_of(user);
        
        setup_config(admin, 100);  // 1% fee
        setup_token(admin, 10_000);
        setup_token(user, 0);
        
        // Simulate transfer
        let admin_token = borrow_global_mut<Token>(admin_addr);
        let amount = 1000u64;
        admin_token.amount = admin_token.amount - amount;
        
        let user_token = borrow_global_mut<Token>(user_addr);
        user_token.amount = user_token.amount + amount;
    }
    
    #[test(admin = @0x1, user = @0x2)]
    public fun test_standard_scenario(admin: &signer, user: &signer) acquires Token {
        run_standard_scenario(admin, user);
        
        let admin_addr = signer::address_of(admin);
        let user_addr = signer::address_of(user);
        
        assert_token_amount(admin_addr, 9_000);
        assert_token_amount(user_addr, 1_000);
    }
    
    // ============================================
    // Property-based test patterns
    // ============================================
    
    #[test]
    public fun test_invariants() {
        let values = vector[1u64, 2u64, 3u64, 4u64, 5u64];
        
        // Test sum preservation
        let sum_before = vector_sum(&values);
        
        // Reverse doesn't change sum
        let reversed = vector_reverse_copy(&values);
        let sum_after = vector_sum(&reversed);
        
        assert!(sum_before == sum_after, 0);
    }
    
    #[test_only]
    fun vector_sum(v: &vector<u64>): u64 {
        let sum = 0u64;
        let i = 0u64;
        while (i < vector::length(v)) {
            sum = sum + *vector::borrow(v, i);
            i = i + 1;
        };
        sum
    }
    
    #[test_only]
    fun vector_reverse_copy(v: &vector<u64>): vector<u64> {
        let result = vector::empty<u64>();
        let len = vector::length(v);
        let i = len;
        while (i > 0) {
            i = i - 1;
            vector::push_back(&mut result, *vector::borrow(v, i));
        };
        result
    }
}
```

---

## Expected Failures

```move
module learning::expected_failures {
    use std::signer;
    
    const E_UNAUTHORIZED: u64 = 1;
    const E_ZERO_AMOUNT: u64 = 2;
    const E_OVERFLOW: u64 = 3;
    
    fun require_admin(caller: address, admin: address) {
        assert!(caller == admin, E_UNAUTHORIZED);
    }
    
    fun require_positive(amount: u64) {
        assert!(amount > 0, E_ZERO_AMOUNT);
    }
    
    // ============================================
    // Testing error scenarios
    // ============================================
    
    #[test]
    #[expected_failure]
    public fun test_any_failure() {
        require_positive(0);
    }
    
    #[test]
    #[expected_failure(abort_code = 2)]
    public fun test_specific_error_code() {
        require_positive(0);
    }
    
    #[test]
    #[expected_failure(abort_code = 2, location = Self)]
    public fun test_error_location() {
        require_positive(0);
    }
    
    // ============================================
    // Testing with arithmetic errors
    // ============================================
    
    #[test]
    #[expected_failure(arithmetic_error, location = std)]
    public fun test_overflow() {
        let max: u64 = 18446744073709551615u64;
        let _ = max + 1;  // Should overflow
    }
    
    #[test]
    #[expected_failure(arithmetic_error, location = std)]
    public fun test_underflow() {
        let _ = 0u64 - 1u64;  // Should underflow
    }
    
    // ============================================
    // Multiple failure scenarios
    // ============================================
    
    fun divide(a: u64, b: u64): u64 {
        assert!(b != 0, 10);
        a / b
    }
    
    #[test]
    #[expected_failure(abort_code = 10)]
    public fun test_divide_by_zero() {
        divide(100, 0);
    }
    
    #[test]
    public fun test_divide_success() {
        assert!(divide(100, 4) == 25, 0);
        assert!(divide(0, 5) == 0, 1);
        assert!(divide(7, 2) == 3, 2);  // integer division
    }
    
    // ============================================
    // Auth failure tests
    // ============================================
    
    #[test(user = @0x1)]
    #[expected_failure(abort_code = 1)]
    public fun test_unauthorized_access(user: &signer) {
        let user_addr = signer::address_of(user);
        let admin_addr = @0x999;  // different address
        require_admin(user_addr, admin_addr);
    }
    
    #[test(admin = @0x999)]
    public fun test_authorized_access(admin: &signer) {
        let admin_addr = signer::address_of(admin);
        require_admin(admin_addr, admin_addr);  // Should succeed
    }
}
```

---

## Test Coverage

```move
module learning::test_coverage {
    
    // ============================================
    // Edge cases to always test
    // ============================================
    
    public fun bounded_add(a: u64, b: u64, max: u64): u64 {
        let sum = a + b;
        if (sum > max) max else sum
    }
    
    #[test]
    public fun test_bounded_add_coverage() {
        // Normal case
        assert!(bounded_add(10, 20, 100) == 30, 0);
        
        // Boundary: exactly at max
        assert!(bounded_add(50, 50, 100) == 100, 1);
        
        // Boundary: just over max
        assert!(bounded_add(50, 51, 100) == 100, 2);
        
        // Zero cases
        assert!(bounded_add(0, 0, 100) == 0, 3);
        assert!(bounded_add(0, 100, 100) == 100, 4);
        assert!(bounded_add(0, 101, 100) == 100, 5);
        
        // Max with max
        assert!(bounded_add(100, 100, 100) == 100, 6);
    }
    
    // ============================================
    // State machine coverage
    // ============================================
    
    const STATE_CREATED: u8 = 0;
    const STATE_ACTIVE: u8 = 1;
    const STATE_PAUSED: u8 = 2;
    const STATE_CLOSED: u8 = 3;
    
    public fun transition(from: u8, to: u8): bool {
        if (from == STATE_CREATED && to == STATE_ACTIVE) { true }
        else if (from == STATE_ACTIVE && to == STATE_PAUSED) { true }
        else if (from == STATE_PAUSED && to == STATE_ACTIVE) { true }
        else if (from == STATE_ACTIVE && to == STATE_CLOSED) { true }
        else if (from == STATE_PAUSED && to == STATE_CLOSED) { true }
        else { false }
    }
    
    #[test]
    public fun test_valid_transitions() {
        assert!(transition(STATE_CREATED, STATE_ACTIVE), 0);
        assert!(transition(STATE_ACTIVE, STATE_PAUSED), 1);
        assert!(transition(STATE_PAUSED, STATE_ACTIVE), 2);
        assert!(transition(STATE_ACTIVE, STATE_CLOSED), 3);
        assert!(transition(STATE_PAUSED, STATE_CLOSED), 4);
    }
    
    #[test]
    public fun test_invalid_transitions() {
        assert!(!transition(STATE_CREATED, STATE_PAUSED), 0);
        assert!(!transition(STATE_CREATED, STATE_CLOSED), 1);
        assert!(!transition(STATE_CLOSED, STATE_ACTIVE), 2);
        assert!(!transition(STATE_CLOSED, STATE_PAUSED), 3);
        assert!(!transition(STATE_CLOSED, STATE_CREATED), 4);
        assert!(!transition(STATE_ACTIVE, STATE_CREATED), 5);
    }
    
    // ============================================
    // Fuzzing-style tests
    // ============================================
    
    public fun clamp(value: u64, min: u64, max: u64): u64 {
        if (value < min) min
        else if (value > max) max
        else value
    }
    
    #[test]
    public fun test_clamp_properties() {
        // Property: result always in [min, max]
        let test_cases = vector[
            (0u64, 10u64, 20u64),
            (15u64, 10u64, 20u64),
            (25u64, 10u64, 20u64),
            (10u64, 10u64, 20u64),
            (20u64, 10u64, 20u64),
        ];
        
        let i = 0u64;
        use std::vector;
        while (i < vector::length(&test_cases)) {
            let (val, min, max) = *vector::borrow(&test_cases, i);
            let result = clamp(val, min, max);
            assert!(result >= min, (i * 2) as u64);
            assert!(result <= max, (i * 2 + 1) as u64);
            i = i + 1;
        };
    }
}
```

---

## ตัวอย่างโปรแกรม: Complete Test Suite

```move
module learning::token_test_suite {
    use std::signer;
    use std::vector;
    
    // ============================================
    // Production Code
    // ============================================
    
    const E_ZERO_AMOUNT: u64 = 1;
    const E_INSUFFICIENT_BALANCE: u64 = 2;
    const E_NOT_AUTHORIZED: u64 = 3;
    const E_PAUSED: u64 = 4;
    const E_ACCOUNT_EXISTS: u64 = 5;
    
    struct Token has key {
        balance: u64,
    }
    
    struct MintCap has key {
        max_per_tx: u64,
    }
    
    struct Config has key {
        admin: address,
        total_supply: u64,
        is_paused: bool,
    }
    
    public fun initialize(admin: &signer, max_mint: u64) {
        let addr = signer::address_of(admin);
        move_to(admin, Config {
            admin: addr,
            total_supply: 0,
            is_paused: false,
        });
        move_to(admin, MintCap { max_per_tx: max_mint });
    }
    
    public fun register(account: &signer) {
        let addr = signer::address_of(account);
        assert!(!exists<Token>(addr), E_ACCOUNT_EXISTS);
        move_to(account, Token { balance: 0 });
    }
    
    public fun mint(
        minter: &signer,
        config_addr: address,
        to: address,
        amount: u64,
    ) acquires Config, MintCap, Token {
        assert!(amount > 0, E_ZERO_AMOUNT);
        
        let minter_addr = signer::address_of(minter);
        let cap = borrow_global<MintCap>(minter_addr);
        assert!(amount <= cap.max_per_tx, E_NOT_AUTHORIZED);
        
        let config = borrow_global_mut<Config>(config_addr);
        assert!(!config.is_paused, E_PAUSED);
        
        assert!(exists<Token>(to), E_NOT_AUTHORIZED);
        
        let token = borrow_global_mut<Token>(to);
        token.balance = token.balance + amount;
        config.total_supply = config.total_supply + amount;
    }
    
    public fun transfer(
        from: &signer,
        to: address,
        config_addr: address,
        amount: u64,
    ) acquires Config, Token {
        assert!(amount > 0, E_ZERO_AMOUNT);
        
        let config = borrow_global<Config>(config_addr);
        assert!(!config.is_paused, E_PAUSED);
        
        let from_addr = signer::address_of(from);
        assert!(exists<Token>(to), E_NOT_AUTHORIZED);
        
        let from_token = borrow_global_mut<Token>(from_addr);
        assert!(from_token.balance >= amount, E_INSUFFICIENT_BALANCE);
        from_token.balance = from_token.balance - amount;
        
        let to_token = borrow_global_mut<Token>(to);
        to_token.balance = to_token.balance + amount;
    }
    
    public fun pause(admin: &signer, config_addr: address) acquires Config {
        let config = borrow_global_mut<Config>(config_addr);
        assert!(signer::address_of(admin) == config.admin, E_NOT_AUTHORIZED);
        config.is_paused = true;
    }
    
    public fun balance_of(addr: address): u64 acquires Token {
        if (!exists<Token>(addr)) { return 0 };
        borrow_global<Token>(addr).balance
    }
    
    // ============================================
    // Test Helpers
    // ============================================
    
    #[test_only]
    fun setup(admin: &signer, user1: &signer, user2: &signer) acquires Config, MintCap, Token {
        let admin_addr = signer::address_of(admin);
        
        initialize(admin, 10_000);
        register(admin);
        register(user1);
        register(user2);
        
        // Mint initial supply to admin
        mint(admin, admin_addr, admin_addr, 10_000);
    }
    
    // ============================================
    // Happy Path Tests
    // ============================================
    
    #[test(admin = @0x1, user1 = @0x2, user2 = @0x3)]
    public fun test_initialize_and_mint(
        admin: &signer,
        user1: &signer,
        user2: &signer,
    ) acquires Config, MintCap, Token {
        let admin_addr = signer::address_of(admin);
        let user1_addr = signer::address_of(user1);
        
        setup(admin, user1, user2);
        
        assert!(balance_of(admin_addr) == 10_000, 0);
        
        // Mint to user1
        mint(admin, admin_addr, user1_addr, 1_000);
        assert!(balance_of(user1_addr) == 1_000, 1);
    }
    
    #[test(admin = @0x1, user1 = @0x2, user2 = @0x3)]
    public fun test_transfer(
        admin: &signer,
        user1: &signer,
        user2: &signer,
    ) acquires Config, MintCap, Token {
        let admin_addr = signer::address_of(admin);
        let user1_addr = signer::address_of(user1);
        let user2_addr = signer::address_of(user2);
        
        setup(admin, user1, user2);
        
        // Transfer 2000 from admin to user1
        transfer(admin, user1_addr, admin_addr, 2_000);
        
        assert!(balance_of(admin_addr) == 8_000, 0);
        assert!(balance_of(user1_addr) == 2_000, 1);
        
        // user1 transfers 500 to user2
        transfer(user1, user2_addr, admin_addr, 500);
        
        assert!(balance_of(user1_addr) == 1_500, 2);
        assert!(balance_of(user2_addr) == 500, 3);
        
        // Total supply unchanged
        assert!(balance_of(admin_addr) + balance_of(user1_addr) + balance_of(user2_addr) == 10_000, 4);
    }
    
    // ============================================
    // Error Path Tests
    // ============================================
    
    #[test(admin = @0x1, user1 = @0x2, user2 = @0x3)]
    #[expected_failure(abort_code = 1)]
    public fun test_zero_mint(
        admin: &signer,
        user1: &signer,
        user2: &signer,
    ) acquires Config, MintCap, Token {
        let admin_addr = signer::address_of(admin);
        let user1_addr = signer::address_of(user1);
        
        setup(admin, user1, user2);
        mint(admin, admin_addr, user1_addr, 0);  // Should fail: E_ZERO_AMOUNT
    }
    
    #[test(admin = @0x1, user1 = @0x2, user2 = @0x3)]
    #[expected_failure(abort_code = 2)]
    public fun test_insufficient_balance(
        admin: &signer,
        user1: &signer,
        user2: &signer,
    ) acquires Config, MintCap, Token {
        let admin_addr = signer::address_of(admin);
        let user1_addr = signer::address_of(user1);
        
        setup(admin, user1, user2);
        // user1 has 0 balance, try to transfer 100
        transfer(user1, admin_addr, admin_addr, 100);  // Should fail: E_INSUFFICIENT_BALANCE
    }
    
    #[test(admin = @0x1, user1 = @0x2, user2 = @0x3)]
    #[expected_failure(abort_code = 4)]
    public fun test_paused_transfer(
        admin: &signer,
        user1: &signer,
        user2: &signer,
    ) acquires Config, MintCap, Token {
        let admin_addr = signer::address_of(admin);
        let user1_addr = signer::address_of(user1);
        
        setup(admin, user1, user2);
        pause(admin, admin_addr);
        
        // Transfer when paused should fail
        transfer(admin, user1_addr, admin_addr, 100);  // Should fail: E_PAUSED
    }
    
    // ============================================
    // Edge Cases
    // ============================================
    
    #[test(admin = @0x1, user1 = @0x2, user2 = @0x3)]
    public fun test_transfer_all(
        admin: &signer,
        user1: &signer,
        user2: &signer,
    ) acquires Config, MintCap, Token {
        let admin_addr = signer::address_of(admin);
        let user1_addr = signer::address_of(user1);
        
        setup(admin, user1, user2);
        
        // Transfer entire balance
        transfer(admin, user1_addr, admin_addr, 10_000);
        
        assert!(balance_of(admin_addr) == 0, 0);
        assert!(balance_of(user1_addr) == 10_000, 1);
    }
}
```

---

## สรุป Testing

| Attribute | Usage |
|-----------|-------|
| `#[test]` | Basic test function |
| `#[test(addr = @0x1)]` | Test with signer parameter |
| `#[test_only]` | Only compiled for tests |
| `#[expected_failure]` | Test should abort |
| `#[expected_failure(abort_code = N)]` | Test should abort with code N |

---

## แบบฝึกหัด

### Exercise 14.1: Test a Vault
สร้าง test suite สำหรับ Vault contract ที่ test:
- Deposit
- Withdraw
- Emergency withdraw (admin only)
- All error cases

### Exercise 14.2: Property Testing
สร้าง tests ที่ verify properties เช่น:
- Conservation: total supply never changes
- Non-negative: balances always >= 0

---

**ต่อไป**: [Part 15 - Standard Library →](part-15-standard-library.md)
