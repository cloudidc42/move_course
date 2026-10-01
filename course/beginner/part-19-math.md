# Part 19: Math and Fixed-Point Arithmetic

## สารบัญ
- [Integer Math](#integer-math)
- [Fixed-Point Numbers](#fixed-point-numbers)
- [Financial Math](#financial-math)
- [DeFi Formulas](#defi-formulas)
- [ตัวอย่างโปรแกรม: Pricing Oracle](#ตัวอย่างโปรแกรม-pricing-oracle)

---

## Integer Math

```move
module learning::integer_math {
    
    // ============================================
    // Basic operations
    // ============================================
    
    // Move integer types: u8, u16, u32, u64, u128, u256
    // All arithmetic operations abort on overflow/underflow
    
    public fun safe_operations_demo() {
        let a: u64 = 1_000_000;
        let b: u64 = 500_000;
        
        let sum = a + b;       // 1,500,000
        let diff = a - b;      // 500,000
        let product = a * b;   // 500,000,000,000
        let quotient = a / b;  // 2
        let remainder = a % b; // 0
        
        let _ = (sum, diff, product, quotient, remainder);
    }
    
    // ============================================
    // Overflow-safe math
    // ============================================
    
    const U64_MAX: u64 = 18446744073709551615u64;
    const U128_MAX: u128 = 340282366920938463463374607431768211455u128;
    
    public fun checked_add(a: u64, b: u64): (bool, u64) {
        if (a > U64_MAX - b) {
            (false, 0)  // would overflow
        } else {
            (true, a + b)
        }
    }
    
    public fun checked_mul(a: u64, b: u64): (bool, u64) {
        if (b == 0) return (true, 0);
        if (a > U64_MAX / b) {
            (false, 0)
        } else {
            (true, a * b)
        }
    }
    
    // Use u128 for intermediate calculations to prevent overflow
    public fun mul_div(a: u64, b: u64, c: u64): u64 {
        assert!(c != 0, 1);
        let result = (a as u128) * (b as u128) / (c as u128);
        assert!(result <= (U64_MAX as u128), 2);
        result as u64
    }
    
    // ============================================
    // Integer sqrt
    // ============================================
    
    public fun sqrt(n: u64): u64 {
        if (n == 0) return 0;
        
        let mut x = n;
        let mut y = (x + 1) / 2;
        while (y < x) {
            x = y;
            y = (x + n / x) / 2;
        };
        x
    }
    
    public fun sqrt_128(n: u128): u128 {
        if (n == 0) return 0;
        
        let mut x = n;
        let mut y = (x + 1) / 2;
        while (y < x) {
            x = y;
            y = (x + n / x) / 2;
        };
        x
    }
    
    // ============================================
    // Bit operations
    // ============================================
    
    public fun is_power_of_two(n: u64): bool {
        n > 0 && (n & (n - 1)) == 0
    }
    
    public fun next_power_of_two(n: u64): u64 {
        if (n == 0) return 1;
        if (is_power_of_two(n)) return n;
        
        let mut result = 1u64;
        while (result < n) {
            result = result << 1;
        };
        result
    }
    
    public fun count_bits(n: u64): u64 {
        let mut count = 0u64;
        let mut x = n;
        while (x > 0) {
            count = count + (x & 1);
            x = x >> 1;
        };
        count
    }
    
    // ============================================
    // GCD and LCM
    // ============================================
    
    public fun gcd(a: u64, b: u64): u64 {
        if (b == 0) return a;
        gcd(b, a % b)
    }
    
    public fun lcm(a: u64, b: u64): u64 {
        if (a == 0 || b == 0) return 0;
        a / gcd(a, b) * b
    }
    
    // ============================================
    // Power functions
    // ============================================
    
    public fun pow(base: u64, exp: u64): u64 {
        if (exp == 0) return 1;
        
        let mut result = 1u64;
        let mut base_mut = base;
        let mut exp_mut = exp;
        
        while (exp_mut > 0) {
            if (exp_mut & 1 == 1) {
                result = result * base_mut;
            };
            base_mut = base_mut * base_mut;
            exp_mut = exp_mut >> 1;
        };
        result
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test]
    public fun test_math() {
        assert!(sqrt(16) == 4, 0);
        assert!(sqrt(25) == 5, 1);
        assert!(sqrt(2) == 1, 2);  // floor
        
        assert!(gcd(12, 8) == 4, 3);
        assert!(lcm(4, 6) == 12, 4);
        
        assert!(pow(2, 10) == 1024, 5);
        assert!(pow(3, 3) == 27, 6);
        
        assert!(is_power_of_two(16), 7);
        assert!(!is_power_of_two(12), 8);
        assert!(next_power_of_two(5) == 8, 9);
    }
}
```

---

## Fixed-Point Numbers

```move
module learning::fixed_point {
    
    // ============================================
    // Fixed-point representation
    // ============================================
    
    // Instead of floating point (not in Move), use scaled integers
    // 
    // Common scales:
    // - 1e6 (6 decimals) - like USDC
    // - 1e8 (8 decimals) - like BTC/APT
    // - 1e18 (18 decimals) - like ETH/EVM
    // - WAD = 1e18 (standard in DeFi)
    // - RAY = 1e27 (high precision in Aave-style)
    
    const PRECISION: u64 = 1_000_000_000_000_000_000;  // 1e18 (WAD)
    const HALF_PRECISION: u64 = 500_000_000_000_000_000;  // 0.5 WAD
    
    // Fixed-point type
    struct FixedPoint {
        value: u128,  // stored as value * PRECISION
        scale: u64,   // the precision (e.g., 1e18)
    }
    
    // ============================================
    // Fixed-point arithmetic with WAD (1e18)
    // ============================================
    
    const WAD: u128 = 1_000_000_000_000_000_000;  // 1e18
    const HALF_WAD: u128 = 500_000_000_000_000_000;  // 0.5e18
    
    // Multiply: (a * b) / WAD
    public fun wad_mul(a: u128, b: u128): u128 {
        // Add HALF_WAD for rounding
        (a * b + HALF_WAD) / WAD
    }
    
    // Divide: (a * WAD) / b
    public fun wad_div(a: u128, b: u128): u128 {
        assert!(b != 0, 1);
        // Add b/2 for rounding
        (a * WAD + b / 2) / b
    }
    
    // Convert integer to WAD
    public fun to_wad(n: u64): u128 {
        (n as u128) * WAD
    }
    
    // Convert WAD to integer (floor)
    public fun from_wad(wad: u128): u64 {
        (wad / WAD) as u64
    }
    
    // Convert WAD to integer (round)
    public fun from_wad_rounded(wad: u128): u64 {
        ((wad + HALF_WAD) / WAD) as u64
    }
    
    // ============================================
    // RAY arithmetic (1e27)
    // ============================================
    
    const RAY: u128 = 1_000_000_000_000_000_000_000_000_000;  // 1e27
    const HALF_RAY: u128 = 500_000_000_000_000_000_000_000_000;
    
    public fun ray_mul(a: u128, b: u128): u128 {
        (a * b + HALF_RAY) / RAY
    }
    
    public fun ray_div(a: u128, b: u128): u128 {
        assert!(b != 0, 1);
        (a * RAY + b / 2) / b
    }
    
    public fun ray_to_wad(ray: u128): u128 {
        let wad_ray_ratio = RAY / WAD;  // 1e9
        (ray + wad_ray_ratio / 2) / wad_ray_ratio
    }
    
    public fun wad_to_ray(wad: u128): u128 {
        wad * (RAY / WAD)
    }
    
    // ============================================
    // Basis Points (BPS)
    // ============================================
    
    const BPS_BASE: u64 = 10_000;  // 1 = 0.01%, 10000 = 100%
    
    public fun apply_bps(amount: u64, bps: u64): u64 {
        (amount as u128 * bps as u128 / BPS_BASE as u128) as u64
    }
    
    public fun bps_to_wad(bps: u64): u128 {
        // 10000 bps = 1.0 WAD
        (bps as u128) * WAD / (BPS_BASE as u128)
    }
    
    // ============================================
    // Percentage calculations
    // ============================================
    
    public fun percentage_of(amount: u64, percent: u64, precision: u64): u64 {
        // percent is in precision units (e.g., precision=100 means percent is 0-100)
        (amount as u128 * percent as u128 / precision as u128) as u64
    }
    
    public fun inverse_percentage(result: u64, percent: u64, precision: u64): u64 {
        // Find original amount given result and percentage
        assert!(percent != 0, 1);
        (result as u128 * precision as u128 / percent as u128) as u64
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test]
    public fun test_wad_math() {
        let one_wad = WAD;
        let half_wad = HALF_WAD;
        
        // 1.5 * 2 = 3
        let one_and_half = one_wad + half_wad;
        let two = one_wad * 2;
        assert!(wad_mul(one_and_half, two) == 3 * one_wad, 0);
        
        // 1 / 4 = 0.25
        let quarter = wad_div(one_wad, 4 * one_wad);
        assert!(quarter == one_wad / 4, 1);
    }
    
    #[test]
    public fun test_bps() {
        // 100 bps = 1%
        let fee = apply_bps(10_000, 100);
        assert!(fee == 100, 0);
        
        // 3000 bps = 30%
        let fee2 = apply_bps(1_000, 3_000);
        assert!(fee2 == 300, 1);
    }
}
```

---

## Financial Math

```move
module learning::financial_math {
    
    const WAD: u128 = 1_000_000_000_000_000_000;
    
    // ============================================
    // Interest calculations
    // ============================================
    
    // Simple interest: P * r * t
    public fun simple_interest(
        principal: u64,
        rate_wad: u128,  // annual rate in WAD (1e18 = 100%)
        time_seconds: u64,
        seconds_per_year: u64,
    ): u64 {
        let p = principal as u128;
        let t = time_seconds as u128;
        let year = seconds_per_year as u128;
        
        let interest = p * rate_wad * t / year / WAD;
        interest as u64
    }
    
    // Compound interest approximation (using Taylor expansion)
    // For small rates: (1 + r)^t ≈ 1 + r*t + r²*t*(t-1)/2
    public fun compound_interest_approx(
        principal: u64,
        rate_per_period_wad: u128,  // rate per period in WAD
        periods: u64,
    ): u64 {
        let p = principal as u128;
        let r = rate_per_period_wad;
        let n = periods as u128;
        
        // First order: P * r * n
        let first_order = p * r * n / WAD;
        
        // Second order: P * r² * n * (n-1) / 2
        let second_order = if (n > 1) {
            p * r / WAD * r / WAD * n * (n - 1) / 2
        } else {
            0
        };
        
        let total_interest = first_order + second_order;
        (p + total_interest) as u64
    }
    
    // ============================================
    // Liquidation math
    // ============================================
    
    // Health factor = (collateral * collateral_factor) / debt
    // Health factor < 1 = liquidatable
    public fun health_factor(
        collateral_value: u64,
        collateral_factor_bps: u64,  // e.g., 7500 = 75%
        debt_value: u64,
    ): u128 {  // returns in WAD units
        if (debt_value == 0) return WAD * 100;  // infinite health
        
        let collateral_128 = collateral_value as u128;
        let factor_128 = collateral_factor_bps as u128;
        let debt_128 = debt_value as u128;
        
        collateral_128 * factor_128 * WAD / 10_000 / debt_128
    }
    
    public fun is_liquidatable(
        collateral_value: u64,
        collateral_factor_bps: u64,
        debt_value: u64,
    ): bool {
        health_factor(collateral_value, collateral_factor_bps, debt_value) < WAD
    }
    
    // Maximum borrowable amount
    public fun max_borrow(
        collateral_value: u64,
        collateral_factor_bps: u64,
        current_debt: u64,
    ): u64 {
        let max_debt = (collateral_value as u128) * (collateral_factor_bps as u128) / 10_000;
        if (max_debt <= current_debt as u128) { return 0 };
        (max_debt - current_debt as u128) as u64
    }
    
    // ============================================
    // Price impact calculation
    // ============================================
    
    // For AMM with x*y=k:
    // Price impact = (amount_out / reserve_out) before vs after
    public fun price_impact_bps(
        reserve_in: u64,
        reserve_out: u64,
        amount_in: u64,
    ): u64 {
        let new_reserve_in = reserve_in + amount_in;
        let amount_out = reserve_in * reserve_out / new_reserve_in;
        let new_reserve_out = reserve_out - amount_out;
        
        // Price before = reserve_out / reserve_in
        // Price after = new_reserve_out / new_reserve_in
        // Impact = (price_before - price_after) / price_before
        
        let price_before = (reserve_out as u128) * 10_000 / (reserve_in as u128);
        let price_after = (new_reserve_out as u128) * 10_000 / (new_reserve_in as u128);
        
        if (price_before <= price_after) { return 0 };
        ((price_before - price_after) * 10_000 / price_before) as u64
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test]
    public fun test_health_factor() {
        // 100 ETH collateral, 75% factor, 50 ETH debt
        // HF = 100 * 75% / 50 = 1.5
        let hf = health_factor(100, 7_500, 50);
        assert!(hf == WAD * 3 / 2, 0);  // 1.5 WAD
        
        // Not liquidatable
        assert!(!is_liquidatable(100, 7_500, 50), 1);
        
        // 100 ETH collateral, 75% factor, 80 ETH debt
        // HF = 100 * 75% / 80 = 0.9375 < 1
        assert!(is_liquidatable(100, 7_500, 80), 2);
    }
    
    #[test]
    public fun test_max_borrow() {
        // 1000 USDC collateral, 75% factor, 0 debt
        let max = max_borrow(1_000, 7_500, 0);
        assert!(max == 750, 0);  // can borrow 750 USDC
        
        // 1000 USDC collateral, 75% factor, 300 debt
        let max2 = max_borrow(1_000, 7_500, 300);
        assert!(max2 == 450, 1);  // 750 - 300 = 450
    }
}
```

---

## DeFi Formulas

```move
module learning::defi_math {
    
    // ============================================
    // AMM: Constant Product (x*y=k)
    // ============================================
    
    // Get output amount for given input (with fee)
    public fun get_amount_out(
        amount_in: u64,
        reserve_in: u64,
        reserve_out: u64,
        fee_bps: u64,
    ): u64 {
        assert!(amount_in > 0, 1);
        assert!(reserve_in > 0 && reserve_out > 0, 2);
        
        let fee_factor = 10_000 - fee_bps;  // e.g., 9970 for 0.3% fee
        
        let amount_in_with_fee = (amount_in as u128) * (fee_factor as u128);
        let numerator = amount_in_with_fee * (reserve_out as u128);
        let denominator = (reserve_in as u128) * 10_000 + amount_in_with_fee;
        
        (numerator / denominator) as u64
    }
    
    // Get required input for desired output
    public fun get_amount_in(
        amount_out: u64,
        reserve_in: u64,
        reserve_out: u64,
        fee_bps: u64,
    ): u64 {
        assert!(amount_out > 0 && amount_out < reserve_out, 1);
        
        let fee_factor = 10_000 - fee_bps;
        
        let numerator = (reserve_in as u128) * (amount_out as u128) * 10_000;
        let denominator = ((reserve_out as u128) - (amount_out as u128)) * (fee_factor as u128);
        
        ((numerator + denominator - 1) / denominator) as u64  // ceiling
    }
    
    // Calculate LP tokens to mint for adding liquidity
    public fun calc_lp_mint(
        amount_a: u64,
        amount_b: u64,
        reserve_a: u64,
        reserve_b: u64,
        lp_total: u64,
    ): u64 {
        if (lp_total == 0) {
            // Initial liquidity: geometric mean
            let product = (amount_a as u128) * (amount_b as u128);
            let lp = sqrt_128(product);
            lp as u64
        } else {
            // Proportional
            let lp_a = (amount_a as u128) * (lp_total as u128) / (reserve_a as u128);
            let lp_b = (amount_b as u128) * (lp_total as u128) / (reserve_b as u128);
            let min_lp = if (lp_a < lp_b) { lp_a } else { lp_b };
            min_lp as u64
        }
    }
    
    // ============================================
    // Stablecoin AMM: Curve-style
    // ============================================
    
    // Simplified StableSwap: better for same-peg assets
    // x + y = D (simple version, actual Curve uses more complex)
    
    // For stableswap, price is close to 1:1 with small slippage
    public fun stableswap_get_out(
        amount_in: u64,
        reserve_in: u64,
        reserve_out: u64,
        amplifier: u64,  // A parameter, e.g., 100
        fee_bps: u64,
    ): u64 {
        // Simplified version - actual Curve uses Newton's method
        let fee_factor = 10_000 - fee_bps;
        
        // Higher amplifier = less slippage = closer to stablecoin behavior
        let effective_reserve_in = reserve_in * amplifier;
        let effective_reserve_out = reserve_out * amplifier;
        
        let amount_in_adj = amount_in * fee_factor / 10_000;
        let out = amount_in_adj * effective_reserve_out / 
                  (effective_reserve_in + amount_in_adj * amplifier);
        
        out
    }
    
    // ============================================
    // Lending: Interest rate model
    // ============================================
    
    // Utilization rate = borrows / deposits
    public fun utilization_rate(
        total_borrows: u64,
        total_deposits: u64,
    ): u64 {  // returns bps (0-10000)
        if (total_deposits == 0) return 0;
        (total_borrows as u128 * 10_000 / total_deposits as u128) as u64
    }
    
    // Jump rate model borrow APR (bps)
    public fun borrow_rate_bps(
        utilization_bps: u64,
        base_rate_bps: u64,       // e.g., 100 (1%)
        multiplier_bps: u64,      // e.g., 500 (5%)
        jump_multiplier_bps: u64, // e.g., 10000 (100%)
        kink_bps: u64,            // e.g., 8000 (80% utilization)
    ): u64 {
        if (utilization_bps <= kink_bps) {
            // Normal zone: base + multiplier * util
            base_rate_bps + multiplier_bps * utilization_bps / 10_000
        } else {
            // Jump zone: kink rate + jump_multiplier * (util - kink)
            let kink_rate = base_rate_bps + multiplier_bps * kink_bps / 10_000;
            let excess_util = utilization_bps - kink_bps;
            kink_rate + jump_multiplier_bps * excess_util / 10_000
        }
    }
    
    // Supply APR = borrow_rate * utilization
    public fun supply_rate_bps(
        borrow_rate_bps: u64,
        utilization_bps: u64,
        reserve_factor_bps: u64,  // protocol fee, e.g., 1000 (10%)
    ): u64 {
        let gross_rate = borrow_rate_bps * utilization_bps / 10_000;
        let net_factor = 10_000 - reserve_factor_bps;
        gross_rate * net_factor / 10_000
    }
    
    // ============================================
    // Helper: sqrt for u128
    // ============================================
    
    fun sqrt_128(n: u128): u128 {
        if (n == 0) return 0;
        let mut x = n;
        let mut y = (x + 1) / 2;
        while (y < x) {
            x = y;
            y = (x + n / x) / 2;
        };
        x
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test]
    public fun test_amm_math() {
        let r_a = 10_000u64;
        let r_b = 10_000u64;
        
        // Get output for 100 input with 0.3% fee
        let out = get_amount_out(100, r_a, r_b, 30);
        // Expected: ~99 (accounting for fee and price impact)
        assert!(out < 100 && out > 95, 0);
        
        // Get required input for 100 output
        let inp = get_amount_in(100, r_a, r_b, 30);
        assert!(inp > 100, 1);
        
        // Verify: using that input gives at least 100 output
        let back_out = get_amount_out(inp, r_a, r_b, 30);
        assert!(back_out >= 100, 2);
    }
    
    #[test]
    public fun test_interest_model() {
        // 0% utilization → base rate
        let rate_0 = borrow_rate_bps(0, 100, 500, 10000, 8000);
        assert!(rate_0 == 100, 0);
        
        // 80% utilization (at kink) → base + multiplier * 0.8 = 100 + 400 = 500
        let rate_kink = borrow_rate_bps(8000, 100, 500, 10000, 8000);
        assert!(rate_kink == 500, 1);
        
        // 100% utilization → kink_rate + jump * 0.2 = 500 + 2000 = 2500
        let rate_max = borrow_rate_bps(10000, 100, 500, 10000, 8000);
        assert!(rate_max == 2500, 2);
    }
}
```

---

## ตัวอย่างโปรแกรม: Pricing Oracle

```move
module learning::pricing_oracle {
    use std::signer;
    use aptos_std::table::{Self, Table};
    use aptos_framework::timestamp;
    use aptos_framework::event;
    
    // ============================================
    // Types
    // ============================================
    
    struct PriceData has copy, drop, store {
        price: u128,       // price in USD with 8 decimals (1 USD = 100000000)
        confidence: u64,   // confidence interval in bps
        timestamp: u64,
        source: u8,        // 0=aggregated, 1=chainlink, 2=pyth, etc.
    }
    
    struct Oracle has key {
        admin: address,
        prices: Table<address, PriceData>,
        price_history: Table<address, vector<PriceData>>,
        max_price_age: u64,  // max allowed staleness in seconds
        deviation_threshold_bps: u64,  // max deviation from aggregated
    }
    
    #[event]
    struct PriceUpdatedEvent has drop, store {
        token: address,
        old_price: u128,
        new_price: u128,
        source: u8,
        timestamp: u64,
    }
    
    #[event]
    struct PriceDeviationAlert has drop, store {
        token: address,
        oracle_price: u128,
        market_price: u128,
        deviation_bps: u64,
        timestamp: u64,
    }
    
    // ============================================
    // Error codes
    // ============================================
    
    const E_PRICE_STALE: u64 = 1;
    const E_PRICE_NOT_FOUND: u64 = 2;
    const E_ZERO_PRICE: u64 = 3;
    const E_DEVIATION_TOO_HIGH: u64 = 4;
    const E_NOT_ADMIN: u64 = 5;
    
    // ============================================
    // Initialize
    // ============================================
    
    public entry fun initialize_oracle(
        admin: &signer,
        max_price_age: u64,
        deviation_threshold_bps: u64,
    ) {
        let addr = signer::address_of(admin);
        move_to(admin, Oracle {
            admin: addr,
            prices: table::new(),
            price_history: table::new(),
            max_price_age,
            deviation_threshold_bps,
        });
    }
    
    // ============================================
    // Price Updates
    // ============================================
    
    public entry fun update_price(
        oracle_account: &signer,
        oracle_addr: address,
        token: address,
        price: u128,
        confidence: u64,
        source: u8,
    ) acquires Oracle {
        let caller = signer::address_of(oracle_account);
        let oracle = borrow_global_mut<Oracle>(oracle_addr);
        assert!(caller == oracle.admin, E_NOT_ADMIN);
        assert!(price > 0, E_ZERO_PRICE);
        
        let now = timestamp::now_seconds();
        
        // Get old price for event
        let old_price = if (table::contains(&oracle.prices, token)) {
            table::borrow(&oracle.prices, token).price
        } else {
            0
        };
        
        let price_data = PriceData {
            price,
            confidence,
            timestamp: now,
            source,
        };
        
        // Update current price
        if (table::contains(&oracle.prices, token)) {
            let existing = table::borrow_mut(&mut oracle.prices, token);
            *existing = price_data;
        } else {
            table::add(&mut oracle.prices, token, price_data);
        };
        
        event::emit(PriceUpdatedEvent {
            token,
            old_price,
            new_price: price,
            source,
            timestamp: now,
        });
    }
    
    // ============================================
    // Price Queries
    // ============================================
    
    public fun get_price(
        oracle_addr: address,
        token: address,
    ): u128 acquires Oracle {
        let oracle = borrow_global<Oracle>(oracle_addr);
        assert!(table::contains(&oracle.prices, token), E_PRICE_NOT_FOUND);
        
        let data = table::borrow(&oracle.prices, token);
        let now = timestamp::now_seconds();
        
        assert!(now - data.timestamp <= oracle.max_price_age, E_PRICE_STALE);
        data.price
    }
    
    public fun get_price_with_confidence(
        oracle_addr: address,
        token: address,
    ): (u128, u64, u64) acquires Oracle {  // (price, confidence, timestamp)
        let oracle = borrow_global<Oracle>(oracle_addr);
        assert!(table::contains(&oracle.prices, token), E_PRICE_NOT_FOUND);
        
        let data = table::borrow(&oracle.prices, token);
        let now = timestamp::now_seconds();
        assert!(now - data.timestamp <= oracle.max_price_age, E_PRICE_STALE);
        
        (data.price, data.confidence, data.timestamp)
    }
    
    // ============================================
    // Math utilities for oracle users
    // ============================================
    
    // Convert USD value to token amount
    public fun usd_to_token_amount(
        usd_value: u64,    // in USD with 2 decimals ($1 = 100)
        token_price: u128, // price from oracle (8 decimals)
        token_decimals: u8,
    ): u64 {
        // usd_value is in cents, token_price is in 1e-8 USD
        // token amount in token_decimals units
        
        let scale = pow10u128(token_decimals as u64);
        let usd_in_8_decimals = (usd_value as u128) * 1_000_000;  // cents to 8 decimals
        
        (usd_in_8_decimals * scale / token_price) as u64
    }
    
    // Convert token amount to USD value
    public fun token_to_usd_value(
        token_amount: u64,  // in token's native decimals
        token_price: u128,  // from oracle (8 decimals)
        token_decimals: u8,
    ): u64 {  // returns USD in cents
        let scale = pow10u128(token_decimals as u64);
        let value_8_dec = (token_amount as u128) * token_price / scale;
        // Convert from 8 decimals to cents
        (value_8_dec / 1_000_000) as u64
    }
    
    // Check if price deviation is acceptable
    public fun check_deviation(
        oracle_price: u128,
        market_price: u128,
        threshold_bps: u64,
    ): bool {
        let diff = if (oracle_price > market_price) {
            oracle_price - market_price
        } else {
            market_price - oracle_price
        };
        
        let deviation_bps = (diff * 10_000 / oracle_price) as u64;
        deviation_bps <= threshold_bps
    }
    
    // TWAP: Time-weighted average price
    public fun calculate_twap(
        prices: &vector<u128>,
        weights: &vector<u64>,
    ): u128 {
        use std::vector;
        
        let len = vector::length(prices);
        assert!(len == vector::length(weights) && len > 0, 1);
        
        let weighted_sum = 0u128;
        let total_weight = 0u128;
        let i = 0u64;
        
        while (i < len) {
            let price = *vector::borrow(prices, i);
            let weight = (*vector::borrow(weights, i)) as u128;
            
            weighted_sum = weighted_sum + price * weight;
            total_weight = total_weight + weight;
            i = i + 1;
        };
        
        assert!(total_weight > 0, 2);
        weighted_sum / total_weight
    }
    
    // ============================================
    // View Functions
    // ============================================
    
    #[view]
    public fun current_price(oracle_addr: address, token: address): u128 acquires Oracle {
        get_price(oracle_addr, token)
    }
    
    #[view]
    public fun price_is_fresh(oracle_addr: address, token: address): bool acquires Oracle {
        let oracle = borrow_global<Oracle>(oracle_addr);
        if (!table::contains(&oracle.prices, token)) return false;
        
        let data = table::borrow(&oracle.prices, token);
        let now = timestamp::now_seconds();
        now - data.timestamp <= oracle.max_price_age
    }
    
    // ============================================
    // Helpers
    // ============================================
    
    fun pow10u128(exp: u64): u128 {
        let result = 1u128;
        let i = 0u64;
        while (i < exp) {
            result = result * 10;
            i = i + 1;
        };
        result
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test]
    public fun test_usd_conversions() {
        // ETH price: $3000 (= 300000000000 in 8 decimals)
        let eth_price: u128 = 300_000_000_000;  // $3000 with 8 decimals
        
        // How much ETH for $150?
        let eth_amount = usd_to_token_amount(15_000, eth_price, 8);
        // $150 / $3000 = 0.05 ETH = 5000000 (8 decimals)
        assert!(eth_amount == 5_000_000, 0);
        
        // Value of 2 ETH = 200000000 base units
        let usd_val = token_to_usd_value(200_000_000, eth_price, 8);
        // 2 ETH * $3000 = $6000 = 600000 cents
        assert!(usd_val == 600_000, 1);
    }
    
    #[test]
    public fun test_twap() {
        use std::vector;
        
        let prices = vector[100u128, 110u128, 120u128, 90u128, 100u128];
        let weights = vector[1u64, 2u64, 3u64, 2u64, 1u64];
        
        let twap = calculate_twap(&prices, &weights);
        // (100*1 + 110*2 + 120*3 + 90*2 + 100*1) / (1+2+3+2+1)
        // = (100 + 220 + 360 + 180 + 100) / 9
        // = 960 / 9 = 106
        assert!(twap == 106, 0);
    }
}
```

---

## สรุป Math and Fixed-Point

| Concept | Implementation |
|---------|---------------|
| Integer overflow | Use `u128` for intermediate calculations |
| Fractions | Scale integers (WAD = 1e18) |
| Percentages | Basis points (BPS = 1e4) |
| Rounding | Add half divisor before divide |
| Square root | Newton-Raphson iteration |
| Financial | wad_mul, wad_div, ray_mul |

---

**ต่อไป**: [Part 20 - Beginner Summary →](part-20-beginner-summary.md)
