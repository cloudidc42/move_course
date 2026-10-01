# Part 32: Upgrade Patterns & Contract Versioning

## สารบัญ
- [ทำไมต้อง Upgrade](#ทำไมต้อง-upgrade)
- [Aptos Upgrade Mechanism](#aptos-upgrade-mechanism)
- [Data Migration Patterns](#data-migration-patterns)
- [Feature Flags](#feature-flags)
- [Versioned State Pattern](#versioned-state-pattern)
- [ตัวอย่าง: Upgradeable Protocol](#ตัวอย่าง-upgradeable-protocol)

---

## ทำไมต้อง Upgrade

```
Move Contract Lifecycle:
  Deploy → Use → Bug Found → Need Fix → ???

Aptos: สามารถ upgrade bytecode ได้ (ถ้าตั้งค่าไว้)
Sui: แต่ละ object มี version field ที่เพิ่มได้
Move: State (resources) ยังคงอยู่ หลัง upgrade

ข้อจำกัด:
  - ไม่สามารถ delete existing resource fields
  - ไม่สามารถเปลี่ยน type ของ field
  - เพิ่ม field ใหม่ได้ แต่ต้อง backward compatible
```

---

## Aptos Upgrade Mechanism

```move
// Move.toml - กำหนด upgrade policy
[package]
name = "MyProtocol"
version = "2.0.0"

[addresses]
my_protocol = "0xCAFE"

// Upgrade policies (set at deploy time):
// - immutable: ไม่สามารถ upgrade ได้
// - compatible: เพิ่ม function/struct ได้ แต่ไม่ลบ
// - arbitrary: เปลี่ยนได้ทุกอย่าง (ต้อง admin แน่นอน)
```

```move
module my_protocol::versioning {
    use std::signer;
    
    // ============================================
    // Version tracking ใน state
    // ============================================
    
    struct GlobalConfig has key {
        version: u64,
        admin: address,
        // v1 fields
        fee_bps: u64,
        // v2 fields (added in upgrade)
        max_slippage_bps: u64,
        emergency_pause: bool,
    }
    
    const VERSION_1: u64 = 1;
    const VERSION_2: u64 = 2;
    const CURRENT_VERSION: u64 = 2;
    
    const E_VERSION_MISMATCH: u64 = 1;
    const E_NOT_ADMIN: u64 = 2;
    
    // ============================================
    // Initialize (v1)
    // ============================================
    
    public entry fun initialize_v1(admin: &signer, fee_bps: u64) {
        let addr = signer::address_of(admin);
        move_to(admin, GlobalConfig {
            version: VERSION_1,
            admin: addr,
            fee_bps,
            // default values for v2 fields
            max_slippage_bps: 500,  // 5% default
            emergency_pause: false,
        });
    }
    
    // ============================================
    // Migrate to v2
    // ============================================
    
    public entry fun migrate_to_v2(
        admin: &signer,
        config_addr: address,
        max_slippage_bps: u64,
    ) acquires GlobalConfig {
        let config = borrow_global_mut<GlobalConfig>(config_addr);
        assert!(signer::address_of(admin) == config.admin, E_NOT_ADMIN);
        assert!(config.version == VERSION_1, E_VERSION_MISMATCH);
        
        config.version = VERSION_2;
        config.max_slippage_bps = max_slippage_bps;
        // emergency_pause was initialized to false, no change needed
    }
    
    // ============================================
    // Version-gated operations
    // ============================================
    
    public fun require_version(config_addr: address, min_version: u64) acquires GlobalConfig {
        let config = borrow_global<GlobalConfig>(config_addr);
        assert!(config.version >= min_version, E_VERSION_MISMATCH);
    }
    
    // v2-only feature
    public fun check_slippage(
        config_addr: address,
        actual_slippage_bps: u64,
    ) acquires GlobalConfig {
        let config = borrow_global<GlobalConfig>(config_addr);
        assert!(config.version >= VERSION_2, E_VERSION_MISMATCH);
        assert!(actual_slippage_bps <= config.max_slippage_bps, 100);
    }
    
    #[view]
    public fun get_version(config_addr: address): u64 acquires GlobalConfig {
        borrow_global<GlobalConfig>(config_addr).version
    }
}
```

---

## Data Migration Patterns

```move
module my_protocol::data_migration {
    use std::signer;
    use aptos_std::table::{Self, Table};
    
    // ============================================
    // Lazy Migration Pattern
    // แต่ละ record migrate ตอน access
    // ============================================
    
    // Old format (v1)
    struct UserDataV1 has store {
        balance: u64,
        last_update: u64,
    }
    
    // New format (v2)  
    struct UserDataV2 has store {
        balance: u64,
        last_update: u64,
        // new fields
        rewards_claimed: u64,
        tier: u8,
    }
    
    struct UserRegistry has key {
        version: u64,
        // v1 data (kept for migration)
        old_data: Table<address, UserDataV1>,
        // v2 data
        new_data: Table<address, UserDataV2>,
        migrated_count: u64,
        total_count: u64,
    }
    
    // ============================================
    // Lazy migrate on access
    // ============================================
    
    fun ensure_migrated(registry: &mut UserRegistry, user: address) {
        // If already in new_data, done
        if (table::contains(&registry.new_data, user)) return;
        
        // If in old_data, migrate
        if (table::contains(&registry.old_data, user)) {
            let old = table::remove(&mut registry.old_data, user);
            let tier = calculate_tier(old.balance);
            table::add(&mut registry.new_data, user, UserDataV2 {
                balance: old.balance,
                last_update: old.last_update,
                rewards_claimed: 0,
                tier,
            });
            registry.migrated_count = registry.migrated_count + 1;
        };
        // else: new user, will be created fresh
    }
    
    fun calculate_tier(balance: u64): u8 {
        if (balance >= 1_000_000_000) 3      // Platinum: ≥1000 APT
        else if (balance >= 100_000_000) 2   // Gold: ≥100 APT
        else if (balance >= 10_000_000) 1    // Silver: ≥10 APT
        else 0                               // Bronze
    }
    
    public fun get_user_data(
        registry_addr: address,
        user: address,
    ): UserDataV2 acquires UserRegistry {
        let registry = borrow_global_mut<UserRegistry>(registry_addr);
        ensure_migrated(registry, user);
        
        if (!table::contains(&registry.new_data, user)) {
            // New user default
            return UserDataV2 {
                balance: 0,
                last_update: 0,
                rewards_claimed: 0,
                tier: 0,
            }
        };
        *table::borrow(&registry.new_data, user)
    }
    
    // ============================================
    // Batch Migration Pattern
    // Admin migrates in batches to avoid gas limits
    // ============================================
    
    struct MigrationTracker has key {
        users: vector<address>,
        next_index: u64,
        completed: bool,
    }
    
    public entry fun batch_migrate(
        admin: &signer,
        registry_addr: address,
        tracker_addr: address,
        batch_size: u64,
    ) acquires UserRegistry, MigrationTracker {
        let tracker = borrow_global_mut<MigrationTracker>(tracker_addr);
        let registry = borrow_global_mut<UserRegistry>(registry_addr);
        
        let start = tracker.next_index;
        let end = start + batch_size;
        let total = std::vector::length(&tracker.users);
        
        if (end > total) { end = total; };
        
        let mut i = start;
        while (i < end) {
            let user = *std::vector::borrow(&tracker.users, i);
            ensure_migrated(registry, user);
            i = i + 1;
        };
        
        tracker.next_index = end;
        tracker.completed = (end >= total);
    }
}
```

---

## Feature Flags

```move
module my_protocol::feature_flags {
    use std::signer;
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // Feature Flag System
    // Control which features are active
    // ============================================
    
    struct FeatureFlags has key {
        admin: address,
        flags: SmartTable<vector<u8>, bool>,
    }
    
    // Feature names (use constants to avoid typos)
    const FEATURE_V2_SWAP: vector<u8> = b"v2_swap";
    const FEATURE_FLASH_LOANS: vector<u8> = b"flash_loans";
    const FEATURE_LIMIT_ORDERS: vector<u8> = b"limit_orders";
    const FEATURE_NFT_STAKING: vector<u8> = b"nft_staking";
    
    const E_NOT_ADMIN: u64 = 1;
    const E_FEATURE_DISABLED: u64 = 2;
    
    public entry fun initialize(admin: &signer) {
        move_to(admin, FeatureFlags {
            admin: signer::address_of(admin),
            flags: smart_table::new(),
        });
    }
    
    public entry fun set_flag(
        admin: &signer,
        flags_addr: address,
        feature: vector<u8>,
        enabled: bool,
    ) acquires FeatureFlags {
        let flags = borrow_global_mut<FeatureFlags>(flags_addr);
        assert!(signer::address_of(admin) == flags.admin, E_NOT_ADMIN);
        
        if (smart_table::contains(&flags.flags, feature)) {
            *smart_table::borrow_mut(&mut flags.flags, feature) = enabled;
        } else {
            smart_table::add(&mut flags.flags, feature, enabled);
        };
    }
    
    public fun require_feature(flags_addr: address, feature: vector<u8>) acquires FeatureFlags {
        let flags = borrow_global<FeatureFlags>(flags_addr);
        let enabled = if (smart_table::contains(&flags.flags, feature)) {
            *smart_table::borrow(&flags.flags, feature)
        } else {
            false
        };
        assert!(enabled, E_FEATURE_DISABLED);
    }
    
    #[view]
    public fun is_enabled(flags_addr: address, feature: vector<u8>): bool acquires FeatureFlags {
        let flags = borrow_global<FeatureFlags>(flags_addr);
        if (!smart_table::contains(&flags.flags, feature)) return false;
        *smart_table::borrow(&flags.flags, feature)
    }
    
    // ============================================
    // Usage in other modules
    // ============================================
    
    // In your module:
    // fun my_v2_swap(...) {
    //     feature_flags::require_feature(FLAGS_ADDR, feature_flags::FEATURE_V2_SWAP);
    //     // ... v2 logic
    // }
}
```

---

## Versioned State Pattern

```move
module my_protocol::versioned_state {
    use std::signer;
    use aptos_framework::event;
    
    // ============================================
    // Use enum-like approach for state versions
    // ============================================
    
    struct ProtocolStateV1 has key {
        fee_bps: u64,
        total_volume: u64,
    }
    
    struct ProtocolStateV2 has key {
        fee_bps: u64,
        total_volume: u64,
        // v2 additions
        protocol_fee_bps: u64,
        fee_recipient: address,
        total_fees_collected: u64,
    }
    
    struct ProtocolStateV3 has key {
        fee_bps: u64,
        total_volume: u64,
        protocol_fee_bps: u64,
        fee_recipient: address,
        total_fees_collected: u64,
        // v3 additions
        is_paused: bool,
        governance: address,
        min_trade_size: u64,
    }
    
    struct VersionPointer has key {
        current_version: u64,
    }
    
    #[event]
    struct Upgraded has drop, store {
        from_version: u64,
        to_version: u64,
        timestamp: u64,
    }
    
    // ============================================
    // Upgrade functions
    // ============================================
    
    public entry fun upgrade_v1_to_v2(
        admin: &signer,
        protocol_addr: address,
        fee_recipient: address,
        protocol_fee_bps: u64,
    ) acquires ProtocolStateV1, VersionPointer {
        let pointer = borrow_global_mut<VersionPointer>(protocol_addr);
        assert!(pointer.current_version == 1, 1);
        
        let ProtocolStateV1 { fee_bps, total_volume } = 
            move_from<ProtocolStateV1>(protocol_addr);
        
        // Get admin's signer to move new state to protocol_addr
        // In practice: use resource account or admin address == protocol_addr
        move_to(admin, ProtocolStateV2 {
            fee_bps,
            total_volume,
            protocol_fee_bps,
            fee_recipient,
            total_fees_collected: 0,
        });
        
        pointer.current_version = 2;
        
        event::emit(Upgraded {
            from_version: 1,
            to_version: 2,
            timestamp: aptos_framework::timestamp::now_seconds(),
        });
    }
    
    // ============================================
    // Generic reader that works across versions
    // ============================================
    
    #[view]
    public fun get_fee_bps(protocol_addr: address): u64 acquires 
        VersionPointer, ProtocolStateV1, ProtocolStateV2, ProtocolStateV3 {
        let pointer = borrow_global<VersionPointer>(protocol_addr);
        if (pointer.current_version == 1) {
            borrow_global<ProtocolStateV1>(protocol_addr).fee_bps
        } else if (pointer.current_version == 2) {
            borrow_global<ProtocolStateV2>(protocol_addr).fee_bps
        } else {
            borrow_global<ProtocolStateV3>(protocol_addr).fee_bps
        }
    }
    
    #[view]
    public fun get_version(protocol_addr: address): u64 acquires VersionPointer {
        borrow_global<VersionPointer>(protocol_addr).current_version
    }
}
```

---

## ตัวอย่าง: Upgradeable Protocol

```move
module my_protocol::upgradeable_dex {
    use std::signer;
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::event;
    
    // ============================================
    // Full upgradeable DEX protocol
    // ============================================
    
    struct DexState has key {
        version: u64,
        admin: address,
        paused: bool,
        
        // Core state
        fee_bps: u64,
        
        // v2 state (upgrading from here)
        protocol_fee_bps: u64,
        treasury: address,
        
        // v3 state
        governance: address,
        min_liquidity: u64,
        
        // stats
        total_pools: u64,
        total_volume_usd: u128,
    }
    
    struct Pool<phantom X, phantom Y> has key {
        version: u64,
        reserve_x: Coin<X>,
        reserve_y: Coin<Y>,
        lp_supply: u64,
        fee_bps: u64,
        // v2: per-pool fees
        protocol_fee_bps: u64,
        accumulated_fee_x: u64,
        accumulated_fee_y: u64,
    }
    
    #[event]
    struct PoolCreated has drop, store {
        pool_addr: address,
        token_x: string::String,
        token_y: string::String,
        version: u64,
    }
    
    #[event]
    struct ProtocolUpgraded has drop, store {
        from_version: u64,
        to_version: u64,
        upgrader: address,
    }
    
    use std::string;
    
    const VERSION_1: u64 = 1;
    const VERSION_2: u64 = 2;  
    const VERSION_3: u64 = 3;
    const CURRENT_VERSION: u64 = 3;
    
    const E_NOT_ADMIN: u64 = 1;
    const E_PAUSED: u64 = 2;
    const E_WRONG_VERSION: u64 = 3;
    const E_OUTDATED_POOL: u64 = 4;
    
    // ============================================
    // Initialize
    // ============================================
    
    public entry fun initialize(
        admin: &signer,
        fee_bps: u64,
        treasury: address,
        governance: address,
    ) {
        let addr = signer::address_of(admin);
        move_to(admin, DexState {
            version: CURRENT_VERSION,
            admin: addr,
            paused: false,
            fee_bps,
            protocol_fee_bps: 5,  // 0.05%
            treasury,
            governance,
            min_liquidity: 1000,
            total_pools: 0,
            total_volume_usd: 0,
        });
    }
    
    // ============================================
    // Upgrade checks
    // ============================================
    
    fun assert_not_paused(state: &DexState) {
        assert!(!state.paused, E_PAUSED);
    }
    
    fun assert_admin(admin: &signer, state: &DexState) {
        assert!(signer::address_of(admin) == state.admin, E_NOT_ADMIN);
    }
    
    // Ensure pool is on current version before major ops
    fun assert_pool_current<X, Y>(pool: &Pool<X, Y>, dex_version: u64) {
        // Allow pools within 1 major version
        assert!(pool.version >= dex_version - 1, E_OUTDATED_POOL);
    }
    
    // ============================================
    // Migrate pool to latest version
    // ============================================
    
    public entry fun migrate_pool<X, Y>(
        admin: &signer,
        dex_addr: address,
        pool_addr: address,
    ) acquires DexState, Pool<X, Y> {
        let state = borrow_global<DexState>(dex_addr);
        assert_admin(admin, state);
        
        let pool = borrow_global_mut<Pool<X, Y>>(pool_addr);
        
        if (pool.version < VERSION_2) {
            // Migrate v1 → v2: add protocol fees
            pool.protocol_fee_bps = state.protocol_fee_bps;
            pool.accumulated_fee_x = 0;
            pool.accumulated_fee_y = 0;
        };
        
        pool.version = state.version;
    }
    
    // ============================================
    // Emergency controls
    // ============================================
    
    public entry fun pause(admin: &signer, dex_addr: address) acquires DexState {
        let state = borrow_global_mut<DexState>(dex_addr);
        assert_admin(admin, state);
        state.paused = true;
    }
    
    public entry fun unpause(admin: &signer, dex_addr: address) acquires DexState {
        let state = borrow_global_mut<DexState>(dex_addr);
        assert_admin(admin, state);
        state.paused = false;
    }
    
    // ============================================
    // Admin parameter updates (post-upgrade)
    // ============================================
    
    public entry fun update_fees(
        admin: &signer,
        dex_addr: address,
        new_fee_bps: u64,
        new_protocol_fee_bps: u64,
    ) acquires DexState {
        let state = borrow_global_mut<DexState>(dex_addr);
        assert_admin(admin, state);
        assert!(new_fee_bps <= 1000, 100);         // Max 10%
        assert!(new_protocol_fee_bps <= 100, 101); // Max 1%
        state.fee_bps = new_fee_bps;
        state.protocol_fee_bps = new_protocol_fee_bps;
    }
    
    public entry fun transfer_admin(
        admin: &signer,
        dex_addr: address,
        new_admin: address,
    ) acquires DexState {
        let state = borrow_global_mut<DexState>(dex_addr);
        assert_admin(admin, state);
        state.admin = new_admin;
    }
    
    // ============================================
    // Timelock for critical upgrades
    // ============================================
    
    struct UpgradeProposal has key {
        proposer: address,
        new_params: UpgradeParams,
        proposed_at: u64,
        executable_at: u64,
        executed: bool,
    }
    
    struct UpgradeParams has store {
        new_fee_bps: u64,
        new_treasury: address,
    }
    
    const TIMELOCK_DURATION: u64 = 172800;  // 2 days
    
    public entry fun propose_upgrade(
        admin: &signer,
        dex_addr: address,
        new_fee_bps: u64,
        new_treasury: address,
    ) acquires DexState {
        let state = borrow_global<DexState>(dex_addr);
        assert_admin(admin, state);
        
        let now = aptos_framework::timestamp::now_seconds();
        move_to(admin, UpgradeProposal {
            proposer: signer::address_of(admin),
            new_params: UpgradeParams { new_fee_bps, new_treasury },
            proposed_at: now,
            executable_at: now + TIMELOCK_DURATION,
            executed: false,
        });
    }
    
    public entry fun execute_upgrade(
        admin: &signer,
        dex_addr: address,
        proposal_addr: address,
    ) acquires DexState, UpgradeProposal {
        let proposal = borrow_global_mut<UpgradeProposal>(proposal_addr);
        let now = aptos_framework::timestamp::now_seconds();
        
        assert!(!proposal.executed, 200);
        assert!(now >= proposal.executable_at, 201);
        
        let state = borrow_global_mut<DexState>(dex_addr);
        assert_admin(admin, state);
        
        state.fee_bps = proposal.new_params.new_fee_bps;
        state.treasury = proposal.new_params.new_treasury;
        proposal.executed = true;
        
        event::emit(ProtocolUpgraded {
            from_version: state.version,
            to_version: state.version,
            upgrader: signer::address_of(admin),
        });
    }
    
    // ============================================
    // View state
    // ============================================
    
    #[view]
    public fun get_state(dex_addr: address): (u64, bool, u64) acquires DexState {
        let s = borrow_global<DexState>(dex_addr);
        (s.version, s.paused, s.fee_bps)
    }
}
```

---

## สรุป Upgrade Strategies

```
Strategy              | Pros                    | Cons
---------------------|-------------------------|--------------------
Lazy Migration        | No downtime, gradual    | Complex read logic
Batch Migration       | Clean state             | Requires downtime window
Feature Flags         | Safe rollout            | Code complexity
Versioned State       | Clear history           | Multiple struct types
Timelock              | Security, transparency  | Slow to deploy fixes
```

**Best Practices:**
1. Version ทุก struct ตั้งแต่แรก
2. ใช้ Feature Flags สำหรับ risky features
3. Timelock สำหรับ parameter changes ที่สำคัญ
4. Test migration scripts บน testnet ก่อนเสมอ
5. Keep old struct definitions จนกว่าจะ migrate 100%

---

**ก่อนหน้า**: [Part 31 - Advanced Patterns ←](part-31-advanced-patterns.md)
**ต่อไป**: [Part 33 - Oracle Integration →](part-33-oracle-integration.md)
