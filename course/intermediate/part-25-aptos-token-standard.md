# Part 25: Aptos Digital Asset (Token v2)

## สารบัญ
- [Token Standard Evolution](#token-standard-evolution)
- [Digital Asset Framework](#digital-asset-framework)
- [สร้าง Collection](#สร้าง-collection)
- [Mint NFT](#mint-nft)
- [Token Properties](#token-properties)
- [ตัวอย่าง: Gaming NFT System](#ตัวอย่าง-gaming-nft-system)

---

## Token Standard Evolution

```
Token v1 (Legacy)                Token v2 / Digital Asset
─────────────────────────        ─────────────────────────────
• Direct resource storage         • Built on Aptos Objects
• Complex configuration           • Simpler, composable
• Hard to extend                  • Easy extension with refs
• Separate royalty system         • Integrated royalty
• No on-chain metadata            • On-chain metadata standard
```

Token v2 (Digital Asset) = Aptos's NFT standard
- ใช้ Object model
- Composable: เพิ่ม custom resources ได้
- On-chain metadata
- Built-in royalty support
- Soul-bound ได้

---

## Digital Asset Framework

```move
module learning::da_concepts {
    // ============================================
    // Core modules
    // ============================================
    
    // aptos_token_objects::collection - NFT Collections
    // aptos_token_objects::token      - Individual NFTs
    // aptos_token_objects::property_map - Key-value properties
    // aptos_token_objects::royalty    - Royalty configuration
    
    // ============================================
    // Key types
    // ============================================
    
    // Object<Collection>  - A collection of tokens
    // Object<Token>       - An individual NFT
    // MutatorRef          - Can mutate token fields
    // BurnRef             - Can burn the token
    // TransferRef         - Can transfer despite frozen
    // PropertyMap         - K-V properties on token
}
```

---

## สร้าง Collection

```move
module learning::collection_creation {
    use std::string::{Self, String};
    use std::option;
    use std::signer;
    use aptos_framework::object::{Self, Object};
    use aptos_token_objects::collection::{Self, Collection, MutatorRef};
    use aptos_token_objects::royalty;
    
    // ============================================
    // Collection types
    // ============================================
    
    // Fixed supply collection
    public entry fun create_fixed_collection(
        creator: &signer,
        name: String,
        description: String,
        uri: String,
        max_supply: u64,
    ) {
        collection::create_fixed_collection(
            creator,
            description,
            max_supply,
            name,
            option::none(),  // no royalty
            uri,
        );
    }
    
    // Unlimited supply collection
    public entry fun create_unlimited_collection(
        creator: &signer,
        name: String,
        description: String,
        uri: String,
    ) {
        collection::create_unlimited_collection(
            creator,
            description,
            name,
            option::none(),  // no royalty
            uri,
        );
    }
    
    // Collection with royalty
    public entry fun create_collection_with_royalty(
        creator: &signer,
        name: String,
        description: String,
        uri: String,
        royalty_numerator: u64,    // e.g. 500 = 5%
        royalty_denominator: u64,  // e.g. 10000
        payee: address,
    ) {
        let royalty = royalty::create(royalty_numerator, royalty_denominator, payee);
        
        collection::create_unlimited_collection(
            creator,
            description,
            name,
            option::some(royalty),
            uri,
        );
    }
    
    // ============================================
    // Custom collection with extension
    // ============================================
    
    struct MyCollection has key {
        mutator_ref: collection::MutatorRef,
        mint_count: u64,
        whitelist: aptos_std::smart_table::SmartTable<address, bool>,
    }
    
    public entry fun create_custom_collection(
        creator: &signer,
        name: String,
        description: String,
        uri: String,
    ) {
        let constructor_ref = collection::create_unlimited_collection(
            creator,
            description,
            name,
            option::none(),
            uri,
        );
        
        let mutator_ref = collection::generate_mutator_ref(&constructor_ref);
        let obj_signer = object::generate_signer(&constructor_ref);
        
        move_to(&obj_signer, MyCollection {
            mutator_ref,
            mint_count: 0,
            whitelist: aptos_std::smart_table::new(),
        });
    }
    
    // Update collection description
    public entry fun update_description(
        creator: &signer,
        collection_addr: address,
        new_description: String,
    ) acquires MyCollection {
        let my_col = borrow_global<MyCollection>(collection_addr);
        collection::set_description(&my_col.mutator_ref, new_description);
    }
    
    // ============================================
    // View collection
    // ============================================
    
    #[view]
    public fun get_collection_name(collection: Object<Collection>): String {
        collection::name(collection)
    }
    
    #[view]
    public fun get_collection_supply(collection: Object<Collection>): option::Option<u64> {
        collection::count(collection)
    }
    
    #[view]
    public fun collection_address(creator: address, name: String): address {
        collection::create_collection_address(&creator, &name)
    }
}
```

---

## Mint NFT

```move
module learning::nft_minting {
    use std::string::{Self, String};
    use std::option;
    use std::signer;
    use aptos_framework::object::{Self, Object};
    use aptos_token_objects::collection;
    use aptos_token_objects::token::{Self, Token, MutatorRef, BurnRef};
    use aptos_framework::event;
    
    // ============================================
    // NFT with custom data
    // ============================================
    
    struct HeroNFT has key {
        class: u8,          // 1=Warrior, 2=Mage, 3=Rogue
        level: u64,
        experience: u64,
        power: u64,
        mutator_ref: MutatorRef,
        burn_ref: BurnRef,
    }
    
    struct MinterCap has key {
        collection_name: String,
        minted: u64,
    }
    
    #[event]
    struct HeroMinted has drop, store {
        token: address,
        owner: address,
        class: u8,
        power: u64,
    }
    
    const E_NOT_AUTHORIZED: u64 = 1;
    
    // ============================================
    // Initialize
    // ============================================
    
    public entry fun initialize(
        creator: &signer,
        collection_name: String,
        description: String,
        uri: String,
    ) {
        // Create collection first
        collection::create_unlimited_collection(
            creator,
            description,
            collection_name,
            option::none(),
            uri,
        );
        
        // Store minter capability
        move_to(creator, MinterCap {
            collection_name,
            minted: 0,
        });
    }
    
    // ============================================
    // Mint Hero NFT
    // ============================================
    
    public entry fun mint_hero(
        creator: &signer,
        recipient: address,
        name: String,
        description: String,
        uri: String,
        class: u8,
        power: u64,
    ) acquires MinterCap {
        let creator_addr = signer::address_of(creator);
        let cap = borrow_global_mut<MinterCap>(creator_addr);
        
        // Mint token (creates Object<Token>)
        let constructor_ref = token::create(
            creator,
            cap.collection_name,
            description,
            name,
            option::none(),  // no royalty override
            uri,
        );
        
        let token_signer = object::generate_signer(&constructor_ref);
        let mutator_ref = token::generate_mutator_ref(&constructor_ref);
        let burn_ref = token::generate_burn_ref(&constructor_ref);
        let transfer_ref = object::generate_transfer_ref(&constructor_ref);
        
        // Attach custom data
        move_to(&token_signer, HeroNFT {
            class,
            level: 1,
            experience: 0,
            power,
            mutator_ref,
            burn_ref,
        });
        
        // Transfer to recipient
        let linear_ref = object::generate_linear_transfer_ref(&transfer_ref);
        object::transfer_with_ref(linear_ref, recipient);
        
        let token_obj = object::object_from_constructor_ref::<Token>(&constructor_ref);
        
        cap.minted = cap.minted + 1;
        
        event::emit(HeroMinted {
            token: object::object_address(&token_obj),
            owner: recipient,
            class,
            power,
        });
    }
    
    // ============================================
    // Level up
    // ============================================
    
    public entry fun gain_experience(
        owner: &signer,
        token_obj: Object<Token>,
        exp: u64,
    ) acquires HeroNFT {
        assert!(object::is_owner(token_obj, signer::address_of(owner)), E_NOT_AUTHORIZED);
        
        let token_addr = object::object_address(&token_obj);
        let hero = borrow_global_mut<HeroNFT>(token_addr);
        
        hero.experience = hero.experience + exp;
        
        // Level up every 1000 XP
        let new_level = hero.experience / 1000 + 1;
        if (new_level > hero.level) {
            hero.level = new_level;
            hero.power = hero.power + (new_level - hero.level + 1) * 10;
        };
    }
    
    // ============================================
    // Update token URI (evolution)
    // ============================================
    
    public entry fun evolve(
        owner: &signer,
        token_obj: Object<Token>,
        new_uri: String,
    ) acquires HeroNFT {
        assert!(object::is_owner(token_obj, signer::address_of(owner)), E_NOT_AUTHORIZED);
        
        let token_addr = object::object_address(&token_obj);
        let hero = borrow_global<HeroNFT>(token_addr);
        
        // Update token URI via mutator
        token::set_uri(&hero.mutator_ref, new_uri);
    }
    
    // ============================================
    // Burn NFT
    // ============================================
    
    public entry fun burn_hero(
        owner: &signer,
        token_obj: Object<Token>,
    ) acquires HeroNFT {
        assert!(object::is_owner(token_obj, signer::address_of(owner)), E_NOT_AUTHORIZED);
        
        let token_addr = object::object_address(&token_obj);
        let HeroNFT {
            class: _, level: _, experience: _, power: _,
            mutator_ref: _,
            burn_ref,
        } = move_from<HeroNFT>(token_addr);
        
        token::burn(burn_ref);
    }
    
    // ============================================
    // View functions
    // ============================================
    
    #[view]
    public fun get_hero_stats(token_obj: Object<Token>): (u8, u64, u64, u64) acquires HeroNFT {
        let token_addr = object::object_address(&token_obj);
        let hero = borrow_global<HeroNFT>(token_addr);
        (hero.class, hero.level, hero.experience, hero.power)
    }
    
    #[view]
    public fun get_token_name(token_obj: Object<Token>): String {
        token::name(token_obj)
    }
    
    #[view]
    public fun get_token_uri(token_obj: Object<Token>): String {
        token::uri(token_obj)
    }
    
    #[view]
    public fun get_token_owner(token_obj: Object<Token>): address {
        object::owner(token_obj)
    }
}
```

---

## Token Properties

```move
module learning::token_properties {
    use std::string::{Self, String};
    use std::signer;
    use aptos_framework::object::{Self, Object};
    use aptos_token_objects::token::{Self, Token};
    use aptos_token_objects::property_map::{Self, PropertyMap};
    
    // ============================================
    // PropertyMap = on-chain key-value for tokens
    // ============================================
    
    // Supports types: bool, u8, u16, u32, u64, u128, u256, address, String, bytes
    
    struct NFTWithProps has key {
        property_mutator_ref: property_map::MutatorRef,
        token_mutator_ref: token::MutatorRef,
    }
    
    const E_NOT_OWNER: u64 = 1;
    
    public entry fun mint_with_properties(
        creator: &signer,
        collection_name: String,
        name: String,
        description: String,
        uri: String,
        // Properties
        rarity: String,
        attack: u64,
        defense: u64,
    ) {
        let constructor_ref = token::create(
            creator,
            collection_name,
            description,
            name,
            std::option::none(),
            uri,
        );
        
        // Initialize PropertyMap
        let properties = property_map::prepare_input(
            vector[
                string::utf8(b"rarity"),
                string::utf8(b"attack"),
                string::utf8(b"defense"),
            ],
            vector[
                string::utf8(b"String"),
                string::utf8(b"u64"),
                string::utf8(b"u64"),
            ],
            vector[
                bcs::to_bytes(&rarity),
                bcs::to_bytes(&attack),
                bcs::to_bytes(&defense),
            ],
        );
        property_map::init(&constructor_ref, properties);
        
        let property_mutator_ref = property_map::generate_mutator_ref(&constructor_ref);
        let token_mutator_ref = token::generate_mutator_ref(&constructor_ref);
        let obj_signer = object::generate_signer(&constructor_ref);
        
        move_to(&obj_signer, NFTWithProps {
            property_mutator_ref,
            token_mutator_ref,
        });
    }
    
    // ============================================
    // Update property
    // ============================================
    
    public entry fun upgrade_attack(
        owner: &signer,
        token_obj: Object<Token>,
        new_attack: u64,
    ) acquires NFTWithProps {
        assert!(object::is_owner(token_obj, signer::address_of(owner)), E_NOT_OWNER);
        
        let token_addr = object::object_address(&token_obj);
        let nft = borrow_global<NFTWithProps>(token_addr);
        
        property_map::update_typed<u64>(
            &nft.property_mutator_ref,
            &string::utf8(b"attack"),
            new_attack,
        );
    }
    
    // ============================================
    // Read properties
    // ============================================
    
    #[view]
    public fun get_attack(token_obj: Object<Token>): u64 {
        property_map::read_u64(&token_obj, &string::utf8(b"attack"))
    }
    
    #[view]
    public fun get_rarity(token_obj: Object<Token>): String {
        property_map::read_string(&token_obj, &string::utf8(b"rarity"))
    }
    
    #[view]
    public fun get_all_properties(token_obj: Object<Token>): PropertyMap {
        property_map::read(token_obj)
    }
    
    use aptos_framework::bcs;
}
```

---

## ตัวอย่าง: Gaming NFT System

```move
module learning::gaming_nft {
    use std::string::{Self, String};
    use std::option;
    use std::signer;
    use std::vector;
    use aptos_framework::object::{Self, Object};
    use aptos_framework::event;
    use aptos_token_objects::collection;
    use aptos_token_objects::token::{Self, Token};
    
    // ============================================
    // Game Types
    // ============================================
    
    struct Character has key {
        class: u8,         // 1=Warrior 2=Mage 3=Ranger
        level: u64,
        exp: u64,
        hp: u64,
        max_hp: u64,
        attack: u64,
        defense: u64,
        equipped_items: vector<address>,  // Object addresses
        token_mutator: token::MutatorRef,
        burn_ref: token::BurnRef,
    }
    
    struct Equipment has key {
        slot: u8,       // 1=Weapon 2=Armor 3=Accessory
        bonus_attack: u64,
        bonus_defense: u64,
        equipped_to: option::Option<address>,  // character address
        burn_ref: token::BurnRef,
    }
    
    struct GameConfig has key {
        admin: address,
        character_collection: String,
        equipment_collection: String,
        total_characters: u64,
        total_equipment: u64,
    }
    
    // ============================================
    // Events
    // ============================================
    
    #[event]
    struct CharacterMinted has drop, store {
        character: address,
        owner: address,
        class: u8,
    }
    
    #[event]
    struct ItemEquipped has drop, store {
        character: address,
        equipment: address,
        slot: u8,
    }
    
    #[event]
    struct LevelUp has drop, store {
        character: address,
        new_level: u64,
        new_attack: u64,
    }
    
    // ============================================
    // Errors
    // ============================================
    
    const E_NOT_ADMIN: u64 = 1;
    const E_NOT_OWNER: u64 = 2;
    const E_ALREADY_EQUIPPED: u64 = 3;
    const E_INVALID_CLASS: u64 = 4;
    
    // ============================================
    // Initialize game
    // ============================================
    
    public entry fun initialize_game(admin: &signer) {
        let admin_addr = signer::address_of(admin);
        
        let char_collection = string::utf8(b"Heroes of Aptos");
        let equip_collection = string::utf8(b"Legendary Equipment");
        
        // Create character collection
        collection::create_unlimited_collection(
            admin,
            string::utf8(b"Epic heroes for the blockchain"),
            char_collection,
            option::none(),
            string::utf8(b"https://game.example.com/heroes"),
        );
        
        // Create equipment collection
        collection::create_unlimited_collection(
            admin,
            string::utf8(b"Powerful equipment to enhance your heroes"),
            equip_collection,
            option::none(),
            string::utf8(b"https://game.example.com/equipment"),
        );
        
        move_to(admin, GameConfig {
            admin: admin_addr,
            character_collection: char_collection,
            equipment_collection: equip_collection,
            total_characters: 0,
            total_equipment: 0,
        });
    }
    
    // ============================================
    // Mint character
    // ============================================
    
    public entry fun mint_character(
        admin: &signer,
        recipient: address,
        name: String,
        class: u8,
    ) acquires GameConfig {
        let admin_addr = signer::address_of(admin);
        let config = borrow_global_mut<GameConfig>(admin_addr);
        assert!(config.admin == admin_addr, E_NOT_ADMIN);
        assert!(class >= 1 && class <= 3, E_INVALID_CLASS);
        
        config.total_characters = config.total_characters + 1;
        let token_id = config.total_characters;
        
        // Base stats by class
        let (hp, attack, defense) = class_base_stats(class);
        
        let description = class_description(class);
        let uri = build_character_uri(class, 1);
        
        let constructor_ref = token::create(
            admin,
            config.character_collection,
            description,
            name,
            option::none(),
            uri,
        );
        
        let token_signer = object::generate_signer(&constructor_ref);
        let transfer_ref = object::generate_transfer_ref(&constructor_ref);
        
        move_to(&token_signer, Character {
            class,
            level: 1,
            exp: 0,
            hp,
            max_hp: hp,
            attack,
            defense,
            equipped_items: vector::empty(),
            token_mutator: token::generate_mutator_ref(&constructor_ref),
            burn_ref: token::generate_burn_ref(&constructor_ref),
        });
        
        // Send to recipient
        let linear_ref = object::generate_linear_transfer_ref(&transfer_ref);
        object::transfer_with_ref(linear_ref, recipient);
        
        let token_obj = object::object_from_constructor_ref::<Token>(&constructor_ref);
        
        event::emit(CharacterMinted {
            character: object::object_address(&token_obj),
            owner: recipient,
            class,
        });
        let _ = token_id;
    }
    
    // ============================================
    // Gain experience and level up
    // ============================================
    
    public entry fun gain_exp(
        owner: &signer,
        character_obj: Object<Token>,
        exp_gained: u64,
    ) acquires Character {
        assert!(object::is_owner(character_obj, signer::address_of(owner)), E_NOT_OWNER);
        
        let char_addr = object::object_address(&character_obj);
        let char = borrow_global_mut<Character>(char_addr);
        
        char.exp = char.exp + exp_gained;
        
        // Level up formula: level = exp / (level * 100)
        let levels_gained = 0u64;
        loop {
            let exp_needed = char.level * 100;
            if (char.exp < exp_needed) break;
            char.exp = char.exp - exp_needed;
            char.level = char.level + 1;
            levels_gained = levels_gained + 1;
            
            // Stat increases
            char.attack = char.attack + char.class as u64 * 2 + 3;
            char.defense = char.defense + 2;
            let hp_increase = 20u64;
            char.max_hp = char.max_hp + hp_increase;
            char.hp = char.max_hp;  // full heal on level up
        };
        
        if (levels_gained > 0) {
            // Update token URI to reflect new level
            let new_uri = build_character_uri(char.class, char.level);
            token::set_uri(&char.token_mutator, new_uri);
            
            event::emit(LevelUp {
                character: char_addr,
                new_level: char.level,
                new_attack: char.attack,
            });
        };
    }
    
    // ============================================
    // Equipment system
    // ============================================
    
    public entry fun mint_equipment(
        admin: &signer,
        recipient: address,
        name: String,
        slot: u8,
        bonus_attack: u64,
        bonus_defense: u64,
    ) acquires GameConfig {
        let admin_addr = signer::address_of(admin);
        let config = borrow_global_mut<GameConfig>(admin_addr);
        assert!(config.admin == admin_addr, E_NOT_ADMIN);
        
        config.total_equipment = config.total_equipment + 1;
        
        let constructor_ref = token::create(
            admin,
            config.equipment_collection,
            string::utf8(b"Legendary equipment"),
            name,
            option::none(),
            string::utf8(b"https://game.example.com/items"),
        );
        
        let token_signer = object::generate_signer(&constructor_ref);
        let transfer_ref = object::generate_transfer_ref(&constructor_ref);
        
        move_to(&token_signer, Equipment {
            slot,
            bonus_attack,
            bonus_defense,
            equipped_to: option::none(),
            burn_ref: token::generate_burn_ref(&constructor_ref),
        });
        
        let linear_ref = object::generate_linear_transfer_ref(&transfer_ref);
        object::transfer_with_ref(linear_ref, recipient);
    }
    
    public entry fun equip_item(
        owner: &signer,
        character_obj: Object<Token>,
        equipment_obj: Object<Token>,
    ) acquires Character, Equipment {
        let owner_addr = signer::address_of(owner);
        assert!(object::is_owner(character_obj, owner_addr), E_NOT_OWNER);
        assert!(object::is_owner(equipment_obj, owner_addr), E_NOT_OWNER);
        
        let char_addr = object::object_address(&character_obj);
        let equip_addr = object::object_address(&equipment_obj);
        
        let equipment = borrow_global_mut<Equipment>(equip_addr);
        assert!(option::is_none(&equipment.equipped_to), E_ALREADY_EQUIPPED);
        
        // Apply bonuses
        let char = borrow_global_mut<Character>(char_addr);
        char.attack = char.attack + equipment.bonus_attack;
        char.defense = char.defense + equipment.bonus_defense;
        vector::push_back(&mut char.equipped_items, equip_addr);
        
        // Mark as equipped
        equipment.equipped_to = option::some(char_addr);
        
        event::emit(ItemEquipped {
            character: char_addr,
            equipment: equip_addr,
            slot: equipment.slot,
        });
    }
    
    // ============================================
    // View functions
    // ============================================
    
    #[view]
    public fun get_character_stats(character_obj: Object<Token>): (u8, u64, u64, u64, u64) 
    acquires Character {
        let char = borrow_global<Character>(object::object_address(&character_obj));
        (char.class, char.level, char.attack, char.defense, char.hp)
    }
    
    #[view]
    public fun get_equipment_info(equipment_obj: Object<Token>): (u8, u64, u64) 
    acquires Equipment {
        let equip = borrow_global<Equipment>(object::object_address(&equipment_obj));
        (equip.slot, equip.bonus_attack, equip.bonus_defense)
    }
    
    // ============================================
    // Helpers
    // ============================================
    
    fun class_base_stats(class: u8): (u64, u64, u64) {
        if (class == 1) (150, 20, 15)       // Warrior: high HP/DEF
        else if (class == 2) (80, 35, 5)    // Mage: high ATK
        else (100, 25, 10)                  // Ranger: balanced
    }
    
    fun class_description(class: u8): String {
        if (class == 1) string::utf8(b"A mighty warrior")
        else if (class == 2) string::utf8(b"A powerful mage")
        else string::utf8(b"A skilled ranger")
    }
    
    fun build_character_uri(class: u8, level: u64): String {
        let base = string::utf8(b"https://game.example.com/heroes/class");
        string::append_utf8(&mut base, vector[class + 48]);
        string::append_utf8(&mut base, b"/level");
        string::append_utf8(&mut base, level_to_bytes(level));
        string::append_utf8(&mut base, b".json");
        base
    }
    
    fun level_to_bytes(n: u64): vector<u8> {
        if (n == 0) return b"0";
        let result = vector::empty<u8>();
        let temp = n;
        while (temp > 0) {
            vector::push_back(&mut result, ((temp % 10) as u8) + 48);
            temp = temp / 10;
        };
        vector::reverse(&mut result);
        result
    }
}
```

---

## สรุป Digital Asset Standard

| Component | Purpose |
|-----------|---------|
| `Collection` | Group of related tokens |
| `Token` | Individual NFT |
| `MutatorRef` | Update name/description/URI |
| `BurnRef` | Burn the token |
| `PropertyMap` | On-chain key-value data |
| `Royalty` | Creator royalty on sales |

---

**ก่อนหน้า**: [Part 24 - Tables and SmartTable ←](part-24-tables-smarttable.md)
**ต่อไป**: [Part 26 - Sui Objects →](part-26-sui-objects.md)
