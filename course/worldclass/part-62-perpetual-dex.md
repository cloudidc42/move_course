# Part 62: Perpetual DEX Architecture

## สารบัญ
- [Perpetual Futures Fundamentals](#perpetual-futures-fundamentals)
- [Virtual AMM (vAMM) Architecture](#virtual-amm-vamm-architecture)
- [Order Book Perps Architecture](#order-book-perps-architecture)
- [Funding Rate Mechanism](#funding-rate-mechanism)
- [Liquidation Engine](#liquidation-engine)
- [Full Implementation: Move Perps](#full-implementation-move-perps)

---

## Perpetual Futures Fundamentals

```
Perpetual Futures (Perps):
  - Futures contracts with NO expiry date
  - Settle via funding rates (not physical delivery)
  - Allow leverage up to 100x
  - Most popular DeFi derivative

Key Concepts:
  Mark Price: Index price + premium/discount
    = used for PnL calculation (not manipulable)
    = prevents isolated market manipulation
    
  Index Price: Weighted average from CEX/oracles
    = BTCUSDT on Binance, Coinbase, etc.
    
  Funding Rate: Balance between longs and shorts
    = If more longs → longs pay shorts
    = If more shorts → shorts pay longs
    = Keeps perp price close to spot
    
  Margin: Collateral deposited
    Initial Margin: to open position (e.g., 10% for 10x)
    Maintenance Margin: minimum to keep position (e.g., 5%)
    
  Liquidation: Position closed when margin < maintenance
    
PnL Calculation:
  Long PnL = (exit_price - entry_price) * size
  Short PnL = (entry_price - exit_price) * size
  
  Realized PnL: after closing
  Unrealized PnL: mark_price change (for risk)

Architecture Types:
  1. vAMM (Virtual AMM): 
     - No real liquidity pool
     - Virtual x*y=k determines price
     - Pros: permissionless, always liquid
     - Cons: price diverges, funding needed
     
  2. Order Book:
     - Like CEX
     - Off-chain order matching, on-chain settlement
     - Pros: price efficiency, flexible
     - Cons: complex, centralized matching
     
  3. LP-based (GMX style):
     - LPs provide liquidity, earn fees
     - Traders vs LPs (zero-sum)
     - Pros: deep liquidity, simple
     - Cons: LPs take directional risk
```

---

## Virtual AMM (vAMM) Architecture

```move
module perps::vamm {
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::coin;
    
    // ============================================
    // vAMM: Virtual x*y=k for perps pricing
    // No real assets in the AMM
    // Positions stored separately, AMM just determines price
    // ============================================
    
    struct VirtualPool has key {
        // Virtual reserves (NOT real tokens)
        virtual_base: u128,   // e.g., BTC (scaled 1e18)
        virtual_quote: u128,  // e.g., USDC (scaled 1e6)
        // k = virtual_base * virtual_quote
        
        // Real collateral pool (actual USDC from traders)
        collateral_pool: u64,
        
        // Position tracking
        open_interest_long: u128,  // Total notional of long positions
        open_interest_short: u128, // Total notional of short positions
        
        // Fees
        trading_fee_bps: u64,    // 10 = 0.1%
        liquidation_fee_bps: u64, // 100 = 1%
        
        // Index price oracle
        oracle_addr: address,
        
        // Funding
        funding_rate: i64,           // Per hour (signed, scaled 1e10)
        last_funding_time: u64,
        
        // Position management
        positions: SmartTable<address, Position>,
        next_position_id: u64,
        
        // Leverage limits
        max_leverage: u64,  // 100 = 100x
        
        // Insurance fund
        insurance_fund: u64,
    }
    
    struct Position has store, copy, drop {
        trader: address,
        is_long: bool,
        
        size: u128,           // Position size in quote currency (USDC)
        entry_price: u64,     // Price when position was opened
        
        margin: u64,          // Collateral deposited
        leverage: u64,        // 10 = 10x
        
        // Accumulated funding payments
        funding_accumulated: i64,
        last_funding_index: u64,
        
        opened_at: u64,
    }
    
    // ============================================
    // Open position
    // ============================================
    
    public entry fun open_position(
        trader: &signer,
        pool_addr: address,
        is_long: bool,
        margin_amount: u64,  // USDC collateral
        leverage: u64,       // 1-100
        min_price: u64,      // Slippage protection
        max_price: u64,
    ) acquires VirtualPool {
        let pool = borrow_global_mut<VirtualPool>(pool_addr);
        let trader_addr = std::signer::address_of(trader);
        
        assert!(!smart_table::contains(&pool.positions, trader_addr), 1);
        assert!(leverage > 0 && leverage <= pool.max_leverage, 2);
        assert!(margin_amount > 0, 3);
        
        // Calculate position size
        let notional = (margin_amount as u128) * (leverage as u128);
        
        // Get mark price (with vAMM impact)
        let (entry_price, price_impact) = get_open_price(pool, notional, is_long);
        
        // Check slippage
        if (is_long) {
            assert!(entry_price <= max_price, 4);
        } else {
            assert!(entry_price >= min_price, 4);
        };
        
        // Collect margin + trading fee
        let fee = (notional as u64) * pool.trading_fee_bps / 10_000;
        let total_cost = margin_amount + fee;
        
        let payment = coin::withdraw<USDC>(trader, total_cost);
        coin::deposit(pool_addr, payment);
        
        pool.collateral_pool = pool.collateral_pool + margin_amount;
        pool.insurance_fund = pool.insurance_fund + fee / 2;
        
        // Update virtual reserves
        if (is_long) {
            // Buying base with quote: base decreases, quote increases
            pool.virtual_quote = pool.virtual_quote + notional;
            pool.virtual_base = pool.virtual_base * (pool.virtual_quote - notional) / pool.virtual_quote;
            pool.open_interest_long = pool.open_interest_long + notional;
        } else {
            pool.virtual_base = pool.virtual_base + notional;
            pool.virtual_quote = pool.virtual_quote * (pool.virtual_base - notional) / pool.virtual_base;
            pool.open_interest_short = pool.open_interest_short + notional;
        };
        
        // Store position
        smart_table::add(&mut pool.positions, trader_addr, Position {
            trader: trader_addr,
            is_long,
            size: notional,
            entry_price,
            margin: margin_amount,
            leverage,
            funding_accumulated: 0,
            last_funding_index: 0,
            opened_at: aptos_framework::timestamp::now_seconds(),
        });
    }
    
    // ============================================
    // Close position
    // ============================================
    
    public entry fun close_position(
        trader: &signer,
        pool_addr: address,
    ) acquires VirtualPool {
        let pool = borrow_global_mut<VirtualPool>(pool_addr);
        let trader_addr = std::signer::address_of(trader);
        
        assert!(smart_table::contains(&pool.positions, trader_addr), 1);
        let position = smart_table::remove(&mut pool.positions, trader_addr);
        
        // Settle funding before closing
        let (exit_price, _) = get_close_price(pool, position.size, position.is_long);
        
        // Calculate PnL
        let pnl = calculate_pnl(&position, exit_price);
        
        // Close fee
        let fee = (position.size as u64) * pool.trading_fee_bps / 10_000;
        
        // Total payout
        let payout = if (pnl >= 0) {
            position.margin + (pnl as u64) - fee
        } else {
            let loss = (-pnl) as u64;
            if (loss >= position.margin) {
                0  // Wiped out
            } else {
                position.margin - loss - fee
            }
        };
        
        // Update virtual reserves (reverse of open)
        if (position.is_long) {
            pool.open_interest_long = pool.open_interest_long - position.size;
        } else {
            pool.open_interest_short = pool.open_interest_short - position.size;
        };
        
        pool.collateral_pool = pool.collateral_pool - position.margin;
        
        // Pay trader
        if (payout > 0) {
            // coin::transfer<USDC>(pool_signer, trader_addr, payout);
        };
    }
    
    // ============================================
    // Price calculation from vAMM
    // ============================================
    
    fun get_open_price(
        pool: &VirtualPool,
        notional: u128,
        is_long: bool,
    ): (u64, u64) {
        let current_price = get_spot_price(pool);
        
        // Price impact from virtual AMM
        let (new_base, new_quote) = if (is_long) {
            let new_quote = pool.virtual_quote + notional;
            let new_base = pool.virtual_base * pool.virtual_quote / new_quote;
            (new_base, new_quote)
        } else {
            let new_base = pool.virtual_base + notional;
            let new_quote = pool.virtual_base * pool.virtual_quote / new_base;
            (new_base, new_quote)
        };
        
        // Average execution price
        let avg_price = (pool.virtual_quote + new_quote) / (pool.virtual_base + new_base) as u64;
        let impact = if (avg_price > current_price) {
            avg_price - current_price
        } else {
            current_price - avg_price
        };
        
        (avg_price, impact)
    }
    
    fun get_close_price(pool: &VirtualPool, notional: u128, was_long: bool): (u64, u64) {
        get_open_price(pool, notional, !was_long)
    }
    
    fun get_spot_price(pool: &VirtualPool): u64 {
        // Price = quote / base (USDC per BTC)
        (pool.virtual_quote / pool.virtual_base) as u64
    }
    
    fun calculate_pnl(position: &Position, exit_price: u64): i64 {
        if (position.is_long) {
            (exit_price as i64 - position.entry_price as i64) * (position.size as i64) / (position.entry_price as i64)
        } else {
            (position.entry_price as i64 - exit_price as i64) * (position.size as i64) / (position.entry_price as i64)
        }
    }
    
    struct USDC has store {}
}
```

---

## Funding Rate Mechanism

```move
module perps::funding {
    
    // ============================================
    // Funding Rate: Keeps perp price close to spot
    // ============================================
    
    // Funding Rate = (Premium + Clamp) / Interval
    //
    // Premium = (Mark Price - Index Price) / Index Price
    // 
    // If premium > 0 (perp > spot): longs pay shorts
    // If premium < 0 (perp < spot): shorts pay longs
    //
    // Rate clamped between -0.1% and +0.1% per hour
    // Settlement every hour
    
    const FUNDING_INTERVAL: u64 = 3600;  // 1 hour
    const MAX_FUNDING_RATE: i64 = 100_000;  // 0.01% per interval (scaled 1e10)
    const MIN_FUNDING_RATE: i64 = -100_000;
    
    struct FundingState has key {
        // Current funding rate (signed, scaled 1e10)
        current_rate: i64,
        
        // Cumulative funding index
        // Each open position records this value at open
        // Funding owed = (current_index - position_index) * position_size
        cumulative_index: i64,
        
        last_update: u64,
        
        // TWAP of mark price and index price (for rate calculation)
        mark_price_twap: u64,
        index_price_twap: u64,
    }
    
    // Calculate funding rate for next interval
    public fun compute_funding_rate(
        mark_price_twap: u64,
        index_price_twap: u64,
    ): i64 {
        if (index_price_twap == 0) return 0;
        
        // Premium = (mark - index) / index
        // Scaled to 1e10 integer arithmetic
        let premium: i64 = ((mark_price_twap as i64 - index_price_twap as i64) * 10_000_000_000)
            / (index_price_twap as i64);
        
        // Interest rate component (typically 0.01% per interval)
        let interest: i64 = 100_000;
        
        // Total funding = premium / 8 (funding settles every hour, 8 intervals per day)
        let total = premium / 8 + interest;
        
        // Clamp to [-0.1%, +0.1%] per interval
        if (total > MAX_FUNDING_RATE) {
            MAX_FUNDING_RATE
        } else if (total < MIN_FUNDING_RATE) {
            MIN_FUNDING_RATE
        } else {
            total
        }
    }
    
    // Settle funding for a position
    public fun settle_position_funding(
        state: &FundingState,
        position_size: u128,
        is_long: bool,
        position_index: i64,
    ): i64 {
        let index_delta = state.cumulative_index - position_index;
        
        // Positive funding = longs pay (index_delta > 0 = longs pay)
        let funding_payment = (position_size as i64) * index_delta / 10_000_000_000;
        
        if (is_long) {
            -funding_payment  // Longs pay when funding positive
        } else {
            funding_payment   // Shorts receive when funding positive
        }
    }
    
    // Update funding every interval
    public entry fun update_funding(
        keeper: &signer,
        state_addr: address,
        oracle_addr: address,
    ) acquires FundingState {
        let state = borrow_global_mut<FundingState>(state_addr);
        let now = aptos_framework::timestamp::now_seconds();
        
        assert!(now >= state.last_update + FUNDING_INTERVAL, 1);
        
        let index_price = get_index_price(oracle_addr);
        let mark_price = get_mark_price(state_addr);
        
        let new_rate = compute_funding_rate(mark_price, index_price);
        state.current_rate = new_rate;
        
        // Update cumulative index
        state.cumulative_index = state.cumulative_index + new_rate;
        state.last_update = now;
    }
    
    fun get_index_price(_oracle_addr: address): u64 { 50_000_000_000 }  // $50,000
    fun get_mark_price(_pool_addr: address): u64 { 50_100_000_000 }     // $50,100
}
```

---

## Liquidation Engine

```move
module perps::liquidation {
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // Liquidation: Close underwater positions
    // ============================================
    
    struct LiquidationEngine has key {
        // Maintenance margin ratio (bps)
        // At 10x leverage: maintenance = 5% → liquidated when loss > 95% of margin
        maintenance_margin_bps: u64,
        
        // Liquidation penalty (to incentivize liquidators)
        liquidator_fee_bps: u64,    // e.g., 50 = 0.5%
        insurance_fee_bps: u64,     // e.g., 50 = 0.5%
        
        // Insurance fund
        insurance_fund: u64,
        
        // Keeper bot registry (anyone can liquidate, get fee)
        liquidation_queue: SmartTable<address, u64>,  // position → liquidation_price
    }
    
    // Check if position is liquidatable
    public fun is_liquidatable(
        margin: u64,
        unrealized_pnl: i64,
        notional: u128,
        maintenance_margin_bps: u64,
    ): bool {
        let equity = if (unrealized_pnl >= 0) {
            margin + (unrealized_pnl as u64)
        } else {
            let loss = (-unrealized_pnl) as u64;
            if (loss >= margin) return true;
            margin - loss
        };
        
        let maintenance = (notional as u64) * maintenance_margin_bps / 10_000;
        equity < maintenance
    }
    
    // Liquidate a position
    public entry fun liquidate(
        liquidator: &signer,
        engine_addr: address,
        pool_addr: address,
        trader_addr: address,
    ) acquires LiquidationEngine {
        let engine = borrow_global_mut<LiquidationEngine>(engine_addr);
        let liquidator_addr = std::signer::address_of(liquidator);
        
        // Get position (must be liquidatable)
        let position = get_position(pool_addr, trader_addr);
        let mark_price = get_mark_price(pool_addr);
        let unrealized_pnl = calculate_pnl_from_mark(&position, mark_price);
        
        assert!(
            is_liquidatable(
                position.margin,
                unrealized_pnl,
                position.size,
                engine.maintenance_margin_bps,
            ),
            1
        );
        
        // Calculate remaining equity
        let equity = if (unrealized_pnl >= 0) {
            position.margin + (unrealized_pnl as u64)
        } else {
            let loss = (-unrealized_pnl) as u64;
            if (loss >= position.margin) {
                // Insolvent! Use insurance fund
                engine.insurance_fund = engine.insurance_fund - 
                    (loss - position.margin);
                0
            } else {
                position.margin - loss
            }
        };
        
        // Distribute liquidation fees
        let liquidator_fee = equity * engine.liquidator_fee_bps / 10_000;
        let insurance_fee = equity * engine.insurance_fee_bps / 10_000;
        let trader_refund = equity - liquidator_fee - insurance_fee;
        
        engine.insurance_fund = engine.insurance_fund + insurance_fee;
        
        // Pay liquidator
        // coin::transfer<USDC>(pool_signer, liquidator_addr, liquidator_fee);
        
        // Refund remaining to trader
        // if (trader_refund > 0) coin::transfer<USDC>(pool_signer, trader_addr, trader_refund);
        
        // Close position at mark price
        close_position_for_liquidation(pool_addr, trader_addr, mark_price);
    }
    
    // Batch liquidation (gas efficient)
    public entry fun batch_liquidate(
        liquidator: &signer,
        engine_addr: address,
        pool_addr: address,
        traders: vector<address>,
    ) acquires LiquidationEngine {
        let len = std::vector::length(&traders);
        let mut i = 0u64;
        while (i < len) {
            let trader_addr = *std::vector::borrow(&traders, i);
            
            // Check if liquidatable before calling (avoid reverts)
            let position = get_position(pool_addr, trader_addr);
            let mark_price = get_mark_price(pool_addr);
            let pnl = calculate_pnl_from_mark(&position, mark_price);
            
            if (is_liquidatable(
                position.margin, pnl, position.size,
                borrow_global<LiquidationEngine>(engine_addr).maintenance_margin_bps
            )) {
                // Liquidate (in production, use try/catch equivalent)
                close_position_for_liquidation(pool_addr, trader_addr, mark_price);
            };
            
            i = i + 1;
        };
    }
    
    fun get_position(_pool: address, _trader: address): PositionInfo {
        PositionInfo { margin: 0, size: 0, entry_price: 0, is_long: true }
    }
    
    fun get_mark_price(_pool: address): u64 { 0 }
    
    fun calculate_pnl_from_mark(pos: &PositionInfo, mark_price: u64): i64 {
        if (pos.is_long) {
            (mark_price as i64 - pos.entry_price as i64) * (pos.size as i64) / (pos.entry_price as i64)
        } else {
            (pos.entry_price as i64 - mark_price as i64) * (pos.size as i64) / (pos.entry_price as i64)
        }
    }
    
    fun close_position_for_liquidation(_pool: address, _trader: address, _price: u64) {}
    
    struct PositionInfo has copy, drop {
        margin: u64,
        size: u128,
        entry_price: u64,
        is_long: bool,
    }
}
```

---

## สรุป Perpetual DEX

```
Perps DEX Architecture Summary:

Three Main Designs:
  1. vAMM (Perpetual Protocol v1)
     + Always liquid, permissionless
     - Price drift, funding needed, large impact
     
  2. Order Book (dYdX, Hyperliquid)
     + Efficient price, tight spreads
     - Complex, centralized matching engine
     
  3. LP Pool (GMX, GNS)
     + Simple, deep liquidity
     - LPs take directional risk

Risk Management:
  □ Funding rate keeps perp near index
  □ Mark price uses oracle (not manipulable)
  □ Auto-delever (ADL) for insurance fund
  □ Position limits prevent market manipulation
  □ Circuit breakers for extreme volatility

Liquidation Order:
  1. Check health factor < 1
  2. Close position at mark price
  3. Distribute: liquidator fee → insurance → trader refund
  4. If insolvent: insurance fund covers shortfall
  5. If fund depleted: Auto-deleverage counterparty positions

Key Parameters (common values):
  Initial margin: 10% (10x leverage)
  Maintenance margin: 5% (50% of initial)
  Liquidation fee: 0.5% to liquidator, 0.5% to insurance
  Funding period: 1 hour
  Max funding rate: ±0.1% per hour (±2.4% per day)
  
Gas Optimization for Perps:
  - Batch liquidations
  - Efficient position storage (pack structs)
  - Keeper network for liquidations
  - Off-chain order matching where possible
```

---

**ก่อนหน้า**: [Part 61 - Advanced Governance ←](part-61-advanced-governance.md)
**ต่อไป**: [Part 63 - Options & Derivatives →](part-63-options-derivatives.md)
