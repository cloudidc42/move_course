# Part 39: Move Prover & Formal Verification

## สารบัญ
- [Move Prover คืออะไร](#move-prover-คืออะไร)
- [Specification Language](#specification-language)
- [Basic Specifications](#basic-specifications)
- [Invariants](#invariants)
- [Aborts Specifications](#aborts-specifications)
- [ตัวอย่าง: Verified Token](#ตัวอย่าง-verified-token)

---

## Move Prover คืออะไร

```
Move Prover = เครื่องมือ formal verification สำหรับ Move
  - พิสูจน์ว่า code ทำงานตามที่ spec กำหนด
  - ค้นหา bugs ที่ tests อาจไม่พบ
  - ทำงานผ่าน Z3 SMT solver

ประโยชน์:
  ✓ พิสูจน์ absence of bugs (not just find bugs)
  ✓ Automatically check all inputs
  ✓ Find edge cases humans miss
  ✓ Required by some protocols for mainnet deploy

Run:
  aptos move prove --package-dir . 
  # or: move prove --named-addresses my_module=default

Spec language keywords:
  spec module { ... }   - module-level specs
  spec fun foo { ... }  - function specs
  requires              - preconditions
  ensures               - postconditions
  aborts_if             - when function should abort
  modifies              - what storage is modified
  invariant             - always-true conditions
  pragma                - prover settings
```

---

## Specification Language

```move
module verified::spec_basics {
    // ============================================
    // Basic specs: ensures and requires
    // ============================================
    
    public fun add(x: u64, y: u64): u64 {
        x + y
    }
    
    // Spec for add function
    spec add {
        // Precondition: no overflow
        requires x + y <= MAX_U64;
        // Postcondition: result equals sum
        ensures result == x + y;
    }
    
    // ============================================
    // Pure math verification
    // ============================================
    
    public fun multiply(x: u64, y: u64): u64 {
        x * y
    }
    
    spec multiply {
        requires (x as u128) * (y as u128) <= (MAX_U64 as u128);
        ensures (result as u128) == (x as u128) * (y as u128);
        ensures result >= x || x == 0;
        ensures result >= y || y == 0;
    }
    
    // ============================================
    // Conditional ensures
    // ============================================
    
    public fun safe_div(numerator: u64, denominator: u64): u64 {
        if (denominator == 0) {
            0
        } else {
            numerator / denominator
        }
    }
    
    spec safe_div {
        ensures denominator == 0 ==> result == 0;
        ensures denominator != 0 ==> result == numerator / denominator;
        // This function never aborts (handles div by zero)
        aborts_if false;
    }
    
    // ============================================
    // Ghost variables (for spec only)
    // ============================================
    
    // Ghost variable: exists only in spec, not in code
    // Used for tracking state across function calls
    
    spec module {
        global total_supply_ghost: num;
    }
    
    // ============================================
    // Quantifiers
    // ============================================
    
    spec fun all_positive(v: vector<u64>): bool {
        // forall: universal quantifier
        forall i in 0..len(v): v[i] > 0
    }
    
    spec fun contains_zero(v: vector<u64>): bool {
        // exists: existential quantifier
        exists i in 0..len(v): v[i] == 0
    }
    
    // ============================================
    // Old values (before function execution)
    // ============================================
    
    struct Counter has key { value: u64 }
    
    public entry fun increment(addr: address) acquires Counter {
        let c = borrow_global_mut<Counter>(addr);
        c.value = c.value + 1;
    }
    
    spec increment {
        requires exists<Counter>(addr);
        requires global<Counter>(addr).value < MAX_U64;
        // old(): value of expr before function ran
        ensures global<Counter>(addr).value == old(global<Counter>(addr).value) + 1;
    }
    
    // ============================================
    // Spec helper functions
    // ============================================
    
    spec fun is_valid_address(addr: address): bool {
        addr != @0x0
    }
    
    spec fun token_balance(holder: address): u64 {
        // Access global state in spec
        if (exists<Counter>(holder)) {
            global<Counter>(holder).value
        } else {
            0
        }
    }
}
```

---

## Basic Specifications

```move
module verified::basic_specs {
    use std::signer;
    
    // ============================================
    // Resource management specs
    // ============================================
    
    struct Token has key {
        balance: u64,
    }
    
    public entry fun create_account(user: &signer, initial: u64) {
        move_to(user, Token { balance: initial });
    }
    
    spec create_account {
        let addr = signer::address_of(user);
        // Precondition: account must not exist
        requires !exists<Token>(addr);
        // Postcondition: account created with correct balance
        ensures exists<Token>(addr);
        ensures global<Token>(addr).balance == initial;
        // Aborts if account already exists
        aborts_if exists<Token>(addr);
    }
    
    public entry fun deposit(user: &signer, amount: u64) acquires Token {
        let addr = signer::address_of(user);
        let token = borrow_global_mut<Token>(addr);
        token.balance = token.balance + amount;
    }
    
    spec deposit {
        let addr = signer::address_of(user);
        // Preconditions
        requires exists<Token>(addr);
        requires global<Token>(addr).balance + amount <= MAX_U64;
        // Postconditions
        ensures global<Token>(addr).balance == old(global<Token>(addr).balance) + amount;
        // Abort conditions
        aborts_if !exists<Token>(addr);
        aborts_if global<Token>(addr).balance + amount > MAX_U64;
    }
    
    public entry fun withdraw(user: &signer, amount: u64) acquires Token {
        let addr = signer::address_of(user);
        let token = borrow_global_mut<Token>(addr);
        assert!(token.balance >= amount, 1);
        token.balance = token.balance - amount;
    }
    
    spec withdraw {
        let addr = signer::address_of(user);
        requires exists<Token>(addr);
        requires global<Token>(addr).balance >= amount;
        ensures global<Token>(addr).balance == old(global<Token>(addr).balance) - amount;
        aborts_if !exists<Token>(addr);
        aborts_if global<Token>(addr).balance < amount with 1;
    }
    
    public entry fun transfer(
        sender: &signer,
        recipient: address,
        amount: u64,
    ) acquires Token {
        let sender_addr = signer::address_of(sender);
        let sender_token = borrow_global_mut<Token>(sender_addr);
        assert!(sender_token.balance >= amount, 1);
        sender_token.balance = sender_token.balance - amount;
        
        let recipient_token = borrow_global_mut<Token>(recipient);
        recipient_token.balance = recipient_token.balance + amount;
    }
    
    spec transfer {
        let sender_addr = signer::address_of(sender);
        
        requires exists<Token>(sender_addr);
        requires exists<Token>(recipient);
        requires global<Token>(sender_addr).balance >= amount;
        requires global<Token>(recipient).balance + amount <= MAX_U64;
        // Self-transfer: balance unchanged
        requires sender_addr != recipient;
        
        // Sender balance decreases
        ensures global<Token>(sender_addr).balance == 
            old(global<Token>(sender_addr).balance) - amount;
        // Recipient balance increases
        ensures global<Token>(recipient).balance == 
            old(global<Token>(recipient).balance) + amount;
        
        // Conservation: total supply unchanged
        ensures global<Token>(sender_addr).balance + global<Token>(recipient).balance ==
            old(global<Token>(sender_addr).balance + global<Token>(recipient).balance);
        
        aborts_if !exists<Token>(sender_addr);
        aborts_if !exists<Token>(recipient);
        aborts_if global<Token>(sender_addr).balance < amount with 1;
        aborts_if global<Token>(recipient).balance + amount > MAX_U64;
    }
}
```

---

## Invariants

```move
module verified::invariants {
    use std::signer;
    
    // ============================================
    // Module invariants: always true conditions
    // ============================================
    
    struct Treasury has key {
        reserves: u64,
        total_issued: u64,
    }
    
    // Module-level invariant: reserves always >= 0
    spec module {
        // Global invariant: treasury reserves must equal issued
        // (This is checked after every public function)
        invariant forall addr: address where exists<Treasury>(addr):
            global<Treasury>(addr).reserves >= 0;
    }
    
    // Struct invariant
    spec Treasury {
        // reserves must always be <= total_issued (can't have more reserves than issued)
        // Actually: total_issued tracks how many tokens were created
        invariant reserves <= total_issued;
    }
    
    // ============================================
    // Update invariant (checked before and after)
    // ============================================
    
    struct Pool has key {
        reserve_x: u64,
        reserve_y: u64,
        k: u128,  // constant product
    }
    
    spec Pool {
        // k must equal reserve_x * reserve_y
        invariant (k as u128) == (reserve_x as u128) * (reserve_y as u128);
        // Both reserves must be positive
        invariant reserve_x > 0;
        invariant reserve_y > 0;
    }
    
    public entry fun swap(pool_addr: address, dx: u64) acquires Pool {
        let pool = borrow_global_mut<Pool>(pool_addr);
        let ry = pool.reserve_y;
        let rx = pool.reserve_x;
        
        // dy = ry * dx / (rx + dx)
        let dy = ry * dx / (rx + dx);
        
        pool.reserve_x = rx + dx;
        pool.reserve_y = ry - dy;
        pool.k = (pool.reserve_x as u128) * (pool.reserve_y as u128);
    }
    
    spec swap {
        let pool_before = global<Pool>(pool_addr);
        
        requires exists<Pool>(pool_addr);
        requires dx > 0;
        // After swap, k should be >= original (fees make it increase)
        ensures global<Pool>(pool_addr).k >= pool_before.k;
        // Reserves stay positive
        ensures global<Pool>(pool_addr).reserve_x > 0;
        ensures global<Pool>(pool_addr).reserve_y > 0;
    }
    
    // ============================================
    // Data structure invariants
    // ============================================
    
    struct SortedList has key {
        values: vector<u64>,
    }
    
    spec SortedList {
        // Values must be in ascending order
        invariant forall i in 0..len(values), j in 0..len(values):
            i < j ==> values[i] <= values[j];
    }
    
    public fun insert_sorted(list: &mut SortedList, value: u64) {
        let mut i = 0u64;
        let len = std::vector::length(&list.values);
        
        while (i < len && *std::vector::borrow(&list.values, i) < value) {
            i = i + 1;
        };
        
        std::vector::insert(&mut list.values, value, i);
    }
    
    spec insert_sorted {
        ensures len(list.values) == len(old(list.values)) + 1;
        ensures contains(list.values, value);
        // Sorted order maintained (checked via SortedList invariant)
    }
}
```

---

## Aborts Specifications

```move
module verified::abort_specs {
    // ============================================
    // Complete abort specification
    // ============================================
    
    const E_UNAUTHORIZED: u64 = 1;
    const E_INSUFFICIENT: u64 = 2;
    const E_PAUSED: u64 = 3;
    const E_ZERO: u64 = 4;
    
    struct State has key {
        admin: address,
        paused: bool,
        balance: u64,
    }
    
    public entry fun admin_withdraw(
        caller: &signer,
        state_addr: address,
        amount: u64,
    ) acquires State {
        let state = borrow_global_mut<State>(state_addr);
        assert!(std::signer::address_of(caller) == state.admin, E_UNAUTHORIZED);
        assert!(!state.paused, E_PAUSED);
        assert!(amount > 0, E_ZERO);
        assert!(state.balance >= amount, E_INSUFFICIENT);
        state.balance = state.balance - amount;
    }
    
    spec admin_withdraw {
        let caller_addr = std::signer::address_of(caller);
        let pre_state = global<State>(state_addr);
        
        // Every abort condition specified
        aborts_if !exists<State>(state_addr);
        aborts_if caller_addr != pre_state.admin with E_UNAUTHORIZED;
        aborts_if pre_state.paused with E_PAUSED;
        aborts_if amount == 0 with E_ZERO;
        aborts_if pre_state.balance < amount with E_INSUFFICIENT;
        
        // On success:
        ensures !aborts;
        ensures global<State>(state_addr).balance == pre_state.balance - amount;
        
        // Pragma: tell prover about known abort codes
        pragma aborts_if_is_strict = true;
    }
    
    // ============================================
    // Pragma options
    // ============================================
    
    spec module {
        // Global prover settings
        pragma timeout = 60;         // seconds
        pragma verify = true;        // enable verification
        pragma aborts_if_is_partial; // allow partial abort specs
    }
    
    // ============================================
    // Helper specs (reusable conditions)
    // ============================================
    
    spec fun state_valid(state_addr: address): bool {
        exists<State>(state_addr) &&
        global<State>(state_addr).balance <= MAX_U64
    }
    
    spec fun is_admin(caller: address, state_addr: address): bool {
        state_valid(state_addr) &&
        caller == global<State>(state_addr).admin
    }
    
    // ============================================
    // Opaque function (spec without implementation)
    // ============================================
    
    // For external functions where implementation is unknown
    // spec fun external_price(): u64;
    
    // Mock spec for testing
    spec fun mock_price(asset: address): u64 {
        1_000_000  // always return $1 in tests
    }
}
```

---

## ตัวอย่าง: Verified Token

```move
module verified::verified_token {
    use std::signer;
    
    // ============================================
    // Fully verified token with conservation proofs
    // ============================================
    
    struct VerifiedToken has key {
        balance: u64,
    }
    
    struct TokenManager has key {
        admin: address,
        total_supply: u64,
        is_paused: bool,
    }
    
    // Conservation invariant: sum of all balances == total_supply
    // (Hard to express across all addresses; use ghost variables)
    
    spec module {
        global ghost_total: num;
        
        // Total supply never exceeds u64 max
        invariant forall addr: address where exists<TokenManager>(addr):
            global<TokenManager>(addr).total_supply <= MAX_U64;
    }
    
    public entry fun initialize(admin: &signer) {
        move_to(admin, TokenManager {
            admin: signer::address_of(admin),
            total_supply: 0,
            is_paused: false,
        });
    }
    
    spec initialize {
        let addr = signer::address_of(admin);
        aborts_if exists<TokenManager>(addr);
        ensures global<TokenManager>(addr).total_supply == 0;
        ensures global<TokenManager>(addr).admin == addr;
        ensures !global<TokenManager>(addr).is_paused;
    }
    
    public entry fun mint(
        admin: &signer,
        manager_addr: address,
        recipient: address,
        amount: u64,
    ) acquires TokenManager, VerifiedToken {
        let manager = borrow_global_mut<TokenManager>(manager_addr);
        assert!(signer::address_of(admin) == manager.admin, 1);
        assert!(!manager.is_paused, 2);
        assert!(manager.total_supply + amount <= MAX_U64, 3);
        
        manager.total_supply = manager.total_supply + amount;
        
        if (exists<VerifiedToken>(recipient)) {
            let token = borrow_global_mut<VerifiedToken>(recipient);
            assert!(token.balance + amount <= MAX_U64, 3);
            token.balance = token.balance + amount;
        } else {
            // Can only mint to self in this design (need recipient's signer)
            // In production: use resource account or different pattern
        };
    }
    
    spec mint {
        let manager_addr_pre = global<TokenManager>(manager_addr);
        
        requires exists<TokenManager>(manager_addr);
        requires signer::address_of(admin) == manager_addr_pre.admin;
        requires !manager_addr_pre.is_paused;
        requires manager_addr_pre.total_supply + amount <= MAX_U64;
        
        // Total supply increases
        ensures global<TokenManager>(manager_addr).total_supply == 
            old(global<TokenManager>(manager_addr).total_supply) + amount;
        
        aborts_if !exists<TokenManager>(manager_addr);
        aborts_if signer::address_of(admin) != manager_addr_pre.admin with 1;
        aborts_if manager_addr_pre.is_paused with 2;
        aborts_if manager_addr_pre.total_supply + amount > MAX_U64 with 3;
    }
    
    // ============================================
    // Transfer with full conservation proof
    // ============================================
    
    public entry fun transfer(
        sender: &signer,
        recipient: address,
        amount: u64,
    ) acquires VerifiedToken {
        let sender_addr = signer::address_of(sender);
        
        assert!(exists<VerifiedToken>(sender_addr), 10);
        assert!(exists<VerifiedToken>(recipient), 11);
        assert!(amount > 0, 12);
        
        let sender_token = borrow_global_mut<VerifiedToken>(sender_addr);
        assert!(sender_token.balance >= amount, 13);
        sender_token.balance = sender_token.balance - amount;
        
        let recipient_token = borrow_global_mut<VerifiedToken>(recipient);
        recipient_token.balance = recipient_token.balance + amount;
    }
    
    spec transfer {
        let sender_addr = signer::address_of(sender);
        let pre_sender = global<VerifiedToken>(sender_addr).balance;
        let pre_recipient = global<VerifiedToken>(recipient).balance;
        
        requires sender_addr != recipient;
        requires exists<VerifiedToken>(sender_addr);
        requires exists<VerifiedToken>(recipient);
        requires amount > 0;
        requires pre_sender >= amount;
        requires pre_recipient + amount <= MAX_U64;
        
        // Sender loses exactly `amount`
        ensures global<VerifiedToken>(sender_addr).balance == pre_sender - amount;
        // Recipient gains exactly `amount`
        ensures global<VerifiedToken>(recipient).balance == pre_recipient + amount;
        // Conservation: total unchanged
        ensures 
            global<VerifiedToken>(sender_addr).balance + 
            global<VerifiedToken>(recipient).balance == 
            pre_sender + pre_recipient;
        
        aborts_if !exists<VerifiedToken>(sender_addr) with 10;
        aborts_if !exists<VerifiedToken>(recipient) with 11;
        aborts_if amount == 0 with 12;
        aborts_if pre_sender < amount with 13;
        aborts_if pre_recipient + amount > MAX_U64;
    }
    
    // ============================================
    // Verification test
    // ============================================
    
    #[test]
    fun test_transfer_conservation() acquires VerifiedToken {
        let alice = std::account::create_account_for_test(@0xALICE);
        let bob = std::account::create_account_for_test(@0xBOB);
        
        move_to(&alice, VerifiedToken { balance: 1000 });
        move_to(&bob, VerifiedToken { balance: 500 });
        
        transfer(&alice, @0xBOB, 300);
        
        assert!(borrow_global<VerifiedToken>(@0xALICE).balance == 700, 1);
        assert!(borrow_global<VerifiedToken>(@0xBOB).balance == 800, 2);
        // Total: 700 + 800 = 1500 = original 1000 + 500 ✓
    }
}
```

---

## Running the Prover

```bash
# Install Move Prover dependencies
aptos init  # configure Aptos CLI

# Prove a module
aptos move prove \
  --package-dir . \
  --named-addresses my_module=0x1

# Prove with verbose output
aptos move prove \
  --package-dir . \
  --verbose

# Common prover output:
# [SUCCESS] proving verify_token.move ... (3.2s)
# [FAILURE] proving invariant in swap ... 
#   Counterexample: reserve_x=0 violates invariant reserve_x > 0

# Prover settings in Move.toml:
# [prover]
# timeout = 120
# backend = "z3"
```

---

## สรุป Move Prover

```
Spec Constructs    | Meaning
------------------|----------------------------------
requires P        | P must hold before function
ensures P         | P holds after function (on success)
aborts_if P       | Function aborts iff P
aborts_if P with E| Aborts with error code E iff P
modifies x        | Function may modify x
invariant P       | P always holds for struct/module
old(expr)         | Value of expr before function
forall x: T ...   | Universal quantifier
exists x: T ...   | Existential quantifier
global<T>(addr)   | Global storage access in spec
```

**Best Practices:**
1. Specify all abort conditions
2. Prove conservation properties (total supply unchanged)
3. Add invariants to structs that must maintain shape
4. Use `pragma aborts_if_is_strict` for complete specs
5. Start simple, add complexity gradually
6. Run prover in CI/CD pipeline

---

**ก่อนหน้า**: [Part 38 - Advanced Sui ←](part-38-sui-advanced.md)
**ต่อไป**: [Part 40 - Production Deployment →](part-40-production-deployment.md)
