# Part 24: Tables and SmartTable

## สารบัญ
- [Table คืออะไร](#table-คืออะไร)
- [aptos_std::table](#aptos_stdtable)
- [aptos_std::table_with_length](#aptos_stdtable_with_length)
- [aptos_std::smart_table](#aptos_stdsmart_table)
- [aptos_std::simple_map](#aptos_stdsimple_map)
- [ตัวอย่าง: Orderbook DEX](#ตัวอย่าง-orderbook-dex)

---

## Table คืออะไร

```
Table<K, V>
  ├── Stored at: module/resource address
  ├── Key type K: must have copy + drop
  ├── Value type V: can be any (including non-drop)
  └── Stored in global storage separately (not inline)

เมื่อไหร่ใช้ Table:
  ✓ Large datasets (100+ entries)
  ✓ Values accessed individually
  ✓ Don't need iteration
  
เมื่อไหร่ใช้ vector:
  ✓ Small datasets (<100 entries)
  ✓ Need to iterate
  ✓ Know the max size

เมื่อไหร่ใช้ SmartTable:
  ✓ Medium to large datasets
  ✓ Need occasional iteration
  ✓ Want automatic bucketing
```

---

## aptos_std::table

```move
module learning::table_basics {
    use std::signer;
    use aptos_std::table::{Self, Table};
    
    // ============================================
    // Basic Table operations
    // ============================================
    
    struct Registry has key {
        entries: Table<address, UserInfo>,
    }
    
    struct UserInfo has copy, drop, store {
        name: vector<u8>,
        score: u64,
        registered_at: u64,
    }
    
    // Initialize
    public entry fun init_registry(admin: &signer) {
        move_to(admin, Registry {
            entries: table::new<address, UserInfo>(),
        });
    }
    
    // Add entry (fails if key exists)
    public entry fun register_user(
        registry_owner: address,
        user_addr: address,
        name: vector<u8>,
    ) acquires Registry {
        let registry = borrow_global_mut<Registry>(registry_owner);
        
        // add: fails if key already exists
        table::add(&mut registry.entries, user_addr, UserInfo {
            name,
            score: 0,
            registered_at: 0,
        });
    }
    
    // Upsert (add or replace)
    public entry fun upsert_user(
        registry_owner: address,
        user_addr: address,
        name: vector<u8>,
        score: u64,
    ) acquires Registry {
        let registry = borrow_global_mut<Registry>(registry_owner);
        
        if (table::contains(&registry.entries, user_addr)) {
            let user = table::borrow_mut(&mut registry.entries, user_addr);
            user.name = name;
            user.score = score;
        } else {
            table::add(&mut registry.entries, user_addr, UserInfo {
                name,
                score,
                registered_at: 0,
            });
        };
    }
    
    // Read entry (fails if key missing)
    #[view]
    public fun get_user(registry_owner: address, user_addr: address): UserInfo acquires Registry {
        let registry = borrow_global<Registry>(registry_owner);
        *table::borrow(&registry.entries, user_addr)
    }
    
    // Safe read
    #[view]
    public fun get_user_score(registry_owner: address, user_addr: address): u64 acquires Registry {
        let registry = borrow_global<Registry>(registry_owner);
        if (!table::contains(&registry.entries, user_addr)) return 0;
        table::borrow(&registry.entries, user_addr).score
    }
    
    // Remove entry (returns value)
    public entry fun remove_user(
        registry_owner: address,
        user_addr: address,
    ) acquires Registry {
        let registry = borrow_global_mut<Registry>(registry_owner);
        let UserInfo { name: _, score: _, registered_at: _ } = 
            table::remove(&mut registry.entries, user_addr);
    }
    
    // Check existence
    public fun user_exists(registry_owner: address, user_addr: address): bool acquires Registry {
        let registry = borrow_global<Registry>(registry_owner);
        table::contains(&registry.entries, user_addr)
    }
    
    // ============================================
    // Table<u64, V> for indexed data
    // ============================================
    
    struct EventLog has key {
        events: Table<u64, EventEntry>,
        next_id: u64,
    }
    
    struct EventEntry has store {
        data: vector<u8>,
        timestamp: u64,
    }
    
    public entry fun init_log(admin: &signer) {
        move_to(admin, EventLog {
            events: table::new(),
            next_id: 0,
        });
    }
    
    public entry fun log_event(
        log_owner: address,
        data: vector<u8>,
    ) acquires EventLog {
        let log = borrow_global_mut<EventLog>(log_owner);
        let id = log.next_id;
        table::add(&mut log.events, id, EventEntry { data, timestamp: 0 });
        log.next_id = id + 1;
    }
    
    // ============================================
    // Destroying Table
    // ============================================
    
    // Table<K, V> has drop only if V has drop
    // If V doesn't have drop, must manually clean up
    
    struct ResourceTable has key {
        map: Table<u64, ResourceEntry>,
        keys: vector<u64>,  // Track keys for cleanup
    }
    
    struct ResourceEntry has store {  // no drop!
        value: u64,
    }
    
    public entry fun cleanup_table(admin: &signer, admin_addr: address) acquires ResourceTable {
        let ResourceTable { map, keys } = move_from<ResourceTable>(admin_addr);
        
        // Must remove all entries before destroy
        let i = 0;
        while (i < std::vector::length(&keys)) {
            let key = *std::vector::borrow(&keys, i);
            let ResourceEntry { value: _ } = table::remove(&mut map, key);
            i = i + 1;
        };
        
        table::destroy_empty(map);
        // keys has drop so it's automatically dropped
    }
}
```

---

## aptos_std::table_with_length

```move
module learning::table_with_length {
    use aptos_std::table_with_length::{Self, TableWithLength};
    
    // ============================================
    // Table with length tracking
    // ============================================
    
    struct Leaderboard has key {
        scores: TableWithLength<address, u64>,
    }
    
    public entry fun init_leaderboard(admin: &signer) {
        move_to(admin, Leaderboard {
            scores: table_with_length::new<address, u64>(),
        });
    }
    
    public entry fun set_score(
        board_owner: address,
        player: address,
        score: u64,
    ) acquires Leaderboard {
        let board = borrow_global_mut<Leaderboard>(board_owner);
        
        if (table_with_length::contains(&board.scores, player)) {
            *table_with_length::borrow_mut(&mut board.scores, player) = score;
        } else {
            table_with_length::add(&mut board.scores, player, score);
        };
    }
    
    #[view]
    public fun total_players(board_owner: address): u64 acquires Leaderboard {
        let board = borrow_global<Leaderboard>(board_owner);
        table_with_length::length(&board.scores)
    }
    
    #[view]
    public fun is_empty(board_owner: address): bool acquires Leaderboard {
        let board = borrow_global<Leaderboard>(board_owner);
        table_with_length::empty(&board.scores)
    }
}
```

---

## aptos_std::smart_table

```move
module learning::smart_table_usage {
    use std::signer;
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // SmartTable = Table with bucketing
    // ============================================
    
    // SmartTable splits data into buckets for better gas
    // when the table grows large
    // Supports iteration (unlike Table)
    
    struct TokenBalances has key {
        balances: SmartTable<address, u64>,
    }
    
    struct MultiToken has key {
        holders: SmartTable<address, u64>,
        total_supply: u128,
        name: vector<u8>,
    }
    
    // ============================================
    // Basic SmartTable operations
    // ============================================
    
    public entry fun init_balances(admin: &signer) {
        move_to(admin, TokenBalances {
            balances: smart_table::new<address, u64>(),
        });
    }
    
    public fun set_balance(
        balances: &mut SmartTable<address, u64>,
        addr: address,
        amount: u64,
    ) {
        if (smart_table::contains(balances, addr)) {
            *smart_table::borrow_mut(balances, addr) = amount;
        } else {
            smart_table::add(balances, addr, amount);
        };
    }
    
    public fun get_balance(balances: &SmartTable<address, u64>, addr: address): u64 {
        if (!smart_table::contains(balances, addr)) return 0;
        *smart_table::borrow(balances, addr)
    }
    
    public fun transfer_balance(
        balances: &mut SmartTable<address, u64>,
        from: address,
        to: address,
        amount: u64,
    ) {
        let from_bal = get_balance(balances, from);
        assert!(from_bal >= amount, 1);
        
        set_balance(balances, from, from_bal - amount);
        let to_bal = get_balance(balances, to);
        set_balance(balances, to, to_bal + amount);
    }
    
    // ============================================
    // Iteration (key feature of SmartTable)
    // ============================================
    
    // for_each_ref iterates all entries
    public fun count_nonzero_balances(
        balances: &SmartTable<address, u64>
    ): u64 {
        let count = 0u64;
        smart_table::for_each_ref(balances, |_addr, bal| {
            if (*bal > 0) {
                count = count + 1;
            };
        });
        count
    }
    
    public fun sum_all_balances(balances: &SmartTable<address, u64>): u128 {
        let total = 0u128;
        smart_table::for_each_ref(balances, |_addr, bal| {
            total = total + (*bal as u128);
        });
        total
    }
    
    // Find top holder
    public fun find_top_holder(
        balances: &SmartTable<address, u64>
    ): (address, u64) {
        let top_addr = @0x0;
        let top_bal = 0u64;
        
        smart_table::for_each_ref(balances, |addr, bal| {
            if (*bal > top_bal) {
                top_addr = *addr;
                top_bal = *bal;
            };
        });
        
        (top_addr, top_bal)
    }
    
    // ============================================
    // SmartTable config
    // ============================================
    
    public entry fun init_with_config(admin: &signer) {
        // new_with_config(num_initial_slots, target_bucket_size, load_factor_10x)
        let table = smart_table::new_with_config<address, u64>(
            4,    // initial slots
            10,   // target bucket size
            75,   // load factor 75%
        );
        
        move_to(admin, TokenBalances { balances: table });
    }
    
    // ============================================
    // Length
    // ============================================
    
    public fun holder_count(balances: &SmartTable<address, u64>): u64 {
        smart_table::length(balances)
    }
    
    // ============================================
    // Destroy SmartTable
    // ============================================
    
    // SmartTable<K, V> has drop if both K and V have drop
    // For non-drop V, must clear manually
    
    public entry fun destroy_balances(admin: &signer) acquires TokenBalances {
        let addr = signer::address_of(admin);
        let TokenBalances { balances } = move_from<TokenBalances>(addr);
        smart_table::destroy(balances);  // works if u64 has drop
    }
}
```

---

## aptos_std::simple_map

```move
module learning::simple_map_usage {
    use aptos_std::simple_map::{Self, SimpleMap};
    
    // ============================================
    // SimpleMap = sorted vector of key-value pairs
    // ============================================
    
    // Best for: small collections (<20 entries) that need iteration
    // Stored inline (no separate storage like Table)
    // Supports copy if both K and V support copy
    
    struct Config has key {
        settings: SimpleMap<vector<u8>, vector<u8>>,
    }
    
    public entry fun init_config(admin: &signer) {
        let map = simple_map::create<vector<u8>, vector<u8>>();
        move_to(admin, Config { settings: map });
    }
    
    public entry fun set_setting(
        config_addr: address,
        key: vector<u8>,
        value: vector<u8>,
    ) acquires Config {
        let config = borrow_global_mut<Config>(config_addr);
        
        if (simple_map::contains_key(&config.settings, &key)) {
            let val = simple_map::borrow_mut(&mut config.settings, &key);
            *val = value;
        } else {
            simple_map::add(&mut config.settings, key, value);
        };
    }
    
    #[view]
    public fun get_setting(config_addr: address, key: vector<u8>): vector<u8> acquires Config {
        let config = borrow_global<Config>(config_addr);
        if (!simple_map::contains_key(&config.settings, &key)) return b"";
        *simple_map::borrow(&config.settings, &key)
    }
    
    // Get all keys
    public fun get_all_keys(config: &Config): vector<vector<u8>> {
        simple_map::keys(&config.settings)
    }
    
    // Get all values
    public fun get_all_values(config: &Config): vector<vector<u8>> {
        simple_map::values(&config.settings)
    }
    
    // Convert to two vectors
    public fun to_vectors(config: &Config): (vector<vector<u8>>, vector<vector<u8>>) {
        simple_map::to_vec_pair(config.settings)
    }
    
    // Count
    public fun setting_count(config: &Config): u64 {
        simple_map::length(&config.settings)
    }
    
    public entry fun remove_setting(config_addr: address, key: vector<u8>) acquires Config {
        let config = borrow_global_mut<Config>(config_addr);
        simple_map::remove(&mut config.settings, &key);
    }
}
```

---

## ตัวอย่าง: Orderbook DEX

```move
module learning::orderbook {
    use std::signer;
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::event;
    use aptos_framework::timestamp;
    
    // ============================================
    // Types
    // ============================================
    
    struct Order has copy, drop, store {
        id: u64,
        maker: address,
        price: u64,      // price per unit (in quote token, scaled 1e6)
        quantity: u64,   // amount of base token
        filled: u64,     // how much has been filled
        is_buy: bool,    // true = buy order, false = sell order
        created_at: u64,
    }
    
    struct Orderbook has key {
        buy_orders: SmartTable<u64, Order>,   // order_id -> Order
        sell_orders: SmartTable<u64, Order>,
        next_order_id: u64,
        base_token: address,   // token being traded
        quote_token: address,  // pricing token (e.g. USDC)
        min_order_size: u64,
    }
    
    struct UserOrders has key {
        order_ids: SmartTable<u64, bool>,  // set of order ids
    }
    
    // ============================================
    // Events
    // ============================================
    
    #[event]
    struct OrderPlaced has drop, store {
        order_id: u64,
        maker: address,
        price: u64,
        quantity: u64,
        is_buy: bool,
    }
    
    #[event]
    struct OrderCancelled has drop, store {
        order_id: u64,
        maker: address,
    }
    
    #[event]
    struct OrderFilled has drop, store {
        buy_order_id: u64,
        sell_order_id: u64,
        price: u64,
        quantity: u64,
        buyer: address,
        seller: address,
    }
    
    // ============================================
    // Errors
    // ============================================
    
    const E_NOT_MAKER: u64 = 1;
    const E_ORDER_NOT_FOUND: u64 = 2;
    const E_ORDER_TOO_SMALL: u64 = 3;
    const E_INVALID_PRICE: u64 = 4;
    const E_BOOK_NOT_INITIALIZED: u64 = 5;
    
    // ============================================
    // Initialize
    // ============================================
    
    public entry fun initialize_book(
        admin: &signer,
        base_token: address,
        quote_token: address,
        min_order_size: u64,
    ) {
        move_to(admin, Orderbook {
            buy_orders: smart_table::new(),
            sell_orders: smart_table::new(),
            next_order_id: 1,
            base_token,
            quote_token,
            min_order_size,
        });
    }
    
    // ============================================
    // Place Order
    // ============================================
    
    public entry fun place_buy_order(
        maker: &signer,
        book_addr: address,
        price: u64,
        quantity: u64,
    ) acquires Orderbook, UserOrders {
        assert!(price > 0, E_INVALID_PRICE);
        let book = borrow_global_mut<Orderbook>(book_addr);
        assert!(quantity >= book.min_order_size, E_ORDER_TOO_SMALL);
        
        let maker_addr = signer::address_of(maker);
        let order_id = book.next_order_id;
        book.next_order_id = order_id + 1;
        
        let order = Order {
            id: order_id,
            maker: maker_addr,
            price,
            quantity,
            filled: 0,
            is_buy: true,
            created_at: timestamp::now_seconds(),
        };
        
        smart_table::add(&mut book.buy_orders, order_id, order);
        
        // Track user's orders
        if (!exists<UserOrders>(maker_addr)) {
            move_to(maker, UserOrders {
                order_ids: smart_table::new(),
            });
        };
        let user_orders = borrow_global_mut<UserOrders>(maker_addr);
        smart_table::add(&mut user_orders.order_ids, order_id, true);
        
        event::emit(OrderPlaced {
            order_id,
            maker: maker_addr,
            price,
            quantity,
            is_buy: true,
        });
    }
    
    public entry fun place_sell_order(
        maker: &signer,
        book_addr: address,
        price: u64,
        quantity: u64,
    ) acquires Orderbook, UserOrders {
        assert!(price > 0, E_INVALID_PRICE);
        let book = borrow_global_mut<Orderbook>(book_addr);
        assert!(quantity >= book.min_order_size, E_ORDER_TOO_SMALL);
        
        let maker_addr = signer::address_of(maker);
        let order_id = book.next_order_id;
        book.next_order_id = order_id + 1;
        
        let order = Order {
            id: order_id,
            maker: maker_addr,
            price,
            quantity,
            filled: 0,
            is_buy: false,
            created_at: timestamp::now_seconds(),
        };
        
        smart_table::add(&mut book.sell_orders, order_id, order);
        
        if (!exists<UserOrders>(maker_addr)) {
            move_to(maker, UserOrders {
                order_ids: smart_table::new(),
            });
        };
        let user_orders = borrow_global_mut<UserOrders>(maker_addr);
        smart_table::add(&mut user_orders.order_ids, order_id, false);
        
        event::emit(OrderPlaced {
            order_id,
            maker: maker_addr,
            price,
            quantity,
            is_buy: false,
        });
    }
    
    // ============================================
    // Cancel Order
    // ============================================
    
    public entry fun cancel_order(
        maker: &signer,
        book_addr: address,
        order_id: u64,
        is_buy: bool,
    ) acquires Orderbook, UserOrders {
        let maker_addr = signer::address_of(maker);
        let book = borrow_global_mut<Orderbook>(book_addr);
        
        let order = if (is_buy) {
            assert!(smart_table::contains(&book.buy_orders, order_id), E_ORDER_NOT_FOUND);
            *smart_table::borrow(&book.buy_orders, order_id)
        } else {
            assert!(smart_table::contains(&book.sell_orders, order_id), E_ORDER_NOT_FOUND);
            *smart_table::borrow(&book.sell_orders, order_id)
        };
        
        assert!(order.maker == maker_addr, E_NOT_MAKER);
        
        // Remove from book
        if (is_buy) {
            smart_table::remove(&mut book.buy_orders, order_id);
        } else {
            smart_table::remove(&mut book.sell_orders, order_id);
        };
        
        // Remove from user tracking
        let user_orders = borrow_global_mut<UserOrders>(maker_addr);
        smart_table::remove(&mut user_orders.order_ids, order_id);
        
        event::emit(OrderCancelled { order_id, maker: maker_addr });
    }
    
    // ============================================
    // Match Orders (simple price-time matching)
    // ============================================
    
    public entry fun match_order(
        _matcher: &signer,
        book_addr: address,
        buy_order_id: u64,
        sell_order_id: u64,
    ) acquires Orderbook {
        let book = borrow_global_mut<Orderbook>(book_addr);
        
        assert!(smart_table::contains(&book.buy_orders, buy_order_id), E_ORDER_NOT_FOUND);
        assert!(smart_table::contains(&book.sell_orders, sell_order_id), E_ORDER_NOT_FOUND);
        
        let buy = *smart_table::borrow(&book.buy_orders, buy_order_id);
        let sell = *smart_table::borrow(&book.sell_orders, sell_order_id);
        
        // Check price matches
        assert!(buy.price >= sell.price, E_INVALID_PRICE);
        
        // Determine fill quantity
        let buy_remaining = buy.quantity - buy.filled;
        let sell_remaining = sell.quantity - sell.filled;
        let fill_qty = if (buy_remaining < sell_remaining) buy_remaining else sell_remaining;
        
        // Use sell price (maker price)
        let fill_price = sell.price;
        
        // Update orders
        let buy_mut = smart_table::borrow_mut(&mut book.buy_orders, buy_order_id);
        buy_mut.filled = buy_mut.filled + fill_qty;
        
        let sell_mut = smart_table::borrow_mut(&mut book.sell_orders, sell_order_id);
        sell_mut.filled = sell_mut.filled + fill_qty;
        
        event::emit(OrderFilled {
            buy_order_id,
            sell_order_id,
            price: fill_price,
            quantity: fill_qty,
            buyer: buy.maker,
            seller: sell.maker,
        });
        
        // Remove fully filled orders
        let buy_order = *smart_table::borrow(&book.buy_orders, buy_order_id);
        if (buy_order.filled >= buy_order.quantity) {
            smart_table::remove(&mut book.buy_orders, buy_order_id);
        };
        
        let sell_order = *smart_table::borrow(&book.sell_orders, sell_order_id);
        if (sell_order.filled >= sell_order.quantity) {
            smart_table::remove(&mut book.sell_orders, sell_order_id);
        };
    }
    
    // ============================================
    // View functions
    // ============================================
    
    #[view]
    public fun get_order(book_addr: address, order_id: u64, is_buy: bool): Order 
    acquires Orderbook {
        let book = borrow_global<Orderbook>(book_addr);
        if (is_buy) {
            *smart_table::borrow(&book.buy_orders, order_id)
        } else {
            *smart_table::borrow(&book.sell_orders, order_id)
        }
    }
    
    #[view]
    public fun buy_order_count(book_addr: address): u64 acquires Orderbook {
        let book = borrow_global<Orderbook>(book_addr);
        smart_table::length(&book.buy_orders)
    }
    
    #[view]
    public fun sell_order_count(book_addr: address): u64 acquires Orderbook {
        let book = borrow_global<Orderbook>(book_addr);
        smart_table::length(&book.sell_orders)
    }
    
    // Find best buy price (highest)
    public fun best_bid(book_addr: address): u64 acquires Orderbook {
        let book = borrow_global<Orderbook>(book_addr);
        let best = 0u64;
        smart_table::for_each_ref(&book.buy_orders, |_id, order| {
            if (order.price > best) best = order.price;
        });
        best
    }
    
    // Find best sell price (lowest)
    public fun best_ask(book_addr: address): u64 acquires Orderbook {
        let book = borrow_global<Orderbook>(book_addr);
        let best = 0xFFFFFFFFFFFFFFFFu64;
        smart_table::for_each_ref(&book.sell_orders, |_id, order| {
            if (order.price < best) best = order.price;
        });
        if (best == 0xFFFFFFFFFFFFFFFFu64) { 0 } else { best }
    }
}
```

---

## เปรียบเทียบ Collections

| | `Table` | `TableWithLength` | `SmartTable` | `SimpleMap` | `vector` |
|--|---------|-------------------|--------------|-------------|----------|
| Max size | Unlimited | Unlimited | Unlimited | ~20 entries | ~100 entries |
| Iteration | ❌ | ❌ | ✓ | ✓ | ✓ |
| Length | ❌ | ✓ | ✓ | ✓ | ✓ |
| Gas per op | O(1) | O(1) | O(1) | O(log n) | O(1) or O(n) |
| Inline storage | ❌ | ❌ | ❌ | ✓ | ✓ |
| Use case | Large KV | Large KV+count | Large iterable | Small map | Small list |

---

**ก่อนหน้า**: [Part 23 - Aptos Objects ←](part-23-aptos-objects.md)
**ต่อไป**: [Part 25 - Aptos Token Standard →](part-25-aptos-token-standard.md)
