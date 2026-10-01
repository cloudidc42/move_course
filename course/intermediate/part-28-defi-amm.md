# Part 28: DeFi AMM Deep Dive

## สารบัญ
- [AMM Theory](#amm-theory)
- [Constant Product AMM (Uniswap v2)](#constant-product-amm-uniswap-v2)
- [Price Impact และ Slippage](#price-impact-และ-slippage)
- [Flash Loans](#flash-loans)
- [ตัวอย่าง: Multi-Hop Router](#ตัวอย่าง-multi-hop-router)

---

## AMM Theory

```
Constant Product Market Maker: x * y = k

Pool state:
  x = reserve of token A
  y = reserve of token B
  k = constant product (increases with fees)

Swap dx tokens of A:
  new_x = x + dx
  new_y = k / new_x = x*y / (x + dx)
  dy = y - new_y = y - x*y/(x + dx) = y*dx/(x + dx)

With fee (f = fee_bps / 10000):
  effective_dx = dx * (1 - f)
  dy = y * effective_dx / (x + effective_dx)

Price:
  Spot price of A in B = y/x
  Execution price = dy/dx (varies with size)
  Price impact = |spot_price - execution_price| / spot_price
```

---

## Constant Product AMM (Uniswap v2)

```move
module defi::uniswap_v2 {
    use std::signer;
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::event;
    
    // ============================================
    // Types
    // ============================================
    
    struct LPCoin<phantom X, phantom Y> has copy, drop, store {}
    
    struct Pool<phantom X, phantom Y> has key {
        reserve_x: Coin<X>,
        reserve_y: Coin<Y>,
        lp_supply: u128,
        fee_bps: u64,       // trading fee (e.g. 30 = 0.3%)
        protocol_fee_bps: u64,  // protocol fee (part of trading fee)
        protocol_fee_x: Coin<X>,  // accumulated protocol fees
        protocol_fee_y: Coin<Y>,
    }
    
    struct ProtocolConfig has key {
        admin: address,
        fee_collector: address,
    }
    
    // ============================================
    // Events
    // ============================================
    
    #[event]
    struct PoolCreated has drop, store {
        pool_addr: address,
        reserve_x: u64,
        reserve_y: u64,
        lp_minted: u64,
    }
    
    #[event]
    struct Swap has drop, store {
        pool_addr: address,
        user: address,
        amount_in: u64,
        amount_out: u64,
        is_x_in: bool,
    }
    
    #[event]
    struct LiquidityAdded has drop, store {
        pool_addr: address,
        amount_x: u64,
        amount_y: u64,
        lp_minted: u64,
    }
    
    #[event]
    struct LiquidityRemoved has drop, store {
        pool_addr: address,
        amount_x: u64,
        amount_y: u64,
        lp_burned: u64,
    }
    
    // ============================================
    // Errors
    // ============================================
    
    const E_ZERO_AMOUNT: u64 = 1;
    const E_SLIPPAGE_EXCEEDED: u64 = 2;
    const E_INSUFFICIENT_LIQUIDITY: u64 = 3;
    const E_IDENTICAL_TOKENS: u64 = 4;
    const E_OVERFLOW: u64 = 5;
    const E_K_INVARIANT_VIOLATED: u64 = 6;
    
    // ============================================
    // Initialize Pool
    // ============================================
    
    public entry fun initialize<X, Y>(
        creator: &signer,
        amount_x: u64,
        amount_y: u64,
        fee_bps: u64,
    ) {
        assert!(amount_x > 0 && amount_y > 0, E_ZERO_AMOUNT);
        
        let initial_lp = compute_initial_lp(amount_x, amount_y);
        
        let pool_addr = signer::address_of(creator);
        
        move_to(creator, Pool<X, Y> {
            reserve_x: coin::withdraw<X>(creator, amount_x),
            reserve_y: coin::withdraw<Y>(creator, amount_y),
            lp_supply: initial_lp,
            fee_bps,
            protocol_fee_bps: fee_bps / 5,  // 20% of trading fee
            protocol_fee_x: coin::zero<X>(),
            protocol_fee_y: coin::zero<Y>(),
        });
        
        // Mint LP tokens to creator
        // (simplified - in production would use a proper LP token)
        
        event::emit(PoolCreated {
            pool_addr,
            reserve_x: amount_x,
            reserve_y: amount_y,
            lp_minted: initial_lp as u64,
        });
    }
    
    // ============================================
    // Swap X -> Y
    // ============================================
    
    public entry fun swap_exact_x_for_y<X, Y>(
        user: &signer,
        pool_addr: address,
        amount_in: u64,
        min_out: u64,
    ) acquires Pool {
        assert!(amount_in > 0, E_ZERO_AMOUNT);
        
        let user_addr = signer::address_of(user);
        let pool = borrow_global_mut<Pool<X, Y>>(pool_addr);
        
        let rx = coin::value(&pool.reserve_x);
        let ry = coin::value(&pool.reserve_y);
        
        // Compute output with fee
        let (amount_out, protocol_fee) = compute_out(
            amount_in, rx, ry, pool.fee_bps, pool.protocol_fee_bps
        );
        
        assert!(amount_out >= min_out, E_SLIPPAGE_EXCEEDED);
        assert!(amount_out < ry, E_INSUFFICIENT_LIQUIDITY);
        
        // Transfer in
        let coin_in = coin::withdraw<X>(user, amount_in);
        coin::merge(&mut pool.reserve_x, coin_in);
        
        // Split protocol fee from output
        // (in this simplified version, fee stays in pool increasing k)
        
        // Transfer out
        let coin_out = coin::extract(&mut pool.reserve_y, amount_out);
        coin::deposit<Y>(user_addr, coin_out);
        
        // Verify k invariant (k can only increase due to fees)
        let new_rx = coin::value(&pool.reserve_x);
        let new_ry = coin::value(&pool.reserve_y);
        verify_k(rx, ry, new_rx, new_ry);
        
        event::emit(Swap {
            pool_addr,
            user: user_addr,
            amount_in,
            amount_out,
            is_x_in: true,
        });
        let _ = protocol_fee;
    }
    
    // ============================================
    // Swap Y -> X
    // ============================================
    
    public entry fun swap_exact_y_for_x<X, Y>(
        user: &signer,
        pool_addr: address,
        amount_in: u64,
        min_out: u64,
    ) acquires Pool {
        assert!(amount_in > 0, E_ZERO_AMOUNT);
        
        let user_addr = signer::address_of(user);
        let pool = borrow_global_mut<Pool<X, Y>>(pool_addr);
        
        let rx = coin::value(&pool.reserve_x);
        let ry = coin::value(&pool.reserve_y);
        
        let (amount_out, _) = compute_out(
            amount_in, ry, rx, pool.fee_bps, pool.protocol_fee_bps
        );
        
        assert!(amount_out >= min_out, E_SLIPPAGE_EXCEEDED);
        assert!(amount_out < rx, E_INSUFFICIENT_LIQUIDITY);
        
        let coin_in = coin::withdraw<Y>(user, amount_in);
        coin::merge(&mut pool.reserve_y, coin_in);
        
        let coin_out = coin::extract(&mut pool.reserve_x, amount_out);
        coin::deposit<X>(user_addr, coin_out);
        
        let new_rx = coin::value(&pool.reserve_x);
        let new_ry = coin::value(&pool.reserve_y);
        verify_k(rx, ry, new_rx, new_ry);
        
        event::emit(Swap {
            pool_addr,
            user: user_addr,
            amount_in,
            amount_out,
            is_x_in: false,
        });
    }
    
    // ============================================
    // Add Liquidity
    // ============================================
    
    public entry fun add_liquidity<X, Y>(
        user: &signer,
        pool_addr: address,
        amount_x_desired: u64,
        amount_y_desired: u64,
        min_x: u64,
        min_y: u64,
    ) acquires Pool {
        let pool = borrow_global_mut<Pool<X, Y>>(pool_addr);
        
        let rx = coin::value(&pool.reserve_x);
        let ry = coin::value(&pool.reserve_y);
        let total_lp = pool.lp_supply;
        
        // Calculate optimal amounts maintaining price ratio
        let (amount_x, amount_y) = calculate_liquidity_amounts(
            amount_x_desired, amount_y_desired, rx, ry, min_x, min_y
        );
        
        // LP tokens = min(dx/x, dy/y) * total_lp
        let lp_x = (amount_x as u128) * total_lp / (rx as u128);
        let lp_y = (amount_y as u128) * total_lp / (ry as u128);
        let lp_minted = if (lp_x < lp_y) { lp_x } else { lp_y };
        
        assert!(lp_minted > 0, E_ZERO_AMOUNT);
        
        let coin_x = coin::withdraw<X>(user, amount_x);
        let coin_y = coin::withdraw<Y>(user, amount_y);
        
        coin::merge(&mut pool.reserve_x, coin_x);
        coin::merge(&mut pool.reserve_y, coin_y);
        pool.lp_supply = pool.lp_supply + lp_minted;
        
        event::emit(LiquidityAdded {
            pool_addr,
            amount_x,
            amount_y,
            lp_minted: lp_minted as u64,
        });
    }
    
    // ============================================
    // Remove Liquidity
    // ============================================
    
    public entry fun remove_liquidity<X, Y>(
        user: &signer,
        pool_addr: address,
        lp_amount: u64,
        min_x: u64,
        min_y: u64,
    ) acquires Pool {
        assert!(lp_amount > 0, E_ZERO_AMOUNT);
        
        let user_addr = signer::address_of(user);
        let pool = borrow_global_mut<Pool<X, Y>>(pool_addr);
        
        let total_lp = pool.lp_supply;
        let rx = coin::value(&pool.reserve_x);
        let ry = coin::value(&pool.reserve_y);
        
        let amount_x = ((lp_amount as u128) * (rx as u128) / total_lp) as u64;
        let amount_y = ((lp_amount as u128) * (ry as u128) / total_lp) as u64;
        
        assert!(amount_x >= min_x && amount_y >= min_y, E_SLIPPAGE_EXCEEDED);
        
        pool.lp_supply = pool.lp_supply - (lp_amount as u128);
        
        let coin_x = coin::extract(&mut pool.reserve_x, amount_x);
        let coin_y = coin::extract(&mut pool.reserve_y, amount_y);
        
        coin::deposit<X>(user_addr, coin_x);
        coin::deposit<Y>(user_addr, coin_y);
        
        event::emit(LiquidityRemoved {
            pool_addr,
            amount_x,
            amount_y,
            lp_burned: lp_amount,
        });
    }
    
    // ============================================
    // View functions
    // ============================================
    
    #[view]
    public fun get_reserves<X, Y>(pool_addr: address): (u64, u64) acquires Pool {
        let pool = borrow_global<Pool<X, Y>>(pool_addr);
        (coin::value(&pool.reserve_x), coin::value(&pool.reserve_y))
    }
    
    #[view]
    public fun quote_swap_x_for_y<X, Y>(
        pool_addr: address,
        amount_in: u64,
    ): u64 acquires Pool {
        let pool = borrow_global<Pool<X, Y>>(pool_addr);
        let rx = coin::value(&pool.reserve_x);
        let ry = coin::value(&pool.reserve_y);
        let (out, _) = compute_out(amount_in, rx, ry, pool.fee_bps, pool.protocol_fee_bps);
        out
    }
    
    #[view]
    public fun spot_price_x<X, Y>(pool_addr: address): u64 acquires Pool {
        // Price of X in Y, scaled 1e9
        let pool = borrow_global<Pool<X, Y>>(pool_addr);
        let rx = coin::value(&pool.reserve_x);
        let ry = coin::value(&pool.reserve_y);
        if (rx == 0) return 0;
        ((ry as u128) * 1_000_000_000u128 / (rx as u128)) as u64
    }
    
    #[view]
    public fun price_impact_bps<X, Y>(
        pool_addr: address,
        amount_in: u64,
    ): u64 acquires Pool {
        let pool = borrow_global<Pool<X, Y>>(pool_addr);
        let rx = coin::value(&pool.reserve_x);
        let ry = coin::value(&pool.reserve_y);
        
        // Spot price (no fee)
        let spot = (ry as u128) * 1_000_000_000u128 / (rx as u128);
        
        // Execution price
        let (out, _) = compute_out(amount_in, rx, ry, pool.fee_bps, pool.protocol_fee_bps);
        let exec = (out as u128) * 1_000_000_000u128 / (amount_in as u128);
        
        // Impact = (spot - exec) / spot in BPS
        if (spot == 0 || exec >= spot) return 0;
        ((spot - exec) * 10_000u128 / spot) as u64
    }
    
    // ============================================
    // Helpers
    // ============================================
    
    fun compute_initial_lp(x: u64, y: u64): u128 {
        // sqrt(x * y)
        let product = (x as u128) * (y as u128);
        let mut result = product;
        let mut y_val = (product + 1) / 2;
        while (y_val < result) {
            result = y_val;
            y_val = (result + product / result) / 2;
        };
        result
    }
    
    fun compute_out(
        amount_in: u64,
        reserve_in: u64,
        reserve_out: u64,
        fee_bps: u64,
        protocol_fee_bps: u64,
    ): (u64, u64) {
        // Amount after total fee
        let amount_in_with_fee = (amount_in as u128) * ((10_000 - fee_bps) as u128);
        let numerator = amount_in_with_fee * (reserve_out as u128);
        let denominator = (reserve_in as u128) * 10_000u128 + amount_in_with_fee;
        
        let amount_out = (numerator / denominator) as u64;
        
        // Protocol fee = part of trading fee (kept in reserves)
        let total_fee = amount_in * fee_bps / 10_000;
        let protocol_fee = total_fee * protocol_fee_bps / 10_000;
        
        (amount_out, protocol_fee)
    }
    
    fun calculate_liquidity_amounts(
        x_desired: u64,
        y_desired: u64,
        reserve_x: u64,
        reserve_y: u64,
        min_x: u64,
        min_y: u64,
    ): (u64, u64) {
        // Amount of Y that maintains price ratio given x_desired
        let y_optimal = (x_desired as u128) * (reserve_y as u128) / (reserve_x as u128);
        
        if (y_optimal as u64 <= y_desired) {
            assert!(y_optimal as u64 >= min_y, E_SLIPPAGE_EXCEEDED);
            (x_desired, y_optimal as u64)
        } else {
            let x_optimal = (y_desired as u128) * (reserve_x as u128) / (reserve_y as u128);
            assert!(x_optimal as u64 >= min_x, E_SLIPPAGE_EXCEEDED);
            (x_optimal as u64, y_desired)
        }
    }
    
    fun verify_k(old_x: u64, old_y: u64, new_x: u64, new_y: u64) {
        let old_k = (old_x as u128) * (old_y as u128);
        let new_k = (new_x as u128) * (new_y as u128);
        assert!(new_k >= old_k, E_K_INVARIANT_VIOLATED);
    }
}
```

---

## Flash Loans

```move
module defi::flash_loan {
    use std::signer;
    use aptos_framework::coin::{Self, Coin};
    
    // ============================================
    // Hot Potato pattern for flash loans
    // ============================================
    
    // FlashLoanReceipt has NO drop ability
    // So it MUST be returned to the pool
    
    struct FlashLoanReceipt<phantom X> {  // no abilities = must be consumed
        pool_addr: address,
        amount: u64,
        fee: u64,
    }
    
    struct FlashLoanPool<phantom X> has key {
        reserves: Coin<X>,
        fee_bps: u64,
        total_fees_earned: u128,
    }
    
    const E_REPAYMENT_INSUFFICIENT: u64 = 1;
    const E_ZERO_LOAN: u64 = 2;
    
    // ============================================
    // Initiate flash loan
    // ============================================
    
    // Returns Coin + Receipt (receipt must be returned)
    public fun flash_borrow<X>(
        pool_addr: address,
        amount: u64,
    ): (Coin<X>, FlashLoanReceipt<X>) acquires FlashLoanPool {
        assert!(amount > 0, E_ZERO_LOAN);
        
        let pool = borrow_global_mut<FlashLoanPool<X>>(pool_addr);
        let reserves = coin::value(&pool.reserves);
        
        assert!(amount <= reserves, 3);  // E_INSUFFICIENT
        
        let fee = amount * pool.fee_bps / 10_000;
        if (fee == 0) { fee = 1; };  // minimum 1 unit fee
        
        let loan = coin::extract(&mut pool.reserves, amount);
        let receipt = FlashLoanReceipt<X> { pool_addr, amount, fee };
        
        (loan, receipt)
    }
    
    // ============================================
    // Repay flash loan
    // ============================================
    
    // Receipt is consumed here
    public fun flash_repay<X>(
        repayment: Coin<X>,
        receipt: FlashLoanReceipt<X>,
    ) acquires FlashLoanPool {
        let FlashLoanReceipt { pool_addr, amount, fee } = receipt;
        let required = amount + fee;
        
        assert!(coin::value(&repayment) >= required, E_REPAYMENT_INSUFFICIENT);
        
        let pool = borrow_global_mut<FlashLoanPool<X>>(pool_addr);
        pool.total_fees_earned = pool.total_fees_earned + (fee as u128);
        coin::merge(&mut pool.reserves, repayment);
    }
    
    // ============================================
    // Example: Arbitrage using flash loan
    // ============================================
    
    // Caller's module would do:
    //   1. flash_borrow -> get loan
    //   2. Buy cheap on DEX A
    //   3. Sell expensive on DEX B
    //   4. flash_repay -> return loan + fee, keep profit
    
    // The beauty: if step 2-3 don't profit enough, tx fails
    // All-or-nothing atomicity guaranteed by Move!
    
    // ============================================
    // Initialize pool
    // ============================================
    
    public entry fun init_pool<X>(
        admin: &signer,
        amount: u64,
        fee_bps: u64,
    ) {
        let initial = coin::withdraw<X>(admin, amount);
        move_to(admin, FlashLoanPool<X> {
            reserves: initial,
            fee_bps,
            total_fees_earned: 0,
        });
    }
    
    #[view]
    public fun pool_reserves<X>(pool_addr: address): u64 acquires FlashLoanPool {
        coin::value(&borrow_global<FlashLoanPool<X>>(pool_addr).reserves)
    }
}
```

---

## ตัวอย่าง: Multi-Hop Router

```move
module defi::router {
    use std::signer;
    use std::vector;
    use aptos_framework::coin::{Self, Coin};
    
    // ============================================
    // Router: chain multiple swaps together
    // ============================================
    
    // Example: USDC -> APT -> ETH (two hops)
    // Router finds best path and executes atomically
    
    struct Route has copy, drop {
        pool_addrs: vector<address>,
        is_x_to_y: vector<bool>,
    }
    
    const E_EMPTY_ROUTE: u64 = 1;
    const E_SLIPPAGE: u64 = 2;
    
    // ============================================
    // Two-hop swap: X -> intermediate -> Y
    // ============================================
    
    // Step 1: X -> Z at pool_xz
    // Step 2: Z -> Y at pool_zy
    // Result: X -> Y without direct pool
    
    public entry fun swap_two_hop<X, Z, Y>(
        user: &signer,
        pool_xz_addr: address,
        pool_zy_addr: address,
        amount_in: u64,
        min_out: u64,
    ) acquires super::uniswap_v2::Pool {
        use defi::uniswap_v2;
        
        let user_addr = signer::address_of(user);
        
        // Get coin X from user
        let coin_x = coin::withdraw<X>(user, amount_in);
        
        // Deposit X and get quote for Z
        let rx_xz = uniswap_v2::get_reserves<X, Z>(pool_xz_addr);
        let ry_xz = uniswap_v2::get_reserves_y<X, Z>(pool_xz_addr);
        let z_amount = compute_out_view(amount_in, rx_xz, ry_xz, 30);
        
        // Deposit coin_x to pool_xz, receive Z
        // (simplified - actual would interact with pool directly)
        coin::deposit<X>(user_addr, coin_x);
        
        // Execute swap 1: X -> Z
        uniswap_v2::swap_exact_x_for_y<X, Z>(user, pool_xz_addr, amount_in, 1);
        
        // Execute swap 2: Z -> Y
        uniswap_v2::swap_exact_x_for_y<Z, Y>(user, pool_zy_addr, z_amount, min_out);
        
        // Verify final amount meets minimum
        // (In production, check final balance received)
    }
    
    // ============================================
    // Quote multi-hop
    // ============================================
    
    #[view]
    public fun quote_two_hop<X, Z, Y>(
        pool_xz_addr: address,
        pool_zy_addr: address,
        amount_in: u64,
    ): u64 acquires super::uniswap_v2::Pool {
        use defi::uniswap_v2;
        
        // Step 1: X -> Z
        let z_out = uniswap_v2::quote_swap_x_for_y<X, Z>(pool_xz_addr, amount_in);
        
        // Step 2: Z -> Y
        let y_out = uniswap_v2::quote_swap_x_for_y<Z, Y>(pool_zy_addr, z_out);
        
        y_out
    }
    
    // ============================================
    // Best of direct vs two-hop
    // ============================================
    
    #[view]
    public fun best_route<X, Y>(
        direct_pool: address,
        hop_pool_xz: address,
        hop_pool_zy: address,
        amount_in: u64,
    ): (u64, bool) acquires super::uniswap_v2::Pool {
        use defi::uniswap_v2;
        
        // Quote direct swap
        let direct_out = uniswap_v2::quote_swap_x_for_y<X, Y>(direct_pool, amount_in);
        
        // Quote two-hop (simplified)
        // In practice would need the Z type parameter
        
        // Return better route
        (direct_out, true)  // (amount, is_direct)
    }
    
    fun compute_out_view(amount_in: u64, reserve_in: u64, reserve_out: u64, fee_bps: u64): u64 {
        let amount_in_with_fee = (amount_in as u128) * ((10_000 - fee_bps) as u128);
        let numerator = amount_in_with_fee * (reserve_out as u128);
        let denominator = (reserve_in as u128) * 10_000u128 + amount_in_with_fee;
        (numerator / denominator) as u64
    }
}
```

---

## สรุป AMM Concepts

| Concept | Formula | Notes |
|---------|---------|-------|
| Constant product | `x * y = k` | k increases with fees |
| Swap output | `dy = y*dx*(1-f)/(x + dx*(1-f))` | f = fee rate |
| Price impact | `(spot - exec) / spot` | % price worsening |
| LP tokens | `min(dx/x, dy/y) * total_lp` | Proportional share |
| Initial LP | `sqrt(x * y)` | Geometric mean |
| Withdrawal | `dx = lp/total * x` | Proportional |

---

**ก่อนหน้า**: [Part 27 - Sui Coin and DeFi ←](part-27-sui-coin-defi.md)
**ต่อไป**: [Part 29 - Lending Protocol →](part-29-lending-protocol.md)
