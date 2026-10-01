# Part 20: Beginner Summary & Capstone Project

## สารบัญ
- [สรุป Beginner Level](#สรุป-beginner-level)
- [Cheat Sheet](#cheat-sheet)
- [Capstone Project: SimpleBank](#capstone-project-simplebank)
- [สิ่งที่เรียนรู้แล้ว](#สิ่งที่เรียนรู้แล้ว)
- [ต่อไป: Intermediate Level](#ต่อไป-intermediate-level)

---

## สรุป Beginner Level

### Part 1-5: Foundation
| Part | Topic | Key Learning |
|------|-------|-------------|
| 01 | Introduction | Move history, vs Solidity |
| 02 | Setup | CLI, tooling, first project |
| 03 | Syntax & Types | u8-u256, bool, address, vector |
| 04 | Variables | let, let mut, destructuring |
| 05 | Functions | fun, entry, visibility |

### Part 6-10: Core Concepts
| Part | Topic | Key Learning |
|------|-------|-------------|
| 06 | Control Flow | if, loop, while, abort |
| 07 | Modules | module, use, visibility, friends |
| 08 | Resources | Linear types, global storage |
| 09 | References | &T, &mut T, acquires |
| 10 | Structs | abilities, nested, generics |

### Part 11-15: Language Features
| Part | Topic | Key Learning |
|------|-------|-------------|
| 11 | Abilities | copy, drop, store, key |
| 12 | Generics | T: constraints, phantom types |
| 13 | Error Handling | assert!, abort, error codes |
| 14 | Testing | #[test], expected_failure |
| 15 | Standard Library | vector, string, option, table |

### Part 16-19: Advanced Beginners
| Part | Topic | Key Learning |
|------|-------|-------------|
| 16 | Events | EventHandle, #[event], emit |
| 17 | Vectors Deep Dive | algorithms, patterns |
| 18 | Strings | UTF-8, formatting, JSON |
| 19 | Math | fixed-point, DeFi formulas |

---

## Cheat Sheet

### Types
```move
// Primitives
let x: u8 = 255;
let y: u64 = 1_000_000;
let z: u128 = 1_000_000_000_000_000_000u128;
let b: bool = true;
let addr: address = @0x1;

// Collections
let v: vector<u64> = vector[1, 2, 3];
let s: String = string::utf8(b"hello");

// Struct
struct MyStruct has copy, drop, store, key {
    field: u64,
}
```

### Functions
```move
// Private
fun helper() {}

// Public
public fun api_function() {}

// Entry (callable from transactions)
public entry fun transaction_function(user: &signer) {}

// View (read-only query)
#[view]
public fun query_function(): u64 { 42 }

// Generic
public fun generic<T: copy + drop>(value: T): T { value }
```

### Resources
```move
// Store resource at address
move_to(signer, MyResource { value: 0 });

// Read resource
let r = borrow_global<MyResource>(addr);

// Modify resource
let r_mut = borrow_global_mut<MyResource>(addr);
r_mut.value = 100;

// Remove resource
let MyResource { value: _ } = move_from<MyResource>(addr);

// Check existence
let exists = exists<MyResource>(addr);
```

### Error Handling
```move
const E_ZERO: u64 = 1;
const E_UNAUTH: u64 = 2;

// Assert pattern
assert!(condition, error_code);

// Abort pattern
if (!condition) { abort error_code };

// aptos_std::error
assert!(condition, error::invalid_argument(CODE));
```

### Testing
```move
#[test]
public fun test_basic() {
    assert!(1 + 1 == 2, 0);
}

#[test(user = @0x1)]
public fun test_with_signer(user: &signer) acquires Resource {
    // test with signer
}

#[test]
#[expected_failure(abort_code = 1)]
public fun test_failure() {
    abort 1
}

#[test_only]
public fun setup_helper() {
    // only in tests
}
```

---

## Capstone Project: SimpleBank

สร้าง Bank protocol ที่สมบูรณ์โดยใช้ทุกสิ่งที่เรียนรู้มา:

```move
module learning::simple_bank {
    use std::signer;
    use std::string::{Self, String};
    use std::vector;
    use aptos_framework::event;
    use aptos_framework::timestamp;
    use aptos_std::table::{Self, Table};
    
    // ============================================
    // Types (covering all ability combinations)
    // ============================================
    
    // Account record - key ability (global storage)
    struct Account has key {
        owner: address,
        balance: u64,
        locked_balance: u64,
        total_deposited: u64,
        total_withdrawn: u64,
        created_at: u64,
        last_action: u64,
        is_active: bool,
    }
    
    // Transaction record - store ability (stored in Table)
    struct Transaction has copy, drop, store {
        id: u64,
        tx_type: u8,     // 1=deposit, 2=withdraw, 3=transfer
        amount: u64,
        from: address,
        to: address,
        timestamp: u64,
        memo: String,
    }
    
    // Transaction history - key ability (global storage per account)
    struct TransactionHistory has key {
        txs: Table<u64, Transaction>,
        count: u64,
    }
    
    // Bank config - key ability (singleton)
    struct BankConfig has key {
        admin: address,
        daily_withdrawal_limit: u64,
        transfer_fee_bps: u64,
        min_balance: u64,
        is_paused: bool,
        total_accounts: u64,
        total_deposits: u64,
        total_withdrawals: u64,
    }
    
    // Admin Capability - no copy (singleton permission)
    struct AdminCap has key {
        can_pause: bool,
        can_set_limits: bool,
        can_view_all: bool,
    }
    
    // ============================================
    // Events
    // ============================================
    
    #[event]
    struct DepositEvent has drop, store {
        account: address,
        amount: u64,
        new_balance: u64,
        timestamp: u64,
    }
    
    #[event]
    struct WithdrawEvent has drop, store {
        account: address,
        amount: u64,
        new_balance: u64,
        fee: u64,
        timestamp: u64,
    }
    
    #[event]
    struct TransferEvent has drop, store {
        from: address,
        to: address,
        amount: u64,
        fee: u64,
        timestamp: u64,
    }
    
    #[event]
    struct AccountCreatedEvent has drop, store {
        owner: address,
        timestamp: u64,
    }
    
    // ============================================
    // Error Codes
    // ============================================
    
    // State errors (1-19)
    const E_BANK_PAUSED: u64 = 1;
    const E_BANK_NOT_INITIALIZED: u64 = 2;
    const E_ACCOUNT_EXISTS: u64 = 3;
    const E_ACCOUNT_NOT_FOUND: u64 = 4;
    const E_ACCOUNT_INACTIVE: u64 = 5;
    
    // Input errors (20-39)
    const E_ZERO_AMOUNT: u64 = 20;
    const E_INVALID_RECIPIENT: u64 = 21;
    const E_SELF_TRANSFER: u64 = 22;
    const E_MEMO_TOO_LONG: u64 = 23;
    
    // Balance errors (40-59)
    const E_INSUFFICIENT_BALANCE: u64 = 40;
    const E_BELOW_MIN_BALANCE: u64 = 41;
    const E_DAILY_LIMIT_EXCEEDED: u64 = 42;
    
    // Auth errors (60-79)
    const E_NOT_AUTHORIZED: u64 = 60;
    const E_NOT_ACCOUNT_OWNER: u64 = 61;
    
    // ============================================
    // Constants
    // ============================================
    
    const TX_DEPOSIT: u8 = 1;
    const TX_WITHDRAW: u8 = 2;
    const TX_TRANSFER: u8 = 3;
    
    const MAX_MEMO_LEN: u64 = 256;
    
    // ============================================
    // Initialization
    // ============================================
    
    public entry fun initialize_bank(
        admin: &signer,
        daily_limit: u64,
        transfer_fee_bps: u64,
        min_balance: u64,
    ) {
        let addr = signer::address_of(admin);
        assert!(!exists<BankConfig>(addr), E_BANK_NOT_INITIALIZED);
        
        move_to(admin, BankConfig {
            admin: addr,
            daily_withdrawal_limit: daily_limit,
            transfer_fee_bps,
            min_balance,
            is_paused: false,
            total_accounts: 0,
            total_deposits: 0,
            total_withdrawals: 0,
        });
        
        move_to(admin, AdminCap {
            can_pause: true,
            can_set_limits: true,
            can_view_all: true,
        });
    }
    
    // ============================================
    // Account Management
    // ============================================
    
    public entry fun open_account(user: &signer, bank_addr: address) acquires BankConfig {
        let user_addr = signer::address_of(user);
        assert!(!exists<Account>(user_addr), E_ACCOUNT_EXISTS);
        
        let config = borrow_global_mut<BankConfig>(bank_addr);
        assert!(!config.is_paused, E_BANK_PAUSED);
        
        let now = timestamp::now_seconds();
        
        move_to(user, Account {
            owner: user_addr,
            balance: 0,
            locked_balance: 0,
            total_deposited: 0,
            total_withdrawn: 0,
            created_at: now,
            last_action: now,
            is_active: true,
        });
        
        move_to(user, TransactionHistory {
            txs: table::new(),
            count: 0,
        });
        
        config.total_accounts = config.total_accounts + 1;
        
        event::emit(AccountCreatedEvent {
            owner: user_addr,
            timestamp: now,
        });
    }
    
    public entry fun close_account(user: &signer, bank_addr: address) acquires Account, BankConfig, TransactionHistory {
        let user_addr = signer::address_of(user);
        assert!(exists<Account>(user_addr), E_ACCOUNT_NOT_FOUND);
        
        let account = borrow_global<Account>(user_addr);
        assert!(account.balance == 0, E_INSUFFICIENT_BALANCE);
        
        let config = borrow_global_mut<BankConfig>(bank_addr);
        
        let Account {
            owner: _, balance: _, locked_balance: _,
            total_deposited: _, total_withdrawn: _,
            created_at: _, last_action: _, is_active: _
        } = move_from<Account>(user_addr);
        
        // Clean up transaction history
        // Note: in production, would archive first
        let TransactionHistory { txs, count: _ } = move_from<TransactionHistory>(user_addr);
        table::destroy_empty(txs);  // only works if table is empty - simplified
        
        config.total_accounts = config.total_accounts - 1;
    }
    
    // ============================================
    // Core Banking Operations
    // ============================================
    
    public entry fun deposit(
        user: &signer,
        bank_addr: address,
        amount: u64,
        memo: vector<u8>,
    ) acquires Account, BankConfig, TransactionHistory {
        assert!(amount > 0, E_ZERO_AMOUNT);
        assert!(vector::length(&memo) <= MAX_MEMO_LEN, E_MEMO_TOO_LONG);
        
        let user_addr = signer::address_of(user);
        
        // Check bank state
        let config = borrow_global_mut<BankConfig>(bank_addr);
        assert!(!config.is_paused, E_BANK_PAUSED);
        
        // Check account
        assert!(exists<Account>(user_addr), E_ACCOUNT_NOT_FOUND);
        let account = borrow_global_mut<Account>(user_addr);
        assert!(account.is_active, E_ACCOUNT_INACTIVE);
        
        let now = timestamp::now_seconds();
        
        // Update balance
        account.balance = account.balance + amount;
        account.total_deposited = account.total_deposited + amount;
        account.last_action = now;
        
        // Update bank stats
        config.total_deposits = config.total_deposits + amount;
        
        // Record transaction
        record_transaction(
            user_addr, user_addr, amount, TX_DEPOSIT, memo, now
        );
        
        event::emit(DepositEvent {
            account: user_addr,
            amount,
            new_balance: account.balance,
            timestamp: now,
        });
    }
    
    public entry fun withdraw(
        user: &signer,
        bank_addr: address,
        amount: u64,
        memo: vector<u8>,
    ) acquires Account, BankConfig, TransactionHistory {
        assert!(amount > 0, E_ZERO_AMOUNT);
        
        let user_addr = signer::address_of(user);
        let config = borrow_global_mut<BankConfig>(bank_addr);
        assert!(!config.is_paused, E_BANK_PAUSED);
        assert!(exists<Account>(user_addr), E_ACCOUNT_NOT_FOUND);
        
        let account = borrow_global_mut<Account>(user_addr);
        assert!(account.is_active, E_ACCOUNT_INACTIVE);
        
        // Check daily limit
        assert!(amount <= config.daily_withdrawal_limit, E_DAILY_LIMIT_EXCEEDED);
        
        // Check sufficient balance (maintaining min_balance)
        let needed = amount + config.min_balance;
        assert!(account.balance >= needed, E_INSUFFICIENT_BALANCE);
        
        let now = timestamp::now_seconds();
        
        account.balance = account.balance - amount;
        account.total_withdrawn = account.total_withdrawn + amount;
        account.last_action = now;
        
        config.total_withdrawals = config.total_withdrawals + amount;
        
        record_transaction(user_addr, user_addr, amount, TX_WITHDRAW, memo, now);
        
        event::emit(WithdrawEvent {
            account: user_addr,
            amount,
            new_balance: account.balance,
            fee: 0,
            timestamp: now,
        });
    }
    
    public entry fun transfer(
        from_user: &signer,
        to_addr: address,
        bank_addr: address,
        amount: u64,
        memo: vector<u8>,
    ) acquires Account, BankConfig, TransactionHistory {
        assert!(amount > 0, E_ZERO_AMOUNT);
        
        let from_addr = signer::address_of(from_user);
        assert!(from_addr != to_addr, E_SELF_TRANSFER);
        assert!(exists<Account>(to_addr), E_INVALID_RECIPIENT);
        
        let config = borrow_global<BankConfig>(bank_addr);
        assert!(!config.is_paused, E_BANK_PAUSED);
        
        // Calculate fee
        let fee = amount * config.transfer_fee_bps / 10_000;
        let total_debit = amount + fee;
        
        let now = timestamp::now_seconds();
        
        // Debit from sender
        let from_account = borrow_global_mut<Account>(from_addr);
        assert!(from_account.is_active, E_ACCOUNT_INACTIVE);
        assert!(from_account.balance >= total_debit + config.min_balance, E_INSUFFICIENT_BALANCE);
        
        from_account.balance = from_account.balance - total_debit;
        from_account.total_withdrawn = from_account.total_withdrawn + total_debit;
        from_account.last_action = now;
        
        // Credit to recipient
        let to_account = borrow_global_mut<Account>(to_addr);
        assert!(to_account.is_active, E_ACCOUNT_INACTIVE);
        to_account.balance = to_account.balance + amount;
        to_account.total_deposited = to_account.total_deposited + amount;
        to_account.last_action = now;
        
        // Record transactions for both parties
        record_transaction(from_addr, to_addr, amount, TX_TRANSFER, memo, now);
        
        event::emit(TransferEvent {
            from: from_addr,
            to: to_addr,
            amount,
            fee,
            timestamp: now,
        });
    }
    
    // ============================================
    // Admin Functions
    // ============================================
    
    public entry fun set_pause_state(
        admin: &signer,
        bank_addr: address,
        paused: bool,
    ) acquires BankConfig {
        let addr = signer::address_of(admin);
        let config = borrow_global_mut<BankConfig>(bank_addr);
        assert!(addr == config.admin, E_NOT_AUTHORIZED);
        config.is_paused = paused;
    }
    
    public entry fun update_limits(
        admin: &signer,
        bank_addr: address,
        daily_limit: u64,
        fee_bps: u64,
        min_balance: u64,
    ) acquires BankConfig {
        let addr = signer::address_of(admin);
        let config = borrow_global_mut<BankConfig>(bank_addr);
        assert!(addr == config.admin, E_NOT_AUTHORIZED);
        
        config.daily_withdrawal_limit = daily_limit;
        config.transfer_fee_bps = fee_bps;
        config.min_balance = min_balance;
    }
    
    public entry fun freeze_account(
        admin: &signer,
        bank_addr: address,
        account_addr: address,
    ) acquires BankConfig, Account {
        let addr = signer::address_of(admin);
        let config = borrow_global<BankConfig>(bank_addr);
        assert!(addr == config.admin, E_NOT_AUTHORIZED);
        
        let account = borrow_global_mut<Account>(account_addr);
        account.is_active = false;
    }
    
    // ============================================
    // View Functions
    // ============================================
    
    #[view]
    public fun get_balance(account_addr: address): u64 acquires Account {
        if (!exists<Account>(account_addr)) { return 0 };
        borrow_global<Account>(account_addr).balance
    }
    
    #[view]
    public fun get_account_info(account_addr: address): (u64, u64, u64, bool) acquires Account {
        assert!(exists<Account>(account_addr), E_ACCOUNT_NOT_FOUND);
        let account = borrow_global<Account>(account_addr);
        (
            account.balance,
            account.total_deposited,
            account.total_withdrawn,
            account.is_active,
        )
    }
    
    #[view]
    public fun get_bank_stats(bank_addr: address): (u64, u64, u64) acquires BankConfig {
        let config = borrow_global<BankConfig>(bank_addr);
        (config.total_accounts, config.total_deposits, config.total_withdrawals)
    }
    
    #[view]
    public fun transaction_count(account_addr: address): u64 acquires TransactionHistory {
        if (!exists<TransactionHistory>(account_addr)) { return 0 };
        borrow_global<TransactionHistory>(account_addr).count
    }
    
    // ============================================
    // Internal Helpers
    // ============================================
    
    fun record_transaction(
        from: address,
        to: address,
        amount: u64,
        tx_type: u8,
        memo: vector<u8>,
        timestamp: u64,
    ) acquires TransactionHistory {
        if (!exists<TransactionHistory>(from)) return;
        
        let history = borrow_global_mut<TransactionHistory>(from);
        let tx_id = history.count;
        
        table::add(&mut history.txs, tx_id, Transaction {
            id: tx_id,
            tx_type,
            amount,
            from,
            to,
            timestamp,
            memo: string::utf8(memo),
        });
        
        history.count = history.count + 1;
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test(admin = @0x1, alice = @0x2, bob = @0x3)]
    public fun test_full_bank_flow(
        admin: &signer,
        alice: &signer,
        bob: &signer,
    ) acquires Account, BankConfig, TransactionHistory {
        let admin_addr = signer::address_of(admin);
        let alice_addr = signer::address_of(alice);
        let bob_addr = signer::address_of(bob);
        
        // Setup: initialize bank
        initialize_bank(admin, 10_000, 100, 100);
        
        // Open accounts
        open_account(alice, admin_addr);
        open_account(bob, admin_addr);
        
        // Alice deposits 5000
        deposit(alice, admin_addr, 5_000, b"Initial deposit");
        assert!(get_balance(alice_addr) == 5_000, 0);
        
        // Bob deposits 2000
        deposit(bob, admin_addr, 2_000, b"Bob's deposit");
        assert!(get_balance(bob_addr) == 2_000, 1);
        
        // Alice transfers 1000 to Bob (1% fee = 10)
        transfer(alice, bob_addr, admin_addr, 1_000, b"Payment to Bob");
        
        // Alice: 5000 - 1000 - 10 fee = 3990
        assert!(get_balance(alice_addr) == 3_990, 2);
        // Bob: 2000 + 1000 = 3000
        assert!(get_balance(bob_addr) == 3_000, 3);
        
        // Alice withdraws 1000
        withdraw(alice, admin_addr, 1_000, b"Withdrawal");
        // Alice: 3990 - 1000 = 2990
        assert!(get_balance(alice_addr) == 2_990, 4);
        
        // Check bank stats
        let (accounts, deposits, withdrawals) = get_bank_stats(admin_addr);
        assert!(accounts == 2, 5);
        assert!(deposits == 7_000, 6);  // 5000 + 2000
        assert!(withdrawals == 1_000, 7);
        
        // Transaction counts
        assert!(transaction_count(alice_addr) == 3, 8);  // deposit, transfer, withdraw
        assert!(transaction_count(bob_addr) == 1, 9);     // deposit only (transfer comes from alice)
        
        // Test account info
        let (balance, deposited, withdrawn, active) = get_account_info(alice_addr);
        assert!(balance == 2_990, 10);
        assert!(deposited == 5_000, 11);
        assert!(withdrawn == 2_010, 12);  // 1000 transfer + 10 fee + 1000 withdraw
        assert!(active, 13);
    }
    
    #[test(admin = @0x1, user = @0x2)]
    #[expected_failure(abort_code = 20)]
    public fun test_zero_deposit_fails(admin: &signer, user: &signer) acquires Account, BankConfig, TransactionHistory {
        let admin_addr = signer::address_of(admin);
        initialize_bank(admin, 10_000, 100, 0);
        open_account(user, admin_addr);
        deposit(user, admin_addr, 0, b"");  // Should fail: E_ZERO_AMOUNT
    }
    
    #[test(admin = @0x1, user = @0x2)]
    #[expected_failure(abort_code = 40)]
    public fun test_overdraft_fails(admin: &signer, user: &signer) acquires Account, BankConfig, TransactionHistory {
        let admin_addr = signer::address_of(admin);
        initialize_bank(admin, 10_000, 100, 0);
        open_account(user, admin_addr);
        deposit(user, admin_addr, 100, b"");
        withdraw(user, admin_addr, 200, b"");  // Should fail: E_INSUFFICIENT_BALANCE
    }
    
    #[test(admin = @0x1, user = @0x2)]
    #[expected_failure(abort_code = 1)]
    public fun test_paused_deposit_fails(admin: &signer, user: &signer) acquires Account, BankConfig, TransactionHistory {
        let admin_addr = signer::address_of(admin);
        initialize_bank(admin, 10_000, 100, 0);
        open_account(user, admin_addr);
        set_pause_state(admin, admin_addr, true);
        deposit(user, admin_addr, 100, b"");  // Should fail: E_BANK_PAUSED
    }
}
```

---

## สิ่งที่เรียนรู้แล้ว

ใน Beginner Level นี้ครอบคลุม:

1. **Move Fundamentals**: Types, variables, functions, modules
2. **Safety Mechanisms**: Linear types, abilities, borrow checker
3. **Resource Management**: Global storage, move_to/from, borrow
4. **Error Handling**: assert!, abort, error codes
5. **Testing**: Unit tests, expected failures
6. **Standard Library**: vector, string, option, table
7. **Events**: emit patterns for blockchain transparency
8. **Math**: Fixed-point arithmetic สำหรับ DeFi

---

## ต่อไป: Intermediate Level

**Part 21-50** จะครอบคลุม:

- **Aptos Framework**: Coin standard, Account model, Object model
- **Sui Framework**: Object-centric design, sui::transfer
- **DeFi Building Blocks**: AMM, Lending, Staking protocols
- **Token Standards**: Fungible assets, NFTs
- **Advanced Patterns**: Capability pattern, proxy pattern
- **Cross-module Interactions**: Complex protocol architecture

---

## แบบฝึกหัดสุดท้าย

### Final Project: Multi-Token Wallet
สร้าง wallet ที่:
1. รองรับ multiple token types (generics)
2. มี allowance system (delegated spending)
3. มี spending limits
4. Emit events ทุก operation
5. มี test suite ครบถ้วน

---

**ต่อไป**: [Part 21 - Aptos Framework →](../intermediate/part-21-aptos-framework.md)
