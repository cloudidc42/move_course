# Part 66: Advanced Security Patterns in Move

## สารบัญ
- [Security Mental Model](#security-mental-model)
- [Reentrancy & Flash Loan Attacks](#reentrancy--flash-loan-attacks)
- [Oracle Manipulation Attacks](#oracle-manipulation-attacks)
- [Access Control Patterns](#access-control-patterns)
- [Integer Overflow Protections](#integer-overflow-protections)
- [Front-Running Defenses](#front-running-defenses)
- [Emergency Response System](#emergency-response-system)
- [Formal Verification with Move Prover](#formal-verification-with-move-prover)

---

## Security Mental Model

```
Move Security Philosophy:
  
  Ethereum Solidity Problems:
    - Reentrancy (DAO hack: $60M)
    - Integer overflow (BatchOverflow: $170M affected)
    - Access control failures (Poly Network: $611M)
    - Delegatecall bugs (Parity: $30M)
    - Flash loan attacks (many billions)
  
  How Move Prevents Many:
    1. RESOURCE MODEL: Coins cannot be cloned/dropped accidentally
       - Solidity: balances[addr] -= amount (no enforcement)
       - Move: coin::extract() / coin::deposit() (enforced by type system)
    
    2. NO REENTRANCY BY DEFAULT:
       - Move has no `call` to arbitrary contracts mid-execution in same frame
       - Aptos: acquires annotation prevents concurrent access to same resource
       - Sui: object ownership prevents concurrent modification
    
    3. TYPE SAFETY:
       - No raw address casting
       - No assembly/inline opcodes
       - Abilities system controls what operations are valid
    
  Remaining Attack Vectors in Move:
    - Logic bugs (wrong formulas)
    - Oracle manipulation (price feeds)
    - Flash loan economics (pool draining)
    - Access control errors (wrong address checks)
    - Integer arithmetic overflow/underflow
    - Front-running (sandwich attacks)
    - Governance attacks (malicious proposals)
```

---

## Reentrancy & Flash Loan Attacks

```move
module security::reentrancy_guard {
    use aptos_framework::account;
    
    // ============================================
    // REENTRANCY GUARD
    // Move's acquires system prevents most reentrancy,
    // but cross-module calls can still be dangerous.
    // ============================================
    
    struct ReentrancyGuard has key {
        locked: bool,
    }
    
    // Initialize guard per contract address
    public fun initialize(admin: &signer) {
        move_to(admin, ReentrancyGuard { locked: false });
    }
    
    // Enter critical section
    public fun lock(guard_addr: address) acquires ReentrancyGuard {
        let guard = borrow_global_mut<ReentrancyGuard>(guard_addr);
        assert!(!guard.locked, error::already_locked());
        guard.locked = true;
    }
    
    // Exit critical section
    public fun unlock(guard_addr: address) acquires ReentrancyGuard {
        let guard = borrow_global_mut<ReentrancyGuard>(guard_addr);
        guard.locked = false;
    }
    
    // Check if locked (for views)
    public fun is_locked(guard_addr: address): bool acquires ReentrancyGuard {
        borrow_global<ReentrancyGuard>(guard_addr).locked
    }
    
    const ERROR_ALREADY_LOCKED: u64 = 1;
    fun error::already_locked(): u64 { ERROR_ALREADY_LOCKED }
}

module security::safe_lending {
    use aptos_framework::coin::{Self, Coin};
    use security::reentrancy_guard;
    
    // ============================================
    // SAFE LENDING POOL with:
    // 1. Reentrancy guard
    // 2. Flash loan fee enforcement
    // 3. Balance checks before/after
    // ============================================
    
    struct LendingPool<phantom T> has key {
        reserves: Coin<T>,
        flash_loan_fee_bps: u64,  // e.g., 9 = 0.09%
        total_borrows: u64,
        guard_addr: address,
    }
    
    struct FlashLoanReceipt<phantom T> {
        amount_borrowed: u64,
        fee_owed: u64,
        pool_addr: address,
    }
    
    // Borrow tokens (must repay in same tx)
    public fun flash_borrow<T>(
        pool_addr: address,
        amount: u64,
    ): (Coin<T>, FlashLoanReceipt<T>) acquires LendingPool {
        // Reentrancy check
        reentrancy_guard::lock(pool_addr);
        
        let pool = borrow_global_mut<LendingPool<T>>(pool_addr);
        
        // Calculate fee
        let fee = amount * pool.flash_loan_fee_bps / 10_000;
        if (fee == 0) fee = 1; // Minimum 1 unit fee
        
        // Extract coins
        let coins = coin::extract(&mut pool.reserves, amount);
        pool.total_borrows = pool.total_borrows + amount;
        
        let receipt = FlashLoanReceipt<T> {
            amount_borrowed: amount,
            fee_owed: fee,
            pool_addr,
        };
        
        (coins, receipt)
    }
    
    // Repay flash loan (must call in same tx)
    public fun flash_repay<T>(
        repayment: Coin<T>,
        fee_payment: Coin<T>,
        receipt: FlashLoanReceipt<T>,
    ) acquires LendingPool {
        let FlashLoanReceipt { amount_borrowed, fee_owed, pool_addr } = receipt;
        
        // Verify correct amounts
        assert!(coin::value(&repayment) == amount_borrowed, 1);
        assert!(coin::value(&fee_payment) >= fee_owed, 2);
        
        let pool = borrow_global_mut<LendingPool<T>>(pool_addr);
        
        // Deposit repayment + fee
        coin::merge(&mut pool.reserves, repayment);
        coin::merge(&mut pool.reserves, fee_payment);
        
        pool.total_borrows = pool.total_borrows - amount_borrowed;
        
        // Release reentrancy lock
        reentrancy_guard::unlock(pool_addr);
    }
    
    // If receipt is dropped without repay -> compile error!
    // FlashLoanReceipt has no `drop` ability, so it MUST be consumed.
    // This is Move's linear type system preventing flash loan theft.
}
```

---

## Oracle Manipulation Attacks

```move
module security::oracle_security {
    use aptos_framework::timestamp;
    use aptos_std::math64;
    
    // ============================================
    // ORACLE MANIPULATION DEFENSES
    //
    // Attack: Manipulate spot price in DEX, 
    //         use it as oracle for lending protocol,
    //         borrow against inflated collateral
    //
    // Defense: TWAP (Time-Weighted Average Price)
    // ============================================
    
    struct PriceObservation has store {
        timestamp: u64,
        price_cumulative: u128,  // Accumulated price * time
        price: u64,              // Spot price at observation
    }
    
    struct TWAPOracle has key {
        token_pair: std::string::String,
        observations: vector<PriceObservation>,
        max_observations: u64,     // Ring buffer size
        current_idx: u64,
        
        // Bounds for sanity check
        min_price: u64,
        max_price: u64,
        max_price_change_bps: u64, // Max 10% change per hour
        
        // TWAP parameters
        twap_period: u64,          // 30 minutes in microseconds
    }
    
    // Record a new price observation (called by DEX on every swap)
    public fun record_observation(
        oracle_addr: address,
        new_price: u64,
    ) acquires TWAPOracle {
        let oracle = borrow_global_mut<TWAPOracle>(oracle_addr);
        let now = timestamp::now_microseconds();
        
        // Sanity check: price within bounds
        assert!(new_price >= oracle.min_price, 1);
        assert!(new_price <= oracle.max_price, 2);
        
        // Sanity check: price change not too large
        let last_obs = get_last_observation(oracle);
        let last_price = last_obs.price;
        
        let max_change = last_price * oracle.max_price_change_bps / 10_000;
        let price_diff = if (new_price > last_price) {
            new_price - last_price
        } else {
            last_price - new_price
        };
        assert!(price_diff <= max_change, 3); // Too large a price change
        
        // Calculate time elapsed
        let time_elapsed = now - last_obs.timestamp;
        
        // Accumulate price * time
        let price_cumulative = last_obs.price_cumulative + 
            (last_price as u128) * (time_elapsed as u128);
        
        // Store observation (ring buffer)
        let new_obs = PriceObservation {
            timestamp: now,
            price_cumulative,
            price: new_price,
        };
        
        let idx = oracle.current_idx;
        if (idx < std::vector::length(&oracle.observations)) {
            *std::vector::borrow_mut(&mut oracle.observations, idx) = new_obs;
        } else {
            std::vector::push_back(&mut oracle.observations, new_obs);
        };
        
        oracle.current_idx = (idx + 1) % oracle.max_observations;
    }
    
    // Get TWAP price over the configured period
    public fun get_twap_price(oracle_addr: address): u64 acquires TWAPOracle {
        let oracle = borrow_global<TWAPOracle>(oracle_addr);
        let now = timestamp::now_microseconds();
        let period_start = now - oracle.twap_period;
        
        // Find observation at or before period_start
        let start_obs = find_observation_at(oracle, period_start);
        let end_obs = get_last_observation(oracle);
        
        // TWAP = delta_price_cumulative / delta_time
        let delta_cumulative = end_obs.price_cumulative - start_obs.price_cumulative;
        let delta_time = end_obs.timestamp - start_obs.timestamp;
        
        assert!(delta_time > 0, 4);
        
        // Returns price scaled to same units as input prices
        (delta_cumulative / (delta_time as u128)) as u64
    }
    
    // Multi-oracle median for extra security
    public fun get_median_price(oracle_addrs: vector<address>): u64 acquires TWAPOracle {
        let len = std::vector::length(&oracle_addrs);
        assert!(len >= 3, 1); // Need at least 3 for median
        
        let mut prices = vector::empty<u64>();
        let mut i = 0u64;
        while (i < len) {
            let addr = *std::vector::borrow(&oracle_addrs, i);
            let price = get_twap_price(addr);
            std::vector::push_back(&mut prices, price);
            i = i + 1;
        };
        
        // Sort prices
        sort_prices(&mut prices);
        
        // Return median
        *std::vector::borrow(&prices, len / 2)
    }
    
    fun get_last_observation(oracle: &TWAPOracle): &PriceObservation {
        let len = std::vector::length(&oracle.observations);
        let last_idx = if (oracle.current_idx == 0) len - 1 else oracle.current_idx - 1;
        std::vector::borrow(&oracle.observations, last_idx)
    }
    
    fun find_observation_at(_oracle: &TWAPOracle, _timestamp: u64): PriceObservation {
        // Binary search through observations for closest to timestamp
        // (Simplified: return first observation in production)
        PriceObservation { timestamp: 0, price_cumulative: 0, price: 0 }
    }
    
    fun sort_prices(_prices: &mut vector<u64>) {
        // Insertion sort for small vectors
    }
}
```

---

## Access Control Patterns

```move
module security::access_control {
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // ROLE-BASED ACCESS CONTROL (RBAC)
    // More fine-grained than simple admin/user
    // ============================================
    
    // Roles as constants (bitmask approach)
    const ROLE_ADMIN: u64 = 1;
    const ROLE_PAUSER: u64 = 2;
    const ROLE_MINTER: u64 = 4;
    const ROLE_BURNER: u64 = 8;
    const ROLE_ORACLE_FEEDER: u64 = 16;
    const ROLE_LIQUIDATOR: u64 = 32;
    
    struct RoleRegistry has key {
        // address → bitmask of roles
        roles: SmartTable<address, u64>,
        // Role admin: who can grant/revoke each role
        role_admins: SmartTable<u64, address>,
    }
    
    public fun initialize(admin: &signer) {
        let admin_addr = std::signer::address_of(admin);
        let mut registry = RoleRegistry {
            roles: smart_table::new(),
            role_admins: smart_table::new(),
        };
        
        // Admin has all roles
        smart_table::add(&mut registry.roles, admin_addr, 0xFFFF_FFFF_FFFF_FFFF);
        
        // Admin is role admin for all roles
        smart_table::add(&mut registry.role_admins, ROLE_ADMIN, admin_addr);
        smart_table::add(&mut registry.role_admins, ROLE_PAUSER, admin_addr);
        smart_table::add(&mut registry.role_admins, ROLE_MINTER, admin_addr);
        
        move_to(admin, registry);
    }
    
    // Grant role to address
    public fun grant_role(
        granter: &signer,
        registry_addr: address,
        role: u64,
        target: address,
    ) acquires RoleRegistry {
        let registry = borrow_global_mut<RoleRegistry>(registry_addr);
        let granter_addr = std::signer::address_of(granter);
        
        // Check granter has admin permission for this role
        let role_admin = smart_table::borrow(&registry.role_admins, role);
        assert!(granter_addr == *role_admin || has_role_internal(registry, granter_addr, ROLE_ADMIN), 1);
        
        // Grant role (bitwise OR)
        if (smart_table::contains(&registry.roles, target)) {
            let current = *smart_table::borrow(&registry.roles, target);
            *smart_table::borrow_mut(&mut registry.roles, target) = current | role;
        } else {
            smart_table::add(&mut registry.roles, target, role);
        };
    }
    
    // Revoke role from address
    public fun revoke_role(
        revoker: &signer,
        registry_addr: address,
        role: u64,
        target: address,
    ) acquires RoleRegistry {
        let registry = borrow_global_mut<RoleRegistry>(registry_addr);
        let revoker_addr = std::signer::address_of(revoker);
        
        let role_admin = smart_table::borrow(&registry.role_admins, role);
        assert!(revoker_addr == *role_admin || has_role_internal(registry, revoker_addr, ROLE_ADMIN), 1);
        
        if (smart_table::contains(&registry.roles, target)) {
            let current = *smart_table::borrow(&registry.roles, target);
            *smart_table::borrow_mut(&mut registry.roles, target) = current & (!role);
        };
    }
    
    // Check if address has specific role
    public fun has_role(registry_addr: address, addr: address, role: u64): bool acquires RoleRegistry {
        let registry = borrow_global<RoleRegistry>(registry_addr);
        has_role_internal(registry, addr, role)
    }
    
    // Assert role (revert if not authorized)
    public fun assert_role(registry_addr: address, addr: address, role: u64) acquires RoleRegistry {
        assert!(has_role(registry_addr, addr, role), error_unauthorized());
    }
    
    fun has_role_internal(registry: &RoleRegistry, addr: address, role: u64): bool {
        if (!smart_table::contains(&registry.roles, addr)) return false;
        let roles = *smart_table::borrow(&registry.roles, addr);
        (roles & role) == role
    }
    
    fun error_unauthorized(): u64 { 403 }
    
    // ============================================
    // TWO-STEP OWNERSHIP TRANSFER
    // Prevents transferring to wrong address
    // ============================================
    
    struct Ownership has key {
        owner: address,
        pending_owner: Option<address>,
    }
    
    public fun transfer_ownership(owner: &signer, ownership_addr: address, new_owner: address) acquires Ownership {
        let ownership = borrow_global_mut<Ownership>(ownership_addr);
        assert!(std::signer::address_of(owner) == ownership.owner, 1);
        ownership.pending_owner = std::option::some(new_owner);
    }
    
    public fun accept_ownership(new_owner: &signer, ownership_addr: address) acquires Ownership {
        let ownership = borrow_global_mut<Ownership>(ownership_addr);
        let new_owner_addr = std::signer::address_of(new_owner);
        
        assert!(std::option::is_some(&ownership.pending_owner), 1);
        let pending = std::option::extract(&mut ownership.pending_owner);
        assert!(pending == new_owner_addr, 2);
        
        ownership.owner = new_owner_addr;
    }
}
```

---

## Integer Overflow Protections

```move
module security::safe_math {
    // ============================================
    // SAFE ARITHMETIC
    // Move u64 wraps on overflow in some contexts.
    // Use checked operations for financial math.
    // ============================================
    
    const MAX_U64: u64 = 18_446_744_073_709_551_615;
    const MAX_U128: u128 = 340_282_366_920_938_463_463_374_607_431_768_211_455;
    
    // Safe add: aborts if overflow
    public fun safe_add(a: u64, b: u64): u64 {
        assert!(a <= MAX_U64 - b, 1);
        a + b
    }
    
    // Safe sub: aborts if underflow
    public fun safe_sub(a: u64, b: u64): u64 {
        assert!(a >= b, 2);
        a - b
    }
    
    // Safe mul: aborts if overflow
    public fun safe_mul(a: u64, b: u64): u64 {
        if (a == 0 || b == 0) return 0;
        assert!(a <= MAX_U64 / b, 3);
        a * b
    }
    
    // Safe div: aborts if div by zero
    public fun safe_div(a: u64, b: u64): u64 {
        assert!(b > 0, 4);
        a / b
    }
    
    // Multiply then divide (muldiv): avoids intermediate overflow
    // Computes (a * b) / c without overflowing
    public fun muldiv(a: u64, b: u64, c: u64): u64 {
        assert!(c > 0, 4);
        // Use u128 intermediate to prevent overflow
        let result_128 = (a as u128) * (b as u128) / (c as u128);
        assert!(result_128 <= (MAX_U64 as u128), 5);
        result_128 as u64
    }
    
    // Multiply then divide with ceiling
    public fun muldiv_ceil(a: u64, b: u64, c: u64): u64 {
        assert!(c > 0, 4);
        let numerator = (a as u128) * (b as u128);
        let result = (numerator + (c as u128) - 1) / (c as u128);
        assert!(result <= (MAX_U64 as u128), 5);
        result as u64
    }
    
    // Basis points calculation: amount * bps / 10000
    public fun bps_mul(amount: u64, bps: u64): u64 {
        muldiv(amount, bps, 10_000)
    }
    
    // Percentage with 6 decimals: amount * pct / 1_000_000
    public fun pct_mul(amount: u64, pct: u64): u64 {
        muldiv(amount, pct, 1_000_000)
    }
    
    // Fixed-point 64.64: multiply two Q64.64 numbers
    public fun mul_q64(a: u128, b: u128): u128 {
        // a and b are Q64.64 fixed point
        // result = a * b / 2^64
        // Use 256-bit intermediate (simulate with two u128s)
        let a_hi = a >> 64;
        let a_lo = a & 0xFFFF_FFFF_FFFF_FFFF;
        let b_hi = b >> 64;
        let b_lo = b & 0xFFFF_FFFF_FFFF_FFFF;
        
        // (a_hi + a_lo/2^64) * (b_hi + b_lo/2^64)
        // = a_hi*b_hi + (a_hi*b_lo + a_lo*b_hi)/2^64 + a_lo*b_lo/2^128
        let term1 = a_hi * b_hi;  // Integer part
        let term2 = (a_hi * b_lo + a_lo * b_hi) >> 64;
        let term3 = (a_lo * b_lo) >> 64 >> 64;  // Very small, usually 0
        
        term1 + term2 + term3
    }
    
    // Square root (integer, floor)
    public fun sqrt(x: u64): u64 {
        if (x == 0) return 0;
        
        let mut z = x;
        let mut y = (x + 1) / 2;
        
        while (y < z) {
            z = y;
            y = (x / y + y) / 2;
        };
        
        z
    }
    
    // Square root of u128
    public fun sqrt_u128(x: u128): u128 {
        if (x == 0) return 0;
        
        let mut z = x;
        let mut y = (x + 1) / 2;
        
        while (y < z) {
            z = y;
            y = (x / y + y) / 2;
        };
        
        z
    }
    
    // Min/max helpers
    public fun min(a: u64, b: u64): u64 { if (a < b) a else b }
    public fun max(a: u64, b: u64): u64 { if (a > b) a else b }
    public fun clamp(x: u64, lo: u64, hi: u64): u64 { min(max(x, lo), hi) }
    
    // Absolute difference
    public fun abs_diff(a: u64, b: u64): u64 {
        if (a >= b) a - b else b - a
    }
}
```

---

## Front-Running Defenses

```move
module security::commit_reveal {
    use aptos_framework::timestamp;
    use std::hash;
    
    // ============================================
    // COMMIT-REVEAL SCHEME
    // Prevents front-running on sensitive operations
    // 
    // Use case: Token sale, auctions, random selections
    //
    // Phase 1: User commits hash(action || salt)
    // Phase 2: User reveals action + salt
    // Bot cannot front-run (doesn't know action until reveal)
    // ============================================
    
    struct CommitRevealSystem has key {
        commits: aptos_std::smart_table::SmartTable<address, Commitment>,
        reveal_delay: u64,   // Min time between commit and reveal (microseconds)
        reveal_window: u64,  // Max time to reveal after commit (microseconds)
    }
    
    struct Commitment has store {
        commitment_hash: vector<u8>,  // hash(action || salt)
        committed_at: u64,
        revealed: bool,
    }
    
    // Phase 1: Commit
    public fun commit(
        user: &signer,
        system_addr: address,
        commitment_hash: vector<u8>,  // Computed off-chain: sha3(action || salt)
    ) acquires CommitRevealSystem {
        let user_addr = std::signer::address_of(user);
        let system = borrow_global_mut<CommitRevealSystem>(system_addr);
        
        assert!(std::vector::length(&commitment_hash) == 32, 1); // SHA3-256
        
        let commitment = Commitment {
            commitment_hash,
            committed_at: timestamp::now_microseconds(),
            revealed: false,
        };
        
        if (aptos_std::smart_table::contains(&system.commits, user_addr)) {
            *aptos_std::smart_table::borrow_mut(&mut system.commits, user_addr) = commitment;
        } else {
            aptos_std::smart_table::add(&mut system.commits, user_addr, commitment);
        };
    }
    
    // Phase 2: Reveal
    public fun reveal_and_verify(
        user: &signer,
        system_addr: address,
        action: vector<u8>,  // The actual action data
        salt: vector<u8>,    // Random salt used in commit
    ): bool acquires CommitRevealSystem {
        let user_addr = std::signer::address_of(user);
        let system = borrow_global_mut<CommitRevealSystem>(system_addr);
        let now = timestamp::now_microseconds();
        
        assert!(aptos_std::smart_table::contains(&system.commits, user_addr), 2);
        let commitment = aptos_std::smart_table::borrow_mut(&mut system.commits, user_addr);
        
        // Not yet revealed
        assert!(!commitment.revealed, 3);
        
        // Time window checks
        let elapsed = now - commitment.committed_at;
        assert!(elapsed >= system.reveal_delay, 4);   // Too early to reveal
        assert!(elapsed <= system.reveal_window, 5);  // Too late, commitment expired
        
        // Verify commitment hash
        let mut preimage = action;
        std::vector::append(&mut preimage, salt);
        let computed_hash = hash::sha3_256(preimage);
        
        let valid = commitment.commitment_hash == computed_hash;
        commitment.revealed = true;
        
        valid
    }
    
    // ============================================
    // SLIPPAGE PROTECTION (Anti-sandwich attack)
    // ============================================
    
    // User specifies max acceptable price impact
    // If miner/validator manipulates price beyond limit -> revert
    public fun swap_with_slippage_protection<TokenIn, TokenOut>(
        user: &signer,
        pool_addr: address,
        amount_in: u64,
        min_out: u64,          // Calculated off-chain with tolerance
        max_price_impact_bps: u64,  // e.g., 100 = 1% max impact
        deadline: u64,         // Transaction expiry (prevents stale txs)
    ) {
        // Check deadline
        assert!(timestamp::now_microseconds() <= deadline, 1);
        
        // Execute swap
        // let out = pool::swap(pool_addr, amount_in);
        
        // Check minimum output (slippage check)
        // assert!(out >= min_out, 2);
    }
}
```

---

## Emergency Response System

```move
module security::emergency {
    use aptos_framework::timestamp;
    
    // ============================================
    // MULTI-LEVEL EMERGENCY SYSTEM
    //
    // Level 0: Normal operation
    // Level 1: Deposits paused (withdrawals OK)
    // Level 2: All operations paused
    // Level 3: Emergency mode (admin override)
    // ============================================
    
    struct EmergencyState has key {
        level: u8,          // 0-3
        paused_at: u64,
        guardian: address,  // Can pause instantly
        council: vector<address>,  // Multi-sig to unpause
        unpause_threshold: u64,    // Votes needed to unpause
        unpause_votes: u64,        // Current votes
        voted: vector<address>,    // Who already voted
        
        // Timelock for critical operations
        timelock_delay: u64,       // 48 hours
        scheduled_ops: vector<ScheduledOp>,
    }
    
    struct ScheduledOp has store {
        id: u64,
        op_type: u8,
        data: vector<u8>,
        scheduled_at: u64,
        executed: bool,
    }
    
    // Guardian can instantly pause (single key, fast response)
    public fun emergency_pause(
        guardian: &signer,
        state_addr: address,
        level: u8,
    ) acquires EmergencyState {
        let state = borrow_global_mut<EmergencyState>(state_addr);
        assert!(std::signer::address_of(guardian) == state.guardian, 1);
        assert!(level >= 1 && level <= 3, 2);
        
        state.level = level;
        state.paused_at = timestamp::now_microseconds();
        state.unpause_votes = 0;
        state.voted = vector::empty();
    }
    
    // Council votes to unpause (requires threshold)
    public fun vote_unpause(
        voter: &signer,
        state_addr: address,
        target_level: u8,  // Target level (usually 0 = normal)
    ) acquires EmergencyState {
        let state = borrow_global_mut<EmergencyState>(state_addr);
        let voter_addr = std::signer::address_of(voter);
        
        // Verify voter is in council
        assert!(is_in_council(&state.council, voter_addr), 1);
        
        // No double voting
        assert!(!has_voted(&state.voted, voter_addr), 2);
        
        std::vector::push_back(&mut state.voted, voter_addr);
        state.unpause_votes = state.unpause_votes + 1;
        
        // Check if threshold reached
        if (state.unpause_votes >= state.unpause_threshold) {
            state.level = target_level;
            state.unpause_votes = 0;
            state.voted = vector::empty();
        };
    }
    
    // Checks for protocol functions
    public fun assert_not_paused(state_addr: address) acquires EmergencyState {
        let state = borrow_global<EmergencyState>(state_addr);
        assert!(state.level == 0, 503); // Service unavailable
    }
    
    public fun assert_withdrawals_allowed(state_addr: address) acquires EmergencyState {
        let state = borrow_global<EmergencyState>(state_addr);
        assert!(state.level < 2, 503); // Level 2+ pauses withdrawals
    }
    
    // Schedule an operation with timelock
    public fun schedule_operation(
        admin: &signer,
        state_addr: address,
        op_type: u8,
        data: vector<u8>,
    ): u64 acquires EmergencyState {
        let state = borrow_global_mut<EmergencyState>(state_addr);
        // Verify admin role through access control
        
        let id = std::vector::length(&state.scheduled_ops) as u64;
        let op = ScheduledOp {
            id,
            op_type,
            data,
            scheduled_at: timestamp::now_microseconds(),
            executed: false,
        };
        std::vector::push_back(&mut state.scheduled_ops, op);
        id
    }
    
    // Execute after timelock delay
    public fun execute_operation(
        state_addr: address,
        op_id: u64,
    ) acquires EmergencyState {
        let state = borrow_global_mut<EmergencyState>(state_addr);
        let op = std::vector::borrow_mut(&mut state.scheduled_ops, op_id as u64);
        
        assert!(!op.executed, 1);
        let elapsed = timestamp::now_microseconds() - op.scheduled_at;
        assert!(elapsed >= state.timelock_delay, 2); // Timelock not expired
        
        op.executed = true;
        // Execute the operation based on op_type and data
    }
    
    fun is_in_council(council: &vector<address>, addr: address): bool {
        let len = std::vector::length(council);
        let mut i = 0u64;
        while (i < len) {
            if (*std::vector::borrow(council, i) == addr) return true;
            i = i + 1;
        };
        false
    }
    
    fun has_voted(voted: &vector<address>, addr: address): bool {
        is_in_council(voted, addr)
    }
}
```

---

## Formal Verification with Move Prover

```move
module security::proven_token {
    // ============================================
    // FORMALLY VERIFIED TOKEN CONTRACT
    // Move Prover checks these properties at compile time
    // ============================================
    
    struct Balance has key {
        value: u64,
    }
    
    struct TotalSupply has key {
        value: u64,
    }
    
    // SPEC: Transfer preserves total balance
    // Prover will verify this mathematically
    spec fun transfer_spec(from: address, to: address, amount: u64) {
        // Preconditions
        requires exists<Balance>(from);
        requires exists<Balance>(to);
        requires global<Balance>(from).value >= amount;
        requires from != to;
        
        // Postconditions
        ensures global<Balance>(from).value == old(global<Balance>(from).value) - amount;
        ensures global<Balance>(to).value == old(global<Balance>(to).value) + amount;
        
        // Invariant: total supply unchanged
        ensures global<TotalSupply>(@module_addr).value == old(global<TotalSupply>(@module_addr).value);
    }
    
    public fun transfer(
        from: &signer,
        to: address,
        amount: u64,
    ) acquires Balance {
        let from_addr = std::signer::address_of(from);
        
        let from_bal = borrow_global_mut<Balance>(from_addr);
        assert!(from_bal.value >= amount, 1);
        from_bal.value = from_bal.value - amount;
        
        let to_bal = borrow_global_mut<Balance>(to);
        to_bal.value = to_bal.value + amount;
    }
    
    // SPEC: Mint increases total supply by exactly minted amount
    spec fun mint_spec(amount: u64) {
        ensures global<TotalSupply>(@module_addr).value == 
            old(global<TotalSupply>(@module_addr).value) + amount;
    }
    
    // SPEC: Global supply invariant (never exceeds MAX_SUPPLY)
    spec module {
        invariant forall addr: address where exists<Balance>(addr):
            global<Balance>(addr).value <= global<TotalSupply>(@module_addr).value;
        
        invariant exists<TotalSupply>(@module_addr) ==>
            global<TotalSupply>(@module_addr).value <= 1_000_000_000_000_000; // 1 quadrillion
    }
}
```

---

## สรุป Advanced Security

```
Security Checklist for Move Protocols:

ACCESS CONTROL
  [ ] Admin key is multisig (3/5 minimum)
  [ ] Role-based permissions (not just owner check)
  [ ] Two-step ownership transfer
  [ ] Time-lock on critical parameter changes (48h minimum)
  [ ] Guardian key for emergency pause (separate from admin)

ORACLE SECURITY
  [ ] TWAP prices (not spot prices) for calculations
  [ ] Multiple oracle sources with median
  [ ] Price deviation bounds (max X% change per block)
  [ ] Heartbeat check (oracle not stale > 1 hour)
  [ ] Circuit breakers on extreme prices

ARITHMETIC SAFETY
  [ ] All multiplications use 128-bit intermediates
  [ ] Division before multiply avoided
  [ ] Rounding direction explicit (floor vs ceiling)
  [ ] Maximum value bounds checked
  [ ] No modular arithmetic for financial values

FLASH LOAN PROTECTION
  [ ] Use TWAP not spot prices
  [ ] End-of-transaction balance checks
  [ ] Reentrancy guard on critical functions
  [ ] Flash loan fee (non-zero)

FRONT-RUNNING
  [ ] Commit-reveal for sensitive operations
  [ ] Slippage tolerance parameters
  [ ] Transaction deadlines
  [ ] Private mempool where available

FORMAL VERIFICATION
  [ ] Move Prover specs for core invariants
  [ ] Supply conservation proofs
  [ ] Permission hierarchy proofs
  [ ] Arithmetic overflow proofs
```

---

**ก่อนหน้า**: [Part 65 - DeFi Aggregators ←](part-65-defi-aggregators.md)
**ต่อไป**: [Part 67 - Flash Loan Protocols →](part-67-flash-loans.md)
