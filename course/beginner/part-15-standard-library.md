# Part 15: Standard Library

## สารบัญ
- [std::vector](#stdvector)
- [std::string](#stdstring)
- [std::option](#stdoption)
- [std::signer](#stdsigner)
- [aptos_std::table](#aptos_stdtable)
- [aptos_std::math64](#aptos_stdmath64)
- [ตัวอย่างโปรแกรม: NFT Collection](#ตัวอย่างโปรแกรม-nft-collection)

---

## std::vector

```move
module learning::vector_usage {
    use std::vector;
    
    // ============================================
    // Creation
    // ============================================
    
    public fun create_examples() {
        // Empty vector
        let empty: vector<u64> = vector::empty();
        
        // Vector with initial values
        let v1 = vector[1u64, 2u64, 3u64];
        let v2: vector<u8> = vector[10, 20, 30];
        
        // byte string literal (vector<u8>)
        let bytes: vector<u8> = b"Hello Move";
        
        let _ = (empty, v1, v2, bytes);
    }
    
    // ============================================
    // Basic Operations
    // ============================================
    
    public fun basic_ops() {
        let v = vector[1u64, 2u64, 3u64, 4u64, 5u64];
        
        // Length
        let len = vector::length(&v);
        assert!(len == 5, 0);
        
        // Access by index
        let first = *vector::borrow(&v, 0);
        let last = *vector::borrow(&v, len - 1);
        assert!(first == 1 && last == 5, 1);
        
        // Mutable access
        let v_mut = &mut v;
        *vector::borrow_mut(v_mut, 0) = 100;
        assert!(*vector::borrow(&v, 0) == 100, 2);
        
        // Push/pop
        vector::push_back(&mut v, 6);
        let popped = vector::pop_back(&mut v);
        assert!(popped == 6, 3);
        
        // Is empty?
        assert!(!vector::is_empty(&v), 4);
        
        let _ = (first, last);
    }
    
    // ============================================
    // Structural Operations
    // ============================================
    
    public fun structural_ops() {
        let v = vector[1u64, 2u64, 3u64, 4u64, 5u64];
        
        // Append: v1 += v2
        let v2 = vector[6u64, 7u64, 8u64];
        vector::append(&mut v, v2);
        assert!(vector::length(&v) == 8, 0);
        
        // Reverse in-place
        vector::reverse(&mut v);
        assert!(*vector::borrow(&v, 0) == 8, 1);
        
        // Contains
        assert!(vector::contains(&v, &5u64), 2);
        assert!(!vector::contains(&v, &99u64), 3);
        
        // Index of
        let (found, idx) = vector::index_of(&v, &5u64);
        assert!(found, 4);
        let _ = idx;
    }
    
    // ============================================
    // Remove Operations
    // ============================================
    
    public fun remove_ops() {
        let v = vector[1u64, 2u64, 3u64, 4u64, 5u64];
        
        // Remove by index (shifts elements - O(n))
        let removed = vector::remove(&mut v, 2);
        assert!(removed == 3, 0);
        assert!(vector::length(&v) == 4, 1);
        assert!(*vector::borrow(&v, 2) == 4, 2);
        
        // Swap-remove (O(1) but doesn't preserve order)
        let v2 = vector[1u64, 2u64, 3u64, 4u64, 5u64];
        let swapped = vector::swap_remove(&mut v2, 2);
        assert!(swapped == 3, 3);
        // [1, 2, 5, 4] - 3 removed, 5 swapped in
        
        // Destroy empty vector
        let empty = vector::empty<u64>();
        vector::destroy_empty(empty);
    }
    
    // ============================================
    // Common Patterns
    // ============================================
    
    // Filter elements
    public fun filter_positive(v: vector<u64>, threshold: u64): vector<u64> {
        let result = vector::empty<u64>();
        let i = 0u64;
        while (i < vector::length(&v)) {
            let val = *vector::borrow(&v, i);
            if (val > threshold) {
                vector::push_back(&mut result, val);
            };
            i = i + 1;
        };
        // Need to drop v since it has no drop ability - use while loop
        let len = vector::length(&v);
        while (len > 0) {
            vector::pop_back(&mut v);
            len = len - 1;
        };
        vector::destroy_empty(v);
        result
    }
    
    // Sum all elements
    public fun sum_vector(v: &vector<u64>): u64 {
        let sum = 0u64;
        let i = 0u64;
        while (i < vector::length(v)) {
            sum = sum + *vector::borrow(v, i);
            i = i + 1;
        };
        sum
    }
    
    // Find max element
    public fun find_max(v: &vector<u64>): u64 {
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
    
    // Sort (insertion sort)
    public fun insertion_sort(v: &mut vector<u64>) {
        let n = vector::length(v);
        let i = 1u64;
        while (i < n) {
            let j = i;
            while (j > 0 && *vector::borrow(v, j) < *vector::borrow(v, j - 1)) {
                vector::swap(v, j, j - 1);
                j = j - 1;
            };
            i = i + 1;
        };
    }
}
```

---

## std::string

```move
module learning::string_usage {
    use std::string::{Self, String};
    use std::vector;
    
    // ============================================
    // String Creation
    // ============================================
    
    public fun create_strings() {
        // From UTF-8 bytes
        let s1 = string::utf8(b"Hello, Move!");
        
        // Empty string
        let empty = string::utf8(b"");
        
        // From vector<u8>
        let bytes = b"blockchain";
        let s2 = string::utf8(bytes);
        
        let _ = (s1, empty, s2);
    }
    
    // ============================================
    // String Operations
    // ============================================
    
    public fun string_ops() {
        let s = string::utf8(b"Hello, World!");
        
        // Length (in bytes, not chars for UTF-8)
        let len = string::length(&s);
        assert!(len == 13, 0);
        
        // Check if empty
        assert!(!string::is_empty(&s), 1);
        
        // Get bytes
        let bytes = string::bytes(&s);
        assert!(vector::length(bytes) == 13, 2);
        
        // Append string
        let mut_s = string::utf8(b"Hello");
        string::append(&mut mut_s, string::utf8(b", World!"));
        assert!(string::length(&mut_s) == 13, 3);
        
        // Append UTF-8 bytes directly
        string::append_utf8(&mut mut_s, b" More text");
    }
    
    // ============================================
    // String Comparison
    // ============================================
    
    public fun string_compare() {
        let s1 = string::utf8(b"Hello");
        let s2 = string::utf8(b"Hello");
        let s3 = string::utf8(b"World");
        
        // Move String comparison uses == operator
        assert!(s1 == s2, 0);
        assert!(s1 != s3, 1);
    }
    
    // ============================================
    // String building patterns
    // ============================================
    
    public fun build_token_name(prefix: vector<u8>, id: u64): String {
        let name = string::utf8(prefix);
        string::append(&mut name, string::utf8(b" #"));
        
        // Convert u64 to string (simplified)
        // In practice, use aptos_std::string_utils
        let id_str = u64_to_string(id);
        string::append(&mut name, id_str);
        
        name
    }
    
    fun u64_to_string(n: u64): String {
        if (n == 0) {
            return string::utf8(b"0")
        };
        
        let bytes = vector::empty<u8>();
        let n_mut = n;
        while (n_mut > 0) {
            let digit = (n_mut % 10 as u8) + 48;  // '0' = 48
            vector::push_back(&mut bytes, digit);
            n_mut = n_mut / 10;
        };
        
        // Reverse
        vector::reverse(&mut bytes);
        string::utf8(bytes)
    }
    
    #[test]
    public fun test_string() {
        let s = build_token_name(b"NFT", 42);
        assert!(s == string::utf8(b"NFT #42"), 0);
        
        let s2 = build_token_name(b"Token", 0);
        assert!(s2 == string::utf8(b"Token #0"), 1);
    }
}
```

---

## std::option

```move
module learning::option_usage {
    use std::option::{Self, Option};
    
    // ============================================
    // Creating Options
    // ============================================
    
    public fun create_options() {
        let some_val: Option<u64> = option::some(42u64);
        let none_val: Option<u64> = option::none();
        
        assert!(option::is_some(&some_val), 0);
        assert!(option::is_none(&none_val), 1);
        
        let _ = (some_val, none_val);
    }
    
    // ============================================
    // Extracting Values
    // ============================================
    
    public fun extract_options() {
        let some_val = option::some(100u64);
        
        // borrow (doesn't consume)
        let borrowed = option::borrow(&some_val);
        assert!(*borrowed == 100, 0);
        
        // extract (consumes the option)
        let value = option::extract(&mut option::some(100u64));
        assert!(value == 100, 1);
        
        // destroy_some (panics if None)
        let val = option::destroy_some(some_val);
        assert!(val == 100, 2);
        
        // destroy_none (panics if Some)
        let none: Option<u64> = option::none();
        option::destroy_none(none);
    }
    
    // ============================================
    // Option in practice
    // ============================================
    
    struct SearchResult has drop {
        value: u64,
        index: u64,
    }
    
    public fun find_in_vector(v: &vector<u64>, target: u64): Option<SearchResult> {
        use std::vector;
        
        let i = 0u64;
        while (i < vector::length(v)) {
            if (*vector::borrow(v, i) == target) {
                return option::some(SearchResult { value: target, index: i })
            };
            i = i + 1;
        };
        option::none()
    }
    
    // Using the result
    public fun find_and_process(v: &vector<u64>, target: u64): u64 {
        let result = find_in_vector(v, target);
        if (option::is_some(&result)) {
            let sr = option::destroy_some(result);
            sr.index
        } else {
            option::destroy_none(result);
            0xFFFFFFFF  // not found sentinel
        }
    }
    
    // ============================================
    // Chaining operations
    // ============================================
    
    public fun get_config_value(config: &Option<u64>, default: u64): u64 {
        if (option::is_some(config)) {
            *option::borrow(config)
        } else {
            default
        }
    }
    
    public fun update_if_some(opt: &mut Option<u64>, new_val: u64) {
        if (option::is_some(opt)) {
            let _ = option::extract(opt);
            option::fill(opt, new_val);
        }
    }
}
```

---

## std::signer

```move
module learning::signer_usage {
    use std::signer;
    
    // ============================================
    // signer basics
    // ============================================
    
    // signer = proof of account ownership
    // Only obtainable via transaction arguments
    // Cannot be created in Move code
    
    public fun signer_examples(account: &signer) {
        // Get address from signer
        let addr = signer::address_of(account);
        
        // addr is the address of the account
        assert!(addr != @0x0, 0);
    }
    
    // ============================================
    // Authorization patterns
    // ============================================
    
    struct AdminConfig has key {
        admin: address,
    }
    
    public fun require_admin(
        caller: &signer,
        config_addr: address,
    ) acquires AdminConfig {
        let caller_addr = signer::address_of(caller);
        let config = borrow_global<AdminConfig>(config_addr);
        assert!(caller_addr == config.admin, 1);
    }
    
    // ============================================
    // Using signer in entry functions
    // ============================================
    
    struct UserProfile has key {
        name: vector<u8>,
        points: u64,
    }
    
    // Entry functions receive signer directly from transaction
    public entry fun create_profile(
        user: &signer,
        name: vector<u8>,
    ) {
        let addr = signer::address_of(user);
        assert!(!exists<UserProfile>(addr), 0);
        
        move_to(user, UserProfile {
            name,
            points: 0,
        });
    }
    
    public entry fun add_points(
        admin: &signer,
        user_addr: address,
        points: u64,
        config_addr: address,
    ) acquires AdminConfig, UserProfile {
        require_admin(admin, config_addr);
        
        assert!(exists<UserProfile>(user_addr), 1);
        let profile = borrow_global_mut<UserProfile>(user_addr);
        profile.points = profile.points + points;
    }
}
```

---

## aptos_std::table

```move
module learning::table_usage {
    use aptos_std::table::{Self, Table};
    use std::signer;
    
    // ============================================
    // Table basics
    // ============================================
    
    // Table<K, V> - key-value store
    // Unlike vector, doesn't have all contents in memory at once
    // Better for large collections (gas efficient)
    
    struct Registry has key {
        users: Table<address, UserInfo>,
        total_users: u64,
    }
    
    struct UserInfo has store {
        name: vector<u8>,
        score: u64,
        registered_at: u64,
    }
    
    public fun initialize_registry(admin: &signer) {
        move_to(admin, Registry {
            users: table::new(),
            total_users: 0,
        });
    }
    
    public fun register_user(
        registry_addr: address,
        user_addr: address,
        name: vector<u8>,
        timestamp: u64,
    ) acquires Registry {
        let registry = borrow_global_mut<Registry>(registry_addr);
        
        // Check doesn't exist
        assert!(!table::contains(&registry.users, user_addr), 0);
        
        // Add to table
        table::add(&mut registry.users, user_addr, UserInfo {
            name,
            score: 0,
            registered_at: timestamp,
        });
        
        registry.total_users = registry.total_users + 1;
    }
    
    public fun get_user_score(
        registry_addr: address,
        user_addr: address,
    ): u64 acquires Registry {
        let registry = borrow_global<Registry>(registry_addr);
        assert!(table::contains(&registry.users, user_addr), 1);
        
        let user = table::borrow(&registry.users, user_addr);
        user.score
    }
    
    public fun update_score(
        registry_addr: address,
        user_addr: address,
        new_score: u64,
    ) acquires Registry {
        let registry = borrow_global_mut<Registry>(registry_addr);
        assert!(table::contains(&registry.users, user_addr), 1);
        
        let user = table::borrow_mut(&mut registry.users, user_addr);
        user.score = new_score;
    }
    
    public fun remove_user(
        registry_addr: address,
        user_addr: address,
    ) acquires Registry {
        let registry = borrow_global_mut<Registry>(registry_addr);
        assert!(table::contains(&registry.users, user_addr), 1);
        
        // Remove returns the value
        let UserInfo { name: _, score: _, registered_at: _ } = 
            table::remove(&mut registry.users, user_addr);
        registry.total_users = registry.total_users - 1;
    }
    
    // ============================================
    // Table with more complex keys
    // ============================================
    
    struct PairKey has copy, drop, store {
        token_a: address,
        token_b: address,
    }
    
    struct PoolData has store {
        reserve_a: u64,
        reserve_b: u64,
    }
    
    struct DEX has key {
        pools: Table<PairKey, PoolData>,
    }
    
    public fun create_pool(
        dex_addr: address,
        token_a: address,
        token_b: address,
    ) acquires DEX {
        // Normalize key order
        let (ta, tb) = if (token_a < token_b) {
            (token_a, token_b)
        } else {
            (token_b, token_a)
        };
        
        let key = PairKey { token_a: ta, token_b: tb };
        let dex = borrow_global_mut<DEX>(dex_addr);
        
        assert!(!table::contains(&dex.pools, key), 0);
        
        table::add(&mut dex.pools, key, PoolData {
            reserve_a: 0,
            reserve_b: 0,
        });
    }
}
```

---

## aptos_std::math64

```move
module learning::math_usage {
    use aptos_std::math64;
    
    // ============================================
    // Math functions
    // ============================================
    
    public fun math_examples() {
        // Min and max
        let min_val = math64::min(10, 20);
        let max_val = math64::max(10, 20);
        assert!(min_val == 10 && max_val == 20, 0);
        
        // Power of 2
        let pow2 = math64::pow(2, 10);
        assert!(pow2 == 1024, 1);
        
        // Square root (floor)
        let sqrt_val = math64::sqrt(16);
        assert!(sqrt_val == 4, 2);
        
        // Average (prevents overflow)
        let avg = math64::average(100, 200);
        assert!(avg == 150, 3);
        
        // Ceil division
        let ceil_div = math64::ceil_div(7, 3);
        assert!(ceil_div == 3, 4);  // ceil(7/3) = 3
    }
    
    // ============================================
    // Common financial calculations
    // ============================================
    
    // Calculate basis points
    public fun apply_bps(amount: u64, bps: u64): u64 {
        amount * bps / 10_000
    }
    
    // Calculate fee
    public fun calculate_fee(amount: u64, fee_bps: u64): u64 {
        apply_bps(amount, fee_bps)
    }
    
    // Percentage
    public fun percentage(numerator: u64, denominator: u64, scale: u64): u64 {
        numerator * scale / denominator
    }
    
    // Compound interest (simplified)
    public fun compound_interest(
        principal: u64,
        rate_bps: u64,
        periods: u64,
    ): u64 {
        let mut amount = principal;
        let i = 0u64;
        while (i < periods) {
            let interest = apply_bps(amount, rate_bps);
            amount = amount + interest;
            // i = i + 1;  // Need let mut
        };
        // Note: need mut variables in actual code
        principal  // simplified
    }
}
```

---

## ตัวอย่างโปรแกรม: NFT Collection

```move
module learning::nft_collection {
    use std::signer;
    use std::string::{Self, String};
    use std::vector;
    use std::option::{Self, Option};
    use aptos_std::table::{Self, Table};
    
    // ============================================
    // Types
    // ============================================
    
    struct Attribute has copy, drop, store {
        trait_type: String,
        value: String,
    }
    
    struct NFT has key, store {
        id: u64,
        name: String,
        description: String,
        uri: String,
        owner: address,
        attributes: vector<Attribute>,
    }
    
    struct Collection has key {
        name: String,
        description: String,
        max_supply: u64,
        current_supply: u64,
        nfts: Table<u64, NFT>,
        owner_to_nfts: Table<address, vector<u64>>,
        creator: address,
        royalty_bps: u64,
        is_active: bool,
    }
    
    struct MintEvent has drop, store {
        id: u64,
        owner: address,
        timestamp: u64,
    }
    
    // ============================================
    // Error codes
    // ============================================
    
    const E_COLLECTION_NOT_FOUND: u64 = 1;
    const E_MAX_SUPPLY_REACHED: u64 = 2;
    const E_NOT_OWNER: u64 = 3;
    const E_COLLECTION_PAUSED: u64 = 4;
    const E_NFT_NOT_FOUND: u64 = 5;
    const E_ALREADY_INITIALIZED: u64 = 6;
    
    // ============================================
    // Initialize Collection
    // ============================================
    
    public entry fun create_collection(
        creator: &signer,
        name: vector<u8>,
        description: vector<u8>,
        max_supply: u64,
        royalty_bps: u64,
    ) {
        let creator_addr = signer::address_of(creator);
        assert!(!exists<Collection>(creator_addr), E_ALREADY_INITIALIZED);
        
        move_to(creator, Collection {
            name: string::utf8(name),
            description: string::utf8(description),
            max_supply,
            current_supply: 0,
            nfts: table::new(),
            owner_to_nfts: table::new(),
            creator: creator_addr,
            royalty_bps,
            is_active: true,
        });
    }
    
    // ============================================
    // Mint NFT
    // ============================================
    
    public entry fun mint(
        creator: &signer,
        collection_addr: address,
        to: address,
        name: vector<u8>,
        description: vector<u8>,
        uri: vector<u8>,
        _timestamp: u64,
    ) acquires Collection {
        let creator_addr = signer::address_of(creator);
        
        assert!(exists<Collection>(collection_addr), E_COLLECTION_NOT_FOUND);
        let collection = borrow_global_mut<Collection>(collection_addr);
        
        assert!(collection.creator == creator_addr, E_NOT_OWNER);
        assert!(collection.is_active, E_COLLECTION_PAUSED);
        assert!(collection.current_supply < collection.max_supply, E_MAX_SUPPLY_REACHED);
        
        let nft_id = collection.current_supply;
        
        let nft = NFT {
            id: nft_id,
            name: string::utf8(name),
            description: string::utf8(description),
            uri: string::utf8(uri),
            owner: to,
            attributes: vector::empty(),
        };
        
        table::add(&mut collection.nfts, nft_id, nft);
        
        // Update owner's NFT list
        if (table::contains(&collection.owner_to_nfts, to)) {
            let owned = table::borrow_mut(&mut collection.owner_to_nfts, to);
            vector::push_back(owned, nft_id);
        } else {
            let owned = vector[nft_id];
            table::add(&mut collection.owner_to_nfts, to, owned);
        };
        
        collection.current_supply = collection.current_supply + 1;
    }
    
    // ============================================
    // Transfer NFT
    // ============================================
    
    public entry fun transfer_nft(
        from: &signer,
        collection_addr: address,
        nft_id: u64,
        to: address,
    ) acquires Collection {
        let from_addr = signer::address_of(from);
        
        let collection = borrow_global_mut<Collection>(collection_addr);
        assert!(table::contains(&collection.nfts, nft_id), E_NFT_NOT_FOUND);
        
        let nft = table::borrow_mut(&mut collection.nfts, nft_id);
        assert!(nft.owner == from_addr, E_NOT_OWNER);
        
        // Update ownership
        nft.owner = to;
        
        // Remove from sender's list
        if (table::contains(&collection.owner_to_nfts, from_addr)) {
            let owned = table::borrow_mut(&mut collection.owner_to_nfts, from_addr);
            let (found, idx) = vector::index_of(owned, &nft_id);
            if (found) {
                vector::swap_remove(owned, idx);
            };
        };
        
        // Add to recipient's list
        if (table::contains(&collection.owner_to_nfts, to)) {
            let owned = table::borrow_mut(&mut collection.owner_to_nfts, to);
            vector::push_back(owned, nft_id);
        } else {
            table::add(&mut collection.owner_to_nfts, to, vector[nft_id]);
        };
    }
    
    // ============================================
    // View Functions
    // ============================================
    
    #[view]
    public fun get_nft_owner(collection_addr: address, nft_id: u64): address acquires Collection {
        let collection = borrow_global<Collection>(collection_addr);
        assert!(table::contains(&collection.nfts, nft_id), E_NFT_NOT_FOUND);
        table::borrow(&collection.nfts, nft_id).owner
    }
    
    #[view]
    public fun get_collection_supply(collection_addr: address): (u64, u64) acquires Collection {
        let collection = borrow_global<Collection>(collection_addr);
        (collection.current_supply, collection.max_supply)
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test(creator = @0x1, user = @0x2)]
    public fun test_nft_flow(creator: &signer, user: &signer) acquires Collection {
        let creator_addr = signer::address_of(creator);
        let user_addr = signer::address_of(user);
        
        // Create collection
        create_collection(creator, b"My NFTs", b"A test collection", 100, 250);
        
        // Mint to user
        mint(creator, creator_addr, user_addr, b"NFT #0", b"First NFT", b"https://example.com/0", 0);
        
        // Verify ownership
        let owner = get_nft_owner(creator_addr, 0);
        assert!(owner == user_addr, 0);
        
        // Check supply
        let (current, max) = get_collection_supply(creator_addr);
        assert!(current == 1, 1);
        assert!(max == 100, 2);
        
        // Transfer back to creator
        transfer_nft(user, creator_addr, 0, creator_addr);
        let new_owner = get_nft_owner(creator_addr, 0);
        assert!(new_owner == creator_addr, 3);
    }
}
```

---

## สรุป Standard Library

| Module | Key Functions |
|--------|--------------|
| `std::vector` | empty, push_back, pop_back, borrow, length, remove |
| `std::string` | utf8, append, length, bytes |
| `std::option` | some, none, is_some, is_none, borrow, extract |
| `std::signer` | address_of |
| `aptos_std::table` | new, add, borrow, borrow_mut, remove, contains |
| `aptos_std::math64` | min, max, pow, sqrt, average |

---

## แบบฝึกหัด

### Exercise 15.1: String Processing
เขียนฟังก์ชัน:
- `to_lowercase(s: String): String`
- `trim_whitespace(s: String): String`
- `split_by_delimiter(s: String, delimiter: u8): vector<String>`

### Exercise 15.2: Table-based Voting
สร้าง voting system โดยใช้ Table:
- แต่ละ voter สามารถ vote ได้ครั้งเดียว
- Track votes ด้วย Table<address, bool>
- Count results

---

**ต่อไป**: [Part 16 - Events →](part-16-events.md)
