# Part 42: Protocol Composability

## สารบัญ
- [Composability Design Principles](#composability-design-principles)
- [Interface Patterns](#interface-patterns)
- [Shared Liquidity Layers](#shared-liquidity-layers)
- [Aggregator Pattern](#aggregator-pattern)
- [Position Manager](#position-manager)
- [ตัวอย่าง: DeFi Superset Protocol](#ตัวอย่าง-defi-superset-protocol)

---

## Composability Design Principles

```
Move Composability ต่างจาก EVM:
  EVM: dynamic dispatch via ABI (any address can be called)
  Move: static dispatch (must know module at compile time)
  
ดังนั้น Move composability ทำผ่าน:
  1. Generics (phantom types)
  2. Shared objects / shared resources
  3. Capabilities passed between modules
  4. Hot potato pattern
  5. Events as interface
  6. Return values as composition glue
  
ข้อดี:
  - Type-safe composition
  - No reentrancy attacks
  - Predictable gas costs
  
ข้อเสีย:
  - Less flexible than EVM's dynamic dispatch
  - Requires knowing all protocols at compile time
```

---

## Interface Patterns

```move
// ============================================
// Simulating interfaces via generic constraints
// ============================================

// Define "interface" as a phantom type + set of functions
module interfaces::dex {
    // DEX "interface" marker
    struct DEXInterface<phantom Pool> {}
    
    // Any module implementing these functions is a DEX
    // Not enforceable at type level, but by convention
}

// AMM Module (implements DEX interface)
module amm::constant_product {
    public fun swap_in<X, Y>(
        pool_addr: address,
        token_in: aptos_framework::coin::Coin<X>,
        min_out: u64,
    ): aptos_framework::coin::Coin<Y> acquires Pool<X, Y> {
        // ... implementation
        abort 0
    }
    
    public fun get_price<X, Y>(pool_addr: address, amount: u64): u64 acquires Pool<X, Y> {
        // ... implementation
        0
    }
    
    struct Pool<phantom X, phantom Y> has key {
        reserve_x: u64,
        reserve_y: u64,
        fee_bps: u64,
        coins_x: aptos_framework::coin::Coin<X>,
        coins_y: aptos_framework::coin::Coin<Y>,
    }
}

// Another DEX implementation
module clob::order_book {
    public fun place_order<X, Y>(
        trader: &std::signer,
        price: u64,
        amount: u64,
        is_buy: bool,
    ) {
        // ... order book logic
    }
    
    public fun get_best_price<X, Y>(is_buy: bool): u64 acquires OrderBook<X, Y> {
        // ... get best bid/ask
        0
    }
    
    struct OrderBook<phantom X, phantom Y> has key {
        bids: aptos_std::smart_table::SmartTable<u64, Order>,
        asks: aptos_std::smart_table::SmartTable<u64, Order>,
    }
    
    struct Order has copy, drop, store {
        trader: address,
        amount: u64,
        price: u64,
    }
}
```

---

## Shared Liquidity Layers

```move
module shared::liquidity_layer {
    use std::signer;
    use aptos_framework::coin::{Self, Coin};
    
    // ============================================
    // Shared liquidity pool usable by multiple protocols
    // ============================================
    
    struct LiquidityPool<phantom X> has key {
        coins: Coin<X>,
        total_shares: u64,
        // Protocols that can borrow from this pool
        authorized_protocols: vector<address>,
    }
    
    struct BorrowReceipt<phantom X> {
        protocol: address,
        amount: u64,
    }
    
    // Protocol borrows from shared pool (flash loan style)
    public fun borrow<X>(
        protocol: &signer,
        pool_addr: address,
        amount: u64,
    ): (Coin<X>, BorrowReceipt<X>) acquires LiquidityPool<X> {
        let pool = borrow_global_mut<LiquidityPool<X>>(pool_addr);
        let protocol_addr = signer::address_of(protocol);
        
        assert!(
            std::vector::contains(&pool.authorized_protocols, &protocol_addr),
            1
        );
        
        let borrowed = coin::extract(&mut pool.coins, amount);
        (borrowed, BorrowReceipt { protocol: protocol_addr, amount })
    }
    
    // Protocol returns to shared pool
    public fun repay<X>(
        pool_addr: address,
        coins: Coin<X>,
        receipt: BorrowReceipt<X>,
    ) acquires LiquidityPool<X> {
        let BorrowReceipt { protocol: _, amount: owed } = receipt;
        assert!(coin::value(&coins) >= owed, 2);
        
        let pool = borrow_global_mut<LiquidityPool<X>>(pool_addr);
        coin::merge(&mut pool.coins, coins);
    }
    
    // ============================================
    // Concentrated Liquidity Layer
    // ============================================
    
    struct TickRange has copy, drop, store {
        lower: i32,
        upper: i32,
    }
    
    // Note: i32 implementation needs wrapper in Move
    struct I32 has copy, drop, store {
        value: u32,
        negative: bool,
    }
    
    struct Position has key {
        owner: address,
        pool_addr: address,
        range: TickRange,
        liquidity: u128,
        fee_growth_snapshot_x: u128,
        fee_growth_snapshot_y: u128,
        fees_owed_x: u64,
        fees_owed_y: u64,
    }
    
    // Position NFT for composability (can be used as collateral)
    struct PositionNFT has key {
        id: aptos_framework::object::Object<Position>,
    }
}
```

---

## Aggregator Pattern

```move
module aggregator::swap_aggregator {
    use std::signer;
    use aptos_framework::coin::{Self, Coin};
    
    // ============================================
    // Aggregator: finds best price across DEXes
    // ============================================
    
    struct Route<phantom X, phantom Y> has drop {
        dex_type: u8,      // 0=AMM, 1=CLOB, 2=...
        pool_addr: address,
        split_bps: u64,    // what % to route here (sum=10000)
    }
    
    // Split swap across multiple venues
    public fun split_swap<X, Y>(
        user: &signer,
        mut token_in: Coin<X>,
        routes: vector<Route<X, Y>>,
        min_total_out: u64,
    ): Coin<Y> {
        let mut total_out = coin::zero<Y>();
        let total_in = coin::value(&token_in);
        let n = std::vector::length(&routes);
        
        let mut i = 0u64;
        while (i < n) {
            let route = std::vector::borrow(&routes, i);
            
            // Calculate split amount
            let split_amount = if (i == n - 1) {
                // Last route gets remainder to avoid rounding loss
                coin::value(&token_in)
            } else {
                total_in * route.split_bps / 10_000
            };
            
            let split = coin::extract(&mut token_in, split_amount);
            
            // Route to appropriate DEX
            let out = if (route.dex_type == 0) {
                amm::constant_product::swap_in<X, Y>(route.pool_addr, split, 0)
            } else {
                // Other DEX types...
                coin::zero<Y>()  // placeholder
            };
            
            coin::merge(&mut total_out, out);
            i = i + 1;
        };
        
        coin::destroy_zero(token_in);
        
        assert!(coin::value(&total_out) >= min_total_out, 1);
        total_out
    }
    
    // ============================================
    // Find optimal route (view function)
    // ============================================
    
    #[view]
    public fun quote_best_route<X, Y>(
        amount_in: u64,
        amm_pool_addr: address,
        clob_addr: address,
    ): (u64, u8) {
        // Get AMM quote
        let amm_out = amm::constant_product::get_price<X, Y>(amm_pool_addr, amount_in);
        
        // Get CLOB quote
        let clob_out = clob::order_book::get_best_price<X, Y>(true) * amount_in / 1_000_000;
        
        if (amm_out >= clob_out) {
            (amm_out, 0)  // Use AMM
        } else {
            (clob_out, 1)  // Use CLOB
        }
    }
    
    // ============================================
    // Protocol Fee Rebate for aggregator users
    // ============================================
    
    struct AggregatorState has key {
        admin: address,
        fee_rebate_bps: u64,  // Rebate portion of protocol fees
        total_volume: u64,
        total_rebates: u64,
    }
    
    struct UserRebates has key {
        earned: u64,
        claimed: u64,
    }
    
    public fun accrue_rebate(
        user: address,
        trade_amount: u64,
        agg_addr: address,
    ) acquires AggregatorState, UserRebates {
        let agg = borrow_global<AggregatorState>(agg_addr);
        let rebate = trade_amount * agg.fee_rebate_bps / 10_000;
        
        if (exists<UserRebates>(user)) {
            let rebates = borrow_global_mut<UserRebates>(user);
            rebates.earned = rebates.earned + rebate;
        } else {
            // Can't move_to user without their signer
            // In practice: store in aggregator's table
        };
    }
}
```

---

## Position Manager

```move
module position_mgr::manager {
    use std::signer;
    use aptos_framework::coin;
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::event;
    
    // ============================================
    // Position Manager: abstract over multiple protocols
    // Users interact with one interface for all positions
    // ============================================
    
    struct PositionManager has key {
        admin: address,
        positions: SmartTable<u64, Position>,
        next_position_id: u64,
        total_value_locked: u64,
    }
    
    struct Position has store {
        id: u64,
        owner: address,
        protocol_type: u8,     // 0=AMM LP, 1=Lending, 2=Perp, 3=Options
        protocol_addr: address,
        collateral: u64,
        debt: u64,
        last_update: u64,
        status: u8,            // 0=active, 1=closed, 2=liquidated
    }
    
    // Strategy that can manage positions
    struct Strategy has store {
        strategy_addr: address,
        name: vector<u8>,
        target_leverage: u64,   // e.g., 2x = 20000 (2 decimal places)
        rebalance_threshold_bps: u64,
    }
    
    #[event]
    struct PositionOpened has drop, store {
        position_id: u64,
        owner: address,
        protocol_type: u8,
        amount: u64,
    }
    
    #[event]
    struct PositionClosed has drop, store {
        position_id: u64,
        owner: address,
        final_value: u64,
        pnl_signed: bool,  // true = profit, false = loss
        pnl_amount: u64,
    }
    
    const E_NOT_OWNER: u64 = 1;
    const E_POSITION_NOT_FOUND: u64 = 2;
    const E_ALREADY_CLOSED: u64 = 3;
    
    // ============================================
    // Open position in any protocol
    // ============================================
    
    public entry fun open_amm_position<X, Y>(
        user: &signer,
        manager_addr: address,
        pool_addr: address,
        dx: u64,
        dy: u64,
        min_lp: u64,
    ) acquires PositionManager {
        let user_addr = signer::address_of(user);
        let manager = borrow_global_mut<PositionManager>(manager_addr);
        
        // Open LP position in AMM
        // amm::add_liquidity<X, Y>(user, pool_addr, dx, dy, min_lp);
        
        let position_id = manager.next_position_id;
        manager.next_position_id = position_id + 1;
        
        smart_table::add(&mut manager.positions, position_id, Position {
            id: position_id,
            owner: user_addr,
            protocol_type: 0,  // AMM LP
            protocol_addr: pool_addr,
            collateral: dx + dy,  // simplified
            debt: 0,
            last_update: aptos_framework::timestamp::now_seconds(),
            status: 0,
        });
        
        manager.total_value_locked = manager.total_value_locked + dx + dy;
        
        event::emit(PositionOpened {
            position_id,
            owner: user_addr,
            protocol_type: 0,
            amount: dx + dy,
        });
    }
    
    // ============================================
    // One-click close (handles any protocol type)
    // ============================================
    
    public entry fun close_position(
        user: &signer,
        manager_addr: address,
        position_id: u64,
    ) acquires PositionManager {
        let user_addr = signer::address_of(user);
        let manager = borrow_global_mut<PositionManager>(manager_addr);
        
        assert!(smart_table::contains(&manager.positions, position_id), E_POSITION_NOT_FOUND);
        let position = smart_table::borrow_mut(&mut manager.positions, position_id);
        
        assert!(position.owner == user_addr, E_NOT_OWNER);
        assert!(position.status == 0, E_ALREADY_CLOSED);
        
        // Route to appropriate protocol closer
        if (position.protocol_type == 0) {
            // Close AMM LP position
            // amm::remove_liquidity(user, position.protocol_addr, ...);
        } else if (position.protocol_type == 1) {
            // Close lending position
            // lending::repay_and_withdraw(user, position.protocol_addr, ...);
        } else if (position.protocol_type == 2) {
            // Close perp position
            // perp::close_position(user, position.protocol_addr, ...);
        };
        
        position.status = 1;  // Closed
        
        event::emit(PositionClosed {
            position_id,
            owner: user_addr,
            final_value: position.collateral,  // simplified
            pnl_signed: true,
            pnl_amount: 0,
        });
    }
    
    // ============================================
    // Portfolio view
    // ============================================
    
    #[view]
    public fun get_position(
        manager_addr: address,
        position_id: u64,
    ): (address, u8, u64, u64, u8) acquires PositionManager {
        let manager = borrow_global<PositionManager>(manager_addr);
        let pos = smart_table::borrow(&manager.positions, position_id);
        (pos.owner, pos.protocol_type, pos.collateral, pos.debt, pos.status)
    }
    
    #[view]
    public fun total_value_locked(manager_addr: address): u64 acquires PositionManager {
        borrow_global<PositionManager>(manager_addr).total_value_locked
    }
}
```

---

## ตัวอย่าง: DeFi Superset Protocol

```move
module superset::protocol {
    use std::signer;
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::event;
    
    // ============================================
    // Superset Protocol: composing AMM + Lending + Yield
    // "Leverage yield farming with single transaction"
    // ============================================
    
    // Step 1: Deposit collateral to Lending
    // Step 2: Borrow stablecoin against collateral
    // Step 3: Swap borrowed stablecoin for more collateral
    // Step 4: Deposit to yield farm
    // All in ONE transaction!
    
    struct LeveragePosition has key {
        owner: address,
        collateral_asset: address,  // asset type identifier
        collateral_amount: u64,
        borrowed_amount: u64,
        yield_position: u64,
        leverage: u64,
        protocol_chain: ProtocolChain,
    }
    
    struct ProtocolChain has store {
        lending_addr: address,
        amm_addr: address,
        farm_addr: address,
    }
    
    #[event]
    struct LeveragedPositionOpened has drop, store {
        owner: address,
        collateral: u64,
        total_exposure: u64,
        leverage_x: u64,
    }
    
    // ============================================
    // Open leveraged yield farm position
    // ============================================
    
    public entry fun open_leveraged_position<Collateral, Stable, LP>(
        user: &signer,
        protocols: ProtocolChain,
        initial_collateral_amount: u64,
        leverage: u64,  // e.g., 3 = 3x
        min_lp: u64,
    ) acquires LeveragePosition {
        let user_addr = signer::address_of(user);
        
        // 1. User provides initial collateral
        // let collateral = coin::withdraw<Collateral>(user, initial_collateral_amount);
        
        // 2. Deposit collateral to lending protocol
        // lending::deposit<Collateral>(user, protocols.lending_addr, collateral);
        
        // 3. Borrow stablecoins based on leverage
        let borrow_amount = initial_collateral_amount * (leverage - 1);
        // let stable = lending::borrow<Stable>(user, protocols.lending_addr, borrow_amount);
        
        // 4. Swap stable for more collateral
        // let extra_collateral = amm::swap<Stable, Collateral>(stable, protocols.amm_addr, 0);
        
        // 5. Add all collateral to yield farm (as LP)
        // let lp = farm::deposit<Collateral>(user, protocols.farm_addr, all_collateral, min_lp);
        
        let total_exposure = initial_collateral_amount * leverage;
        
        // Record position
        move_to(user, LeveragePosition {
            owner: user_addr,
            collateral_asset: @0x1,  // placeholder for type
            collateral_amount: initial_collateral_amount,
            borrowed_amount: borrow_amount,
            yield_position: 0,
            leverage,
            protocol_chain: protocols,
        });
        
        event::emit(LeveragedPositionOpened {
            owner: user_addr,
            collateral: initial_collateral_amount,
            total_exposure,
            leverage_x: leverage,
        });
    }
    
    // ============================================
    // Close leveraged position (one transaction)
    // ============================================
    
    public entry fun close_leveraged_position<Collateral, Stable>(
        user: &signer,
        position_owner: address,
    ) acquires LeveragePosition {
        let position = borrow_global_mut<LeveragePosition>(position_owner);
        assert!(signer::address_of(user) == position.owner, 1);
        
        // 1. Withdraw from yield farm
        // let (collateral, rewards) = farm::withdraw(user, position.protocol_chain.farm_addr, position.yield_position);
        
        // 2. Swap enough collateral to repay debt
        // let stable_to_repay = amm::swap<Collateral, Stable>(some_collateral, ..., 0);
        
        // 3. Repay lending protocol
        // lending::repay<Stable>(user, position.protocol_chain.lending_addr, stable_to_repay);
        
        // 4. Withdraw initial collateral
        // lending::withdraw<Collateral>(user, position.protocol_chain.lending_addr, position.collateral_amount);
        
        // 5. User receives: initial_collateral + yield - fees
    }
    
    // ============================================
    // Rebalance if leverage drifts
    // ============================================
    
    public entry fun rebalance<Collateral, Stable>(
        keeper: &signer,
        position_owner: address,
        target_leverage: u64,
    ) acquires LeveragePosition {
        let position = borrow_global<LeveragePosition>(position_owner);
        
        // Calculate current leverage
        let current_leverage = (position.collateral_amount + position.borrowed_amount) 
            / position.collateral_amount;
        
        if (current_leverage > target_leverage) {
            // Deleverage: repay some debt
        } else if (current_leverage < target_leverage) {
            // Leverage up: borrow more
        };
    }
    
    #[view]
    public fun current_health(position_owner: address): u64 acquires LeveragePosition {
        let pos = borrow_global<LeveragePosition>(position_owner);
        // health = collateral_value * ltv / borrowed
        // Returns bps (10000 = 100%)
        if (pos.borrowed_amount == 0) return 10_000;
        pos.collateral_amount * 10_000 / pos.borrowed_amount
    }
}
```

---

## สรุป Composability Patterns

```
Pattern                | Move Implementation     | Use Case
-----------------------|------------------------|------------------
Generic protocols      | phantom types          | Any token pair
Protocol chains        | Multiple acquires      | Multi-step DeFi
Position management    | Central registry       | Portfolio tracking  
Aggregation            | Route + split          | Best price
Leverage               | Borrow + swap loop     | Leveraged yield
Hot potato passing     | Non-drop struct        | Flash loans, callbacks
Shared liquidity       | Capability passing     | Cross-protocol pools

Key Principle:
  In Move: composability = chaining function calls in one transaction
  No callbacks, no dynamic dispatch
  = Safer but requires knowing protocols at compile time
```

---

**ก่อนหน้า**: [Part 41 - Performance ←](part-41-performance.md)
**ต่อไป**: [Part 43 - Token Economics →](part-43-tokenomics.md)
