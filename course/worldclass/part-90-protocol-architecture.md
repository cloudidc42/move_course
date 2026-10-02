# Part 90: Advanced Protocol Architecture Patterns

## สารบัญ
- [Upgradeable Architecture](#upgradeable-architecture)
- [Diamond Pattern in Move](#diamond-pattern-in-move)
- [Multi-Sig Admin Framework](#multi-sig-admin-framework)
- [Plugin System](#plugin-system)
- [Cross-Module Communication](#cross-module-communication)
- [Protocol Versioning](#protocol-versioning)

---

## Upgradeable Architecture

```
MOVE UPGRADE CONSTRAINTS

Aptos Move Upgrade Policy:
  IMMUTABLE:      Cannot upgrade (locked forever)
  COMPATIBLE:     Can add new functions/structs, cannot remove/change existing
  ARBITRARY:      Can change anything (dangerous: requires governance approval)

Upgrade Strategies:
  1. Proxy Pattern:     Logic in upgradeable module, state in stable module
  2. Data Migration:    Move state to new struct format on upgrade
  3. Version Field:     Store version number, route to correct handler
  4. Plugin Registry:   Core stable + swappable plugin modules

PUBLISH FLOW
  1. Compile with compatible policy
  2. Submit upgrade proposal (governance)
  3. Timelock period (48h minimum)
  4. Execute upgrade after timelock
  5. Verify correctness on testnet first

KEY RULE:
  Cannot remove/rename structs or functions once published (COMPATIBLE mode)
  Cannot change function signatures
  Can ADD new public functions
  Can ADD new struct fields (with default values)
  Can CHANGE function implementations (same signature)
```

```move
// ============================================
// PROXY PATTERN: Separate logic from storage
// ============================================

// STABLE MODULE: Holds all state (never needs upgrade)
module protocol::state {
    friend protocol::logic_v1;
    friend protocol::logic_v2;  // Future logic module gets access
    
    struct ProtocolState has key {
        reserve_x: u64,
        reserve_y: u64,
        total_lp: u64,
        fee_bps: u64,
        version: u64,
        // Add fields here as protocol evolves (with defaults in migration)
        accumulated_fees_x: u64,  // Added in v2
        accumulated_fees_y: u64,  // Added in v2
    }
    
    public(friend) fun get_reserves(state: &ProtocolState): (u64, u64) {
        (state.reserve_x, state.reserve_y)
    }
    
    public(friend) fun set_reserves(state: &mut ProtocolState, x: u64, y: u64) {
        state.reserve_x = x;
        state.reserve_y = y;
    }
    
    public(friend) fun get_version(state: &ProtocolState): u64 {
        state.version
    }
    
    // Migration: Called during upgrade to v2
    public(friend) fun migrate_to_v2(state: &mut ProtocolState) {
        assert!(state.version == 1, 100);
        state.accumulated_fees_x = 0;
        state.accumulated_fees_y = 0;
        state.version = 2;
    }
}

// LOGIC V1: First implementation
module protocol::logic_v1 {
    use protocol::state;
    
    public fun swap(
        state_addr: address,
        amount_in: u64,
        is_x_to_y: bool,
    ): u64 {
        let state = borrow_global_mut<state::ProtocolState>(state_addr);
        assert!(state::get_version(state) == 1, 1);
        
        let (reserve_x, reserve_y) = state::get_reserves(state);
        
        let amount_out = calculate_out_v1(amount_in, reserve_x, reserve_y, is_x_to_y);
        
        if (is_x_to_y) {
            state::set_reserves(state, reserve_x + amount_in, reserve_y - amount_out);
        } else {
            state::set_reserves(state, reserve_x - amount_out, reserve_y + amount_in);
        };
        
        amount_out
    }
    
    fun calculate_out_v1(amount_in: u64, rx: u64, ry: u64, is_x_to_y: bool): u64 {
        let (res_in, res_out) = if (is_x_to_y) (rx, ry) else (ry, rx);
        let ain_fee = (amount_in as u128) * 9970;
        let num = ain_fee * (res_out as u128);
        let den = (res_in as u128) * 10000 + ain_fee;
        (num / den) as u64
    }
}

// LOGIC V2: Upgraded with fee accumulation
// This module is uploaded during upgrade; state module unchanged
module protocol::logic_v2 {
    use protocol::state;
    
    public fun swap(
        state_addr: address,
        amount_in: u64,
        is_x_to_y: bool,
    ): u64 {
        let state = borrow_global_mut<state::ProtocolState>(state_addr);
        assert!(state::get_version(state) == 2, 1);
        
        let (reserve_x, reserve_y) = state::get_reserves(state);
        
        // V2: Track fees separately
        let fee = amount_in * 30 / 10_000;
        let amount_after_fee = amount_in - fee;
        let amount_out = calculate_out_v2(amount_after_fee, reserve_x, reserve_y, is_x_to_y);
        
        // TODO: accumulate fees to state fields
        let _ = (fee, amount_out, reserve_x, reserve_y, is_x_to_y);
        amount_out
    }
    
    fun calculate_out_v2(amount_in: u64, rx: u64, ry: u64, is_x_to_y: bool): u64 {
        let (res_in, res_out) = if (is_x_to_y) (rx, ry) else (ry, rx);
        let num = (amount_in as u128) * (res_out as u128);
        let den = (res_in as u128) + (amount_in as u128);
        (num / den) as u64
    }
}
```

---

## Diamond Pattern in Move

```move
// ============================================
// DIAMOND / FACET PATTERN
// Single entry point, route to specialized facets
// ============================================

module protocol::router {
    use protocol::swap_facet;
    use protocol::liquidity_facet;
    use protocol::governance_facet;
    use protocol::oracle_facet;
    
    // Single entry point for all protocol actions
    // Routes based on action type
    public entry fun execute(
        caller: &signer,
        action: u8,
        data: vector<u8>,  // BCS-encoded action-specific params
    ) {
        if (action == 0) {
            let (amount_in, min_out, is_x_to_y) = decode_swap_params(&data);
            swap_facet::execute_swap(caller, amount_in, min_out, is_x_to_y);
        } else if (action == 1) {
            let (amount_x, amount_y) = decode_liquidity_params(&data);
            liquidity_facet::add_liquidity(caller, amount_x, amount_y);
        } else if (action == 2) {
            let proposal_id = decode_u64(&data, 0);
            let vote = decode_bool(&data, 8);
            governance_facet::cast_vote(caller, proposal_id, vote);
        } else {
            abort 404  // Unknown action
        }
    }
    
    fun decode_swap_params(data: &vector<u8>): (u64, u64, bool) {
        let amount_in = decode_u64(data, 0);
        let min_out = decode_u64(data, 8);
        let is_x_to_y = *vector::borrow(data, 16) != 0;
        (amount_in, min_out, is_x_to_y)
    }
    
    fun decode_liquidity_params(data: &vector<u8>): (u64, u64) {
        let amount_x = decode_u64(data, 0);
        let amount_y = decode_u64(data, 8);
        (amount_x, amount_y)
    }
    
    fun decode_u64(data: &vector<u8>, offset: u64): u64 {
        let result = 0u64;
        let i = 0u64;
        while (i < 8) {
            result = result | ((*vector::borrow(data, offset + i) as u64) << ((i * 8) as u8));
            i = i + 1;
        };
        result
    }
    
    fun decode_bool(data: &vector<u8>, offset: u64): bool {
        *vector::borrow(data, offset) != 0
    }
}

// SWAP FACET: Only swap logic
module protocol::swap_facet {
    use protocol::shared_state;
    
    struct SwapConfig has key {
        fee_bps: u64,
        max_slippage_bps: u64,
    }
    
    public(friend) fun execute_swap(
        caller: &signer,
        amount_in: u64,
        min_amount_out: u64,
        is_x_to_y: bool,
    ) {
        let config = borrow_global<SwapConfig>(@protocol);
        let _ = (caller, amount_in, min_amount_out, is_x_to_y, config);
    }
}

// LIQUIDITY FACET: Only LP logic
module protocol::liquidity_facet {
    public(friend) fun add_liquidity(
        caller: &signer,
        amount_x: u64,
        amount_y: u64,
    ) {
        let _ = (caller, amount_x, amount_y);
    }
}

module protocol::governance_facet {
    public(friend) fun cast_vote(caller: &signer, proposal_id: u64, vote: bool) {
        let _ = (caller, proposal_id, vote);
    }
}

module protocol::shared_state {}
module protocol::oracle_facet {}
```

---

## Multi-Sig Admin Framework

```move
// ============================================
// MULTI-SIG ADMIN: N-of-M approval for ops
// ============================================

module protocol::multisig {
    use aptos_framework::timestamp;
    use std::vector;
    
    const MAX_SIGNERS: u64 = 10;
    const ERR_NOT_SIGNER: u64 = 1;
    const ERR_ALREADY_APPROVED: u64 = 2;
    const ERR_INSUFFICIENT_APPROVALS: u64 = 3;
    const ERR_EXPIRED: u64 = 4;
    const ERR_ALREADY_EXECUTED: u64 = 5;
    const ERR_INVALID_THRESHOLD: u64 = 6;
    
    struct MultisigWallet has key {
        signers: vector<address>,
        threshold: u64,           // Min approvals required
        proposal_count: u64,
    }
    
    struct Proposal has key {
        id: u64,
        proposer: address,
        action_type: u8,
        action_data: vector<u8>,  // BCS-encoded action params
        approvals: vector<address>,
        rejections: vector<address>,
        created_at: u64,
        expiry: u64,              // Proposal expires if not executed
        executed: bool,
        cancelled: bool,
    }
    
    public fun create_wallet(
        admin: &signer,
        signers: vector<address>,
        threshold: u64,
    ) {
        let n = vector::length(&signers);
        assert!(threshold > 0 && threshold <= n, ERR_INVALID_THRESHOLD);
        assert!(n <= MAX_SIGNERS, ERR_INVALID_THRESHOLD);
        
        move_to(admin, MultisigWallet {
            signers,
            threshold,
            proposal_count: 0,
        });
    }
    
    public entry fun propose(
        proposer: &signer,
        wallet_addr: address,
        action_type: u8,
        action_data: vector<u8>,
        expiry_hours: u64,
    ) {
        let wallet = borrow_global_mut<MultisigWallet>(wallet_addr);
        let proposer_addr = std::signer::address_of(proposer);
        
        assert!(vector::contains(&wallet.signers, &proposer_addr), ERR_NOT_SIGNER);
        
        let proposal_id = wallet.proposal_count;
        wallet.proposal_count = wallet.proposal_count + 1;
        
        let now = timestamp::now_microseconds();
        
        // Proposer auto-approves
        let approvals = vector::singleton(proposer_addr);
        
        move_to(proposer, Proposal {
            id: proposal_id,
            proposer: proposer_addr,
            action_type,
            action_data,
            approvals,
            rejections: vector::empty(),
            created_at: now,
            expiry: now + expiry_hours * 3_600_000_000,
            executed: false,
            cancelled: false,
        });
    }
    
    public entry fun approve(
        signer: &signer,
        proposal_addr: address,
        wallet_addr: address,
    ) {
        let wallet = borrow_global<MultisigWallet>(wallet_addr);
        let signer_addr = std::signer::address_of(signer);
        let proposal = borrow_global_mut<Proposal>(proposal_addr);
        
        assert!(vector::contains(&wallet.signers, &signer_addr), ERR_NOT_SIGNER);
        assert!(!vector::contains(&proposal.approvals, &signer_addr), ERR_ALREADY_APPROVED);
        assert!(!proposal.executed && !proposal.cancelled, ERR_ALREADY_EXECUTED);
        assert!(timestamp::now_microseconds() < proposal.expiry, ERR_EXPIRED);
        
        vector::push_back(&mut proposal.approvals, signer_addr);
    }
    
    public entry fun execute(
        executor: &signer,
        proposal_addr: address,
        wallet_addr: address,
    ) {
        let wallet = borrow_global<MultisigWallet>(wallet_addr);
        let executor_addr = std::signer::address_of(executor);
        let proposal = borrow_global_mut<Proposal>(proposal_addr);
        
        assert!(vector::contains(&wallet.signers, &executor_addr), ERR_NOT_SIGNER);
        assert!(!proposal.executed, ERR_ALREADY_EXECUTED);
        assert!(!proposal.cancelled, ERR_ALREADY_EXECUTED);
        assert!(timestamp::now_microseconds() < proposal.expiry, ERR_EXPIRED);
        assert!(
            vector::length(&proposal.approvals) >= wallet.threshold,
            ERR_INSUFFICIENT_APPROVALS,
        );
        
        proposal.executed = true;
        
        // Route to action handler
        dispatch_action(proposal.action_type, &proposal.action_data, wallet_addr);
    }
    
    fun dispatch_action(action_type: u8, data: &vector<u8>, wallet_addr: address) {
        // Route based on action type
        if (action_type == 0) {
            // Update fee
        } else if (action_type == 1) {
            // Pause/unpause
        } else if (action_type == 2) {
            // Transfer admin
        };
        let _ = (data, wallet_addr);
    }
    
    #[view]
    public fun get_approval_count(proposal_addr: address): (u64, u64) {
        let proposal = borrow_global<Proposal>(proposal_addr);
        let wallet = borrow_global<MultisigWallet>(@protocol);
        (vector::length(&proposal.approvals), wallet.threshold)
    }
    
    #[view]
    public fun can_execute(proposal_addr: address, wallet_addr: address): bool {
        let proposal = borrow_global<Proposal>(proposal_addr);
        let wallet = borrow_global<MultisigWallet>(wallet_addr);
        
        !proposal.executed
        && !proposal.cancelled
        && timestamp::now_microseconds() < proposal.expiry
        && vector::length(&proposal.approvals) >= wallet.threshold
    }
}
```

---

## Plugin System

```move
// ============================================
// PLUGIN / HOOK SYSTEM
// Allow external modules to extend protocol
// ============================================

module protocol::hooks {
    
    /// Hook types that external modules can register for
    const HOOK_PRE_SWAP: u8 = 0;
    const HOOK_POST_SWAP: u8 = 1;
    const HOOK_PRE_LIQUIDITY: u8 = 2;
    const HOOK_POST_LIQUIDITY: u8 = 3;
    
    /// Registered hook: a module address + function identifier
    struct HookEntry has store, copy, drop {
        module_addr: address,
        hook_type: u8,
        priority: u8,    // Lower = runs first
        enabled: bool,
    }
    
    struct HookRegistry has key {
        hooks: vector<HookEntry>,
        admin: address,
    }
    
    public fun initialize(admin: &signer) {
        move_to(admin, HookRegistry {
            hooks: vector::empty(),
            admin: std::signer::address_of(admin),
        });
    }
    
    public entry fun register_hook(
        admin: &signer,
        registry_addr: address,
        module_addr: address,
        hook_type: u8,
        priority: u8,
    ) {
        let registry = borrow_global_mut<HookRegistry>(registry_addr);
        assert!(
            std::signer::address_of(admin) == registry.admin,
            1,
        );
        
        // Insert sorted by priority
        let entry = HookEntry { module_addr, hook_type, priority, enabled: true };
        
        let n = vector::length(&registry.hooks);
        let insert_pos = n;
        let i = 0u64;
        
        while (i < n) {
            let h = vector::borrow(&registry.hooks, i);
            if (h.hook_type == hook_type && h.priority > priority) {
                insert_pos = i;
                break
            };
            i = i + 1;
        };
        
        if (insert_pos == n) {
            vector::push_back(&mut registry.hooks, entry);
        } else {
            // Insert at position (Move doesn't have insert, so we rebuild)
            vector::push_back(&mut registry.hooks, entry);
            // Shift elements right (simplified)
        };
    }
    
    public fun disable_hook(
        admin: &signer,
        registry: &mut HookRegistry,
        module_addr: address,
        hook_type: u8,
    ) {
        assert!(std::signer::address_of(admin) == registry.admin, 1);
        
        let n = vector::length(&registry.hooks);
        let i = 0u64;
        while (i < n) {
            let hook = vector::borrow_mut(&mut registry.hooks, i);
            if (hook.module_addr == module_addr && hook.hook_type == hook_type) {
                hook.enabled = false;
                break
            };
            i = i + 1;
        };
    }
    
    // Protocol calls this before/after each swap
    // Hooks can modify context or abort the transaction
    public fun run_hooks(
        registry: &HookRegistry,
        hook_type: u8,
        amount_in: u64,
        amount_out: u64,
        user: address,
    ): (u64, u64) {
        // Returns potentially modified (amount_in, amount_out)
        // Hooks could add fees, apply limits, etc.
        let result_in = amount_in;
        let result_out = amount_out;
        
        let n = vector::length(&registry.hooks);
        let i = 0u64;
        while (i < n) {
            let hook = vector::borrow(&registry.hooks, i);
            if (hook.hook_type == hook_type && hook.enabled) {
                // Call external hook (simplified - real impl needs capability pattern)
                let _ = (hook.module_addr, user);
                // result = hook::run(hook.module_addr, result_in, result_out)
            };
            i = i + 1;
        };
        
        (result_in, result_out)
    }
}
```

---

## Cross-Module Communication

```move
// ============================================
// CAPABILITY PATTERN: Safe cross-module calls
// ============================================

module protocol::capabilities {
    
    /// Capability to call privileged operations
    /// Only one exists, held by the protocol admin
    struct AdminCap has key, store {}
    
    /// Capability passed to authorized callers
    struct SwapCap has drop {}
    struct LiquidityCap has drop {}
    struct OracleCap has drop {}
    
    /// Grant a temporary swap capability
    /// Only admin can call this
    public fun grant_swap_cap(_admin_cap: &AdminCap): SwapCap {
        SwapCap {}
    }
    
    public fun grant_liquidity_cap(_admin_cap: &AdminCap): LiquidityCap {
        LiquidityCap {}
    }
    
    /// Modules check for capability instead of address
    public fun can_swap(_cap: &SwapCap): bool { true }
    public fun can_add_liquidity(_cap: &LiquidityCap): bool { true }
}

// ============================================
// WITNESS PATTERN: Type-safe module authentication
// ============================================

// Module A defines a witness type
module module_a::auth {
    struct Witness {}  // Only module_a can construct this
    
    // Other modules call this to get a witness
    public fun get_witness(): Witness { Witness {} }
}

// Module B uses the witness for type-safe authentication
module module_b::protected {
    use module_a::auth;
    
    struct ProtectedStore has key {
        value: u64,
    }
    
    // Only callable by someone who holds a witness from module_a
    public fun protected_update(
        _witness: auth::Witness,  // Proves caller got it from module_a
        store: &mut ProtectedStore,
        new_value: u64,
    ) {
        store.value = new_value;
        // Witness is consumed here (has drop, no store)
    }
}

// ============================================
// EVENT-DRIVEN ARCHITECTURE: Decouple modules
// ============================================

module protocol::events {
    use aptos_framework::event;
    
    #[event]
    struct SwapCompleted has drop, store {
        user: address,
        pool: address,
        amount_in: u64,
        amount_out: u64,
        is_x_to_y: bool,
        timestamp: u64,
    }
    
    #[event]
    struct LiquidityChanged has drop, store {
        user: address,
        pool: address,
        amount_x: u64,
        amount_y: u64,
        lp_delta: i64,  // Positive = add, negative = remove
        timestamp: u64,
    }
    
    public fun emit_swap(
        user: address,
        pool: address,
        amount_in: u64,
        amount_out: u64,
        is_x_to_y: bool,
    ) {
        event::emit(SwapCompleted {
            user, pool, amount_in, amount_out, is_x_to_y,
            timestamp: aptos_framework::timestamp::now_microseconds(),
        });
    }
}
```

---

## Protocol Versioning

```move
// ============================================
// PROTOCOL VERSIONING: Graceful migration
// ============================================

module protocol::version_manager {
    use aptos_framework::timestamp;
    
    const ERR_WRONG_VERSION: u64 = 1;
    const ERR_MIGRATION_PENDING: u64 = 2;
    const ERR_TOO_EARLY: u64 = 3;
    
    struct VersionConfig has key {
        current_version: u64,
        min_supported_version: u64,
        migration_deadline: u64,     // Users must migrate by this time
        upgrade_admin: address,
    }
    
    /// Check that a call is from a compatible version
    public fun assert_version_compatible(
        config: &VersionConfig,
        caller_version: u64,
    ) {
        assert!(caller_version >= config.min_supported_version, ERR_WRONG_VERSION);
        assert!(caller_version <= config.current_version, ERR_WRONG_VERSION);
    }
    
    /// Migrate user from old version to current
    public fun migrate_user(
        user: &signer,
        config: &VersionConfig,
        from_version: u64,
    ) {
        assert!(from_version < config.current_version, ERR_WRONG_VERSION);
        
        // Version-specific migration steps
        if (from_version == 1) {
            migrate_v1_to_v2(user);
        };
        if (from_version <= 2) {
            migrate_v2_to_v3(user);
        };
        // Each migration is idempotent
    }
    
    fun migrate_v1_to_v2(user: &signer) {
        // Transform v1 data structures to v2 format
        let _ = user;
    }
    
    fun migrate_v2_to_v3(user: &signer) {
        let _ = user;
    }
    
    /// Force migration: reject old-version calls after deadline
    public fun require_migrated(config: &VersionConfig) {
        let now = timestamp::now_microseconds();
        if (now > config.migration_deadline) {
            assert!(false, ERR_MIGRATION_PENDING);
        };
    }
}
```

---

## สรุป Protocol Architecture Patterns

```
ARCHITECTURE DECISION MATRIX

Pattern           | Upgradeable? | Gas Cost | Complexity | Use When
------------------|--------------|----------|------------|----------
Monolithic        | Limited      | Low      | Low        | Simple protocols
Proxy/Logic Split | High         | Medium   | Medium     | DeFi protocols
Diamond/Facets    | High         | Medium   | High       | Large systems
Plugin/Hooks      | Very High    | High     | High       | Ecosystem platforms

SECURITY CONSIDERATIONS

Upgrade Risks:
  ✅ Use timelocks (48h minimum for mainnet)
  ✅ Multi-sig for upgrade authority
  ✅ Upgrade proposals visible on-chain (transparency)
  ✅ Test migrations on testnet/devnet first
  ❌ Never upgrade logic + data in same tx (two-phase)

Capability Risks:
  ✅ Capabilities are drop-only (can't be stored by attacker)
  ✅ One capability per privilege level
  ✅ Revoke by not issuing new capabilities
  ❌ Never store capabilities in global state

Cross-Module Risks:
  ✅ Use witness pattern for type-safe auth
  ✅ friend declarations limit access
  ✅ Event-driven decoupling (no direct calls)
  ❌ Never use raw address checks for auth

UPGRADE CHECKLIST
  □ Compile with --upgrade-policy compatible
  □ Verify all existing function signatures preserved
  □ Verify all existing struct fields preserved
  □ New fields have sensible defaults
  □ Migration function is idempotent
  □ Test on devnet first
  □ Governance proposal with 48h timelock
  □ Announce migration timeline to users
  □ Monitor after upgrade (24h close watch)
```

---

**ก่อนหน้า**: [Part 89 - Gas Optimization ←](part-89-gas-optimization.md)
**ต่อไป**: [Part 91 - Yield Aggregator Strategies →](part-91-yield-aggregator.md)
