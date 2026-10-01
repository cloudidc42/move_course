# Part 16: Events

## สารบัญ
- [Events คืออะไร](#events-คืออะไร)
- [EventHandle (Aptos)](#eventhandle-aptos)
- [emit_event (Aptos)](#emit_event-aptos)
- [Events ใน Sui](#events-ใน-sui)
- [Event Best Practices](#event-best-practices)
- [ตัวอย่างโปรแกรม: DEX Events](#ตัวอย่างโปรแกรม-dex-events)

---

## Events คืออะไร

Events คือข้อมูลที่ถูกบันทึกไว้ใน blockchain ระหว่าง transaction execution ซึ่ง:
- ถูก emit เมื่อเกิด state change ที่สำคัญ
- สามารถ query ได้จาก clients/indexers
- ไม่ถูกเก็บใน global storage (ประหยัด gas)
- ใช้สำหรับ off-chain notifications

```
On-chain State  →  emits Event  →  stored in event log
                                         ↓
                               Off-chain indexer
                                         ↓
                               Frontend/API reads
```

---

## EventHandle (Aptos)

```move
module learning::aptos_events {
    use std::signer;
    use aptos_framework::event::{Self, EventHandle};
    use aptos_framework::account;
    
    // ============================================
    // Event Type Definition
    // ============================================
    
    // Event structs ต้องมี drop + store abilities
    struct MintEvent has drop, store {
        to: address,
        amount: u64,
        timestamp: u64,
    }
    
    struct TransferEvent has drop, store {
        from: address,
        to: address,
        amount: u64,
        fee: u64,
    }
    
    struct BurnEvent has drop, store {
        from: address,
        amount: u64,
    }
    
    // ============================================
    // EventHandle stored in Resource
    // ============================================
    
    struct TokenEvents has key {
        mint_events: EventHandle<MintEvent>,
        transfer_events: EventHandle<TransferEvent>,
        burn_events: EventHandle<BurnEvent>,
    }
    
    struct Token has key {
        balance: u64,
    }
    
    // ============================================
    // Initialize Events
    // ============================================
    
    public entry fun initialize(account: &signer) {
        move_to(account, TokenEvents {
            mint_events: account::new_event_handle<MintEvent>(account),
            transfer_events: account::new_event_handle<TransferEvent>(account),
            burn_events: account::new_event_handle<BurnEvent>(account),
        });
        move_to(account, Token { balance: 0 });
    }
    
    // ============================================
    // Emitting Events
    // ============================================
    
    public entry fun mint(
        admin: &signer,
        to: address,
        amount: u64,
        timestamp: u64,
    ) acquires TokenEvents, Token {
        let admin_addr = signer::address_of(admin);
        
        // Get mutable reference to Token
        let token = borrow_global_mut<Token>(to);
        token.balance = token.balance + amount;
        
        // Emit event
        let events = borrow_global_mut<TokenEvents>(admin_addr);
        event::emit_event(
            &mut events.mint_events,
            MintEvent { to, amount, timestamp }
        );
    }
    
    public entry fun transfer(
        from: &signer,
        to: address,
        events_addr: address,
        amount: u64,
        fee: u64,
        timestamp: u64,
    ) acquires TokenEvents, Token {
        let from_addr = signer::address_of(from);
        
        // Execute transfer
        let from_token = borrow_global_mut<Token>(from_addr);
        assert!(from_token.balance >= amount + fee, 1);
        from_token.balance = from_token.balance - amount - fee;
        
        let to_token = borrow_global_mut<Token>(to);
        to_token.balance = to_token.balance + amount;
        
        // Emit event
        let events = borrow_global_mut<TokenEvents>(events_addr);
        event::emit_event(
            &mut events.transfer_events,
            TransferEvent { from: from_addr, to, amount, fee }
        );
        
        let _ = timestamp;
    }
    
    // ============================================
    // Event counters
    // ============================================
    
    public fun mint_event_count(events_addr: address): u64 acquires TokenEvents {
        let events = borrow_global<TokenEvents>(events_addr);
        event::counter(&events.mint_events)
    }
}
```

---

## emit_event (Aptos) - New Style

```move
module learning::aptos_events_v2 {
    use aptos_framework::event;
    
    // ============================================
    // New Event Style (Aptos Framework v2)
    // ============================================
    
    // #[event] attribute - no need for EventHandle
    #[event]
    struct SwapEvent has drop, store {
        pool: address,
        user: address,
        token_in: address,
        token_out: address,
        amount_in: u64,
        amount_out: u64,
        timestamp: u64,
    }
    
    #[event]
    struct LiquidityEvent has drop, store {
        pool: address,
        user: address,
        amount_a: u64,
        amount_b: u64,
        lp_amount: u64,
        is_add: bool,  // true = add, false = remove
    }
    
    // ============================================
    // Emitting with new style
    // ============================================
    
    public fun emit_swap(
        pool: address,
        user: address,
        token_in: address,
        token_out: address,
        amount_in: u64,
        amount_out: u64,
        timestamp: u64,
    ) {
        // New style: event::emit directly
        event::emit(SwapEvent {
            pool,
            user,
            token_in,
            token_out,
            amount_in,
            amount_out,
            timestamp,
        });
    }
    
    public fun emit_liquidity(
        pool: address,
        user: address,
        amount_a: u64,
        amount_b: u64,
        lp_amount: u64,
        is_add: bool,
    ) {
        event::emit(LiquidityEvent {
            pool, user, amount_a, amount_b, lp_amount, is_add
        });
    }
}
```

---

## Events ใน Sui

```move
module learning::sui_events {
    use sui::event;
    
    // ============================================
    // Sui Event System
    // ============================================
    
    // ใน Sui ไม่ต้องการ EventHandle
    // emit ได้โดยตรงด้วย sui::event::emit
    
    // Event structs ต้องมี copy + drop abilities
    struct NFTMinted has copy, drop {
        object_id: address,  // UID ของ NFT
        creator: address,
        name: vector<u8>,
        timestamp: u64,
    }
    
    struct NFTTransferred has copy, drop {
        object_id: address,
        from: address,
        to: address,
        price: u64,
        timestamp: u64,
    }
    
    // ============================================
    // Emitting Events in Sui
    // ============================================
    
    public fun emit_nft_minted(
        object_id: address,
        creator: address,
        name: vector<u8>,
        timestamp: u64,
    ) {
        event::emit(NFTMinted {
            object_id,
            creator,
            name,
            timestamp,
        });
    }
    
    public fun emit_nft_transferred(
        object_id: address,
        from: address,
        to: address,
        price: u64,
        timestamp: u64,
    ) {
        event::emit(NFTTransferred {
            object_id, from, to, price, timestamp
        });
    }
    
    // ============================================
    // Sui NFT Module with Events
    // ============================================
    
    use sui::object::{Self, UID};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};
    
    struct GameNFT has key {
        id: UID,
        name: vector<u8>,
        level: u64,
        experience: u64,
    }
    
    struct NFTLevelUp has copy, drop {
        nft_id: address,
        old_level: u64,
        new_level: u64,
        experience_gained: u64,
    }
    
    public fun level_up(nft: &mut GameNFT, exp_gained: u64, ctx: &TxContext) {
        let old_level = nft.level;
        nft.experience = nft.experience + exp_gained;
        
        // Simple leveling formula
        let needed_exp = nft.level * nft.level * 100;
        if (nft.experience >= needed_exp) {
            nft.level = nft.level + 1;
            nft.experience = nft.experience - needed_exp;
        };
        
        event::emit(NFTLevelUp {
            nft_id: object::uid_to_address(&nft.id),
            old_level,
            new_level: nft.level,
            experience_gained: exp_gained,
        });
    }
}
```

---

## Event Best Practices

```move
module learning::event_best_practices {
    use aptos_framework::event;
    
    // ============================================
    // What to include in events
    // ============================================
    
    // ✅ Include: relevant state changes
    #[event]
    struct GoodTransferEvent has drop, store {
        from: address,           // Who sent
        to: address,             // Who received
        amount: u64,             // How much
        fee: u64,                // Cost
        new_balance_from: u64,   // Post-state (helpful for indexers)
        new_balance_to: u64,     // Post-state
        timestamp: u64,          // When
        tx_version: u64,         // Ordering
    }
    
    // ❌ Avoid: redundant or unnecessary data
    // #[event]
    // struct BadTransferEvent has drop, store {
    //     // Missing: amounts, balances, timestamp
    //     from: address,
    //     to: address,
    // }
    
    // ============================================
    // Event hierarchy
    // ============================================
    
    // High-level protocol event
    #[event]
    struct ProtocolEvent has drop, store {
        event_type: u8,  // 1=mint, 2=burn, 3=transfer
        data: vector<u8>,  // encoded event data
        timestamp: u64,
    }
    
    // Specific events (prefer these over generic)
    #[event]
    struct DepositEvent has drop, store {
        user: address,
        token: address,
        amount: u64,
        shares_minted: u64,
        timestamp: u64,
    }
    
    #[event]
    struct WithdrawEvent has drop, store {
        user: address,
        token: address,
        amount: u64,
        shares_burned: u64,
        timestamp: u64,
    }
    
    // ============================================
    // Emit events at the right time
    // ============================================
    
    public fun good_transfer_pattern(from: address, to: address, amount: u64) {
        // 1. Validate inputs
        // 2. Execute state changes
        // 3. Emit event AFTER successful state change
        
        event::emit(GoodTransferEvent {
            from,
            to,
            amount,
            fee: 0,
            new_balance_from: 0,  // would be real balances
            new_balance_to: amount,
            timestamp: 0,
            tx_version: 0,
        });
    }
    
    // ============================================
    // Indexing-friendly patterns
    // ============================================
    
    // Include sequence numbers for ordering
    #[event]
    struct OrderedEvent has drop, store {
        sequence: u64,    // monotonically increasing
        event_type: u8,
        data_hash: vector<u8>,
        timestamp: u64,
    }
    
    // Include enough context for off-chain reconstruction
    #[event]
    struct RichEvent has drop, store {
        // Who
        initiator: address,
        
        // What
        action: u8,       // enum-like
        token: address,
        amount: u64,
        
        // Before state
        balance_before: u64,
        
        // After state
        balance_after: u64,
        
        // When
        block_timestamp: u64,
        
        // Extra
        reference_id: u64,  // for linking related events
    }
}
```

---

## ตัวอย่างโปรแกรม: DEX Events

```move
module learning::dex_with_events {
    use std::signer;
    use aptos_framework::event;
    use aptos_framework::account;
    
    // ============================================
    // Event Types
    // ============================================
    
    #[event]
    struct PoolCreatedEvent has drop, store {
        pool_address: address,
        token_a: address,
        token_b: address,
        creator: address,
        fee_bps: u64,
        timestamp: u64,
    }
    
    #[event]
    struct SwapEvent has drop, store {
        pool_address: address,
        user: address,
        token_in: address,
        token_out: address,
        amount_in: u64,
        amount_out: u64,
        fee_amount: u64,
        price_impact_bps: u64,
        timestamp: u64,
    }
    
    #[event]
    struct LiquidityAddedEvent has drop, store {
        pool_address: address,
        provider: address,
        amount_a: u64,
        amount_b: u64,
        lp_minted: u64,
        pool_share_bps: u64,  // share of pool in bps
        timestamp: u64,
    }
    
    #[event]
    struct LiquidityRemovedEvent has drop, store {
        pool_address: address,
        provider: address,
        amount_a: u64,
        amount_b: u64,
        lp_burned: u64,
        timestamp: u64,
    }
    
    #[event]
    struct FeesClaimedEvent has drop, store {
        pool_address: address,
        claimer: address,
        fee_amount_a: u64,
        fee_amount_b: u64,
        timestamp: u64,
    }
    
    // ============================================
    // Pool State
    // ============================================
    
    struct Pool has key {
        token_a: address,
        token_b: address,
        reserve_a: u64,
        reserve_b: u64,
        lp_total: u64,
        fee_bps: u64,
        fee_accumulated_a: u64,
        fee_accumulated_b: u64,
        swap_count: u64,
        volume_24h: u64,
    }
    
    // ============================================
    // Core Functions with Events
    // ============================================
    
    public entry fun create_pool(
        creator: &signer,
        token_a: address,
        token_b: address,
        initial_a: u64,
        initial_b: u64,
        fee_bps: u64,
        timestamp: u64,
    ) {
        let creator_addr = signer::address_of(creator);
        
        let pool_address = creator_addr;  // simplified
        
        move_to(creator, Pool {
            token_a,
            token_b,
            reserve_a: initial_a,
            reserve_b: initial_b,
            lp_total: calculate_initial_lp(initial_a, initial_b),
            fee_bps,
            fee_accumulated_a: 0,
            fee_accumulated_b: 0,
            swap_count: 0,
            volume_24h: 0,
        });
        
        event::emit(PoolCreatedEvent {
            pool_address,
            token_a,
            token_b,
            creator: creator_addr,
            fee_bps,
            timestamp,
        });
    }
    
    public entry fun swap(
        user: &signer,
        pool_addr: address,
        token_in: address,
        amount_in: u64,
        min_out: u64,
        timestamp: u64,
    ) acquires Pool {
        let user_addr = signer::address_of(user);
        let pool = borrow_global_mut<Pool>(pool_addr);
        
        // Determine direction
        let (reserve_in, reserve_out) = if (pool.token_a == token_in) {
            (pool.reserve_a, pool.reserve_b)
        } else {
            (pool.reserve_b, pool.reserve_a)
        };
        
        let token_out = if (pool.token_a == token_in) {
            pool.token_b
        } else {
            pool.token_a
        };
        
        // Calculate amounts
        let fee_amount = amount_in * pool.fee_bps / 10_000;
        let amount_in_after_fee = amount_in - fee_amount;
        let amount_out = (amount_in_after_fee * reserve_out) / 
                         (reserve_in + amount_in_after_fee);
        
        assert!(amount_out >= min_out, 1);
        
        // Calculate price impact
        let price_before = reserve_out * 10_000 / reserve_in;
        let price_after = (reserve_out - amount_out) * 10_000 / (reserve_in + amount_in);
        let price_impact_bps = if (price_before > price_after) {
            (price_before - price_after) * 10_000 / price_before
        } else {
            0
        };
        
        // Update state
        if (pool.token_a == token_in) {
            pool.reserve_a = pool.reserve_a + amount_in;
            pool.reserve_b = pool.reserve_b - amount_out;
            pool.fee_accumulated_a = pool.fee_accumulated_a + fee_amount;
        } else {
            pool.reserve_b = pool.reserve_b + amount_in;
            pool.reserve_a = pool.reserve_a - amount_out;
            pool.fee_accumulated_b = pool.fee_accumulated_b + fee_amount;
        };
        
        pool.swap_count = pool.swap_count + 1;
        pool.volume_24h = pool.volume_24h + amount_in;
        
        // Emit swap event with rich context
        event::emit(SwapEvent {
            pool_address: pool_addr,
            user: user_addr,
            token_in,
            token_out,
            amount_in,
            amount_out,
            fee_amount,
            price_impact_bps,
            timestamp,
        });
    }
    
    public entry fun add_liquidity(
        provider: &signer,
        pool_addr: address,
        amount_a: u64,
        amount_b: u64,
        timestamp: u64,
    ) acquires Pool {
        let provider_addr = signer::address_of(provider);
        let pool = borrow_global_mut<Pool>(pool_addr);
        
        // Calculate LP tokens
        let lp_a = amount_a * pool.lp_total / pool.reserve_a;
        let lp_b = amount_b * pool.lp_total / pool.reserve_b;
        let lp_minted = if (lp_a < lp_b) { lp_a } else { lp_b };
        
        let pool_share_bps = lp_minted * 10_000 / (pool.lp_total + lp_minted);
        
        pool.reserve_a = pool.reserve_a + amount_a;
        pool.reserve_b = pool.reserve_b + amount_b;
        pool.lp_total = pool.lp_total + lp_minted;
        
        event::emit(LiquidityAddedEvent {
            pool_address: pool_addr,
            provider: provider_addr,
            amount_a,
            amount_b,
            lp_minted,
            pool_share_bps,
            timestamp,
        });
    }
    
    public entry fun remove_liquidity(
        provider: &signer,
        pool_addr: address,
        lp_amount: u64,
        timestamp: u64,
    ) acquires Pool {
        let provider_addr = signer::address_of(provider);
        let pool = borrow_global_mut<Pool>(pool_addr);
        
        // Calculate amounts
        let amount_a = lp_amount * pool.reserve_a / pool.lp_total;
        let amount_b = lp_amount * pool.reserve_b / pool.lp_total;
        
        pool.reserve_a = pool.reserve_a - amount_a;
        pool.reserve_b = pool.reserve_b - amount_b;
        pool.lp_total = pool.lp_total - lp_amount;
        
        event::emit(LiquidityRemovedEvent {
            pool_address: pool_addr,
            provider: provider_addr,
            amount_a,
            amount_b,
            lp_burned: lp_amount,
            timestamp,
        });
    }
    
    // ============================================
    // View Functions
    // ============================================
    
    #[view]
    public fun get_pool_stats(pool_addr: address): (u64, u64, u64, u64) acquires Pool {
        let pool = borrow_global<Pool>(pool_addr);
        (pool.reserve_a, pool.reserve_b, pool.lp_total, pool.swap_count)
    }
    
    // ============================================
    // Helpers
    // ============================================
    
    fun calculate_initial_lp(amount_a: u64, amount_b: u64): u64 {
        // Geometric mean sqrt(a * b), simplified here
        let product = amount_a * amount_b;
        // Simple square root approximation
        let mut x = product;
        if (x > 0) {
            let mut y = (x + 1) / 2;
            while (y < x) {
                x = y;
                y = (x + product / x) / 2;
            };
        };
        x
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test(creator = @0x1, user = @0x2)]
    public fun test_dex_events(creator: &signer, user: &signer) acquires Pool {
        let creator_addr = signer::address_of(creator);
        
        create_pool(
            creator,
            @0x10,  // token_a
            @0x11,  // token_b
            10_000,
            10_000,
            30,  // 0.3% fee
            0,
        );
        
        // Swap
        swap(user, creator_addr, @0x10, 1_000, 900, 0);
        
        let (ra, rb, lp, swaps) = get_pool_stats(creator_addr);
        assert!(ra == 11_000, 0);  // increased
        assert!(rb < 10_000, 1);   // decreased
        assert!(swaps == 1, 2);
        let _ = lp;
    }
}
```

---

## สรุป Events

| Platform | Style | Usage |
|----------|-------|-------|
| Aptos (old) | `EventHandle<T>` + `emit_event` | Legacy |
| Aptos (new) | `#[event]` + `event::emit` | Recommended |
| Sui | `sui::event::emit` | Direct emit |

---

**ต่อไป**: [Part 17 - Vectors Deep Dive →](part-17-vectors.md)
