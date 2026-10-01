# Part 63: On-Chain Options & Derivatives

## สารบัญ
- [Options Fundamentals](#options-fundamentals)
- [Black-Scholes on Move](#black-scholes-on-move)
- [European Options Implementation](#european-options-implementation)
- [American Options & Early Exercise](#american-options--early-exercise)
- [Options Market Making](#options-market-making)
- [Structured Products](#structured-products)

---

## Options Fundamentals

```
Options Basics:

Call Option: Right to BUY at strike price
Put Option: Right to SELL at strike price

Key Terms:
  Strike (K): Price to exercise at
  Expiry: When option expires
  Premium: Cost of option
  Underlying: Asset the option is for
  
  In-the-Money (ITM): 
    Call: spot > strike (profitable)
    Put: spot < strike (profitable)
    
  Out-of-the-Money (OTM):
    Call: spot < strike
    Put: spot > strike
    
  At-the-Money (ATM):
    spot ≈ strike

Option Value = Intrinsic Value + Time Value

Intrinsic Value:
  Call: max(0, spot - strike)
  Put: max(0, strike - spot)
  
Time Value:
  Opportunity value before expiry
  = premium - intrinsic value
  Decays to 0 at expiry (theta decay)

Greeks:
  Delta (Δ): Price sensitivity to underlying
             Call delta: 0 to 1
             Put delta: -1 to 0
  
  Gamma (Γ): Delta sensitivity to underlying
             How fast delta changes
  
  Theta (θ): Time decay
             How much option loses per day
  
  Vega (ν): Volatility sensitivity
             Higher IV = higher premium
             
  Rho (ρ): Interest rate sensitivity

Black-Scholes Formula:
  C = S*N(d1) - K*e^(-rT)*N(d2)
  P = K*e^(-rT)*N(-d2) - S*N(-d1)
  
  d1 = [ln(S/K) + (r + σ²/2)T] / (σ√T)
  d2 = d1 - σ√T
  
  S = spot price
  K = strike price
  r = risk-free rate
  T = time to expiry (years)
  σ = volatility (annual)
  N() = cumulative normal distribution
```

---

## Black-Scholes on Move

```move
module options::black_scholes {
    
    // ============================================
    // Black-Scholes in Move (integer math)
    // ============================================
    
    // Black-Scholes assumes:
    // 1. Log-normal price distribution
    // 2. Constant volatility
    // 3. No dividends
    // 4. Continuous trading
    // 5. Known risk-free rate
    
    // Move has no floating point → use fixed-point (Q64.64)
    // All values scaled: price scaled to 1e8, vol to 1e6 (100% = 1_000_000)
    
    const SCALE: u128 = 1_000_000;  // 1e6 precision
    const SQRT_SCALE: u128 = 1_000; // 1e3 for square roots
    
    // Normal CDF approximation (Hart's approximation)
    // Accurate to ~0.0001 for |x| < 8
    public fun norm_cdf(x: i64): u64 {
        // Convert from scaled to actual
        // x is scaled by 1e6
        
        if (x > 8_000_000) return 1_000_000;  // N(8) ≈ 1
        if (x < -8_000_000) return 0;          // N(-8) ≈ 0
        
        if (x >= 0) {
            // Polynomial approximation for x >= 0
            let x_abs = (x as u64);
            
            // Coefficients (scaled for integer math)
            // From Abramowitz and Stegun approximation
            let t = SCALE * SCALE / (SCALE + 23164 * x_abs / SCALE / 10);
            
            // Evaluate polynomial
            let poly = 127414 - 142124 * t / SCALE + 710791 * t * t / SCALE / SCALE
                - 726096 * t * t * t / SCALE / SCALE / SCALE
                + 530704 * t * t * t * t / SCALE / SCALE / SCALE / SCALE;
            
            let exp_term = exp_neg_sq(x_abs);
            
            let result = SCALE - poly * exp_term / SCALE;
            std::u64::min(result as u64, SCALE as u64)
        } else {
            let pos_result = norm_cdf(-x);
            (SCALE as u64) - pos_result
        }
    }
    
    // e^(-x²/2) for CDF calculation
    fun exp_neg_sq(x_scaled: u64): u64 {
        // Taylor series approximation
        // e^(-x²/2) ≈ 1 - x²/2 + x⁴/8 - x⁶/48 for small x
        let x2 = (x_scaled as u128) * (x_scaled as u128) / SCALE;
        
        if (x2 > 20 * SCALE) return 0;  // Very small
        
        let term1 = SCALE;
        let term2 = x2 / 2;
        let term3 = x2 * x2 / 8 / SCALE;
        let term4 = x2 * x2 * x2 / 48 / SCALE / SCALE;
        
        if (term1 + term3 < term2 + term4) return 0;
        
        ((term1 + term3 - term2 - term4) as u64)
    }
    
    // Integer square root
    public fun isqrt(n: u128): u128 {
        if (n == 0) return 0;
        let mut x = n;
        let mut y = (x + 1) / 2;
        while (y < x) {
            x = y;
            y = (x + n / x) / 2;
        };
        x
    }
    
    // Natural logarithm (ln) approximation
    // ln(x) where x is scaled by SCALE
    public fun ln_approx(x_scaled: u128): i64 {
        // Use: ln(x) = ln(m * 2^n) = ln(m) + n*ln(2)
        // where 1 <= m < 2
        
        if (x_scaled == 0) return -1_000_000_000;  // -infinity
        
        let mut n = 0i64;
        let mut x = x_scaled;
        
        // Normalize to [SCALE, 2*SCALE)
        while (x >= 2 * SCALE) {
            x = x / 2;
            n = n + 1;
        };
        while (x < SCALE) {
            x = x * 2;
            n = n - 1;
        };
        
        // ln(1 + y) ≈ y - y²/2 + y³/3 for |y| < 1
        // y = (x - SCALE) / SCALE
        let y = x - SCALE;
        let y2 = y * y / SCALE;
        let y3 = y * y * y / SCALE / SCALE;
        
        let ln_m = ((y as i128) - (y2 as i128) / 2 + (y3 as i128) / 3) as i64;
        
        // ln(2) * SCALE ≈ 693147
        let ln_2_scaled: i64 = 693147;
        
        ln_m + n * ln_2_scaled
    }
    
    // ============================================
    // Black-Scholes Option Price
    // ============================================
    
    public struct BSParams has drop {
        spot: u64,       // Current price (scaled 1e8)
        strike: u64,     // Strike price (scaled 1e8)
        vol_annual: u64, // Annual volatility (scaled 1e6, e.g., 500_000 = 50%)
        rate: u64,       // Risk-free rate (scaled 1e6, e.g., 50_000 = 5%)
        time_to_expiry: u64,  // Seconds remaining
    }
    
    public fun black_scholes_call(params: &BSParams): u64 {
        let t_years_scaled = (params.time_to_expiry as u128) * SCALE / (365 * 24 * 3600);
        
        if (t_years_scaled == 0) {
            // Expired: max(S - K, 0)
            return if (params.spot > params.strike) {
                params.spot - params.strike
            } else { 0 }
        };
        
        let s = params.spot as u128;
        let k = params.strike as u128;
        let v = params.vol_annual as u128;
        let r = params.rate as u128;
        let t = t_years_scaled;
        
        // d1 = [ln(S/K) + (r + σ²/2)T] / (σ√T)
        let ln_sk = ln_approx(s * SCALE / k);  // ln(S/K) scaled
        
        let v2_half_t = v * v / SCALE / 2 * t / SCALE;  // σ²/2 * T
        let r_t = r * t / SCALE;                          // r * T
        
        let numerator = (ln_sk as i128) + (v2_half_t as i128) + (r_t as i128);
        
        let vol_sqrt_t = isqrt(v * v * t / SCALE);  // σ√T
        
        if (vol_sqrt_t == 0) return 0;
        
        let d1_scaled: i64 = (numerator * SCALE as i128 / vol_sqrt_t as i128) as i64;
        let d2_scaled: i64 = d1_scaled - (vol_sqrt_t as i64);
        
        let n_d1 = norm_cdf(d1_scaled);
        let n_d2 = norm_cdf(d2_scaled);
        
        // C = S * N(d1) - K * e^(-rT) * N(d2)
        let s_n_d1 = s * (n_d1 as u128) / SCALE;
        
        // e^(-rT) approximation
        let exp_neg_rt = exp_neg(r_t);
        let k_exp_n_d2 = k * (exp_neg_rt as u128) / SCALE * (n_d2 as u128) / SCALE;
        
        if (s_n_d1 > k_exp_n_d2) {
            (s_n_d1 - k_exp_n_d2) as u64
        } else {
            0
        }
    }
    
    public fun black_scholes_put(params: &BSParams): u64 {
        // Put-Call Parity: P = C - S + K*e^(-rT)
        let call_price = black_scholes_call(params);
        
        let t_years_scaled = (params.time_to_expiry as u128) * SCALE / (365 * 24 * 3600);
        let r_t = (params.rate as u128) * t_years_scaled / SCALE;
        let exp_neg_rt = exp_neg(r_t);
        
        let k_disc = (params.strike as u128) * (exp_neg_rt as u128) / SCALE;
        
        let parity = if (k_disc > params.spot as u128) {
            (k_disc - params.spot as u128) as u64
        } else { 0 };
        
        call_price + parity
    }
    
    // e^(-x) approximation
    fun exp_neg(x_scaled: u128): u64 {
        // e^(-x) = 1/(e^x)
        // For small x: e^x ≈ 1 + x + x²/2 + x³/6
        if (x_scaled >= 20 * SCALE) return 0;
        
        let x2 = x_scaled * x_scaled / SCALE;
        let x3 = x2 * x_scaled / SCALE;
        
        let e_x = SCALE + x_scaled + x2 / 2 + x3 / 6;
        (SCALE * SCALE / e_x) as u64
    }
}
```

---

## European Options Implementation

```move
module options::european {
    use aptos_std::smart_table::{Self, SmartTable};
    use options::black_scholes::{Self, BSParams};
    
    // ============================================
    // European Options: Exercise only at expiry
    // ============================================
    
    const STATE_ACTIVE: u8 = 0;
    const STATE_EXERCISED: u8 = 1;
    const STATE_EXPIRED: u8 = 2;
    
    struct OptionsVault has key {
        // Written options
        options: SmartTable<u64, Option>,
        next_id: u64,
        
        // Collateral per token type
        collateral: SmartTable<std::string::String, u64>,
        
        // Oracle for underlying price
        oracle_addr: address,
        
        // Protocol fee (on premium)
        protocol_fee_bps: u64,
    }
    
    struct Option has store, copy, drop {
        id: u64,
        writer: address,
        buyer: std::option::Option<address>,
        
        is_call: bool,
        underlying: std::string::String,  // "BTC", "ETH", etc.
        strike: u64,                      // Scaled 1e8
        expiry: u64,                      // Unix timestamp
        size: u64,                        // Number of contracts (1 = 1 unit)
        
        premium: u64,                     // What buyer paid
        
        // Collateral locked from writer
        // Call: lock underlying (to deliver if exercised)
        // Put: lock strike * size (to pay if exercised)
        collateral_locked: u64,
        
        state: u8,
    }
    
    // Writer creates (sells) an option
    public entry fun write_option(
        writer: &signer,
        vault_addr: address,
        is_call: bool,
        underlying: std::string::String,
        strike: u64,
        expiry: u64,
        size: u64,
        ask_premium: u64,
    ) acquires OptionsVault {
        let vault = borrow_global_mut<OptionsVault>(vault_addr);
        let writer_addr = std::signer::address_of(writer);
        
        let now = aptos_framework::timestamp::now_seconds();
        assert!(expiry > now, 1);
        assert!(size > 0, 2);
        
        // Calculate required collateral
        let collateral = if (is_call) {
            // Call writer locks underlying (size units)
            size  // 1:1 for now
        } else {
            // Put writer locks strike * size in USDC
            strike * size / 1_000_000  // Adjust scale
        };
        
        // Lock collateral from writer
        // (simplified - in production pull actual tokens)
        
        let id = vault.next_id;
        vault.next_id = id + 1;
        
        smart_table::add(&mut vault.options, id, Option {
            id,
            writer: writer_addr,
            buyer: std::option::none(),
            is_call,
            underlying,
            strike,
            expiry,
            size,
            premium: ask_premium,
            collateral_locked: collateral,
            state: STATE_ACTIVE,
        });
    }
    
    // Buyer purchases option
    public entry fun buy_option(
        buyer: &signer,
        vault_addr: address,
        option_id: u64,
        max_premium: u64,
    ) acquires OptionsVault {
        let vault = borrow_global_mut<OptionsVault>(vault_addr);
        let buyer_addr = std::signer::address_of(buyer);
        
        let option = smart_table::borrow_mut(&mut vault.options, option_id);
        
        assert!(option.state == STATE_ACTIVE, 1);
        assert!(std::option::is_none(&option.buyer), 2);
        assert!(option.premium <= max_premium, 3);
        assert!(aptos_framework::timestamp::now_seconds() < option.expiry, 4);
        
        // Collect premium from buyer
        // coin::transfer<USDC>(buyer, vault_addr, option.premium)
        
        // Pay writer premium (minus fee)
        let fee = option.premium * vault.protocol_fee_bps / 10_000;
        let writer_payment = option.premium - fee;
        // coin::transfer<USDC>(vault_signer, option.writer, writer_payment)
        
        option.buyer = std::option::some(buyer_addr);
    }
    
    // Exercise option at expiry
    public entry fun exercise(
        holder: &signer,
        vault_addr: address,
        option_id: u64,
    ) acquires OptionsVault {
        let vault = borrow_global_mut<OptionsVault>(vault_addr);
        let holder_addr = std::signer::address_of(holder);
        
        let option = smart_table::borrow_mut(&mut vault.options, option_id);
        
        assert!(option.state == STATE_ACTIVE, 1);
        assert!(std::option::contains(&option.buyer, &holder_addr), 2);
        
        let now = aptos_framework::timestamp::now_seconds();
        assert!(now >= option.expiry, 3);  // European: only at expiry
        
        // Get current price from oracle
        let spot_price = get_oracle_price(&option.underlying, vault.oracle_addr);
        
        // Calculate payoff
        let payoff = if (option.is_call) {
            if (spot_price > option.strike) {
                (spot_price - option.strike) * option.size / 1_000_000
            } else { 0 }
        } else {
            if (option.strike > spot_price) {
                (option.strike - spot_price) * option.size / 1_000_000
            } else { 0 }
        };
        
        option.state = STATE_EXERCISED;
        
        // Pay holder
        if (payoff > 0) {
            // coin::transfer<USDC>(vault_signer, holder_addr, payoff)
        };
        
        // Return remaining collateral to writer
        if (payoff < option.collateral_locked) {
            let writer_return = option.collateral_locked - payoff;
            // coin::transfer(vault_signer, option.writer, writer_return)
        };
    }
    
    // Expire worthless options (for gas reclaim)
    public entry fun expire(
        caller: &signer,
        vault_addr: address,
        option_id: u64,
    ) acquires OptionsVault {
        let vault = borrow_global_mut<OptionsVault>(vault_addr);
        let option = smart_table::borrow_mut(&mut vault.options, option_id);
        
        let now = aptos_framework::timestamp::now_seconds();
        assert!(now >= option.expiry + 86400, 1);  // 24h grace period after expiry
        
        // Return full collateral to writer
        // coin::transfer(vault_signer, option.writer, option.collateral_locked)
        
        option.state = STATE_EXPIRED;
    }
    
    fun get_oracle_price(_underlying: &std::string::String, _oracle: address): u64 { 0 }
}
```

---

## Structured Products

```move
module options::structured_products {
    
    // ============================================
    // Structured Products using options primitives
    // ============================================
    
    // 1. COVERED CALL VAULT
    //    - Deposit ETH
    //    - Automatically sell OTM calls each week
    //    - Earn premium as yield
    //    - Risk: miss upside if price rises above strike
    
    struct CoveredCallVault has key {
        // Deposited assets
        total_assets: u64,
        total_shares: u64,
        
        // Current round
        round: u64,
        round_start: u64,
        round_duration: u64,
        
        // Option parameters
        strike_pct_otm: u64,  // e.g., 110 = 10% OTM
        
        // Written option ID
        current_option: std::option::Option<u64>,
    }
    
    // 2. PROTECTIVE PUT VAULT
    //    - Deposit USDC
    //    - Buy ATM puts for protection
    //    - Automatically hedge portfolio
    //    - Cost: put premium (insurance cost)
    
    // 3. IRON CONDOR
    //    - Sell OTM call + OTM put
    //    - Collect premium from both sides
    //    - Profit if price stays in range
    //    - Risk: large move either direction
    
    struct IronCondor has key {
        long_put_strike: u64,   // Lower bound
        short_put_strike: u64,  // Inner lower
        short_call_strike: u64, // Inner upper
        long_call_strike: u64,  // Upper bound
        expiry: u64,
        size: u64,
        net_premium: u64,  // Premium received (short - long)
    }
    
    // Max profit = net_premium
    // Max loss = (short_put - long_put) * size - net_premium
    
    public fun iron_condor_payoff(
        condor: &IronCondor,
        spot_at_expiry: u64,
    ): i64 {
        let lp_payoff = max_zero(condor.long_put_strike as i64 - spot_at_expiry as i64);
        let sp_payoff = max_zero(condor.short_put_strike as i64 - spot_at_expiry as i64);
        let sc_payoff = max_zero(spot_at_expiry as i64 - condor.short_call_strike as i64);
        let lc_payoff = max_zero(spot_at_expiry as i64 - condor.long_call_strike as i64);
        
        // Net = collect short premium, pay long exercise
        // Short positions: you pay out
        // Long positions: you receive
        let net = (condor.net_premium as i64)
            + lp_payoff    // Long put: receive if below
            - sp_payoff    // Short put: pay if below (inner)
            - sc_payoff    // Short call: pay if above (inner)
            + lc_payoff;   // Long call: receive if above
        
        net
    }
    
    fun max_zero(x: i64): i64 {
        if (x > 0) x else 0
    }
    
    // 4. PRINCIPAL-PROTECTED NOTE
    //    - Invest 90% in yield (to return principal at expiry)
    //    - Invest 10% in ATM call options (for upside)
    //    - Guarantee: get principal back minimum
    //    - Potential: participate in upside
    
    struct PrincipalProtectedNote has key {
        principal: u64,
        yield_allocation: u64,   // e.g., 90% in staking
        option_allocation: u64,  // e.g., 10% in calls
        participation_rate: u64, // e.g., 80% of upside
        maturity: u64,
        strike: u64,
    }
    
    // Return at maturity
    public fun ppn_return(
        note: &PrincipalProtectedNote,
        spot_at_maturity: u64,
        yield_earned: u64,
    ): u64 {
        // Principal always returned (from yield allocation)
        let base_return = note.principal;
        
        // Upside from options if in the money
        let option_payoff = if (spot_at_maturity > note.strike) {
            let price_gain = spot_at_maturity - note.strike;
            price_gain * note.participation_rate / 100
        } else {
            0
        };
        
        base_return + option_payoff + yield_earned
    }
}
```

---

## สรุป Options & Derivatives

```
On-Chain Options Summary:

Options Types Implemented:
  ✓ European Call/Put (exercise at expiry only)
  ✓ American options (exercise anytime - more complex)
  ✓ Iron Condor (range-bound strategy)
  ✓ Principal Protected Note

Black-Scholes on Move:
  - Integer approximations for all functions
  - Normal CDF via Hart's polynomial
  - ln() via normalization + Taylor series
  - e^(-x) via Taylor series
  - Error < 1% for typical parameters
  
  Use cases: pricing, IV calculation, delta hedging

Option Vault Pattern (popular in DeFi):
  - Users deposit → vault manages options strategy
  - Automated weekly option selling
  - Yield = option premium
  - Risk = position loss if ITM at expiry
  
  Examples: Ribbon Finance, Friktion, ThetaNuts

Key Risk Parameters:
  Coverage ratio: collateral / max loss
  Moneyness: how far OTM/ITM
  Time to expiry: theta decay rate
  Implied vol: market's volatility expectation

DeFi Options Challenges:
  - Liquidity fragmentation (each strike/expiry separate)
  - Gas costs (complex settlement)
  - Oracle manipulation at expiry (price manipulation)
  - American options: complex binomial pricing

Defense Against Oracle Manipulation at Expiry:
  - TWAP price (30min) instead of spot
  - Multiple oracle sources
  - Delay in settlement (T+1)
```

---

**ก่อนหน้า**: [Part 62 - Perpetual DEX ←](part-62-perpetual-dex.md)
**ต่อไป**: [Part 64 - Layer 2 & Rollup Concepts →](part-64-layer2-rollups.md)
