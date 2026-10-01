# Part 72: Lending Protocol Design

## สารบัญ
- [Lending Protocol Fundamentals](#lending-protocol-fundamentals)
- [Interest Rate Models](#interest-rate-models)
- [Collateral & Liquidation System](#collateral--liquidation-system)
- [Multi-Asset Lending Pool](#multi-asset-lending-pool)
- [Isolated Margin vs Cross Margin](#isolated-margin-vs-cross-margin)
- [Under-Collateralized Lending](#under-collateralized-lending)
- [Full Implementation](#full-implementation)

---

## Lending Protocol Fundamentals

```
Lending Protocol Architecture:

PARTICIPANTS
  Suppliers (Lenders):
    - Deposit tokens → receive interest-bearing tokens (cTokens, aTokens)
    - Earn: supply APY (from borrower interest)
    - Risk: protocol hack, bad debt if liquidations fail
    
  Borrowers:
    - Deposit collateral → borrow other tokens
    - Pay: borrow APY (interest rate)
    - Risk: liquidation if collateral value drops
    
  Liquidators:
    - Monitor undercollateralized positions
    - Pay off borrower debt → receive collateral + bonus (5-15%)
    - Risk: gas costs, sandwich attacks, failed liquidations
    
KEY METRICS
  Health Factor = sum(collateral_i * liquidation_threshold_i) / total_debt
    Health > 1.0: Safe
    Health = 1.0: Liquidation threshold
    Health < 1.0: Can be liquidated
    
  Utilization Rate = total_borrows / total_supply
    Optimal: ~80% (supply enough for withdrawals, high enough for interest)
    
  Interest Rate (depends on utilization):
    Below optimal: Low rate to attract borrowers
    Above optimal: Very high rate to force repayment / attract suppliers
    
AAVE vs COMPOUND vs Move Protocols
  Aave V3 (Ethereum):  
    - Per-asset eMode (correlated assets like stETH/ETH)
    - Isolation mode (new assets with debt ceiling)
    - Portal (cross-chain collateral)
    
  Compound V3 (Comet):
    - Single base asset per market (simpler)
    - Supply collateral to borrow USDC only
    
  Aries (Aptos):
    - Similar to Compound/Aave on Aptos
    - Uses Move's resource model for safety
    
  Scallop (Sui):
    - Lending on Sui using object model
    - Isolated pools per asset
```

---

## Interest Rate Models

```move
module lending::interest_rate {
    // ============================================
    // INTEREST RATE MODELS
    //
    // 1. Linear: rate = base + utilization * multiplier
    // 2. Kinked (Jump Rate): Two-segment model
    //    - Below optimal: gentle slope
    //    - Above optimal: steep slope (emergency)
    // 3. Proportional-Integral (PI) controller
    //    - Rate adjusts based on target utilization
    // ============================================
    
    // All rates in APY basis points (1 bps = 0.01% per year)
    const SECONDS_PER_YEAR: u64 = 31_536_000;
    const BPS_PER_YEAR: u64 = 10_000;
    
    struct KinkRateModel has store {
        // Below kink
        base_rate_bps: u64,        // Minimum rate (e.g., 100 = 1% APY)
        multiplier_bps: u64,       // Rate per 1% utilization (below kink)
        
        // Above kink
        kink_utilization: u64,     // e.g., 8000 = 80%
        jump_multiplier_bps: u64,  // Steep rate above kink
    }
    
    // Calculate borrow APY for given utilization
    public fun get_borrow_rate(
        model: &KinkRateModel,
        total_borrows: u64,
        total_supply: u64,
    ): u64 {
        if (total_supply == 0) return model.base_rate_bps;
        
        let utilization = total_borrows * 10_000 / total_supply;
        
        if (utilization <= model.kink_utilization) {
            // Linear below kink
            // rate = base + utilization * multiplier / 10000
            model.base_rate_bps + utilization * model.multiplier_bps / 10_000
        } else {
            // Jump rate above kink
            // rate = base + kink * multiplier + (util - kink) * jump_multiplier
            let base_at_kink = model.base_rate_bps 
                + model.kink_utilization * model.multiplier_bps / 10_000;
            let excess_util = utilization - model.kink_utilization;
            base_at_kink + excess_util * model.jump_multiplier_bps / 10_000
        }
    }
    
    // Supply rate derived from borrow rate and utilization
    // Supply rate = borrow rate * utilization * (1 - reserve factor)
    public fun get_supply_rate(
        model: &KinkRateModel,
        total_borrows: u64,
        total_supply: u64,
        reserve_factor_bps: u64,  // e.g., 1000 = 10% goes to protocol
    ): u64 {
        if (total_supply == 0) return 0;
        
        let borrow_rate = get_borrow_rate(model, total_borrows, total_supply);
        let utilization = total_borrows * 10_000 / total_supply;
        
        // Supply rate = borrow_rate * utilization * (1 - reserve_factor)
        borrow_rate * utilization / 10_000 
            * (10_000 - reserve_factor_bps) / 10_000
    }
    
    // Convert APY to per-second rate (for interest accrual)
    public fun apy_to_per_second(apy_bps: u64): u64 {
        // per_second = apy / seconds_per_year
        // To avoid precision loss, keep in higher precision
        // Return rate * 1e18 per second
        apy_bps * 1_000_000_000_000_000_000 / (10_000 * SECONDS_PER_YEAR)
    }
    
    // Compound interest over time
    // Returns: new_balance = principal * (1 + rate_per_second)^seconds
    // Approximation: (1+r)^n ≈ 1 + n*r for small r
    // For higher precision: use Taylor series
    public fun compound_interest(
        principal: u64,
        rate_per_second_scaled: u64,  // rate * 1e18
        seconds_elapsed: u64,
    ): u64 {
        // First order Taylor: new = principal * (1 + rate * seconds)
        // More accurate for small rates
        let interest_factor = 1_000_000_000_000_000_000u128  // 1e18 = 1.0
            + (rate_per_second_scaled as u128) * (seconds_elapsed as u128);
        
        ((principal as u128) * interest_factor / 1_000_000_000_000_000_000u128) as u64
    }
    
    // Typical rate model parameters:
    //   Stablecoins: base=100(1%), kink=8000(80%), mult=200(2%/100%util), jump=3000(30%)
    //   ETH:         base=0, kink=8000, mult=100, jump=1000
    //   Volatile:    base=200, kink=7000, mult=200, jump=5000
    
    public fun usdc_rate_model(): KinkRateModel {
        KinkRateModel {
            base_rate_bps: 100,       // 1% base
            multiplier_bps: 200,      // 2% at 100% utilization  
            kink_utilization: 8_000,  // Kink at 80%
            jump_multiplier_bps: 3_000, // 30% above kink (per 100%)
        }
    }
}
```

---

## Collateral & Liquidation System

```move
module lending::liquidation {
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // LIQUIDATION SYSTEM
    //
    // Key parameters per collateral asset:
    // - LTV (Loan-to-Value): Max borrow against collateral
    //   e.g., 75% LTV: deposit $100, borrow max $75
    //   
    // - Liquidation Threshold: When liquidation triggers
    //   e.g., 80%: position liquidatable when borrow > $80
    //   
    // - Liquidation Bonus: Extra % liquidator gets
    //   e.g., 5% bonus: liquidator pays $80 debt, gets $84 collateral
    //
    // Health Factor = sum(collateral_i * liq_threshold_i) / total_debt
    //   HF > 1: Safe
    //   HF < 1: Liquidatable
    // ============================================
    
    struct AssetConfig has store {
        ltv_bps: u64,                  // Max LTV (e.g., 7500 = 75%)
        liquidation_threshold_bps: u64, // Liquidation trigger (e.g., 8000 = 80%)
        liquidation_bonus_bps: u64,    // Bonus for liquidators (e.g., 500 = 5%)
        can_be_collateral: bool,
        can_be_borrowed: bool,
        borrow_cap: u64,               // Max total borrows for this asset
        supply_cap: u64,               // Max total supply
    }
    
    struct UserPosition has key {
        // Supplied collateral (maps token type → amount)
        collateral: SmartTable<std::string::String, u64>,
        
        // Borrows (maps token type → amount with interest)
        borrows: SmartTable<std::string::String, u64>,
        
        // Scaled debt (to apply interest efficiently)
        debt_scaled: SmartTable<std::string::String, u64>,
        
        is_collateral_enabled: SmartTable<std::string::String, bool>,
    }
    
    // Calculate user's health factor
    public fun get_health_factor(
        user_addr: address,
        asset_configs: &SmartTable<std::string::String, AssetConfig>,
        prices: &SmartTable<std::string::String, u64>,  // Price in USD (scaled 1e6)
    ): u64 acquires UserPosition {
        let position = borrow_global<UserPosition>(user_addr);
        
        // Calculate weighted collateral value
        let mut total_collateral_weighted = 0u128;
        // Iterate over collateral (SmartTable iteration in production)
        // For each token:
        //   weighted_value = collateral_amount * price * liq_threshold / 10000
        
        // Calculate total debt value  
        let mut total_debt_value = 0u128;
        // For each borrowed token:
        //   debt_value = borrow_amount * price
        
        if (total_debt_value == 0) return 10_000; // Max health factor
        
        // Health factor = weighted_collateral / debt
        // Returns bps (10000 = 1.0)
        (total_collateral_weighted * 10_000 / total_debt_value) as u64
    }
    
    // Liquidate an undercollateralized position
    public fun liquidate(
        liquidator: &signer,
        protocol_addr: address,
        user_to_liquidate: address,
        debt_token: std::string::String,    // Token to repay
        collateral_token: std::string::String,  // Collateral to receive
        debt_amount: u64,                   // Amount of debt to repay
    ) acquires UserPosition {
        let liquidator_addr = std::signer::address_of(liquidator);
        
        // Verify position is liquidatable
        // let hf = get_health_factor(user_to_liquidate, ...);
        // assert!(hf < 10_000, 1);  // HF < 1.0
        
        // Max 50% of debt can be repaid in one liquidation (Aave-style)
        // let user_debt = get_debt(user_to_liquidate, debt_token);
        // let max_liquidatable = user_debt / 2;
        // let actual_debt_to_repay = min(debt_amount, max_liquidatable);
        
        // Calculate collateral to give liquidator
        // collateral_out = debt_value_in_collateral * (1 + liquidation_bonus)
        // Example: repay $1000 USDC, get $1050 worth of ETH (5% bonus)
        
        // Transfer: liquidator pays debt, receives collateral
        // 1. Take debt token from liquidator
        // 2. Reduce borrower's debt
        // 3. Give liquidator collateral + bonus
        // 4. Protocol takes 10% of bonus as fee
    }
    
    // Bad debt socialization (if liquidation fails, spread losses to suppliers)
    public fun socialize_bad_debt(
        protocol_addr: address,
        token: std::string::String,
        bad_debt_amount: u64,
    ) {
        // If liquidation leaves bad debt (collateral worth less than debt):
        // Option 1: Insurance fund covers
        // Option 2: Supplier shares diluted (reduce exchange rate)
        // Option 3: Governance emergency (DAO vote to cover from treasury)
    }
}
```

---

## Multi-Asset Lending Pool

```move
module lending::pool {
    use aptos_framework::timestamp;
    use aptos_std::smart_table::{Self, SmartTable};
    
    struct LendingProtocol has key {
        // All supported assets
        markets: SmartTable<std::string::String, Market>,
        
        // Total reserves (protocol fee accumulation)
        total_reserves: SmartTable<std::string::String, u64>,
        
        // Oracle and config
        oracle_addr: address,
        config_addr: address,
        
        // Protocol stats
        total_value_locked: u64,
        total_borrowed: u64,
    }
    
    struct Market has store {
        // State
        total_supply: u64,        // Total tokens supplied (principal)
        total_borrows: u64,       // Total tokens borrowed
        
        // Interest index (grows over time to track accrued interest)
        // All user balances multiplied by their index at deposit time
        borrow_index: u128,       // Scaled 1e18, starts at 1e18
        supply_index: u128,       // Scaled 1e18
        
        // Last update timestamp for interest accrual
        last_interest_update: u64,
        
        // Configuration
        asset_config: lending::liquidation::AssetConfig,
        rate_model: lending::interest_rate::KinkRateModel,
        reserve_factor_bps: u64,
    }
    
    // Supply tokens to earn interest
    public fun supply<T>(
        supplier: &signer,
        protocol_addr: address,
        token_type: std::string::String,
        amount: u64,
    ) acquires LendingProtocol {
        let protocol = borrow_global_mut<LendingProtocol>(protocol_addr);
        
        // Accrue interest first
        accrue_interest(smart_table::borrow_mut(&mut protocol.markets, token_type));
        
        let market = smart_table::borrow_mut(&mut protocol.markets, token_type);
        
        // Take tokens from supplier
        let coins = aptos_framework::coin::withdraw<T>(supplier, amount);
        // Deposit to market reserves
        
        // Calculate share tokens to mint (scaled by supply_index)
        let shares = amount_to_shares(amount, market.supply_index);
        
        // Mint shares to supplier
        // In production: update user's share balance in UserPosition
        market.total_supply = market.total_supply + amount;
    }
    
    // Borrow tokens against collateral
    public fun borrow<T>(
        borrower: &signer,
        protocol_addr: address,
        token_type: std::string::String,
        amount: u64,
    ) acquires LendingProtocol {
        let protocol = borrow_global_mut<LendingProtocol>(protocol_addr);
        let borrower_addr = std::signer::address_of(borrower);
        
        accrue_interest(smart_table::borrow_mut(&mut protocol.markets, token_type));
        
        let market = smart_table::borrow_mut(&mut protocol.markets, token_type);
        
        // Check borrow capacity
        // let hf_after = simulate_borrow_health_factor(borrower_addr, token_type, amount);
        // assert!(hf_after > 10_000, 1);  // Still healthy after borrow
        
        // Check borrow cap
        assert!(market.total_borrows + amount <= market.asset_config.borrow_cap, 2);
        
        // Record debt (scaled by borrow_index for interest tracking)
        let debt_shares = amount_to_shares(amount, market.borrow_index);
        // Update user debt in UserPosition
        
        market.total_borrows = market.total_borrows + amount;
        
        // Send tokens to borrower
        // coin::deposit(borrower_addr, amount);
    }
    
    // Accrue interest (update indexes)
    fun accrue_interest(market: &mut Market) {
        let now = timestamp::now_microseconds();
        let elapsed = now - market.last_interest_update;
        
        if (elapsed == 0) return;
        
        let elapsed_seconds = elapsed / 1_000_000;
        
        // Calculate borrow rate
        let borrow_rate = lending::interest_rate::get_borrow_rate(
            &market.rate_model,
            market.total_borrows,
            market.total_supply,
        );
        
        let rate_per_second = lending::interest_rate::apy_to_per_second(borrow_rate);
        
        // Interest = total_borrows * rate * elapsed
        let interest = market.total_borrows 
            * rate_per_second as u64
            * elapsed_seconds
            / 1_000_000_000_000_000_000;
        
        // Reserve factor portion goes to protocol
        let reserve_portion = interest * market.reserve_factor_bps / 10_000;
        let supplier_portion = interest - reserve_portion;
        
        // Update state
        market.total_borrows = market.total_borrows + interest;
        market.total_supply = market.total_supply + supplier_portion;
        
        // Update indexes (1e18 precision)
        if (market.total_borrows > 0) {
            market.borrow_index = market.borrow_index 
                + market.borrow_index * (rate_per_second as u128) * (elapsed_seconds as u128) 
                / 1_000_000_000_000_000_000;
        };
        
        market.last_interest_update = now;
    }
    
    fun amount_to_shares(amount: u64, index: u128): u64 {
        ((amount as u128) * 1_000_000_000_000_000_000 / index) as u64
    }
    
    fun shares_to_amount(shares: u64, index: u128): u64 {
        ((shares as u128) * index / 1_000_000_000_000_000_000) as u64
    }
}
```

---

## สรุป Lending Protocols

```
Lending Protocol Key Formulas:

INTEREST RATE (Kinked Model)
  if util <= kink:
    rate = base + util * slope1
  else:
    rate = base + kink * slope1 + (util - kink) * slope2

HEALTH FACTOR
  HF = Σ(collateral_i * price_i * liq_threshold_i) / Σ(debt_i * price_i)
  Safe: HF > 1.0
  Liquidatable: HF < 1.0
  
LIQUIDATION MECHANICS
  max_liquidatable = user_debt * 0.5 (50% cap)
  collateral_seized = debt_repaid * debt_price / collateral_price * (1 + bonus)
  protocol_fee = bonus * 0.1 (10% of bonus)
  
EXCHANGE RATE (for interest-bearing tokens)
  exchange_rate = total_supply_underlying / total_shares
  grows over time as interest accrues
  
RISK PARAMETERS
  Asset         LTV   Liq.Thresh  Liq.Bonus
  BTC           70%   75%         8%
  ETH           75%   80%         7%
  USDC          87%   90%         5%
  APT (native)  65%   70%         10%
  Volatile      40%   50%         15%
  
BEST PRACTICES
  [ ] Pause new borrows before deprecating an asset
  [ ] Debt ceiling for new/risky assets
  [ ] Price oracle with TWAP + deviation threshold
  [ ] Bad debt tracking and socialization plan
  [ ] Insurance fund (5-10% of reserves)
  [ ] Liquidation incentive tuning (too low = bad debt, too high = premature liquidation)
```

---

**ก่อนหน้า**: [Part 71 - AMM Design ←](part-71-amm-design.md)
**ต่อไป**: [Part 73 - Token Launch & Vesting →](part-73-token-launch-vesting.md)
