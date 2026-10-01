# Part 33: Oracle Integration & Price Feeds

## สารบัญ
- [Oracle คืออะไร](#oracle-คืออะไร)
- [Pyth Network Integration (Aptos)](#pyth-network-integration-aptos)
- [Switchboard Integration](#switchboard-integration)
- [TWAP Oracle (on-chain)](#twap-oracle-on-chain)
- [Oracle Aggregator](#oracle-aggregator)
- [ตัวอย่าง: DeFi ใช้ Oracle](#ตัวอย่าง-defi-ใช้-oracle)

---

## Oracle คืออะไร

```
Real World Data → Oracle → Blockchain Smart Contract

ตัวอย่างข้อมูล:
  - ราคา Token (ETH/USD, BTC/USD, APT/USD)
  - อัตราแลกเปลี่ยน
  - ผลกีฬา, อากาศ
  - ราคาสินค้าโภคภัณฑ์

Oracle บน Aptos:
  - Pyth Network (push model, high frequency)
  - Switchboard (pull model)
  - Chainlink (limited on Aptos)
  - DIY on-chain TWAP (ใช้ AMM pool)
```

---

## Pyth Network Integration (Aptos)

```move
module defi::pyth_oracle {
    use std::signer;
    use aptos_framework::timestamp;
    
    // ============================================
    // Pyth price feed integration
    // Pyth uses "push" model: price updated on-chain by oracle
    // ============================================
    
    // Pyth price struct (from pyth SDK)
    struct Price has copy, drop, store {
        price: I64,      // price as i64 (with exponent)
        conf: u64,       // confidence interval
        expo: I32,       // exponent (price * 10^expo = real price)
        timestamp: u64,
    }
    
    // Since Move doesn't have native i64/i32, use wrapper
    struct I64 has copy, drop, store {
        magnitude: u64,
        negative: bool,
    }
    
    struct I32 has copy, drop, store {
        magnitude: u32,
        negative: bool,
    }
    
    // Our Price feed config
    struct PriceFeedConfig has key {
        admin: address,
        feeds: aptos_std::smart_table::SmartTable<vector<u8>, PriceFeedInfo>,
        staleness_threshold: u64,  // max age in seconds
    }
    
    struct PriceFeedInfo has copy, drop, store {
        feed_id: vector<u8>,      // Pyth price feed ID (32 bytes)
        symbol: vector<u8>,
        decimals: u8,
        last_price: u64,          // normalized price (18 decimals)
        last_conf: u64,
        last_update: u64,
        enabled: bool,
    }
    
    const E_STALE_PRICE: u64 = 1;
    const E_INVALID_PRICE: u64 = 2;
    const E_HIGH_CONFIDENCE: u64 = 3;
    const E_NOT_ADMIN: u64 = 4;
    const E_FEED_NOT_FOUND: u64 = 5;
    
    const MAX_CONF_RATIO_BPS: u64 = 200;  // 2% max confidence ratio
    const DEFAULT_STALENESS: u64 = 60;    // 60 seconds
    const WAD: u64 = 1_000_000_000_000_000_000;  // 1e18
    
    // ============================================
    // Initialize oracle
    // ============================================
    
    public entry fun initialize(admin: &signer) {
        move_to(admin, PriceFeedConfig {
            admin: signer::address_of(admin),
            feeds: aptos_std::smart_table::new(),
            staleness_threshold: DEFAULT_STALENESS,
        });
    }
    
    // ============================================
    // Register a price feed
    // ============================================
    
    public entry fun register_feed(
        admin: &signer,
        config_addr: address,
        feed_id: vector<u8>,
        symbol: vector<u8>,
        decimals: u8,
    ) acquires PriceFeedConfig {
        use aptos_std::smart_table;
        let config = borrow_global_mut<PriceFeedConfig>(config_addr);
        assert!(signer::address_of(admin) == config.admin, E_NOT_ADMIN);
        
        smart_table::add(&mut config.feeds, symbol, PriceFeedInfo {
            feed_id,
            symbol,
            decimals,
            last_price: 0,
            last_conf: 0,
            last_update: 0,
            enabled: true,
        });
    }
    
    // ============================================
    // Update price (called by keeper/oracle service)
    // ============================================
    
    // In production: Pyth SDK provides pyth_price: pyth::price_feeds::Price
    // Here we simulate with raw values
    public entry fun update_price(
        keeper: &signer,
        config_addr: address,
        symbol: vector<u8>,
        raw_price: u64,    // price * 10^decimals
        confidence: u64,
        decimals: u8,
    ) acquires PriceFeedConfig {
        use aptos_std::smart_table;
        let config = borrow_global_mut<PriceFeedConfig>(config_addr);
        assert!(smart_table::contains(&config.feeds, symbol), E_FEED_NOT_FOUND);
        
        let info = smart_table::borrow_mut(&mut config.feeds, symbol);
        assert!(info.enabled, E_INVALID_PRICE);
        
        // Normalize to 18 decimals
        let normalized = normalize_price(raw_price, decimals);
        
        info.last_price = normalized;
        info.last_conf = normalize_price(confidence, decimals);
        info.last_update = timestamp::now_seconds();
    }
    
    fun normalize_price(price: u64, decimals: u8): u64 {
        // Convert to 18 decimals
        if (decimals == 18) return price;
        if (decimals < 18) {
            let multiplier = pow10((18 - decimals) as u64);
            price * multiplier
        } else {
            let divisor = pow10((decimals - 18) as u64);
            price / divisor
        }
    }
    
    fun pow10(n: u64): u64 {
        let mut result = 1u64;
        let mut i = 0u64;
        while (i < n) {
            result = result * 10;
            i = i + 1;
        };
        result
    }
    
    // ============================================
    // Get price (with staleness and confidence checks)
    // ============================================
    
    public fun get_price(
        config_addr: address,
        symbol: vector<u8>,
    ): u64 acquires PriceFeedConfig {
        use aptos_std::smart_table;
        let config = borrow_global<PriceFeedConfig>(config_addr);
        assert!(smart_table::contains(&config.feeds, symbol), E_FEED_NOT_FOUND);
        
        let info = smart_table::borrow(&config.feeds, symbol);
        let now = timestamp::now_seconds();
        
        // Check staleness
        assert!(
            now <= info.last_update + config.staleness_threshold,
            E_STALE_PRICE
        );
        
        // Check confidence ratio (conf/price < max)
        if (info.last_price > 0) {
            let conf_ratio = info.last_conf * 10_000 / info.last_price;
            assert!(conf_ratio <= MAX_CONF_RATIO_BPS, E_HIGH_CONFIDENCE);
        };
        
        info.last_price
    }
    
    // Get price without staleness check (read-only, for display)
    #[view]
    public fun get_price_unsafe(
        config_addr: address,
        symbol: vector<u8>,
    ): (u64, u64, u64) acquires PriceFeedConfig {
        use aptos_std::smart_table;
        let config = borrow_global<PriceFeedConfig>(config_addr);
        let info = smart_table::borrow(&config.feeds, symbol);
        (info.last_price, info.last_conf, info.last_update)
    }
}
```

---

## TWAP Oracle (on-chain)

```move
module defi::twap_oracle {
    use aptos_framework::timestamp;
    
    // ============================================
    // Time-Weighted Average Price
    // Calculated from AMM pool price history
    // ============================================
    
    struct TWAPState has key {
        // Cumulative price * time
        price_cumulative_last: u128,
        // Last snapshot time
        block_timestamp_last: u64,
        // Current TWAP (updated periodically)
        price_average: u64,
        // Price average window (e.g., 30 minutes)
        window: u64,
    }
    
    // Ring buffer for historical prices
    struct PriceHistory has key {
        prices: vector<PriceSnapshot>,
        head: u64,      // next write position
        capacity: u64,
    }
    
    struct PriceSnapshot has copy, drop, store {
        price: u64,
        cumulative: u128,
        timestamp: u64,
    }
    
    const TWAP_WINDOW: u64 = 1800;   // 30 minutes
    const HISTORY_SIZE: u64 = 60;    // 60 snapshots
    const MIN_PERIOD: u64 = 30;      // min 30 seconds between updates
    
    const E_TOO_SOON: u64 = 1;
    const E_INSUFFICIENT_HISTORY: u64 = 2;
    
    public entry fun initialize(owner: &signer, window: u64) {
        let now = timestamp::now_seconds();
        
        move_to(owner, TWAPState {
            price_cumulative_last: 0,
            block_timestamp_last: now,
            price_average: 0,
            window,
        });
        
        move_to(owner, PriceHistory {
            prices: std::vector::empty(),
            head: 0,
            capacity: HISTORY_SIZE,
        });
    }
    
    // ============================================
    // Update TWAP with current spot price
    // Called by keepers or on each trade
    // ============================================
    
    public entry fun update(
        oracle_addr: address,
        current_spot_price: u64,
    ) acquires TWAPState, PriceHistory {
        let state = borrow_global_mut<TWAPState>(oracle_addr);
        let history = borrow_global_mut<PriceHistory>(oracle_addr);
        
        let now = timestamp::now_seconds();
        let elapsed = now - state.block_timestamp_last;
        
        assert!(elapsed >= MIN_PERIOD, E_TOO_SOON);
        
        // Update cumulative price
        let new_cumulative = state.price_cumulative_last + 
            (current_spot_price as u128) * (elapsed as u128);
        
        // Store snapshot
        let snapshot = PriceSnapshot {
            price: current_spot_price,
            cumulative: new_cumulative,
            timestamp: now,
        };
        
        let len = std::vector::length(&history.prices);
        if (len < history.capacity) {
            std::vector::push_back(&mut history.prices, snapshot);
        } else {
            // Overwrite oldest (ring buffer)
            let idx = history.head % history.capacity;
            *std::vector::borrow_mut(&mut history.prices, idx) = snapshot;
            history.head = history.head + 1;
        };
        
        // Update state
        state.price_cumulative_last = new_cumulative;
        state.block_timestamp_last = now;
        
        // Calculate TWAP over window
        state.price_average = calculate_twap(history, state.window, now, new_cumulative);
    }
    
    fun calculate_twap(
        history: &PriceHistory,
        window: u64,
        now: u64,
        current_cumulative: u128,
    ): u64 {
        let len = std::vector::length(&history.prices);
        if (len == 0) return 0;
        
        // Find oldest snapshot within window
        let window_start = if (now > window) { now - window } else { 0 };
        
        let mut i = 0u64;
        let mut oldest_in_window_idx = 0u64;
        let mut found = false;
        
        while (i < len) {
            let snap = std::vector::borrow(&history.prices, i);
            if (snap.timestamp >= window_start) {
                oldest_in_window_idx = i;
                found = true;
                break
            };
            i = i + 1;
        };
        
        if (!found) return 0;
        
        let oldest = std::vector::borrow(&history.prices, oldest_in_window_idx);
        let time_elapsed = now - oldest.timestamp;
        
        if (time_elapsed == 0) return oldest.price;
        
        let cumulative_diff = current_cumulative - oldest.cumulative;
        (cumulative_diff / (time_elapsed as u128)) as u64
    }
    
    // ============================================
    // Get TWAP price (manipulation resistant)
    // ============================================
    
    #[view]
    public fun get_twap(oracle_addr: address): u64 acquires TWAPState {
        borrow_global<TWAPState>(oracle_addr).price_average
    }
    
    // ============================================
    // Validate spot vs TWAP (circuit breaker)
    // ============================================
    
    public fun validate_price(
        oracle_addr: address,
        spot_price: u64,
        max_deviation_bps: u64,
    ) acquires TWAPState {
        let twap = borrow_global<TWAPState>(oracle_addr).price_average;
        if (twap == 0) return;  // No history yet
        
        let deviation = if (spot_price > twap) {
            (spot_price - twap) * 10_000 / twap
        } else {
            (twap - spot_price) * 10_000 / twap
        };
        
        assert!(deviation <= max_deviation_bps, 100);
    }
}
```

---

## Oracle Aggregator

```move
module defi::oracle_aggregator {
    use std::signer;
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::timestamp;
    
    // ============================================
    // Aggregate multiple oracle sources
    // Use median to resist manipulation
    // ============================================
    
    struct AggregatorConfig has key {
        admin: address,
        feeds: SmartTable<vector<u8>, AggFeedConfig>,
    }
    
    struct AggFeedConfig has copy, drop, store {
        symbol: vector<u8>,
        sources: vector<OracleSource>,
        min_sources: u64,
        staleness_threshold: u64,
    }
    
    struct OracleSource has copy, drop, store {
        oracle_addr: address,
        weight: u64,       // 0-100, relative weight
        enabled: bool,
    }
    
    struct AggregatedPrice has copy, drop, store {
        price: u64,
        confidence: u64,
        timestamp: u64,
        sources_used: u64,
    }
    
    const E_INSUFFICIENT_SOURCES: u64 = 1;
    const E_NO_VALID_PRICE: u64 = 2;
    
    // ============================================
    // Get aggregated price (median of sources)
    // ============================================
    
    public fun get_price(
        agg_addr: address,
        symbol: vector<u8>,
    ): AggregatedPrice acquires AggregatorConfig {
        let config = borrow_global<AggregatorConfig>(agg_addr);
        let feed = smart_table::borrow(&config.feeds, symbol);
        
        let now = timestamp::now_seconds();
        let mut valid_prices = std::vector::empty<u64>();
        
        let mut i = 0u64;
        let n = std::vector::length(&feed.sources);
        while (i < n) {
            let source = std::vector::borrow(&feed.sources, i);
            if (!source.enabled) {
                i = i + 1;
                continue
            };
            
            // Try to get price (in production: use try_borrow pattern)
            // For demo: assume price getter exists
            // let price = other_oracle::get_price(source.oracle_addr, symbol);
            let price = 0u64;  // placeholder
            
            if (price > 0) {
                std::vector::push_back(&mut valid_prices, price);
            };
            
            i = i + 1;
        };
        
        let sources_used = std::vector::length(&valid_prices);
        assert!(sources_used >= feed.min_sources, E_INSUFFICIENT_SOURCES);
        
        // Calculate median
        let median_price = median(&mut valid_prices);
        
        // Calculate confidence (std dev approximation)
        let conf = calculate_spread(&valid_prices, median_price);
        
        AggregatedPrice {
            price: median_price,
            confidence: conf,
            timestamp: now,
            sources_used,
        }
    }
    
    // Sort and find median
    fun median(prices: &mut vector<u64>): u64 {
        // Simple insertion sort
        let n = std::vector::length(prices);
        let mut i = 1u64;
        while (i < n) {
            let mut j = i;
            while (j > 0) {
                let curr = *std::vector::borrow(prices, j);
                let prev = *std::vector::borrow(prices, j - 1);
                if (prev > curr) {
                    // swap
                    *std::vector::borrow_mut(prices, j) = prev;
                    *std::vector::borrow_mut(prices, j - 1) = curr;
                } else {
                    break
                };
                j = j - 1;
            };
            i = i + 1;
        };
        
        if (n % 2 == 1) {
            *std::vector::borrow(prices, n / 2)
        } else {
            let a = *std::vector::borrow(prices, n / 2 - 1);
            let b = *std::vector::borrow(prices, n / 2);
            (a + b) / 2
        }
    }
    
    // Calculate max deviation from median as confidence
    fun calculate_spread(prices: &vector<u64>, median: u64): u64 {
        let n = std::vector::length(prices);
        let mut max_dev = 0u64;
        let mut i = 0u64;
        while (i < n) {
            let p = *std::vector::borrow(prices, i);
            let dev = if (p > median) { p - median } else { median - p };
            if (dev > max_dev) { max_dev = dev; };
            i = i + 1;
        };
        max_dev
    }
}
```

---

## ตัวอย่าง: DeFi ใช้ Oracle

```move
module defi::oracle_lending {
    use std::signer;
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::timestamp;
    
    // ============================================
    // Lending protocol using oracle for valuation
    // ============================================
    
    struct LendingPool<phantom Collateral, phantom Borrow> has key {
        collateral_reserve: Coin<Collateral>,
        borrow_reserve: Coin<Borrow>,
        
        // Oracle addresses
        collateral_oracle: address,
        borrow_oracle: address,
        
        // Risk params
        ltv_bps: u64,          // Loan-to-Value ratio (e.g., 7500 = 75%)
        liq_threshold_bps: u64, // Liquidation threshold (e.g., 8000 = 80%)
        liq_bonus_bps: u64,    // Liquidation bonus (e.g., 500 = 5%)
        
        total_borrowed: u64,
        total_collateral: u64,
    }
    
    struct UserPosition<phantom Collateral, phantom Borrow> has key {
        collateral_amount: u64,
        borrow_amount: u64,
        borrow_index_snapshot: u64,
        opened_at: u64,
    }
    
    const E_INSUFFICIENT_COLLATERAL: u64 = 1;
    const E_HEALTHY_POSITION: u64 = 2;
    const E_STALE_ORACLE: u64 = 3;
    
    // ============================================
    // Calculate collateral value in USD (18 dec)
    // ============================================
    
    fun collateral_value_usd<C, B>(
        pool: &LendingPool<C, B>,
        collateral_amount: u64,
    ): u64 acquires crate::pyth_oracle::PriceFeedConfig {
        // Get price from oracle (18 decimal normalized)
        let price = crate::pyth_oracle::get_price(
            pool.collateral_oracle,
            b"COLLATERAL",
        );
        
        // value = amount * price / 1e18 (if amount has 8 decimals)
        // For simplicity: assume amount is already normalized
        (collateral_amount as u128 * price as u128 / 1_000_000_000_000_000_000u128) as u64
    }
    
    // ============================================
    // Deposit collateral and borrow
    // ============================================
    
    public entry fun deposit_and_borrow<C, B>(
        user: &signer,
        pool_addr: address,
        collateral: Coin<C>,
        borrow_amount: u64,
    ) acquires LendingPool<C, B> {
        let user_addr = signer::address_of(user);
        let pool = borrow_global_mut<LendingPool<C, B>>(pool_addr);
        
        let collateral_amount = coin::value(&collateral);
        
        // Check LTV: borrow_value <= collateral_value * ltv / 10000
        // (simplified: same asset prices)
        let max_borrow = collateral_amount * pool.ltv_bps / 10_000;
        assert!(borrow_amount <= max_borrow, E_INSUFFICIENT_COLLATERAL);
        
        // Accept collateral
        coin::merge(&mut pool.collateral_reserve, collateral);
        pool.total_collateral = pool.total_collateral + collateral_amount;
        
        // Create position
        move_to(user, UserPosition<C, B> {
            collateral_amount,
            borrow_amount,
            borrow_index_snapshot: 1_000_000_000_000_000_000,  // 1e18
            opened_at: timestamp::now_seconds(),
        });
        
        // Send borrowed funds to user
        let borrowed = coin::extract(&mut pool.borrow_reserve, borrow_amount);
        coin::deposit(user_addr, borrowed);
        pool.total_borrowed = pool.total_borrowed + borrow_amount;
    }
    
    // ============================================
    // Liquidate unhealthy position
    // ============================================
    
    public entry fun liquidate<C, B>(
        liquidator: &signer,
        pool_addr: address,
        borrower: address,
        repay_amount: u64,
        repay_coin: Coin<B>,
    ) acquires LendingPool<C, B>, UserPosition<C, B> {
        let pool = borrow_global_mut<LendingPool<C, B>>(pool_addr);
        let position = borrow_global_mut<UserPosition<C, B>>(borrower);
        
        // Check health factor (simplified: direct comparison)
        let health_factor_bps = position.collateral_amount 
            * pool.liq_threshold_bps 
            / position.borrow_amount;
        
        assert!(health_factor_bps < 10_000, E_HEALTHY_POSITION);
        
        // Repay debt
        let repay_value = coin::value(&repay_coin);
        assert!(repay_value >= repay_amount, 100);
        coin::merge(&mut pool.borrow_reserve, repay_coin);
        
        // Seize collateral with bonus
        let collateral_seized = repay_amount 
            * (10_000 + pool.liq_bonus_bps) / 10_000;
        let collateral_seized = if (collateral_seized > position.collateral_amount) {
            position.collateral_amount
        } else {
            collateral_seized
        };
        
        // Update position
        position.borrow_amount = position.borrow_amount - repay_amount;
        position.collateral_amount = position.collateral_amount - collateral_seized;
        pool.total_borrowed = pool.total_borrowed - repay_amount;
        pool.total_collateral = pool.total_collateral - collateral_seized;
        
        // Send seized collateral to liquidator
        let seized = coin::extract(&mut pool.collateral_reserve, collateral_seized);
        coin::deposit(signer::address_of(liquidator), seized);
    }
    
    // ============================================
    // Oracle Circuit Breaker
    // ============================================
    
    struct CircuitBreaker has key {
        oracle_addr: address,
        symbol: vector<u8>,
        last_valid_price: u64,
        last_valid_time: u64,
        tripped: bool,
        max_change_bps: u64,  // max price change per update
    }
    
    public entry fun check_circuit_breaker(
        cb_addr: address,
        new_price: u64,
    ) acquires CircuitBreaker {
        let cb = borrow_global_mut<CircuitBreaker>(cb_addr);
        
        if (cb.last_valid_price == 0) {
            cb.last_valid_price = new_price;
            cb.last_valid_time = timestamp::now_seconds();
            return
        };
        
        let change = if (new_price > cb.last_valid_price) {
            (new_price - cb.last_valid_price) * 10_000 / cb.last_valid_price
        } else {
            (cb.last_valid_price - new_price) * 10_000 / cb.last_valid_price
        };
        
        if (change > cb.max_change_bps) {
            cb.tripped = true;
            // Alert: price moved too much, pause oracle
        } else {
            cb.last_valid_price = new_price;
            cb.last_valid_time = timestamp::now_seconds();
            cb.tripped = false;
        };
    }
    
    #[view]
    public fun is_oracle_healthy(cb_addr: address): bool acquires CircuitBreaker {
        let cb = borrow_global<CircuitBreaker>(cb_addr);
        let now = timestamp::now_seconds();
        !cb.tripped && (now - cb.last_valid_time < 300)
    }
}
```

---

## สรุป Oracle Patterns

| Pattern | Use Case | Pros | Cons |
|---------|----------|------|------|
| Pyth Push | DeFi, DEX | Real-time, frequent | Needs keeper |
| Switchboard Pull | On-demand pricing | Flexible | Higher latency |
| On-chain TWAP | AMM-based protocols | Manipulation resistant | Slow response |
| Aggregator | Critical ops | Very secure | Complex, expensive |
| Circuit Breaker | All protocols | Safety net | May false-trip |

**Security Rules:**
1. เสมอ check staleness (max age)
2. เสมอ check confidence interval  
3. ใช้ TWAP สำหรับ large positions
4. Circuit breaker สำหรับ sudden price moves
5. ไม่ใช้ single oracle สำหรับ critical operations

---

**ก่อนหน้า**: [Part 32 - Upgrade Patterns ←](part-32-upgrade-patterns.md)
**ต่อไป**: [Part 34 - Governance Systems →](part-34-governance.md)
