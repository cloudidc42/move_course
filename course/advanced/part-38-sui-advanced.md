# Part 38: Advanced Sui Patterns

## สารบัญ
- [Kiosk & Transfer Policy](#kiosk--transfer-policy)
- [zkLogin Integration](#zklogin-integration)
- [Programmable Transaction Blocks](#programmable-transaction-blocks)
- [Dynamic Field Patterns](#dynamic-field-patterns)
- [Versioned Objects](#versioned-objects)
- [ตัวอย่าง: NFT Marketplace with Royalties](#ตัวอย่าง-nft-marketplace-with-royalties)

---

## Kiosk & Transfer Policy

```move
module sui_adv::kiosk_example {
    use sui::kiosk::{Self, Kiosk, KioskOwnerCap};
    use sui::transfer_policy::{Self, TransferPolicy, TransferPolicyCap, TransferRequest};
    use sui::package;
    use sui::tx_context::{Self, TxContext};
    
    // ============================================
    // Sui Kiosk: NFT marketplace primitive
    // Enforces creator royalties via TransferPolicy
    // ============================================
    
    // NFT type
    struct MyNFT has key, store {
        id: sui::object::UID,
        name: std::string::String,
        creator: address,
    }
    
    // Witness for transfer policy
    struct MYROYALTY has drop {}
    
    // Royalty rule
    struct RoyaltyRule has drop {}
    
    // ============================================
    // Create NFT and Transfer Policy
    // ============================================
    
    fun init(witness: MYROYALTY, ctx: &mut TxContext) {
        // Create transfer policy requiring royalty payment
        let (policy, policy_cap) = transfer_policy::new<MyNFT>(&witness, ctx);
        
        // Transfer policy to creator
        let sender = tx_context::sender(ctx);
        sui::transfer::public_transfer(policy_cap, sender);
        sui::transfer::public_share_object(policy);
    }
    
    // ============================================
    // Mint NFT into kiosk
    // ============================================
    
    public entry fun mint_to_kiosk(
        kiosk: &mut Kiosk,
        kiosk_cap: &KioskOwnerCap,
        name: std::string::String,
        ctx: &mut TxContext,
    ) {
        let nft = MyNFT {
            id: sui::object::new(ctx),
            name,
            creator: tx_context::sender(ctx),
        };
        
        // Place in kiosk (locks it, requires transfer policy to transfer)
        kiosk::place(kiosk, kiosk_cap, nft);
    }
    
    // ============================================
    // List NFT for sale in kiosk
    // ============================================
    
    public entry fun list_nft(
        kiosk: &mut Kiosk,
        kiosk_cap: &KioskOwnerCap,
        nft_id: sui::object::ID,
        price: u64,
    ) {
        kiosk::list<MyNFT>(kiosk, kiosk_cap, nft_id, price);
    }
    
    // ============================================
    // Buy NFT from kiosk (enforces transfer policy)
    // ============================================
    
    public entry fun buy_nft(
        kiosk: &mut Kiosk,
        policy: &mut TransferPolicy<MyNFT>,
        nft_id: sui::object::ID,
        payment: sui::coin::Coin<sui::sui::SUI>,
        ctx: &mut TxContext,
    ) {
        // Purchase creates TransferRequest that must be resolved
        let (nft, mut request) = kiosk::purchase<MyNFT>(kiosk, nft_id, payment);
        
        // Pay royalty (required by TransferPolicy)
        // In production: add royalty rule to policy
        // royalty_rule::pay(&mut request, ctx);
        
        // Confirm transfer (policy checks all rules satisfied)
        transfer_policy::confirm_request(policy, request);
        
        // Transfer NFT to buyer
        let buyer = tx_context::sender(ctx);
        sui::transfer::public_transfer(nft, buyer);
    }
    
    // ============================================
    // Royalty Rule implementation
    // ============================================
    
    public fun add_royalty_rule(
        policy: &mut TransferPolicy<MyNFT>,
        policy_cap: &TransferPolicyCap<MyNFT>,
        amount_bps: u16,   // royalty in basis points
        min_amount: u64,
    ) {
        // Add rule requiring royalty payment
        // transfer_policy::add_rule(RoyaltyRule {}, policy, policy_cap, (amount_bps, min_amount));
    }
}
```

---

## Programmable Transaction Blocks

```
PTB (Programmable Transaction Blocks) คือ:
  - Aptos: ไม่มี native PTB, แต่ใช้ multi-action txs
  - Sui: PTB is core feature
  
PTB ทำให้:
  1. Compose multiple calls atomically
  2. Pass outputs of one call as inputs to next
  3. No intermediate transaction fees
  4. Gas efficient
  5. One signature for multiple operations

ตัวอย่าง PTB (TypeScript/Sui SDK):
  const tx = new Transaction();
  
  // Split coin
  const [coin] = tx.splitCoins(tx.gas, [1000]);
  
  // Buy NFT from kiosk
  const [nft, request] = tx.moveCall({
    target: '0x...::kiosk::purchase',
    arguments: [kiosk, nftId, coin],
  });
  
  // Pay royalty in same tx
  tx.moveCall({
    target: '0x...::royalty::pay',
    arguments: [request],
  });
  
  // Transfer NFT
  tx.transferObjects([nft], tx.pure(recipient));
  
  await client.signAndExecuteTransaction({ transaction: tx });
```

---

## Dynamic Field Patterns

```move
module sui_adv::dynamic_patterns {
    use sui::dynamic_field;
    use sui::dynamic_object_field;
    use sui::object::{Self, UID};
    use sui::tx_context::TxContext;
    
    // ============================================
    // Dynamic Fields: extensible objects
    // ============================================
    
    // Parent object (acts as map)
    struct DataStore has key {
        id: UID,
    }
    
    // Various data types stored dynamically
    struct Config has store {
        value: u64,
        enabled: bool,
    }
    
    // ============================================
    // Type-safe dynamic field access
    // ============================================
    
    // Keys must have: copy + drop + store
    // But they're encoded by type, so same value but different types = different fields!
    
    struct ConfigKey has copy, drop, store {}
    struct CounterKey has copy, drop, store {}
    struct UserDataKey has copy, drop, store { user: address }
    
    public fun set_config(
        store: &mut DataStore,
        value: u64,
        enabled: bool,
    ) {
        if (dynamic_field::exists_(&store.id, ConfigKey {})) {
            let config: &mut Config = dynamic_field::borrow_mut(&mut store.id, ConfigKey {});
            config.value = value;
            config.enabled = enabled;
        } else {
            dynamic_field::add(&mut store.id, ConfigKey {}, Config { value, enabled });
        };
    }
    
    public fun get_config(store: &DataStore): (u64, bool) {
        if (!dynamic_field::exists_(&store.id, ConfigKey {})) {
            return (0, false)
        };
        let config: &Config = dynamic_field::borrow(&store.id, ConfigKey {});
        (config.value, config.enabled)
    }
    
    // ============================================
    // Dynamic Object Fields (child objects)
    // ============================================
    
    // Child object has its own UID
    struct ChildNFT has key, store {
        id: UID,
        name: std::string::String,
        power: u64,
    }
    
    struct EquipmentKey has copy, drop, store { slot: u8 }
    
    // Equip NFT as child of another NFT
    public fun equip(
        parent: &mut UID,
        slot: u8,
        nft: ChildNFT,
    ) {
        // If slot occupied, remove first
        if (dynamic_object_field::exists_(parent, EquipmentKey { slot })) {
            let _old: ChildNFT = dynamic_object_field::remove(parent, EquipmentKey { slot });
            // old NFT becomes orphaned (needs to be transferred somewhere)
        };
        dynamic_object_field::add(parent, EquipmentKey { slot }, nft);
    }
    
    public fun unequip(
        parent: &mut UID,
        slot: u8,
    ): ChildNFT {
        dynamic_object_field::remove(parent, EquipmentKey { slot })
    }
    
    public fun get_equipment(parent: &UID, slot: u8): &ChildNFT {
        dynamic_object_field::borrow(parent, EquipmentKey { slot })
    }
    
    // ============================================
    // Bag: heterogeneous key-value store
    // ============================================
    
    use sui::bag::{Self, Bag};
    
    struct MultiStore has key {
        id: UID,
        data: Bag,
    }
    
    public fun store_value<T: store>(
        ms: &mut MultiStore,
        key: std::string::String,
        value: T,
    ) {
        bag::add(&mut ms.data, key, value);
    }
    
    public fun get_value<T: store>(
        ms: &MultiStore,
        key: std::string::String,
    ): &T {
        bag::borrow(&ms.data, key)
    }
    
    // ============================================
    // Table: homogeneous key-value store
    // ============================================
    
    use sui::table::{Self, Table};
    
    struct Registry has key {
        id: UID,
        entries: Table<address, u64>,
    }
    
    public fun register(
        registry: &mut Registry,
        user: address,
        score: u64,
    ) {
        if (table::contains(&registry.entries, user)) {
            *table::borrow_mut(&mut registry.entries, user) = score;
        } else {
            table::add(&mut registry.entries, user, score);
        };
    }
    
    public fun get_score(registry: &Registry, user: address): u64 {
        if (!table::contains(&registry.entries, user)) return 0;
        *table::borrow(&registry.entries, user)
    }
    
    // ============================================
    // VecMap: ordered map for small collections
    // ============================================
    
    use sui::vec_map::{Self, VecMap};
    
    struct SmallConfig has key {
        id: UID,
        params: VecMap<std::string::String, u64>,
    }
    
    public fun set_param(
        config: &mut SmallConfig,
        key: std::string::String,
        value: u64,
    ) {
        if (vec_map::contains(&config.params, &key)) {
            *vec_map::get_mut(&mut config.params, &key) = value;
        } else {
            vec_map::insert(&mut config.params, key, value);
        };
    }
}
```

---

## Versioned Objects

```move
module sui_adv::versioned_objects {
    use sui::object::{Self, UID};
    use sui::versioned::{Self, Versioned};
    use sui::tx_context::TxContext;
    
    // ============================================
    // Sui Versioned: upgrade shared objects
    // ============================================
    
    // Old state (V1)
    struct ProtocolV1 has store {
        fee_bps: u64,
        volume: u64,
    }
    
    // New state (V2)
    struct ProtocolV2 has store {
        fee_bps: u64,
        volume: u64,
        // new in V2
        protocol_fee_bps: u64,
        treasury: address,
    }
    
    struct Protocol has key {
        id: UID,
        version: u64,
        inner: Versioned,
    }
    
    const PROTOCOL_VERSION_V1: u64 = 1;
    const PROTOCOL_VERSION_V2: u64 = 2;
    
    // ============================================
    // Initialize with V1
    // ============================================
    
    public entry fun initialize(fee_bps: u64, ctx: &mut TxContext) {
        let inner = versioned::create(
            PROTOCOL_VERSION_V1,
            ProtocolV1 { fee_bps, volume: 0 },
            ctx,
        );
        
        sui::transfer::share_object(Protocol {
            id: object::new(ctx),
            version: PROTOCOL_VERSION_V1,
            inner,
        });
    }
    
    // ============================================
    // Upgrade to V2
    // ============================================
    
    public entry fun upgrade_to_v2(
        protocol: &mut Protocol,
        treasury: address,
        protocol_fee_bps: u64,
        ctx: &mut TxContext,
    ) {
        assert!(protocol.version == PROTOCOL_VERSION_V1, 1);
        
        // Destructure old state
        let v1: ProtocolV1 = versioned::destroy(
            protocol.inner,
            PROTOCOL_VERSION_V1,
        );
        
        // Create new state with migrated data
        protocol.inner = versioned::create(
            PROTOCOL_VERSION_V2,
            ProtocolV2 {
                fee_bps: v1.fee_bps,
                volume: v1.volume,
                protocol_fee_bps,
                treasury,
            },
            ctx,
        );
        protocol.version = PROTOCOL_VERSION_V2;
    }
    
    // ============================================
    // Access with version check
    // ============================================
    
    public fun get_fee_bps(protocol: &Protocol): u64 {
        if (protocol.version == PROTOCOL_VERSION_V1) {
            let v1: &ProtocolV1 = versioned::load_value(&protocol.inner, PROTOCOL_VERSION_V1);
            v1.fee_bps
        } else {
            let v2: &ProtocolV2 = versioned::load_value(&protocol.inner, PROTOCOL_VERSION_V2);
            v2.fee_bps
        }
    }
}
```

---

## ตัวอย่าง: NFT Marketplace with Royalties

```move
module sui_adv::royalty_marketplace {
    use sui::object::{Self, UID, ID};
    use sui::tx_context::{Self, TxContext};
    use sui::coin::{Self, Coin};
    use sui::sui::SUI;
    use sui::transfer;
    use sui::event;
    use sui::dynamic_field;
    use std::string::String;
    
    // ============================================
    // NFT with creator royalties
    // ============================================
    
    struct ArtNFT has key, store {
        id: UID,
        name: String,
        creator: address,
        royalty_bps: u64,   // creator royalty (e.g., 500 = 5%)
        edition: u64,
        max_edition: u64,
    }
    
    struct ListingKey has copy, drop, store {}
    struct ListingData has store {
        price: u64,
        seller: address,
    }
    
    struct Marketplace has key {
        id: UID,
        platform_fee_bps: u64,
        treasury: address,
        volume: u64,
        listing_count: u64,
    }
    
    struct ListingCreated has copy, drop {
        nft_id: ID,
        seller: address,
        price: u64,
    }
    
    struct NFTSold has copy, drop {
        nft_id: ID,
        seller: address,
        buyer: address,
        price: u64,
        royalty_paid: u64,
        platform_fee: u64,
    }
    
    const E_NOT_OWNER: u64 = 1;
    const E_NOT_LISTED: u64 = 2;
    const E_WRONG_PRICE: u64 = 3;
    const E_ALREADY_LISTED: u64 = 4;
    
    // ============================================
    // Mint NFT
    // ============================================
    
    public entry fun mint(
        name: String,
        royalty_bps: u64,
        max_edition: u64,
        ctx: &mut TxContext,
    ) {
        assert!(royalty_bps <= 1000, 100);  // max 10% royalty
        
        let creator = tx_context::sender(ctx);
        let nft = ArtNFT {
            id: object::new(ctx),
            name,
            creator,
            royalty_bps,
            edition: 1,
            max_edition,
        };
        
        transfer::public_transfer(nft, creator);
    }
    
    // ============================================
    // List NFT for sale
    // ============================================
    
    public entry fun list(
        nft: &mut ArtNFT,
        price: u64,
        ctx: &mut TxContext,
    ) {
        let sender = tx_context::sender(ctx);
        
        // Verify not already listed
        assert!(!dynamic_field::exists_(&nft.id, ListingKey {}), E_ALREADY_LISTED);
        
        // Store listing data in NFT's dynamic field
        dynamic_field::add(&mut nft.id, ListingKey {}, ListingData {
            price,
            seller: sender,
        });
        
        event::emit(ListingCreated {
            nft_id: object::id(nft),
            seller: sender,
            price,
        });
    }
    
    // ============================================
    // Delist NFT
    // ============================================
    
    public entry fun delist(
        nft: &mut ArtNFT,
        ctx: &mut TxContext,
    ) {
        assert!(dynamic_field::exists_(&nft.id, ListingKey {}), E_NOT_LISTED);
        let ListingData { price: _, seller } = 
            dynamic_field::remove(&mut nft.id, ListingKey {});
        assert!(seller == tx_context::sender(ctx), E_NOT_OWNER);
    }
    
    // ============================================
    // Buy NFT (with royalty distribution)
    // ============================================
    
    public entry fun buy(
        marketplace: &mut Marketplace,
        nft: ArtNFT,
        mut payment: Coin<SUI>,
        ctx: &mut TxContext,
    ) {
        // Get listing data
        assert!(dynamic_field::exists_(&nft.id, ListingKey {}), E_NOT_LISTED);
        let ListingData { price, seller } = 
            dynamic_field::remove(&mut object::borrow_uid(&nft), ListingKey {});
        
        let paid = coin::value(&payment);
        assert!(paid >= price, E_WRONG_PRICE);
        
        // Calculate fees
        let platform_fee = price * marketplace.platform_fee_bps / 10_000;
        let royalty = price * nft.royalty_bps / 10_000;
        let seller_amount = price - platform_fee - royalty;
        
        let buyer = tx_context::sender(ctx);
        
        // Pay platform fee
        let fee_coin = coin::split(&mut payment, platform_fee, ctx);
        transfer::public_transfer(fee_coin, marketplace.treasury);
        
        // Pay royalty to creator
        let royalty_coin = coin::split(&mut payment, royalty, ctx);
        transfer::public_transfer(royalty_coin, nft.creator);
        
        // Pay seller
        let seller_coin = coin::split(&mut payment, seller_amount, ctx);
        transfer::public_transfer(seller_coin, seller);
        
        // Return overpayment
        if (coin::value(&payment) > 0) {
            transfer::public_transfer(payment, buyer);
        } else {
            coin::destroy_zero(payment);
        };
        
        // Update marketplace stats
        marketplace.volume = marketplace.volume + price;
        marketplace.listing_count = marketplace.listing_count - 1;
        
        // Transfer NFT to buyer
        transfer::public_transfer(nft, buyer);
        
        event::emit(NFTSold {
            nft_id: object::id(&nft),  // ID captured before transfer
            seller,
            buyer,
            price,
            royalty_paid: royalty,
            platform_fee,
        });
    }
    
    // ============================================
    // Auction support
    // ============================================
    
    struct Auction has key {
        id: UID,
        nft_id: ID,
        seller: address,
        start_price: u64,
        current_bid: u64,
        highest_bidder: address,
        end_time: u64,
        bids: std::vector::vector<Bid>,
    }
    
    struct Bid has store {
        bidder: address,
        amount: u64,
        timestamp: u64,
    }
    
    // ============================================
    // View functions
    // ============================================
    
    public fun is_listed(nft: &ArtNFT): bool {
        dynamic_field::exists_(&nft.id, ListingKey {})
    }
    
    public fun listing_price(nft: &ArtNFT): u64 {
        if (!dynamic_field::exists_(&nft.id, ListingKey {})) return 0;
        let data: &ListingData = dynamic_field::borrow(&nft.id, ListingKey {});
        data.price
    }
    
    public fun marketplace_volume(marketplace: &Marketplace): u64 {
        marketplace.volume
    }
}
```

---

## สรุป Advanced Sui Patterns

| Feature | Description | When to Use |
|---------|-------------|-------------|
| Kiosk | NFT escrow with policy | NFT marketplaces |
| TransferPolicy | Enforced royalties | Creator economies |
| PTB | Batch transactions | Complex DeFi ops |
| Dynamic Fields | Extensible objects | Upgradeable state |
| Versioned | Safe upgrades | Protocol evolution |
| Bag/Table | On-chain storage | Collections, registries |

**Sui vs Aptos Design Philosophy:**
```
Sui:
  - Object-centric (objects have IDs, owned/shared/frozen)
  - PTB for composition
  - Transfer functions control ownership
  - Dynamic fields for extension
  
Aptos:
  - Account-centric (resources stored at addresses)
  - Multi-action transactions
  - Object model (newer, similar to Sui)
  - Tables for collections
```

---

**ก่อนหน้า**: [Part 37 - Security ←](part-37-security-mev.md)
**ต่อไป**: [Part 39 - Move Prover →](part-39-move-prover.md)
