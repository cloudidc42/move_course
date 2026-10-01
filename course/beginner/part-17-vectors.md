# Part 17: Vectors Deep Dive

## สารบัญ
- [Vector Internals](#vector-internals)
- [Advanced Operations](#advanced-operations)
- [Vector Algorithms](#vector-algorithms)
- [Memory Management](#memory-management)
- [ตัวอย่างโปรแกรม: Portfolio Tracker](#ตัวอย่างโปรแกรม-portfolio-tracker)

---

## Vector Internals

```move
module learning::vector_internals {
    use std::vector;
    
    // ============================================
    // vector<T> characteristics
    // ============================================
    
    // - Dynamic array (grows as needed)
    // - Zero-indexed
    // - O(1) access by index
    // - O(1) push/pop from end
    // - O(n) insert/remove from middle
    // - Has copy+drop if T has copy+drop
    // - Contiguous memory layout
    
    // ============================================
    // Creation patterns
    // ============================================
    
    public fun create_patterns() {
        // Empty
        let v1: vector<u64> = vector::empty();
        
        // Literal
        let v2 = vector[1u64, 2u64, 3u64];
        
        // Singleton
        let v3 = vector::singleton(42u64);
        
        // Pre-allocated (no built-in, use loop)
        let v4 = vector::empty<u64>();
        let i = 0u64;
        while (i < 10) {
            vector::push_back(&mut v4, 0u64);
            i = i + 1;
        };
        
        let _ = (v1, v2, v3, v4);
    }
    
    // ============================================
    // Borrowing patterns
    // ============================================
    
    public fun borrow_patterns(v: &vector<u64>) {
        let len = vector::length(v);
        if (len == 0) return;
        
        // Immutable borrow - returns &T
        let first: &u64 = vector::borrow(v, 0);
        let last: &u64 = vector::borrow(v, len - 1);
        
        // Dereference to get value (requires T: copy)
        let first_val: u64 = *first;
        let last_val: u64 = *last;
        
        let _ = (first_val, last_val);
    }
    
    public fun mutable_borrow_patterns(v: &mut vector<u64>) {
        if (vector::is_empty(v)) return;
        
        // Mutable borrow - returns &mut T
        let first: &mut u64 = vector::borrow_mut(v, 0);
        *first = 100;  // modify in place
        
        // Note: can't borrow two elements mutably at same time
        // let a = vector::borrow_mut(v, 0);
        // let b = vector::borrow_mut(v, 1);  // ERROR: two mutable borrows
    }
    
    // ============================================
    // Slice-like patterns
    // ============================================
    
    // Get subrange of vector (copy elements)
    public fun slice<T: copy>(v: &vector<T>, start: u64, end: u64): vector<T> {
        assert!(start <= end && end <= vector::length(v), 0);
        let result = vector::empty<T>();
        let i = start;
        while (i < end) {
            vector::push_back(&mut result, *vector::borrow(v, i));
            i = i + 1;
        };
        result
    }
    
    #[test]
    public fun test_slice() {
        let v = vector[1u64, 2u64, 3u64, 4u64, 5u64];
        let s = slice(&v, 1, 4);  // [2, 3, 4]
        assert!(vector::length(&s) == 3, 0);
        assert!(*vector::borrow(&s, 0) == 2, 1);
        assert!(*vector::borrow(&s, 2) == 4, 2);
    }
}
```

---

## Advanced Operations

```move
module learning::vector_advanced {
    use std::vector;
    
    // ============================================
    // Iteration patterns
    // ============================================
    
    // Map: transform each element
    public fun map_u64(v: &vector<u64>, multiplier: u64): vector<u64> {
        let result = vector::empty<u64>();
        let i = 0u64;
        while (i < vector::length(v)) {
            vector::push_back(&mut result, *vector::borrow(v, i) * multiplier);
            i = i + 1;
        };
        result
    }
    
    // Filter: keep elements matching predicate
    public fun filter_greater_than(v: &vector<u64>, threshold: u64): vector<u64> {
        let result = vector::empty<u64>();
        let i = 0u64;
        while (i < vector::length(v)) {
            let val = *vector::borrow(v, i);
            if (val > threshold) {
                vector::push_back(&mut result, val);
            };
            i = i + 1;
        };
        result
    }
    
    // Reduce/fold: aggregate
    public fun fold_sum(v: &vector<u64>): u64 {
        let sum = 0u64;
        let i = 0u64;
        while (i < vector::length(v)) {
            sum = sum + *vector::borrow(v, i);
            i = i + 1;
        };
        sum
    }
    
    public fun fold_max(v: &vector<u64>): u64 {
        assert!(!vector::is_empty(v), 0);
        let max = *vector::borrow(v, 0);
        let i = 1u64;
        while (i < vector::length(v)) {
            let val = *vector::borrow(v, i);
            if (val > max) max = val;
            i = i + 1;
        };
        max
    }
    
    // Zip two vectors together
    public fun zip_add(a: &vector<u64>, b: &vector<u64>): vector<u64> {
        let len = vector::length(a);
        assert!(len == vector::length(b), 0);
        
        let result = vector::empty<u64>();
        let i = 0u64;
        while (i < len) {
            vector::push_back(
                &mut result,
                *vector::borrow(a, i) + *vector::borrow(b, i)
            );
            i = i + 1;
        };
        result
    }
    
    // Flatten nested vector
    public fun flatten(nested: vector<vector<u64>>): vector<u64> {
        let result = vector::empty<u64>();
        let i = 0u64;
        while (i < vector::length(&nested)) {
            let inner = vector::borrow(&nested, i);
            let j = 0u64;
            while (j < vector::length(inner)) {
                vector::push_back(&mut result, *vector::borrow(inner, j));
                j = j + 1;
            };
            i = i + 1;
        };
        // Clean up nested (has copy elements)
        let k = 0u64;
        while (k < vector::length(&nested)) {
            let _ = vector::borrow(&nested, k);
            k = k + 1;
        };
        // Note: need to properly consume nested in real code
        result
    }
    
    // ============================================
    // Set operations
    // ============================================
    
    public fun unique(v: &vector<u64>): vector<u64> {
        let result = vector::empty<u64>();
        let i = 0u64;
        while (i < vector::length(v)) {
            let val = *vector::borrow(v, i);
            if (!vector::contains(&result, &val)) {
                vector::push_back(&mut result, val);
            };
            i = i + 1;
        };
        result
    }
    
    public fun intersection(a: &vector<u64>, b: &vector<u64>): vector<u64> {
        let result = vector::empty<u64>();
        let i = 0u64;
        while (i < vector::length(a)) {
            let val = *vector::borrow(a, i);
            if (vector::contains(b, &val) && !vector::contains(&result, &val)) {
                vector::push_back(&mut result, val);
            };
            i = i + 1;
        };
        result
    }
    
    public fun difference(a: &vector<u64>, b: &vector<u64>): vector<u64> {
        let result = vector::empty<u64>();
        let i = 0u64;
        while (i < vector::length(a)) {
            let val = *vector::borrow(a, i);
            if (!vector::contains(b, &val)) {
                vector::push_back(&mut result, val);
            };
            i = i + 1;
        };
        result
    }
    
    // ============================================
    // Matrix operations (vector of vectors)
    // ============================================
    
    struct Matrix has drop {
        data: vector<vector<u64>>,
        rows: u64,
        cols: u64,
    }
    
    public fun create_matrix(rows: u64, cols: u64, init_val: u64): Matrix {
        let data = vector::empty<vector<u64>>();
        let i = 0u64;
        while (i < rows) {
            let row = vector::empty<u64>();
            let j = 0u64;
            while (j < cols) {
                vector::push_back(&mut row, init_val);
                j = j + 1;
            };
            vector::push_back(&mut data, row);
            i = i + 1;
        };
        Matrix { data, rows, cols }
    }
    
    public fun matrix_get(m: &Matrix, row: u64, col: u64): u64 {
        *vector::borrow(vector::borrow(&m.data, row), col)
    }
    
    public fun matrix_set(m: &mut Matrix, row: u64, col: u64, val: u64) {
        let row_ref = vector::borrow_mut(&mut m.data, row);
        *vector::borrow_mut(row_ref, col) = val;
    }
    
    public fun matrix_multiply(a: &Matrix, b: &Matrix): Matrix {
        assert!(a.cols == b.rows, 0);
        
        let result = create_matrix(a.rows, b.cols, 0);
        let i = 0u64;
        while (i < a.rows) {
            let j = 0u64;
            while (j < b.cols) {
                let sum = 0u64;
                let k = 0u64;
                while (k < a.cols) {
                    sum = sum + matrix_get(a, i, k) * matrix_get(b, k, j);
                    k = k + 1;
                };
                // Can't set on result as immutable/mutable conflict
                // In practice, build result differently
                let _ = sum;
                j = j + 1;
            };
            i = i + 1;
        };
        result
    }
}
```

---

## Vector Algorithms

```move
module learning::vector_algorithms {
    use std::vector;
    
    // ============================================
    // Sorting
    // ============================================
    
    public fun bubble_sort(v: &mut vector<u64>) {
        let n = vector::length(v);
        let i = 0u64;
        while (i < n) {
            let j = 0u64;
            let swapped = false;
            while (j < n - i - 1) {
                let a = *vector::borrow(v, j);
                let b = *vector::borrow(v, j + 1);
                if (a > b) {
                    vector::swap(v, j, j + 1);
                    swapped = true;
                };
                j = j + 1;
            };
            if (!swapped) break;
            i = i + 1;
        };
    }
    
    public fun insertion_sort(v: &mut vector<u64>) {
        let n = vector::length(v);
        let i = 1u64;
        while (i < n) {
            let j = i;
            while (j > 0) {
                let prev = *vector::borrow(v, j - 1);
                let curr = *vector::borrow(v, j);
                if (curr < prev) {
                    vector::swap(v, j - 1, j);
                    j = j - 1;
                } else {
                    break
                };
            };
            i = i + 1;
        };
    }
    
    // Selection sort
    public fun selection_sort(v: &mut vector<u64>) {
        let n = vector::length(v);
        let i = 0u64;
        while (i < n) {
            let min_idx = i;
            let j = i + 1;
            while (j < n) {
                if (*vector::borrow(v, j) < *vector::borrow(v, min_idx)) {
                    min_idx = j;
                };
                j = j + 1;
            };
            if (min_idx != i) {
                vector::swap(v, i, min_idx);
            };
            i = i + 1;
        };
    }
    
    // ============================================
    // Searching
    // ============================================
    
    // Linear search
    public fun linear_search(v: &vector<u64>, target: u64): (bool, u64) {
        let i = 0u64;
        while (i < vector::length(v)) {
            if (*vector::borrow(v, i) == target) {
                return (true, i)
            };
            i = i + 1;
        };
        (false, 0)
    }
    
    // Binary search (requires sorted vector)
    public fun binary_search(v: &vector<u64>, target: u64): (bool, u64) {
        let len = vector::length(v);
        if (len == 0) return (false, 0);
        
        let left = 0u64;
        let right = len - 1;
        
        while (left <= right) {
            let mid = left + (right - left) / 2;
            let val = *vector::borrow(v, mid);
            
            if (val == target) {
                return (true, mid)
            } else if (val < target) {
                left = mid + 1;
            } else {
                if (mid == 0) break;
                right = mid - 1;
            };
        };
        
        (false, 0)
    }
    
    // ============================================
    // Two-pointer techniques
    // ============================================
    
    // Two-sum: find pair that sums to target
    public fun two_sum(v: &vector<u64>, target: u64): (bool, u64, u64) {
        let n = vector::length(v);
        let left = 0u64;
        let right = n - 1;
        
        while (left < right) {
            let sum = *vector::borrow(v, left) + *vector::borrow(v, right);
            if (sum == target) {
                return (true, left, right)
            } else if (sum < target) {
                left = left + 1;
            } else {
                right = right - 1;
            };
        };
        
        (false, 0, 0)
    }
    
    // Check if palindrome (u8 vector)
    public fun is_palindrome(v: &vector<u8>): bool {
        let n = vector::length(v);
        if (n <= 1) return true;
        
        let left = 0u64;
        let right = n - 1;
        
        while (left < right) {
            if (*vector::borrow(v, left) != *vector::borrow(v, right)) {
                return false
            };
            left = left + 1;
            right = right - 1;
        };
        true
    }
    
    // ============================================
    // Sliding window
    // ============================================
    
    // Maximum sum subarray of size k
    public fun max_sum_window(v: &vector<u64>, k: u64): u64 {
        let n = vector::length(v);
        assert!(n >= k && k > 0, 0);
        
        // Initial window
        let window_sum = 0u64;
        let i = 0u64;
        while (i < k) {
            window_sum = window_sum + *vector::borrow(v, i);
            i = i + 1;
        };
        
        let max_sum = window_sum;
        
        // Slide window
        while (i < n) {
            window_sum = window_sum + *vector::borrow(v, i);
            window_sum = window_sum - *vector::borrow(v, i - k);
            if (window_sum > max_sum) {
                max_sum = window_sum;
            };
            i = i + 1;
        };
        
        max_sum
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test]
    public fun test_sorting() {
        let v = vector[5u64, 2u64, 8u64, 1u64, 9u64, 3u64];
        bubble_sort(&mut v);
        
        let i = 0u64;
        while (i < vector::length(&v) - 1) {
            assert!(*vector::borrow(&v, i) <= *vector::borrow(&v, i + 1), i);
            i = i + 1;
        };
    }
    
    #[test]
    public fun test_binary_search() {
        let v = vector[1u64, 3u64, 5u64, 7u64, 9u64, 11u64];
        
        let (found, idx) = binary_search(&v, 7u64);
        assert!(found, 0);
        assert!(idx == 3, 1);
        
        let (not_found, _) = binary_search(&v, 6u64);
        assert!(!not_found, 2);
    }
    
    #[test]
    public fun test_max_window() {
        let v = vector[1u64, 3u64, 5u64, 7u64, 9u64, 2u64, 4u64];
        let max = max_sum_window(&v, 3);
        assert!(max == 21, 0);  // 5+7+9 = 21
    }
}
```

---

## Memory Management

```move
module learning::vector_memory {
    use std::vector;
    
    // ============================================
    // Proper cleanup
    // ============================================
    
    // ✅ Consuming vector<T: drop>
    public fun cleanup_droppable(v: vector<u64>) {
        // u64 has drop, so vector<u64> has drop
        // Automatically cleaned up at end of scope
        let _ = v;  // or just let v go out of scope
    }
    
    // ✅ Consuming vector<T> where T doesn't have drop
    struct Asset {
        id: u64,
        amount: u64,
    }
    
    public fun cleanup_non_droppable(v: vector<Asset>) {
        // Must explicitly consume each element
        while (!vector::is_empty(&v)) {
            let Asset { id: _, amount: _ } = vector::pop_back(&mut v);
        };
        vector::destroy_empty(v);
    }
    
    // ============================================
    // Efficient patterns
    // ============================================
    
    // Prefer swap_remove for unordered removal (O(1))
    public fun remove_by_value(v: &mut vector<u64>, target: u64): bool {
        let len = vector::length(v);
        let i = 0u64;
        while (i < len) {
            if (*vector::borrow(v, i) == target) {
                vector::swap_remove(v, i);
                return true
            };
            i = i + 1;
        };
        false
    }
    
    // Batch remove (process in reverse to maintain indices)
    public fun remove_indices(v: &mut vector<u64>, mut indices: vector<u64>) {
        // Sort indices in descending order
        // Then remove from back to front (indices don't shift)
        let n = vector::length(&indices);
        // Simple insertion sort descending
        let i = 1u64;
        while (i < n) {
            let j = i;
            while (j > 0 && *vector::borrow(&indices, j) > *vector::borrow(&indices, j - 1)) {
                vector::swap(&mut indices, j - 1, j);
                j = j - 1;
            };
            i = i + 1;
        };
        
        // Remove in descending order
        while (!vector::is_empty(&indices)) {
            let idx = vector::pop_back(&mut indices);
            if (idx < vector::length(v)) {
                vector::remove(v, idx);
            };
        };
    }
    
    // ============================================
    // Move vs Copy semantics
    // ============================================
    
    // Moving vector (transfer ownership)
    public fun take_vector(v: vector<u64>): u64 {
        // v is moved here, caller can't use v anymore
        let sum = 0u64;
        let len = vector::length(&v);
        let i = 0u64;
        while (i < len) {
            sum = sum + *vector::borrow(&v, i);
            i = i + 1;
        };
        sum  // v goes out of scope and is dropped
    }
    
    // Borrowing vector (no ownership transfer)
    public fun sum_borrowed(v: &vector<u64>): u64 {
        // v is borrowed, caller still owns it
        let sum = 0u64;
        let i = 0u64;
        while (i < vector::length(v)) {
            sum = sum + *vector::borrow(v, i);
            i = i + 1;
        };
        sum
    }
    
    #[test]
    public fun test_remove_by_value() {
        let v = vector[1u64, 2u64, 3u64, 4u64, 5u64];
        let removed = remove_by_value(&mut v, 3);
        assert!(removed, 0);
        assert!(vector::length(&v) == 4, 1);
        assert!(!vector::contains(&v, &3u64), 2);
    }
}
```

---

## ตัวอย่างโปรแกรม: Portfolio Tracker

```move
module learning::portfolio_tracker {
    use std::signer;
    use std::vector;
    use std::string::{Self, String};
    
    // ============================================
    // Types
    // ============================================
    
    struct Position has copy, drop, store {
        token: address,
        symbol: String,
        amount: u64,
        avg_cost: u64,  // in USD * 1e8
        current_price: u64,
    }
    
    struct Portfolio has key {
        owner: address,
        positions: vector<Position>,
        cash: u64,
        total_invested: u64,
        trade_count: u64,
    }
    
    struct Trade has drop, store {
        token: address,
        is_buy: bool,
        amount: u64,
        price: u64,
        timestamp: u64,
    }
    
    struct TradeHistory has key {
        trades: vector<Trade>,
    }
    
    // ============================================
    // Error codes
    // ============================================
    
    const E_NOT_INITIALIZED: u64 = 1;
    const E_INSUFFICIENT_CASH: u64 = 2;
    const E_POSITION_NOT_FOUND: u64 = 3;
    const E_INSUFFICIENT_POSITION: u64 = 4;
    const E_ALREADY_INITIALIZED: u64 = 5;
    
    // ============================================
    // Initialize
    // ============================================
    
    public entry fun initialize(user: &signer, initial_cash: u64) {
        let addr = signer::address_of(user);
        assert!(!exists<Portfolio>(addr), E_ALREADY_INITIALIZED);
        
        move_to(user, Portfolio {
            owner: addr,
            positions: vector::empty(),
            cash: initial_cash,
            total_invested: 0,
            trade_count: 0,
        });
        
        move_to(user, TradeHistory {
            trades: vector::empty(),
        });
    }
    
    // ============================================
    // Buy Token
    // ============================================
    
    public entry fun buy(
        user: &signer,
        token: address,
        symbol: vector<u8>,
        amount: u64,
        price: u64,
        timestamp: u64,
    ) acquires Portfolio, TradeHistory {
        let addr = signer::address_of(user);
        let cost = amount * price / 1_000_000_00;  // normalize
        
        let portfolio = borrow_global_mut<Portfolio>(addr);
        assert!(portfolio.cash >= cost, E_INSUFFICIENT_CASH);
        
        portfolio.cash = portfolio.cash - cost;
        portfolio.total_invested = portfolio.total_invested + cost;
        portfolio.trade_count = portfolio.trade_count + 1;
        
        // Find existing position or add new
        let found = false;
        let i = 0u64;
        while (i < vector::length(&portfolio.positions)) {
            let pos = vector::borrow_mut(&mut portfolio.positions, i);
            if (pos.token == token) {
                // Update average cost
                let total_cost = pos.amount * pos.avg_cost + amount * price;
                let new_amount = pos.amount + amount;
                pos.avg_cost = total_cost / new_amount;
                pos.amount = new_amount;
                found = true;
                break
            };
            i = i + 1;
        };
        
        if (!found) {
            vector::push_back(&mut portfolio.positions, Position {
                token,
                symbol: string::utf8(symbol),
                amount,
                avg_cost: price,
                current_price: price,
            });
        };
        
        // Record trade
        let history = borrow_global_mut<TradeHistory>(addr);
        vector::push_back(&mut history.trades, Trade {
            token,
            is_buy: true,
            amount,
            price,
            timestamp,
        });
    }
    
    // ============================================
    // Sell Token
    // ============================================
    
    public entry fun sell(
        user: &signer,
        token: address,
        amount: u64,
        price: u64,
        timestamp: u64,
    ) acquires Portfolio, TradeHistory {
        let addr = signer::address_of(user);
        let portfolio = borrow_global_mut<Portfolio>(addr);
        
        // Find position
        let found = false;
        let pos_idx = 0u64;
        let i = 0u64;
        while (i < vector::length(&portfolio.positions)) {
            let pos = vector::borrow(&portfolio.positions, i);
            if (pos.token == token) {
                assert!(pos.amount >= amount, E_INSUFFICIENT_POSITION);
                pos_idx = i;
                found = true;
                break
            };
            i = i + 1;
        };
        
        assert!(found, E_POSITION_NOT_FOUND);
        
        let proceeds = amount * price / 1_000_000_00;
        portfolio.cash = portfolio.cash + proceeds;
        portfolio.trade_count = portfolio.trade_count + 1;
        
        // Update or remove position
        let pos = vector::borrow_mut(&mut portfolio.positions, pos_idx);
        pos.amount = pos.amount - amount;
        
        if (pos.amount == 0) {
            vector::swap_remove(&mut portfolio.positions, pos_idx);
        };
        
        // Record trade
        let history = borrow_global_mut<TradeHistory>(addr);
        vector::push_back(&mut history.trades, Trade {
            token,
            is_buy: false,
            amount,
            price,
            timestamp,
        });
    }
    
    // ============================================
    // Update Prices
    // ============================================
    
    public entry fun update_prices(
        user: &signer,
        tokens: vector<address>,
        prices: vector<u64>,
    ) acquires Portfolio {
        let addr = signer::address_of(user);
        let portfolio = borrow_global_mut<Portfolio>(addr);
        
        assert!(vector::length(&tokens) == vector::length(&prices), 0);
        
        let i = 0u64;
        while (i < vector::length(&portfolio.positions)) {
            let pos = vector::borrow_mut(&mut portfolio.positions, i);
            let j = 0u64;
            while (j < vector::length(&tokens)) {
                if (pos.token == *vector::borrow(&tokens, j)) {
                    pos.current_price = *vector::borrow(&prices, j);
                    break
                };
                j = j + 1;
            };
            i = i + 1;
        };
    }
    
    // ============================================
    // View Functions
    // ============================================
    
    #[view]
    public fun get_portfolio_value(addr: address): (u64, u64, u64) acquires Portfolio {
        let portfolio = borrow_global<Portfolio>(addr);
        
        // Calculate current value of all positions
        let positions_value = 0u64;
        let i = 0u64;
        while (i < vector::length(&portfolio.positions)) {
            let pos = vector::borrow(&portfolio.positions, i);
            positions_value = positions_value + pos.amount * pos.current_price / 1_000_000_00;
            i = i + 1;
        };
        
        let total_value = portfolio.cash + positions_value;
        let pnl = if (total_value >= portfolio.total_invested) {
            total_value - portfolio.total_invested
        } else {
            0  // Would be negative in real impl
        };
        
        (total_value, pnl, portfolio.trade_count)
    }
    
    #[view]
    public fun get_top_positions(addr: address, n: u64): vector<Position> acquires Portfolio {
        let portfolio = borrow_global<Portfolio>(addr);
        let positions = &portfolio.positions;
        
        // Get top n by value
        let result = vector::empty<Position>();
        let len = vector::length(positions);
        let count = if (len < n) { len } else { n };
        
        // Simple: just take first n (in real impl, sort by value first)
        let i = 0u64;
        while (i < count) {
            vector::push_back(&mut result, *vector::borrow(positions, i));
            i = i + 1;
        };
        
        result
    }
    
    // ============================================
    // Analytics
    // ============================================
    
    #[view]
    public fun get_win_rate(addr: address): u64 acquires Portfolio, TradeHistory {
        let history = borrow_global<TradeHistory>(addr);
        let trades = &history.trades;
        let total = vector::length(trades);
        
        if (total == 0) return 0;
        
        // Count profitable trades (simplified)
        let profitable = 0u64;
        let i = 0u64;
        while (i < total) {
            let trade = vector::borrow(trades, i);
            // In real impl, compare buy/sell pairs
            // Here simplified
            if (!trade.is_buy) {
                profitable = profitable + 1;
            };
            i = i + 1;
        };
        
        profitable * 100 / total
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test(user = @0x1)]
    public fun test_portfolio(user: &signer) acquires Portfolio, TradeHistory {
        initialize(user, 100_000_00u64);  // $100,000
        
        let addr = signer::address_of(user);
        
        // Buy ETH at $3000
        buy(user, @0x10, b"ETH", 10_000_000_00, 3_000_000_000_00, 0);
        
        let (total, _pnl, trade_count) = get_portfolio_value(addr);
        assert!(trade_count == 1, 0);
        
        // Update ETH price to $3500
        update_prices(user, vector[@0x10], vector[3_500_000_000_00]);
        
        // Sell half ETH
        sell(user, @0x10, 5_000_000_00, 3_500_000_000_00, 1);
        
        let (_, _, count) = get_portfolio_value(addr);
        assert!(count == 2, 1);
        
        let _ = total;
    }
}
```

---

## สรุป Vectors

| Operation | Complexity | Function |
|-----------|-----------|---------|
| Access by index | O(1) | `borrow`, `borrow_mut` |
| Push to end | O(1) amortized | `push_back` |
| Pop from end | O(1) | `pop_back` |
| Swap-remove | O(1) | `swap_remove` |
| Remove by index | O(n) | `remove` |
| Contains | O(n) | `contains` |
| Sort | O(n²) | custom |

---

**ต่อไป**: [Part 18 - Strings Deep Dive →](part-18-strings.md)
