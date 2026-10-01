# Part 29: Lending Protocol

## สารบัญ
- [Lending Protocol Overview](#lending-protocol-overview)
- [Interest Rate Model](#interest-rate-model)
- [Collateralized Borrowing](#collateralized-borrowing)
- [Liquidation Mechanism](#liquidation-mechanism)
- [ตัวอย่าง: Complete Lending Market](#ตัวอย่าง-complete-lending-market)

---

## Lending Protocol Overview

```
Lending Protocol Flow:
──────────────────────────────────────────────────────

Lenders deposit → get interest-bearing tokens (aTokens/cTokens)
Borrowers: collateral → borrow up to LTV ratio
Interest: borrowers pay → distributed to lenders

Key Concepts:
  LTV (Loan-to-Value): max borrow / collateral value
  Utilization = total_borrow / total_supply
  Borrow APY = f(utilization)
  Supply APY = borrow_APY * utilization * (1 - reserve_factor)
  Health Factor = collateral_value * LF / total_debt
    - HF < 1.0 → liquidatable
    
Liquidation:
  Liquidator pays borrower's debt
  Gets collateral at discount (liquidation bonus)
```

---

## Interest Rate Model

```move
module lending::interest_rate {
    // ============================================
    // Jump Rate Model (like Compound v2)
    // ============================================
    
    // Params:
    //   base_rate:    rate at 0% utilization
    //   slope1:       rate increase before kink
    //   slope2:       rate increase after kink (steep)
    //   kink:         utilization at which slope changes
    
    struct RateModel has copy, drop, store {
        base_rate_bps: u64,    // e.g. 200 = 2%/year
        slope1_bps: u64,       // e.g. 400 = 4%/year at kink
        slope2_bps: u64,       // e.g. 7500 = 75%/year at 100%
        kink_bps: u64,         // e.g. 8000 = 80% utilization kink
    }
    
    // Utilization = total_borrow / total_supply
    public fun utilization_bps(total_borrow: u64, total_supply: u64): u64 {
        if (total_supply == 0) return 0;
        (total_borrow as u128 * 10_000u128 / total_supply as u128) as u64
    }
    
    // Borrow rate per year in BPS
    public fun borrow_rate_bps(model: &RateModel, utilization: u64): u64 {
        if (utilization <= model.kink_bps) {
            // Linear below kink
            let slope = model.slope1_bps * utilization / model.kink_bps;
            model.base_rate_bps + slope
        } else {
            // Steeper above kink
            let excess = utilization - model.kink_bps;
            let max_excess = 10_000 - model.kink_bps;
            let slope2_portion = model.slope2_bps * excess / max_excess;
            model.base_rate_bps + model.slope1_bps + slope2_portion
        }
    }
    
    // Supply rate = borrow_rate * utilization * (1 - reserve_factor)
    public fun supply_rate_bps(
        model: &RateModel,
        utilization: u64,
        reserve_factor_bps: u64,
    ): u64 {
        let borrow_rate = borrow_rate_bps(model, utilization);
        let gross_rate = (borrow_rate as u128) * (utilization as u128) / 10_000u128;
        let net_rate = gross_rate * (10_000 - reserve_factor_bps as u128) / 10_000u128;
        net_rate as u64
    }
    
    // Accumulate interest for elapsed time
    // Simple: new_debt = debt * (1 + rate * time / year)
    public fun accrue_interest(
        debt: u64,
        borrow_rate_bps: u64,
        elapsed_seconds: u64,
    ): u64 {
        let seconds_per_year = 365 * 24 * 3600u64;
        // interest = debt * rate * time / year
        let interest = (debt as u128)
            * (borrow_rate_bps as u128)
            * (elapsed_seconds as u128)
            / (10_000u128 * seconds_per_year as u128);
        debt + interest as u64
    }
    
    // Compound interest (approximation for continuous compounding)
    // e^(rate*t) ≈ 1 + rate*t + (rate*t)^2/2
    public fun accrue_compound(
        debt: u64,
        borrow_rate_bps: u64,
        elapsed_seconds: u64,
    ): u64 {
        let seconds_per_year = 365u128 * 24 * 3600;
        let rate_times_time = (borrow_rate_bps as u128)
            * (elapsed_seconds as u128)
            / (10_000u128 * seconds_per_year);
        
        // First order: 1 + rt
        let first_order = debt as u128 * rate_times_time;
        // Second order: (rt)^2/2
        let second_order = debt as u128 * rate_times_time * rate_times_time / 2;
        
        debt + (first_order + second_order) as u64
    }
    
    // Index-based interest (like Compound cToken)
    // borrow_index grows over time
    // user_debt_at_time_T = user_shares * index_T
    
    struct BorrowIndex has copy, drop, store {
        value: u128,        // current index (starts at 1e18)
        last_updated: u64,  // timestamp
    }
    
    const INDEX_SCALE: u128 = 1_000_000_000_000_000_000;  // 1e18
    
    public fun update_index(
        index: &mut BorrowIndex,
        borrow_rate_bps: u64,
        current_time: u64,
    ) {
        if (current_time <= index.last_updated) return;
        
        let elapsed = (current_time - index.last_updated) as u128;
        let seconds_per_year = 365u128 * 24 * 3600;
        
        // new_index = old_index * (1 + rate * dt)
        let rate_factor = INDEX_SCALE + (borrow_rate_bps as u128)
            * elapsed * INDEX_SCALE / (10_000u128 * seconds_per_year);
        
        index.value = index.value * rate_factor / INDEX_SCALE;
        index.last_updated = current_time;
    }
    
    public fun debt_at_index(shares: u128, current_index: u128): u64 {
        (shares * current_index / INDEX_SCALE) as u64
    }
    
    public fun shares_for_debt(debt: u64, current_index: u128): u128 {
        (debt as u128) * INDEX_SCALE / current_index
    }
}
```

---

## Collateralized Borrowing

```move
module lending::market {
    use std::signer;
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::timestamp;
    use aptos_framework::event;
    use lending::interest_rate::{Self, BorrowIndex, RateModel};
    
    // ============================================
    // Types
    // ============================================
    
    struct Market<phantom Asset> has key {
        // Reserves
        total_supply: u64,        // Total supplied (including interest)
        total_borrow: u64,        // Total borrowed (including accrued interest)
        reserves: Coin<Asset>,    // Actual tokens in vault
        protocol_reserves: u64,  // Fees collected
        
        // Interest tracking
        borrow_index: BorrowIndex,
        supply_index: BorrowIndex,
        rate_model: RateModel,
        reserve_factor_bps: u64,
        
        // Risk params
        collateral_factor_bps: u64,  // max LTV (e.g. 7500 = 75%)
        liquidation_factor_bps: u64, // liquidation threshold (e.g. 8000 = 80%)
        liquidation_bonus_bps: u64,  // bonus for liquidators (e.g. 500 = 5%)
    }
    
    struct UserPosition<phantom Asset> has key {
        // Supply tracking
        supply_shares: u128,      // shares of total supply
        // Borrow tracking
        borrow_shares: u128,      // shares of total borrow
        borrow_index_at_entry: u128,  // index when position opened
    }
    
    struct OraclePrice has key {
        // Simplified: in practice use a real oracle
        prices: aptos_std::simple_map::SimpleMap<vector<u8>, u64>,
        last_updated: u64,
    }
    
    // ============================================
    // Events
    // ============================================
    
    #[event]
    struct Supplied has drop, store {
        user: address,
        amount: u64,
        shares: u128,
    }
    
    #[event]
    struct Withdrawn has drop, store {
        user: address,
        amount: u64,
        shares: u128,
    }
    
    #[event]
    struct Borrowed has drop, store {
        user: address,
        amount: u64,
        total_debt: u64,
    }
    
    #[event]
    struct Repaid has drop, store {
        user: address,
        amount: u64,
        remaining_debt: u64,
    }
    
    #[event]
    struct Liquidated has drop, store {
        borrower: address,
        liquidator: address,
        repaid_amount: u64,
        collateral_seized: u64,
    }
    
    // ============================================
    // Errors
    // ============================================
    
    const E_ZERO_AMOUNT: u64 = 1;
    const E_INSUFFICIENT_COLLATERAL: u64 = 2;
    const E_BORROW_TOO_LARGE: u64 = 3;
    const E_POSITION_HEALTHY: u64 = 4;
    const E_INSUFFICIENT_BALANCE: u64 = 5;
    const E_MARKET_NOT_INITIALIZED: u64 = 6;
    
    // ============================================
    // Initialize Market
    // ============================================
    
    public entry fun initialize_market<Asset>(
        admin: &signer,
        collateral_factor_bps: u64,
        liquidation_factor_bps: u64,
        liquidation_bonus_bps: u64,
        reserve_factor_bps: u64,
    ) {
        let now = timestamp::now_seconds();
        
        move_to(admin, Market<Asset> {
            total_supply: 0,
            total_borrow: 0,
            reserves: coin::zero<Asset>(),
            protocol_reserves: 0,
            borrow_index: BorrowIndex { value: 1_000_000_000_000_000_000u128, last_updated: now },
            supply_index: BorrowIndex { value: 1_000_000_000_000_000_000u128, last_updated: now },
            rate_model: RateModel {
                base_rate_bps: 200,      // 2%
                slope1_bps: 800,         // 8% at kink
                slope2_bps: 7500,        // 75% at 100%
                kink_bps: 8000,          // kink at 80% util
            },
            reserve_factor_bps,
            collateral_factor_bps,
            liquidation_factor_bps,
            liquidation_bonus_bps,
        });
    }
    
    // ============================================
    // Accrue Interest (must call before state-changing ops)
    // ============================================
    
    fun accrue<Asset>(market: &mut Market<Asset>) {
        let now = timestamp::now_seconds();
        if (now <= market.borrow_index.last_updated) return;
        
        let util = interest_rate::utilization_bps(market.total_borrow, market.total_supply);
        let borrow_rate = interest_rate::borrow_rate_bps(&market.rate_model, util);
        
        // Accrue interest on borrows
        let old_borrow = market.total_borrow;
        interest_rate::update_index(&mut market.borrow_index, borrow_rate, now);
        
        // Calculate interest accrued
        let new_borrow = interest_rate::debt_at_index(
            market.total_borrow as u128,
            market.borrow_index.value
        );
        let interest = new_borrow - old_borrow;
        
        // Split interest: protocol gets reserve_factor, rest goes to suppliers
        let protocol_share = interest * market.reserve_factor_bps / 10_000;
        market.protocol_reserves = market.protocol_reserves + protocol_share;
        market.total_supply = market.total_supply + interest - protocol_share;
        market.total_borrow = new_borrow;
    }
    
    // ============================================
    // Supply
    // ============================================
    
    public entry fun supply<Asset>(
        user: &signer,
        market_addr: address,
        amount: u64,
    ) acquires Market, UserPosition {
        assert!(amount > 0, E_ZERO_AMOUNT);
        
        let user_addr = signer::address_of(user);
        let market = borrow_global_mut<Market<Asset>>(market_addr);
        accrue(market);
        
        // Shares = amount / (total_supply / supply_tokens)
        // Simplified: 1:1 for first depositor
        let shares = if (market.total_supply == 0) {
            amount as u128
        } else {
            (amount as u128) * (market.total_supply as u128) / (market.total_supply as u128)
        };
        
        market.total_supply = market.total_supply + amount;
        coin::merge(&mut market.reserves, coin::withdraw<Asset>(user, amount));
        
        // Update user position
        if (!exists<UserPosition<Asset>>(user_addr)) {
            move_to(user, UserPosition<Asset> {
                supply_shares: 0,
                borrow_shares: 0,
                borrow_index_at_entry: market.borrow_index.value,
            });
        };
        
        let position = borrow_global_mut<UserPosition<Asset>>(user_addr);
        position.supply_shares = position.supply_shares + shares;
        
        event::emit(Supplied { user: user_addr, amount, shares });
    }
    
    // ============================================
    // Borrow
    // ============================================
    
    public entry fun borrow<Collateral, Debt>(
        user: &signer,
        collateral_market: address,
        debt_market: address,
        borrow_amount: u64,
    ) acquires Market, UserPosition {
        assert!(borrow_amount > 0, E_ZERO_AMOUNT);
        
        let user_addr = signer::address_of(user);
        
        // Check collateral health
        let max_borrow = compute_max_borrow<Collateral>(user_addr, collateral_market);
        
        // Get existing debt
        let current_debt = get_user_debt<Debt>(user_addr, debt_market);
        
        assert!(current_debt + borrow_amount <= max_borrow, E_BORROW_TOO_LARGE);
        
        let debt_market_ref = borrow_global_mut<Market<Debt>>(debt_market);
        accrue(debt_market_ref);
        
        assert!(borrow_amount <= coin::value(&debt_market_ref.reserves), E_INSUFFICIENT_BALANCE);
        
        debt_market_ref.total_borrow = debt_market_ref.total_borrow + borrow_amount;
        
        let borrowed = coin::extract(&mut debt_market_ref.reserves, borrow_amount);
        coin::deposit<Debt>(user_addr, borrowed);
        
        // Track borrow
        if (!exists<UserPosition<Debt>>(user_addr)) {
            move_to(user, UserPosition<Debt> {
                supply_shares: 0,
                borrow_shares: 0,
                borrow_index_at_entry: debt_market_ref.borrow_index.value,
            });
        };
        
        let position = borrow_global_mut<UserPosition<Debt>>(user_addr);
        position.borrow_shares = position.borrow_shares + (borrow_amount as u128);
        
        event::emit(Borrowed {
            user: user_addr,
            amount: borrow_amount,
            total_debt: current_debt + borrow_amount,
        });
    }
    
    // ============================================
    // Repay
    // ============================================
    
    public entry fun repay<Asset>(
        user: &signer,
        market_addr: address,
        repay_amount: u64,
    ) acquires Market, UserPosition {
        let user_addr = signer::address_of(user);
        let market = borrow_global_mut<Market<Asset>>(market_addr);
        accrue(market);
        
        let position = borrow_global_mut<UserPosition<Asset>>(user_addr);
        
        // Can't repay more than debt
        let current_debt = interest_rate::debt_at_index(
            position.borrow_shares, market.borrow_index.value
        );
        let actual_repay = if (repay_amount > current_debt) { current_debt } else { repay_amount };
        
        // Reduce borrow shares
        let shares_to_remove = (actual_repay as u128) * 1_000_000_000_000_000_000u128
            / market.borrow_index.value;
        position.borrow_shares = if (shares_to_remove > position.borrow_shares) {
            0
        } else {
            position.borrow_shares - shares_to_remove
        };
        
        market.total_borrow = market.total_borrow - actual_repay;
        
        let payment = coin::withdraw<Asset>(user, actual_repay);
        coin::merge(&mut market.reserves, payment);
        
        event::emit(Repaid {
            user: user_addr,
            amount: actual_repay,
            remaining_debt: current_debt - actual_repay,
        });
    }
    
    // ============================================
    // Liquidate
    // ============================================
    
    public entry fun liquidate<Collateral, Debt>(
        liquidator: &signer,
        borrower: address,
        collateral_market: address,
        debt_market: address,
        repay_amount: u64,
    ) acquires Market, UserPosition {
        // Check borrower is underwater
        let health_factor = compute_health_factor<Collateral, Debt>(
            borrower, collateral_market, debt_market
        );
        assert!(health_factor < 10_000, E_POSITION_HEALTHY);  // HF < 1.0
        
        let liquidator_addr = signer::address_of(liquidator);
        
        // Repay borrower's debt
        let debt_market_ref = borrow_global_mut<Market<Debt>>(debt_market);
        accrue(debt_market_ref);
        
        let payment = coin::withdraw<Debt>(liquidator, repay_amount);
        coin::merge(&mut debt_market_ref.reserves, payment);
        debt_market_ref.total_borrow = debt_market_ref.total_borrow - repay_amount;
        
        // Seize collateral (with bonus)
        let col_market = borrow_global_mut<Market<Collateral>>(collateral_market);
        accrue(col_market);
        
        // collateral_seized = repay * (1 + liquidation_bonus) / collateral_price * debt_price
        // Simplified: assume 1:1 price for now
        let collateral_seized = repay_amount
            * (10_000 + col_market.liquidation_bonus_bps)
            / 10_000;
        
        assert!(collateral_seized <= coin::value(&col_market.reserves), E_INSUFFICIENT_BALANCE);
        
        let seized = coin::extract(&mut col_market.reserves, collateral_seized);
        coin::deposit<Collateral>(liquidator_addr, seized);
        
        // Remove from borrower's position
        let borrower_pos = borrow_global_mut<UserPosition<Collateral>>(borrower);
        let shares_to_remove = (collateral_seized as u128);
        borrower_pos.supply_shares = if (shares_to_remove > borrower_pos.supply_shares) {
            0
        } else {
            borrower_pos.supply_shares - shares_to_remove
        };
        
        col_market.total_supply = col_market.total_supply - collateral_seized;
        
        event::emit(Liquidated {
            borrower,
            liquidator: liquidator_addr,
            repaid_amount: repay_amount,
            collateral_seized,
        });
    }
    
    // ============================================
    // View functions
    // ============================================
    
    #[view]
    public fun get_user_debt<Asset>(user_addr: address, market_addr: address): u64 acquires Market, UserPosition {
        if (!exists<UserPosition<Asset>>(user_addr)) return 0;
        
        let market = borrow_global<Market<Asset>>(market_addr);
        let position = borrow_global<UserPosition<Asset>>(user_addr);
        
        interest_rate::debt_at_index(position.borrow_shares, market.borrow_index.value)
    }
    
    #[view]
    public fun get_user_supply<Asset>(user_addr: address, market_addr: address): u64 acquires Market, UserPosition {
        if (!exists<UserPosition<Asset>>(user_addr)) return 0;
        
        let market = borrow_global<Market<Asset>>(market_addr);
        let position = borrow_global<UserPosition<Asset>>(user_addr);
        
        if (market.total_supply == 0) return 0;
        
        (position.supply_shares * (market.total_supply as u128) / market.total_supply as u128) as u64
    }
    
    #[view]
    public fun get_utilization<Asset>(market_addr: address): u64 acquires Market {
        let market = borrow_global<Market<Asset>>(market_addr);
        interest_rate::utilization_bps(market.total_borrow, market.total_supply)
    }
    
    #[view]
    public fun get_borrow_apy<Asset>(market_addr: address): u64 acquires Market {
        let market = borrow_global<Market<Asset>>(market_addr);
        let util = interest_rate::utilization_bps(market.total_borrow, market.total_supply);
        interest_rate::borrow_rate_bps(&market.rate_model, util)
    }
    
    // ============================================
    // Helpers
    // ============================================
    
    fun compute_max_borrow<Collateral>(user_addr: address, market_addr: address): u64 acquires Market, UserPosition {
        let supply = get_user_supply<Collateral>(user_addr, market_addr);
        let market = borrow_global<Market<Collateral>>(market_addr);
        supply * market.collateral_factor_bps / 10_000
    }
    
    fun compute_health_factor<Collateral, Debt>(
        user_addr: address,
        collateral_market: address,
        debt_market: address,
    ): u64 acquires Market, UserPosition {
        let collateral = get_user_supply<Collateral>(user_addr, collateral_market);
        let debt = get_user_debt<Debt>(user_addr, debt_market);
        
        if (debt == 0) return 20_000;  // HF = 2.0 if no debt
        
        let col_market = borrow_global<Market<Collateral>>(collateral_market);
        let collateral_value = collateral * col_market.liquidation_factor_bps / 10_000;
        
        // Health factor = collateral_value / debt (scaled 10000 = 1.0)
        (collateral_value as u128 * 10_000u128 / debt as u128) as u64
    }
}
```

---

## สรุป Lending Protocol

| Concept | Formula |
|---------|---------|
| Utilization | `borrow / supply` |
| Borrow APY | `base + slope1 * util` (below kink) |
| Supply APY | `borrow_APY * util * (1 - reserve_factor)` |
| Health Factor | `(collateral * LF) / debt` |
| Liquidatable when | `HF < 1.0` |
| Liquidation bonus | `collateral_seized = repay * (1 + bonus)` |

---

**ก่อนหน้า**: [Part 28 - DeFi AMM ←](part-28-defi-amm.md)
**ต่อไป**: [Part 30 - Staking and Yield →](part-30-staking-yield.md)
