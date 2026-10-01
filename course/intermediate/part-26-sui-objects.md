# Part 26: Sui Object Model

## สารบัญ
- [Sui Object Model Overview](#sui-object-model-overview)
- [Object Types](#object-types)
- [Creating Objects](#creating-objects)
- [Transfer and Share](#transfer-and-share)
- [Dynamic Fields](#dynamic-fields)
- [ตัวอย่าง: Sui NFT Marketplace](#ตัวอย่าง-sui-nft-marketplace)

---

## Sui Object Model Overview

```
Aptos Model                     Sui Model
────────────────────────        ────────────────────────────
Global storage by address        Every value is an Object
  @0x1 → Resource A             Object has UID
  @0x1 → Resource B               └── unique ID
                                    └── owner info
Resources = key-value store         └── your data fields

Move To / Move From              Transfer / Share / Wrap
```

Sui ต่างจาก Aptos ตรงที่:
- ไม่มี global storage (`move_to`, `borrow_global`)
- ทุก "resource" เป็น Object ที่มี `UID`
- Object ถูก pass ผ่าน function parameters
- Ownership ติดกับ Object เอง

---

## Object Types

```move
module game::object_types {
    use sui::object::{Self, UID};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};
    
    // ============================================
    // Owned Object
    // ============================================
    
    // Most common: owned by one address
    struct Sword has key, store {
        id: UID,
        damage: u64,
        durability: u64,
    }
    
    // ============================================
    // Shared Object
    // ============================================
    
    // Accessible by anyone, but requires consensus
    struct GameConfig has key {
        id: UID,
        admin: address,
        paused: bool,
        fee_bps: u64,
    }
    
    // ============================================
    // Frozen Object (immutable)
    // ============================================
    
    struct TokenMetadata has key {
        id: UID,
        name: std::string::String,
        symbol: std::string::String,
        decimals: u8,
    }
    
    // ============================================
    // Wrapped Object
    // ============================================
    
    // Object stored inside another object
    struct Inventory has key {
        id: UID,
        owner: address,
        sword: Sword,  // wrapped!
    }
    
    // ============================================
    // Object without key (not an object, just a value)
    // ============================================
    
    struct Stats has copy, drop, store {
        strength: u64,
        agility: u64,
        intelligence: u64,
    }
}
```

---

## Creating Objects

```move
module game::create_objects {
    use sui::object::{Self, UID};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};
    use std::string::String;
    
    // ============================================
    // Pattern: Create and Transfer
    // ============================================
    
    struct NFT has key, store {
        id: UID,
        name: String,
        uri: String,
        creator: address,
    }
    
    struct AdminCap has key {
        id: UID,
    }
    
    // Create owned object
    public entry fun mint_nft(
        name: String,
        uri: String,
        ctx: &mut TxContext,
    ) {
        let creator = tx_context::sender(ctx);
        
        let nft = NFT {
            id: object::new(ctx),  // generate unique ID
            name,
            uri,
            creator,
        };
        
        // Transfer to sender (or anyone)
        transfer::transfer(nft, creator);
    }
    
    // ============================================
    // Entry functions with objects
    // ============================================
    
    // Pass object by value (consumes it)
    public entry fun burn_nft(nft: NFT) {
        let NFT { id, name: _, uri: _, creator: _ } = nft;
        object::delete(id);  // must delete UID when consuming object
    }
    
    // Pass object by mutable reference (modify)
    public entry fun update_uri(
        nft: &mut NFT,
        new_uri: String,
        ctx: &TxContext,
    ) {
        // Only creator can update
        assert!(nft.creator == tx_context::sender(ctx), 0);
        nft.uri = new_uri;
    }
    
    // Pass object by reference (read only)
    public fun get_creator(nft: &NFT): address {
        nft.creator
    }
    
    // ============================================
    // Initialize (like Aptos's init_module)
    // ============================================
    
    fun init(ctx: &mut TxContext) {
        // Called once when module is published
        let admin_cap = AdminCap {
            id: object::new(ctx),
        };
        transfer::transfer(admin_cap, tx_context::sender(ctx));
    }
    
    // ============================================
    // Multiple return values
    // ============================================
    
    struct Chest has key {
        id: UID,
        gold: u64,
    }
    
    struct Key has key, store {
        id: UID,
        chest_id: address,
    }
    
    // Create chest and key together
    public entry fun create_locked_chest(
        gold: u64,
        ctx: &mut TxContext,
    ) {
        let chest_id = object::new(ctx);
        let chest_addr = object::uid_to_address(&chest_id);
        
        let chest = Chest { id: chest_id, gold };
        let key = Key {
            id: object::new(ctx),
            chest_id: chest_addr,
        };
        
        let sender = tx_context::sender(ctx);
        transfer::transfer(chest, sender);
        transfer::transfer(key, sender);
    }
    
    public entry fun open_chest(
        chest: Chest,
        key: Key,
        ctx: &mut TxContext,
    ) {
        let chest_addr = object::uid_to_address(&chest.id);
        assert!(key.chest_id == chest_addr, 1);
        
        // Destroy key
        let Key { id: key_id, chest_id: _ } = key;
        object::delete(key_id);
        
        // Extract gold and destroy chest
        let Chest { id: chest_id, gold } = chest;
        object::delete(chest_id);
        
        // Gold is a u64 so just use it
        // In practice, would mint coin or do something with gold
        let _ = gold;
    }
}
```

---

## Transfer and Share

```move
module game::transfer_share {
    use sui::object::{Self, UID};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};
    
    struct Resource has key, store {
        id: UID,
        value: u64,
    }
    
    struct SharedPool has key {
        id: UID,
        total: u64,
    }
    
    struct FrozenConfig has key {
        id: UID,
        max_value: u64,
    }
    
    // ============================================
    // transfer::transfer - give to one address
    // ============================================
    
    public entry fun give_to(
        resource: Resource,
        recipient: address,
    ) {
        transfer::transfer(resource, recipient);
    }
    
    // ============================================
    // transfer::public_transfer - for objects with `store`
    // ============================================
    
    public entry fun public_give(
        resource: Resource,
        recipient: address,
    ) {
        // public_transfer works when object has both key + store
        transfer::public_transfer(resource, recipient);
    }
    
    // ============================================
    // transfer::share_object - anyone can access
    // ============================================
    
    public entry fun create_shared_pool(ctx: &mut TxContext) {
        let pool = SharedPool {
            id: object::new(ctx),
            total: 0,
        };
        transfer::share_object(pool);
        // After this, anyone can pass &mut SharedPool in their txn
    }
    
    // Using shared object: passed as mut reference
    public entry fun add_to_pool(
        pool: &mut SharedPool,
        amount: u64,
    ) {
        pool.total = pool.total + amount;
    }
    
    // ============================================
    // transfer::freeze_object - immutable forever
    // ============================================
    
    public entry fun publish_config(
        max_value: u64,
        ctx: &mut TxContext,
    ) {
        let config = FrozenConfig {
            id: object::new(ctx),
            max_value,
        };
        transfer::freeze_object(config);
        // After this, config can only be used as &FrozenConfig (immutable ref)
    }
    
    // Using frozen object: passed as immutable reference
    public fun check_value(config: &FrozenConfig, value: u64): bool {
        value <= config.max_value
    }
    
    // ============================================
    // Returning objects
    // ============================================
    
    // Can also return object (PTB will handle it)
    public fun create_resource(
        value: u64,
        ctx: &mut TxContext,
    ): Resource {
        Resource { id: object::new(ctx), value }
    }
    
    // In Programmable Transaction Block (PTB):
    // let res = create_resource(100);
    // give_to(res, recipient);
}
```

---

## Dynamic Fields

```move
module game::dynamic_fields {
    use sui::object::{Self, UID};
    use sui::dynamic_field;
    use sui::dynamic_object_field;
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};
    use std::string::String;
    
    // ============================================
    // Dynamic Fields = extensible storage
    // ============================================
    
    // dynamic_field::add/borrow/borrow_mut/remove
    //   - Value is stored inline (not as object)
    //   - Value doesn't need key ability
    
    // dynamic_object_field::add/borrow/borrow_mut/remove
    //   - Value is stored as child object
    //   - Value must have key ability
    //   - Can be transferred/shared independently
    
    struct Player has key {
        id: UID,
        name: String,
        level: u64,
        // No fixed fields for items - use dynamic fields!
    }
    
    struct Item has key, store {
        id: UID,
        name: String,
        power: u64,
    }
    
    // ============================================
    // Dynamic field (value type)
    // ============================================
    
    // Add stat as dynamic field
    public entry fun add_stat(
        player: &mut Player,
        stat_name: String,
        value: u64,
    ) {
        dynamic_field::add(&mut player.id, stat_name, value);
    }
    
    public fun get_stat(player: &Player, stat_name: String): u64 {
        *dynamic_field::borrow<String, u64>(&player.id, stat_name)
    }
    
    public entry fun update_stat(
        player: &mut Player,
        stat_name: String,
        value: u64,
    ) {
        *dynamic_field::borrow_mut<String, u64>(&mut player.id, stat_name) = value;
    }
    
    public fun stat_exists(player: &Player, stat_name: String): bool {
        dynamic_field::exists_with_type<String, u64>(&player.id, stat_name)
    }
    
    // ============================================
    // Dynamic object field (object type)
    // ============================================
    
    // Equip item to player (object becomes child of player)
    public entry fun equip_item(
        player: &mut Player,
        item: Item,
        slot: String,
    ) {
        // Item is now a child object of player
        dynamic_object_field::add(&mut player.id, slot, item);
    }
    
    public fun borrow_item(player: &Player, slot: String): &Item {
        dynamic_object_field::borrow<String, Item>(&player.id, slot)
    }
    
    // Unequip: remove from player and give back
    public entry fun unequip_item(
        player: &mut Player,
        slot: String,
        ctx: &mut TxContext,
    ) {
        let item = dynamic_object_field::remove<String, Item>(&mut player.id, slot);
        transfer::transfer(item, tx_context::sender(ctx));
    }
    
    // ============================================
    // Bag pattern (heterogeneous storage)
    // ============================================
    
    // Can add different types with different keys
    public entry fun add_various(player: &mut Player) {
        // Different key types for different value types
        dynamic_field::add(&mut player.id, b"strength", 100u64);
        dynamic_field::add(&mut player.id, b"is_vip", true);
        dynamic_field::add(&mut player.id, b"guild", b"Dragons");
    }
    
    // ============================================
    // Create player
    // ============================================
    
    public entry fun create_player(name: String, ctx: &mut TxContext) {
        let player = Player {
            id: object::new(ctx),
            name,
            level: 1,
        };
        transfer::transfer(player, tx_context::sender(ctx));
    }
}
```

---

## ตัวอย่าง: Sui NFT Marketplace

```move
module marketplace::listing {
    use sui::object::{Self, UID, ID};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};
    use sui::coin::{Self, Coin};
    use sui::sui::SUI;
    use sui::dynamic_object_field;
    use sui::event;
    use std::option::{Self, Option};
    
    // ============================================
    // Types
    // ============================================
    
    struct Marketplace has key {
        id: UID,
        admin: address,
        fee_bps: u64,
        fee_collected: Balance,
    }
    
    use sui::balance::{Self, Balance};
    
    struct Listing<phantom T: key + store> has key {
        id: UID,
        item_id: ID,        // ID of the item being sold
        seller: address,
        price: u64,         // in MIST (SUI smallest unit)
        created_at: u64,
    }
    
    struct ListingCap has key {
        id: UID,
        listing_id: ID,
    }
    
    // ============================================
    // Events
    // ============================================
    
    struct ItemListed has copy, drop {
        listing_id: ID,
        item_id: ID,
        seller: address,
        price: u64,
    }
    
    struct ItemSold has copy, drop {
        listing_id: ID,
        item_id: ID,
        seller: address,
        buyer: address,
        price: u64,
    }
    
    struct ListingCancelled has copy, drop {
        listing_id: ID,
        seller: address,
    }
    
    // ============================================
    // Errors
    // ============================================
    
    const E_NOT_SELLER: u64 = 1;
    const E_INSUFFICIENT_PAYMENT: u64 = 2;
    const E_WRONG_LISTING: u64 = 3;
    
    // ============================================
    // Initialize marketplace
    // ============================================
    
    fun init(ctx: &mut TxContext) {
        let marketplace = Marketplace {
            id: object::new(ctx),
            admin: tx_context::sender(ctx),
            fee_bps: 250,  // 2.5%
            fee_collected: balance::zero(),
        };
        transfer::share_object(marketplace);
    }
    
    // ============================================
    // List item for sale
    // ============================================
    
    public entry fun list_item<T: key + store>(
        marketplace: &mut Marketplace,
        item: T,
        price: u64,
        ctx: &mut TxContext,
    ) {
        let item_id = object::id(&item);
        let seller = tx_context::sender(ctx);
        
        let listing_uid = object::new(ctx);
        let listing_id = object::uid_to_inner(&listing_uid);
        
        let listing = Listing<T> {
            id: listing_uid,
            item_id,
            seller,
            price,
            created_at: tx_context::epoch(ctx),
        };
        
        // Store item as child of listing
        dynamic_object_field::add(&mut listing.id, b"item", item);
        
        // Seller gets a cap to cancel
        let cap = ListingCap {
            id: object::new(ctx),
            listing_id,
        };
        transfer::transfer(cap, seller);
        
        event::emit(ItemListed { listing_id, item_id, seller, price });
        
        // Share listing so buyers can access it
        transfer::share_object(listing);
    }
    
    // ============================================
    // Buy item
    // ============================================
    
    public entry fun buy_item<T: key + store>(
        marketplace: &mut Marketplace,
        listing: &mut Listing<T>,
        mut payment: Coin<SUI>,
        ctx: &mut TxContext,
    ) {
        let buyer = tx_context::sender(ctx);
        
        assert!(coin::value(&payment) >= listing.price, E_INSUFFICIENT_PAYMENT);
        
        // Extract exact payment
        let paid = coin::split(&mut payment, listing.price, ctx);
        
        // Calculate and take fee
        let fee_amount = listing.price * marketplace.fee_bps / 10_000;
        let fee_coin = coin::split(&mut paid, fee_amount, ctx);
        balance::join(&mut marketplace.fee_collected, coin::into_balance(fee_coin));
        
        // Send remaining payment to seller
        transfer::public_transfer(paid, listing.seller);
        
        // Return change to buyer
        if (coin::value(&payment) > 0) {
            transfer::public_transfer(payment, buyer);
        } else {
            coin::destroy_zero(payment);
        };
        
        // Take item from listing and give to buyer
        let item = dynamic_object_field::remove<vector<u8>, T>(&mut listing.id, b"item");
        transfer::public_transfer(item, buyer);
        
        event::emit(ItemSold {
            listing_id: object::uid_to_inner(&listing.id),
            item_id: listing.item_id,
            seller: listing.seller,
            buyer,
            price: listing.price,
        });
    }
    
    // ============================================
    // Cancel listing (seller only)
    // ============================================
    
    public entry fun cancel_listing<T: key + store>(
        listing: &mut Listing<T>,
        cap: ListingCap,
        ctx: &mut TxContext,
    ) {
        let sender = tx_context::sender(ctx);
        assert!(listing.seller == sender, E_NOT_SELLER);
        assert!(cap.listing_id == object::uid_to_inner(&listing.id), E_WRONG_LISTING);
        
        // Return item to seller
        let item = dynamic_object_field::remove<vector<u8>, T>(&mut listing.id, b"item");
        transfer::public_transfer(item, sender);
        
        // Destroy cap
        let ListingCap { id: cap_id, listing_id: _ } = cap;
        object::delete(cap_id);
        
        event::emit(ListingCancelled {
            listing_id: object::uid_to_inner(&listing.id),
            seller: sender,
        });
    }
    
    // ============================================
    // Withdraw fees (admin only)
    // ============================================
    
    public entry fun withdraw_fees(
        marketplace: &mut Marketplace,
        ctx: &mut TxContext,
    ) {
        assert!(tx_context::sender(ctx) == marketplace.admin, 0);
        
        let amount = balance::value(&marketplace.fee_collected);
        if (amount == 0) return;
        
        let coin = coin::from_balance(
            balance::split(&mut marketplace.fee_collected, amount),
            ctx,
        );
        transfer::public_transfer(coin, marketplace.admin);
    }
    
    // ============================================
    // View functions
    // ============================================
    
    public fun listing_price<T: key + store>(listing: &Listing<T>): u64 {
        listing.price
    }
    
    public fun listing_seller<T: key + store>(listing: &Listing<T>): address {
        listing.seller
    }
}
```

---

## เปรียบเทียบ Aptos vs Sui

| Aspect | Aptos | Sui |
|--------|-------|-----|
| Storage | Global by address | Per-object |
| Access pattern | `borrow_global<T>(addr)` | Function parameter |
| Ownership | Address-based | Built into Object |
| Shared state | Resources at shared addr | `share_object` |
| Extensibility | Multiple resources at addr | Dynamic fields |
| Events | `event::emit(...)` | `event::emit(...)` |
| Init function | `init_module(admin: &signer)` | `fun init(ctx: &mut TxContext)` |
| Coin type | `Coin<T>` | `Coin<T>` (similar) |

---

**ก่อนหน้า**: [Part 25 - Aptos Token Standard ←](part-25-aptos-token-standard.md)
**ต่อไป**: [Part 27 - Sui Coin and DeFi →](part-27-sui-coin-defi.md)
