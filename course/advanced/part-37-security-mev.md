# Part 37: Security Patterns & MEV Protection

## สารบัญ
- [Common Vulnerabilities in Move](#common-vulnerabilities)
- [Reentrancy in Move](#reentrancy-in-move)
- [Access Control Patterns](#access-control-patterns)
- [MEV Protection](#mev-protection)
- [Flash Loan Attacks](#flash-loan-attacks)
- [ตัวอย่าง: Secure DeFi Protocol](#ตัวอย่าง-secure-defi-protocol)

---

## Common Vulnerabilities in Move

```
Move ป้องกันหลาย vulnerabilities โดย design:
  ✓ Integer overflow → aborts (not wraps)
  ✓ Reentrancy → lineartype system prevents double-use
  ✓ Use-after-free → borrow checker
  ✓ Double-spend → Resource = can't copy

แต่ยังมีช่องโหว่ที่ต้องระวัง:
  ✗ Oracle manipulation → TWAP, circuit breaker
  ✗ Logic errors → formal verification
  ✗ Admin key compromise → multisig, timelocks
  ✗ Sandwich attacks → commit-reveal, min output
  ✗ Flash loan abuse → TWAP, per-block limits
  ✗ Precision loss → careful math order
  ✗ Signature replay → nonces, chain-id
```

---

## Reentrancy in Move

```move
module security::reentrancy {
    // ============================================
    // Move doesn't have traditional reentrancy
    // BUT: callbacks and entry functions can simulate it
    // ============================================
    
    // SAFE pattern: update state BEFORE external call
    
    struct Pool has key {
        balance: u64,
        is_locked: bool,  // reentrancy guard
    }
    
    // ============================================
    // Reentrancy Guard
    // ============================================
    
    struct ReentrancyGuard has key {
        locked: bool,
    }
    
    const E_REENTRANT: u64 = 1;
    
    // Acquire lock - panics if already locked
    fun lock(guard_addr: address) acquires ReentrancyGuard {
        let guard = borrow_global_mut<ReentrancyGuard>(guard_addr);
        assert!(!guard.locked, E_REENTRANT);
        guard.locked = true;
    }
    
    fun unlock(guard_addr: address) acquires ReentrancyGuard {
        let guard = borrow_global_mut<ReentrancyGuard>(guard_addr);
        guard.locked = false;
    }
    
    // ============================================
    // Check-Effects-Interactions Pattern
    // ============================================
    
    public entry fun withdraw(
        user: &signer,
        pool_addr: address,
        amount: u64,
    ) acquires Pool, ReentrancyGuard {
        // LOCK first
        lock(pool_addr);
        
        let pool = borrow_global_mut<Pool>(pool_addr);
        
        // CHECK
        assert!(pool.balance >= amount, 2);
        
        // EFFECTS (update state)
        pool.balance = pool.balance - amount;
        
        // INTERACTIONS (external call last)
        // transfer(user, amount)  <-- any callback here is safe
        
        // UNLOCK last
        unlock(pool_addr);
    }
    
    // ============================================
    // Checks-Effects-Interactions without guard
    // (Safe because state updated before any transfer)
    // ============================================
    
    public entry fun safe_withdraw(
        user: &signer,
        pool_addr: address,
        amount: u64,
    ) acquires Pool {
        let user_addr = std::signer::address_of(user);
        let pool = borrow_global_mut<Pool>(pool_addr);
        
        // CHECK
        assert!(pool.balance >= amount, 2);
        
        // EFFECT (deduct FIRST)
        pool.balance = pool.balance - amount;
        
        // INTERACTION (send after state updated)
        // Even if recipient's receive() calls back into this contract,
        // their balance is already decremented
        aptos_framework::coin::transfer<aptos_framework::aptos_coin::AptosCoin>(
            user, pool_addr, amount
        );
        // In Move: no callbacks on coin transfer, so extra safe
    }
}
```

---

## Access Control Patterns

```move
module security::access_control {
    use std::signer;
    
    // ============================================
    // Role-Based Access Control (RBAC)
    // ============================================
    
    // Role definitions (use constants)
    const ROLE_ADMIN: u8 = 0;
    const ROLE_OPERATOR: u8 = 1;
    const ROLE_MINTER: u8 = 2;
    const ROLE_PAUSER: u8 = 3;
    const ROLE_UPGRADER: u8 = 4;
    
    struct RoleRegistry has key {
        roles: aptos_std::smart_table::SmartTable<RoleKey, bool>,
    }
    
    struct RoleKey has copy, drop, store {
        account: address,
        role: u8,
    }
    
    // ============================================
    // Role management
    // ============================================
    
    public entry fun grant_role(
        admin: &signer,
        registry_addr: address,
        account: address,
        role: u8,
    ) acquires RoleRegistry {
        use aptos_std::smart_table;
        // Only admin can grant roles (except ADMIN itself needs special care)
        assert!(has_role(registry_addr, signer::address_of(admin), ROLE_ADMIN), 1);
        
        let registry = borrow_global_mut<RoleRegistry>(registry_addr);
        let key = RoleKey { account, role };
        if (!smart_table::contains(&registry.roles, key)) {
            smart_table::add(&mut registry.roles, key, true);
        };
    }
    
    public entry fun revoke_role(
        admin: &signer,
        registry_addr: address,
        account: address,
        role: u8,
    ) acquires RoleRegistry {
        use aptos_std::smart_table;
        assert!(has_role(registry_addr, signer::address_of(admin), ROLE_ADMIN), 1);
        
        let registry = borrow_global_mut<RoleRegistry>(registry_addr);
        let key = RoleKey { account, role };
        if (smart_table::contains(&registry.roles, key)) {
            smart_table::remove(&mut registry.roles, key);
        };
    }
    
    public fun has_role(registry_addr: address, account: address, role: u8): bool 
    acquires RoleRegistry {
        use aptos_std::smart_table;
        let registry = borrow_global<RoleRegistry>(registry_addr);
        let key = RoleKey { account, role };
        smart_table::contains(&registry.roles, key) &&
            *smart_table::borrow(&registry.roles, key)
    }
    
    public fun require_role(registry_addr: address, account: address, role: u8) 
    acquires RoleRegistry {
        assert!(has_role(registry_addr, account, role), 2);
    }
    
    // ============================================
    // Two-step ownership transfer
    // Prevents transfer to wrong address
    // ============================================
    
    struct OwnershipConfig has key {
        owner: address,
        pending_owner: address,
    }
    
    const ZERO_ADDR: address = @0x0;
    
    public entry fun propose_ownership(
        owner: &signer,
        config_addr: address,
        new_owner: address,
    ) acquires OwnershipConfig {
        let config = borrow_global_mut<OwnershipConfig>(config_addr);
        assert!(signer::address_of(owner) == config.owner, 1);
        config.pending_owner = new_owner;
    }
    
    public entry fun accept_ownership(
        new_owner: &signer,
        config_addr: address,
    ) acquires OwnershipConfig {
        let config = borrow_global_mut<OwnershipConfig>(config_addr);
        assert!(signer::address_of(new_owner) == config.pending_owner, 2);
        config.owner = config.pending_owner;
        config.pending_owner = ZERO_ADDR;
    }
}
```

---

## MEV Protection

```move
module security::mev_protection {
    use std::signer;
    use aptos_framework::timestamp;
    use aptos_framework::event;
    
    // ============================================
    // MEV = Miner/Validator Extractable Value
    // Sandwich attacks, front-running, back-running
    // ============================================
    
    // ============================================
    // Commit-Reveal for sensitive operations
    // ============================================
    
    struct CommitReveal has key {
        commits: aptos_std::smart_table::SmartTable<address, Commit>,
        commit_timeout: u64,
        reveal_window: u64,
    }
    
    struct Commit has copy, drop, store {
        hash: vector<u8>,      // hash(secret + params)
        committed_at: u64,
        reveal_deadline: u64,
    }
    
    const COMMIT_PERIOD: u64 = 60;   // must commit 60s before reveal
    const REVEAL_WINDOW: u64 = 300;  // 5 minutes to reveal
    
    const E_ALREADY_COMMITTED: u64 = 1;
    const E_COMMIT_NOT_FOUND: u64 = 2;
    const E_REVEAL_PERIOD_NOT_STARTED: u64 = 3;
    const E_REVEAL_EXPIRED: u64 = 4;
    const E_HASH_MISMATCH: u64 = 5;
    
    // Phase 1: Commit (broadcast intention without revealing params)
    public entry fun commit(
        user: &signer,
        cr_addr: address,
        hash: vector<u8>,
    ) acquires CommitReveal {
        use aptos_std::smart_table;
        let user_addr = signer::address_of(user);
        let cr = borrow_global_mut<CommitReveal>(cr_addr);
        
        assert!(!smart_table::contains(&cr.commits, user_addr), E_ALREADY_COMMITTED);
        
        let now = timestamp::now_seconds();
        smart_table::add(&mut cr.commits, user_addr, Commit {
            hash,
            committed_at: now,
            reveal_deadline: now + COMMIT_PERIOD + REVEAL_WINDOW,
        });
    }
    
    // Phase 2: Reveal actual parameters (after commit_period)
    public entry fun reveal_and_execute(
        user: &signer,
        cr_addr: address,
        secret: vector<u8>,
        params: vector<u8>,
    ) acquires CommitReveal {
        use aptos_std::smart_table;
        let user_addr = signer::address_of(user);
        let cr = borrow_global_mut<CommitReveal>(cr_addr);
        
        assert!(smart_table::contains(&cr.commits, user_addr), E_COMMIT_NOT_FOUND);
        let commit = *smart_table::borrow(&cr.commits, user_addr);
        
        let now = timestamp::now_seconds();
        assert!(now >= commit.committed_at + COMMIT_PERIOD, E_REVEAL_PERIOD_NOT_STARTED);
        assert!(now <= commit.reveal_deadline, E_REVEAL_EXPIRED);
        
        // Verify hash(secret || params) == committed hash
        let mut preimage = secret;
        std::vector::append(&mut preimage, params);
        let computed_hash = aptos_std::aptos_hash::keccak256(preimage);
        assert!(computed_hash == commit.hash, E_HASH_MISMATCH);
        
        // Remove commit
        smart_table::remove(&mut cr.commits, user_addr);
        
        // Execute with revealed params (MEV-resistant: params were hidden until now)
    }
    
    // ============================================
    // Slippage Protection
    // ============================================
    
    public fun check_slippage(
        expected: u64,
        actual: u64,
        max_slippage_bps: u64,
    ) {
        // actual >= expected * (1 - slippage)
        let min_acceptable = expected * (10_000 - max_slippage_bps) / 10_000;
        assert!(actual >= min_acceptable, 100);
    }
    
    // ============================================
    // Deadline Protection
    // Prevents mempool lingering attacks
    // ============================================
    
    public fun check_deadline(deadline: u64) {
        assert!(timestamp::now_seconds() <= deadline, 101);
    }
    
    // ============================================
    // Private Mempool Pattern (Aptos doesn't have native mempool privacy)
    // But: use sequencing protection via timestamp ordering
    // ============================================
    
    struct OrderProtection has key {
        min_tx_gap: u64,          // min seconds between user txs
        last_tx_time: aptos_std::smart_table::SmartTable<address, u64>,
    }
    
    public fun check_tx_timing(
        op_addr: address,
        user: address,
    ) acquires OrderProtection {
        use aptos_std::smart_table;
        let op = borrow_global_mut<OrderProtection>(op_addr);
        let now = timestamp::now_seconds();
        
        if (smart_table::contains(&op.last_tx_time, user)) {
            let last = *smart_table::borrow(&op.last_tx_time, user);
            assert!(now >= last + op.min_tx_gap, 102);
            *smart_table::borrow_mut(&mut op.last_tx_time, user) = now;
        } else {
            smart_table::add(&mut op.last_tx_time, user, now);
        };
    }
    
    // ============================================
    // Max Price Impact (prevent large sandwich attacks)
    // ============================================
    
    public fun check_price_impact(
        pool_x: u64,
        pool_y: u64,
        dx: u64,
        max_impact_bps: u64,
    ) {
        // Price before: y/x
        // Price after: (y - dy) / (x + dx)
        // Impact = (price_after - price_before) / price_before
        
        // Simplified: use ratio of dx to pool_x
        let impact = dx * 10_000 / pool_x;
        assert!(impact <= max_impact_bps, 103);
    }
}
```

---

## Flash Loan Attacks

```move
module security::flash_loan_defense {
    use aptos_framework::timestamp;
    
    // ============================================
    // Defenses against flash loan manipulation
    // ============================================
    
    // ============================================
    // 1. Per-block price smoothing
    // ============================================
    
    struct PriceSmoothing has key {
        last_block_time: u64,
        last_block_price: u64,
        smoothed_price: u64,
        alpha_bps: u64,    // EMA factor (e.g., 2000 = 20% weight to new)
    }
    
    // Exponential Moving Average
    public fun update_ema_price(
        smoothing_addr: address,
        spot_price: u64,
    ) acquires PriceSmoothing {
        let s = borrow_global_mut<PriceSmoothing>(smoothing_addr);
        let now = timestamp::now_seconds();
        
        // Only update once per block (roughly)
        // Prevents same-block manipulation
        if (now <= s.last_block_time) return;
        
        // EMA = alpha * new + (1-alpha) * old
        let new_contribution = spot_price * s.alpha_bps / 10_000;
        let old_contribution = s.smoothed_price * (10_000 - s.alpha_bps) / 10_000;
        
        s.smoothed_price = new_contribution + old_contribution;
        s.last_block_time = now;
        s.last_block_price = spot_price;
    }
    
    // ============================================
    // 2. Reserve ratio checks
    // Ensure pool ratios aren't manipulated
    // ============================================
    
    struct ReserveSnapshot has key {
        reserve_x: u64,
        reserve_y: u64,
        timestamp: u64,
    }
    
    public fun check_reserve_manipulation(
        snapshot_addr: address,
        current_x: u64,
        current_y: u64,
        max_change_bps: u64,
    ) acquires ReserveSnapshot {
        let snap = borrow_global<ReserveSnapshot>(snapshot_addr);
        
        // Check x hasn't changed too much
        let x_change = if (current_x > snap.reserve_x) {
            (current_x - snap.reserve_x) * 10_000 / snap.reserve_x
        } else {
            (snap.reserve_x - current_x) * 10_000 / snap.reserve_x
        };
        assert!(x_change <= max_change_bps, 1);
        
        // Check y hasn't changed too much
        let y_change = if (current_y > snap.reserve_y) {
            (current_y - snap.reserve_y) * 10_000 / snap.reserve_y
        } else {
            (snap.reserve_y - current_y) * 10_000 / snap.reserve_y
        };
        assert!(y_change <= max_change_bps, 2);
    }
    
    // ============================================
    // 3. Same-transaction Oracle manipulation detection
    // ============================================
    
    struct OracleGuard has key {
        reads_this_tx: u64,
        max_reads_per_tx: u64,
        last_tx_seq: u64,
    }
    
    // Track oracle reads to detect manipulation
    public fun read_price_guarded(
        guard_addr: address,
        oracle_addr: address,
        symbol: vector<u8>,
    ): u64 acquires OracleGuard {
        let guard = borrow_global_mut<OracleGuard>(guard_addr);
        
        // Reset counter per "transaction" (using sequence-like mechanism)
        // In practice: use aptos_framework::transaction_context
        
        guard.reads_this_tx = guard.reads_this_tx + 1;
        assert!(guard.reads_this_tx <= guard.max_reads_per_tx, 3);
        
        // Get actual price
        0  // placeholder
    }
    
    // ============================================
    // 4. Timelocked operations for large amounts
    // ============================================
    
    struct LargeOpTimelock has key {
        threshold: u64,
        delay: u64,
        pending: aptos_std::smart_table::SmartTable<u64, PendingOp>,
        next_id: u64,
    }
    
    struct PendingOp has store {
        op_type: u8,
        params: vector<u8>,
        amount: u64,
        requested_at: u64,
        executable_at: u64,
    }
    
    const LARGE_OP_THRESHOLD: u64 = 100_000_000_000;  // 100K tokens
    const LARGE_OP_DELAY: u64 = 3600;  // 1 hour delay
    
    public entry fun request_large_op(
        op_type: u8,
        params: vector<u8>,
        amount: u64,
        timelock_addr: address,
    ) acquires LargeOpTimelock {
        use aptos_std::smart_table;
        let tl = borrow_global_mut<LargeOpTimelock>(timelock_addr);
        
        assert!(amount >= tl.threshold, 4);
        
        let now = timestamp::now_seconds();
        let id = tl.next_id;
        tl.next_id = id + 1;
        
        smart_table::add(&mut tl.pending, id, PendingOp {
            op_type,
            params,
            amount,
            requested_at: now,
            executable_at: now + tl.delay,
        });
    }
}
```

---

## ตัวอย่าง: Secure DeFi Protocol

```move
module security::secure_amm {
    use std::signer;
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::timestamp;
    use aptos_framework::event;
    
    // ============================================
    // Production-grade secure AMM
    // ============================================
    
    struct SecurePool<phantom X, phantom Y> has key {
        // Core
        reserve_x: Coin<X>,
        reserve_y: Coin<Y>,
        lp_supply: u64,
        
        // Security
        is_paused: bool,
        admin: address,
        last_k: u128,  // for invariant checks
        
        // Rate limiting
        max_swap_bps: u64,  // max swap as % of reserves
        
        // Oracle protection
        twap_x: u64,        // TWAP of x/y ratio
        twap_update_time: u64,
        
        // MEV protection
        min_block_gap: u64,
    }
    
    const E_PAUSED: u64 = 1;
    const E_SLIPPAGE: u64 = 2;
    const E_DEADLINE: u64 = 3;
    const E_EXCESSIVE_SWAP: u64 = 4;
    const E_K_INVARIANT: u64 = 5;
    const E_ZERO_AMOUNT: u64 = 6;
    
    public entry fun swap_x_to_y<X, Y>(
        user: &signer,
        pool_addr: address,
        dx: Coin<X>,
        min_dy: u64,
        deadline: u64,
    ) acquires SecurePool<X, Y> {
        let pool = borrow_global_mut<SecurePool<X, Y>>(pool_addr);
        
        // Security checks
        assert!(!pool.is_paused, E_PAUSED);
        assert!(timestamp::now_seconds() <= deadline, E_DEADLINE);
        assert!(coin::value(&dx) > 0, E_ZERO_AMOUNT);
        
        let dx_amount = coin::value(&dx);
        let rx = coin::value(&pool.reserve_x);
        let ry = coin::value(&pool.reserve_y);
        
        // Rate limit: max 5% of reserves per swap
        assert!(dx_amount * 100 / rx <= pool.max_swap_bps, E_EXCESSIVE_SWAP);
        
        // Constant product formula with 0.3% fee
        let fee_bps = 30u64;
        let dx_with_fee = dx_amount * (10_000 - fee_bps);
        let dy = ry * dx_with_fee / (rx * 10_000 + dx_with_fee);
        
        // Slippage check
        assert!(dy >= min_dy, E_SLIPPAGE);
        
        // Save k before
        let k_before = (rx as u128) * (ry as u128);
        
        // Apply swap
        coin::merge(&mut pool.reserve_x, dx);
        let dy_coin = coin::extract(&mut pool.reserve_y, dy);
        
        // Invariant check: k must not decrease
        let rx_new = coin::value(&pool.reserve_x);
        let ry_new = coin::value(&pool.reserve_y);
        let k_after = (rx_new as u128) * (ry_new as u128);
        assert!(k_after >= k_before, E_K_INVARIANT);
        
        // Update TWAP
        let new_ratio = ry_new * 1_000_000 / rx_new;  // y per x, 6 dec precision
        let now = timestamp::now_seconds();
        let elapsed = now - pool.twap_update_time;
        if (elapsed > 0) {
            let alpha = if (elapsed > 600) { 2000u64 } else { elapsed * 2000 / 600 };
            pool.twap_x = (new_ratio * alpha + pool.twap_x * (10_000 - alpha)) / 10_000;
            pool.twap_update_time = now;
        };
        
        // Deliver output
        let user_addr = signer::address_of(user);
        coin::deposit(user_addr, dy_coin);
        
        event::emit(SwapEvent {
            user: user_addr,
            dx: dx_amount,
            dy,
            reserve_x: rx_new,
            reserve_y: ry_new,
        });
    }
    
    #[event]
    struct SwapEvent has drop, store {
        user: address,
        dx: u64,
        dy: u64,
        reserve_x: u64,
        reserve_y: u64,
    }
    
    // ============================================
    // Emergency Pause
    // ============================================
    
    public entry fun emergency_pause<X, Y>(
        admin: &signer,
        pool_addr: address,
    ) acquires SecurePool<X, Y> {
        let pool = borrow_global_mut<SecurePool<X, Y>>(pool_addr);
        assert!(signer::address_of(admin) == pool.admin, 10);
        pool.is_paused = true;
    }
    
    public entry fun resume<X, Y>(
        admin: &signer,
        pool_addr: address,
    ) acquires SecurePool<X, Y> {
        let pool = borrow_global_mut<SecurePool<X, Y>>(pool_addr);
        assert!(signer::address_of(admin) == pool.admin, 10);
        pool.is_paused = false;
    }
}
```

---

## Security Checklist

```
Before Deploying:
  □ Integer math: check for overflow in critical calculations
  □ Access control: every admin function guarded
  □ Oracle: staleness checks, TWAP for large ops
  □ Slippage: min_out parameter on all swaps
  □ Deadline: expiry on all time-sensitive operations
  □ Rate limits: per-user and per-tx limits
  □ Reentrancy: state updated before external calls
  □ Events: all state changes emit events
  □ Pause: emergency stop mechanism
  □ Upgrade: timelock on critical changes
  □ Testing: fuzz tests on all math functions
  □ Audit: professional security review
  □ Bug bounty: post-launch continuous security

In Production:
  □ Monitor: anomaly detection on pool metrics
  □ Alerts: Discord/Telegram on large transactions
  □ Response: incident response playbook ready
  □ Insurance: cover protocol with DeFi insurance
```

---

**ก่อนหน้า**: [Part 36 - Cross-chain ←](part-36-cross-chain.md)
**ต่อไป**: [Part 38 - Advanced Sui Patterns →](part-38-sui-advanced.md)
