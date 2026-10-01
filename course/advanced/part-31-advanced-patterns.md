# Part 31: Advanced Move Patterns

## สารบัญ
- [Capability Pattern](#capability-pattern)
- [Witness Pattern](#witness-pattern)
- [Proxy Pattern](#proxy-pattern)
- [Two-Phase Commit](#two-phase-commit)
- [Observer Pattern](#observer-pattern)
- [ตัวอย่าง: Plugin System](#ตัวอย่าง-plugin-system)

---

## Capability Pattern

```move
module advanced::capability_pattern {
    use std::signer;
    use aptos_framework::event;
    
    // ============================================
    // Capability = a proof of permission
    // Stored as a resource, non-copyable
    // ============================================
    
    // Admin capability - master control
    struct AdminCap has key {
        id: u64,  // unique to prevent confusion
    }
    
    // Minter capability - can create tokens
    struct MinterCap has key {
        token_type: u8,
        max_per_tx: u64,
    }
    
    // Pauser capability - can pause operations
    struct PauserCap has key {}
    
    // Upgrader capability - can upgrade contract
    struct UpgraderCap has key {
        version: u64,
    }
    
    struct ProtocolState has key {
        is_paused: bool,
        version: u64,
        admin: address,
    }
    
    // ============================================
    // Initialize: give caps to deployer
    // ============================================
    
    public entry fun initialize(deployer: &signer) {
        let addr = signer::address_of(deployer);
        
        move_to(deployer, AdminCap { id: 1 });
        move_to(deployer, MinterCap { token_type: 0, max_per_tx: 1_000_000 });
        move_to(deployer, PauserCap {});
        move_to(deployer, UpgraderCap { version: 1 });
        
        move_to(deployer, ProtocolState {
            is_paused: false,
            version: 1,
            admin: addr,
        });
    }
    
    // ============================================
    // Delegate capabilities to sub-addresses
    // ============================================
    
    // Admin can grant Minter cap to another address
    public entry fun grant_minter(
        admin: &signer,
        admin_addr: address,
        recipient: &signer,
        max_per_tx: u64,
    ) acquires AdminCap {
        // Verify caller has AdminCap
        assert!(exists<AdminCap>(signer::address_of(admin)), 1);
        assert!(signer::address_of(admin) == admin_addr, 2);
        
        // Borrow to verify (doesn't consume)
        let _cap = borrow_global<AdminCap>(admin_addr);
        
        // Grant MinterCap to recipient
        move_to(recipient, MinterCap {
            token_type: 0,
            max_per_tx,
        });
    }
    
    // ============================================
    // Revoke a capability
    // ============================================
    
    public entry fun revoke_minter(
        admin: &signer,
        admin_addr: address,
        minter_addr: address,
    ) acquires AdminCap, MinterCap {
        assert!(exists<AdminCap>(signer::address_of(admin)), 1);
        
        let MinterCap { token_type: _, max_per_tx: _ } = 
            move_from<MinterCap>(minter_addr);
    }
    
    // ============================================
    // Capability-gated operations
    // ============================================
    
    public entry fun mint_tokens(
        minter: &signer,
        minter_addr: address,
        amount: u64,
    ) acquires MinterCap, ProtocolState {
        let cap = borrow_global<MinterCap>(minter_addr);
        assert!(signer::address_of(minter) == minter_addr, 3);
        assert!(amount <= cap.max_per_tx, 4);
        
        // Check not paused
        let state = borrow_global<ProtocolState>(@advanced);
        assert!(!state.is_paused, 5);
        
        // ... actual minting logic
    }
    
    public entry fun pause(pauser: &signer, pauser_addr: address) 
    acquires PauserCap, ProtocolState {
        assert!(exists<PauserCap>(signer::address_of(pauser)), 6);
        let state = borrow_global_mut<ProtocolState>(@advanced);
        state.is_paused = true;
    }
    
    // ============================================
    // Transferable capability (with signer)
    // ============================================
    
    // Capability that can be passed to another module as parameter
    // (not stored as resource, just passed by reference)
    
    struct TemporaryAuth has drop {
        user: address,
        expires_at: u64,
        scope: u8,
    }
    
    public fun create_temp_auth(
        admin: &signer,
        admin_addr: address,
        user: address,
        scope: u8,
        duration: u64,
    ): TemporaryAuth acquires AdminCap {
        assert!(exists<AdminCap>(signer::address_of(admin)), 1);
        TemporaryAuth {
            user,
            expires_at: aptos_framework::timestamp::now_seconds() + duration,
            scope,
        }
    }
    
    public fun use_temp_auth(
        auth: &TemporaryAuth,
        user: address,
        required_scope: u8,
    ) {
        assert!(auth.user == user, 7);
        assert!(auth.scope >= required_scope, 8);
        let now = aptos_framework::timestamp::now_seconds();
        assert!(now <= auth.expires_at, 9);
    }
}
```

---

## Witness Pattern

```move
module advanced::witness_pattern {
    // ============================================
    // Witness = a proof that you are a specific module
    // One-time-witness (OTW): same name as module, has drop
    // ============================================
    
    // Used for:
    // - Guaranteeing only one instance of something per module
    // - Type-level proof of module identity
    // - Coin/FA initialization (prevent clones)
    
    // ============================================
    // One-Time-Witness pattern
    // ============================================
    
    struct MY_TOKEN has drop {}  // OTW: same name as module in caps
    
    // OTW is passed as argument to init()
    // Only the module itself can create the OTW (given by runtime)
    
    struct TokenRegistry has key {
        token_count: u64,
    }
    
    public fun register_with_witness(
        witness: MY_TOKEN,  // proves we're the MY_TOKEN module
        admin: &signer,
    ) {
        // Witness is consumed (has drop, but we don't need to keep it)
        let _ = witness;
        
        move_to(admin, TokenRegistry { token_count: 1 });
    }
    
    // ============================================
    // Regular Witness pattern (proof of type)
    // ============================================
    
    // Used to prove generic T is from a specific module
    
    struct Witness<phantom T> has drop {}
    
    // Protocol accepts any token type if its module provides a Witness
    struct Protocol has key {
        registered_count: u64,
    }
    
    public fun register_token<T>(_witness: Witness<T>, protocol_addr: address) 
    acquires Protocol {
        let protocol = borrow_global_mut<Protocol>(protocol_addr);
        protocol.registered_count = protocol.registered_count + 1;
    }
    
    // Only the TOKEN module can create Witness<TOKEN>
    // because it's a private struct within that module
    
    // ============================================
    // Phantom type witness
    // ============================================
    
    // Different modules create different phantom types
    // These can only be created by their respective modules
    
    struct PoolType<phantom X, phantom Y> has copy, drop, store {}
    
    // Creates a unique type for each token pair
    public fun pool_type<X, Y>(): PoolType<X, Y> {
        PoolType<X, Y> {}
    }
    
    // Check if a pool exists for a pair
    public fun pool_exists<X, Y>(pool_addr: address): bool {
        exists<PoolType<X, Y>>(pool_addr)
    }
}
```

---

## Proxy Pattern

```move
module advanced::proxy_pattern {
    use std::signer;
    use aptos_framework::account;
    
    // ============================================
    // Proxy = Resource Account with SignerCapability
    // ============================================
    
    // Use cases:
    // - Protocol treasury (holds assets on behalf of protocol)
    // - Escrow account
    // - Multisig proxy
    
    struct ProxyConfig has key {
        signer_cap: account::SignerCapability,
        proxy_addr: address,
        owner: address,
        operators: vector<address>,
    }
    
    struct ProxyOperation has drop {
        nonce: u64,
        target: address,
        data: vector<u8>,
    }
    
    // ============================================
    // Create proxy (resource account)
    // ============================================
    
    public entry fun create_proxy(
        owner: &signer,
        seed: vector<u8>,
        initial_operators: vector<address>,
    ) {
        let (proxy_signer, cap) = account::create_resource_account(owner, seed);
        let proxy_addr = signer::address_of(&proxy_signer);
        let owner_addr = signer::address_of(owner);
        
        move_to(owner, ProxyConfig {
            signer_cap: cap,
            proxy_addr,
            owner: owner_addr,
            operators: initial_operators,
        });
    }
    
    // ============================================
    // Execute via proxy
    // ============================================
    
    // Get proxy signer to use in other operations
    public fun get_proxy_signer(owner_addr: address): signer acquires ProxyConfig {
        let config = borrow_global<ProxyConfig>(owner_addr);
        account::create_signer_with_capability(&config.signer_cap)
    }
    
    public entry fun operator_action(
        caller: &signer,
        proxy_owner: address,
    ) acquires ProxyConfig {
        let config = borrow_global<ProxyConfig>(proxy_owner);
        let caller_addr = signer::address_of(caller);
        
        // Must be operator or owner
        let is_authorized = caller_addr == config.owner ||
            std::vector::contains(&config.operators, &caller_addr);
        assert!(is_authorized, 1);
        
        // Get proxy signer
        let _proxy_signer = account::create_signer_with_capability(&config.signer_cap);
        
        // Use proxy_signer to perform actions on behalf of proxy account
        // e.g., transfer assets, call other modules
    }
    
    // ============================================
    // Multisig proxy
    // ============================================
    
    struct MultiSigProxy has key {
        signer_cap: account::SignerCapability,
        signers: vector<address>,
        threshold: u64,  // min signatures required
        pending_ops: aptos_std::table::Table<u64, PendingOp>,
        next_op_id: u64,
    }
    
    struct PendingOp has store {
        proposer: address,
        signatures: vector<address>,
        data: vector<u8>,
        executed: bool,
    }
    
    public entry fun propose_operation(
        caller: &signer,
        proxy_addr: address,
        op_data: vector<u8>,
    ) acquires MultiSigProxy {
        use aptos_std::table;
        let proxy = borrow_global_mut<MultiSigProxy>(proxy_addr);
        let caller_addr = signer::address_of(caller);
        
        assert!(std::vector::contains(&proxy.signers, &caller_addr), 2);
        
        let op_id = proxy.next_op_id;
        proxy.next_op_id = op_id + 1;
        
        let mut sigs = std::vector::empty<address>();
        std::vector::push_back(&mut sigs, caller_addr);
        
        table::add(&mut proxy.pending_ops, op_id, PendingOp {
            proposer: caller_addr,
            signatures: sigs,
            data: op_data,
            executed: false,
        });
    }
    
    public entry fun approve_operation(
        caller: &signer,
        proxy_addr: address,
        op_id: u64,
    ) acquires MultiSigProxy {
        use aptos_std::table;
        let proxy = borrow_global_mut<MultiSigProxy>(proxy_addr);
        let caller_addr = signer::address_of(caller);
        
        assert!(std::vector::contains(&proxy.signers, &caller_addr), 2);
        
        let op = table::borrow_mut(&mut proxy.pending_ops, op_id);
        assert!(!op.executed, 3);
        assert!(!std::vector::contains(&op.signatures, &caller_addr), 4);
        
        std::vector::push_back(&mut op.signatures, caller_addr);
        
        // Execute if threshold reached
        if (std::vector::length(&op.signatures) >= proxy.threshold) {
            op.executed = true;
            let _proxy_signer = account::create_signer_with_capability(&proxy.signer_cap);
            // Execute op.data using proxy_signer
        };
    }
}
```

---

## Two-Phase Commit

```move
module advanced::two_phase {
    use std::signer;
    use aptos_framework::timestamp;
    
    // ============================================
    // Two-Phase Commit for atomic cross-module ops
    // ============================================
    
    // Phase 1: Lock/Reserve resources
    // Phase 2: Complete or Rollback
    
    struct PendingTransfer has key {
        from: address,
        to: address,
        amount: u64,
        asset_type: u8,
        locked_at: u64,
        expires_at: u64,
        status: u8,  // 0=pending, 1=committed, 2=rolled_back
    }
    
    const STATUS_PENDING: u8 = 0;
    const STATUS_COMMITTED: u8 = 1;
    const STATUS_ROLLED_BACK: u8 = 2;
    
    const LOCK_TIMEOUT: u64 = 300;  // 5 minutes
    
    const E_NOT_FOUND: u64 = 1;
    const E_EXPIRED: u64 = 2;
    const E_NOT_AUTHORIZED: u64 = 3;
    const E_WRONG_STATUS: u64 = 4;
    
    // ============================================
    // Phase 1: Prepare (lock resources)
    // ============================================
    
    public entry fun prepare_transfer(
        sender: &signer,
        transfer_id: u64,
        to: address,
        amount: u64,
        asset_type: u8,
    ) {
        let from = signer::address_of(sender);
        let now = timestamp::now_seconds();
        
        // Lock the funds (in production: actually move to escrow)
        // Here we just record the intent
        
        move_to(sender, PendingTransfer {
            from,
            to,
            amount,
            asset_type,
            locked_at: now,
            expires_at: now + LOCK_TIMEOUT,
            status: STATUS_PENDING,
        });
        
        // In production: withdraw from sender, hold in escrow
    }
    
    // ============================================
    // Phase 2a: Commit
    // ============================================
    
    public entry fun commit_transfer(
        sender: &signer,
        from: address,
    ) acquires PendingTransfer {
        let pending = borrow_global_mut<PendingTransfer>(from);
        
        assert!(pending.status == STATUS_PENDING, E_WRONG_STATUS);
        assert!(timestamp::now_seconds() <= pending.expires_at, E_EXPIRED);
        
        pending.status = STATUS_COMMITTED;
        
        // In production: release from escrow to recipient
    }
    
    // ============================================
    // Phase 2b: Rollback (on timeout or failure)
    // ============================================
    
    public entry fun rollback_transfer(from: address) acquires PendingTransfer {
        let pending = borrow_global_mut<PendingTransfer>(from);
        
        assert!(pending.status == STATUS_PENDING, E_WRONG_STATUS);
        
        // Can rollback if: expired, or called by sender
        let now = timestamp::now_seconds();
        assert!(now > pending.expires_at, E_NOT_AUTHORIZED);
        
        pending.status = STATUS_ROLLED_BACK;
        
        // In production: return funds from escrow to sender
    }
}
```

---

## Observer Pattern

```move
module advanced::observer_pattern {
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::event;
    
    // ============================================
    // Observer = callbacks on state change
    // ============================================
    
    // In Move: implement via events (best practice)
    // Or via registered callbacks (for on-chain processing)
    
    struct PriceOracle has key {
        prices: SmartTable<vector<u8>, u64>,  // symbol -> price
        observers: vector<address>,  // addresses to notify
        update_count: u64,
    }
    
    #[event]
    struct PriceUpdated has drop, store {
        symbol: vector<u8>,
        old_price: u64,
        new_price: u64,
        deviation_bps: u64,
    }
    
    #[event]
    struct LargeDeviationAlert has drop, store {
        symbol: vector<u8>,
        deviation_bps: u64,
        threshold_bps: u64,
    }
    
    const ALERT_THRESHOLD_BPS: u64 = 500;  // 5% = alert
    
    public entry fun update_price(
        oracle_addr: address,
        symbol: vector<u8>,
        new_price: u64,
    ) acquires PriceOracle {
        let oracle = borrow_global_mut<PriceOracle>(oracle_addr);
        
        let old_price = if (smart_table::contains(&oracle.prices, symbol)) {
            *smart_table::borrow(&oracle.prices, symbol)
        } else {
            0
        };
        
        // Calculate deviation
        let deviation_bps = if (old_price > 0) {
            let diff = if (new_price > old_price) {
                new_price - old_price
            } else {
                old_price - new_price
            };
            diff * 10_000 / old_price
        } else {
            0
        };
        
        // Update price
        if (smart_table::contains(&oracle.prices, symbol)) {
            *smart_table::borrow_mut(&mut oracle.prices, symbol) = new_price;
        } else {
            smart_table::add(&mut oracle.prices, symbol, new_price);
        };
        oracle.update_count = oracle.update_count + 1;
        
        // Emit standard event
        event::emit(PriceUpdated {
            symbol,
            old_price,
            new_price,
            deviation_bps,
        });
        
        // Emit alert if large deviation
        if (deviation_bps >= ALERT_THRESHOLD_BPS) {
            event::emit(LargeDeviationAlert {
                symbol,
                deviation_bps,
                threshold_bps: ALERT_THRESHOLD_BPS,
            });
        };
    }
    
    #[view]
    public fun get_price(oracle_addr: address, symbol: vector<u8>): u64 acquires PriceOracle {
        let oracle = borrow_global<PriceOracle>(oracle_addr);
        if (!smart_table::contains(&oracle.prices, symbol)) return 0;
        *smart_table::borrow(&oracle.prices, symbol)
    }
}
```

---

## ตัวอย่าง: Plugin System

```move
module advanced::plugin_system {
    use std::signer;
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::event;
    
    // ============================================
    // Plugin System using capabilities and generics
    // ============================================
    
    struct PluginRegistry has key {
        admin: address,
        plugins: SmartTable<vector<u8>, PluginInfo>,
        plugin_count: u64,
    }
    
    struct PluginInfo has copy, drop, store {
        name: vector<u8>,
        version: u64,
        author: address,
        enabled: bool,
        calls: u64,
    }
    
    // Capability granted to plugin authors
    struct PluginCap has key {
        plugin_name: vector<u8>,
        registry: address,
    }
    
    #[event]
    struct PluginRegistered has drop, store {
        name: vector<u8>,
        author: address,
        version: u64,
    }
    
    #[event]
    struct PluginCalled has drop, store {
        name: vector<u8>,
        caller: address,
        data: vector<u8>,
    }
    
    const E_NOT_ADMIN: u64 = 1;
    const E_PLUGIN_EXISTS: u64 = 2;
    const E_PLUGIN_NOT_FOUND: u64 = 3;
    const E_PLUGIN_DISABLED: u64 = 4;
    
    // ============================================
    // Initialize registry
    // ============================================
    
    public entry fun initialize_registry(admin: &signer) {
        move_to(admin, PluginRegistry {
            admin: signer::address_of(admin),
            plugins: smart_table::new(),
            plugin_count: 0,
        });
    }
    
    // ============================================
    // Register plugin
    // ============================================
    
    public entry fun register_plugin(
        admin: &signer,
        registry_addr: address,
        plugin_author: &signer,
        name: vector<u8>,
        version: u64,
    ) acquires PluginRegistry {
        let registry = borrow_global_mut<PluginRegistry>(registry_addr);
        assert!(signer::address_of(admin) == registry.admin, E_NOT_ADMIN);
        assert!(!smart_table::contains(&registry.plugins, name), E_PLUGIN_EXISTS);
        
        let author = signer::address_of(plugin_author);
        
        smart_table::add(&mut registry.plugins, name, PluginInfo {
            name,
            version,
            author,
            enabled: true,
            calls: 0,
        });
        registry.plugin_count = registry.plugin_count + 1;
        
        // Grant PluginCap to author
        move_to(plugin_author, PluginCap {
            plugin_name: name,
            registry: registry_addr,
        });
        
        event::emit(PluginRegistered { name, author, version });
    }
    
    // ============================================
    // Invoke plugin (tracks usage)
    // ============================================
    
    public fun invoke_plugin(
        caller: &signer,
        registry_addr: address,
        plugin_name: vector<u8>,
        data: vector<u8>,
    ) acquires PluginRegistry {
        let registry = borrow_global_mut<PluginRegistry>(registry_addr);
        assert!(smart_table::contains(&registry.plugins, plugin_name), E_PLUGIN_NOT_FOUND);
        
        let plugin = smart_table::borrow_mut(&mut registry.plugins, plugin_name);
        assert!(plugin.enabled, E_PLUGIN_DISABLED);
        
        plugin.calls = plugin.calls + 1;
        
        event::emit(PluginCalled {
            name: plugin_name,
            caller: signer::address_of(caller),
            data,
        });
    }
    
    // ============================================
    // Disable/enable plugin
    // ============================================
    
    public entry fun set_plugin_enabled(
        admin: &signer,
        registry_addr: address,
        plugin_name: vector<u8>,
        enabled: bool,
    ) acquires PluginRegistry {
        let registry = borrow_global_mut<PluginRegistry>(registry_addr);
        assert!(signer::address_of(admin) == registry.admin, E_NOT_ADMIN);
        
        let plugin = smart_table::borrow_mut(&mut registry.plugins, plugin_name);
        plugin.enabled = enabled;
    }
    
    // ============================================
    // View functions
    // ============================================
    
    #[view]
    public fun plugin_info(
        registry_addr: address,
        name: vector<u8>,
    ): PluginInfo acquires PluginRegistry {
        let registry = borrow_global<PluginRegistry>(registry_addr);
        *smart_table::borrow(&registry.plugins, name)
    }
    
    #[view]
    public fun plugin_count(registry_addr: address): u64 acquires PluginRegistry {
        borrow_global<PluginRegistry>(registry_addr).plugin_count
    }
}
```

---

## สรุป Advanced Patterns

| Pattern | When to Use | Key Mechanism |
|---------|-------------|---------------|
| Capability | Permission control | Resource stored at authorized address |
| Witness | Type-level proof | Struct with drop that only creator can make |
| Proxy | Indirect execution | Resource account + SignerCapability |
| Two-Phase | Atomic cross-module | Lock → Commit/Rollback |
| Observer | React to state changes | Events (preferred) or registered hooks |
| Plugin | Extensible systems | Registry + capabilities |

---

**ก่อนหน้า**: [Part 30 - Staking ←](../intermediate/part-30-staking-yield.md)
**ต่อไป**: [Part 32 - Upgrade Patterns →](part-32-upgrade-patterns.md)
