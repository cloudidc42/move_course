# Part 71: Advanced AMM Design

## สารบัญ
- [AMM Taxonomy](#amm-taxonomy)
- [Constant Product (x*y=k) Deep Dive](#constant-product-xyk-deep-dive)
- [Stable Swap (Curve) Algorithm](#stable-swap-curve-algorithm)
- [Virtual Liquidity AMMs](#virtual-liquidity-amms)
- [TWAMM (Time-Weighted AMM)](#twamm-time-weighted-amm)
- [Dynamic Fee AMM](#dynamic-fee-amm)
- [AMM Aggregation & Routing](#amm-aggregation--routing)

---

## AMM Taxonomy

```
AMM Design Space:

TYPE 1: CONSTANT PRODUCT (x*y=k)
  Formula: x * y = k
  Examples: Uniswap V2, Liquidswap, Pancakeswap
  Best for: General token pairs
  Slippage: High for large orders (unbounded)
  Pros: Simple, works for any pair
  Cons: High price impact, "infinite" slippage at extremes

TYPE 2: STABLE SWAP (Curve formula)
  Formula: A*n^n*sum(x) + D = A*D*n^n + D^(n+1)/(n^n * prod(x))
  Examples: Curve, Ellipsis
  Best for: Like-asset pairs (USDC/USDT, stETH/ETH)
  Slippage: Very low near peg
  Pros: Extreme capital efficiency for stablecoins
  Cons: Bad for non-correlated assets (impermanent loss)

TYPE 3: CONCENTRATED LIQUIDITY (Uniswap V3)
  Formula: x*y=k within price range [Pa, Pb]
  Examples: Uniswap V3, Quickswap V3, Cetus (Sui)
  Best for: All pairs with active LPs
  Slippage: Low if liquidity concentrated near price
  Pros: Capital efficient (100x vs V2)
  Cons: LP management complexity, more impermanent loss

TYPE 4: VIRTUAL AMM (vAMM)
  Formula: Virtual x*y=k (no actual tokens in pool)
  Examples: Perpetual Protocol, Drift Protocol
  Best for: Perpetual futures trading
  Pros: No liquidity needed, infinite liquidity
  Cons: Funding rates needed to keep price anchored

TYPE 5: PROACTIVE MARKET MAKER (PMM)
  Formula: Price formula adjusts to follow oracle price
  Examples: DODO
  Best for: Reducing impermanent loss for LPs
  Pros: Better capital efficiency, lower IL
  Cons: More complex, requires oracle

TYPE 6: TIME-WEIGHTED AMM (TWAMM)
  Split large order over time using virtual swaps
  Best for: Executing large orders without moving price
  Examples: FranklinDAO TWAMM, Paraswap
  Pros: VWAP execution for large orders
  Cons: Complex state management
```

---

## Constant Product (x*y=k) Deep Dive

```move
module amm::constant_product {
    // ============================================
    // FULL CONSTANT PRODUCT AMM (x*y=k)
    // With fee, concentrated at this price range
    // ============================================
    
    const FEE_BPS: u64 = 30;  // 0.3%
    const BPS_BASE: u64 = 10_000;
    const MINIMUM_LIQUIDITY: u64 = 1_000;  // Burn minimum LP to prevent division by zero
    
    struct Pool<phantom X, phantom Y> has key {
        reserve_x: u64,
        reserve_y: u64,
        lp_supply: u64,
        fee_bps: u64,
        
        // Accumulated fees (can be collected by protocol)
        fee_x: u64,
        fee_y: u64,
    }
    
    // Calculate output for exact input swap
    // Uses: dy = y * dx_with_fee / (x + dx_with_fee)
    public fun get_amount_out(
        amount_in: u64,
        reserve_in: u64,
        reserve_out: u64,
        fee_bps: u64,
    ): u64 {
        assert!(amount_in > 0, 1);
        assert!(reserve_in > 0 && reserve_out > 0, 2);
        
        // Deduct fee first
        let amount_in_with_fee = (amount_in as u128) * (BPS_BASE - fee_bps as u128) as u128;
        let numerator = amount_in_with_fee * (reserve_out as u128);
        let denominator = (reserve_in as u128) * (BPS_BASE as u128) + amount_in_with_fee;
        
        (numerator / denominator) as u64
    }
    
    // Calculate input required for exact output
    // Uses: dx = x * dy / ((y - dy) * (1 - fee)) + 1 (rounded up)
    public fun get_amount_in(
        amount_out: u64,
        reserve_in: u64,
        reserve_out: u64,
        fee_bps: u64,
    ): u64 {
        assert!(amount_out > 0, 1);
        assert!(reserve_in > 0 && reserve_out > amount_out, 2);
        
        let numerator = (reserve_in as u128) * (amount_out as u128) * (BPS_BASE as u128);
        let denominator = ((reserve_out - amount_out) as u128) * ((BPS_BASE - fee_bps) as u128);
        
        // Round up (ceiling division)
        ((numerator + denominator - 1) / denominator) as u64
    }
    
    // Calculate LP tokens for adding liquidity
    public fun get_lp_tokens_to_mint(
        amount_x: u64,
        amount_y: u64,
        reserve_x: u64,
        reserve_y: u64,
        lp_supply: u64,
    ): (u64, u64, u64) {  // (optimal_x, optimal_y, lp_tokens)
        if (lp_supply == 0) {
            // First liquidity: geometric mean, burn minimum
            let lp_tokens = sqrt(
                (amount_x as u128) * (amount_y as u128)
            ) - MINIMUM_LIQUIDITY;
            (amount_x, amount_y, lp_tokens)
        } else {
            // Subsequent: proportional
            // Use whichever is smaller to avoid taking more than deposited
            let lp_from_x = (amount_x as u128) * (lp_supply as u128) / (reserve_x as u128);
            let lp_from_y = (amount_y as u128) * (lp_supply as u128) / (reserve_y as u128);
            
            if (lp_from_x <= lp_from_y) {
                // X is limiting factor: adjust Y proportionally
                let optimal_y = (amount_x as u128) * (reserve_y as u128) / (reserve_x as u128);
                (amount_x, optimal_y as u64, lp_from_x as u64)
            } else {
                // Y is limiting factor: adjust X proportionally
                let optimal_x = (amount_y as u128) * (reserve_x as u128) / (reserve_y as u128);
                (optimal_x as u64, amount_y, lp_from_y as u64)
            }
        }
    }
    
    // Calculate tokens received for burning LP
    public fun get_amounts_for_lp(
        lp_amount: u64,
        reserve_x: u64,
        reserve_y: u64,
        lp_supply: u64,
    ): (u64, u64) {
        let amount_x = (lp_amount as u128) * (reserve_x as u128) / (lp_supply as u128);
        let amount_y = (lp_amount as u128) * (reserve_y as u128) / (lp_supply as u128);
        (amount_x as u64, amount_y as u64)
    }
    
    // Price impact for a given swap
    public fun price_impact_bps(
        amount_in: u64,
        reserve_in: u64,
        reserve_out: u64,
    ): u64 {
        // Ideal output (no impact) = reserve_out * amount_in / reserve_in
        let ideal_out = (reserve_out as u128) * (amount_in as u128) / (reserve_in as u128);
        
        // Actual output
        let actual_out = get_amount_out(amount_in, reserve_in, reserve_out, 0) as u128;
        
        if (ideal_out == 0) return 10_000;
        
        // Impact = (ideal - actual) / ideal * 10000
        ((ideal_out - actual_out) * 10_000 / ideal_out) as u64
    }
    
    fun sqrt(x: u128): u64 {
        if (x == 0) return 0;
        let mut z = x;
        let mut y = (x + 1) / 2;
        while (y < z) { z = y; y = (x / y + y) / 2; };
        z as u64
    }
}
```

---

## Stable Swap (Curve) Algorithm

```move
module amm::stable_swap {
    // ============================================
    // STABLESWAP / CURVE ALGORITHM
    //
    // For stablecoin pairs (USDC/USDT, etc.)
    // Combines constant product and constant sum
    //
    // Formula: A*n^n*sum(x_i) + D = A*D*n^n + D^(n+1)/(n^n * prod(x_i))
    //
    // As A → 0: constant product (x*y=k)
    // As A → ∞: constant sum (x+y=k)
    // A=100 is typical for stableswaps
    // ============================================
    
    const N_COINS: u64 = 2;       // 2-coin pool
    const A_PRECISION: u64 = 100; // Amplification coefficient precision
    const ITERATIONS: u64 = 255;  // Newton's method iterations
    
    // Calculate D (invariant) given reserves
    // Solve: A*n^n*sum(x_i) + D = A*D*n^n + D^(n+1)/(n^n * prod(x_i))
    public fun get_d(x: u64, y: u64, amp: u64): u64 {
        let s = (x as u128) + (y as u128);  // sum of reserves
        if (s == 0) return 0;
        
        let ann = (amp * N_COINS * N_COINS) as u128;  // A*n^n
        
        let mut d = s;
        let mut d_prev: u128;
        
        let mut i = 0u64;
        while (i < ITERATIONS) {
            // d_prod = D^(n+1) / (n^n * prod(x_i))
            let d_prod = d * d / (x as u128) * d / (y as u128) / 4; // n^n = 4 for n=2
            
            d_prev = d;
            // Newton's method step:
            // D_{n+1} = (A*n^n*S + n*D_prod) * D / ((A*n^n - 1) * D + (n+1)*D_prod)
            d = (ann * s + N_COINS as u128 * d_prod) * d 
                / ((ann - 1) * d + (N_COINS as u128 + 1) * d_prod);
            
            if (abs_diff_u128(d, d_prev) <= 1) break;
            i = i + 1;
        };
        
        d as u64
    }
    
    // Calculate output y for given input x change
    // Used for swap calculation
    public fun get_y(
        x_new: u64,  // New reserve X after adding input
        d: u64,      // Current D invariant
        amp: u64,
    ): u64 {
        let ann = (amp * N_COINS * N_COINS) as u128;
        let d_128 = d as u128;
        let x = x_new as u128;
        
        // Solve for y: A*n^n*(x+y) + D = A*D*n^n + D^3/(4*x*y)
        // Rearranging: y^2 + b*y - c = 0 where:
        let b = x + d_128 / ann;
        let c = d_128 * d_128 * d_128 / (4 * x * ann);
        
        // Newton's method for y
        let mut y = d_128;
        let mut y_prev: u128;
        
        let mut i = 0u64;
        while (i < ITERATIONS) {
            y_prev = y;
            y = (y * y + c) / (2 * y + b - d_128);
            if (abs_diff_u128(y, y_prev) <= 1) break;
            i = i + 1;
        };
        
        y as u64
    }
    
    // Swap dx of token X, get dy of token Y
    public fun stable_swap(
        dx: u64,
        reserve_x: u64,
        reserve_y: u64,
        amp: u64,
        fee_bps: u64,
    ): u64 {
        // Get invariant D
        let d = get_d(reserve_x, reserve_y, amp);
        
        // New X reserve after adding input
        let x_new = reserve_x + dx;
        
        // Get new Y reserve
        let y_new = get_y(x_new, d, amp);
        
        // Output = old Y - new Y
        let dy = reserve_y - y_new;
        
        // Apply fee
        let fee = dy * fee_bps / 10_000;
        
        dy - fee
    }
    
    // Calculate price at current reserves (for UI display)
    // Price = dy/dx for infinitesimal dx
    public fun spot_price(reserve_x: u64, reserve_y: u64, amp: u64): u64 {
        let dx = 1_000;  // Small test amount
        let dy = stable_swap(dx, reserve_x, reserve_y, amp, 0);
        dy * 1_000_000 / dx  // Returns price with 6 decimal places
    }
    
    fun abs_diff_u128(a: u128, b: u128): u128 {
        if (a >= b) a - b else b - a
    }
}
```

---

## TWAMM (Time-Weighted AMM)

```move
module amm::twamm {
    use aptos_framework::timestamp;
    
    // ============================================
    // TIME-WEIGHTED AMM (TWAMM)
    // Execute large orders over time
    // Avoids front-running and price impact
    //
    // How it works:
    // 1. User submits "sell 1M USDC over 24 hours"
    // 2. Protocol virtually executes ~694 USDC/minute
    // 3. Each swap settles against virtual counterparty
    // 4. If another order goes in opposite direction:
    //    they net against each other (no price impact!)
    // ============================================
    
    const EXPIRY_BLOCKS: u64 = 10_000;  // ~24 hours
    
    struct TWAMMPool<phantom X, phantom Y> has key {
        reserve_x: u64,
        reserve_y: u64,
        fee_bps: u64,
        
        // Active orders
        orders_sell_x: vector<Order>,  // Orders selling X for Y
        orders_sell_y: vector<Order>,  // Orders selling Y for X
        
        // Virtual order state
        last_virtual_order_timestamp: u64,
        
        // Cumulative rate (tokens per block)
        rate_x: u64,  // Aggregate X sold per block
        rate_y: u64,  // Aggregate Y sold per block
    }
    
    struct Order has store {
        id: u64,
        owner: address,
        amount_remaining: u64,
        amount_per_block: u64,
        expiry_time: u64,
        is_sell_x: bool,
    }
    
    // Submit a long-running order
    public fun submit_long_term_order<X, Y>(
        user: &signer,
        pool_addr: address,
        is_sell_x: bool,
        total_amount: u64,
        blocks_to_execute: u64,
    ) acquires TWAMMPool {
        let pool = borrow_global_mut<TWAMMPool<X, Y>>(pool_addr);
        let user_addr = std::signer::address_of(user);
        
        // Execute any pending virtual orders first
        execute_virtual_orders(pool);
        
        let amount_per_block = total_amount / blocks_to_execute;
        let now = timestamp::now_microseconds();
        
        let order = Order {
            id: (std::vector::length(&pool.orders_sell_x) as u64),
            owner: user_addr,
            amount_remaining: total_amount,
            amount_per_block,
            expiry_time: now + blocks_to_execute * 1_000_000, // Approx 1s per block
            is_sell_x,
        };
        
        if (is_sell_x) {
            // Take X from user
            let coins = aptos_framework::coin::withdraw<X>(user, total_amount);
            // Store coins...
            std::vector::push_back(&mut pool.orders_sell_x, order);
            pool.rate_x = pool.rate_x + amount_per_block;
        } else {
            // Take Y from user
            let coins = aptos_framework::coin::withdraw<Y>(user, total_amount);
            std::vector::push_back(&mut pool.orders_sell_y, order);
            pool.rate_y = pool.rate_y + amount_per_block;
        };
    }
    
    // Execute virtual orders up to current time
    // Called before any pool interaction
    fun execute_virtual_orders<X, Y>(pool: &mut TWAMMPool<X, Y>) {
        let now = timestamp::now_microseconds();
        let elapsed = now - pool.last_virtual_order_timestamp;
        
        if (elapsed == 0) return;
        
        // Virtual execution: pretend swaps happened over time
        // If rate_x > 0 and rate_y > 0, they net against each other
        
        let virtual_x = pool.rate_x * elapsed / 1_000_000; // Tokens per block
        let virtual_y = pool.rate_y * elapsed / 1_000_000;
        
        if (virtual_x > 0 && virtual_y > 0) {
            // Orders net against each other
            // This is the key innovation: no price impact when orders cancel!
            let min_virtual = if (virtual_x < virtual_y) virtual_x else virtual_y;
            
            // Net settlement (simplified)
            // In practice: complex math to maintain invariant
        } else if (virtual_x > 0) {
            // Only sell X orders: execute against reserves
            let out_y = get_amount_out(virtual_x, pool.reserve_x, pool.reserve_y);
            pool.reserve_x = pool.reserve_x + virtual_x;
            pool.reserve_y = pool.reserve_y - out_y;
        } else if (virtual_y > 0) {
            // Only sell Y orders
            let out_x = get_amount_out(virtual_y, pool.reserve_y, pool.reserve_x);
            pool.reserve_y = pool.reserve_y + virtual_y;
            pool.reserve_x = pool.reserve_x - out_x;
        };
        
        pool.last_virtual_order_timestamp = now;
        
        // Clean up expired orders
        // (Simplified: in production iterate and remove expired)
    }
    
    fun get_amount_out(amount_in: u64, reserve_in: u64, reserve_out: u64): u64 {
        let numerator = (amount_in as u128) * 9970 * (reserve_out as u128);
        let denominator = (reserve_in as u128) * 10000 + (amount_in as u128) * 9970;
        (numerator / denominator) as u64
    }
}
```

---

## Dynamic Fee AMM

```move
module amm::dynamic_fee {
    use aptos_framework::timestamp;
    
    // ============================================
    // DYNAMIC FEE AMM
    // Fee adjusts based on volatility:
    // - High volatility → Higher fee (protect LPs from IL)
    // - Low volatility → Lower fee (attract more volume)
    //
    // Fee = base_fee + volatility_fee
    // volatility_fee = vol_estimate * multiplier
    // ============================================
    
    struct DynamicFeePool<phantom X, phantom Y> has key {
        reserve_x: u64,
        reserve_y: u64,
        lp_supply: u64,
        
        // Fee parameters
        base_fee_bps: u64,     // e.g., 5 = 0.05% minimum
        max_fee_bps: u64,      // e.g., 100 = 1% maximum
        vol_multiplier: u64,   // How much to multiply volatility
        
        // TWAP for volatility measurement
        last_price: u64,       // Price at last interaction
        price_sum: u128,       // Cumulative price sum
        price_count: u64,      // Number of observations
        last_update_time: u64,
        
        // Rolling volatility estimate (EMA)
        vol_ema: u64,          // Exponential moving average of |price_change|
        ema_alpha: u64,        // EMA decay (e.g., 5 = 5% weight to new observation)
    }
    
    // Get current dynamic fee
    public fun get_current_fee<X, Y>(pool_addr: address): u64 acquires DynamicFeePool {
        let pool = borrow_global<DynamicFeePool<X, Y>>(pool_addr);
        calculate_fee(pool)
    }
    
    fun calculate_fee<X, Y>(pool: &DynamicFeePool<X, Y>): u64 {
        // fee = base + vol_ema * multiplier
        let vol_fee = pool.vol_ema * pool.vol_multiplier / 10_000;
        let total_fee = pool.base_fee_bps + vol_fee;
        
        // Clamp to max
        if (total_fee > pool.max_fee_bps) pool.max_fee_bps else total_fee
    }
    
    fun update_volatility<X, Y>(pool: &mut DynamicFeePool<X, Y>, new_price: u64) {
        let now = timestamp::now_microseconds();
        
        if (pool.last_price > 0 && pool.last_update_time > 0) {
            // Calculate price change in bps
            let price_change = if (new_price > pool.last_price) {
                (new_price - pool.last_price) * 10_000 / pool.last_price
            } else {
                (pool.last_price - new_price) * 10_000 / pool.last_price
            };
            
            // Update EMA: new_ema = alpha * new_val + (1 - alpha) * old_ema
            // alpha = ema_alpha / 100
            let alpha = pool.ema_alpha;
            pool.vol_ema = (price_change * alpha + pool.vol_ema * (100 - alpha)) / 100;
        };
        
        pool.last_price = new_price;
        pool.last_update_time = now;
    }
}
```

---

## สรุป AMM Design

```
AMM Design Decision Tree:

1. What assets am I pooling?
   Stablecoins / pegged assets → Stable Swap (Curve)
   General assets             → Constant Product (x*y=k)
   Perpetuals                 → vAMM
   Any with active management → Concentrated Liquidity
   
2. What's my volume profile?
   Small orders              → Any AMM works
   Large orders frequently   → TWAMM
   Large orders occasionally → Split routing via aggregator
   
3. How sophisticated are my LPs?
   Passive (set and forget)  → Full-range constant product
   Active management         → CLMM (Uniswap V3 style)
   Want auto-management      → Gamma, Arrakis vaults
   
4. What fee model?
   Stable pairs              → Low fixed fee (0.04%)
   General pairs             → Medium fixed fee (0.3%)
   Volatile pairs            → Dynamic fee (0.05%-1%)
   
AMM Comparison on Aptos/Sui:
  Liquidswap:  Constant product, stable swap
  Hippo:       Aggregator (routes to best AMM)
  Cetus (Sui): CLMM (concentrated)
  Turbos (Sui): CLMM
  
Gas Considerations on Aptos:
  Simple AMM swap:     ~3,000 gas
  CLMM swap (1 tick):  ~8,000 gas
  Multi-hop 3 pools:   ~25,000 gas
  
  Optimize: Pre-compute routes off-chain
  Send: Single optimized route on-chain
```

---

**ก่อนหน้า**: [Part 70 - Protocol Architecture ←](part-70-protocol-architecture.md)
**ต่อไป**: [Part 72 - Lending Protocol Design →](part-72-lending-protocols.md)
