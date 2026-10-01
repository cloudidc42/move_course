# Part 55: NFT Infrastructure & Royalties

## สารบัญ
- [NFT Standards Comparison](#nft-standards-comparison)
- [Aptos Digital Asset Standard](#aptos-digital-asset-standard)
- [Sui Object-Based NFTs](#sui-object-based-nfts)
- [Royalty Enforcement Mechanisms](#royalty-enforcement-mechanisms)
- [NFT Marketplace Architecture](#nft-marketplace-architecture)
- [Advanced: Dynamic NFTs & Composable NFTs](#advanced-dynamic-nfts--composable-nfts)

---

## NFT Standards Comparison

```
NFT Standard Evolution:

Ethereum:
  ERC-721: Basic NFT (ownerOf, transfer, approve)
  ERC-1155: Multi-token (batch transfers, fungible+NFT)
  ERC-2981: Royalty standard (royaltyInfo function)
  
  Problem: Royalties not enforced at protocol level
  Opensea marketplace can bypass royalties
  "Creator Royalty" crisis 2022-2023

Solana:
  Metaplex: Creator royalty in metadata
  Still marketplace-dependent enforcement

Aptos:
  Token V1: Royalty in token config (old system)
  Digital Asset (Token V2): Modern composable NFTs
  - PropertyMap for flexible metadata
  - Soul-bound tokens
  - Built-in royalty with APT enforcement
  
Sui:
  Object model: NFT IS an object
  Kiosk system: Marketplace + royalty enforcement
  TransferPolicy: On-chain royalty rules
  
Key insight: Sui Kiosk = first on-chain enforced royalties
No marketplace can bypass Kiosk transfer policies!
```

---

## Aptos Digital Asset Standard

```move
module nft::aptos_nft {
    use aptos_framework::object::{Self, Object, ConstructorRef};
    use aptos_token_objects::collection;
    use aptos_token_objects::token;
    use aptos_token_objects::royalty;
    use aptos_token_objects::property_map;
    
    // ============================================
    // Collection (Group of NFTs)
    // ============================================
    
    struct GameCollection has key {
        collection_name: std::string::String,
        creator: address,
        max_supply: u64,
        minted: u64,
        base_uri: std::string::String,
        paused: bool,
    }
    
    struct CollectionAdmin has key {
        extend_ref: object::ExtendRef,
    }
    
    public entry fun create_collection(
        creator: &signer,
        name: std::string::String,
        description: std::string::String,
        uri: std::string::String,
        max_supply: u64,
        royalty_numerator: u64,
        royalty_denominator: u64,
    ) {
        let creator_addr = std::signer::address_of(creator);
        
        // Create royalty config
        let royalty = royalty::create(
            royalty_numerator,
            royalty_denominator,
            creator_addr,
        );
        
        // Create collection with fixed supply
        let constructor_ref = collection::create_fixed_collection(
            creator,
            description,
            max_supply,
            name,
            std::option::some(royalty),
            uri,
        );
        
        let collection_signer = object::generate_signer(&constructor_ref);
        let extend_ref = object::generate_extend_ref(&constructor_ref);
        
        move_to(&collection_signer, GameCollection {
            collection_name: name,
            creator: creator_addr,
            max_supply,
            minted: 0,
            base_uri: uri,
            paused: false,
        });
        
        move_to(&collection_signer, CollectionAdmin { extend_ref });
    }
    
    // ============================================
    // Minting NFTs
    // ============================================
    
    struct HeroNFT has key {
        // Game attributes
        power: u64,
        defense: u64,
        speed: u64,
        level: u64,
        experience: u64,
        class: std::string::String,
        
        // Mutability
        mutator_ref: token::MutatorRef,
        burn_ref: token::BurnRef,
        property_mutator_ref: property_map::MutatorRef,
        transfer_ref: object::TransferRef,
    }
    
    public entry fun mint_hero(
        creator: &signer,
        recipient: address,
        name: std::string::String,
        class: std::string::String,
        power: u64,
        defense: u64,
        speed: u64,
        collection_name: std::string::String,
    ) {
        let constructor_ref = token::create_named_token(
            creator,
            collection_name,
            std::string::utf8(b"A powerful hero"),
            name,
            std::option::none(),
            std::string::utf8(b"https://example.com/hero/1.json"),
        );
        
        let object_signer = object::generate_signer(&constructor_ref);
        let mutator_ref = token::generate_mutator_ref(&constructor_ref);
        let burn_ref = token::generate_burn_ref(&constructor_ref);
        let transfer_ref = object::generate_transfer_ref(&constructor_ref);
        let property_mutator_ref = property_map::generate_mutator_ref(&constructor_ref);
        
        // Initialize property map with game attributes
        let properties = property_map::prepare_input(
            vector[
                std::string::utf8(b"power"),
                std::string::utf8(b"defense"),
                std::string::utf8(b"speed"),
                std::string::utf8(b"class"),
                std::string::utf8(b"level"),
            ],
            vector[
                std::string::utf8(b"u64"),
                std::string::utf8(b"u64"),
                std::string::utf8(b"u64"),
                std::string::utf8(b"0x1::string::String"),
                std::string::utf8(b"u64"),
            ],
            vector[
                bcs::to_bytes(&power),
                bcs::to_bytes(&defense),
                bcs::to_bytes(&speed),
                bcs::to_bytes(&class),
                bcs::to_bytes(&1u64),
            ],
        );
        
        property_map::init(&constructor_ref, properties);
        
        move_to(&object_signer, HeroNFT {
            power,
            defense,
            speed,
            level: 1,
            experience: 0,
            class,
            mutator_ref,
            burn_ref,
            property_mutator_ref,
            transfer_ref,
        });
        
        // Transfer to recipient
        let linear_transfer = object::generate_linear_transfer_ref(&transfer_ref);
        object::transfer_with_ref(linear_transfer, recipient);
    }
    
    // ============================================
    // Dynamic NFT: Level Up
    // ============================================
    
    public entry fun gain_experience(
        owner: &signer,
        nft_obj: Object<HeroNFT>,
        xp_gained: u64,
    ) acquires HeroNFT {
        let owner_addr = std::signer::address_of(owner);
        assert!(object::is_owner(nft_obj, owner_addr), 1);
        
        let nft_addr = object::object_address(&nft_obj);
        let hero = borrow_global_mut<HeroNFT>(nft_addr);
        
        hero.experience = hero.experience + xp_gained;
        
        // Level up every 1000 XP
        let new_level = (hero.experience / 1000) + 1;
        if (new_level > hero.level) {
            hero.level = new_level;
            hero.power = hero.power + 10;
            hero.defense = hero.defense + 5;
            
            // Update property map to reflect new stats (on-chain metadata)
            property_map::update_typed<u64>(
                &hero.property_mutator_ref,
                &std::string::utf8(b"level"),
                new_level,
            );
            property_map::update_typed<u64>(
                &hero.property_mutator_ref,
                &std::string::utf8(b"power"),
                hero.power,
            );
        };
    }
    
    // ============================================
    // Soul-Bound Token (Non-Transferable)
    // ============================================
    
    struct AchievementBadge has key {
        achievement_type: std::string::String,
        earned_at: u64,
        // NO transfer_ref = soul-bound
    }
    
    public entry fun award_achievement(
        creator: &signer,
        recipient: address,
        achievement_type: std::string::String,
        collection_name: std::string::String,
    ) {
        let constructor_ref = token::create_named_token(
            creator,
            collection_name,
            std::string::utf8(b"Achievement Badge"),
            achievement_type,
            std::option::none(),
            std::string::utf8(b"https://example.com/badge.json"),
        );
        
        // Disable transfer → soul-bound
        object::disable_ungated_transfer(&object::generate_transfer_ref(&constructor_ref));
        
        let object_signer = object::generate_signer(&constructor_ref);
        
        move_to(&object_signer, AchievementBadge {
            achievement_type,
            earned_at: aptos_framework::timestamp::now_seconds(),
        });
        
        // Transfer to recipient at mint
        let object = object::object_from_constructor_ref::<AchievementBadge>(&constructor_ref);
        object::transfer(creator, object, recipient);
    }
    
    // ============================================
    // Composable NFTs (Equip items)
    // ============================================
    
    struct Equipment has key {
        slot: std::string::String,  // "weapon", "armor", "ring"
        bonus_power: u64,
        bonus_defense: u64,
        equipped_to: std::option::Option<address>,  // Address of hero object
    }
    
    // Equip item to hero (object-owns-object)
    public entry fun equip_item(
        owner: &signer,
        hero_obj: Object<HeroNFT>,
        item_obj: Object<Equipment>,
    ) acquires HeroNFT, Equipment {
        let owner_addr = std::signer::address_of(owner);
        assert!(object::is_owner(hero_obj, owner_addr), 1);
        assert!(object::is_owner(item_obj, owner_addr), 2);
        
        let hero_addr = object::object_address(&hero_obj);
        let item_addr = object::object_address(&item_obj);
        
        let equipment = borrow_global_mut<Equipment>(item_addr);
        assert!(std::option::is_none(&equipment.equipped_to), 3);
        equipment.equipped_to = std::option::some(hero_addr);
        
        // Apply bonuses
        let hero = borrow_global_mut<HeroNFT>(hero_addr);
        hero.power = hero.power + equipment.bonus_power;
        hero.defense = hero.defense + equipment.bonus_defense;
        
        // Transfer item object to hero object (composability!)
        object::transfer(owner, item_obj, hero_addr);
    }
    
    public entry fun unequip_item(
        owner: &signer,
        hero_obj: Object<HeroNFT>,
        item_obj: Object<Equipment>,
    ) acquires HeroNFT, Equipment {
        let owner_addr = std::signer::address_of(owner);
        assert!(object::is_owner(hero_obj, owner_addr), 1);
        
        let hero_addr = object::object_address(&hero_obj);
        let item_addr = object::object_address(&item_obj);
        
        let equipment = borrow_global_mut<Equipment>(item_addr);
        equipment.equipped_to = std::option::none();
        
        let hero = borrow_global_mut<HeroNFT>(hero_addr);
        hero.power = hero.power - equipment.bonus_power;
        hero.defense = hero.defense - equipment.bonus_defense;
        
        // Transfer back to owner
        object::transfer_raw(owner, item_addr, owner_addr);
    }
}
```

---

## Sui Object-Based NFTs

```move
module nft::sui_nft {
    use sui::object::{Self, UID, ID};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};
    use sui::event;
    use sui::package;
    use sui::display;
    use std::string::{Self, String};
    
    // ============================================
    // One-Time Witness pattern for Display
    // ============================================
    
    struct GAME_NFT has drop {}
    
    // Called once at publish time
    fun init(witness: GAME_NFT, ctx: &mut TxContext) {
        let publisher = package::claim(witness, ctx);
        
        // Set up Display (on-chain metadata for wallets/marketplaces)
        let keys = vector[
            string::utf8(b"name"),
            string::utf8(b"description"),
            string::utf8(b"image_url"),
            string::utf8(b"attributes"),
        ];
        
        let values = vector[
            string::utf8(b"{name}"),
            string::utf8(b"A Hero NFT with level {level}"),
            string::utf8(b"https://game.example.com/nft/{id}/image.png"),
            string::utf8(b"Power: {power}, Defense: {defense}"),
        ];
        
        let mut d = display::new_with_fields<HeroNFT>(&publisher, keys, values, ctx);
        display::update_version(&mut d);
        
        transfer::public_transfer(publisher, tx_context::sender(ctx));
        transfer::public_transfer(d, tx_context::sender(ctx));
    }
    
    // ============================================
    // NFT structs
    // ============================================
    
    struct HeroNFT has key, store {
        id: UID,
        name: String,
        class: String,
        power: u64,
        defense: u64,
        speed: u64,
        level: u64,
        experience: u64,
    }
    
    struct Equipment has key, store {
        id: UID,
        name: String,
        slot: String,
        bonus_power: u64,
        bonus_defense: u64,
    }
    
    // ============================================
    // Minting
    // ============================================
    
    public entry fun mint_hero(
        name: String,
        class: String,
        power: u64,
        defense: u64,
        speed: u64,
        recipient: address,
        ctx: &mut TxContext,
    ) {
        let hero = HeroNFT {
            id: object::new(ctx),
            name,
            class,
            power,
            defense,
            speed,
            level: 1,
            experience: 0,
        };
        
        event::emit(HeroMinted {
            id: object::id(&hero),
            name: hero.name,
            class: hero.class,
        });
        
        transfer::transfer(hero, recipient);
    }
    
    struct HeroMinted has copy, drop {
        id: ID,
        name: String,
        class: String,
    }
    
    // ============================================
    // Level Up (requires ownership)
    // ============================================
    
    public entry fun level_up(
        hero: &mut HeroNFT,
        xp: u64,
        ctx: &TxContext,
    ) {
        hero.experience = hero.experience + xp;
        let new_level = hero.experience / 1000 + 1;
        if (new_level > hero.level) {
            hero.level = new_level;
            hero.power = hero.power + 10;
            hero.defense = hero.defense + 5;
        };
    }
    
    // ============================================
    // Composable NFTs via wrapping
    // ============================================
    
    struct EquippedHero has key {
        id: UID,
        hero: HeroNFT,
        weapon: std::option::Option<Equipment>,
        armor: std::option::Option<Equipment>,
    }
    
    public entry fun create_equipped_hero(
        hero: HeroNFT,
        ctx: &mut TxContext,
    ) {
        let equipped = EquippedHero {
            id: object::new(ctx),
            hero,
            weapon: std::option::none(),
            armor: std::option::none(),
        };
        transfer::transfer(equipped, tx_context::sender(ctx));
    }
    
    public entry fun equip_weapon(
        hero: &mut EquippedHero,
        weapon: Equipment,
        ctx: &TxContext,
    ) {
        assert!(std::option::is_none(&hero.weapon), 1);
        hero.hero.power = hero.hero.power + weapon.bonus_power;
        std::option::fill(&mut hero.weapon, weapon);
    }
    
    public entry fun unequip_weapon(
        hero: &mut EquippedHero,
        ctx: &mut TxContext,
    ) {
        assert!(std::option::is_some(&hero.weapon), 1);
        let weapon = std::option::extract(&mut hero.weapon);
        hero.hero.power = hero.hero.power - weapon.bonus_power;
        transfer::transfer(weapon, tx_context::sender(ctx));
    }
}
```

---

## Royalty Enforcement Mechanisms

```move
module nft::royalty_enforced {
    use sui::object::{Self, UID, ID};
    use sui::transfer_policy::{Self, TransferPolicy, TransferPolicyCap, TransferRequest};
    use sui::kiosk::{Self, Kiosk, KioskOwnerCap};
    use sui::coin::{Self, Coin};
    use sui::sui::SUI;
    use sui::tx_context::{Self, TxContext};
    use sui::package;
    
    // ============================================
    // Sui Kiosk: On-chain royalty enforcement
    // ============================================
    
    // TransferPolicy = rules that MUST be satisfied to complete a transfer
    // No marketplace can bypass this!
    
    struct RoyaltyRule has drop {}
    
    // Creator sets up royalty in TransferPolicy
    public fun setup_royalty_policy(
        policy: &mut TransferPolicy<HeroNFT>,
        policy_cap: &TransferPolicyCap<HeroNFT>,
        royalty_bps: u64,  // e.g., 500 = 5%
    ) {
        transfer_policy::add_rule(
            RoyaltyRule {},
            policy,
            policy_cap,
            royalty_bps,
        );
    }
    
    struct HeroNFT has key, store {
        id: UID,
        name: std::string::String,
        level: u64,
    }
    
    // Prove royalty was paid → complete transfer
    public fun pay_royalty_and_transfer(
        policy: &mut TransferPolicy<HeroNFT>,
        request: TransferRequest<HeroNFT>,
        payment: Coin<SUI>,
        ctx: &mut TxContext,
    ) {
        let royalty_bps = transfer_policy::get_rule<HeroNFT, RoyaltyRule, u64>(
            RoyaltyRule {},
            policy,
        );
        
        let item_price = transfer_policy::paid(policy, &request);
        let required_royalty = item_price * royalty_bps / 10_000;
        
        assert!(coin::value(&payment) >= required_royalty, 1);
        
        // Pay royalty to policy owner (creator)
        transfer_policy::add_to_balance(policy, payment);
        
        // Confirm rule satisfied
        transfer_policy::confirm_request(RoyaltyRule {}, policy, request);
    }
    
    // Withdraw accumulated royalties (creator only)
    public entry fun withdraw_royalties(
        policy: &mut TransferPolicy<HeroNFT>,
        cap: &TransferPolicyCap<HeroNFT>,
        ctx: &mut TxContext,
    ) {
        let amount = transfer_policy::revenue_value(policy);
        let revenue = transfer_policy::withdraw<HeroNFT, SUI>(policy, cap, std::option::some(amount), ctx);
        sui::transfer::public_transfer(revenue, tx_context::sender(ctx));
    }
}
```

---

## NFT Marketplace Architecture

```move
module nft::marketplace {
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::object::{Self, Object};
    use aptos_framework::coin;
    use aptos_framework::aptos_coin::AptosCoin;
    
    // ============================================
    // P2P NFT Marketplace (Aptos)
    // ============================================
    
    struct Marketplace has key {
        // Active listings: object_address → Listing
        listings: SmartTable<address, Listing>,
        
        // Protocol fee (bps)
        protocol_fee_bps: u64,
        protocol_treasury: address,
        
        // Volume stats
        total_volume: u64,
        total_sales: u64,
    }
    
    struct Listing has store, drop {
        seller: address,
        nft_address: address,
        price: u64,
        listed_at: u64,
        expiry: u64,
    }
    
    struct Offer has key {
        buyer: address,
        nft_address: address,
        offer_amount: u64,
        expiry: u64,
        escrowed_payment: coin::Coin<AptosCoin>,
    }
    
    // ============================================
    // List NFT for sale
    // ============================================
    
    public entry fun list_nft<T: key>(
        seller: &signer,
        marketplace_addr: address,
        nft_obj: Object<T>,
        price: u64,
        duration_secs: u64,
    ) acquires Marketplace {
        let seller_addr = std::signer::address_of(seller);
        assert!(object::is_owner(nft_obj, seller_addr), 1);
        
        let nft_address = object::object_address(&nft_obj);
        let marketplace = borrow_global_mut<Marketplace>(marketplace_addr);
        
        assert!(!smart_table::contains(&marketplace.listings, nft_address), 2);
        
        let now = aptos_framework::timestamp::now_seconds();
        
        // Transfer NFT to marketplace escrow
        object::transfer(seller, nft_obj, marketplace_addr);
        
        smart_table::add(&mut marketplace.listings, nft_address, Listing {
            seller: seller_addr,
            nft_address,
            price,
            listed_at: now,
            expiry: now + duration_secs,
        });
    }
    
    // ============================================
    // Buy NFT
    // ============================================
    
    public entry fun buy_nft<T: key>(
        buyer: &signer,
        marketplace_addr: address,
        nft_address: address,
        max_price: u64,  // Slippage protection
    ) acquires Marketplace {
        let buyer_addr = std::signer::address_of(buyer);
        let marketplace = borrow_global_mut<Marketplace>(marketplace_addr);
        
        assert!(smart_table::contains(&marketplace.listings, nft_address), 1);
        let listing = smart_table::remove(&mut marketplace.listings, nft_address);
        
        assert!(listing.price <= max_price, 2);
        assert!(aptos_framework::timestamp::now_seconds() < listing.expiry, 3);
        
        let price = listing.price;
        let protocol_fee = price * marketplace.protocol_fee_bps / 10_000;
        let seller_amount = price - protocol_fee;
        
        // Collect payment
        let payment = coin::withdraw<AptosCoin>(buyer, price);
        let protocol_payment = coin::extract(&mut payment, protocol_fee);
        
        // Pay seller
        coin::deposit(listing.seller, payment);
        // Pay protocol
        coin::deposit(marketplace.protocol_treasury, protocol_payment);
        
        // Transfer NFT to buyer (from marketplace escrow)
        // Note: In production, marketplace needs ExtendRef to sign
        // Simplified: assume marketplace has transfer capability
        
        marketplace.total_volume = marketplace.total_volume + price;
        marketplace.total_sales = marketplace.total_sales + 1;
    }
    
    // ============================================
    // Offer system (buyer makes offer, seller accepts)
    // ============================================
    
    public entry fun make_offer(
        buyer: &signer,
        nft_address: address,
        offer_amount: u64,
        duration_secs: u64,
    ) {
        let buyer_addr = std::signer::address_of(buyer);
        
        // Escrow payment
        let escrowed_payment = coin::withdraw<AptosCoin>(buyer, offer_amount);
        
        move_to(buyer, Offer {
            buyer: buyer_addr,
            nft_address,
            offer_amount,
            expiry: aptos_framework::timestamp::now_seconds() + duration_secs,
            escrowed_payment,
        });
    }
    
    public entry fun accept_offer<T: key>(
        seller: &signer,
        nft_obj: Object<T>,
        buyer_addr: address,
        marketplace_addr: address,
    ) acquires Offer, Marketplace {
        let seller_addr = std::signer::address_of(seller);
        assert!(object::is_owner(nft_obj, seller_addr), 1);
        
        let Offer {
            buyer,
            nft_address,
            offer_amount,
            expiry,
            escrowed_payment,
        } = move_from<Offer>(buyer_addr);
        
        assert!(buyer == buyer_addr, 2);
        assert!(aptos_framework::timestamp::now_seconds() < expiry, 3);
        
        let marketplace = borrow_global_mut<Marketplace>(marketplace_addr);
        let protocol_fee = offer_amount * marketplace.protocol_fee_bps / 10_000;
        let mut payment = escrowed_payment;
        let protocol_payment = coin::extract(&mut payment, protocol_fee);
        
        coin::deposit(seller_addr, payment);
        coin::deposit(marketplace.protocol_treasury, protocol_payment);
        
        // Transfer NFT to buyer
        object::transfer(seller, nft_obj, buyer_addr);
        
        marketplace.total_volume = marketplace.total_volume + offer_amount;
    }
    
    public entry fun cancel_offer(
        buyer: &signer,
        nft_address: address,
    ) acquires Offer {
        let buyer_addr = std::signer::address_of(buyer);
        let Offer { buyer: _, nft_address: _, offer_amount: _, expiry: _, escrowed_payment } =
            move_from<Offer>(buyer_addr);
        
        coin::deposit(buyer_addr, escrowed_payment);
    }
}
```

---

## Advanced: Dynamic NFTs & Composable NFTs

```move
module nft::dynamic_nft {
    use aptos_framework::object::{Self, Object, ConstructorRef};
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_token_objects::token;
    
    // ============================================
    // Dynamic NFT: Stats change based on game events
    // ============================================
    
    struct LivingNFT has key {
        // Core immutable identity
        token_id: u64,
        birth_timestamp: u64,
        
        // Dynamic attributes (change over time)
        health: u64,
        max_health: u64,
        hunger: u64,      // 0-100 (0=starving, 100=full)
        happiness: u64,   // 0-100
        
        // Time-decay mechanics
        last_fed: u64,
        last_played: u64,
        
        // Evolution state
        stage: u8,          // 0=egg, 1=baby, 2=adult, 3=elder
        stage_xp: u64,
        stage_thresholds: vector<u64>,  // XP needed for each stage
        
        // Appearance URI (updates with stage)
        current_uri: std::string::String,
        
        // Refs for mutation
        mutator_ref: token::MutatorRef,
    }
    
    // Feed the NFT (increases hunger + happiness)
    public entry fun feed(
        owner: &signer,
        nft_obj: Object<LivingNFT>,
        food_quality: u64,  // 1-10
    ) acquires LivingNFT {
        let owner_addr = std::signer::address_of(owner);
        assert!(object::is_owner(nft_obj, owner_addr), 1);
        
        let nft = borrow_global_mut<LivingNFT>(object::object_address(&nft_obj));
        let now = aptos_framework::timestamp::now_seconds();
        
        // Decay hunger since last feed
        let time_since_feed = now - nft.last_fed;
        let hunger_decay = time_since_feed / 3600 * 5;  // -5 hunger per hour
        if (hunger_decay >= nft.hunger) {
            nft.hunger = 0;
        } else {
            nft.hunger = nft.hunger - hunger_decay;
        };
        
        // Feed effect
        nft.hunger = std::u64::min(100, nft.hunger + food_quality * 10);
        nft.happiness = std::u64::min(100, nft.happiness + food_quality * 2);
        nft.last_fed = now;
        nft.stage_xp = nft.stage_xp + food_quality;
        
        // Check evolution
        check_evolution(nft);
    }
    
    fun check_evolution(nft: &mut LivingNFT) {
        if ((nft.stage as u64) < std::vector::length(&nft.stage_thresholds)) {
            let threshold = *std::vector::borrow(&nft.stage_thresholds, nft.stage as u64);
            if (nft.stage_xp >= threshold) {
                nft.stage = nft.stage + 1;
                
                // Update metadata URI for new appearance
                let new_uri = get_stage_uri(nft.token_id, nft.stage);
                token::set_uri(&nft.mutator_ref, new_uri);
                
                // Boost stats on evolution
                nft.max_health = nft.max_health + 50;
                nft.health = nft.max_health;
            }
        }
    }
    
    fun get_stage_uri(token_id: u64, stage: u8): std::string::String {
        // Return URI based on stage
        std::string::utf8(b"https://game.example.com/nft/stage/")
    }
}
```

---

## สรุป NFT Infrastructure

```
Key Takeaways:

Aptos NFT Best Practices:
  ✓ Use Digital Asset standard (not Token V1)
  ✓ Store refs (MutatorRef, BurnRef, TransferRef) for future updates
  ✓ PropertyMap for flexible on-chain attributes
  ✓ Soul-bound = disable_ungated_transfer
  ✓ Composable = transfer object to another object's address

Sui NFT Best Practices:
  ✓ One-Time Witness + Display for wallet/marketplace support  
  ✓ Kiosk system for royalty enforcement
  ✓ TransferPolicy for custom transfer rules
  ✓ Wrapping for composability
  ✓ Dynamic fields for expandable attributes

Royalty Enforcement:
  Aptos: Built into collection, enforced by framework (not 100% guaranteed)
  Sui: TransferPolicy + Kiosk = FIRST on-chain enforced royalties
  
Marketplace Architecture:
  - Listing → escrow NFT in marketplace
  - Offer system → escrow payment
  - Protocol fee split from each sale
  - Volume tracking for analytics

Dynamic NFTs:
  - MutatorRef allows post-mint changes
  - Time-decay mechanics (hunger, happiness)
  - Evolution stages change appearance URI
  - On-chain metadata = verifiable stats
  
Production Checklist:
  □ Collection supply cap
  □ Metadata IPFS pinning
  □ Royalty rate defined before launch
  □ Reveal mechanism (if blind mint)
  □ Upgrade path (object extend_ref)
  □ Emergency pause
```

---

**ก่อนหน้า**: [Part 54 - Cross-Chain DeFi ←](part-54-crosschain-defi.md)
**ต่อไป**: [Part 56 - On-Chain Randomness & VRF →](part-56-randomness-vrf.md)
