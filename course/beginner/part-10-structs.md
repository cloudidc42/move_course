# Part 10: Structs

## สารบัญ
- [Struct Declaration](#struct-declaration)
- [Struct Instantiation](#struct-instantiation)
- [Field Access](#field-access)
- [Struct Methods Pattern](#struct-methods-pattern)
- [Nested Structs](#nested-structs)
- [Generic Structs](#generic-structs)
- [Struct Design Patterns](#struct-design-patterns)
- [ตัวอย่างโปรแกรม: NFT System](#ตัวอย่างโปรแกรม-nft-system)

---

## Struct Declaration

```move
module learning::struct_basics {
    use std::string::String;
    use std::vector;
    
    // ============================================
    // Basic Struct
    // ============================================
    
    // Simple struct ที่ไม่มี abilities
    struct Point {
        x: u64,
        y: u64,
    }
    
    // Struct ที่มี abilities
    struct Color has copy, drop, store {
        r: u8,
        g: u8,
        b: u8,
    }
    
    // Resource struct
    struct Token has key {
        id: u64,
        owner: address,
        amount: u64,
    }
    
    // Struct ที่มี various field types
    struct ComplexStruct has key, store {
        id: u64,
        name: String,
        tags: vector<String>,
        metadata: vector<u8>,
        owner: address,
        is_active: bool,
        score: u64,
        sub_data: SubData,
    }
    
    struct SubData has store, copy, drop {
        level: u64,
        exp: u64,
    }
    
    // ============================================
    // Naming Conventions
    // ============================================
    
    // ✅ ดี: PascalCase สำหรับ struct names
    struct UserAccount has key { }
    struct LiquidityPool has key { }
    struct TokenMetadata has store { }
    
    // ✅ ดี: snake_case สำหรับ field names
    struct GoodNaming {
        user_id: u64,
        total_supply: u64,
        is_paused: bool,
        fee_basis_points: u64,
    }
}
```

---

## Struct Instantiation

```move
module learning::struct_creation {
    use std::string::{Self, String};
    use std::vector;
    
    struct Player has key, store {
        name: String,
        level: u64,
        health: u64,
        max_health: u64,
        attack: u64,
        defense: u64,
    }
    
    // ============================================
    // Various ways to create structs
    // ============================================
    
    // 1. Field-by-field initialization
    public fun create_player_verbose(name: vector<u8>): Player {
        Player {
            name: string::utf8(name),
            level: 1,
            health: 100,
            max_health: 100,
            attack: 10,
            defense: 5,
        }
    }
    
    // 2. Using variables with same name as fields (shorthand)
    public fun create_player_shorthand(
        name: String,
        level: u64,
        health: u64,
    ): Player {
        let max_health = health;  // same as field name
        Player {
            name,     // shorthand: name: name
            level,    // shorthand: level: level
            health,   // shorthand: health: health
            max_health,
            attack: 10,
            defense: 5,
        }
    }
    
    // 3. Constructor function pattern
    public fun new_player(name: vector<u8>): Player {
        let base_health = 100u64;
        Player {
            name: string::utf8(name),
            level: 1,
            health: base_health,
            max_health: base_health,
            attack: 10,
            defense: 5,
        }
    }
    
    // 4. Creating with computed values
    public fun create_scaled_player(
        name: vector<u8>,
        level: u64,
    ): Player {
        let base = 100u64;
        let health = base + (level - 1) * 50;
        let attack = 10 + (level - 1) * 3;
        let defense = 5 + (level - 1) * 2;
        
        Player {
            name: string::utf8(name),
            level,
            health,
            max_health: health,
            attack,
            defense,
        }
    }
    
    // ============================================
    // Destructuring (unpacking)
    // ============================================
    
    // Consume struct by destructuring
    public fun consume_player(player: Player): (String, u64) {
        let Player { name, level, health: _, max_health: _, attack: _, defense: _ } = player;
        (name, level)
    }
    
    // Partial destructuring (wildcard _)
    public fun get_stats(player: Player): (u64, u64, u64) {
        let Player { level, attack, defense, name: _, health: _, max_health: _ } = player;
        (level, attack, defense)
    }
}
```

---

## Field Access

```move
module learning::field_access {
    use std::string::String;
    
    struct Character has key {
        name: String,
        hp: u64,
        mp: u64,
        level: u64,
        position_x: u64,
        position_y: u64,
    }
    
    // ============================================
    // Read fields
    // ============================================
    
    // Direct field access
    public fun get_hp(c: &Character): u64 {
        c.hp
    }
    
    // Multiple fields
    public fun get_position(c: &Character): (u64, u64) {
        (c.position_x, c.position_y)
    }
    
    // Computed from fields
    public fun distance_from_origin(c: &Character): u64 {
        // Manhattan distance
        c.position_x + c.position_y
    }
    
    // ============================================
    // Modify fields
    // ============================================
    
    // แก้ไข single field
    public fun take_damage(c: &mut Character, damage: u64) {
        if (damage >= c.hp) {
            c.hp = 0;
        } else {
            c.hp = c.hp - damage;
        }
    }
    
    // แก้ไข multiple fields
    public fun move_to(c: &mut Character, x: u64, y: u64) {
        c.position_x = x;
        c.position_y = y;
    }
    
    // Level up - แก้ไขหลาย fields
    public fun level_up(c: &mut Character) {
        c.level = c.level + 1;
        let bonus_hp = 50u64;
        c.hp = c.hp + bonus_hp;
        c.mp = c.mp + 20;
    }
    
    // ============================================
    // Complex field operations
    // ============================================
    
    // Reference to field
    public fun borrow_hp(c: &Character): &u64 {
        &c.hp
    }
    
    public fun borrow_hp_mut(c: &mut Character): &mut u64 {
        &mut c.hp
    }
    
    // Conditional field access
    public fun is_alive(c: &Character): bool {
        c.hp > 0
    }
    
    public fun is_max_hp(c: &Character): bool {
        // ต้องเก็บ max_hp ด้วยเพื่อ compare
        c.hp > 0  // simplified
    }
    
    // Field validation
    public fun validate_character(c: &Character): bool {
        c.hp > 0 
            && c.level > 0 
            && c.level <= 100
    }
    
    #[test]
    fun test_field_access() {
        let mut hero = Character {
            name: std::string::utf8(b"Hero"),
            hp: 100,
            mp: 50,
            level: 1,
            position_x: 0,
            position_y: 0,
        };
        
        assert!(get_hp(&hero) == 100, 0);
        assert!(is_alive(&hero) == true, 1);
        
        take_damage(&mut hero, 30);
        assert!(get_hp(&hero) == 70, 2);
        
        move_to(&mut hero, 5, 10);
        let (x, y) = get_position(&hero);
        assert!(x == 5 && y == 10, 3);
        
        level_up(&mut hero);
        assert!(hero.level == 2, 4);
        assert!(hero.hp == 120, 5);  // 70 + 50
        
        take_damage(&mut hero, 200);
        assert!(hero.hp == 0, 6);
        assert!(is_alive(&hero) == false, 7);
    }
}
```

---

## Struct Methods Pattern

Move ไม่มี methods แบบ OOP แต่สามารถ simulate ได้

```move
module learning::struct_methods {
    use std::string::{Self, String};
    use std::vector;
    
    // ============================================
    // "Methods" ใน Move - ใช้ naming convention
    // ============================================
    
    struct Stack has key {
        elements: vector<u64>,
        capacity: u64,
    }
    
    // "Constructor"
    public fun stack_new(capacity: u64): Stack {
        Stack {
            elements: vector::empty<u64>(),
            capacity,
        }
    }
    
    // "Instance methods" - รับ self (&Stack หรือ &mut Stack)
    public fun stack_push(self: &mut Stack, value: u64) {
        assert!(vector::length(&self.elements) < self.capacity, 1);
        vector::push_back(&mut self.elements, value);
    }
    
    public fun stack_pop(self: &mut Stack): u64 {
        assert!(!vector::is_empty(&self.elements), 2);
        vector::pop_back(&mut self.elements)
    }
    
    public fun stack_peek(self: &Stack): u64 {
        assert!(!vector::is_empty(&self.elements), 2);
        *vector::borrow(&self.elements, vector::length(&self.elements) - 1)
    }
    
    public fun stack_len(self: &Stack): u64 {
        vector::length(&self.elements)
    }
    
    public fun stack_is_empty(self: &Stack): bool {
        vector::is_empty(&self.elements)
    }
    
    public fun stack_is_full(self: &Stack): bool {
        vector::length(&self.elements) >= self.capacity
    }
    
    // "Destructor"
    public fun stack_destroy(self: Stack) {
        let Stack { elements: _, capacity: _ } = self;
    }
    
    // ============================================
    // LinkedList Pattern
    // ============================================
    
    struct ListNode has store {
        value: u64,
        // ไม่สามารถ self-reference ได้ใน Move
        // ใช้ index แทน
    }
    
    struct LinkedList has key {
        nodes: vector<ListNode>,
        size: u64,
    }
    
    public fun list_new(): LinkedList {
        LinkedList {
            nodes: vector::empty<ListNode>(),
            size: 0,
        }
    }
    
    public fun list_push_front(list: &mut LinkedList, value: u64) {
        let node = ListNode { value };
        vector::insert(&mut list.nodes, 0, node);
        list.size = list.size + 1;
    }
    
    public fun list_push_back(list: &mut LinkedList, value: u64) {
        vector::push_back(&mut list.nodes, ListNode { value });
        list.size = list.size + 1;
    }
    
    public fun list_get(list: &LinkedList, index: u64): u64 {
        assert!(index < list.size, 0);
        vector::borrow(&list.nodes, index).value
    }
    
    public fun list_len(list: &LinkedList): u64 {
        list.size
    }
    
    #[test]
    fun test_stack() {
        let mut stack = stack_new(5);
        
        assert!(stack_is_empty(&stack) == true, 0);
        
        stack_push(&mut stack, 10);
        stack_push(&mut stack, 20);
        stack_push(&mut stack, 30);
        
        assert!(stack_len(&stack) == 3, 1);
        assert!(stack_peek(&stack) == 30, 2);
        
        let popped = stack_pop(&mut stack);
        assert!(popped == 30, 3);
        assert!(stack_len(&stack) == 2, 4);
        
        stack_destroy(stack);
    }
}
```

---

## Nested Structs

```move
module learning::nested_structs {
    use std::string::String;
    use std::vector;
    
    // ============================================
    // Nested struct design
    // ============================================
    
    struct Coordinates has copy, drop, store {
        x: u64,
        y: u64,
    }
    
    struct BoundingBox has copy, drop, store {
        top_left: Coordinates,
        bottom_right: Coordinates,
    }
    
    struct Map has key {
        width: u64,
        height: u64,
        bounds: BoundingBox,
        entities: vector<Entity>,
    }
    
    struct Entity has store, copy, drop {
        id: u64,
        name: String,
        position: Coordinates,
        health: u64,
    }
    
    // ============================================
    // Working with nested structs
    // ============================================
    
    public fun get_entity_x(entity: &Entity): u64 {
        entity.position.x  // access nested field
    }
    
    public fun move_entity(entity: &mut Entity, new_x: u64, new_y: u64) {
        entity.position.x = new_x;
        entity.position.y = new_y;
    }
    
    public fun is_in_bounds(entity: &Entity, map: &Map): bool {
        let pos = entity.position;
        pos.x <= map.bounds.bottom_right.x
            && pos.y <= map.bounds.bottom_right.y
            && pos.x >= map.bounds.top_left.x
            && pos.y >= map.bounds.top_left.y
    }
    
    // Distance between two entities
    public fun distance(a: &Entity, b: &Entity): u64 {
        let dx = if (a.position.x > b.position.x) {
            a.position.x - b.position.x
        } else {
            b.position.x - a.position.x
        };
        let dy = if (a.position.y > b.position.y) {
            a.position.y - b.position.y
        } else {
            b.position.y - a.position.y
        };
        dx + dy  // Manhattan distance
    }
    
    // Find entity in map
    public fun find_entity(map: &Map, entity_id: u64): (u64, bool) {
        let i = 0u64;
        let len = vector::length(&map.entities);
        
        while (i < len) {
            let entity = vector::borrow(&map.entities, i);
            if (entity.id == entity_id) {
                return (i, true)
            };
            i = i + 1;
        };
        
        (0, false)
    }
}
```

---

## Generic Structs

```move
module learning::generic_structs {
    use std::vector;
    
    // ============================================
    // Generic containers
    // ============================================
    
    // Simple Box
    struct Box<T: store> has key, store {
        value: T,
    }
    
    public fun box_new<T: store>(value: T): Box<T> {
        Box { value }
    }
    
    public fun box_get<T: store + copy>(b: &Box<T>): T {
        b.value
    }
    
    public fun box_set<T: store>(b: &mut Box<T>, new_value: T): T {
        let old = std::mem::replace(&mut b.value, new_value);
        old
        // Note: std::mem::replace ไม่มีใน Move แต่แสดงเพื่อ concept
        // ใน Move จริงต้องทำแบบอื่น
    }
    
    public fun box_open<T: store>(b: Box<T>): T {
        let Box { value } = b;
        value
    }
    
    // Pair/Tuple struct
    struct Pair<A: store + copy + drop, B: store + copy + drop> has copy, drop, store {
        first: A,
        second: B,
    }
    
    public fun pair_new<A: store + copy + drop, B: store + copy + drop>(
        first: A,
        second: B,
    ): Pair<A, B> {
        Pair { first, second }
    }
    
    public fun pair_swap<A: store + copy + drop, B: store + copy + drop>(
        p: Pair<A, B>
    ): Pair<B, A> {
        Pair { first: p.second, second: p.first }
    }
    
    // Optional type (simulated)
    struct Option<T: store + copy + drop> has copy, drop, store {
        has_value: bool,
        value: T,
        default_value: T,  // needed because we can't have None without default
    }
    
    public fun some<T: store + copy + drop>(value: T, default: T): Option<T> {
        Option { has_value: true, value, default_value: default }
    }
    
    public fun none<T: store + copy + drop>(default: T): Option<T> {
        Option { has_value: false, value: default, default_value: default }
    }
    
    public fun is_some<T: store + copy + drop>(opt: &Option<T>): bool {
        opt.has_value
    }
    
    public fun unwrap<T: store + copy + drop>(opt: Option<T>): T {
        assert!(opt.has_value, 1);
        opt.value
    }
    
    public fun unwrap_or<T: store + copy + drop>(opt: Option<T>): T {
        if (opt.has_value) { opt.value } else { opt.default_value }
    }
    
    // Generic Queue
    struct Queue<T: store + copy + drop> has key, store {
        items: vector<T>,
    }
    
    public fun queue_new<T: store + copy + drop>(): Queue<T> {
        Queue { items: vector::empty<T>() }
    }
    
    public fun queue_enqueue<T: store + copy + drop>(q: &mut Queue<T>, item: T) {
        vector::push_back(&mut q.items, item);
    }
    
    public fun queue_dequeue<T: store + copy + drop>(q: &mut Queue<T>): T {
        assert!(!vector::is_empty(&q.items), 1);
        vector::remove(&mut q.items, 0)
    }
    
    public fun queue_peek<T: store + copy + drop>(q: &Queue<T>): T {
        assert!(!vector::is_empty(&q.items), 1);
        *vector::borrow(&q.items, 0)
    }
    
    public fun queue_len<T: store + copy + drop>(q: &Queue<T>): u64 {
        vector::length(&q.items)
    }
    
    #[test]
    fun test_generic_structs() {
        // Box
        let b = box_new(42u64);
        assert!(box_get(&b) == 42, 0);
        let val = box_open(b);
        assert!(val == 42, 1);
        
        // Pair
        let p = pair_new(10u64, true);
        assert!(p.first == 10, 2);
        assert!(p.second == true, 3);
        
        let swapped = pair_swap(p);
        assert!(swapped.first == true, 4);
        assert!(swapped.second == 10, 5);
        
        // Queue
        let mut q = queue_new<u64>();
        queue_enqueue(&mut q, 1);
        queue_enqueue(&mut q, 2);
        queue_enqueue(&mut q, 3);
        
        assert!(queue_len(&q) == 3, 6);
        assert!(queue_peek(&q) == 1, 7);
        assert!(queue_dequeue(&mut q) == 1, 8);
        assert!(queue_dequeue(&mut q) == 2, 9);
        assert!(queue_len(&q) == 1, 10);
    }
}
```

---

## Struct Design Patterns

### Pattern 1: Builder Pattern

```move
module learning::builder_pattern {
    use std::string::{Self, String};
    use std::vector;
    
    struct TokenConfig has key {
        name: String,
        symbol: String,
        decimals: u8,
        max_supply: u64,
        is_mintable: bool,
        is_burnable: bool,
        is_transferable: bool,
        admin: address,
        minters: vector<address>,
    }
    
    // Builder struct (intermediate)
    struct TokenConfigBuilder {
        name: String,
        symbol: String,
        decimals: u8,
        max_supply: u64,
        is_mintable: bool,
        is_burnable: bool,
        is_transferable: bool,
        admin: address,
        minters: vector<address>,
    }
    
    // Start builder
    public fun builder_new(admin: address): TokenConfigBuilder {
        TokenConfigBuilder {
            name: string::utf8(b""),
            symbol: string::utf8(b""),
            decimals: 8,
            max_supply: 0,
            is_mintable: false,
            is_burnable: false,
            is_transferable: true,
            admin,
            minters: vector::empty<address>(),
        }
    }
    
    // Builder methods (return self for chaining)
    public fun with_name(mut b: TokenConfigBuilder, name: vector<u8>): TokenConfigBuilder {
        b.name = string::utf8(name);
        b
    }
    
    public fun with_symbol(mut b: TokenConfigBuilder, symbol: vector<u8>): TokenConfigBuilder {
        b.symbol = string::utf8(symbol);
        b
    }
    
    public fun with_decimals(mut b: TokenConfigBuilder, decimals: u8): TokenConfigBuilder {
        b.decimals = decimals;
        b
    }
    
    public fun with_max_supply(mut b: TokenConfigBuilder, max_supply: u64): TokenConfigBuilder {
        b.max_supply = max_supply;
        b
    }
    
    public fun mintable(mut b: TokenConfigBuilder): TokenConfigBuilder {
        b.is_mintable = true;
        b
    }
    
    public fun burnable(mut b: TokenConfigBuilder): TokenConfigBuilder {
        b.is_burnable = true;
        b
    }
    
    public fun add_minter(mut b: TokenConfigBuilder, minter: address): TokenConfigBuilder {
        vector::push_back(&mut b.minters, minter);
        b
    }
    
    // Build final struct
    public fun build(b: TokenConfigBuilder): TokenConfig {
        assert!(string::length(&b.name) > 0, 1);
        assert!(string::length(&b.symbol) > 0, 2);
        assert!(b.max_supply > 0, 3);
        
        TokenConfig {
            name: b.name,
            symbol: b.symbol,
            decimals: b.decimals,
            max_supply: b.max_supply,
            is_mintable: b.is_mintable,
            is_burnable: b.is_burnable,
            is_transferable: b.is_transferable,
            admin: b.admin,
            minters: b.minters,
        }
    }
    
    // ใช้ Builder pattern
    #[test(admin = @0x1)]
    fun test_builder(admin: &signer) {
        use std::signer;
        let addr = signer::address_of(admin);
        
        let config = build(
            add_minter(
                burnable(
                    mintable(
                        with_max_supply(
                            with_symbol(
                                with_name(
                                    builder_new(addr),
                                    b"My Token",
                                ),
                                b"MTK",
                            ),
                            1_000_000_000,
                        ),
                    ),
                ),
                addr,
            )
        );
        
        assert!(config.is_mintable == true, 0);
        assert!(config.is_burnable == true, 1);
        assert!(config.max_supply == 1_000_000_000, 2);
    }
}
```

---

## ตัวอย่างโปรแกรม: NFT System

```move
module learning::nft_system {
    use std::signer;
    use std::string::{Self, String};
    use std::vector;
    
    // ============================================
    // Types
    // ============================================
    
    struct NFTCollection has key {
        name: String,
        symbol: String,
        description: String,
        creator: address,
        max_supply: u64,
        current_supply: u64,
        is_minting_active: bool,
        royalty_bps: u64,
        royalty_recipient: address,
    }
    
    struct NFT has key, store {
        id: u64,
        collection: address,    // address ของ collection
        name: String,
        description: String,
        uri: String,            // IPFS/Arweave URI
        owner: address,
        attributes: vector<Attribute>,
        rarity: u8,             // 1=Common, 2=Uncommon, 3=Rare, 4=Epic, 5=Legendary
        created_at: u64,
    }
    
    struct Attribute has copy, drop, store {
        trait_type: String,
        value: String,
        score: u64,
    }
    
    struct UserNFTWallet has key {
        owned_nfts: vector<u64>,  // list of NFT IDs
    }
    
    // ============================================
    // Constants
    // ============================================
    
    const MAX_ROYALTY_BPS: u64 = 2000;  // max 20%
    const E_NOT_CREATOR: u64 = 100;
    const E_NOT_OWNER: u64 = 101;
    const E_MINTING_INACTIVE: u64 = 200;
    const E_EXCEEDS_MAX_SUPPLY: u64 = 201;
    const E_COLLECTION_NOT_FOUND: u64 = 300;
    const E_NFT_NOT_FOUND: u64 = 301;
    const E_INVALID_ROYALTY: u64 = 302;
    
    // ============================================
    // Collection Management
    // ============================================
    
    public entry fun create_collection(
        creator: &signer,
        name: vector<u8>,
        symbol: vector<u8>,
        description: vector<u8>,
        max_supply: u64,
        royalty_bps: u64,
        royalty_recipient: address,
    ) {
        assert!(royalty_bps <= MAX_ROYALTY_BPS, E_INVALID_ROYALTY);
        
        let addr = signer::address_of(creator);
        
        move_to(creator, NFTCollection {
            name: string::utf8(name),
            symbol: string::utf8(symbol),
            description: string::utf8(description),
            creator: addr,
            max_supply,
            current_supply: 0,
            is_minting_active: true,
            royalty_bps,
            royalty_recipient,
        });
    }
    
    public entry fun toggle_minting(
        creator: &signer,
        collection_addr: address,
    ) acquires NFTCollection {
        let addr = signer::address_of(creator);
        let collection = borrow_global_mut<NFTCollection>(collection_addr);
        assert!(collection.creator == addr, E_NOT_CREATOR);
        collection.is_minting_active = !collection.is_minting_active;
    }
    
    // ============================================
    // NFT Minting
    // ============================================
    
    public entry fun mint_nft(
        minter: &signer,
        collection_addr: address,
        name: vector<u8>,
        description: vector<u8>,
        uri: vector<u8>,
        rarity: u8,
        attribute_types: vector<vector<u8>>,
        attribute_values: vector<vector<u8>>,
        attribute_scores: vector<u64>,
        timestamp: u64,
    ) acquires NFTCollection, UserNFTWallet {
        let minter_addr = signer::address_of(minter);
        
        // Validate collection
        assert!(exists<NFTCollection>(collection_addr), E_COLLECTION_NOT_FOUND);
        
        let nft_id;
        {
            let collection = borrow_global_mut<NFTCollection>(collection_addr);
            assert!(collection.is_minting_active, E_MINTING_INACTIVE);
            assert!(collection.current_supply < collection.max_supply, E_EXCEEDS_MAX_SUPPLY);
            
            collection.current_supply = collection.current_supply + 1;
            nft_id = collection.current_supply;
        };
        
        // Build attributes
        let attrs_len = vector::length(&attribute_types);
        let attributes = vector::empty<Attribute>();
        let i = 0u64;
        
        while (i < attrs_len && i < vector::length(&attribute_values)) {
            let attr = Attribute {
                trait_type: string::utf8(*vector::borrow(&attribute_types, i)),
                value: string::utf8(*vector::borrow(&attribute_values, i)),
                score: *vector::borrow(&attribute_scores, i),
            };
            vector::push_back(&mut attributes, attr);
            i = i + 1;
        };
        
        // Create NFT
        let nft = NFT {
            id: nft_id,
            collection: collection_addr,
            name: string::utf8(name),
            description: string::utf8(description),
            uri: string::utf8(uri),
            owner: minter_addr,
            attributes,
            rarity,
            created_at: timestamp,
        };
        
        // Store NFT (simplified - ใน production ใช้ Table)
        // ใช้ minter's address + NFT id เป็น key
        move_to(minter, nft);
        
        // Update wallet
        if (exists<UserNFTWallet>(minter_addr)) {
            let wallet = borrow_global_mut<UserNFTWallet>(minter_addr);
            vector::push_back(&mut wallet.owned_nfts, nft_id);
        } else {
            move_to(minter, UserNFTWallet {
                owned_nfts: vector[nft_id],
            });
        }
    }
    
    // ============================================
    // NFT Operations
    // ============================================
    
    public entry fun transfer_nft(
        from: &signer,
        to: address,
        nft_addr: address,
    ) acquires NFT, UserNFTWallet {
        let from_addr = signer::address_of(from);
        
        // Get NFT and verify ownership
        let nft = borrow_global_mut<NFT>(nft_addr);
        assert!(nft.owner == from_addr, E_NOT_OWNER);
        
        let nft_id = nft.id;
        nft.owner = to;
        
        // Update wallets
        // Remove from sender's wallet
        if (exists<UserNFTWallet>(from_addr)) {
            let wallet = borrow_global_mut<UserNFTWallet>(from_addr);
            let (found, idx) = vector::index_of(&wallet.owned_nfts, &nft_id);
            if (found) {
                vector::remove(&mut wallet.owned_nfts, idx);
            }
        };
        
        // Add to recipient's wallet  
        if (exists<UserNFTWallet>(to)) {
            let wallet = borrow_global_mut<UserNFTWallet>(to);
            vector::push_back(&mut wallet.owned_nfts, nft_id);
        }
    }
    
    // ============================================
    // View Functions
    // ============================================
    
    #[view]
    public fun get_collection_info(
        collection_addr: address
    ): (String, u64, u64, bool) acquires NFTCollection {
        let c = borrow_global<NFTCollection>(collection_addr);
        (c.name, c.current_supply, c.max_supply, c.is_minting_active)
    }
    
    #[view]
    public fun get_nft_rarity(nft_addr: address): u8 acquires NFT {
        borrow_global<NFT>(nft_addr).rarity
    }
    
    #[view]
    public fun get_nft_owner(nft_addr: address): address acquires NFT {
        borrow_global<NFT>(nft_addr).owner
    }
    
    #[view]
    public fun get_user_nft_count(user_addr: address): u64 acquires UserNFTWallet {
        if (!exists<UserNFTWallet>(user_addr)) { return 0 };
        vector::length(&borrow_global<UserNFTWallet>(user_addr).owned_nfts)
    }
    
    // ============================================
    // Rarity Helpers
    // ============================================
    
    public fun rarity_name(rarity: u8): vector<u8> {
        if (rarity == 1) { b"Common" }
        else if (rarity == 2) { b"Uncommon" }
        else if (rarity == 3) { b"Rare" }
        else if (rarity == 4) { b"Epic" }
        else if (rarity == 5) { b"Legendary" }
        else { b"Unknown" }
    }
    
    public fun calculate_rarity_score(attributes: &vector<Attribute>): u64 {
        let total = 0u64;
        let i = 0u64;
        let len = vector::length(attributes);
        
        while (i < len) {
            total = total + vector::borrow(attributes, i).score;
            i = i + 1;
        };
        
        total
    }
}
```

---

## สรุป

| แนวคิด | สิ่งที่เรียนรู้ |
|-------|--------------|
| Declaration | `struct Name has abilities { fields }` |
| Instantiation | `StructName { field: value, ... }` |
| Field Access | `struct.field`, `struct.nested.field` |
| Destructuring | `let StructName { f1, f2 } = value` |
| Methods Pattern | ฟังก์ชันที่รับ `&Self` หรือ `&mut Self` |
| Generics | `struct Container<T: ability> { }` |
| Builder Pattern | สร้าง struct ด้วย method chaining |

---

## แบบฝึกหัด

### Exercise 10.1: Design Structs
ออกแบบ structs สำหรับ:
1. DeFi Protocol: Pool, Position, Reward
2. Game: Character, Item, Quest
3. DAO: Proposal, Vote, Treasury

### Exercise 10.2: Implement Methods
สร้าง `PriorityQueue<T>` struct พร้อม:
- `new()`, `push(item, priority)`, `pop()`, `peek()`
- `len()`, `is_empty()`

### Exercise 10.3: Builder Pattern
ออกแบบ Builder สำหรับ NFT metadata

---

**ก่อนหน้า**: [Part 09 - References](part-09-references.md)  
**ต่อไป**: [Part 11 - Abilities →](part-11-abilities.md)
