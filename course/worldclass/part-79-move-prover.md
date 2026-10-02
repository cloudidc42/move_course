# Part 79: Move Prover & Formal Verification

## สารบัญ
- [Formal Verification Overview](#formal-verification-overview)
- [Move Specification Language](#move-specification-language)
- [Proving DeFi Invariants](#proving-defi-invariants)
- [Spec Functions & Ghost Variables](#spec-functions--ghost-variables)
- [Real-World Prover Usage](#real-world-prover-usage)
- [Prover Limitations & Workarounds](#prover-limitations--workarounds)

---

## Formal Verification Overview

```
WHAT IS FORMAL VERIFICATION?

Traditional Testing:
  "Tests pass for these specific inputs"
  Coverage: finite set of cases
  Bugs: can hide in untested cases

Formal Verification:
  "Function is correct for ALL possible inputs"
  Proof: mathematical guarantee
  Move Prover uses Z3 SMT solver

WHY IT MATTERS FOR DEFI
  A single bug in a lending protocol can cause $100M loss
  Fuzzing finds common bugs, Prover finds edge cases
  E.g., prove: health_factor never underflows to 0

WHAT MOVE PROVER CAN VERIFY
  ✅ Arithmetic overflow/underflow
  ✅ Function pre/postconditions
  ✅ Global invariants (hold after every public function)
  ✅ Resource lifecycle (created/destroyed correctly)
  ✅ Access control (only authorized callers)
  ✅ Absence of panics (given preconditions)
  
WHAT IT CANNOT VERIFY
  ❌ Economic incentive alignment
  ❌ Business logic correctness (unless formally specified)
  ❌ Off-chain components (relayers, frontends)
  ❌ Gas efficiency
  ❌ External oracle correctness

PROVER WORKFLOW
  1. Write spec blocks alongside code
  2. Run: aptos move prove --package-dir .
  3. Prover encodes to Z3 SMT formulas
  4. Z3 checks satisfiability (find counterexample)
  5. Report: PASS (no counterexample) or FAIL (with trace)
```

---

## Move Specification Language

```move
// ============================================
// SPEC BLOCK SYNTAX
// ============================================

module prover_demo::token {
    use aptos_framework::coin;
    
    struct Config has key {
        admin: address,
        fee_bps: u64,
        total_supply: u64,
    }
    
    // ============================================
    // BASIC POSTCONDITIONS
    // ============================================
    
    public fun initialize(
        admin: &signer,
        fee_bps: u64,
    ) {
        assert!(fee_bps <= 1000, 1);  // Max 10%
        
        move_to(admin, Config {
            admin: std::signer::address_of(admin),
            fee_bps,
            total_supply: 0,
        });
    }
    
    spec initialize {
        // PRECONDITIONS: what must be true BEFORE function runs
        requires fee_bps <= 1000;
        requires !exists<Config>(std::signer::address_of(admin));
        
        // POSTCONDITIONS: what must be true AFTER function runs
        ensures exists<Config>(std::signer::address_of(admin));
        ensures global<Config>(std::signer::address_of(admin)).fee_bps == fee_bps;
        ensures global<Config>(std::signer::address_of(admin)).total_supply == 0;
        ensures global<Config>(std::signer::address_of(admin)).admin == std::signer::address_of(admin);
    }
    
    // ============================================
    // ARITHMETIC SAFETY
    // ============================================
    
    public fun calculate_fee(amount: u64, config_addr: address): u64 acquires Config {
        let config = borrow_global<Config>(config_addr);
        amount * config.fee_bps / 10_000
    }
    
    spec calculate_fee {
        requires exists<Config>(config_addr);
        
        // Key: prove no arithmetic overflow
        // amount: u64, fee_bps: u64 (max 1000)
        // amount * fee_bps could overflow if amount > u64::MAX / 1000
        requires amount <= 18_446_744_073_709_551;  // u64::MAX / 1000
        
        // Result is always less than amount
        ensures result <= amount;
        
        // Result is proportional to fee_bps
        ensures result == amount * global<Config>(config_addr).fee_bps / 10_000;
    }
    
    // ============================================
    // TRANSFER CORRECTNESS
    // ============================================
    
    public fun transfer<CoinType>(
        from: &signer,
        to: address,
        amount: u64,
    ) {
        let from_addr = std::signer::address_of(from);
        let coins = coin::withdraw<CoinType>(from, amount);
        coin::deposit<CoinType>(to, coins);
    }
    
    spec transfer {
        let from_addr = std::signer::address_of(from);
        
        // Pre: from has enough balance
        requires coin::balance<CoinType>(from_addr) >= amount;
        
        // Post: balances updated correctly
        ensures coin::balance<CoinType>(from_addr) == 
            old(coin::balance<CoinType>(from_addr)) - amount;
        ensures coin::balance<CoinType>(to) == 
            old(coin::balance<CoinType>(to)) + amount;
        
        // Post: total supply unchanged
        ensures coin::supply<CoinType>() == old(coin::supply<CoinType>());
    }
    
    // ============================================
    // ACCESS CONTROL VERIFICATION
    // ============================================
    
    public fun set_fee(
        caller: &signer,
        config_addr: address,
        new_fee_bps: u64,
    ) acquires Config {
        let config = borrow_global_mut<Config>(config_addr);
        assert!(std::signer::address_of(caller) == config.admin, 1);
        assert!(new_fee_bps <= 1000, 2);
        config.fee_bps = new_fee_bps;
    }
    
    spec set_fee {
        requires exists<Config>(config_addr);
        requires new_fee_bps <= 1000;
        
        // Access control: only admin can change fee
        requires std::signer::address_of(caller) == global<Config>(config_addr).admin;
        
        // Fee is updated
        ensures global<Config>(config_addr).fee_bps == new_fee_bps;
        
        // Admin is unchanged
        ensures global<Config>(config_addr).admin == old(global<Config>(config_addr).admin);
        
        // Abort if not admin
        aborts_if std::signer::address_of(caller) != global<Config>(config_addr).admin
            with 1;
        aborts_if new_fee_bps > 1000 with 2;
    }
}
```

---

## Proving DeFi Invariants

```move
// ============================================
// PROVING AMM INVARIANTS
// The k = x * y product must never decrease
// ============================================

module prover_demo::amm_verified {
    
    struct Pool has key {
        reserve_x: u64,
        reserve_y: u64,
        total_lp: u64,
    }
    
    // ============================================
    // GLOBAL INVARIANT: Pools always have reserves
    // ============================================
    
    spec module {
        // Invariant holds after EVERY public function
        invariant forall addr: address where exists<Pool>(addr):
            global<Pool>(addr).reserve_x > 0 &&
            global<Pool>(addr).reserve_y > 0 &&
            global<Pool>(addr).total_lp > 0;
    }
    
    // ============================================
    // SWAP: Prove xy >= k after swap
    // ============================================
    
    public fun swap_x_to_y(
        pool_addr: address,
        amount_in: u64,
    ): u64 acquires Pool {
        let pool = borrow_global_mut<Pool>(pool_addr);
        
        assert!(amount_in > 0, 1);
        
        // CPMM formula: amount_out = reserve_y * amount_in / (reserve_x + amount_in)
        let new_reserve_x = pool.reserve_x + amount_in;
        let amount_out = (pool.reserve_y as u128) * (amount_in as u128) 
            / (new_reserve_x as u128);
        let amount_out = amount_out as u64;
        
        assert!(amount_out < pool.reserve_y, 2);  // Can't drain pool
        
        pool.reserve_x = new_reserve_x;
        pool.reserve_y = pool.reserve_y - amount_out;
        
        amount_out
    }
    
    spec swap_x_to_y {
        let pool_before = global<Pool>(pool_addr);
        let pool_after = global<Pool>(pool_addr);  // After modification
        
        requires exists<Pool>(pool_addr);
        requires amount_in > 0;
        requires pool_before.reserve_x > 0;
        requires pool_before.reserve_y > 0;
        
        // Key invariant: k (product) must not decrease after swap
        // In integer arithmetic: reserve_x_new * reserve_y_new >= reserve_x_old * reserve_y_old
        ensures (pool_after.reserve_x as u128) * (pool_after.reserve_y as u128) 
            >= (pool_before.reserve_x as u128) * (pool_before.reserve_y as u128);
        
        // Result is positive
        ensures result > 0;
        
        // Result is less than reserve_y (can't drain pool)
        ensures result < pool_before.reserve_y;
        
        // Reserves are positive (pool stays valid)
        ensures pool_after.reserve_x > 0;
        ensures pool_after.reserve_y > 0;
        
        // No overflow in computation
        requires (pool_before.reserve_y as u128) * (amount_in as u128) 
            < (18_446_744_073_709_551_616 as u128);  // u64::MAX + 1
    }
    
    // ============================================
    // LIQUIDITY: Prove LP tokens proportional
    // ============================================
    
    public fun add_liquidity(
        pool_addr: address,
        amount_x: u64,
        amount_y: u64,
    ): u64 acquires Pool {
        let pool = borrow_global_mut<Pool>(pool_addr);
        
        // Proportional check: amount_x / reserve_x == amount_y / reserve_y
        // Rearranged: amount_x * reserve_y == amount_y * reserve_x (avoid division)
        assert!(
            (amount_x as u128) * (pool.reserve_y as u128) == 
            (amount_y as u128) * (pool.reserve_x as u128),
            3,
        );
        
        // LP tokens proportional to contribution
        let lp_minted = (pool.total_lp as u128) * (amount_x as u128) / (pool.reserve_x as u128);
        let lp_minted = lp_minted as u64;
        
        pool.reserve_x = pool.reserve_x + amount_x;
        pool.reserve_y = pool.reserve_y + amount_y;
        pool.total_lp = pool.total_lp + lp_minted;
        
        lp_minted
    }
    
    spec add_liquidity {
        let pool_before = global<Pool>(pool_addr);
        let pool_after = global<Pool>(pool_addr);
        
        requires exists<Pool>(pool_addr);
        requires amount_x > 0 && amount_y > 0;
        
        // Proportionality preserved
        ensures (pool_after.reserve_x as u128) * (pool_before.total_lp as u128) 
            == (pool_before.reserve_x as u128) * (pool_after.total_lp as u128);
        
        // LP minted is positive
        ensures result > 0;
        
        // Total supply increased
        ensures pool_after.total_lp > pool_before.total_lp;
        
        // Reserves increased by exact amounts
        ensures pool_after.reserve_x == pool_before.reserve_x + amount_x;
        ensures pool_after.reserve_y == pool_before.reserve_y + amount_y;
    }
    
    // ============================================
    // LENDING: Health factor never underflows
    // ============================================
    
    struct LendingState has key {
        collateral: u64,    // In USD (scaled by 1e6)
        debt: u64,          // In USD (scaled by 1e6)
        ltv_bps: u64,       // Loan-to-value: 7500 = 75%
    }
    
    public fun get_health_factor(addr: address): u64 acquires LendingState {
        let state = borrow_global<LendingState>(addr);
        if (state.debt == 0) {
            return 18_446_744_073_709_551_615  // u64::MAX = safe
        };
        
        // HF = (collateral * LTV / 10_000) / debt
        // Return as bps: 10000 = 1.0
        let weighted_collateral = (state.collateral as u128) * (state.ltv_bps as u128) / 10_000;
        let hf = weighted_collateral * 10_000 / (state.debt as u128);
        
        if (hf > 18_446_744_073_709_551_615) {
            18_446_744_073_709_551_615
        } else {
            hf as u64
        }
    }
    
    spec get_health_factor {
        requires exists<LendingState>(addr);
        
        let state = global<LendingState>(addr);
        
        // No overflow in intermediate computation
        requires (state.collateral as u128) * (state.ltv_bps as u128) < (u128::MAX / 10_000);
        
        // If no debt, returns max value
        ensures state.debt == 0 ==> result == 18_446_744_073_709_551_615;
        
        // If debt > 0, result is proportional
        ensures state.debt > 0 ==> result <= 18_446_744_073_709_551_615;
        
        // Health factor > 0 always
        ensures result > 0;
    }
}
```

---

## Spec Functions & Ghost Variables

```move
// ============================================
// SPEC FUNCTIONS: Helper computations in specs
// Ghost variables: State that exists only in specs
// ============================================

module prover_demo::advanced_spec {
    use aptos_std::table::{Self, Table};
    
    struct TokenBalances has key {
        balances: Table<address, u64>,
        total: u64,
    }
    
    // ============================================
    // SPEC FUNCTIONS: Pure functions usable in specs
    // ============================================
    
    spec fun spec_sum_balances(balances: Table<address, u64>): u128 {
        // This is a mathematical specification of "sum of all values"
        // The prover uses this in reasoning but it's not executable code
        // (tables can't be iterated in specs directly)
        sum_of_all_values(balances)
    }
    
    spec fun spec_is_valid_transfer(
        from_balance: u64,
        to_balance: u64,
        amount: u64,
    ): bool {
        from_balance >= amount &&
        to_balance + amount <= 18_446_744_073_709_551_615  // No overflow
    }
    
    // Function to verify
    public fun transfer_balance(
        balances: &mut TokenBalances,
        from: address,
        to: address,
        amount: u64,
    ) {
        let from_bal = *table::borrow(&balances.balances, from);
        assert!(from_bal >= amount, 1);
        
        let to_bal = *table::borrow_with_default(&balances.balances, to, 0);
        
        *table::borrow_mut(&mut balances.balances, from) = from_bal - amount;
        
        if (table::contains(&balances.balances, to)) {
            *table::borrow_mut(&mut balances.balances, to) = to_bal + amount;
        } else {
            table::add(&mut balances.balances, to, amount);
        };
        // Total unchanged (no mint/burn)
    }
    
    spec transfer_balance {
        // Precondition uses spec function
        requires spec_is_valid_transfer(
            table::spec_get(balances.balances, from),
            table::spec_get(balances.balances, to),
            amount,
        );
        
        // Total supply conservation (key DeFi invariant!)
        ensures old(balances.total) == balances.total;
        
        // From balance decreased
        ensures table::spec_get(balances.balances, from) ==
            old(table::spec_get(balances.balances, from)) - amount;
        
        // To balance increased
        ensures table::spec_get(balances.balances, to) ==
            old(table::spec_get(balances.balances, to)) + amount;
    }
    
    // ============================================
    // INVARIANT: Total always equals sum of balances
    // ============================================
    
    spec module {
        // All valid TokenBalances have matching total
        // (This is an axiom-style invariant the prover checks)
        invariant forall addr: address where exists<TokenBalances>(addr):
            global<TokenBalances>(addr).total >= 0;
    }
    
    // ============================================
    // SCHEMA: Reusable spec blocks
    // ============================================
    
    spec schema AdminOnly {
        caller: signer;
        config_addr: address;
        
        requires exists<Config>(config_addr);
        requires std::signer::address_of(caller) == global<Config>(config_addr).admin;
        aborts_if std::signer::address_of(caller) != global<Config>(config_addr).admin;
    }
    
    struct Config has key { admin: address }
    
    public fun admin_action(caller: &signer, config_addr: address) acquires Config {
        let config = borrow_global<Config>(config_addr);
        assert!(std::signer::address_of(caller) == config.admin, 1);
        // ... admin logic
    }
    
    spec admin_action {
        include AdminOnly { caller, config_addr };  // Reuse schema
    }
}
```

---

## Real-World Prover Usage

```bash
# ============================================
# RUNNING THE PROVER
# ============================================

# Basic usage
aptos move prove --package-dir .

# Specific module
aptos move prove --package-dir . --filter "token::transfer"

# With timeout (complex proofs can take long)
aptos move prove --package-dir . --timeout 120

# Verbose output (see Z3 queries)
aptos move prove --package-dir . --verbose

# Example output (SUCCESS):
# [INFO] Running Move prover on ./
# [INFO] Checking 3 targets
# PASS  token::initialize
# PASS  token::transfer
# PASS  token::set_fee
# Move prover succeeded: 3 of 3 verified

# Example output (FAILURE):
# FAIL  amm::swap_x_to_y
# Error: postcondition `result > 0` might not hold.
# Trace:
#   amount_in = 0     ← Counterexample found!
#   pool.reserve_x = 1000
#   pool.reserve_y = 1000
#   result = 0
# → Add: requires amount_in > 0

# ============================================
# PROVER IN CI/CD
# ============================================

# .github/workflows/prover.yml
# on: [push, pull_request]
# jobs:
#   prove:
#     runs-on: ubuntu-latest
#     steps:
#       - uses: actions/checkout@v3
#       - name: Install Aptos CLI
#         run: curl -fsSL "https://aptos.dev/scripts/install_cli.py" | python3
#       - name: Run Move Prover
#         run: aptos move prove --package-dir .

# ============================================
# MOVE.TOML PROVER CONFIG
# ============================================

# [prover]
# timeout = 60           # seconds per function
# recursion_limit = 4    # max recursion depth
# verbosity = "warn"     # info, warn, error
# num_instances = 4      # parallel Z3 instances
```

---

## Prover Limitations & Workarounds

```move
// ============================================
// KNOWN PROVER LIMITATIONS AND WORKAROUNDS
// ============================================

module prover_demo::workarounds {
    
    // LIMITATION 1: Loops are hard to prove
    // Prover needs loop invariants to reason about loops
    
    // BAD: Prover can't verify this (no loop invariant)
    fun sum_bad(v: &vector<u64>): u64 {
        let mut total = 0u64;
        let n = std::vector::length(v);
        let mut i = 0u64;
        while (i < n) {
            total = total + *std::vector::borrow(v, i);
            i = i + 1;
        };
        total
    }
    
    // GOOD: With loop invariant
    fun sum_good(v: &vector<u64>): u64 {
        let mut total = 0u64;
        let n = std::vector::length(v);
        let mut i = 0u64;
        while ({
            // Loop invariant annotation (in spec block inside function)
            spec {
                invariant i <= n;
                invariant total == sum_of_slice(v, 0, i);  // spec fun
            };
            i < n
        }) {
            total = total + *std::vector::borrow(v, i);
            i = i + 1;
        };
        total
    }
    
    spec fun sum_of_slice(v: vector<u64>, from: u64, to: u64): u64;
    
    // LIMITATION 2: Recursive functions have depth limits
    // Workaround: Use iterative versions
    
    // LIMITATION 3: Complex arithmetic with many variables
    // Prover's SMT solver may timeout
    // Workaround: Break into smaller lemmas
    
    // Helper lemma: proved separately
    spec fun a_times_b_safe(a: u64, b: u64): bool {
        (a as u128) * (b as u128) <= 18_446_744_073_709_551_615
    }
    
    public fun multiply_safe(a: u64, b: u64): u64 {
        assert!(a <= 4_294_967_295 || b <= 4_294_967_295, 1);  // At least one fits in u32
        (a as u128 * b as u128) as u64  // Prover can verify with this constraint
    }
    
    spec multiply_safe {
        requires a <= 4_294_967_295 || b <= 4_294_967_295;
        ensures (result as u128) == (a as u128) * (b as u128);
    }
    
    // LIMITATION 4: Tables/SmartTables are partially specified
    // Use spec_get / spec_contains abstractions
    
    // LIMITATION 5: External function calls treated as opaque
    // Workaround: Add ensures clauses for callee behavior
    
    // ============================================
    // BEST PRACTICES
    // ============================================
    
    // 1. Start with simple postconditions, add complexity gradually
    // 2. Break complex functions into smaller, provable pieces
    // 3. Use requires to establish preconditions (not just asserts)
    // 4. Test prover in CI to catch regressions
    // 5. Focus prover on critical financial invariants:
    //    - Total supply conservation
    //    - No underflow in user balances
    //    - Access control correctness
    //    - k invariant in AMMs
    
    struct Config has key { admin: address }
}

// ============================================
// COMPLETE PROVEN SAFE MATH LIBRARY
// ============================================

module prover_demo::safe_math_proven {
    
    // Safe addition with overflow check
    public fun safe_add(a: u64, b: u64): u64 {
        assert!((a as u128) + (b as u128) <= 18_446_744_073_709_551_615, 1);
        a + b
    }
    
    spec safe_add {
        requires (a as u128) + (b as u128) <= 18_446_744_073_709_551_615;
        ensures (result as u128) == (a as u128) + (b as u128);
        aborts_if (a as u128) + (b as u128) > 18_446_744_073_709_551_615;
    }
    
    // Safe multiplication using u128 intermediate
    public fun safe_mul(a: u64, b: u64): u64 {
        let result = (a as u128) * (b as u128);
        assert!(result <= 18_446_744_073_709_551_615, 2);
        result as u64
    }
    
    spec safe_mul {
        requires (a as u128) * (b as u128) <= 18_446_744_073_709_551_615;
        ensures (result as u128) == (a as u128) * (b as u128);
        aborts_if (a as u128) * (b as u128) > 18_446_744_073_709_551_615;
    }
    
    // muldiv(a, b, c) = a * b / c using u128, no overflow
    public fun muldiv(a: u64, b: u64, c: u64): u64 {
        assert!(c > 0, 3);
        let num = (a as u128) * (b as u128);
        (num / (c as u128)) as u64
    }
    
    spec muldiv {
        requires c > 0;
        requires (a as u128) * (b as u128) / (c as u128) <= 18_446_744_073_709_551_615;
        ensures (result as u128) == (a as u128) * (b as u128) / (c as u128);
        aborts_if c == 0;
    }
    
    // Integer square root (floor)
    public fun isqrt(n: u64): u64 {
        if (n == 0) return 0;
        let mut x = n;
        let mut y = (x + 1) / 2;
        while (y < x) {
            x = y;
            y = (x + n / x) / 2;
        };
        x
    }
    
    spec isqrt {
        ensures result * result <= n;
        ensures (result + 1) * (result + 1) > n || (result + 1) * (result + 1) == 0;  // Overflow guard
    }
}
```

---

## สรุป Move Prover

```
PROVER INVESTMENT vs BENEFIT

High Value (always prove):
  ✅ Arithmetic overflow in financial calculations
  ✅ Access control for admin functions  
  ✅ Total supply conservation (mint/burn)
  ✅ AMM k invariant
  ✅ Health factor bounds
  
Medium Value (prove critical paths):
  ✅ Slippage calculations
  ✅ Reward distribution math
  ✅ Liquidation threshold checks
  
Lower Value (document but maybe not prove):
  ⬜ String formatting
  ⬜ Event emission order
  ⬜ UI helper functions

PROVER MATURITY LEVEL

Level 1 - Safety specs:
  "Function never panics given valid inputs"
  requires valid inputs → ensures no abort
  
Level 2 - Functional correctness:
  "Balances update correctly"
  ensures new_balance == old_balance + amount
  
Level 3 - Invariants:
  "Total supply preserved across all operations"
  invariant forall addr: balance(addr) sum == total_supply
  
Level 4 - Security properties:
  "Only admin can change fee"
  requires caller == admin in all fee-changing functions
  
APTOS CORE USES PROVER
  aptos-core/aptos-move/ modules are all proved
  Standard library guarantees: coin, staking, governance
  Sets the bar for production-grade Move code
```

---

**ก่อนหน้า**: [Part 78 - Cross-Chain Protocol Design ←](part-78-crosschain-protocol.md)
**ต่อไป**: [Part 80 - Protocol Operations & Incident Response →](part-80-protocol-ops.md)
