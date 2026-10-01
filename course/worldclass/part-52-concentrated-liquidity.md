# Part 52: Concentrated Liquidity AMM (CLMM)

## สารบัญ
- [Concentrated Liquidity คืออะไร](#concentrated-liquidity-คืออะไร)
- [Tick-based Architecture](#tick-based-architecture)
- [Position Management](#position-management)
- [Swap through Ticks](#swap-through-ticks)
- [Fee Collection](#fee-collection)
- [ตัวอย่าง: Uniswap V3-style CLMM in Move](#ตัวอย่าง-uniswap-v3-style-clmm-in-move)

---

## Concentrated Liquidity คืออะไร

```
Traditional AMM (Uniswap V2):
  Liquidity spread across [0, ∞) price range
  Capital efficiency: ~1%
  LPs earn fees on all trades, even at weird price ranges

Concentrated Liquidity (Uniswap V3 concept):
  LPs choose specific price range [lower, upper]
  Capital efficiency: up to 4000x better
  
  Example:
  ETH/USDC at $2000
  V2 LP: covers $0 to ∞ (most unused)
  V3 LP: covers $1800-$2200 (all capital active near current price)
  
  Mathematical foundation:
  Virtual reserves: x_virtual = x_real + L/sqrt(upper)
                    y_virtual = y_real + L*sqrt(lower)
  where L = liquidity (constant within range)
  
  Price formula: P = y_virtual / x_virtual
  
  Ticks: Price represented as sqrt(price) at discrete intervals
  tick_spacing: minimum gap between ticks (e.g., 1, 10, 60)
  
  Liquidity Ranges:
  Position = (lower_tick, upper_tick, liquidity)
  
  Active Liquidity = sum of all positions that span current tick
  
Benefits:
  ✓ Higher fee APY for concentrated LPs
  ✓ Lower slippage for traders
  ✓ Capital efficient
  
Risks:
  ✗ Higher impermanent loss when price moves out of range
  ✗ More complex to manage
  ✗ Position becomes inactive outside range
```

---

## Tick-based Architecture

```move
module clmm::ticks {
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // Tick System
    // ============================================
    
    // Tick = discrete price level
    // price = 1.0001^tick (each tick = 0.01% price change)
    // tick_spacing = min spacing between initialized ticks
    
    // Tick bitmap: efficient lookup of initialized ticks
    // For each word (128 ticks), store which are initialized
    
    struct TickBitmap has key {
        words: SmartTable<i32, u256>,  // word_pos → bitmap
    }
    
    // I128 representation (Move doesn't have native i128)
    struct I32 has copy, drop, store {
        value: u32,
        is_negative: bool,
    }
    
    // Tick info stored per initialized tick
    struct TickInfo has copy, drop, store {
        // Net liquidity change when crossing this tick
        liquidity_net: I128,
        // Total liquidity referencing this tick
        liquidity_gross: u128,
        // Fee growth outside this tick (for fee calculation)
        fee_growth_outside_x128: u256,
        fee_growth_outside_y128: u256,
    }
    
    struct I128 has copy, drop, store {
        abs_value: u128,
        is_negative: bool,
    }
    
    fun make_i128(value: u128, negative: bool): I128 {
        I128 { abs_value: value, is_negative: negative }
    }
    
    fun add_i128(a: I128, b: I128): I128 {
        if (a.is_negative == b.is_negative) {
            // Same sign: add absolute values
            I128 { abs_value: a.abs_value + b.abs_value, is_negative: a.is_negative }
        } else {
            // Different signs: subtract
            if (a.abs_value >= b.abs_value) {
                I128 { abs_value: a.abs_value - b.abs_value, is_negative: a.is_negative }
            } else {
                I128 { abs_value: b.abs_value - a.abs_value, is_negative: b.is_negative }
            }
        }
    }
    
    // ============================================
    // Tick Math: convert between ticks and sqrt prices
    // ============================================
    
    // Precision: Q64.96 fixed point
    // sqrt_price_x96 = sqrt(price) * 2^96
    
    const Q96: u128 = 79_228_162_514_264_337_593_543_950_336u128;  // 2^96
    
    // Compute sqrt_price_x96 from tick
    // Formula: sqrt(1.0001^tick) * 2^96
    public fun tick_to_sqrt_price(tick: i64): u128 {
        // Precomputed table for efficiency
        // In production: lookup table for common ticks
        // Simplified approximation:
        let abs_tick = if (tick >= 0) { tick as u128 } else { (-tick) as u128 };
        
        // sqrt(1.0001^tick) ≈ e^(tick * ln(1.0001) / 2)
        // ln(1.0001) ≈ 0.00009999500033
        // For small ticks: use binomial approximation
        
        // Simplified: return Q96 * (1 + tick/100000)^(1/2)
        let price_x96 = Q96;  // placeholder
        price_x96
    }
    
    // Compute tick from sqrt_price_x96
    public fun sqrt_price_to_tick(sqrt_price_x96: u128): i64 {
        // tick = log(price) / log(1.0001)
        // price = (sqrt_price_x96 / 2^96)^2
        0  // placeholder
    }
    
    // ============================================
    // Tick bitmap operations
    // ============================================
    
    public fun flip_tick(
        bitmap: &mut TickBitmap,
        tick: i64,
        tick_spacing: i64,
    ) {
        let (word_pos, bit_pos) = position(tick / tick_spacing);
        let mask = 1u256 << (bit_pos as u8);
        
        if (smart_table::contains(&bitmap.words, word_pos)) {
            let word = smart_table::borrow_mut(&mut bitmap.words, word_pos);
            *word = *word ^ mask;
        } else {
            smart_table::add(&mut bitmap.words, word_pos, mask);
        };
    }
    
    // Find next initialized tick in direction
    public fun next_initialized_tick(
        bitmap: &TickBitmap,
        tick: i64,
        tick_spacing: i64,
        lte: bool,  // true = search left (lower), false = search right (upper)
    ): (i64, bool) {  // (tick, initialized)
        // Find next set bit in bitmap direction
        // Simplified: return (tick, false) as placeholder
        (tick, false)
    }
    
    fun position(tick: i64): (i32, i64) {
        let word_pos = if (tick >= 0) { (tick / 256) as i32 } else { -(((-tick - 1) / 256 + 1) as i32) };
        let bit_pos = if (tick >= 0) { tick % 256 } else { 255 - ((-tick - 1) % 256) };
        (word_pos, bit_pos)
    }
    
    // Type alias for i32 (using i64 in practice)
    type i32 = i64;
    type i64 = u64;  // Simplified representation
}
```

---

## Position Management

```move
module clmm::positions {
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // LP Position Management
    // ============================================
    
    // Position key: (owner, lower_tick, upper_tick)
    struct PositionKey has copy, drop, store {
        owner: address,
        lower_tick: i64,
        upper_tick: i64,
    }
    
    struct Position has copy, drop, store {
        // Liquidity owned by this position
        liquidity: u128,
        
        // Fee growth inside range at last snapshot
        fee_growth_inside_x128_last: u256,
        fee_growth_inside_y128_last: u256,
        
        // Uncollected fees
        fees_x: u64,
        fees_y: u64,
    }
    
    struct PositionRegistry has key {
        positions: SmartTable<PositionKey, Position>,
    }
    
    // Compute amount of tokens for given liquidity in range
    public fun liquidity_to_amounts(
        liquidity: u128,
        sqrt_price_x96: u128,       // Current price
        lower_sqrt_price_x96: u128, // Lower bound
        upper_sqrt_price_x96: u128, // Upper bound
    ): (u64, u64) {  // (amount_x, amount_y)
        let amount_x: u64;
        let amount_y: u64;
        
        if (sqrt_price_x96 <= lower_sqrt_price_x96) {
            // All in token X (current price below range)
            amount_x = liquidity_to_amount_x(liquidity, lower_sqrt_price_x96, upper_sqrt_price_x96);
            amount_y = 0;
        } else if (sqrt_price_x96 < upper_sqrt_price_x96) {
            // Mixed (current price in range)
            amount_x = liquidity_to_amount_x(liquidity, sqrt_price_x96, upper_sqrt_price_x96);
            amount_y = liquidity_to_amount_y(liquidity, lower_sqrt_price_x96, sqrt_price_x96);
        } else {
            // All in token Y (current price above range)
            amount_x = 0;
            amount_y = liquidity_to_amount_y(liquidity, lower_sqrt_price_x96, upper_sqrt_price_x96);
        };
        
        (amount_x, amount_y)
    }
    
    // L * (1/sqrt_lower - 1/sqrt_upper) = L * (sqrt_upper - sqrt_lower) / (sqrt_lower * sqrt_upper)
    fun liquidity_to_amount_x(
        liquidity: u128,
        lower_sqrt_price_x96: u128,
        upper_sqrt_price_x96: u128,
    ): u64 {
        let Q96: u128 = 79_228_162_514_264_337_593_543_950_336u128;
        
        let numerator = (liquidity as u256) * (Q96 as u256)
            * ((upper_sqrt_price_x96 - lower_sqrt_price_x96) as u256);
        let denominator = (upper_sqrt_price_x96 as u256) * (lower_sqrt_price_x96 as u256);
        
        (numerator / denominator) as u64
    }
    
    // L * (sqrt_upper - sqrt_lower) / 2^96
    fun liquidity_to_amount_y(
        liquidity: u128,
        lower_sqrt_price_x96: u128,
        upper_sqrt_price_x96: u128,
    ): u64 {
        let Q96: u128 = 79_228_162_514_264_337_593_543_950_336u128;
        
        ((liquidity as u256) * ((upper_sqrt_price_x96 - lower_sqrt_price_x96) as u256)
            / (Q96 as u256)) as u64
    }
    
    // Compute liquidity from token amounts in range
    public fun amounts_to_liquidity(
        amount_x: u64,
        amount_y: u64,
        sqrt_price_x96: u128,
        lower_sqrt_price_x96: u128,
        upper_sqrt_price_x96: u128,
    ): u128 {
        let Q96: u128 = 79_228_162_514_264_337_593_543_950_336u128;
        
        if (sqrt_price_x96 <= lower_sqrt_price_x96) {
            // Below range: only X matters
            let l_x = (amount_x as u256) * (upper_sqrt_price_x96 as u256) 
                     * (lower_sqrt_price_x96 as u256)
                     / ((Q96 as u256) * ((upper_sqrt_price_x96 - lower_sqrt_price_x96) as u256));
            l_x as u128
        } else if (sqrt_price_x96 < upper_sqrt_price_x96) {
            // In range: use minimum liquidity from both
            let l_x = (amount_x as u256) * (upper_sqrt_price_x96 as u256) 
                     * (sqrt_price_x96 as u256)
                     / ((Q96 as u256) * ((upper_sqrt_price_x96 - sqrt_price_x96) as u256));
            let l_y = (amount_y as u256) * (Q96 as u256)
                     / ((sqrt_price_x96 - lower_sqrt_price_x96) as u256);
            std::math::min(l_x as u128, l_y as u128)
        } else {
            // Above range: only Y matters
            let l_y = (amount_y as u256) * (Q96 as u256)
                     / ((upper_sqrt_price_x96 - lower_sqrt_price_x96) as u256);
            l_y as u128
        }
    }
    
    type i64 = u64;  // Simplified
}
```

---

## Swap through Ticks

```move
module clmm::swap {
    
    // ============================================
    // CLMM Swap Algorithm
    // ============================================
    
    // Swap state tracks computation across tick crossings
    struct SwapState has drop {
        // Remaining input to process
        amount_remaining: u64,
        // Accumulated output so far
        amount_calculated: u64,
        // Current sqrt price
        sqrt_price_x96: u128,
        // Current tick
        tick: i64,
        // Current active liquidity
        liquidity: u128,
        // Fee growth accumulators
        fee_growth_global_x128: u256,
    }
    
    struct StepComputation has drop {
        // Tick we're stepping to
        sqrt_price_start_x96: u128,
        // Next tick
        tick_next: i64,
        // Next tick's sqrt price
        sqrt_price_next_x96: u128,
        // Amount in to reach next tick
        amount_in: u64,
        // Amount out at next tick
        amount_out: u64,
        // Fees incurred
        fee_amount: u64,
    }
    
    // Swap X for Y (price goes down = tick goes left)
    public fun swap_x_for_y(
        pool_addr: address,
        amount_in: u64,
        sqrt_price_limit_x96: u128,  // Stop if price reaches this
    ): u64 {
        let mut state = SwapState {
            amount_remaining: amount_in,
            amount_calculated: 0,
            sqrt_price_x96: get_current_sqrt_price(pool_addr),
            tick: get_current_tick(pool_addr),
            liquidity: get_current_liquidity(pool_addr),
            fee_growth_global_x128: 0,
        };
        
        // Iterate through ticks until amount consumed
        while (state.amount_remaining > 0 && state.sqrt_price_x96 != sqrt_price_limit_x96) {
            // Find next initialized tick
            let (tick_next, initialized) = find_next_tick(state.tick, true);
            let sqrt_price_next_x96 = tick_to_sqrt_price(tick_next);
            
            // Use price limit if it would be hit first
            let target_price = std::math64::max(
                sqrt_price_next_x96,
                sqrt_price_limit_x96,
            );
            
            // Compute swap in this step
            let step = compute_swap_step(
                state.sqrt_price_x96,
                target_price,
                state.liquidity,
                state.amount_remaining,
                FEE_BPS,
            );
            
            state.amount_remaining = state.amount_remaining - step.amount_in - step.fee_amount;
            state.amount_calculated = state.amount_calculated + step.amount_out;
            state.sqrt_price_x96 = step.sqrt_price_start_x96;  // New price after step
            
            // If we reached the next tick, cross it
            if (state.sqrt_price_x96 == sqrt_price_next_x96) {
                if (initialized) {
                    // Cross tick: update liquidity
                    let liquidity_net = get_tick_liquidity_net(tick_next);
                    // When going left (x→y), subtract liquidity_net
                    state.liquidity = if (is_negative(liquidity_net)) {
                        state.liquidity - abs_value(liquidity_net)
                    } else {
                        state.liquidity + abs_value(liquidity_net)
                    };
                };
                state.tick = tick_next - 1;
            } else {
                state.tick = sqrt_price_to_tick(state.sqrt_price_x96);
            };
        };
        
        state.amount_calculated
    }
    
    // Compute how much comes out when swapping with given liquidity
    // between two price points
    fun compute_swap_step(
        sqrt_price_current_x96: u128,
        sqrt_price_target_x96: u128,
        liquidity: u128,
        amount_remaining: u64,
        fee_bps: u64,
    ): StepComputation {
        let exact_in = true;  // We're specifying exact input
        let Q96 = 79_228_162_514_264_337_593_543_950_336u128;
        
        // Calculate maximum amount we can swap to reach target price
        // For x→y (price decreasing):
        // amount_x_max = L * (1/sqrt_target - 1/sqrt_current) * Q96
        
        let amount_in_max = ((liquidity as u256) * (Q96 as u256)
            * ((sqrt_price_current_x96 - sqrt_price_target_x96) as u256)
            / ((sqrt_price_target_x96 as u256) * (sqrt_price_current_x96 as u256))) as u64;
        
        let fee_factor = 10_000 - fee_bps;
        let amount_in_after_fee = ((amount_remaining as u128) * (fee_factor as u128) / 10_000u128) as u64;
        
        let (amount_in, sqrt_price_next_x96, amount_out): (u64, u128, u64);
        
        if (amount_in_after_fee >= amount_in_max) {
            // Enough to reach target price
            amount_in = amount_in_max;
            sqrt_price_next_x96 = sqrt_price_target_x96;
        } else {
            // Not enough: compute new price
            amount_in = amount_in_after_fee;
            // New price after using amount_in
            sqrt_price_next_x96 = sqrt_price_current_x96;  // Simplified
        };
        
        // Amount out: L * (sqrt_current - sqrt_next) / Q96
        amount_out = ((liquidity as u256) * ((sqrt_price_current_x96 - sqrt_price_next_x96) as u256)
            / (Q96 as u256)) as u64;
        
        let fee_amount = amount_remaining - amount_in;
        
        StepComputation {
            sqrt_price_start_x96: sqrt_price_next_x96,
            tick_next: 0,  // Simplified
            sqrt_price_next_x96,
            amount_in,
            amount_out,
            fee_amount,
        }
    }
    
    const FEE_BPS: u64 = 30;  // 0.3%
    
    // Placeholders
    fun get_current_sqrt_price(_pool: address): u128 { 0 }
    fun get_current_tick(_pool: address): i64 { 0 }
    fun get_current_liquidity(_pool: address): u128 { 0 }
    fun find_next_tick(_tick: i64, _lte: bool): (i64, bool) { (0, false) }
    fun tick_to_sqrt_price(_tick: i64): u128 { 0 }
    fun sqrt_price_to_tick(_price: u128): i64 { 0 }
    fun get_tick_liquidity_net(_tick: i64): I128 { I128 { abs_value: 0, is_negative: false } }
    fun is_negative(x: I128): bool { x.is_negative }
    fun abs_value(x: I128): u128 { x.abs_value }
    
    type i64 = u64;
    
    struct I128 has copy, drop, store {
        abs_value: u128,
        is_negative: bool,
    }
}
```

---

## Fee Collection

```move
module clmm::fees {
    
    // ============================================
    // Fee Accumulation in CLMM
    // ============================================
    
    // Global: fee_growth_global_x128 (increases with every swap)
    // Per tick: fee_growth_outside_x128 (tracks fees outside tick)
    // Per position: fee_growth_inside_x128_last (snapshot at last update)
    
    // Fees earned by position =
    //   liquidity * (fee_growth_inside_current - fee_growth_inside_last)
    
    // fee_growth_inside = fee_growth_global - fees_above - fees_below
    
    public fun compute_fee_growth_inside(
        lower_tick: i64,
        upper_tick: i64,
        current_tick: i64,
        fee_growth_global_x128: u256,
        // Tick info
        lower_fee_growth_outside_x128: u256,
        upper_fee_growth_outside_x128: u256,
    ): u256 {
        // Fees below lower tick
        let fee_growth_below = if (current_tick >= lower_tick) {
            lower_fee_growth_outside_x128
        } else {
            fee_growth_global_x128 - lower_fee_growth_outside_x128
        };
        
        // Fees above upper tick
        let fee_growth_above = if (current_tick < upper_tick) {
            upper_fee_growth_outside_x128
        } else {
            fee_growth_global_x128 - upper_fee_growth_outside_x128
        };
        
        // Fees inside range
        fee_growth_global_x128 - fee_growth_below - fee_growth_above
    }
    
    // Calculate fees earned by a position
    public fun compute_fees_earned(
        liquidity: u128,
        fee_growth_inside_current: u256,
        fee_growth_inside_last: u256,
    ): u64 {
        let Q128: u256 = 340_282_366_920_938_463_463_374_607_431_768_211_456u256;  // 2^128
        
        // Wrap around arithmetic for u256
        let fee_growth_delta = if (fee_growth_inside_current >= fee_growth_inside_last) {
            fee_growth_inside_current - fee_growth_inside_last
        } else {
            // Wrapped around
            (u256::max_value() - fee_growth_inside_last) + fee_growth_inside_current + 1
        };
        
        ((liquidity as u256) * fee_growth_delta / Q128) as u64
    }
    
    type i64 = u64;
}
```

---

## ตัวอย่าง: Uniswap V3-style CLMM in Move

```move
module clmm::pool {
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::event;
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // Full CLMM Pool (simplified)
    // ============================================
    
    struct Pool<phantom X, phantom Y> has key {
        // Current state
        sqrt_price_x96: u128,
        current_tick: i64,
        liquidity: u128,         // Active liquidity at current price
        
        // Fee tracking
        fee_bps: u64,
        fee_growth_global_x128: u256,
        fee_growth_global_y128: u256,
        
        // Reserves
        reserve_x: u64,
        reserve_y: u64,
        
        // Protocol fee
        protocol_fee_x: u64,
        protocol_fee_y: u64,
    }
    
    struct TickData has copy, drop, store {
        liquidity_gross: u128,
        liquidity_net: I128,
        fee_growth_outside_x128: u256,
        fee_growth_outside_y128: u256,
        initialized: bool,
    }
    
    struct PoolTicks<phantom X, phantom Y> has key {
        ticks: SmartTable<i64, TickData>,
    }
    
    struct PoolPositions<phantom X, phantom Y> has key {
        positions: SmartTable<PositionKey, PositionData>,
    }
    
    struct PositionKey has copy, drop, store {
        owner: address,
        lower: i64,
        upper: i64,
    }
    
    struct PositionData has copy, drop, store {
        liquidity: u128,
        fee_growth_inside_x128_last: u256,
        fee_growth_inside_y128_last: u256,
        fees_x: u64,
        fees_y: u64,
    }
    
    struct I128 has copy, drop, store {
        abs: u128,
        neg: bool,
    }
    
    type i64 = u64;  // Simplified representation
    
    #[event]
    struct Mint has drop, store {
        owner: address,
        lower: i64,
        upper: i64,
        liquidity_delta: u128,
        amount_x: u64,
        amount_y: u64,
    }
    
    #[event]
    struct Swap has drop, store {
        recipient: address,
        amount_x: i64,  // Can be negative (paid out)
        amount_y: i64,
        sqrt_price_x96: u128,
        liquidity: u128,
        tick: i64,
    }
    
    // Add liquidity to CLMM position
    public entry fun mint<X, Y>(
        provider: &signer,
        pool_addr: address,
        lower_tick: i64,
        upper_tick: i64,
        liquidity_delta: u128,
    ) acquires Pool<X,Y>, PoolTicks<X,Y>, PoolPositions<X,Y> {
        let pool = borrow_global_mut<Pool<X, Y>>(pool_addr);
        
        // Calculate required token amounts for this liquidity
        let lower_sqrt = tick_to_sqrt_price(lower_tick);
        let upper_sqrt = tick_to_sqrt_price(upper_tick);
        
        let (amount_x, amount_y) = liquidity_to_amounts(
            liquidity_delta,
            pool.sqrt_price_x96,
            lower_sqrt,
            upper_sqrt,
        );
        
        // Update tick data
        let ticks = borrow_global_mut<PoolTicks<X, Y>>(pool_addr);
        update_tick(ticks, lower_tick, liquidity_delta, false);  // lower tick
        update_tick(ticks, upper_tick, liquidity_delta, true);   // upper tick (negative net)
        
        // Update active liquidity if position is active
        if (lower_tick <= pool.current_tick && pool.current_tick < upper_tick) {
            pool.liquidity = pool.liquidity + liquidity_delta;
        };
        
        // Update position
        let positions = borrow_global_mut<PoolPositions<X, Y>>(pool_addr);
        let key = PositionKey {
            owner: std::signer::address_of(provider),
            lower: lower_tick,
            upper: upper_tick,
        };
        
        if (smart_table::contains(&positions.positions, key)) {
            let pos = smart_table::borrow_mut(&mut positions.positions, key);
            pos.liquidity = pos.liquidity + liquidity_delta;
        } else {
            smart_table::add(&mut positions.positions, key, PositionData {
                liquidity: liquidity_delta,
                fee_growth_inside_x128_last: 0,
                fee_growth_inside_y128_last: 0,
                fees_x: 0,
                fees_y: 0,
            });
        };
        
        // Collect tokens from provider
        let coin_x = coin::withdraw<X>(provider, amount_x);
        let coin_y = coin::withdraw<Y>(provider, amount_y);
        coin::deposit<X>(pool_addr, coin_x);
        coin::deposit<Y>(pool_addr, coin_y);
        
        pool.reserve_x = pool.reserve_x + amount_x;
        pool.reserve_y = pool.reserve_y + amount_y;
        
        event::emit(Mint {
            owner: std::signer::address_of(provider),
            lower: lower_tick,
            upper: upper_tick,
            liquidity_delta,
            amount_x,
            amount_y,
        });
    }
    
    // ============================================
    // View functions
    // ============================================
    
    #[view]
    public fun get_current_price<X, Y>(pool_addr: address): u128 acquires Pool<X,Y> {
        borrow_global<Pool<X, Y>>(pool_addr).sqrt_price_x96
    }
    
    #[view]
    public fun get_position_amounts<X, Y>(
        pool_addr: address,
        owner: address,
        lower_tick: i64,
        upper_tick: i64,
    ): (u64, u64) acquires Pool<X,Y>, PoolPositions<X,Y> {
        let pool = borrow_global<Pool<X, Y>>(pool_addr);
        let positions = borrow_global<PoolPositions<X, Y>>(pool_addr);
        
        let key = PositionKey { owner, lower: lower_tick, upper: upper_tick };
        if (!smart_table::contains(&positions.positions, key)) return (0, 0);
        
        let pos = smart_table::borrow(&positions.positions, key);
        
        liquidity_to_amounts(
            pos.liquidity,
            pool.sqrt_price_x96,
            tick_to_sqrt_price(lower_tick),
            tick_to_sqrt_price(upper_tick),
        )
    }
    
    // Helpers
    fun tick_to_sqrt_price(_tick: i64): u128 { 0 }
    
    fun liquidity_to_amounts(_l: u128, _price: u128, _lo: u128, _hi: u128): (u64, u64) {
        (0, 0)
    }
    
    fun update_tick(_ticks: &mut PoolTicks<X,Y>, _tick: i64, _delta: u128, _upper: bool) {}
}
```

---

## สรุป CLMM Design

```
CLMM vs Traditional AMM:

Traditional AMM:                CLMM:
Fee APY: 1-5%                  Fee APY: 50-500%+ (concentrated)
Capital efficiency: ~1%         Capital efficiency: up to 4000x
LP experience: passive          LP experience: active management
Slippage: higher                Slippage: much lower (near peg)
IL risk: lower                  IL risk: higher (price leaves range)

CLMM Implementation Complexity:
  - Tick system (O(1) tick operations via bitmap)
  - Q64.96 fixed-point math (precision without floats)
  - Fee accumulation per tick (global→tick→position)
  - Swap algorithm crosses ticks dynamically

Key Math:
  Price = (sqrt_price_x96 / 2^96)^2
  Liquidity L = dx / d(1/sqrt_P) = dy / d(sqrt_P)
  Amount X in range [P_a, P_b]:
    dx = L * (1/sqrt(P_a) - 1/sqrt(P_b))
  Amount Y in range [P_a, P_b]:
    dy = L * (sqrt(P_b) - sqrt(P_a))

Real CLMM Protocols:
  - PancakeSwap V3 on Aptos
  - Cetus on Sui
  - Turbos on Sui
  All use this same mathematical framework
```

---

**ก่อนหน้า**: [Part 51 - MEV Strategies ←](part-51-mev-strategies.md)
**ต่อไป**: [Part 53 - Advanced Stablecoin Design →](part-53-stablecoin-design.md)
