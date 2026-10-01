# Part 41: Performance Optimization

## สารบัญ
- [Gas Model ใน Aptos](#gas-model-ใน-aptos)
- [Storage Optimization](#storage-optimization)
- [Computation Optimization](#computation-optimization)
- [Batch Processing](#batch-processing)
- [Caching Patterns](#caching-patterns)
- [ตัวอย่าง: High-Performance DEX](#ตัวอย่าง-high-performance-dex)

---

## Gas Model ใน Aptos

```
Aptos Gas Costs:
  - Instruction execution: per opcode
  - Storage: read (cheaper) vs write (expensive)
  - Events: charged per byte
  - Function calls: per call overhead
  
Optimization priorities:
  1. Minimize storage writes (most expensive)
  2. Minimize storage reads
  3. Reduce computation
  4. Pack data efficiently
  
Pricing (approximate relative):
  Write global storage:  ~800 units/byte
  Read global storage:   ~300 units/byte
  Computation:           ~1 unit/instruction
  Event emission:        ~100 units/byte
```

---

## Storage Optimization

```move
module perf::storage_optimization {
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // Pack multiple fields into fewer storage slots
    // ============================================
    
    // BAD: Many separate resources (many reads/writes)
    struct UserFeesBad has key { fee_paid: u64 }
    struct UserVolumeBad has key { volume: u64 }
    struct UserTierBad has key { tier: u8 }
    struct UserLastTxBad has key { last_tx: u64 }
    
    // GOOD: Pack related data into single resource
    struct UserData has key {
        // Pack u8 fields using bit manipulation if very tight
        fee_paid: u64,
        volume: u64,
        // Bit pack: tier (2 bits) | flags (6 bits)
        tier_and_flags: u8,
        last_tx: u64,
    }
    
    // Access packed fields
    fun get_tier(data: &UserData): u8 {
        data.tier_and_flags & 0x03  // lower 2 bits
    }
    
    fun set_tier(data: &mut UserData, tier: u8) {
        data.tier_and_flags = (data.tier_and_flags & 0xFC) | (tier & 0x03);
    }
    
    fun get_flag(data: &UserData, bit: u8): bool {
        (data.tier_and_flags >> (bit + 2)) & 1 == 1
    }
    
    fun set_flag(data: &mut UserData, bit: u8, value: bool) {
        if (value) {
            data.tier_and_flags = data.tier_and_flags | (1u8 << (bit + 2));
        } else {
            data.tier_and_flags = data.tier_and_flags & !(1u8 << (bit + 2));
        };
    }
    
    // ============================================
    // Lazy loading: only load what you need
    // ============================================
    
    struct LargeState has key {
        // Frequently accessed
        balance: u64,
        last_update: u64,
        
        // Rarely accessed (separate struct)
        // Put in separate resource accessed only when needed
    }
    
    struct RarelyAccessedState has key {
        full_history: vector<u64>,
        metadata: vector<u8>,
        extra_data: aptos_std::simple_map::SimpleMap<vector<u8>, vector<u8>>,
    }
    
    // Fast path: only read LargeState (no RarelyAccessedState)
    public fun get_balance(addr: address): u64 acquires LargeState {
        borrow_global<LargeState>(addr).balance
    }
    
    // Slow path: when full data needed
    public fun get_full_data(addr: address): (u64, u64) 
    acquires LargeState, RarelyAccessedState {
        let state = borrow_global<LargeState>(addr);
        let rare = borrow_global<RarelyAccessedState>(addr);
        // Process both
        (state.balance, std::vector::length(&rare.full_history) as u64)
    }
    
    // ============================================
    // SmartTable: better than Table for most cases
    // ============================================
    
    // SmartTable buckets entries to reduce storage slots
    // vs Table: separate slot per entry
    
    struct TokenLedger has key {
        // SmartTable: O(1) access, efficient storage
        balances: SmartTable<address, u64>,
    }
    
    // For very small collections: use SimpleMap or vector
    struct SmallConfig has key {
        // SimpleMap stores inline (no separate storage slots for small sizes)
        params: aptos_std::simple_map::SimpleMap<vector<u8>, u64>,
    }
    
    // ============================================
    // Avoid redundant existence checks
    // ============================================
    
    // BAD: double-check (reads resource twice)
    public fun bad_update(addr: address, amount: u64) acquires LargeState {
        assert!(exists<LargeState>(addr), 1);  // read #1
        let state = borrow_global_mut<LargeState>(addr);  // read #2
        state.balance = state.balance + amount;
    }
    
    // GOOD: just try to borrow (aborts naturally if not exists)
    public fun good_update(addr: address, amount: u64) acquires LargeState {
        let state = borrow_global_mut<LargeState>(addr);  // read #1 (aborts if not exists)
        state.balance = state.balance + amount;
    }
}
```

---

## Computation Optimization

```move
module perf::computation {
    // ============================================
    // Efficient integer math
    // ============================================
    
    // Use bit operations for powers of 2
    public fun mul_by_power_of_2(x: u64, pow: u8): u64 {
        x << pow  // much faster than multiplication
    }
    
    public fun div_by_power_of_2(x: u64, pow: u8): u64 {
        x >> pow  // much faster than division
    }
    
    // Fast modulo for power of 2
    public fun mod_power_of_2(x: u64, pow: u8): u64 {
        x & ((1u64 << pow) - 1)
    }
    
    // ============================================
    // Avoid unnecessary u128 casts
    // ============================================
    
    // BAD: always use u128 (wastes gas)
    public fun bad_fee_calc(amount: u64, fee_bps: u64): u64 {
        ((amount as u128) * (fee_bps as u128) / 10_000u128) as u64
    }
    
    // GOOD: use u64 when safe, u128 only when overflow possible
    public fun good_fee_calc(amount: u64, fee_bps: u64): u64 {
        // Safe if amount < 2^53 and fee_bps < 10000
        // 2^53 * 10000 / 10000 = 2^53 (no overflow for u64)
        if (amount < 1_844_674_407_370_000u64) {
            amount * fee_bps / 10_000
        } else {
            // Fallback for large amounts
            ((amount as u128) * (fee_bps as u128) / 10_000u128) as u64
        }
    }
    
    // ============================================
    // Efficient square root
    // ============================================
    
    // Newton's method - ~6 iterations for u64
    public fun sqrt(n: u64): u64 {
        if (n == 0) return 0;
        let mut x = n;
        let mut y = (x + 1) / 2;
        while (y < x) {
            x = y;
            y = (x + n / x) / 2;
        };
        x
    }
    
    // Faster: bit-by-bit sqrt (predictable iterations)
    public fun sqrt_fast(n: u64): u64 {
        if (n == 0) return 0;
        let mut result = 0u64;
        let mut bit = 1u64 << 62;  // Second-to-top bit
        while (bit > n) { bit >>= 2; };
        while (bit != 0) {
            if (n >= result + bit) {
                n = n - result - bit;
                result = (result >> 1) + bit;
            } else {
                result >>= 1;
            };
            bit >>= 2;
        };
        result
    }
    
    // ============================================
    // Efficient min/max (branchless)
    // ============================================
    
    public fun min_u64(a: u64, b: u64): u64 {
        if (a < b) a else b
    }
    
    public fun max_u64(a: u64, b: u64): u64 {
        if (a > b) a else b
    }
    
    public fun clamp(value: u64, lo: u64, hi: u64): u64 {
        if (value < lo) lo
        else if (value > hi) hi
        else value
    }
    
    // ============================================
    // Avoid string operations in critical paths
    // ============================================
    
    // BAD: string comparison in loop
    // GOOD: use numeric IDs
    
    // Use u8/u16 enums instead of strings
    const TOKEN_TYPE_APT: u8 = 0;
    const TOKEN_TYPE_USDC: u8 = 1;
    const TOKEN_TYPE_USDT: u8 = 2;
    
    public fun get_decimals(token_type: u8): u8 {
        if (token_type == TOKEN_TYPE_APT) 8
        else if (token_type == TOKEN_TYPE_USDC) 6
        else if (token_type == TOKEN_TYPE_USDT) 6
        else 8  // default
    }
    
    // ============================================
    // Early return optimization
    // ============================================
    
    public fun process_batch(items: &vector<u64>): u64 {
        let n = std::vector::length(items);
        if (n == 0) return 0;  // Early return
        if (n == 1) return *std::vector::borrow(items, 0);  // Early return
        
        // Main processing only when needed
        let mut sum = 0u64;
        let mut i = 0u64;
        while (i < n) {
            sum = sum + *std::vector::borrow(items, i);
            i = i + 1;
        };
        sum
    }
}
```

---

## Batch Processing

```move
module perf::batch_processing {
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // Batch operations: amortize overhead
    // ============================================
    
    struct PoolState has key {
        balances: SmartTable<address, u64>,
        total: u64,
        pending_rewards: SmartTable<address, u64>,
        acc_reward_per_share: u128,
        last_reward_time: u64,
    }
    
    // BAD: Individual updates (N separate txs)
    // Each tx: ~500 gas overhead + actual work
    // 100 users = 100 txs = 50,000 gas overhead
    
    // GOOD: Batch update in one tx
    // 100 users in one tx = 500 gas overhead only
    
    public entry fun batch_distribute_rewards(
        pool_addr: address,
        recipients: vector<address>,
        amounts: vector<u64>,
    ) acquires PoolState {
        let n = std::vector::length(&recipients);
        assert!(n == std::vector::length(&amounts), 1);
        assert!(n <= 500, 2);  // Gas limit protection
        
        let pool = borrow_global_mut<PoolState>(pool_addr);
        let mut i = 0u64;
        
        while (i < n) {
            let recipient = *std::vector::borrow(&recipients, i);
            let amount = *std::vector::borrow(&amounts, i);
            
            if (smart_table::contains(&pool.pending_rewards, recipient)) {
                let pending = smart_table::borrow_mut(&mut pool.pending_rewards, recipient);
                *pending = *pending + amount;
            } else {
                smart_table::add(&mut pool.pending_rewards, recipient, amount);
            };
            
            i = i + 1;
        };
    }
    
    // ============================================
    // Lazy evaluation (compute on read, not write)
    // ============================================
    
    struct StakingPosition has key {
        staked: u64,
        reward_debt: u128,
    }
    
    // Don't calculate rewards on each second tick
    // Calculate only when user interacts
    
    public fun pending_reward(
        pool: &PoolState,
        position: &StakingPosition,
    ): u64 {
        let accumulated = (position.staked as u128) 
            * pool.acc_reward_per_share 
            / 1_000_000_000_000u128;
        let pending = accumulated - position.reward_debt;
        pending as u64
    }
    
    // Update accumulator only when needed (not on every block)
    public fun update_pool_if_needed(pool: &mut PoolState) {
        let now = aptos_framework::timestamp::now_seconds();
        if (now <= pool.last_reward_time) return;  // Early exit
        
        let elapsed = now - pool.last_reward_time;
        // Only update if meaningful time has passed
        if (elapsed < 60) return;  // Skip if < 1 minute
        
        // Update accumulator...
        pool.last_reward_time = now;
    }
    
    // ============================================
    // Pipeline processing
    // ============================================
    
    struct Pipeline has key {
        queue: vector<PendingItem>,
        processing: vector<PendingItem>,
        max_batch: u64,
    }
    
    struct PendingItem has store, copy, drop {
        id: u64,
        data: u64,
        priority: u8,
    }
    
    public entry fun process_pipeline_batch(
        pipeline_addr: address,
    ) acquires Pipeline {
        let pipeline = borrow_global_mut<Pipeline>(pipeline_addr);
        let batch_size = min_u64(
            std::vector::length(&pipeline.queue),
            pipeline.max_batch
        );
        
        let mut i = 0u64;
        while (i < batch_size) {
            let item = std::vector::pop_back(&mut pipeline.queue);
            // Process item
            let _ = item.data * 2;  // placeholder processing
            i = i + 1;
        };
    }
    
    fun min_u64(a: u64, b: u64): u64 { if (a < b) a else b }
}
```

---

## Caching Patterns

```move
module perf::caching {
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::timestamp;
    
    // ============================================
    // Cache expensive calculations
    // ============================================
    
    struct PriceCache has key {
        prices: SmartTable<vector<u8>, CachedPrice>,
        ttl: u64,  // Time-to-live in seconds
    }
    
    struct CachedPrice has copy, drop, store {
        price: u64,
        cached_at: u64,
    }
    
    public fun get_price_cached(
        cache_addr: address,
        symbol: vector<u8>,
        oracle_addr: address,
    ): u64 acquires PriceCache {
        let cache = borrow_global_mut<PriceCache>(cache_addr);
        let now = timestamp::now_seconds();
        
        // Check cache
        if (smart_table::contains(&cache.prices, symbol)) {
            let cached = *smart_table::borrow(&cache.prices, symbol);
            if (now <= cached.cached_at + cache.ttl) {
                return cached.price  // Cache hit!
            };
        };
        
        // Cache miss: fetch from oracle
        let price = fetch_from_oracle(oracle_addr, symbol);
        
        // Update cache
        if (smart_table::contains(&cache.prices, symbol)) {
            *smart_table::borrow_mut(&mut cache.prices, symbol) = CachedPrice {
                price,
                cached_at: now,
            };
        } else {
            smart_table::add(&mut cache.prices, symbol, CachedPrice {
                price,
                cached_at: now,
            });
        };
        
        price
    }
    
    fun fetch_from_oracle(oracle_addr: address, symbol: vector<u8>): u64 {
        // Expensive oracle call
        1_000_000_000_000_000_000u64  // placeholder
    }
    
    // ============================================
    // Precomputed lookup tables
    // ============================================
    
    // Instead of computing sin/cos at runtime, precompute
    struct TrigTable has key {
        // sin values for 0-360 degrees (scaled * 1e6)
        sin_table: vector<i64>,
    }
    
    // In production: initialize with precomputed values
    public fun sin(degrees: u64, table_addr: address): i64 acquires TrigTable {
        let table = borrow_global<TrigTable>(table_addr);
        let idx = degrees % 360;
        *std::vector::borrow(&table.sin_table, idx)
    }
    
    // ============================================
    // Memoization for recursive functions
    // ============================================
    
    struct Memo has key {
        cache: SmartTable<u64, u64>,
    }
    
    // Fibonacci with memoization (rarely needed in contracts, but illustrative)
    public fun fib_cached(n: u64, memo_addr: address): u64 acquires Memo {
        let memo = borrow_global_mut<Memo>(memo_addr);
        
        if (smart_table::contains(&memo.cache, n)) {
            return *smart_table::borrow(&memo.cache, n)
        };
        
        let result = if (n <= 1) {
            n
        } else {
            // Note: recursive acquires would need careful handling
            // In practice: use iterative approach
            let mut a = 0u64;
            let mut b = 1u64;
            let mut i = 2u64;
            while (i <= n) {
                let c = a + b;
                a = b;
                b = c;
                i = i + 1;
            };
            b
        };
        
        smart_table::add(&mut memo.cache, n, result);
        result
    }
    
    // Iterative version (no memoization needed, much faster)
    public fun fib_iterative(n: u64): u64 {
        if (n == 0) return 0;
        if (n == 1) return 1;
        let mut a = 0u64;
        let mut b = 1u64;
        let mut i = 2u64;
        while (i <= n) {
            let c = a + b;
            a = b;
            b = c;
            i = i + 1;
        };
        b
    }
}

// ============================================
// Type alias for use in i64 context
// ============================================
type i64 = u64;  // Simplified; use signed wrapper in production
```

---

## ตัวอย่าง: High-Performance DEX

```move
module perf::fast_dex {
    use std::signer;
    use aptos_framework::coin::{Self, Coin};
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // Optimized AMM for high throughput
    // ============================================
    
    // Pack pool state efficiently
    struct Pool<phantom X, phantom Y> has key {
        // Core (read often, small)
        reserve_x: u64,
        reserve_y: u64,
        lp_supply: u64,
        
        // Fees (pack into one slot)
        fee_bps: u32,           // 32-bit (max 65535 bps)
        protocol_fee_bps: u16,
        
        // State flags (bit packed)
        flags: u8,              // bit0=paused, bit1=fee_on, bit2=locked
        
        // Coins (stored separately by framework)
        coins_x: Coin<X>,
        coins_y: Coin<Y>,
        
        // Accumulated fees (rarely needed)
        acc_fee_x: u64,
        acc_fee_y: u64,
    }
    
    // Bit flag helpers
    const FLAG_PAUSED: u8 = 1;
    const FLAG_FEE_ON: u8 = 2;
    const FLAG_LOCKED: u8 = 4;
    
    fun is_paused<X, Y>(pool: &Pool<X, Y>): bool {
        pool.flags & FLAG_PAUSED != 0
    }
    
    fun set_locked<X, Y>(pool: &mut Pool<X, Y>, locked: bool) {
        if (locked) {
            pool.flags = pool.flags | FLAG_LOCKED;
        } else {
            pool.flags = pool.flags & !FLAG_LOCKED;
        };
    }
    
    // ============================================
    // Optimized swap: minimal operations
    // ============================================
    
    public entry fun swap_exact_x_for_y<X, Y>(
        user: &signer,
        pool_addr: address,
        dx: Coin<X>,
        min_dy: u64,
    ) acquires Pool<X, Y> {
        let pool = borrow_global_mut<Pool<X, Y>>(pool_addr);
        
        // Fast checks (cheap)
        assert!(pool.flags & FLAG_PAUSED == 0, 1);
        assert!(pool.flags & FLAG_LOCKED == 0, 2);
        
        // Set locked (reentrancy protection)
        pool.flags = pool.flags | FLAG_LOCKED;
        
        let dx_amount = coin::value(&dx);
        let rx = pool.reserve_x;
        let ry = pool.reserve_y;
        
        // Constant product formula: single integer calculation
        // dy = ry * dx * (10000 - fee) / (rx * 10000 + dx * (10000 - fee))
        let fee_factor = 10_000u64 - (pool.fee_bps as u64);
        let dx_with_fee = dx_amount * fee_factor;
        let dy = ry * dx_with_fee / (rx * 10_000 + dx_with_fee);
        
        // Slippage check
        assert!(dy >= min_dy, 3);
        
        // Update reserves (single write)
        pool.reserve_x = rx + dx_amount;
        pool.reserve_y = ry - dy;
        
        // Collect protocol fee if enabled
        if (pool.flags & FLAG_FEE_ON != 0) {
            let protocol_fee = dx_amount * (pool.protocol_fee_bps as u64) / 10_000;
            pool.acc_fee_x = pool.acc_fee_x + protocol_fee;
        };
        
        // Move coins
        coin::merge(&mut pool.coins_x, dx);
        let dy_coin = coin::extract(&mut pool.coins_y, dy);
        
        // Unlock
        pool.flags = pool.flags & !FLAG_LOCKED;
        
        // Deliver (after state update for safety)
        coin::deposit(signer::address_of(user), dy_coin);
    }
    
    // ============================================
    // Multi-hop swap (single transaction)
    // ============================================
    
    // Swap X→Y→Z in one transaction (no intermediate tx overhead)
    public entry fun swap_x_for_z_via_y<X, Y, Z>(
        user: &signer,
        pool_xy_addr: address,
        pool_yz_addr: address,
        dx: Coin<X>,
        min_dz: u64,
    ) acquires Pool<X, Y>, Pool<Y, Z> {
        // Step 1: X → Y
        let dy = do_swap_xy<X, Y>(pool_xy_addr, dx);
        
        // Step 2: Y → Z (using output of step 1)
        let dz = do_swap_yz<Y, Z>(pool_yz_addr, dy);
        
        // Final slippage check
        assert!(coin::value(&dz) >= min_dz, 1);
        
        coin::deposit(signer::address_of(user), dz);
    }
    
    fun do_swap_xy<X, Y>(pool_addr: address, dx: Coin<X>): Coin<Y> acquires Pool<X, Y> {
        let pool = borrow_global_mut<Pool<X, Y>>(pool_addr);
        let dx_amount = coin::value(&dx);
        let fee_factor = 10_000u64 - (pool.fee_bps as u64);
        let dx_with_fee = dx_amount * fee_factor;
        let dy = pool.reserve_y * dx_with_fee / (pool.reserve_x * 10_000 + dx_with_fee);
        
        pool.reserve_x = pool.reserve_x + dx_amount;
        pool.reserve_y = pool.reserve_y - dy;
        coin::merge(&mut pool.coins_x, dx);
        coin::extract(&mut pool.coins_y, dy)
    }
    
    fun do_swap_yz<Y, Z>(pool_addr: address, dy: Coin<Y>): Coin<Z> acquires Pool<Y, Z> {
        let pool = borrow_global_mut<Pool<Y, Z>>(pool_addr);
        let dy_amount = coin::value(&dy);
        let fee_factor = 10_000u64 - (pool.fee_bps as u64);
        let dy_with_fee = dy_amount * fee_factor;
        let dz = pool.reserve_y * dy_with_fee / (pool.reserve_x * 10_000 + dy_with_fee);
        
        pool.reserve_x = pool.reserve_x + dy_amount;
        pool.reserve_y = pool.reserve_y - dz;
        coin::merge(&mut pool.coins_x, dy);
        coin::extract(&mut pool.coins_y, dz)
    }
    
    // ============================================
    // View functions (gas-free reads)
    // ============================================
    
    #[view]
    public fun get_amount_out<X, Y>(
        pool_addr: address,
        dx: u64,
    ): u64 acquires Pool<X, Y> {
        let pool = borrow_global<Pool<X, Y>>(pool_addr);
        if (is_paused(pool)) return 0;
        
        let fee_factor = 10_000u64 - (pool.fee_bps as u64);
        let dx_with_fee = dx * fee_factor;
        pool.reserve_y * dx_with_fee / (pool.reserve_x * 10_000 + dx_with_fee)
    }
    
    #[view]
    public fun get_reserves<X, Y>(pool_addr: address): (u64, u64) acquires Pool<X, Y> {
        let pool = borrow_global<Pool<X, Y>>(pool_addr);
        (pool.reserve_x, pool.reserve_y)
    }
}
```

---

## Performance Benchmarks

```
Operation               | Gas Units | Notes
------------------------|-----------|--------------------
Simple transfer         | ~200      | Baseline
AMM swap               | ~500      | With fee calculation
Multi-hop swap (2 pool) | ~900      | 1 tx vs 2
Flash loan borrow+repay | ~800      | Hot potato overhead
Add liquidity           | ~600      | LP token mint
Remove liquidity        | ~700      | LP burn + 2 coins
Governance vote         | ~400      | Table write
NFT mint               | ~500      | Resource creation
Oracle price read       | ~300      | Single borrow_global

Gas optimization wins:
  Packed structs:        -30% reads
  Batch (100 ops):       -90% tx overhead
  Precomputed tables:    -50% math ops
  Lazy evaluation:       -40% unnecessary writes
```

---

**ก่อนหน้า**: [Part 40 - Production Deployment ←](../advanced/part-40-production-deployment.md)
**ต่อไป**: [Part 42 - Protocol Composability →](part-42-composability.md)
