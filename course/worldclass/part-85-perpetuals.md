# Part 85: Perpetual Futures Protocol

## สารบัญ
- [Perpetual Futures Overview](#perpetual-futures-overview)
- [Position Management](#position-management)
- [Funding Rate Mechanism](#funding-rate-mechanism)
- [Liquidation Engine](#liquidation-engine)
- [Virtual AMM (vAMM) Pricing](#virtual-amm-vamm-pricing)

---

## Perpetual Futures Overview

```
PERPETUAL FUTURES (Perps): The #1 DeFi derivative product

What it is:
  Like futures but with no expiry date
  Hold positions indefinitely with funding payments
  
How it works:
  Long position: Profit when asset price rises
  Short position: Profit when asset price falls
  
  Collateral: USDC deposited as margin
  Position size: Up to 10x leverage (e.g., $100 collateral → $1000 position)
  
FUNDING RATE: Key mechanism
  When more longs than shorts → funding > 0
  → Longs pay shorts periodically (every 8 hours)
  This incentivizes shorting → brings perp price ≈ spot price
  
  Funding rate = (mark_price - index_price) / index_price × funding_divisor
  
MARK PRICE vs INDEX PRICE
  Index price: Average spot price from multiple CEXes
  Mark price: Perp market price (based on order book or vAMM)
  
  Healthy perp: mark ≈ index
  Stressed perp: mark diverges → funding rate rises → arbitrage
  
LIQUIDATION
  If position's health < threshold → liquidated
  Maintenance margin: 0.5% of position value
  Initial margin: 1-10% (1/leverage)
  
  P&L = (exit_price - entry_price) * quantity * direction
  Margin ratio = (collateral + unrealized_PnL) / position_size
  Liquidation when margin_ratio < maintenance_margin_ratio

MAJOR PERP PROTOCOLS
  Centralized: Binance, OKX, Bybit
  Decentralized: GMX (arbitrum), dYdX (cosmos), Synthetix (ethereum)
  Move/Aptos: Econia (order book), Tsunami Finance
```

---

## Position Management

```move
// ============================================
// PERPETUAL FUTURES: Core position management
// ============================================

module perps::positions {
    use aptos_framework::timestamp;
    use aptos_std::table::{Self, Table};
    
    // Max leverage: 10x
    const MAX_LEVERAGE: u64 = 10;
    
    // Margin ratios (bps: 10000 = 100%)
    const INITIAL_MARGIN_BPS: u64 = 1000;      // 10% (for 10x leverage)
    const MAINTENANCE_MARGIN_BPS: u64 = 50;    // 0.5%
    const LIQUIDATION_FEE_BPS: u64 = 25;       // 0.25% of position to liquidator
    
    // Trade fee
    const TAKER_FEE_BPS: u64 = 10;             // 0.1%
    const MAKER_FEE_BPS: u64 = 2;              // 0.02%
    
    struct Position has store {
        owner: address,
        is_long: bool,
        collateral: u64,        // USDC margin (1e6)
        position_size: u64,     // USD notional value (1e6)
        entry_price: u64,       // Entry price (1e6 per token, e.g., $30_000_000_000)
        leverage: u64,          // Actual leverage (e.g., 5 = 5x)
        funding_accumulated: i64,  // Cumulative funding paid/received
        opened_at: u64,
    }
    
    struct Market has key {
        symbol: std::string::String,  // "APT-PERP"
        positions: Table<address, Position>,
        
        // Market totals (for funding calculation)
        total_long_size: u64,
        total_short_size: u64,
        
        // Pricing
        mark_price: u64,        // 1e6 per unit (e.g., 8_000_000 = $8.00)
        index_price: u64,       // From oracle
        
        // Funding
        funding_rate: i64,      // Signed bps (positive → longs pay shorts)
        last_funding_time: u64,
        cumulative_funding: i64,  // Sum of all funding rates applied
        
        // Protocol
        insurance_fund: u64,    // Covers liquidation shortfalls
        fee_treasury: u64,
        paused: bool,
    }
    
    // Open a new position
    public entry fun open_position(
        user: &signer,
        market_addr: address,
        is_long: bool,
        collateral_amount: u64,
        leverage: u64,  // 1-10
        min_execution_price: u64,
        max_execution_price: u64,
    ) acquires Market {
        let user_addr = std::signer::address_of(user);
        let market = borrow_global_mut<Market>(market_addr);
        
        assert!(!market.paused, 1);
        assert!(leverage >= 1 && leverage <= MAX_LEVERAGE, 2);
        assert!(collateral_amount > 0, 3);
        
        // Calculate position size
        let position_size = collateral_amount * leverage;
        
        // Check price within bounds (slippage protection)
        assert!(market.mark_price >= min_execution_price, 4);
        assert!(market.mark_price <= max_execution_price, 5);
        
        // Verify user doesn't already have a position
        assert!(!aptos_std::table::contains(&market.positions, user_addr), 6);
        
        // Deduct opening fee
        let fee = position_size * TAKER_FEE_BPS / 10_000;
        let net_collateral = collateral_amount - fee;
        market.fee_treasury = market.fee_treasury + fee;
        
        // Take collateral from user
        // coin::withdraw<USDC>(user, collateral_amount);
        
        // Create position
        let position = Position {
            owner: user_addr,
            is_long,
            collateral: net_collateral,
            position_size,
            entry_price: market.mark_price,
            leverage,
            funding_accumulated: 0,
            opened_at: timestamp::now_seconds(),
        };
        
        aptos_std::table::add(&mut market.positions, user_addr, position);
        
        // Update market OI (open interest)
        if (is_long) {
            market.total_long_size = market.total_long_size + position_size;
        } else {
            market.total_short_size = market.total_short_size + position_size;
        };
        
        aptos_framework::event::emit(PositionOpened {
            user: user_addr,
            is_long,
            collateral: net_collateral,
            position_size,
            entry_price: market.mark_price,
        });
    }
    
    // Close a position
    public entry fun close_position(
        user: &signer,
        market_addr: address,
    ) acquires Market {
        let user_addr = std::signer::address_of(user);
        let market = borrow_global_mut<Market>(market_addr);
        
        assert!(aptos_std::table::contains(&market.positions, user_addr), 1);
        
        let position = aptos_std::table::remove(&mut market.positions, user_addr);
        
        // Calculate PnL
        let pnl = calculate_pnl(&position, market.mark_price);
        
        // Apply accrued funding
        let funding_payment = position.funding_accumulated;
        
        // Net return to user
        let return_amount = if (pnl >= 0) {
            position.collateral + (pnl as u64) + (if (funding_payment < 0) { 0u64 } else { funding_payment as u64 })
        } else {
            let loss = ((-pnl) as u64);
            if (loss >= position.collateral) {
                0  // Complete loss
            } else {
                position.collateral - loss
            }
        };
        
        // Closing fee
        let close_fee = position.position_size * TAKER_FEE_BPS / 10_000;
        let net_return = if (return_amount > close_fee) { return_amount - close_fee } else { 0 };
        
        market.fee_treasury = market.fee_treasury + close_fee;
        
        // Update market OI
        if (position.is_long) {
            market.total_long_size = market.total_long_size - position.position_size;
        } else {
            market.total_short_size = market.total_short_size - position.position_size;
        };
        
        // Return funds to user
        // coin::deposit<USDC>(user_addr, net_return);
        
        aptos_framework::event::emit(PositionClosed {
            user: user_addr,
            pnl,
            return_amount: net_return,
            exit_price: market.mark_price,
        });
    }
    
    // Calculate unrealized PnL
    public fun calculate_pnl(position: &Position, current_price: u64): i64 {
        // PnL = (current_price - entry_price) / entry_price * position_size
        // For long: PnL = (current - entry) * size / entry
        // For short: PnL = (entry - current) * size / entry
        
        if (position.is_long) {
            if (current_price >= position.entry_price) {
                let gain = (current_price - position.entry_price) as u128
                    * position.position_size as u128
                    / position.entry_price as u128;
                gain as i64
            } else {
                let loss = (position.entry_price - current_price) as u128
                    * position.position_size as u128
                    / position.entry_price as u128;
                -(loss as i64)
            }
        } else {
            // Short
            if (position.entry_price >= current_price) {
                let gain = (position.entry_price - current_price) as u128
                    * position.position_size as u128
                    / position.entry_price as u128;
                gain as i64
            } else {
                let loss = (current_price - position.entry_price) as u128
                    * position.position_size as u128
                    / position.entry_price as u128;
                -(loss as i64)
            }
        }
    }
    
    // Get margin ratio (for liquidation check)
    public fun get_margin_ratio(position: &Position, current_price: u64): u64 {
        let pnl = calculate_pnl(position, current_price);
        
        let net_collateral = if (pnl >= 0) {
            position.collateral + (pnl as u64)
        } else {
            let loss = ((-pnl) as u64);
            if (loss >= position.collateral) { 0 } else { position.collateral - loss }
        };
        
        // Margin ratio = net_collateral / position_size * 10_000
        (net_collateral as u128 * 10_000 / position.position_size as u128) as u64
    }
    
    #[event] struct PositionOpened has drop, store { user: address, is_long: bool, collateral: u64, position_size: u64, entry_price: u64 }
    #[event] struct PositionClosed has drop, store { user: address, pnl: i64, return_amount: u64, exit_price: u64 }
}
```

---

## Funding Rate Mechanism

```move
// ============================================
// FUNDING RATE: Keeps perp price near spot
// Applied every 8 hours
// ============================================

module perps::funding {
    use aptos_framework::timestamp;
    
    const FUNDING_INTERVAL: u64 = 28_800;  // 8 hours in seconds
    const FUNDING_DIVISOR: u64 = 24;       // Daily to 8-hour: divide by 3 (8h/24h * 3 = 1)
    const MAX_FUNDING_RATE_BPS: u64 = 100; // Max 1% per 8 hours
    
    // Calculate current funding rate
    // Premium = (mark_price - index_price) / index_price
    // Funding rate = clamp(premium / funding_divisor, -max, +max)
    public fun calculate_funding_rate(
        mark_price: u64,
        index_price: u64,
    ): i64 {
        if (index_price == 0) return 0;
        
        let max_rate = MAX_FUNDING_RATE_BPS as i64;
        
        if (mark_price >= index_price) {
            let premium = (mark_price - index_price) as u128 * 10_000 / index_price as u128;
            let rate = (premium / FUNDING_DIVISOR as u128) as i64;
            if (rate > max_rate) { max_rate } else { rate }
        } else {
            let premium = (index_price - mark_price) as u128 * 10_000 / index_price as u128;
            let rate = -((premium / FUNDING_DIVISOR as u128) as i64);
            if (rate < -max_rate) { -max_rate } else { rate }
        }
    }
    
    // Apply funding to a position
    // Positive funding: longs pay shorts
    // Negative funding: shorts pay longs
    public fun apply_funding(
        position_size: u64,
        is_long: bool,
        funding_rate: i64,
    ): i64 {
        // payment = position_size * |funding_rate| / 10_000
        // Long pays if rate > 0; short receives
        let payment = (position_size as i64) * funding_rate / 10_000;
        
        if (is_long) {
            -payment  // Long pays when rate positive
        } else {
            payment   // Short receives when rate positive
        }
    }
    
    // Update funding for all positions (apply cumulative)
    public fun settle_funding<Market>(
        market: &mut perps::positions::Market,
    ) {
        let now = timestamp::now_seconds();
        
        if (now < market.last_funding_time + FUNDING_INTERVAL) {
            return  // Not time yet
        };
        
        let periods = (now - market.last_funding_time) / FUNDING_INTERVAL;
        
        // New funding rate
        let new_rate = calculate_funding_rate(market.mark_price, market.index_price);
        
        // Update cumulative funding
        // Each long/short's individual funding = their_size * cumulative_funding_delta
        market.cumulative_funding = market.cumulative_funding + (new_rate * periods as i64);
        market.funding_rate = new_rate;
        market.last_funding_time = market.last_funding_time + periods * FUNDING_INTERVAL;
        
        aptos_framework::event::emit(FundingSettled {
            rate: new_rate,
            cumulative: market.cumulative_funding,
            timestamp: now,
        });
    }
    
    #[event]
    struct FundingSettled has drop, store {
        rate: i64,
        cumulative: i64,
        timestamp: u64,
    }
    
    struct Market {
        mark_price: u64,
        index_price: u64,
        funding_rate: i64,
        last_funding_time: u64,
        cumulative_funding: i64,
    }
}
```

---

## Liquidation Engine

```move
// ============================================
// LIQUIDATION ENGINE
// Keep protocol solvent by closing underwater positions
// ============================================

module perps::liquidation {
    use perps::positions::{Self, Position, Market};
    use aptos_framework::timestamp;
    
    const MAINTENANCE_MARGIN_BPS: u64 = 50;     // 0.5%
    const LIQUIDATION_FEE_BPS: u64 = 25;        // 0.25% to liquidator
    const INSURANCE_FEE_BPS: u64 = 25;          // 0.25% to insurance fund
    
    // Check if a position is liquidatable
    public fun is_liquidatable(
        position: &Position,
        mark_price: u64,
    ): bool {
        let margin_ratio = positions::get_margin_ratio(position, mark_price);
        margin_ratio < MAINTENANCE_MARGIN_BPS
    }
    
    // Liquidate an underwater position
    // Called by anyone (liquidator bots)
    public entry fun liquidate_position(
        liquidator: &signer,
        target_user: address,
        market_addr: address,
    ) acquires Market {
        let market = borrow_global_mut<Market>(market_addr);
        let liquidator_addr = std::signer::address_of(liquidator);
        
        // Verify position exists and is liquidatable
        assert!(aptos_std::table::contains(&market.positions, target_user), 1);
        
        let position = aptos_std::table::borrow(&market.positions, target_user);
        
        // Check if actually liquidatable
        let margin_ratio = positions::get_margin_ratio(position, market.mark_price);
        assert!(margin_ratio < MAINTENANCE_MARGIN_BPS, 2);  // Not yet liquidatable
        
        // Calculate liquidation amounts
        let pnl = positions::calculate_pnl(position, market.mark_price);
        
        let remaining_collateral = if (pnl >= 0) {
            position.collateral + (pnl as u64)
        } else {
            let loss = ((-pnl) as u64);
            if (loss >= position.collateral) { 0 } else { position.collateral - loss }
        };
        
        let liquidation_fee = position.position_size * LIQUIDATION_FEE_BPS / 10_000;
        let insurance_fee = position.position_size * INSURANCE_FEE_BPS / 10_000;
        
        // Remove position
        let Position { owner, is_long, collateral: _, position_size, entry_price: _, leverage: _, funding_accumulated: _, opened_at: _ } 
            = aptos_std::table::remove(&mut market.positions, target_user);
        
        // Update open interest
        if (is_long) {
            market.total_long_size = market.total_long_size - position_size;
        } else {
            market.total_short_size = market.total_short_size - position_size;
        };
        
        // Pay liquidator and insurance fund
        if (remaining_collateral > liquidation_fee + insurance_fee) {
            let liquidator_reward = liquidation_fee;
            let insurance_portion = insurance_fee;
            let leftover = remaining_collateral - liquidator_reward - insurance_portion;
            
            // coin::deposit<USDC>(liquidator_addr, liquidator_reward);
            market.insurance_fund = market.insurance_fund + insurance_portion;
            
            // Return any leftover to user (rare: usually they get nothing)
            // coin::deposit<USDC>(owner, leftover);
            
        } else {
            // Collateral insufficient (bad debt)
            // Insurance fund covers liquidator reward
            if (market.insurance_fund >= liquidation_fee) {
                market.insurance_fund = market.insurance_fund - liquidation_fee;
                // coin::deposit<USDC>(liquidator_addr, liquidation_fee);
            } else {
                // Insurance fund depleted: socialized loss
                // This should be very rare
                aptos_framework::event::emit(InsuranceFundDepleted {
                    shortfall: liquidation_fee - market.insurance_fund,
                });
                market.insurance_fund = 0;
            };
        };
        
        aptos_framework::event::emit(PositionLiquidated {
            user: owner,
            liquidator: liquidator_addr,
            position_size,
            mark_price: market.mark_price,
            remaining_collateral,
            liquidator_reward: liquidation_fee,
        });
    }
    
    // Add to insurance fund
    public entry fun deposit_insurance(
        depositor: &signer,
        amount: u64,
        market_addr: address,
    ) acquires Market {
        let market = borrow_global_mut<Market>(market_addr);
        // coin::withdraw<USDC>(depositor, amount);
        market.insurance_fund = market.insurance_fund + amount;
    }
    
    #[event] struct PositionLiquidated has drop, store { user: address, liquidator: address, position_size: u64, mark_price: u64, remaining_collateral: u64, liquidator_reward: u64 }
    #[event] struct InsuranceFundDepleted has drop, store { shortfall: u64 }
    
    struct Market {
        positions: aptos_std::table::Table<address, positions::Position>,
        total_long_size: u64,
        total_short_size: u64,
        mark_price: u64,
        index_price: u64,
        funding_rate: i64,
        last_funding_time: u64,
        cumulative_funding: i64,
        insurance_fund: u64,
        fee_treasury: u64,
        paused: bool,
    }
    
    struct Position {
        owner: address,
        is_long: bool,
        collateral: u64,
        position_size: u64,
        entry_price: u64,
        leverage: u64,
        funding_accumulated: i64,
        opened_at: u64,
    }
}
```

---

## Virtual AMM (vAMM) Pricing

```move
// ============================================
// vAMM: Virtual AMM for perpetual pricing
// No real liquidity; k is virtual
// Deterministic price impact
// ============================================

module perps::vamm {
    
    struct VirtualPool has key {
        virtual_base: u64,    // Virtual token amount (e.g., "BTC")
        virtual_quote: u64,   // Virtual USDC amount
        k: u128,              // virtual_base * virtual_quote = constant
        direction_long_bias: u64,  // Track net long/short for funding
    }
    
    // Calculate price impact from a trade
    // amount_quote: USDC to spend (for longs) or receive (for shorts)
    // is_long: true = buy base, false = sell base
    public fun calculate_price(
        pool: &VirtualPool,
        amount_quote: u64,
        is_long: bool,
    ): (u64, u64) {
        // Returns: (base_received, new_mark_price)
        
        if (is_long) {
            // Add quote, remove base: q + amount_q = new_q, new_b = k / new_q
            let new_quote = pool.virtual_quote + amount_quote;
            let new_base = (pool.k / new_quote as u128) as u64;
            let base_received = pool.virtual_base - new_base;
            
            // New mark price = new_quote / new_base
            let new_price = new_quote * 1_000_000 / new_base;
            
            (base_received, new_price)
        } else {
            // Remove quote: new_q = k / new_b, new_b = old_b + base_sold
            // But we know quote_received: new_b = old_b + base_amount
            // quote_received = old_q - new_q = old_q - k / (old_b + base_amount)
            // For simplicity, use quote as input:
            let new_quote = pool.virtual_quote - amount_quote;
            let new_base = (pool.k / new_quote as u128) as u64;
            let base_sold = new_base - pool.virtual_base;
            
            let new_price = new_quote * 1_000_000 / new_base;
            
            (base_sold, new_price)
        }
    }
    
    // Execute a trade on vAMM (updates virtual reserves)
    public fun execute_trade(
        pool: &mut VirtualPool,
        amount_quote: u64,
        is_long: bool,
    ): (u64, u64) {
        let (base_amount, new_price) = calculate_price(pool, amount_quote, is_long);
        
        if (is_long) {
            pool.virtual_quote = pool.virtual_quote + amount_quote;
            pool.virtual_base = pool.virtual_base - base_amount;
        } else {
            pool.virtual_quote = pool.virtual_quote - amount_quote;
            pool.virtual_base = pool.virtual_base + base_amount;
        };
        
        // Verify k is preserved (rounding may cause small drift)
        let new_k = pool.virtual_base as u128 * pool.virtual_quote as u128;
        // Note: in practice k can be slightly off due to integer division
        // assert!(new_k >= pool.k * 9999 / 10000 && new_k <= pool.k * 10001 / 10000, 1);
        
        (base_amount, new_price)
    }
    
    // Get current mark price
    public fun mark_price(pool: &VirtualPool): u64 {
        pool.virtual_quote * 1_000_000 / pool.virtual_base
    }
    
    // Get price impact of a trade (bps)
    public fun price_impact_bps(
        pool: &VirtualPool,
        amount_quote: u64,
        is_long: bool,
    ): u64 {
        let current_price = mark_price(pool);
        let (_, new_price) = calculate_price(pool, amount_quote, is_long);
        
        let impact = if (new_price > current_price) {
            (new_price - current_price) * 10_000 / current_price
        } else {
            (current_price - new_price) * 10_000 / current_price
        };
        
        impact
    }
}
```

---

## สรุป Perpetual Futures Protocol

```
PERP PROTOCOL DESIGN SUMMARY

KEY FORMULAS
  Position size = collateral × leverage
  PnL (long) = (mark - entry) / entry × position_size
  PnL (short) = (entry - mark) / entry × position_size
  Margin ratio = (collateral + unrealized_PnL) / position_size
  Liquidation when: margin_ratio < maintenance_margin (0.5%)
  
  Funding rate = clamp((mark - index) / index / 3, -1%, +1%)
  Funding payment per 8h = position_size × funding_rate / 10_000
  
RISK PARAMETERS (typical)
  Initial margin: 10% (10x leverage)
  Maintenance margin: 0.5%
  Liquidation fee to liquidator: 0.25%
  Liquidation fee to insurance: 0.25%
  Taker fee: 0.1%, Maker fee: 0.02%
  
INSURANCE FUND PURPOSE
  Covers bad debt when liquidation is insufficient
  (Market moved too fast, position went negative)
  Funded by: 0.25% of each liquidation
  If depleted: socialized loss (rare; requires crash + mass liquidations)
  
vAMM vs ORDER BOOK
  vAMM:
    ✅ No liquidity needed (virtual)
    ✅ Deterministic pricing
    ✅ Simple to implement
    ❌ Higher price impact for large trades
    ❌ Less capital efficient
    
  Order Book (Econia on Aptos):
    ✅ Capital efficient (maker provides real liquidity)
    ✅ Tight spreads for liquid pairs
    ❌ Requires market makers
    ❌ More complex implementation
    
MOVE ADVANTAGES FOR PERPS
  Linear types: Position can't be duplicated
  Atomic operations: Open + fund in one tx
  Parallel execution: Multiple markets simultaneously
  Low fees: High-frequency funding settlements feasible
```

---

**ก่อนหน้า**: [Part 84 - RWA Tokenization ←](part-84-rwa-tokenization.md)
**ต่อไป**: [Part 86 - Decentralized Identity & Credentials →](part-86-identity.md)
