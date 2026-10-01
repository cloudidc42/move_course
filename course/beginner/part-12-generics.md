# Part 12: Generics

## สารบัญ
- [Generic Functions](#generic-functions)
- [Generic Structs](#generic-structs)
- [Ability Constraints](#ability-constraints)
- [Phantom Types](#phantom-types)
- [Multiple Type Parameters](#multiple-type-parameters)
- [ตัวอย่างโปรแกรม: Generic Data Structures](#ตัวอย่างโปรแกรม-generic-data-structures)

---

## Generic Functions

```move
module learning::generic_functions {
    use std::vector;

    // ============================================
    // Basic Generic Function
    // ============================================

    // T คือ type parameter - กำหนดตอน call
    public fun identity<T>(value: T): T {
        value
    }

    // ใช้งาน:
    // identity<u64>(42)     → 42
    // identity<bool>(true)  → true
    // identity<vector<u8>>(b"hello") → b"hello"

    // ============================================
    // Generic with multiple params
    // ============================================

    public fun swap<A, B>(a: A, b: B): (B, A) {
        (b, a)
    }

    // let (b, a) = swap<u64, bool>(1, true);
    // a == 1, b == true

    // ============================================
    // Generic with constraints
    // ============================================

    // T ต้องมี copy เพื่อ copy ค่า
    public fun duplicate<T: copy>(value: T): (T, T) {
        (value, value)
    }

    // T ต้องมี drop เพื่อทิ้งค่า
    public fun discard<T: drop>(value: T) {
        // value ถูก drop ที่ end of scope
    }

    // T ต้องมี copy + drop
    public fun copy_and_discard<T: copy + drop>(value: T): T {
        let copy1 = value;  // copy
        // copy1 จะถูก drop
        value  // return original
    }

    // ============================================
    // Generic swap in vector
    // ============================================

    public fun vector_swap<T>(v: &mut vector<T>, i: u64, j: u64) {
        let len = vector::length(v);
        assert!(i < len && j < len, 0);
        if (i == j) return;
        vector::swap(v, i, j);
    }

    // ============================================
    // Generic map function
    // ============================================

    // Apply function f to each element, collect results
    // Note: Move doesn't have closures, so this is a simplified version
    public fun map_u64<T: store>(
        values: vector<u64>,
        // In Move, we can't pass functions directly
        // This is a pattern workaround
    ): vector<u64> {
        // Actual implementation depends on the transformation
        values
    }
}
```

---

## Generic Structs

```move
module learning::generic_structs {
    use std::vector;
    use std::option::{Self, Option};

    // ============================================
    // Simple Generic Struct
    // ============================================

    struct Box<T> has store {
        value: T,
    }

    public fun new_box<T: store>(value: T): Box<T> {
        Box { value }
    }

    public fun unbox<T: store>(box: Box<T>): T {
        let Box { value } = box;
        value
    }

    public fun borrow_box<T: store>(box: &Box<T>): &T {
        &box.value
    }

    public fun borrow_box_mut<T: store>(box: &mut Box<T>): &mut T {
        &mut box.value
    }

    // ============================================
    // Pair<A, B>
    // ============================================

    struct Pair<A, B> has copy, drop, store {
        first: A,
        second: B,
    }

    public fun new_pair<A: copy + drop + store, B: copy + drop + store>(
        first: A,
        second: B,
    ): Pair<A, B> {
        Pair { first, second }
    }

    public fun get_first<A: copy + drop + store, B: copy + drop + store>(
        pair: &Pair<A, B>
    ): A {
        pair.first
    }

    public fun get_second<A: copy + drop + store, B: copy + drop + store>(
        pair: &Pair<A, B>
    ): B {
        pair.second
    }

    // ============================================
    // Stack<T>
    // ============================================

    struct Stack<T: store> has store {
        items: vector<T>,
    }

    public fun new_stack<T: store>(): Stack<T> {
        Stack { items: vector::empty<T>() }
    }

    public fun push<T: store>(stack: &mut Stack<T>, item: T) {
        vector::push_back(&mut stack.items, item);
    }

    public fun pop<T: store>(stack: &mut Stack<T>): Option<T> {
        if (vector::is_empty(&stack.items)) {
            option::none()
        } else {
            option::some(vector::pop_back(&mut stack.items))
        }
    }

    public fun peek<T: store>(stack: &Stack<T>): Option<&T> {
        let len = vector::length(&stack.items);
        if (len == 0) {
            option::none()
        } else {
            option::some(vector::borrow(&stack.items, len - 1))
        }
    }

    public fun stack_size<T: store>(stack: &Stack<T>): u64 {
        vector::length(&stack.items)
    }

    // ============================================
    // Queue<T>
    // ============================================

    struct Queue<T: store> has store {
        items: vector<T>,
        head: u64,
    }

    public fun new_queue<T: store>(): Queue<T> {
        Queue { items: vector::empty<T>(), head: 0 }
    }

    public fun enqueue<T: store>(queue: &mut Queue<T>, item: T) {
        vector::push_back(&mut queue.items, item);
    }

    public fun dequeue<T: store>(queue: &mut Queue<T>): Option<T> {
        let len = vector::length(&queue.items);
        if (queue.head >= len) {
            option::none()
        } else {
            let item = vector::remove(&mut queue.items, queue.head);
            // Note: In real impl, use circular buffer for efficiency
            option::some(item)
        }
    }

    public fun queue_size<T: store>(queue: &Queue<T>): u64 {
        let len = vector::length(&queue.items);
        if (queue.head >= len) { 0 }
        else { len - queue.head }
    }
}
```

---

## Ability Constraints

```move
module learning::ability_constraints {
    use std::vector;

    // ============================================
    // Constraint Summary Table
    // ============================================

    // T: copy   → can copy, pass by value multiple times
    // T: drop   → can be discarded at end of scope
    // T: store  → can be stored in structs, vectors, tables
    // T: key    → can be top-level global storage resource
    // T: copy + drop   → most convenient, like primitive types
    // T: store  → minimum for use in collections

    // ============================================
    // Constraints in Practice
    // ============================================

    // Collection that stores items (needs store)
    struct Collection<T: store> has store {
        items: vector<T>,
        count: u64,
    }

    public fun new_collection<T: store>(): Collection<T> {
        Collection { items: vector::empty(), count: 0 }
    }

    public fun add_item<T: store>(col: &mut Collection<T>, item: T) {
        vector::push_back(&mut col.items, item);
        col.count = col.count + 1;
    }

    // Needs copy to peek without consuming
    public fun get_item<T: copy + store>(col: &Collection<T>, idx: u64): T {
        *vector::borrow(&col.items, idx)
    }

    // ============================================
    // Printable pattern (workaround for no Display trait)
    // ============================================

    // สร้าง "trait-like" behavior ด้วย generic constraints
    // ใน Move ไม่มี trait แต่สามารถใช้ phantom types ช่วยได้

    struct Printable<T: copy + drop + store> has copy, drop, store {
        value: T,
        label: vector<u8>,
    }

    public fun wrap_printable<T: copy + drop + store>(
        value: T,
        label: vector<u8>,
    ): Printable<T> {
        Printable { value, label }
    }

    public fun get_value<T: copy + drop + store>(p: &Printable<T>): T {
        p.value
    }

    // ============================================
    // Constraint Propagation
    // ============================================

    // ถ้า T มี copy + drop, Container<T> ก็มี copy + drop ได้
    struct Container<T: copy + drop + store> has copy, drop, store {
        data: T,
        metadata: u64,
    }

    // ✅ สร้างได้เพราะ u64 มี copy + drop + store
    // let c: Container<u64> = Container { data: 42, metadata: 0 };
    // let c2 = c;  // copy ได้

    // ============================================
    // Complex constraints example
    // ============================================

    // Sorter ต้องการ T ที่ copy + drop (เพราะต้อง compare และ swap)
    public fun bubble_sort<T: copy + drop>(v: &mut vector<T>, less_than: |(&T, &T)| bool) {
        let n = vector::length(v);
        let i = 0u64;
        while (i < n) {
            let j = 0u64;
            while (j < n - i - 1) {
                let a = vector::borrow(v, j);
                let b = vector::borrow(v, j + 1);
                if (less_than(b, a)) {
                    vector::swap(v, j, j + 1);
                };
                j = j + 1;
            };
            i = i + 1;
        };
    }
}
```

---

## Phantom Types

```move
module learning::phantom_types {
    use std::signer;

    // ============================================
    // Phantom Types สำหรับ Type Safety
    // ============================================

    // phantom T ไม่ได้ถูกใช้ใน fields แต่ใช้สำหรับ type safety
    struct Coin<phantom CoinType> has key, store {
        amount: u64,
    }

    // CoinTypes
    struct APT {}
    struct USDC {}
    struct ETH {}

    // Coin<APT> และ Coin<USDC> เป็น types คนละตัว!
    // ดังนั้นไม่สามารถ mix ได้โดยไม่ตั้งใจ

    public fun create_apt_coin(amount: u64): Coin<APT> {
        Coin<APT> { amount }
    }

    public fun create_usdc_coin(amount: u64): Coin<USDC> {
        Coin<USDC> { amount }
    }

    // ฟังก์ชันนี้ทำงานกับ APT เท่านั้น
    public fun add_apt(a: Coin<APT>, b: Coin<APT>): Coin<APT> {
        let Coin { amount: amount_a } = a;
        let Coin { amount: amount_b } = b;
        Coin<APT> { amount: amount_a + amount_b }
    }

    // ❌ add_apt(apt_coin, usdc_coin)  // ERROR: type mismatch!

    // ============================================
    // Phantom for Access Control
    // ============================================

    struct ReadOnly {}
    struct ReadWrite {}

    struct DatabaseAccess<phantom Permission> has store {
        db_id: u64,
    }

    // ฟังก์ชัน read ทำงานกับทั้ง ReadOnly และ ReadWrite
    public fun read_data<P>(access: &DatabaseAccess<P>): u64 {
        access.db_id
    }

    // ฟังก์ชัน write ทำงานกับ ReadWrite เท่านั้น
    public fun write_data(access: &DatabaseAccess<ReadWrite>, value: u64) {
        // Only ReadWrite access can write
        // Compile-time guarantee!
        let _ = value;
    }

    // ============================================
    // Phantom for State Machine
    // ============================================

    struct Pending {}
    struct Active {}
    struct Expired {}

    struct Subscription<phantom State> has store {
        id: u64,
        user: address,
        expires_at: u64,
    }

    // ต้อง activate Pending subscription ก่อนใช้งาน
    public fun activate(
        sub: Subscription<Pending>
    ): Subscription<Active> {
        let Subscription { id, user, expires_at } = sub;
        Subscription<Active> { id, user, expires_at }
    }

    // ต้อง expire Active subscription เท่านั้น
    public fun expire(
        sub: Subscription<Active>
    ): Subscription<Expired> {
        let Subscription { id, user, expires_at } = sub;
        Subscription<Expired> { id, user, expires_at }
    }

    // ❌ activate(expired_sub)  // ERROR: type mismatch!
}
```

---

## Multiple Type Parameters

```move
module learning::multiple_params {
    use std::vector;

    // ============================================
    // Two type parameters
    // ============================================

    struct Map<K: copy + drop + store, V: store> has store {
        keys: vector<K>,
        values: vector<V>,
    }

    public fun new_map<K: copy + drop + store, V: store>(): Map<K, V> {
        Map { keys: vector::empty(), values: vector::empty() }
    }

    public fun map_insert<K: copy + drop + store, V: store>(
        map: &mut Map<K, V>,
        key: K,
        value: V,
    ) {
        // Simple implementation (no duplicate check for brevity)
        vector::push_back(&mut map.keys, key);
        vector::push_back(&mut map.values, value);
    }

    public fun map_get<K: copy + drop + store, V: copy + store>(
        map: &Map<K, V>,
        key: &K,
    ): (bool, V) {
        let len = vector::length(&map.keys);
        let i = 0u64;
        // Need default value workaround since Move lacks Option easily
        let found = false;
        let result_idx = 0u64;

        while (i < len) {
            // Simplified: compare by position
            // Real impl would need equality check
            i = i + 1;
        };

        if (found) {
            (true, *vector::borrow(&map.values, result_idx))
        } else {
            // This is a simplification - real code needs better handling
            (false, *vector::borrow(&map.values, 0))
        }
    }

    // ============================================
    // Three type parameters: Result<T, E>
    // ============================================

    struct Result<T: store, E: copy + drop + store> has store {
        is_ok: bool,
        ok_val: vector<T>,      // contains 0 or 1 element
        err_val: vector<E>,     // contains 0 or 1 element
    }

    public fun ok<T: store, E: copy + drop + store>(value: T): Result<T, E> {
        let ok_val = vector::empty<T>();
        vector::push_back(&mut ok_val, value);
        Result {
            is_ok: true,
            ok_val,
            err_val: vector::empty<E>(),
        }
    }

    public fun err<T: store, E: copy + drop + store>(error: E): Result<T, E> {
        let err_val = vector::empty<E>();
        vector::push_back(&mut err_val, error);
        Result {
            is_ok: false,
            ok_val: vector::empty<T>(),
            err_val,
        }
    }

    public fun is_ok<T: store, E: copy + drop + store>(result: &Result<T, E>): bool {
        result.is_ok
    }

    public fun unwrap_err<T: store, E: copy + drop + store>(result: Result<T, E>): E {
        assert!(!result.is_ok, 0);
        let Result { is_ok: _, ok_val, err_val } = result;
        // Clean up ok_val (empty)
        vector::destroy_empty(ok_val);
        vector::pop_back(&mut err_val)
        // Note: err_val needs to be cleaned up too in real impl
    }
}
```

---

## ตัวอย่างโปรแกรม: Generic Data Structures

```move
module learning::generic_ds {
    use std::vector;
    use std::option::{Self, Option};

    // ============================================
    // Generic Binary Heap (Min-Heap)
    // ============================================

    struct MinHeap<T: copy + drop + store> has store {
        data: vector<T>,
    }

    public fun new_heap<T: copy + drop + store>(): MinHeap<T> {
        MinHeap { data: vector::empty() }
    }

    public fun heap_size<T: copy + drop + store>(heap: &MinHeap<T>): u64 {
        vector::length(&heap.data)
    }

    public fun heap_push_u64(heap: &mut MinHeap<u64>, val: u64) {
        vector::push_back(&mut heap.data, val);
        let len = vector::length(&heap.data);
        // Sift up
        let i = len - 1;
        while (i > 0) {
            let parent = (i - 1) / 2;
            if (*vector::borrow(&heap.data, i) < *vector::borrow(&heap.data, parent)) {
                vector::swap(&mut heap.data, i, parent);
                i = parent;
            } else {
                break
            }
        };
    }

    public fun heap_pop_u64(heap: &mut MinHeap<u64>): Option<u64> {
        if (vector::is_empty(&heap.data)) {
            return option::none()
        };

        let len = vector::length(&heap.data);
        vector::swap(&mut heap.data, 0, len - 1);
        let min_val = vector::pop_back(&mut heap.data);

        // Sift down
        let len = vector::length(&heap.data);
        let i = 0u64;
        loop {
            let left = 2 * i + 1;
            let right = 2 * i + 2;
            let smallest = i;

            if (left < len && *vector::borrow(&heap.data, left) < *vector::borrow(&heap.data, smallest)) {
                smallest = left;
            };
            if (right < len && *vector::borrow(&heap.data, right) < *vector::borrow(&heap.data, smallest)) {
                smallest = right;
            };

            if (smallest != i) {
                vector::swap(&mut heap.data, i, smallest);
                i = smallest;
            } else {
                break
            }
        };

        option::some(min_val)
    }

    // ============================================
    // Generic Trie (Simplified)
    // ============================================

    struct TrieNode<V: copy + drop + store> has store {
        children: vector<u8>,         // child characters
        child_indices: vector<u64>,   // indices into nodes array
        value: Option<V>,
    }

    struct Trie<V: copy + drop + store> has store {
        nodes: vector<TrieNode<V>>,
    }

    public fun new_trie<V: copy + drop + store>(): Trie<V> {
        let root = TrieNode<V> {
            children: vector::empty(),
            child_indices: vector::empty(),
            value: option::none(),
        };
        let nodes = vector::empty();
        vector::push_back(&mut nodes, root);
        Trie { nodes }
    }

    // ============================================
    // Generic Graph (Adjacency List)
    // ============================================

    struct Graph<N: copy + drop + store> has store {
        nodes: vector<N>,
        edges: vector<vector<u64>>,  // adjacency list
    }

    public fun new_graph<N: copy + drop + store>(): Graph<N> {
        Graph {
            nodes: vector::empty(),
            edges: vector::empty(),
        }
    }

    public fun add_node<N: copy + drop + store>(graph: &mut Graph<N>, node: N): u64 {
        let id = vector::length(&graph.nodes);
        vector::push_back(&mut graph.nodes, node);
        vector::push_back(&mut graph.edges, vector::empty());
        id
    }

    public fun add_edge<N: copy + drop + store>(
        graph: &mut Graph<N>,
        from: u64,
        to: u64,
    ) {
        let from_edges = vector::borrow_mut(&mut graph.edges, from);
        vector::push_back(from_edges, to);
    }

    public fun get_neighbors<N: copy + drop + store>(
        graph: &Graph<N>,
        node_id: u64,
    ): &vector<u64> {
        vector::borrow(&graph.edges, node_id)
    }

    // BFS using the graph
    public fun bfs<N: copy + drop + store>(
        graph: &Graph<N>,
        start: u64,
    ): vector<u64> {
        let visited = vector::empty<u64>();
        let queue = vector::empty<u64>();

        vector::push_back(&mut queue, start);

        while (!vector::is_empty(&queue)) {
            let current = vector::remove(&mut queue, 0);

            // Check if already visited
            let already_visited = false;
            let vi = 0u64;
            while (vi < vector::length(&visited)) {
                if (*vector::borrow(&visited, vi) == current) {
                    already_visited = true;
                    break
                };
                vi = vi + 1;
            };

            if (!already_visited) {
                vector::push_back(&mut visited, current);
                let neighbors = vector::borrow(&graph.edges, current);
                let ni = 0u64;
                while (ni < vector::length(neighbors)) {
                    vector::push_back(&mut queue, *vector::borrow(neighbors, ni));
                    ni = ni + 1;
                };
            };
        };

        visited
    }

    // ============================================
    // Tests
    // ============================================

    #[test]
    public fun test_heap() {
        let heap = new_heap<u64>();

        heap_push_u64(&mut heap, 5);
        heap_push_u64(&mut heap, 3);
        heap_push_u64(&mut heap, 8);
        heap_push_u64(&mut heap, 1);
        heap_push_u64(&mut heap, 4);

        // Should pop in order: 1, 3, 4, 5, 8
        let v1 = option::destroy_some(heap_pop_u64(&mut heap));
        let v2 = option::destroy_some(heap_pop_u64(&mut heap));
        let v3 = option::destroy_some(heap_pop_u64(&mut heap));

        assert!(v1 == 1, 0);
        assert!(v2 == 3, 1);
        assert!(v3 == 4, 2);

        // Cleanup
        let MinHeap { data } = heap;
        while (!vector::is_empty(&data)) {
            vector::pop_back(&mut data);
        };
        vector::destroy_empty(data);
    }

    #[test]
    public fun test_graph() {
        let graph = new_graph<u64>();

        let n0 = add_node(&mut graph, 0u64);
        let n1 = add_node(&mut graph, 1u64);
        let n2 = add_node(&mut graph, 2u64);
        let n3 = add_node(&mut graph, 3u64);

        add_edge(&mut graph, n0, n1);
        add_edge(&mut graph, n0, n2);
        add_edge(&mut graph, n1, n3);
        add_edge(&mut graph, n2, n3);

        let visited = bfs(&graph, n0);
        assert!(vector::length(&visited) == 4, 0);

        // Cleanup
        let Graph { nodes, edges } = graph;
        while (!vector::is_empty(&nodes)) { vector::pop_back(&mut nodes); };
        vector::destroy_empty(nodes);
        while (!vector::is_empty(&edges)) {
            let e = vector::pop_back(&mut edges);
            while (!vector::is_empty(&e)) { vector::pop_back(&mut e); };
            vector::destroy_empty(e);
        };
        vector::destroy_empty(edges);
        let _ = visited;
    }
}
```

---

## สรุป Generics

| Pattern | Syntax | Use case |
|---------|--------|---------|
| Generic function | `fun f<T>()` | Reusable algorithms |
| With constraints | `fun f<T: copy + drop>()` | Need specific operations |
| Generic struct | `struct S<T>` | Data containers |
| Phantom type | `struct S<phantom T>` | Type-level tags |
| Multi-params | `struct M<K, V>` | Key-value structures |

---

## แบบฝึกหัด

### Exercise 12.1: Implement Option<T>
สร้าง `Option<T>` ของตัวเองที่มี:
- `some(value: T): Option<T>`
- `none(): Option<T>`
- `is_some(opt: &Option<T>): bool`
- `unwrap(opt: Option<T>): T`

### Exercise 12.2: Generic Cache
สร้าง cache ที่:
- เก็บ key-value pairs
- มี TTL (time-to-live)
- Generic บน key type และ value type

---

**ต่อไป**: [Part 13 - Error Handling →](part-13-error-handling.md)
