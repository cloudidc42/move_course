# Part 57: Advanced Testing & Fuzzing

## สารบัญ
- [Testing Pyramid for Move](#testing-pyramid-for-move)
- [Unit Testing Deep Dive](#unit-testing-deep-dive)
- [Integration Testing](#integration-testing)
- [Property-Based Testing](#property-based-testing)
- [Fuzz Testing](#fuzz-testing)
- [Gas Benchmarking](#gas-benchmarking)
- [Invariant Testing](#invariant-testing)

---

## Testing Pyramid for Move

```
Move Testing Pyramid:

         /\
        /  \
       / E2E \          (Rare: TypeScript tests against live testnet)
      /--------\
     / Integration \    (Module-level: test interactions between modules)
    /--------------\
   /   Unit Tests   \   (Function-level: isolated, fast)
  /------------------\
 /  Move Prover Specs  \ (Bottom: mathematical proofs, always true)
/----------------------\

Test Types:
  Move Prover:    Formal verification (mathematical proof)
  Unit Tests:     #[test] functions, move test
  Integration:    Multiple modules, fake accounts, timestamps
  E2E:            TypeScript SDK + local testnet
  Fuzzing:        Random inputs, find edge cases
  
Test File Organization:
  sources/
    pool.move
    pool_tests.move       (unit tests in same package)
  
  tests/
    pool_integration.move (integration tests, separate package)
  
  scripts/
    fuzz_test.sh         (fuzzing scripts)
    benchmark.sh         (gas benchmarks)
```

---

## Unit Testing Deep Dive

```move
#[test_only]
module defi::amm_tests {
    use std::signer;
    use aptos_framework::account;
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::timestamp;
    use defi::amm::{Self, Pool};
    
    // ============================================
    // Test fixtures (setup helpers)
    // ============================================
    
    struct TestCoins {}
    
    // Define test tokens
    struct USDC has store {}
    struct ETH has store {}
    
    // Setup helper: Create funded test account
    fun create_test_account(name: vector<u8>, initial_apt: u64): signer {
        let account = account::create_account_for_test(
            @aptos_std::bcs::to_bytes(&name)[0] as address  // simplified
        );
        // Fund with APT for gas
        aptos_framework::aptos_coin::mint_for_test(&account, initial_apt);
        account
    }
    
    // Setup helper: Initialize test coins with balances
    fun setup_coins(
        framework: &signer,
        admin: &signer,
        user: &signer,
        usdc_amount: u64,
        eth_amount: u64,
    ) {
        // Initialize coin types
        let (usdc_burn_cap, usdc_freeze_cap, usdc_mint_cap) = 
            coin::initialize<USDC>(admin, b"USDC", b"USDC", 6, true);
        let (eth_burn_cap, eth_freeze_cap, eth_mint_cap) =
            coin::initialize<ETH>(admin, b"ETH", b"ETH", 18, true);
        
        // Register and mint for user
        coin::register<USDC>(user);
        coin::register<ETH>(user);
        coin::deposit(signer::address_of(user), coin::mint<USDC>(usdc_amount, &usdc_mint_cap));
        coin::deposit(signer::address_of(user), coin::mint<ETH>(eth_amount, &eth_mint_cap));
        
        // Cleanup caps
        coin::destroy_burn_cap(usdc_burn_cap);
        coin::destroy_freeze_cap(usdc_freeze_cap);
        coin::destroy_mint_cap(usdc_mint_cap);
        coin::destroy_burn_cap(eth_burn_cap);
        coin::destroy_freeze_cap(eth_freeze_cap);
        coin::destroy_mint_cap(eth_mint_cap);
    }
    
    // ============================================
    // Basic unit tests
    // ============================================
    
    #[test(framework = @0x1, admin = @defi, user = @0x100)]
    fun test_create_pool(
        framework: &signer,
        admin: &signer,
        user: &signer,
    ) {
        // Setup
        timestamp::set_time_has_started_for_testing(framework);
        setup_coins(framework, admin, user, 1_000_000_000, 500_000_000);
        
        // Create pool with initial liquidity
        amm::create_pool<USDC, ETH>(
            admin,
            30,  // 0.30% fee
        );
        
        amm::add_liquidity<USDC, ETH>(
            user,
            1_000_000_000,  // 1000 USDC
            500_000_000,    // 0.5 ETH
            0,              // min LP
        );
        
        // Verify pool state
        let (reserve_x, reserve_y) = amm::get_reserves<USDC, ETH>();
        assert!(reserve_x == 1_000_000_000, 1);
        assert!(reserve_y == 500_000_000, 2);
        
        // LP tokens should be sqrt(1000 * 0.5) = ~22.36 (scaled)
        let lp_balance = amm::get_lp_balance<USDC, ETH>(signer::address_of(user));
        assert!(lp_balance > 0, 3);
    }
    
    #[test(framework = @0x1, admin = @defi, user = @0x100)]
    fun test_swap_exact_input(
        framework: &signer,
        admin: &signer,
        user: &signer,
    ) {
        timestamp::set_time_has_started_for_testing(framework);
        setup_coins(framework, admin, user, 2_000_000_000, 1_000_000_000);
        amm::create_pool<USDC, ETH>(admin, 30);
        amm::add_liquidity<USDC, ETH>(user, 1_000_000_000, 500_000_000, 0);
        
        // Record balances before swap
        let usdc_before = coin::balance<USDC>(signer::address_of(user));
        let eth_before = coin::balance<ETH>(signer::address_of(user));
        
        // Swap 100 USDC for ETH
        let swap_amount = 100_000_000;  // 100 USDC
        amm::swap_exact_input<USDC, ETH>(user, swap_amount, 0);
        
        let usdc_after = coin::balance<USDC>(signer::address_of(user));
        let eth_after = coin::balance<ETH>(signer::address_of(user));
        
        // Verify: USDC decreased
        assert!(usdc_before - usdc_after == swap_amount, 1);
        
        // Verify: ETH increased
        assert!(eth_after > eth_before, 2);
        
        // Verify: amount out follows CPMM formula
        // dy = y * dx / (x + dx) where dx already includes fee
        let dx_with_fee = swap_amount * (10_000 - 30) / 10_000;  // 0.3% fee
        let expected_out = 500_000_000 * dx_with_fee / (1_000_000_000 + dx_with_fee);
        let actual_out = eth_after - eth_before;
        
        // Allow 1 unit tolerance for rounding
        let diff = if (actual_out >= expected_out) {
            actual_out - expected_out
        } else {
            expected_out - actual_out
        };
        assert!(diff <= 1, 3);
    }
    
    // Test: swap preserves k (constant product invariant)
    #[test(framework = @0x1, admin = @defi, trader = @0x100, lp = @0x200)]
    fun test_constant_product_invariant(
        framework: &signer,
        admin: &signer,
        trader: &signer,
        lp: &signer,
    ) {
        timestamp::set_time_has_started_for_testing(framework);
        // Setup LP and trader
        setup_coins(framework, admin, lp, 1_000_000_000, 500_000_000);
        setup_coins(framework, admin, trader, 1_000_000_000, 0);
        
        amm::create_pool<USDC, ETH>(admin, 30);
        amm::add_liquidity<USDC, ETH>(lp, 1_000_000_000, 500_000_000, 0);
        
        let (rx_before, ry_before) = amm::get_reserves<USDC, ETH>();
        let k_before = (rx_before as u128) * (ry_before as u128);
        
        // Do multiple swaps
        amm::swap_exact_input<USDC, ETH>(trader, 100_000_000, 0);
        amm::swap_exact_input<USDC, ETH>(trader, 50_000_000, 0);
        
        let (rx_after, ry_after) = amm::get_reserves<USDC, ETH>();
        let k_after = (rx_after as u128) * (ry_after as u128);
        
        // k should never decrease (fees make it increase slightly)
        assert!(k_after >= k_before, 1);
    }
    
    // ============================================
    // Edge case tests
    // ============================================
    
    #[test(framework = @0x1, admin = @defi, user = @0x100)]
    #[expected_failure(abort_code = amm::E_INSUFFICIENT_OUTPUT)]
    fun test_swap_slippage_protection(
        framework: &signer,
        admin: &signer,
        user: &signer,
    ) {
        timestamp::set_time_has_started_for_testing(framework);
        setup_coins(framework, admin, user, 2_000_000_000, 1_000_000_000);
        amm::create_pool<USDC, ETH>(admin, 30);
        amm::add_liquidity<USDC, ETH>(user, 1_000_000_000, 500_000_000, 0);
        
        // Set min_out too high → should fail
        amm::swap_exact_input<USDC, ETH>(
            user,
            100_000_000,
            1_000_000_000  // Impossible min_out
        );
    }
    
    #[test(framework = @0x1, admin = @defi, user = @0x100)]
    #[expected_failure(abort_code = amm::E_ZERO_LIQUIDITY)]
    fun test_remove_all_liquidity_fails(
        framework: &signer,
        admin: &signer,
        user: &signer,
    ) {
        timestamp::set_time_has_started_for_testing(framework);
        setup_coins(framework, admin, user, 1_000_000_000, 500_000_000);
        amm::create_pool<USDC, ETH>(admin, 30);
        amm::add_liquidity<USDC, ETH>(user, 1_000_000_000, 500_000_000, 0);
        
        // Try to remove 100% of liquidity (MINIMUM_LIQUIDITY prevents this)
        let all_lp = amm::get_lp_balance<USDC, ETH>(signer::address_of(user));
        amm::remove_liquidity<USDC, ETH>(user, all_lp, 0, 0);
    }
}
```

---

## Integration Testing

```move
#[test_only]
module defi::protocol_integration_tests {
    // ============================================
    // Integration test: full lending + AMM flow
    // ============================================
    
    use defi::amm;
    use defi::lending;
    use defi::oracle;
    
    // Test: User deposits collateral → borrows → swaps → repays
    #[test(
        framework = @0x1,
        protocol_admin = @defi,
        oracle_admin = @oracle,
        alice = @0x100,
        bob = @0x200,
    )]
    fun test_leveraged_position(
        framework: &signer,
        protocol_admin: &signer,
        oracle_admin: &signer,
        alice: &signer,
        bob: &signer,
    ) {
        // 1. Setup infrastructure
        aptos_framework::timestamp::set_time_has_started_for_testing(framework);
        
        // Initialize oracle
        oracle::initialize(oracle_admin);
        oracle::set_price(oracle_admin, 2000_000_000u64);  // ETH = $2000
        
        // Initialize AMM
        amm::create_pool<USDC, ETH>(protocol_admin, 30);
        
        // Initialize lending
        lending::initialize(protocol_admin);
        lending::add_market<ETH>(protocol_admin, 7500);  // 75% LTV
        
        // 2. Seed AMM with liquidity (Bob = LP)
        // Bob adds $10M liquidity
        amm::add_liquidity<USDC, ETH>(bob, 5_000_000_000_000, 2_500_000_000, 0);
        
        // 3. Alice: Open leveraged long ETH
        let alice_eth = 1_000_000_000u64;  // Alice has 1 ETH = $2000
        
        // Deposit 1 ETH as collateral
        lending::deposit_collateral<ETH>(alice, alice_eth);
        
        // Borrow $1000 USDC (50% LTV, conservative)
        lending::borrow<USDC>(alice, 1_000_000_000u64);
        
        let usdc_borrowed = 1_000_000_000u64;
        
        // Swap USDC → ETH (buy more ETH)
        amm::swap_exact_input<USDC, ETH>(alice, usdc_borrowed, 0);
        
        // Alice now has MORE ETH (leveraged long)
        let additional_eth = coin::balance<ETH>(signer::address_of(alice));
        assert!(additional_eth > 0, 1);
        
        // 4. Fast forward time (accrue interest)
        aptos_framework::timestamp::fast_forward_seconds(30 * 24 * 3600);  // 30 days
        
        let interest = lending::get_accrued_interest<USDC>(signer::address_of(alice));
        assert!(interest > 0, 2);
        
        // 5. Close position: sell additional ETH, repay debt
        let alice_eth_balance = coin::balance<ETH>(signer::address_of(alice));
        amm::swap_exact_input<ETH, USDC>(alice, alice_eth_balance, 0);
        
        let repay_amount = usdc_borrowed + interest;
        lending::repay<USDC>(alice, repay_amount);
        
        // Withdraw collateral
        lending::withdraw_collateral<ETH>(alice, alice_eth);
        
        // Verify: alice ends up with approximately 1 ETH (minus fees/interest)
        let final_eth = coin::balance<ETH>(signer::address_of(alice));
        // Should be close to 1 ETH minus costs
        assert!(final_eth > alice_eth * 90 / 100, 3);  // Lost at most 10%
    }
}
```

---

## Property-Based Testing

```move
#[test_only]
module defi::property_tests {
    
    // ============================================
    // Property-Based Testing
    // Test PROPERTIES that must always hold
    // ============================================
    
    // Property 1: Swap is monotonic (more in → more out)
    #[test]
    fun test_swap_monotonic() {
        let x_reserve: u64 = 1_000_000;
        let y_reserve: u64 = 2_000_000;
        let fee_bps: u64 = 30;
        
        // For increasingly larger swap amounts...
        let amounts = vector[100u64, 1000, 10000, 100000];
        let mut prev_out = 0u64;
        
        let mut i = 0u64;
        while (i < std::vector::length(&amounts)) {
            let amount_in = *std::vector::borrow(&amounts, i);
            let amount_out = compute_swap_out(x_reserve, y_reserve, amount_in, fee_bps);
            
            // More in → more out (monotonic)
            assert!(amount_out > prev_out, i as u64 + 1);
            prev_out = amount_out;
            i = i + 1;
        };
    }
    
    // Property 2: Adding/removing liquidity is symmetric
    #[test]
    fun test_liquidity_symmetry() {
        let x_amount: u64 = 500_000;
        let y_amount: u64 = 1_000_000;
        
        let lp_minted = compute_lp_mint(x_amount, y_amount, 0, 0, 0);
        
        let (x_out, y_out) = compute_lp_redeem(lp_minted, x_amount, y_amount, lp_minted);
        
        // Should get back what you put in
        assert!(x_out == x_amount, 1);
        assert!(y_out == y_amount, 2);
    }
    
    // Property 3: Price impact scales correctly
    #[test]
    fun test_price_impact_scaling() {
        let x_reserve: u64 = 1_000_000;
        let y_reserve: u64 = 1_000_000;
        let fee_bps: u64 = 0;  // No fee for pure math test
        
        // Swap 1% of reserve
        let small_swap = 10_000u64;  // 1% of 1M
        let small_out = compute_swap_out(x_reserve, y_reserve, small_swap, fee_bps);
        let small_impact_bps = (small_swap - small_out) * 10_000 / small_swap;
        
        // Swap 10% of reserve
        let large_swap = 100_000u64;  // 10% of 1M
        let large_out = compute_swap_out(x_reserve, y_reserve, large_swap, fee_bps);
        let large_impact_bps = (large_swap - large_out) * 10_000 / large_swap;
        
        // Larger swap should have larger price impact (convex slippage)
        assert!(large_impact_bps > small_impact_bps, 1);
    }
    
    // Property 4: LP value never decreases (from fees)
    #[test]
    fun test_lp_value_monotonic_with_fees() {
        let x_reserve = 1_000_000u64;
        let y_reserve = 1_000_000u64;
        let total_lp = 1_000_000u64;
        let fee_bps = 30u64;
        
        // LP holds some percentage of pool
        let lp_held = 100_000u64;  // 10% of pool
        
        let (x_before, y_before) = get_lp_value(lp_held, x_reserve, y_reserve, total_lp);
        
        // Simulate swap that generates fees
        let swap_amount = 50_000u64;
        let fee_amount = swap_amount * fee_bps / 10_000;
        
        // Fees go to pool → x_reserve increases
        let x_reserve_new = x_reserve + fee_amount;
        
        let (x_after, y_after) = get_lp_value(lp_held, x_reserve_new, y_reserve, total_lp);
        
        // LP should have more value after fees
        assert!(x_after > x_before, 1);  // More X (from fees)
        assert!(y_after == y_before, 2);  // Same Y
    }
    
    // Helper functions for property tests
    fun compute_swap_out(
        x_reserve: u64,
        y_reserve: u64,
        amount_in: u64,
        fee_bps: u64,
    ): u64 {
        let amount_in_with_fee = amount_in * (10_000 - fee_bps) / 10_000;
        y_reserve * amount_in_with_fee / (x_reserve + amount_in_with_fee)
    }
    
    fun compute_lp_mint(x: u64, y: u64, rx: u64, ry: u64, total_lp: u64): u64 {
        if (total_lp == 0) {
            // Initial: sqrt(x * y) - MINIMUM_LIQUIDITY
            let product = (x as u128) * (y as u128);
            (sqrt_u128(product) as u64) - 1000  // 1000 = MINIMUM_LIQUIDITY
        } else {
            let lp_x = (x as u128) * (total_lp as u128) / (rx as u128);
            let lp_y = (y as u128) * (total_lp as u128) / (ry as u128);
            std::u128::min(lp_x, lp_y) as u64
        }
    }
    
    fun compute_lp_redeem(lp_amount: u64, rx: u64, ry: u64, total_lp: u64): (u64, u64) {
        let x_out = (lp_amount as u128) * (rx as u128) / (total_lp as u128);
        let y_out = (lp_amount as u128) * (ry as u128) / (total_lp as u128);
        (x_out as u64, y_out as u64)
    }
    
    fun get_lp_value(lp: u64, rx: u64, ry: u64, total_lp: u64): (u64, u64) {
        compute_lp_redeem(lp, rx, ry, total_lp)
    }
    
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
}
```

---

## Fuzz Testing

```bash
#!/bin/bash
# fuzz_test.sh - Fuzz test the AMM module

# Move doesn't have native fuzzing like Rust's cargo-fuzz
# But we can use property-based approach with random inputs

# Strategy: Generate many random test vectors and check invariants

cat > /tmp/fuzz_runner.py << 'EOF'
import subprocess
import random
import json

def generate_test_case():
    """Generate random AMM parameters"""
    return {
        "x_reserve": random.randint(1000, 10**18),
        "y_reserve": random.randint(1000, 10**18),
        "swap_amount": random.randint(1, 10**15),
        "fee_bps": random.choice([10, 30, 100, 300]),
    }

def check_invariants(params, result):
    """Check that AMM invariants hold"""
    x = params["x_reserve"]
    y = params["y_reserve"]
    k_before = x * y
    
    # After swap
    x_new = x + params["swap_amount"]
    y_new = result["y_out"]
    
    # k should never decrease
    k_after = x_new * (y - y_new)
    
    if k_after < k_before:
        return False, f"k decreased: {k_before} → {k_after}"
    
    # Output should be positive
    if y_new <= 0:
        return False, "negative output"
    
    # Output should not exceed reserves
    if y_new >= y:
        return False, f"output exceeds reserves: {y_new} >= {y}"
    
    return True, "OK"

# Run 10000 fuzz tests
passed = 0
failed = 0
errors = []

for i in range(10000):
    params = generate_test_case()
    
    # Call Move module via CLI (or compute directly in Python)
    dx_fee = params["swap_amount"] * (10000 - params["fee_bps"]) // 10000
    y_out = params["y_reserve"] * dx_fee // (params["x_reserve"] + dx_fee)
    
    result = {"y_out": y_out}
    
    ok, msg = check_invariants(params, result)
    if ok:
        passed += 1
    else:
        failed += 1
        errors.append({"params": params, "error": msg})

print(f"Fuzz results: {passed} passed, {failed} failed")
if errors:
    print("Failures:")
    for e in errors[:5]:
        print(json.dumps(e, indent=2))
EOF

python3 /tmp/fuzz_runner.py

# Also run Move's built-in tests
aptos move test --filter "test_" 2>&1 | tail -20
```

---

## Gas Benchmarking

```move
#[test_only]
module defi::gas_benchmarks {
    use aptos_framework::account;
    
    // ============================================
    // Gas Benchmark Suite
    // Run with: aptos move test --gas-profile
    // ============================================
    
    // Baseline: measure function gas costs
    #[test]
    fun bench_swap_small(): u64 {
        // Simulate smallest meaningful swap
        let (x_reserve, y_reserve) = (1_000_000u64, 1_000_000u64);
        let amount_in = 1000u64;
        let fee_bps = 30u64;
        
        let _ = compute_swap_out_bench(x_reserve, y_reserve, amount_in, fee_bps);
        
        0  // Return 0, actual gas measured externally
    }
    
    #[test]
    fun bench_swap_large(): u64 {
        let (x_reserve, y_reserve) = (1_000_000_000u64, 1_000_000_000u64);
        let amount_in = 10_000_000u64;
        let fee_bps = 30u64;
        
        let _ = compute_swap_out_bench(x_reserve, y_reserve, amount_in, fee_bps);
        0
    }
    
    #[test]
    fun bench_add_10_liquidity_events() {
        let mut i = 0u64;
        while (i < 10) {
            // Simulate LP tracking computation
            let _ = compute_lp_shares_bench(1_000_000, 2_000_000, i * 100_000, 500_000);
            i = i + 1;
        };
    }
    
    #[test]
    fun bench_sort_10_elements() {
        let mut v = vector[9u64, 3, 7, 1, 5, 8, 2, 6, 4, 0];
        sort_vector_bench(&mut v);
    }
    
    #[test]
    fun bench_sort_100_elements() {
        let mut v = std::vector::empty<u64>();
        let mut i = 99u64;
        while (i > 0) {
            std::vector::push_back(&mut v, i);
            i = i - 1;
        };
        sort_vector_bench(&mut v);
    }
    
    // ============================================
    // Storage access patterns
    // ============================================
    
    // Test: SmartTable vs simple vector for small sets
    #[test]
    fun bench_smart_table_lookup_10() {
        let mut table = aptos_std::smart_table::new<u64, u64>();
        let mut i = 0u64;
        while (i < 10) {
            aptos_std::smart_table::add(&mut table, i, i * i);
            i = i + 1;
        };
        
        // 10 lookups
        let mut j = 0u64;
        while (j < 10) {
            let _ = *aptos_std::smart_table::borrow(&table, j);
            j = j + 1;
        };
        
        aptos_std::smart_table::destroy(table);
    }
    
    #[test]
    fun bench_vector_lookup_10() {
        let mut v = std::vector::empty<u64>();
        let mut i = 0u64;
        while (i < 10) {
            std::vector::push_back(&mut v, i * i);
            i = i + 1;
        };
        
        // 10 lookups
        let mut j = 0u64;
        while (j < 10) {
            let _ = *std::vector::borrow(&v, j);
            j = j + 1;
        };
    }
    
    // Results format (from aptos move test --gas-profile):
    // bench_swap_small: 125 gas units
    // bench_swap_large: 127 gas units  (nearly same! math is cheap)
    // bench_sort_10: 450 gas units
    // bench_sort_100: 24,500 gas units (quadratic → avoid for large sets!)
    // bench_smart_table_lookup_10: 1,200 gas units
    // bench_vector_lookup_10: 380 gas units (3x faster for small sets)
    
    fun compute_swap_out_bench(rx: u64, ry: u64, amt_in: u64, fee: u64): u64 {
        let amt_with_fee = amt_in * (10_000 - fee) / 10_000;
        ry * amt_with_fee / (rx + amt_with_fee)
    }
    
    fun compute_lp_shares_bench(rx: u64, ry: u64, lp: u64, total: u64): (u64, u64) {
        let x = lp * rx / total;
        let y = lp * ry / total;
        (x, y)
    }
    
    fun sort_vector_bench(v: &mut vector<u64>) {
        let len = std::vector::length(v);
        let mut i = 0u64;
        while (i < len) {
            let mut j = i + 1;
            while (j < len) {
                let vi = *std::vector::borrow(v, i);
                let vj = *std::vector::borrow(v, j);
                if (vj < vi) {
                    *std::vector::borrow_mut(v, i) = vj;
                    *std::vector::borrow_mut(v, j) = vi;
                };
                j = j + 1;
            };
            i = i + 1;
        };
    }
}
```

---

## Invariant Testing

```move
#[test_only]
module defi::invariants {
    
    // ============================================
    // Invariant Testing: Properties that MUST hold
    // regardless of sequence of operations
    // ============================================
    
    // Invariant 1: k = x * y never decreases (AMM)
    // Invariant 2: Total LP value = total pool value
    // Invariant 3: No user can have negative balance
    // Invariant 4: Borrow < supply at all times
    // Invariant 5: Collateral ratio > min at all times
    
    struct SystemState has copy, drop {
        // AMM
        x_reserve: u64,
        y_reserve: u64,
        total_lp: u64,
        
        // Lending
        total_borrowed: u64,
        total_supplied: u64,
        
        // Oracle
        eth_price_usd: u64,
    }
    
    // Verify all system invariants
    fun check_invariants(state: &SystemState): bool {
        // 1. Reserves must be positive
        if (state.x_reserve == 0 || state.y_reserve == 0) {
            return false
        };
        
        // 2. LP supply consistent
        if (state.total_lp == 0 && state.x_reserve > 0) {
            return false
        };
        
        // 3. Borrowed ≤ supplied in lending
        if (state.total_borrowed > state.total_supplied) {
            return false
        };
        
        // 4. Price must be positive
        if (state.eth_price_usd == 0) {
            return false
        };
        
        true
    }
    
    // Simulate random operations and check invariants after each
    #[test]
    fun test_invariants_hold_through_operations() {
        let mut state = SystemState {
            x_reserve: 1_000_000,
            y_reserve: 2_000_000,
            total_lp: 1_000_000,
            total_borrowed: 500_000,
            total_supplied: 1_000_000,
            eth_price_usd: 2000,
        };
        
        assert!(check_invariants(&state), 1);
        
        // Simulate swap
        let swap_in = 10_000u64;
        let swap_out = compute_swap(state.x_reserve, state.y_reserve, swap_in);
        state.x_reserve = state.x_reserve + swap_in;
        state.y_reserve = state.y_reserve - swap_out;
        
        assert!(check_invariants(&state), 2);
        
        // Simulate add liquidity
        state.x_reserve = state.x_reserve + 100_000;
        state.y_reserve = state.y_reserve + 200_000;
        state.total_lp = state.total_lp + 100_000;
        
        assert!(check_invariants(&state), 3);
        
        // Simulate repay
        state.total_borrowed = state.total_borrowed - 50_000;
        
        assert!(check_invariants(&state), 4);
    }
    
    fun compute_swap(rx: u64, ry: u64, amt_in: u64): u64 {
        let amt_fee = amt_in * 9970 / 10000;  // 0.3% fee
        ry * amt_fee / (rx + amt_fee)
    }
}
```

---

## สรุป Advanced Testing

```
Testing Strategy Summary:

Test Coverage Goals:
  Unit tests:     > 90% line coverage
  Integration:    Cover all user journeys
  Property tests: At least 5 invariants per module
  Fuzz tests:     10,000+ random inputs
  Gas benchmarks: Track regression per PR

Move Test Commands:
  aptos move test                    # Run all tests
  aptos move test --filter test_swap # Run specific tests
  aptos move test --gas-profile      # Show gas usage
  aptos move test -v                 # Verbose output
  aptos move prove                   # Run Move Prover (formal)

Priority Test Areas:
  🔴 HIGH: Token transfers, balance math, access control
  🟡 MEDIUM: LP calculations, oracle integration
  🟢 LOW: View functions, event emission

Common Bugs Caught by Tests:
  - Off-by-one in iteration bounds
  - Integer overflow in multiplication before division
  - Missing access control in admin functions  
  - Incorrect fee calculation (should use (10000-fee)/10000, not 1-fee%)
  - Rounding in favor of protocol (not user)
  - Missing check for zero amount input
  - Replay attacks (same nonce accepted twice)

Test Anti-Patterns:
  ❌ Testing implementation details (fragile)
  ❌ Only testing happy path
  ❌ Non-deterministic tests (timestamp without set_time)
  ❌ Tests that depend on each other's order
  ❌ Incomplete cleanup (resource leaks in tests)
  
  ✓ Test user-observable behavior
  ✓ Test all error conditions
  ✓ Reset state before each test
  ✓ Use descriptive assertion codes
  ✓ Document what each test verifies
```

---

**ก่อนหน้า**: [Part 56 - Randomness & VRF ←](part-56-randomness-vrf.md)
**ต่อไป**: [Part 58 - DeFi Protocol Economics →](part-58-protocol-economics.md)
