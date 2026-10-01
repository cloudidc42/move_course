# Part 27: Sui Coin and DeFi Basics

## สารบัญ
- [Sui Coin Standard](#sui-coin-standard)
- [Balance และ Coin Operations](#balance-และ-coin-operations)
- [SUI Native Token](#sui-native-token)
- [AMM บน Sui](#amm-บน-sui)
- [ตัวอย่าง: Liquidity Pool](#ตัวอย่าง-liquidity-pool)

---

## Sui Coin Standard

```move
module defi::coin_basics {
    use sui::coin::{Self, Coin, TreasuryCap, CoinMetadata};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};
    use std::option;
    
    // ============================================
    // Define a coin type
    // ============================================
    
    // One-time-witness pattern for coin creation
    struct USDC has drop {}
    
    // TreasuryCap<USDC> = mint/burn capability
    // CoinMetadata<USDC> = name/symbol/decimals (frozen object)
    
    fun init(witness: USDC, ctx: &mut TxContext) {
        let (treasury_cap, metadata) = coin::create_currency(
            witness,
            6,                              // decimals
            b"USDC",                        // symbol
            b"USD Coin",                    // name
            b"Stablecoin by Circle",        // description
            option::none(),                 // icon URL
            ctx,
        );
        
        // Freeze metadata (immutable)
        transfer::public_freeze_object(metadata);
        
        // Transfer treasury cap to deployer
        transfer::transfer(treasury_cap, tx_context::sender(ctx));
    }
    
    // ============================================
    // Mint
    // ============================================
    
    public entry fun mint(
        cap: &mut TreasuryCap<USDC>,
        amount: u64,
        recipient: address,
        ctx: &mut TxContext,
    ) {
        let coin = coin::mint(cap, amount, ctx);
        transfer::public_transfer(coin, recipient);
    }
    
    // ============================================
    // Burn
    // ============================================
    
    public entry fun burn(
        cap: &mut TreasuryCap<USDC>,
        coin: Coin<USDC>,
    ) {
        coin::burn(cap, coin);
    }
    
    // ============================================
    // View supply
    // ============================================
    
    public fun total_supply(cap: &TreasuryCap<USDC>): u64 {
        coin::total_supply(cap)
    }
}
```

---

## Balance และ Coin Operations

```move
module defi::balance_ops {
    use sui::coin::{Self, Coin};
    use sui::balance::{Self, Balance};
    use sui::sui::SUI;
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};
    
    // ============================================
    // Coin vs Balance
    // ============================================
    
    // Coin<T>    = user-facing, transferable object
    // Balance<T> = internal, not transferable, stores in structs
    
    // Convert
    public fun coin_to_balance(coin: Coin<SUI>): Balance<SUI> {
        coin::into_balance(coin)
    }
    
    public fun balance_to_coin(balance: Balance<SUI>, ctx: &mut TxContext): Coin<SUI> {
        coin::from_balance(balance, ctx)
    }
    
    // ============================================
    // Split and Join
    // ============================================
    
    public fun split_coin(
        coin: &mut Coin<SUI>,
        amount: u64,
        ctx: &mut TxContext,
    ): Coin<SUI> {
        coin::split(coin, amount, ctx)
    }
    
    public fun join_coins(base: &mut Coin<SUI>, other: Coin<SUI>) {
        coin::join(base, other);
    }
    
    // Balance operations
    public fun split_balance(
        balance: &mut Balance<SUI>,
        amount: u64,
    ): Balance<SUI> {
        balance::split(balance, amount)
    }
    
    public fun join_balances(base: &mut Balance<SUI>, other: Balance<SUI>) {
        balance::join(base, other);
    }
    
    // ============================================
    // Common patterns
    // ============================================
    
    struct Vault has key {
        id: sui::object::UID,
        reserves: Balance<SUI>,
    }
    
    // Deposit to vault
    public entry fun deposit(
        vault: &mut Vault,
        payment: Coin<SUI>,
    ) {
        let balance = coin::into_balance(payment);
        balance::join(&mut vault.reserves, balance);
    }
    
    // Withdraw from vault
    public entry fun withdraw(
        vault: &mut Vault,
        amount: u64,
        ctx: &mut TxContext,
    ) {
        let balance = balance::split(&mut vault.reserves, amount);
        let coin = coin::from_balance(balance, ctx);
        transfer::public_transfer(coin, tx_context::sender(ctx));
    }
    
    // Check balance
    public fun vault_balance(vault: &Vault): u64 {
        balance::value(&vault.reserves)
    }
    
    // ============================================
    // Fee extraction pattern
    // ============================================
    
    public fun take_fee(
        payment: &mut Coin<SUI>,
        fee_bps: u64,
        ctx: &mut TxContext,
    ): Coin<SUI> {
        let total = coin::value(payment);
        let fee = total * fee_bps / 10_000;
        coin::split(payment, fee, ctx)
    }
    
    // ============================================
    // Zero balance (empty)
    // ============================================
    
    public fun new_zero_balance(): Balance<SUI> {
        balance::zero<SUI>()
    }
    
    public fun destroy_zero(balance: Balance<SUI>) {
        balance::destroy_zero(balance);
    }
}
```

---

## SUI Native Token

```move
module defi::sui_usage {
    use sui::sui::SUI;
    use sui::coin::{Self, Coin};
    use sui::balance::Balance;
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};
    
    // ============================================
    // SUI = native token (like APT on Aptos)
    // ============================================
    
    // 1 SUI = 1_000_000_000 MIST (smallest unit)
    const MIST_PER_SUI: u64 = 1_000_000_000;
    
    public fun to_mist(sui_amount: u64): u64 {
        sui_amount * MIST_PER_SUI
    }
    
    public fun to_sui(mist_amount: u64): u64 {
        mist_amount / MIST_PER_SUI
    }
    
    // ============================================
    // Common SUI operations
    // ============================================
    
    // Transfer SUI by splitting
    public entry fun transfer_sui(
        mut coin: Coin<SUI>,
        amount: u64,
        recipient: address,
        ctx: &mut TxContext,
    ) {
        let payment = coin::split(&mut coin, amount, ctx);
        transfer::public_transfer(payment, recipient);
        
        // Return change to sender
        if (coin::value(&coin) > 0) {
            transfer::public_transfer(coin, tx_context::sender(ctx));
        } else {
            coin::destroy_zero(coin);
        };
    }
    
    // Collect fee in SUI
    struct FeeCollector has key {
        id: sui::object::UID,
        collected: Balance<SUI>,
        fee_bps: u64,
    }
    
    public entry fun service_with_fee(
        collector: &mut FeeCollector,
        mut payment: Coin<SUI>,
        ctx: &mut TxContext,
    ) {
        let total = coin::value(&payment);
        let fee = total * collector.fee_bps / 10_000;
        
        let fee_coin = coin::split(&mut payment, fee, ctx);
        sui::balance::join(
            &mut collector.collected,
            coin::into_balance(fee_coin),
        );
        
        // Return remainder
        transfer::public_transfer(payment, tx_context::sender(ctx));
    }
}
```

---

## AMM บน Sui

```move
module defi::amm_pool {
    use sui::object::{Self, UID};
    use sui::coin::{Self, Coin};
    use sui::balance::{Self, Balance, Supply};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};
    use sui::event;
    
    // ============================================
    // LP Token
    // ============================================
    
    struct LP<phantom X, phantom Y> has drop {}
    
    // ============================================
    // Pool
    // ============================================
    
    struct Pool<phantom X, phantom Y> has key {
        id: UID,
        reserve_x: Balance<X>,
        reserve_y: Balance<Y>,
        lp_supply: Supply<LP<X, Y>>,
        fee_bps: u64,
    }
    
    // ============================================
    // Events
    // ============================================
    
    struct SwapEvent has copy, drop {
        pool_id: sui::object::ID,
        amount_in: u64,
        amount_out: u64,
        is_x_to_y: bool,
    }
    
    struct LiquidityEvent has copy, drop {
        pool_id: sui::object::ID,
        amount_x: u64,
        amount_y: u64,
        lp_minted: u64,
    }
    
    // ============================================
    // Errors
    // ============================================
    
    const E_ZERO_AMOUNT: u64 = 0;
    const E_SLIPPAGE: u64 = 1;
    const E_INSUFFICIENT_LIQUIDITY: u64 = 2;
    
    // ============================================
    // Create Pool
    // ============================================
    
    public entry fun create_pool<X, Y>(
        coin_x: Coin<X>,
        coin_y: Coin<Y>,
        fee_bps: u64,
        ctx: &mut TxContext,
    ) {
        let amount_x = coin::value(&coin_x);
        let amount_y = coin::value(&coin_y);
        assert!(amount_x > 0 && amount_y > 0, E_ZERO_AMOUNT);
        
        let pool_uid = object::new(ctx);
        let pool_id = object::uid_to_inner(&pool_uid);
        
        // Initialize LP supply
        let lp_witness = LP<X, Y> {};
        let mut lp_supply = balance::create_supply(lp_witness);
        
        // Calculate initial LP tokens = sqrt(x * y)
        let lp_amount = sqrt((amount_x as u128) * (amount_y as u128));
        let lp_balance = balance::increase_supply(&mut lp_supply, lp_amount);
        let lp_coin = coin::from_balance(lp_balance, ctx);
        
        let pool = Pool<X, Y> {
            id: pool_uid,
            reserve_x: coin::into_balance(coin_x),
            reserve_y: coin::into_balance(coin_y),
            lp_supply,
            fee_bps,
        };
        
        event::emit(LiquidityEvent { pool_id, amount_x, amount_y, lp_minted: lp_amount });
        
        // Send LP to creator
        transfer::public_transfer(lp_coin, tx_context::sender(ctx));
        transfer::share_object(pool);
    }
    
    // ============================================
    // Add Liquidity
    // ============================================
    
    public entry fun add_liquidity<X, Y>(
        pool: &mut Pool<X, Y>,
        coin_x: Coin<X>,
        coin_y: Coin<Y>,
        min_lp: u64,
        ctx: &mut TxContext,
    ) {
        let amount_x = coin::value(&coin_x);
        let amount_y = coin::value(&coin_y);
        assert!(amount_x > 0 && amount_y > 0, E_ZERO_AMOUNT);
        
        let reserve_x = balance::value(&pool.reserve_x);
        let reserve_y = balance::value(&pool.reserve_y);
        let total_lp = balance::supply_value(&pool.lp_supply);
        
        // LP = min(dx/x, dy/y) * total_lp
        let lp_x = (amount_x as u128) * (total_lp as u128) / (reserve_x as u128);
        let lp_y = (amount_y as u128) * (total_lp as u128) / (reserve_y as u128);
        let lp_amount = if (lp_x < lp_y) { lp_x as u64 } else { lp_y as u64 };
        
        assert!(lp_amount >= min_lp, E_SLIPPAGE);
        
        let pool_id = object::uid_to_inner(&pool.id);
        
        balance::join(&mut pool.reserve_x, coin::into_balance(coin_x));
        balance::join(&mut pool.reserve_y, coin::into_balance(coin_y));
        
        let lp_balance = balance::increase_supply(&mut pool.lp_supply, lp_amount);
        let lp_coin = coin::from_balance(lp_balance, ctx);
        
        event::emit(LiquidityEvent { pool_id, amount_x, amount_y, lp_minted: lp_amount });
        
        transfer::public_transfer(lp_coin, tx_context::sender(ctx));
    }
    
    // ============================================
    // Remove Liquidity
    // ============================================
    
    public entry fun remove_liquidity<X, Y>(
        pool: &mut Pool<X, Y>,
        lp_coin: Coin<LP<X, Y>>,
        min_x: u64,
        min_y: u64,
        ctx: &mut TxContext,
    ) {
        let lp_amount = coin::value(&lp_coin);
        assert!(lp_amount > 0, E_ZERO_AMOUNT);
        
        let total_lp = balance::supply_value(&pool.lp_supply);
        let reserve_x = balance::value(&pool.reserve_x);
        let reserve_y = balance::value(&pool.reserve_y);
        
        // Proportional withdrawal
        let amount_x = (lp_amount as u128) * (reserve_x as u128) / (total_lp as u128);
        let amount_y = (lp_amount as u128) * (reserve_y as u128) / (total_lp as u128);
        
        assert!(amount_x as u64 >= min_x && amount_y as u64 >= min_y, E_SLIPPAGE);
        
        // Burn LP
        balance::decrease_supply(&mut pool.lp_supply, coin::into_balance(lp_coin));
        
        let sender = tx_context::sender(ctx);
        
        let coin_x = coin::from_balance(
            balance::split(&mut pool.reserve_x, amount_x as u64), ctx
        );
        let coin_y = coin::from_balance(
            balance::split(&mut pool.reserve_y, amount_y as u64), ctx
        );
        
        transfer::public_transfer(coin_x, sender);
        transfer::public_transfer(coin_y, sender);
    }
    
    // ============================================
    // Swap X -> Y
    // ============================================
    
    public entry fun swap_x_to_y<X, Y>(
        pool: &mut Pool<X, Y>,
        coin_in: Coin<X>,
        min_out: u64,
        ctx: &mut TxContext,
    ) {
        let amount_in = coin::value(&coin_in);
        assert!(amount_in > 0, E_ZERO_AMOUNT);
        
        let reserve_x = balance::value(&pool.reserve_x);
        let reserve_y = balance::value(&pool.reserve_y);
        
        let amount_out = get_amount_out(amount_in, reserve_x, reserve_y, pool.fee_bps);
        assert!(amount_out >= min_out, E_SLIPPAGE);
        assert!(amount_out < reserve_y, E_INSUFFICIENT_LIQUIDITY);
        
        balance::join(&mut pool.reserve_x, coin::into_balance(coin_in));
        
        let coin_out = coin::from_balance(
            balance::split(&mut pool.reserve_y, amount_out), ctx
        );
        
        event::emit(SwapEvent {
            pool_id: object::uid_to_inner(&pool.id),
            amount_in,
            amount_out,
            is_x_to_y: true,
        });
        
        transfer::public_transfer(coin_out, tx_context::sender(ctx));
    }
    
    // ============================================
    // Swap Y -> X
    // ============================================
    
    public entry fun swap_y_to_x<X, Y>(
        pool: &mut Pool<X, Y>,
        coin_in: Coin<Y>,
        min_out: u64,
        ctx: &mut TxContext,
    ) {
        let amount_in = coin::value(&coin_in);
        assert!(amount_in > 0, E_ZERO_AMOUNT);
        
        let reserve_x = balance::value(&pool.reserve_x);
        let reserve_y = balance::value(&pool.reserve_y);
        
        let amount_out = get_amount_out(amount_in, reserve_y, reserve_x, pool.fee_bps);
        assert!(amount_out >= min_out, E_SLIPPAGE);
        assert!(amount_out < reserve_x, E_INSUFFICIENT_LIQUIDITY);
        
        balance::join(&mut pool.reserve_y, coin::into_balance(coin_in));
        
        let coin_out = coin::from_balance(
            balance::split(&mut pool.reserve_x, amount_out), ctx
        );
        
        event::emit(SwapEvent {
            pool_id: object::uid_to_inner(&pool.id),
            amount_in,
            amount_out,
            is_x_to_y: false,
        });
        
        transfer::public_transfer(coin_out, tx_context::sender(ctx));
    }
    
    // ============================================
    // View functions
    // ============================================
    
    public fun get_reserves<X, Y>(pool: &Pool<X, Y>): (u64, u64) {
        (
            balance::value(&pool.reserve_x),
            balance::value(&pool.reserve_y),
        )
    }
    
    public fun get_lp_supply<X, Y>(pool: &Pool<X, Y>): u64 {
        balance::supply_value(&pool.lp_supply)
    }
    
    public fun price_x_in_y<X, Y>(pool: &Pool<X, Y>): u64 {
        // Price of 1 X in Y (scaled 1e9)
        let rx = balance::value(&pool.reserve_x);
        let ry = balance::value(&pool.reserve_y);
        if (rx == 0) return 0;
        (ry as u128 * 1_000_000_000u128 / rx as u128) as u64
    }
    
    public fun quote_swap_x<X, Y>(
        pool: &Pool<X, Y>,
        amount_x: u64,
    ): u64 {
        let rx = balance::value(&pool.reserve_x);
        let ry = balance::value(&pool.reserve_y);
        get_amount_out(amount_x, rx, ry, pool.fee_bps)
    }
    
    // ============================================
    // Helpers
    // ============================================
    
    // Constant product AMM: dx * (y - dy) = x * y
    // dy = y * dx / (x + dx) with fee
    fun get_amount_out(
        amount_in: u64,
        reserve_in: u64,
        reserve_out: u64,
        fee_bps: u64,
    ): u64 {
        let amount_in_after_fee = amount_in * (10_000 - fee_bps);
        let numerator = (amount_in_after_fee as u128) * (reserve_out as u128);
        let denominator = (reserve_in as u128) * 10_000u128 + (amount_in_after_fee as u128);
        (numerator / denominator) as u64
    }
    
    fun sqrt(n: u128): u64 {
        if (n == 0) return 0;
        let mut x = n;
        let mut y = (x + 1) / 2;
        while (y < x) {
            x = y;
            y = (x + n / x) / 2;
        };
        x as u64
    }
}
```

---

## สรุป Sui DeFi Patterns

| Pattern | Aptos | Sui |
|---------|-------|-----|
| Token definition | `struct T {}` + `coin::initialize` | `struct T has drop {}` + `coin::create_currency` |
| Store balance | `Coin<T>` in resource | `Balance<T>` in struct |
| Mint | `coin::mint(&mint_cap, amount)` | `coin::mint(&mut treasury_cap, amount, ctx)` |
| User balance | `CoinStore<T>` at address | `Coin<T>` object |
| AMM reserves | `Coin<T>` in global resource | `Balance<T>` in Pool object |
| Pool | Shared resource at address | Shared object |

---

**ก่อนหน้า**: [Part 26 - Sui Objects ←](part-26-sui-objects.md)
**ต่อไป**: [Part 28 - DeFi AMM Deep Dive →](part-28-defi-amm.md)
