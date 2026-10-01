# Part 22: Fungible Assets (New Standard)

## สารบัญ
- [ทำไมต้อง Fungible Assets?](#ทำไมต้อง-fungible-assets)
- [Core Concepts](#core-concepts)
- [สร้าง Fungible Asset](#สร้าง-fungible-asset)
- [Primary Fungible Store](#primary-fungible-store)
- [Dispatchable Hooks](#dispatchable-hooks)
- [ตัวอย่าง: ERC-20 Style Token](#ตัวอย่าง-erc-20-style-token)

---

## ทำไมต้อง Fungible Assets?

```
Coin Standard (Legacy)          Fungible Asset (New)
─────────────────────────       ─────────────────────
• One type per module           • Object-based
• Hard to extend                • Metadata on-chain
• No built-in freeze/burn       • Built-in controls
• Limited composability         • Fully composable
• Separate mint/burn caps       • Unified management
```

Fungible Asset (FA) standard ถูกออกแบบมาเพื่อแก้ปัญหาของ Coin standard:
- Metadata เก็บ on-chain (ชื่อ, symbol, decimals)
- รองรับ freeze/unfreeze ได้ง่าย
- Composable กับ Aptos Object model
- รองรับ primary/secondary stores

---

## Core Concepts

```move
module learning::fa_concepts {
    // ============================================
    // Key Types ใน fungible_asset module
    // ============================================
    
    // Object<Metadata> = represents the FA type (like "USDC")
    // FungibleAsset = actual coins/tokens (like holding $100 USDC)
    // FungibleStore = account's balance of a specific FA
    // MintRef = capability to mint
    // BurnRef = capability to burn
    // TransferRef = capability to transfer (override freeze)
    
    // ============================================
    // Relationship
    // ============================================
    
    // Object<Metadata>
    //   └── describes: name, symbol, decimals, supply
    //
    // FungibleStore
    //   ├── metadata: Object<Metadata>  <- which FA this is
    //   └── balance: u64
    //
    // FungibleAsset (value type, not stored)
    //   ├── metadata: Object<Metadata>
    //   └── amount: u64
}
```

---

## สร้าง Fungible Asset

```move
module learning::create_fa {
    use std::option;
    use std::string;
    use aptos_framework::object::{Self, Object};
    use aptos_framework::fungible_asset::{
        Self, 
        Metadata, 
        FungibleAsset,
        MintRef,
        BurnRef, 
        TransferRef,
    };
    use aptos_framework::primary_fungible_store;
    
    // ============================================
    // Marker struct for our FA
    // ============================================
    
    struct StableCoin {}  // phantom type marker
    
    // ============================================
    // Store capabilities
    // ============================================
    
    struct StableCoinAdmin has key {
        metadata: Object<Metadata>,
        mint_ref: MintRef,
        burn_ref: BurnRef,
        transfer_ref: TransferRef,
    }
    
    // ============================================
    // Create FA
    // ============================================
    
    public entry fun create_stablecoin(creator: &signer) {
        // Create an Object to hold FA metadata
        let constructor_ref = object::create_named_object(
            creator,
            b"USDC",  // seed for deterministic address
        );
        
        // Initialize as fungible asset
        primary_fungible_store::create_primary_store_enabled_fungible_asset(
            &constructor_ref,
            option::some(1_000_000_000_000u128),  // max supply (optional)
            string::utf8(b"USD Coin"),              // name
            string::utf8(b"USDC"),                  // symbol
            6,                                       // decimals
            string::utf8(b"https://usdc.circle.com/icon.png"),  // icon URI
            string::utf8(b"https://usdc.circle.com"),           // project URI
        );
        
        // Get capabilities
        let mint_ref = fungible_asset::generate_mint_ref(&constructor_ref);
        let burn_ref = fungible_asset::generate_burn_ref(&constructor_ref);
        let transfer_ref = fungible_asset::generate_transfer_ref(&constructor_ref);
        
        // Get metadata object
        let metadata = object::object_from_constructor_ref::<Metadata>(&constructor_ref);
        
        // Store capabilities
        use std::signer;
        move_to(creator, StableCoinAdmin {
            metadata,
            mint_ref,
            burn_ref,
            transfer_ref,
        });
    }
    
    // ============================================
    // Mint tokens
    // ============================================
    
    public entry fun mint(
        creator: &signer,
        to: address,
        amount: u64,
    ) acquires StableCoinAdmin {
        use std::signer;
        let creator_addr = signer::address_of(creator);
        let admin = borrow_global<StableCoinAdmin>(creator_addr);
        
        // Mint creates FungibleAsset value
        let fa = fungible_asset::mint(&admin.mint_ref, amount);
        
        // Deposit to primary store
        primary_fungible_store::deposit(to, fa);
    }
    
    // ============================================
    // Burn tokens  
    // ============================================
    
    public entry fun burn(
        creator: &signer,
        from: address,
        amount: u64,
    ) acquires StableCoinAdmin {
        use std::signer;
        let creator_addr = signer::address_of(creator);
        let admin = borrow_global<StableCoinAdmin>(creator_addr);
        
        // Withdraw from primary store (requires TransferRef to bypass freeze)
        let fa = primary_fungible_store::withdraw_with_ref(
            &admin.transfer_ref, from, amount
        );
        
        // Burn
        fungible_asset::burn(&admin.burn_ref, fa);
    }
    
    // ============================================
    // Freeze/Unfreeze account
    // ============================================
    
    public entry fun freeze_account(
        creator: &signer,
        target: address,
    ) acquires StableCoinAdmin {
        use std::signer;
        let creator_addr = signer::address_of(creator);
        let admin = borrow_global<StableCoinAdmin>(creator_addr);
        
        let store = primary_fungible_store::ensure_primary_store_exists(
            target, admin.metadata
        );
        fungible_asset::set_frozen_flag(&admin.transfer_ref, store, true);
    }
    
    public entry fun unfreeze_account(
        creator: &signer,
        target: address,
    ) acquires StableCoinAdmin {
        use std::signer;
        let creator_addr = signer::address_of(creator);
        let admin = borrow_global<StableCoinAdmin>(creator_addr);
        
        let store = primary_fungible_store::ensure_primary_store_exists(
            target, admin.metadata
        );
        fungible_asset::set_frozen_flag(&admin.transfer_ref, store, false);
    }
    
    // ============================================
    // Force transfer (admin can move despite freeze)
    // ============================================
    
    public entry fun force_transfer(
        creator: &signer,
        from: address,
        to: address,
        amount: u64,
    ) acquires StableCoinAdmin {
        use std::signer;
        let creator_addr = signer::address_of(creator);
        let admin = borrow_global<StableCoinAdmin>(creator_addr);
        
        let fa = primary_fungible_store::withdraw_with_ref(
            &admin.transfer_ref, from, amount
        );
        primary_fungible_store::deposit_with_ref(
            &admin.transfer_ref, to, fa
        );
    }
    
    // ============================================
    // View functions
    // ============================================
    
    #[view]
    public fun balance(addr: address, metadata: Object<Metadata>): u64 {
        primary_fungible_store::balance(addr, metadata)
    }
    
    #[view]
    public fun total_supply(metadata: Object<Metadata>): option::Option<u128> {
        fungible_asset::supply(metadata)
    }
    
    #[view]
    public fun is_frozen(addr: address, metadata: Object<Metadata>): bool {
        primary_fungible_store::is_frozen(addr, metadata)
    }
    
    #[view]
    public fun get_metadata_address(creator_addr: address): address {
        object::create_object_address(&creator_addr, b"USDC")
    }
}
```

---

## Primary Fungible Store

```move
module learning::pfs_usage {
    use aptos_framework::object::Object;
    use aptos_framework::fungible_asset::{Self, Metadata, FungibleAsset};
    use aptos_framework::primary_fungible_store;
    
    // ============================================
    // Primary Store = the "default" store for an address
    // ============================================
    
    // Each address has ONE primary store per FA type
    // Created automatically on first deposit
    // Most user operations use primary store
    
    // ============================================
    // Common operations
    // ============================================
    
    public fun transfer_fa(
        from: &signer,
        to: address,
        metadata: Object<Metadata>,
        amount: u64,
    ) {
        // Simple transfer between primary stores
        primary_fungible_store::transfer(from, metadata, to, amount);
    }
    
    public fun get_balance(addr: address, metadata: Object<Metadata>): u64 {
        primary_fungible_store::balance(addr, metadata)
    }
    
    public fun store_exists(addr: address, metadata: Object<Metadata>): bool {
        primary_fungible_store::primary_store_exists(addr, metadata)
    }
    
    // ============================================
    // Working with FungibleAsset directly
    // ============================================
    
    // Withdraw → get FungibleAsset value
    public fun withdraw(
        from: &signer,
        metadata: Object<Metadata>,
        amount: u64,
    ): FungibleAsset {
        primary_fungible_store::withdraw(from, metadata, amount)
    }
    
    // Deposit → put FungibleAsset into store
    public fun deposit_fa(to: address, fa: FungibleAsset) {
        primary_fungible_store::deposit(to, fa);
    }
    
    // ============================================
    // In DeFi protocols: hold FA in module
    // ============================================
    
    // A pool or vault might hold FungibleAsset in its own store
    // rather than in primary store
    
    struct LiquidityPool has key {
        store_a: fungible_asset::FungibleStore,  // conceptual
        store_b: fungible_asset::FungibleStore,
        lp_metadata: Object<Metadata>,
    }
}
```

---

## Dispatchable Hooks

```move
module learning::dispatchable_fa {
    use aptos_framework::fungible_asset::{
        Self, Metadata, FungibleAsset, FungibleStore, TransferRef,
        DispatchFunctionStore,
    };
    use aptos_framework::object::{Self, Object, ConstructorRef};
    use aptos_framework::function_info;
    
    // ============================================
    // Dispatchable = custom logic on transfer
    // ============================================
    
    // Allows hooking into deposit/withdraw with custom logic
    // Use cases:
    // - Auto-compounding (like stETH)
    // - Tax on transfer
    // - Rebasing tokens
    // - Yield-bearing tokens
    
    struct RebasingToken {}
    
    struct RebasingConfig has key {
        total_shares: u128,
        total_assets: u128,  // grows over time
        last_rebase: u64,
    }
    
    // ============================================
    // Initialize with custom hooks
    // ============================================
    
    public fun create_rebasing_token(creator: &signer) {
        let constructor_ref = object::create_named_object(creator, b"rebase_token");
        
        // Initialize FA
        primary_fungible_store_init(&constructor_ref);
        
        // Register dispatchable withdraw hook
        // When someone withdraws, our custom logic runs
        fungible_asset::register_dispatch_functions(
            &constructor_ref,
            option::some(function_info::new_function_info(
                creator,
                std::string::utf8(b"dispatchable_fa"),
                std::string::utf8(b"withdraw"),
            )),
            option::none(),  // no deposit hook
            option::none(),  // no derived_balance hook
        );
    }
    
    // Custom withdraw: converts shares to assets
    public fun withdraw<T: key>(
        store: Object<T>,
        amount: u64,  // this is "assets" amount
        transfer_ref: &TransferRef,
    ): FungibleAsset acquires RebasingConfig {
        // amount in = assets requested
        // convert to shares
        let config = borrow_global<RebasingConfig>(@learning);
        let shares = amount * config.total_shares as u64 / config.total_assets as u64;
        
        // Withdraw the shares
        fungible_asset::withdraw_with_ref(transfer_ref, store, shares)
    }
    
    fun primary_fungible_store_init(constructor_ref: &ConstructorRef) {
        use std::option;
        use std::string;
        use aptos_framework::primary_fungible_store;
        
        primary_fungible_store::create_primary_store_enabled_fungible_asset(
            constructor_ref,
            option::none(),
            string::utf8(b"Rebasing Token"),
            string::utf8(b"rbTKN"),
            18,
            string::utf8(b""),
            string::utf8(b""),
        );
    }
    
    use std::option;
}
```

---

## ตัวอย่าง: ERC-20 Style Token

```move
module learning::erc20_token {
    use std::string::{Self, String};
    use std::option;
    use std::signer;
    use aptos_framework::object::{Self, Object};
    use aptos_framework::fungible_asset::{Self, Metadata, MintRef, BurnRef, TransferRef};
    use aptos_framework::primary_fungible_store;
    use aptos_framework::event;
    
    // ============================================
    // Token Definition
    // ============================================
    
    struct TokenAdmin has key {
        metadata: Object<Metadata>,
        mint_ref: MintRef,
        burn_ref: BurnRef,
        transfer_ref: TransferRef,
        owner: address,
        paused: bool,
    }
    
    // ============================================
    // Events
    // ============================================
    
    #[event]
    struct TransferEvent has drop, store {
        from: address,
        to: address,
        amount: u64,
    }
    
    #[event]
    struct MintEvent has drop, store {
        to: address,
        amount: u64,
    }
    
    #[event]
    struct BurnEvent has drop, store {
        from: address,
        amount: u64,
    }
    
    // ============================================
    // Error codes
    // ============================================
    
    const E_NOT_OWNER: u64 = 1;
    const E_PAUSED: u64 = 2;
    const E_ZERO_AMOUNT: u64 = 3;
    const E_FROZEN: u64 = 4;
    const E_ALREADY_INITIALIZED: u64 = 5;
    
    // ============================================
    // Initialize
    // ============================================
    
    public entry fun initialize(
        creator: &signer,
        name: String,
        symbol: String,
        decimals: u8,
        initial_supply: u64,
    ) {
        let creator_addr = signer::address_of(creator);
        assert!(!exists<TokenAdmin>(creator_addr), E_ALREADY_INITIALIZED);
        
        // Create object for this token
        let seed = *string::bytes(&symbol);
        let constructor_ref = object::create_named_object(creator, seed);
        
        // Initialize as primary-store-enabled FA
        primary_fungible_store::create_primary_store_enabled_fungible_asset(
            &constructor_ref,
            option::none(),  // no max supply
            name,
            symbol,
            decimals,
            string::utf8(b""),
            string::utf8(b""),
        );
        
        let mint_ref = fungible_asset::generate_mint_ref(&constructor_ref);
        let burn_ref = fungible_asset::generate_burn_ref(&constructor_ref);
        let transfer_ref = fungible_asset::generate_transfer_ref(&constructor_ref);
        let metadata = object::object_from_constructor_ref::<Metadata>(&constructor_ref);
        
        // Mint initial supply to creator
        if (initial_supply > 0) {
            let fa = fungible_asset::mint(&mint_ref, initial_supply);
            primary_fungible_store::deposit(creator_addr, fa);
            
            event::emit(MintEvent { to: creator_addr, amount: initial_supply });
        };
        
        move_to(creator, TokenAdmin {
            metadata,
            mint_ref,
            burn_ref,
            transfer_ref,
            owner: creator_addr,
            paused: false,
        });
    }
    
    // ============================================
    // User functions
    // ============================================
    
    public entry fun transfer(
        from: &signer,
        to: address,
        creator_addr: address,
        amount: u64,
    ) acquires TokenAdmin {
        assert!(amount > 0, E_ZERO_AMOUNT);
        let admin = borrow_global<TokenAdmin>(creator_addr);
        assert!(!admin.paused, E_PAUSED);
        
        let from_addr = signer::address_of(from);
        primary_fungible_store::transfer(from, admin.metadata, to, amount);
        
        event::emit(TransferEvent { from: from_addr, to, amount });
    }
    
    public entry fun burn_my_tokens(
        from: &signer,
        creator_addr: address,
        amount: u64,
    ) acquires TokenAdmin {
        assert!(amount > 0, E_ZERO_AMOUNT);
        let admin = borrow_global<TokenAdmin>(creator_addr);
        let from_addr = signer::address_of(from);
        
        let fa = primary_fungible_store::withdraw(from, admin.metadata, amount);
        fungible_asset::burn(&admin.burn_ref, fa);
        
        event::emit(BurnEvent { from: from_addr, amount });
    }
    
    // ============================================
    // Admin functions
    // ============================================
    
    public entry fun mint(
        owner: &signer,
        creator_addr: address,
        to: address,
        amount: u64,
    ) acquires TokenAdmin {
        let admin = borrow_global<TokenAdmin>(creator_addr);
        assert!(signer::address_of(owner) == admin.owner, E_NOT_OWNER);
        assert!(amount > 0, E_ZERO_AMOUNT);
        
        let fa = fungible_asset::mint(&admin.mint_ref, amount);
        primary_fungible_store::deposit(to, fa);
        
        event::emit(MintEvent { to, amount });
    }
    
    public entry fun pause(owner: &signer, creator_addr: address) acquires TokenAdmin {
        let admin = borrow_global_mut<TokenAdmin>(creator_addr);
        assert!(signer::address_of(owner) == admin.owner, E_NOT_OWNER);
        admin.paused = true;
    }
    
    public entry fun unpause(owner: &signer, creator_addr: address) acquires TokenAdmin {
        let admin = borrow_global_mut<TokenAdmin>(creator_addr);
        assert!(signer::address_of(owner) == admin.owner, E_NOT_OWNER);
        admin.paused = false;
    }
    
    public entry fun freeze_user(
        owner: &signer,
        creator_addr: address,
        user: address,
    ) acquires TokenAdmin {
        let admin = borrow_global<TokenAdmin>(creator_addr);
        assert!(signer::address_of(owner) == admin.owner, E_NOT_OWNER);
        
        let store = primary_fungible_store::ensure_primary_store_exists(
            user, admin.metadata
        );
        fungible_asset::set_frozen_flag(&admin.transfer_ref, store, true);
    }
    
    public entry fun unfreeze_user(
        owner: &signer,
        creator_addr: address,
        user: address,
    ) acquires TokenAdmin {
        let admin = borrow_global<TokenAdmin>(creator_addr);
        assert!(signer::address_of(owner) == admin.owner, E_NOT_OWNER);
        
        let store = primary_fungible_store::ensure_primary_store_exists(
            user, admin.metadata
        );
        fungible_asset::set_frozen_flag(&admin.transfer_ref, store, false);
    }
    
    // ============================================
    // View functions
    // ============================================
    
    #[view]
    public fun balance_of(addr: address, creator_addr: address): u64 acquires TokenAdmin {
        let admin = borrow_global<TokenAdmin>(creator_addr);
        if (!primary_fungible_store::primary_store_exists(addr, admin.metadata)) {
            return 0
        };
        primary_fungible_store::balance(addr, admin.metadata)
    }
    
    #[view]
    public fun total_supply(creator_addr: address): u128 acquires TokenAdmin {
        let admin = borrow_global<TokenAdmin>(creator_addr);
        let supply_opt = fungible_asset::supply(admin.metadata);
        if (option::is_some(&supply_opt)) {
            *option::borrow(&supply_opt)
        } else {
            0u128
        }
    }
    
    #[view]
    public fun is_paused(creator_addr: address): bool acquires TokenAdmin {
        borrow_global<TokenAdmin>(creator_addr).paused
    }
    
    #[view]  
    public fun is_user_frozen(addr: address, creator_addr: address): bool acquires TokenAdmin {
        let admin = borrow_global<TokenAdmin>(creator_addr);
        if (!primary_fungible_store::primary_store_exists(addr, admin.metadata)) {
            return false
        };
        primary_fungible_store::is_frozen(addr, admin.metadata)
    }
    
    #[view]
    public fun get_metadata(creator_addr: address): Object<Metadata> acquires TokenAdmin {
        borrow_global<TokenAdmin>(creator_addr).metadata
    }
    
    // ============================================
    // Tests
    // ============================================
    
    #[test_only]
    use aptos_framework::account;
    
    #[test(framework = @aptos_framework, creator = @0x100, user = @0x200)]
    public fun test_full_lifecycle(
        framework: &signer,
        creator: &signer,
        user: &signer,
    ) acquires TokenAdmin {
        // Setup
        timestamp::set_time_has_started_for_testing(framework);
        account::create_account_for_test(signer::address_of(creator));
        account::create_account_for_test(signer::address_of(user));
        
        let creator_addr = signer::address_of(creator);
        let user_addr = signer::address_of(user);
        
        // Initialize with 1000 tokens
        initialize(
            creator,
            string::utf8(b"Test Token"),
            string::utf8(b"TST"),
            6,
            1_000_000_000,  // 1000 TST (6 decimals)
        );
        
        // Check initial balance
        assert!(balance_of(creator_addr, creator_addr) == 1_000_000_000, 1);
        assert!(total_supply(creator_addr) == 1_000_000_000u128, 2);
        
        // Transfer to user
        transfer(creator, user_addr, creator_addr, 100_000_000);
        assert!(balance_of(user_addr, creator_addr) == 100_000_000, 3);
        
        // Mint more
        mint(creator, creator_addr, creator_addr, 500_000_000);
        assert!(total_supply(creator_addr) == 1_500_000_000u128, 4);
        
        // User burns their tokens
        burn_my_tokens(user, creator_addr, 50_000_000);
        assert!(balance_of(user_addr, creator_addr) == 50_000_000, 5);
    }
    
    #[test_only]
    use aptos_framework::timestamp;
}
```

---

## เปรียบเทียบ Coin vs Fungible Asset

| Feature | Coin Standard | Fungible Asset |
|---------|--------------|----------------|
| Metadata | In code only | On-chain Object |
| Stores | CoinStore per type | Primary + secondary |
| Freeze | Via FreezeCapability | Via TransferRef |
| Hooks | None | Dispatchable functions |
| Supply tracking | Optional | Built-in |
| Composability | Limited | Full |
| Recommended | Legacy | Yes (new projects) |

---

**ก่อนหน้า**: [Part 21 - Aptos Framework ←](part-21-aptos-framework.md)
**ต่อไป**: [Part 23 - Aptos Objects →](part-23-aptos-objects.md)
