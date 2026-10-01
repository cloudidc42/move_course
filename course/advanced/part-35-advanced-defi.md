# Part 35: Advanced DeFi - Perpetuals & Options

## สารบัญ
- [Perpetual Futures Overview](#perpetual-futures-overview)
- [Funding Rate Mechanism](#funding-rate-mechanism)
- [Position Management](#position-management)
- [Options Pricing Basics](#options-pricing-basics)
- [Vault-Based Options](#vault-based-options)
- [ตัวอย่าง: Mini-Perpetual Protocol](#ตัวอย่าง-mini-perpetual-protocol)

---

## Perpetual Futures Overview

```
Perpetual = futures contract ที่ไม่มีวันหมดอายุ

Key Concepts:
  - Mark Price: ราคา perp ปัจจุบัน (จาก oracle + funding)
  - Index Price: ราคา spot ของ underlying asset
  - Funding Rate: ค่า fee ที่ longs จ่าย shorts (หรือกลับกัน)
    เพื่อให้ mark price ≈ index price
  - Open Interest: total long + short positions
  - Leverage: amplify exposure (e.g., 10x)
  - Margin: collateral backing positions
  - Liquidation: margin < maintenance requirement

Funding Rate = (Mark - Index) / Index * funding_factor
  หาก positive: longs จ่าย shorts
  หาก negative: shorts จ่าย longs
  อัตราใช้ทุก 8 ชั่วโมง (Binance style)
```

---

## Funding Rate Mechanism

```move
module perp::funding {
    use aptos_framework::timestamp;
    
    // ============================================
    // Funding Rate calculation and payment
    // ============================================
    
    struct FundingState has key {
        // Cumulative funding index (per long unit)
        // Increases when longs pay shorts, decreases when shorts pay longs
        long_funding_index: I128,    // 18 decimal fixed point
        short_funding_index: I128,
        
        // Last funding calculation time
        last_funding_time: u64,
        
        // Current rates
        funding_rate_bps: I64,       // current 8h funding rate
        
        // Config
        funding_interval: u64,       // 8 hours = 28800 seconds
        max_funding_rate_bps: u64,   // e.g., 75 bps = 0.75% per 8h
        funding_factor: u64,         // how fast mark converges to index
    }
    
    // Use signed integers via wrapper
    struct I128 has copy, drop, store {
        value: u128,
        negative: bool,
    }
    
    struct I64 has copy, drop, store {
        value: u64,
        negative: bool,
    }
    
    const PRECISION: u128 = 1_000_000_000_000_000_000;  // 1e18
    const E_FUNDING_NOT_DUE: u64 = 1;
    
    public fun accrue_funding(
        state_addr: address,
        mark_price: u64,
        index_price: u64,
        open_interest_long: u64,
        open_interest_short: u64,
    ) acquires FundingState {
        let state = borrow_global_mut<FundingState>(state_addr);
        let now = timestamp::now_seconds();
        
        assert!(now >= state.last_funding_time + state.funding_interval, E_FUNDING_NOT_DUE);
        
        // Calculate funding rate
        // rate = clamp((mark - index) / index * 100, -max, max)
        let rate_bps = calculate_funding_rate(
            mark_price,
            index_price,
            state.max_funding_rate_bps,
        );
        
        state.funding_rate_bps = rate_bps;
        state.last_funding_time = now;
        
        // Update cumulative indices
        // If rate positive: longs pay shorts
        // funding_per_long = rate_bps * mark_price / 10000
        // long_index += funding_per_long (longs owe this)
        // short_index += funding_per_long * open_interest_long / open_interest_short (shorts receive)
        
        if (open_interest_long > 0 && open_interest_short > 0) {
            let funding_per_long_unit = (mark_price as u128) 
                * (rate_bps.value as u128) 
                / 10_000u128;
            
            let funding_per_short_unit = funding_per_long_unit 
                * (open_interest_long as u128) 
                / (open_interest_short as u128);
            
            if (!rate_bps.negative) {
                // Positive rate: longs pay
                state.long_funding_index = i128_add(
                    state.long_funding_index, 
                    I128 { value: funding_per_long_unit, negative: false }
                );
                state.short_funding_index = i128_add(
                    state.short_funding_index,
                    I128 { value: funding_per_short_unit, negative: false }
                );
            } else {
                // Negative rate: shorts pay
                state.short_funding_index = i128_add(
                    state.short_funding_index,
                    I128 { value: funding_per_long_unit, negative: true }
                );
                state.long_funding_index = i128_add(
                    state.long_funding_index,
                    I128 { value: funding_per_short_unit, negative: true }
                );
            };
        };
    }
    
    fun calculate_funding_rate(mark: u64, index: u64, max_bps: u64): I64 {
        if (mark == index) return I64 { value: 0, negative: false };
        
        let (diff, negative) = if (mark > index) {
            (mark - index, false)
        } else {
            (index - mark, true)
        };
        
        let rate_bps = diff * 10_000 / index;
        let clamped = if (rate_bps > max_bps) { max_bps } else { rate_bps };
        
        I64 { value: clamped, negative }
    }
    
    fun i128_add(a: I128, b: I128): I128 {
        if (a.negative == b.negative) {
            I128 { value: a.value + b.value, negative: a.negative }
        } else if (a.value >= b.value) {
            I128 { value: a.value - b.value, negative: a.negative }
        } else {
            I128 { value: b.value - a.value, negative: b.negative }
        }
    }
    
    #[view]
    public fun current_funding_rate(state_addr: address): (u64, bool) acquires FundingState {
        let state = borrow_global<FundingState>(state_addr);
        (state.funding_rate_bps.value, state.funding_rate_bps.negative)
    }
}
```

---

## Position Management

```move
module perp::positions {
    use std::signer;
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::timestamp;
    use aptos_framework::event;
    
    // ============================================
    // Perpetual Position management
    // ============================================
    
    struct PerpConfig has key {
        admin: address,
        oracle_addr: address,
        collateral_token: address,
        
        // Risk parameters
        max_leverage: u64,         // e.g., 20 (20x)
        maintenance_margin_bps: u64,  // e.g., 500 = 5%
        liquidation_fee_bps: u64,     // e.g., 100 = 1% to liquidator
        
        // Fee parameters  
        open_fee_bps: u64,
        close_fee_bps: u64,
        
        // State
        open_interest_long: u64,
        open_interest_short: u64,
        total_collateral: u64,
    }
    
    struct Position has key {
        trader: address,
        size: u64,            // position size in USD (18 dec)
        collateral: u64,      // margin in collateral token
        entry_price: u64,     // price when opened (18 dec)
        is_long: bool,
        leverage: u64,        // stored leverage
        
        // Funding tracking
        funding_index_snapshot: I128,
        funding_owed: I64,    // accumulated funding payment
        
        // Timestamps
        opened_at: u64,
        last_updated: u64,
    }
    
    use perp::funding::{I128, I64};
    
    #[event]
    struct PositionOpened has drop, store {
        trader: address,
        size: u64,
        is_long: bool,
        entry_price: u64,
        leverage: u64,
    }
    
    #[event]
    struct PositionClosed has drop, store {
        trader: address,
        size: u64,
        pnl: I64,
        funding_paid: I64,
    }
    
    #[event]
    struct PositionLiquidated has drop, store {
        trader: address,
        liquidator: address,
        price: u64,
        remaining_margin: u64,
    }
    
    const E_ABOVE_MAX_LEVERAGE: u64 = 1;
    const E_INSUFFICIENT_MARGIN: u64 = 2;
    const E_POSITION_NOT_FOUND: u64 = 3;
    const E_HEALTHY: u64 = 4;
    const E_INSUFFICIENT_SIZE: u64 = 5;
    
    // ============================================
    // Open Position
    // ============================================
    
    public entry fun open_position(
        trader: &signer,
        config_addr: address,
        margin_amount: u64,
        leverage: u64,
        is_long: bool,
    ) acquires PerpConfig {
        let trader_addr = signer::address_of(trader);
        let config = borrow_global_mut<PerpConfig>(config_addr);
        
        // Validate leverage
        assert!(leverage <= config.max_leverage, E_ABOVE_MAX_LEVERAGE);
        assert!(leverage >= 1, E_INSUFFICIENT_SIZE);
        
        // Get mark price from oracle
        let mark_price = 100_000_000_000_000_000_000u64;  // placeholder: $100 (18 dec)
        
        // Calculate position size
        let size_usd = margin_amount * leverage;
        
        // Charge open fee
        let open_fee = size_usd * config.open_fee_bps / 10_000;
        let net_margin = margin_amount - open_fee / leverage;
        
        // Update open interest
        if (is_long) {
            config.open_interest_long = config.open_interest_long + size_usd;
        } else {
            config.open_interest_short = config.open_interest_short + size_usd;
        };
        config.total_collateral = config.total_collateral + net_margin;
        
        // Create position
        move_to(trader, Position {
            trader: trader_addr,
            size: size_usd,
            collateral: net_margin,
            entry_price: mark_price,
            is_long,
            leverage,
            funding_index_snapshot: I128 { value: 0, negative: false },
            funding_owed: I64 { value: 0, negative: false },
            opened_at: timestamp::now_seconds(),
            last_updated: timestamp::now_seconds(),
        });
        
        event::emit(PositionOpened {
            trader: trader_addr,
            size: size_usd,
            is_long,
            entry_price: mark_price,
            leverage,
        });
    }
    
    // ============================================
    // Close Position (full or partial)
    // ============================================
    
    public entry fun close_position(
        trader: &signer,
        config_addr: address,
        trader_addr: address,
        close_size: u64,  // size to close (0 = full)
    ) acquires PerpConfig, Position {
        let config = borrow_global_mut<PerpConfig>(config_addr);
        let position = borrow_global_mut<Position>(trader_addr);
        
        assert!(signer::address_of(trader) == trader_addr, 1);
        
        let actual_close_size = if (close_size == 0 || close_size >= position.size) {
            position.size
        } else {
            close_size
        };
        
        // Get current mark price
        let mark_price = 110_000_000_000_000_000_000u64;  // placeholder: $110
        
        // Calculate PnL
        let pnl = calculate_pnl(
            position.is_long,
            actual_close_size,
            position.entry_price,
            mark_price,
        );
        
        // Apply close fee
        let close_fee = actual_close_size * config.close_fee_bps / 10_000;
        
        // Calculate returned margin
        let margin_share = position.collateral * actual_close_size / position.size;
        
        // Final settlement = margin + pnl - fees
        // (simplified, signed arithmetic omitted)
        
        // Update state
        if (position.is_long) {
            config.open_interest_long = config.open_interest_long - actual_close_size;
        } else {
            config.open_interest_short = config.open_interest_short - actual_close_size;
        };
        
        if (actual_close_size >= position.size) {
            // Full close - remove position
            position.size = 0;
            position.collateral = 0;
        } else {
            // Partial close
            position.size = position.size - actual_close_size;
            position.collateral = position.collateral - margin_share;
        };
        
        event::emit(PositionClosed {
            trader: trader_addr,
            size: actual_close_size,
            pnl,
            funding_paid: I64 { value: 0, negative: false },
        });
    }
    
    // ============================================
    // Liquidate undercollateralized position
    // ============================================
    
    public entry fun liquidate(
        liquidator: &signer,
        config_addr: address,
        trader_addr: address,
    ) acquires PerpConfig, Position {
        let config = borrow_global<PerpConfig>(config_addr);
        let position = borrow_global_mut<Position>(trader_addr);
        
        let mark_price = 105_000_000_000_000_000_000u64;  // placeholder
        
        // Check health factor
        let margin_ratio = get_margin_ratio(position, mark_price);
        assert!(margin_ratio < config.maintenance_margin_bps, E_HEALTHY);
        
        // Calculate liquidation
        let liquidation_fee = position.collateral 
            * config.liquidation_fee_bps 
            / 10_000;
        let remaining = if (position.collateral > liquidation_fee) {
            position.collateral - liquidation_fee
        } else { 0 };
        
        // Pay liquidator
        // coin::transfer to liquidator = liquidation_fee
        
        // If remaining > 0, return to trader
        // coin::transfer to trader_addr = remaining
        
        let liquidator_addr = signer::address_of(liquidator);
        
        event::emit(PositionLiquidated {
            trader: trader_addr,
            liquidator: liquidator_addr,
            price: mark_price,
            remaining_margin: remaining,
        });
        
        position.size = 0;
        position.collateral = 0;
    }
    
    // ============================================
    // Helpers
    // ============================================
    
    fun calculate_pnl(is_long: bool, size: u64, entry: u64, current: u64): I64 {
        // PnL = size * (current - entry) / entry * is_long_sign
        let (diff, price_positive) = if (current > entry) {
            (current - entry, true)
        } else {
            (entry - current, false)
        };
        
        let pnl_value = (size as u128) * (diff as u128) / (entry as u128);
        let profit = if (is_long) price_positive else !price_positive;
        
        I64 { value: pnl_value as u64, negative: !profit }
    }
    
    fun get_margin_ratio(pos: &Position, current_price: u64): u64 {
        // margin_ratio = (collateral + pnl) / (size / leverage) * 10000
        // Simplified:
        if (pos.size == 0) return 10_000;
        
        let pnl = calculate_pnl(pos.is_long, pos.size, pos.entry_price, current_price);
        
        let effective_margin = if (!pnl.negative) {
            pos.collateral + pnl.value
        } else if (pos.collateral > pnl.value) {
            pos.collateral - pnl.value
        } else {
            return 0  // fully underwater
        };
        
        effective_margin * 10_000 / (pos.size / pos.leverage)
    }
    
    #[view]
    public fun position_health(trader_addr: address, current_price: u64): u64 
    acquires Position {
        let pos = borrow_global<Position>(trader_addr);
        get_margin_ratio(pos, current_price)
    }
}
```

---

## Vault-Based Options

```move
module perp::options_vault {
    use std::signer;
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::timestamp;
    
    // ============================================
    // European Options via Covered Call Vault
    // (Simplified Ribbon Finance style)
    // ============================================
    
    struct OptionsVault has key {
        admin: address,
        underlying_price_oracle: address,
        
        // Vault state
        total_deposited: u64,
        pending_deposit: u64,
        
        // Current option round
        current_round: u64,
        strike_price: u64,       // Strike in USD (18 dec)
        expiry: u64,             // Unix timestamp
        is_call: bool,           // true=call, false=put
        
        // Round results
        round_premium_collected: u64,
        round_expired_itm: bool,  // In-The-Money at expiry
    }
    
    struct UserShare has key {
        vault_addr: address,
        shares: u64,
        deposited_at_round: u64,
    }
    
    const PRECISION: u64 = 1_000_000_000_000_000_000;  // 1e18
    
    // ============================================
    // Deposit into vault
    // ============================================
    
    public entry fun deposit(
        user: &signer,
        vault_addr: address,
        amount: u64,
    ) acquires OptionsVault, UserShare {
        let vault = borrow_global_mut<OptionsVault>(vault_addr);
        let user_addr = signer::address_of(user);
        
        // Accept deposit
        vault.pending_deposit = vault.pending_deposit + amount;
        
        // Issue shares (simplified: 1:1 for first deposit)
        let shares = if (vault.total_deposited == 0) {
            amount
        } else {
            amount * vault.total_deposited / vault.total_deposited // TODO: proper share math
        };
        
        if (exists<UserShare>(user_addr)) {
            let user_share = borrow_global_mut<UserShare>(user_addr);
            user_share.shares = user_share.shares + shares;
        } else {
            move_to(user, UserShare {
                vault_addr,
                shares,
                deposited_at_round: vault.current_round,
            });
        };
    }
    
    // ============================================
    // Settle expired option
    // ============================================
    
    public entry fun settle_round(
        admin: &signer,
        vault_addr: address,
        expiry_price: u64,
    ) acquires OptionsVault {
        let vault = borrow_global_mut<OptionsVault>(vault_addr);
        assert!(signer::address_of(admin) == vault.admin, 1);
        assert!(timestamp::now_seconds() >= vault.expiry, 2);
        
        // Check if expired ITM
        vault.round_expired_itm = if (vault.is_call) {
            expiry_price > vault.strike_price
        } else {
            expiry_price < vault.strike_price
        };
        
        if (vault.round_expired_itm) {
            // Option exercised: vault pays out (collateral taken)
            let payout = if (vault.is_call) {
                // Call payout = max(spot - strike, 0) per contract
                let diff = expiry_price - vault.strike_price;
                vault.total_deposited * diff / vault.strike_price
            } else {
                let diff = vault.strike_price - expiry_price;
                vault.total_deposited * diff / vault.strike_price
            };
            vault.total_deposited = if (vault.total_deposited > payout) {
                vault.total_deposited - payout
            } else { 0 };
        };
        // If OTM: vault keeps all collateral + premium
        
        vault.current_round = vault.current_round + 1;
    }
    
    // ============================================
    // Black-Scholes approximation (integer math)
    // For reference only - production uses oracle
    // ============================================
    
    // d1 = (ln(S/K) + (r + σ²/2)*T) / (σ*sqrt(T))
    // Approximation using Taylor series (integer, 4 decimal precision)
    
    fun approx_call_premium(
        spot: u64,     // in USD, 18 dec
        strike: u64,   // in USD, 18 dec
        vol_bps: u64,  // annualized vol in bps (e.g., 8000 = 80%)
        days_to_expiry: u64,
    ): u64 {
        // Simplified: premium ≈ 0.4 * σ * sqrt(T/365) * S
        // For ATM options as approximation
        
        let vol = vol_bps;  // in bps
        
        // sqrt(days/365) approximation
        // Using integer newton's method
        let t_ratio = days_to_expiry * 10_000 / 365;
        let sqrt_t = integer_sqrt(t_ratio);  // in units where 100 = 1.0
        
        // premium = 0.4 * vol * sqrt_t * spot / (10000 * 100)
        let premium = spot / 10_000 * 4 * vol / 10_000 * sqrt_t / 100;
        premium
    }
    
    fun integer_sqrt(n: u64): u64 {
        if (n == 0) return 0;
        let mut x = n;
        let mut y = (x + 1) / 2;
        while (y < x) {
            x = y;
            y = (x + n / x) / 2;
        };
        x
    }
    
    #[view]
    public fun estimated_premium(
        spot_usd: u64,
        strike_usd: u64,
        vol_bps: u64,
        days: u64,
    ): u64 {
        approx_call_premium(spot_usd, strike_usd, vol_bps, days)
    }
}
```

---

## สรุป Advanced DeFi

```
Protocol           | Complexity | Key Challenge
------------------|------------|------------------
AMM (v2)          | Medium     | LP impermanent loss
Lending           | High       | Oracle risk, liquidation
Perp Futures      | Very High  | Funding, socialized loss
Options Vaults    | High       | Black-Scholes pricing
Structured Prods  | Very High  | Composability risk
```

**Risk Framework:**
1. **Oracle Risk**: ราคา manipulate ได้ → ใช้ TWAP + circuit breaker
2. **Liquidation Risk**: ตลาดเคลื่อนเร็ว → sufficient margin buffer
3. **Funding Rate Risk**: extreme funding → cap + insurance fund
4. **Smart Contract Risk**: logic bugs → formal verification, audits
5. **Composability Risk**: inter-protocol dependencies → exposure limits

---

**ก่อนหน้า**: [Part 34 - Governance ←](part-34-governance.md)
**ต่อไป**: [Part 36 - Cross-chain Bridges →](part-36-cross-chain.md)
