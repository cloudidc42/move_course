# Part 23: Aptos Object Model

## สารบัญ
- [Object Model คืออะไร](#object-model-คืออะไร)
- [สร้าง Object](#สร้าง-object)
- [Object Ownership](#object-ownership)
- [Extending Objects](#extending-objects)
- [ตัวอย่าง: NFT Collection](#ตัวอย่าง-nft-collection)

---

## Object Model คืออะไร

```
แบบเดิม (Resource-based)          Aptos Object Model
──────────────────────────        ──────────────────────────
Resources at address              Objects = typed addresses
  @alice                            Object<NFT> at 0xABC
    NFT { ... }                       ObjectCore { owner: @alice }
    Coin { ... }                      NFT { name: "...", ... }
                                      
  Problems:                         Benefits:
  - One per type per address        - Multiple of same type
  - Hard to transfer               - Easy transfer: change owner
  - No composability               - Composable: Object in Object
  - No events on ownership         - Events on transfer
```

Object = ที่อยู่พิเศษที่มี `ObjectCore` resource
- มี owner ที่เปลี่ยนได้
- มีที่อยู่ของตัวเอง (ไม่ใช่ user account)
- สามารถ hold resources อื่นได้
- ส่งต่อ ownership ได้ง่าย

---

## สร้าง Object

```move
module learning::object_basics {
    use std::signer;
    use aptos_framework::object::{
        Self,
        Object,
        ObjectCore,
        ConstructorRef,
        ExtendRef,
        DeleteRef,
        TransferRef,
    };
    
    // ============================================
    // Types ที่สร้างได้บน Object
    // ============================================
    
    struct GameItem has key {
        name: vector<u8>,
        power: u64,
        durability: u64,
    }
    
    struct ItemExtension has key {
        enchantment: vector<u8>,
        bonus_power: u64,
    }
    
    // ============================================
    // Object Refs
    // ============================================
    
    // ConstructorRef - สร้าง object ได้ (ใช้ได้ครั้งเดียวในฟังก์ชัน create)
    // ExtendRef     - เพิ่ม resources ให้ object ได้ทีหลัง
    // DeleteRef     - ลบ object ได้
    // TransferRef   - ย้าย object ได้ (bypass frozen)
    
    struct ItemStore has key {
        extend_ref: ExtendRef,
        delete_ref: DeleteRef,
        transfer_ref: TransferRef,
    }
    
    // ============================================
    // สร้าง Object 3 แบบ
    // ============================================
    
    // 1. Named Object = deterministic address จาก creator + seed
    public fun create_named_item(creator: &signer, seed: vector<u8>, name: vector<u8>) {
        let constructor_ref = object::create_named_object(creator, seed);
        
        // Get signer for the new object (to move_to it)
        let obj_signer = object::generate_signer(&constructor_ref);
        
        move_to(&obj_signer, GameItem { name, power: 10, durability: 100 });
        
        // Named objects cannot be deleted (no DeleteRef available)
        let extend_ref = object::generate_extend_ref(&constructor_ref);
        let transfer_ref = object::generate_transfer_ref(&constructor_ref);
        
        move_to(&obj_signer, ItemStore {
            extend_ref,
            delete_ref: object::generate_delete_ref(&constructor_ref),  // ERROR: named objects can't be deleted
            transfer_ref,
        });
        // Note: In practice named objects don't support delete_ref
    }
    
    // 2. Sticky Object = like named but with different seed mechanism
    // object::create_sticky_object creates non-deletable objects
    
    // 3. Regular Object = random address, can be deleted
    public entry fun create_item(
        creator: &signer,
        name: vector<u8>,
        power: u64,
    ) {
        let constructor_ref = object::create_object(signer::address_of(creator));
        let obj_signer = object::generate_signer(&constructor_ref);
        
        // Store the actual data
        move_to(&obj_signer, GameItem { name, power, durability: 100 });
        
        // Save refs for later use
        let extend_ref = object::generate_extend_ref(&constructor_ref);
        let delete_ref = object::generate_delete_ref(&constructor_ref);
        let transfer_ref = object::generate_transfer_ref(&constructor_ref);
        
        move_to(&obj_signer, ItemStore { extend_ref, delete_ref, transfer_ref });
        
        // Object is now owned by creator (default)
    }
    
    // ============================================
    // Work with existing Objects
    // ============================================
    
    #[view]
    public fun get_item(item_obj: Object<GameItem>): (vector<u8>, u64, u64) acquires GameItem {
        let item = borrow_global<GameItem>(object::object_address(&item_obj));
        (item.name, item.power, item.durability)
    }
    
    public entry fun upgrade_power(
        owner: &signer,
        item_obj: Object<GameItem>,
        bonus: u64,
    ) acquires GameItem {
        // Must be owner to upgrade
        assert!(
            object::is_owner(item_obj, signer::address_of(owner)),
            1,  // E_NOT_OWNER
        );
        
        let item = borrow_global_mut<GameItem>(object::object_address(&item_obj));
        item.power = item.power + bonus;
    }
    
    public entry fun use_item(
        owner: &signer,
        item_obj: Object<GameItem>,
    ) acquires GameItem {
        assert!(object::is_owner(item_obj, signer::address_of(owner)), 1);
        
        let item = borrow_global_mut<GameItem>(object::object_address(&item_obj));
        if (item.durability > 0) {
            item.durability = item.durability - 1;
        };
    }
    
    // ============================================
    // Transfer Object
    // ============================================
    
    public entry fun transfer_item(
        from: &signer,
        item_obj: Object<GameItem>,
        to: address,
    ) {
        // Simple transfer (respects frozen state)
        object::transfer(from, item_obj, to);
    }
    
    // Transfer with ref (bypass frozen)
    public fun force_transfer_item(
        item_addr: address,
        to: address,
    ) acquires ItemStore {
        let store = borrow_global<ItemStore>(item_addr);
        let linear_ref = object::generate_linear_transfer_ref(&store.transfer_ref);
        object::transfer_with_ref(linear_ref, to);
    }
    
    // ============================================
    // Delete Object
    // ============================================
    
    public entry fun delete_item(
        owner: &signer,
        item_obj: Object<GameItem>,
    ) acquires GameItem, ItemStore {
        let item_addr = object::object_address(&item_obj);
        assert!(object::is_owner(item_obj, signer::address_of(owner)), 1);
        
        // First remove all resources
        let GameItem { name: _, power: _, durability: _ } = move_from<GameItem>(item_addr);
        let ItemStore { extend_ref: _, delete_ref, transfer_ref: _ } = 
            move_from<ItemStore>(item_addr);
        
        // Then delete the object
        object::delete(delete_ref);
    }
    
    // ============================================
    // Extend Object (add resources later)
    // ============================================
    
    public entry fun enchant_item(
        owner: &signer,
        item_obj: Object<GameItem>,
        enchantment: vector<u8>,
        bonus: u64,
    ) acquires ItemStore {
        let item_addr = object::object_address(&item_obj);
        assert!(object::is_owner(item_obj, signer::address_of(owner)), 1);
        
        let store = borrow_global<ItemStore>(item_addr);
        let obj_signer = object::generate_signer_for_extending(&store.extend_ref);
        
        // Add new resource to existing object
        move_to(&obj_signer, ItemExtension {
            enchantment,
            bonus_power: bonus,
        });
    }
    
    // ============================================
    // Query Object info
    // ============================================
    
    #[view]
    public fun get_owner(item_obj: Object<GameItem>): address {
        object::owner(item_obj)
    }
    
    #[view]
    public fun is_owner(item_obj: Object<GameItem>, addr: address): bool {
        object::is_owner(item_obj, addr)
    }
    
    #[view]
    public fun get_address(item_obj: Object<GameItem>): address {
        object::object_address(&item_obj)
    }
    
    // Check if address is an object
    public fun is_object(addr: address): bool {
        object::is_object(addr)
    }
    
    // Convert from address to typed Object (asserts type exists)
    public fun get_item_object(addr: address): Object<GameItem> {
        object::address_to_object<GameItem>(addr)
    }
}
```

---

## Object Ownership

```move
module learning::object_ownership {
    use std::signer;
    use aptos_framework::object::{Self, Object};
    
    struct Container has key {
        item: Object<Item>,  // Object owning another Object
    }
    
    struct Item has key {
        value: u64,
    }
    
    // ============================================
    // Nested ownership
    // ============================================
    
    public entry fun create_with_container(creator: &signer) {
        let creator_addr = signer::address_of(creator);
        
        // Create inner item
        let item_ref = object::create_object(creator_addr);
        let item_signer = object::generate_signer(&item_ref);
        move_to(&item_signer, Item { value: 42 });
        let item_obj = object::object_from_constructor_ref::<Item>(&item_ref);
        
        // Create container
        let container_ref = object::create_object(creator_addr);
        let container_signer = object::generate_signer(&container_ref);
        
        // Transfer item to container (container owns item)
        let item_transfer = object::generate_transfer_ref(&item_ref);
        let linear_ref = object::generate_linear_transfer_ref(&item_transfer);
        object::transfer_with_ref(linear_ref, object::object_address(&object::object_from_constructor_ref::<Container>(&container_ref)));
        
        // Container stores reference to item
        move_to(&container_signer, Container { item: item_obj });
    }
    
    // ============================================
    // Soulbound (non-transferable)
    // ============================================
    
    struct SoulboundNFT has key {
        name: vector<u8>,
        owner: address,
    }
    
    public entry fun create_soulbound(creator: &signer, to: address, name: vector<u8>) {
        let constructor_ref = object::create_object(signer::address_of(creator));
        let obj_signer = object::generate_signer(&constructor_ref);
        
        // Disable transfer
        object::set_untransferable(&constructor_ref);
        
        // Transfer to recipient
        let transfer_ref = object::generate_transfer_ref(&constructor_ref);
        let linear_ref = object::generate_linear_transfer_ref(&transfer_ref);
        object::transfer_with_ref(linear_ref, to);
        
        move_to(&obj_signer, SoulboundNFT { name, owner: to });
        
        // Note: after set_untransferable, even the owner can't transfer
    }
}
```

---

## ตัวอย่าง: NFT Collection

```move
module learning::nft_collection {
    use std::string::{Self, String};
    use std::signer;
    use std::vector;
    use aptos_framework::object::{Self, Object, ExtendRef};
    use aptos_framework::event;
    
    // ============================================
    // Types
    // ============================================
    
    struct Collection has key {
        name: String,
        creator: address,
        max_supply: u64,
        minted: u64,
        base_uri: String,
        extend_ref: ExtendRef,
    }
    
    struct NFT has key {
        collection: Object<Collection>,
        token_id: u64,
        name: String,
        description: String,
        uri: String,
    }
    
    struct CreatorCap has key {
        collection: Object<Collection>,
    }
    
    // ============================================
    // Events
    // ============================================
    
    #[event]
    struct CollectionCreated has drop, store {
        creator: address,
        collection: address,
        name: String,
        max_supply: u64,
    }
    
    #[event]
    struct NFTMinted has drop, store {
        collection: address,
        token_id: u64,
        recipient: address,
        name: String,
    }
    
    #[event]
    struct NFTTransferred has drop, store {
        token: address,
        from: address,
        to: address,
    }
    
    // ============================================
    // Errors
    // ============================================
    
    const E_COLLECTION_FULL: u64 = 1;
    const E_NOT_CREATOR: u64 = 2;
    const E_NOT_OWNER: u64 = 3;
    
    // ============================================
    // Create Collection
    // ============================================
    
    public entry fun create_collection(
        creator: &signer,
        name: String,
        max_supply: u64,
        base_uri: String,
    ) {
        let creator_addr = signer::address_of(creator);
        
        // Use collection name as seed for deterministic address
        let seed = *string::bytes(&name);
        let constructor_ref = object::create_named_object(creator, seed);
        let obj_signer = object::generate_signer(&constructor_ref);
        let extend_ref = object::generate_extend_ref(&constructor_ref);
        
        move_to(&obj_signer, Collection {
            name,
            creator: creator_addr,
            max_supply,
            minted: 0,
            base_uri,
            extend_ref,
        });
        
        let collection_obj = object::object_from_constructor_ref::<Collection>(&constructor_ref);
        let collection_addr = object::object_address(&collection_obj);
        
        // Give creator capability
        move_to(creator, CreatorCap { collection: collection_obj });
        
        event::emit(CollectionCreated {
            creator: creator_addr,
            collection: collection_addr,
            name: string::utf8(b""),  // simplified
            max_supply,
        });
    }
    
    // ============================================
    // Mint NFT
    // ============================================
    
    public entry fun mint_nft(
        creator: &signer,
        collection_obj: Object<Collection>,
        recipient: address,
        description: String,
        custom_uri: String,
    ) acquires Collection, CreatorCap {
        let creator_addr = signer::address_of(creator);
        let cap = borrow_global<CreatorCap>(creator_addr);
        assert!(cap.collection == collection_obj, E_NOT_CREATOR);
        
        let collection_addr = object::object_address(&collection_obj);
        let collection = borrow_global_mut<Collection>(collection_addr);
        
        assert!(collection.minted < collection.max_supply, E_COLLECTION_FULL);
        collection.minted = collection.minted + 1;
        let token_id = collection.minted;
        
        // Build token name: "Collection #1", "Collection #2", etc.
        let name = build_token_name(&collection.name, token_id);
        
        // Build URI
        let uri = if (string::length(&custom_uri) > 0) {
            custom_uri
        } else {
            build_uri(&collection.base_uri, token_id)
        };
        
        // Create NFT object
        // Use extend_ref to create from collection's signer
        let nft_ref = object::create_object(collection_addr);
        let nft_signer = object::generate_signer(&nft_ref);
        
        let transfer_ref = object::generate_transfer_ref(&nft_ref);
        
        move_to(&nft_signer, NFT {
            collection: collection_obj,
            token_id,
            name,
            description,
            uri,
        });
        
        // Transfer to recipient
        let linear_ref = object::generate_linear_transfer_ref(&transfer_ref);
        object::transfer_with_ref(linear_ref, recipient);
        
        let nft_obj = object::object_from_constructor_ref::<NFT>(&nft_ref);
        
        event::emit(NFTMinted {
            collection: collection_addr,
            token_id,
            recipient,
            name: string::utf8(b""),
        });
        let _ = nft_obj;
    }
    
    // ============================================
    // Transfer NFT
    // ============================================
    
    public entry fun transfer_nft(
        owner: &signer,
        nft_obj: Object<NFT>,
        to: address,
    ) {
        assert!(
            object::is_owner(nft_obj, signer::address_of(owner)),
            E_NOT_OWNER,
        );
        
        let from = signer::address_of(owner);
        object::transfer(owner, nft_obj, to);
        
        event::emit(NFTTransferred {
            token: object::object_address(&nft_obj),
            from,
            to,
        });
    }
    
    // ============================================
    // View Functions
    // ============================================
    
    #[view]
    public fun get_nft_info(nft_obj: Object<NFT>): (u64, String, String) acquires NFT {
        let nft = borrow_global<NFT>(object::object_address(&nft_obj));
        (nft.token_id, nft.name, nft.uri)
    }
    
    #[view]
    public fun get_nft_owner(nft_obj: Object<NFT>): address {
        object::owner(nft_obj)
    }
    
    #[view]
    public fun get_collection_info(collection_obj: Object<Collection>): (String, u64, u64) 
    acquires Collection {
        let col = borrow_global<Collection>(object::object_address(&collection_obj));
        (col.name, col.max_supply, col.minted)
    }
    
    #[view]
    public fun collection_address(creator: address, name: String): address {
        let seed = *string::bytes(&name);
        object::create_object_address(&creator, seed)
    }
    
    // ============================================
    // Helpers
    // ============================================
    
    fun build_token_name(collection_name: &String, id: u64): String {
        let name = *collection_name;
        string::append_utf8(&mut name, b" #");
        string::append_utf8(&mut name, u64_to_bytes(id));
        name
    }
    
    fun build_uri(base: &String, id: u64): String {
        let uri = *base;
        string::append_utf8(&mut uri, u64_to_bytes(id));
        string::append_utf8(&mut uri, b".json");
        uri
    }
    
    fun u64_to_bytes(n: u64): vector<u8> {
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

## สรุป Object Model

| Concept | Description |
|---------|-------------|
| `Object<T>` | Typed reference to an object holding resource T |
| `ConstructorRef` | One-time ref for initialization |
| `ExtendRef` | Add resources later |
| `DeleteRef` | Delete the object |
| `TransferRef` | Transfer with override |
| Named Object | Deterministic address, non-deletable |
| Regular Object | Random address, deletable |
| Soulbound | Non-transferable after creation |

---

**ก่อนหน้า**: [Part 22 - Fungible Assets ←](part-22-fungible-assets.md)
**ต่อไป**: [Part 24 - Tables and SmartTable →](part-24-tables-smarttable.md)
