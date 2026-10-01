# Part 46: Protocol Security Auditing

## สารบัญ
- [Security Auditing Framework](#security-auditing-framework)
- [Common Move Vulnerabilities](#common-move-vulnerabilities)
- [Audit Checklist](#audit-checklist)
- [Formal Verification for Auditing](#formal-verification-for-auditing)
- [ตัวอย่าง: Vulnerable vs Secure Code](#ตัวอย่าง-vulnerable-vs-secure-code)

---

## Security Auditing Framework

```
Move Security Audit Process:

Phase 1: Reconnaissance
  - Understand protocol architecture
  - Map all modules and dependencies
  - Identify trusted/untrusted inputs
  - Document admin privileges

Phase 2: Static Analysis
  - Review all ability constraints
  - Check arithmetic operations
  - Verify access control
  - Analyze resource lifecycle

Phase 3: Logic Analysis
  - Financial invariants
  - State machine correctness
  - Cross-module interactions
  - Edge cases and boundary conditions

Phase 4: Dynamic Testing
  - Unit tests coverage
  - Integration tests
  - Fuzz testing (boundary values)
  - Scenario-based tests

Phase 5: Formal Verification
  - Move Prover specs
  - Invariant proofs
  - Pre/postcondition verification

Severity Levels:
  Critical: Funds at risk, immediate exploit
  High: Significant funds at risk, complex exploit
  Medium: Limited funds at risk, specific conditions
  Low: Best practice violations, edge cases
  Info: Documentation, code quality
```

---

## Common Move Vulnerabilities

```move
module audit::vulnerabilities {
    
    // ============================================
    // VULNERABILITY 1: Integer Overflow
    // ============================================
    
    // VULNERABLE: Multiplication before division can overflow
    public fun fee_bad(amount: u64, fee_bps: u64): u64 {
        // If amount = 1e18 and fee_bps = 100, this overflows u64
        amount * fee_bps / 10_000
    }
    
    // SECURE: Use u128 for intermediate calculation
    public fun fee_good(amount: u64, fee_bps: u64): u64 {
        ((amount as u128) * (fee_bps as u128) / 10_000u128) as u64
    }
    
    // ============================================
    // VULNERABILITY 2: Access Control Missing
    // ============================================
    
    // VULNERABLE: Anyone can call this
    public entry fun set_fee_bad(
        _caller: &signer,
        protocol_addr: address,
        new_fee: u64,
    ) acquires ProtocolConfig {
        let config = borrow_global_mut<ProtocolConfig>(protocol_addr);
        config.fee_bps = new_fee;  // No auth check!
    }
    
    // SECURE: Only admin can call
    public entry fun set_fee_good(
        admin: &signer,
        protocol_addr: address,
        new_fee: u64,
    ) acquires ProtocolConfig {
        let config = borrow_global_mut<ProtocolConfig>(protocol_addr);
        assert!(std::signer::address_of(admin) == config.admin, E_NOT_ADMIN);
        assert!(new_fee <= MAX_FEE_BPS, E_FEE_TOO_HIGH);
        config.fee_bps = new_fee;
    }
    
    // ============================================
    // VULNERABILITY 3: Price Manipulation (Spot Oracle)
    // ============================================
    
    // VULNERABLE: Uses spot price, manipulable in same tx
    public fun get_borrow_limit_bad(
        pool: &AmmPool,
        collateral_amount: u64,
    ): u64 {
        // Spot price: attacker can manipulate before calling
        let price = pool.reserve_y / pool.reserve_x;
        collateral_amount * price * LTV_BPS / 10_000
    }
    
    // SECURE: Use TWAP or external oracle
    public fun get_borrow_limit_good(
        twap: &TwapState,
        collateral_amount: u64,
    ): u64 acquires TwapState {
        // TWAP price: resistant to single-tx manipulation
        let price = twap.price_cumulative / twap.observations;
        collateral_amount * price * LTV_BPS / 10_000
    }
    
    // ============================================
    // VULNERABILITY 4: Reentrancy (Move-specific)
    // ============================================
    
    // Move doesn't have direct reentrancy like Solidity,
    // BUT cross-module calls can create logical reentrancy
    
    // VULNERABLE: State updated after external call
    struct Pool has key { balance: u64, total_shares: u64 }
    
    public fun withdraw_bad(
        user: &signer,
        pool_addr: address,
        shares: u64,
    ) acquires Pool {
        let pool = borrow_global_mut<Pool>(pool_addr);
        let amount = shares * pool.balance / pool.total_shares;
        
        // Send tokens first...
        // some_module::transfer(user, amount);  // External call!
        
        // Then update state - logical reentrancy if some_module
        // calls back into withdraw before this line
        pool.balance = pool.balance - amount;
        pool.total_shares = pool.total_shares - shares;
    }
    
    // SECURE: Update state before external calls
    public fun withdraw_good(
        user: &signer,
        pool_addr: address,
        shares: u64,
    ) acquires Pool {
        let pool = borrow_global_mut<Pool>(pool_addr);
        let amount = shares * pool.balance / pool.total_shares;
        
        // Update state FIRST
        pool.balance = pool.balance - amount;
        pool.total_shares = pool.total_shares - shares;
        
        // Then external call (safe: state already updated)
        // some_module::transfer(user, amount);
    }
    
    // ============================================
    // VULNERABILITY 5: Flash Loan Attack
    // ============================================
    
    // VULNERABLE: Snapshot check at wrong time
    public fun liquidate_bad(
        liquidator: &signer,
        borrower: address,
        pool_addr: address,
    ) acquires LendingPool {
        let pool = borrow_global<LendingPool>(pool_addr);
        
        // Price can be flash-loan manipulated before this call!
        let collateral_value = get_spot_collateral_value(borrower, pool);
        let debt = get_debt(borrower, pool);
        
        assert!(collateral_value < debt * LIQUIDATION_THRESHOLD / 10_000, 1);
        // Liquidate...
    }
    
    // SECURE: Use time-delayed oracle + circuit breaker
    public fun liquidate_good(
        liquidator: &signer,
        borrower: address,
        pool_addr: address,
    ) acquires LendingPool {
        let pool = borrow_global<LendingPool>(pool_addr);
        
        // TWAP price resistant to flash loans
        let collateral_value = get_twap_collateral_value(borrower, pool);
        let debt = get_debt(borrower, pool);
        
        // Also check price hasn't moved too fast (circuit breaker)
        let spot = get_spot_collateral_value(borrower, pool);
        let deviation = if (spot > collateral_value) {
            (spot - collateral_value) * 10_000 / collateral_value
        } else {
            (collateral_value - spot) * 10_000 / collateral_value
        };
        assert!(deviation < MAX_DEVIATION_BPS, E_PRICE_DEVIATION);
        
        assert!(collateral_value < debt * LIQUIDATION_THRESHOLD / 10_000, 1);
    }
    
    // ============================================
    // VULNERABILITY 6: Unchecked Arithmetic
    // ============================================
    
    // Move panics on overflow by default - but watch out for:
    
    // VULNERABLE: Subtraction can underflow and panic
    public fun calculate_reward_bad(
        earned: u64,
        claimed: u64,
    ): u64 {
        earned - claimed  // Panics if claimed > earned due to rounding
    }
    
    // SECURE: Check before subtract
    public fun calculate_reward_good(
        earned: u64,
        claimed: u64,
    ): u64 {
        if (claimed >= earned) 0
        else earned - claimed
    }
    
    // ============================================
    // VULNERABILITY 7: Improper Initialization
    // ============================================
    
    // VULNERABLE: No check if already initialized
    public entry fun initialize_bad(deployer: &signer) {
        let addr = std::signer::address_of(deployer);
        move_to(deployer, Config { admin: addr, paused: false });
        // If called twice: ABORTS (resource exists), but
        // could cause issues if init logic has side effects
    }
    
    // SECURE: Explicit check + meaningful error
    public entry fun initialize_good(deployer: &signer) {
        let addr = std::signer::address_of(deployer);
        assert!(!exists<Config>(addr), E_ALREADY_INITIALIZED);
        move_to(deployer, Config { admin: addr, paused: false });
    }
    
    // ============================================
    // VULNERABILITY 8: Missing Slippage Protection
    // ============================================
    
    // VULNERABLE: No slippage check on swap
    public entry fun swap_bad(
        user: &signer,
        pool_addr: address,
        amount_in: u64,
    ) acquires AmmPool {
        let out = calculate_out(amount_in, pool_addr);
        // Execute swap with whatever output amount
        do_swap(pool_addr, amount_in, out);
    }
    
    // SECURE: Minimum output protection
    public entry fun swap_good(
        user: &signer,
        pool_addr: address,
        amount_in: u64,
        min_out: u64,       // User-specified minimum
        deadline: u64,      // Transaction deadline
    ) acquires AmmPool {
        assert!(aptos_framework::timestamp::now_seconds() <= deadline, E_DEADLINE);
        let out = calculate_out(amount_in, pool_addr);
        assert!(out >= min_out, E_SLIPPAGE);
        do_swap(pool_addr, amount_in, out);
    }
    
    // Placeholder types
    struct ProtocolConfig has key { admin: address, fee_bps: u64 }
    struct AmmPool has key { reserve_x: u64, reserve_y: u64 }
    struct TwapState has key { price_cumulative: u64, observations: u64 }
    struct LendingPool has key { }
    struct Config has key { admin: address, paused: bool }
    
    const E_NOT_ADMIN: u64 = 1;
    const E_FEE_TOO_HIGH: u64 = 2;
    const E_PRICE_DEVIATION: u64 = 3;
    const E_ALREADY_INITIALIZED: u64 = 4;
    const E_DEADLINE: u64 = 5;
    const E_SLIPPAGE: u64 = 6;
    const MAX_FEE_BPS: u64 = 1000;
    const LTV_BPS: u64 = 7500;
    const LIQUIDATION_THRESHOLD: u64 = 8000;
    const MAX_DEVIATION_BPS: u64 = 500;
    
    fun get_spot_collateral_value(_borrower: address, _pool: &LendingPool): u64 { 0 }
    fun get_twap_collateral_value(_borrower: address, _pool: &LendingPool): u64 { 0 }
    fun get_debt(_borrower: address, _pool: &LendingPool): u64 { 0 }
    fun calculate_out(_amount_in: u64, _pool_addr: address): u64 { 0 }
    fun do_swap(_pool_addr: address, _amount_in: u64, _out: u64) {}
}
```

---

## Audit Checklist

```
MOVE SECURITY AUDIT CHECKLIST

=== RESOURCE MANAGEMENT ===
□ All resources properly initialized before use
□ Resources destroyed when no longer needed
□ No resource leaks (lost resources without drop ability)
□ Correct ability constraints (store/key/copy/drop)
□ Resource ownership verified before operations

=== ACCESS CONTROL ===
□ Admin/owner check on all privileged operations  
□ Role-based access properly implemented
□ Capability pattern used for fine-grained permissions
□ No privilege escalation paths
□ Module upgrade authorization secured

=== ARITHMETIC ===
□ All multiplications use u128 intermediates when overflow risk
□ Division-before-multiplication avoided
□ Subtraction underflow checked
□ Fixed-point arithmetic uses sufficient precision
□ Rounding direction intentional (floor/ceil)

=== ORACLES & PRICES ===
□ No spot price usage for critical financial ops
□ TWAP or external oracle for liquidations
□ Staleness checks on oracle updates
□ Circuit breakers for extreme price moves
□ Multiple oracle sources for critical prices

=== AMM INVARIANTS ===  
□ K invariant maintained after swaps
□ Minimum liquidity locked (prevent price manipulation)
□ LP token minting/burning at correct rates
□ Price impact bounded

=== LENDING PROTOCOL ===
□ Health factor calculated correctly
□ Liquidation threshold enforced
□ Interest accrual without overflow
□ Bad debt handling
□ Flash loan prevention

=== EXTERNAL CALLS ===
□ State updated before external calls
□ External module return values checked
□ Hot potato pattern used where appropriate
□ No circular dependencies

=== TIME & SEQUENCE ===
□ Timestamps: seconds vs microseconds correct
□ Epoch boundaries handled
□ Vesting/cliff calculations correct
□ Timelock enforced properly
□ Deadline checks on all user ops

=== UPGRADES ===
□ Upgrade policy appropriate (immutable > compatible > arbitrary)
□ Data migration preserves invariants
□ Emergency upgrade path secured
□ Feature flag system if needed

=== TESTING ===
□ >90% line coverage
□ All error codes tested
□ Edge cases: 0, MAX_U64, empty collections
□ Invariant tests (property-based)
□ Formal verification specs
```

---

## Formal Verification for Auditing

```move
module audit::verified_pool {
    
    // ============================================
    // Example: Fully verified AMM pool
    // ============================================
    
    struct Pool<phantom X, phantom Y> has key {
        reserve_x: u64,
        reserve_y: u64,
        lp_supply: u64,
        fee_bps: u64,
    }
    
    // Invariant: k = reserve_x * reserve_y (after fees)
    spec Pool<X, Y> {
        invariant reserve_x > 0 && reserve_y > 0;
        invariant lp_supply > 0 ==> reserve_x > 0 && reserve_y > 0;
        invariant fee_bps <= 100;  // Max 1% fee
    }
    
    // ============================================
    // Add liquidity
    // ============================================
    
    public fun add_liquidity<X, Y>(
        pool: &mut Pool<X, Y>,
        amount_x: u64,
        amount_y: u64,
    ): u64 {
        let lp_minted = if (pool.lp_supply == 0) {
            // Initial liquidity: geometric mean
            sqrt(amount_x, amount_y)
        } else {
            // Proportional to existing
            std::math64::min(
                amount_x * pool.lp_supply / pool.reserve_x,
                amount_y * pool.lp_supply / pool.reserve_y,
            )
        };
        
        pool.reserve_x = pool.reserve_x + amount_x;
        pool.reserve_y = pool.reserve_y + amount_y;
        pool.lp_supply = pool.lp_supply + lp_minted;
        
        lp_minted
    }
    
    spec add_liquidity<X, Y> {
        requires amount_x > 0 && amount_y > 0;
        ensures result > 0;
        // Pool reserves increase
        ensures pool.reserve_x == old(pool.reserve_x) + amount_x;
        ensures pool.reserve_y == old(pool.reserve_y) + amount_y;
        // LP supply increases
        ensures pool.lp_supply > old(pool.lp_supply);
        // K invariant: new_k >= old_k (liquidity addition only improves)
        ensures pool.reserve_x * pool.reserve_y 
                >= old(pool.reserve_x) * old(pool.reserve_y);
    }
    
    // ============================================
    // Swap with fee
    // ============================================
    
    public fun swap_x_for_y<X, Y>(
        pool: &mut Pool<X, Y>,
        amount_in: u64,
    ): u64 {
        // Amount out using constant product formula with fee
        // out = reserve_y * amount_in * (10000 - fee_bps) / 
        //       (reserve_x * 10000 + amount_in * (10000 - fee_bps))
        
        let fee_factor = 10_000 - pool.fee_bps;
        let amount_in_with_fee = (amount_in as u128) * (fee_factor as u128);
        let numerator = (pool.reserve_y as u128) * amount_in_with_fee;
        let denominator = (pool.reserve_x as u128) * 10_000u128 + amount_in_with_fee;
        
        let amount_out = (numerator / denominator) as u64;
        
        pool.reserve_x = pool.reserve_x + amount_in;
        pool.reserve_y = pool.reserve_y - amount_out;
        
        amount_out
    }
    
    spec swap_x_for_y<X, Y> {
        requires amount_in > 0;
        requires pool.reserve_x > 0 && pool.reserve_y > 0;
        
        // Output is positive
        ensures result > 0;
        
        // Don't take more than available
        ensures result < pool.reserve_y;
        
        // Reserve X increases
        ensures pool.reserve_x == old(pool.reserve_x) + amount_in;
        
        // Reserve Y decreases by output
        ensures pool.reserve_y == old(pool.reserve_y) - result;
        
        // K invariant: k after >= k before (fee creates surplus)
        // new_x * new_y >= old_x * old_y
        ensures pool.reserve_x * pool.reserve_y 
                >= old(pool.reserve_x) * old(pool.reserve_y);
        
        // No free money: output value <= input value (at current rate)
        // result * pool.reserve_x <= amount_in * pool.reserve_y  (approx)
        
        aborts_if pool.reserve_y <= result;
        aborts_if amount_in == 0;
    }
    
    // ============================================
    // Remove liquidity
    // ============================================
    
    public fun remove_liquidity<X, Y>(
        pool: &mut Pool<X, Y>,
        lp_amount: u64,
    ): (u64, u64) {
        assert!(lp_amount <= pool.lp_supply, 1);
        
        let x_out = pool.reserve_x * lp_amount / pool.lp_supply;
        let y_out = pool.reserve_y * lp_amount / pool.lp_supply;
        
        pool.reserve_x = pool.reserve_x - x_out;
        pool.reserve_y = pool.reserve_y - y_out;
        pool.lp_supply = pool.lp_supply - lp_amount;
        
        (x_out, y_out)
    }
    
    spec remove_liquidity<X, Y> {
        requires lp_amount > 0;
        requires lp_amount <= pool.lp_supply;
        requires pool.lp_supply > 0;
        
        // Both outputs positive  
        ensures result_1 >= 0 && result_2 >= 0;
        
        // LP supply decreases
        ensures pool.lp_supply == old(pool.lp_supply) - lp_amount;
        
        // Reserves decrease by output amounts
        ensures pool.reserve_x == old(pool.reserve_x) - result_1;
        ensures pool.reserve_y == old(pool.reserve_y) - result_2;
        
        aborts_if lp_amount > pool.lp_supply;
        aborts_if pool.lp_supply == 0;
    }
    
    fun sqrt(x: u64, y: u64): u64 {
        // Geometric mean sqrt(x*y)
        let product = (x as u128) * (y as u128);
        // Newton's method
        if (product == 0) return 0;
        let mut z = product;
        let mut last = 0u128;
        while (z != last) {
            last = z;
            z = (z + product / z) / 2;
        };
        z as u64
    }
}
```

---

## ตัวอย่าง: Vulnerable vs Secure Code

```move
module audit::case_study {
    
    // ============================================
    // Case Study: Vulnerable Yield Vault
    // (Inspired by real exploits)
    // ============================================
    
    // VULNERABLE VERSION
    module case_study::vault_v1 {
        struct Vault has key {
            total_shares: u64,
            total_assets: u64,  // WRONG: not updated from actual balance
        }
        
        struct UserPosition has key {
            shares: u64,
        }
        
        // BUG 1: assets() returns stale stored value, not actual balance
        public fun assets_per_share(vault: &Vault): u64 {
            if (vault.total_shares == 0) return 1_000_000;
            vault.total_assets * 1_000_000 / vault.total_shares
        }
        
        // BUG 2: share price manipulation via direct transfer
        // Attacker donates 1000 APT to vault BEFORE first depositor
        // First depositor: shares = 1 APT / 1001 APT * 0 shares → gets 0 shares!
        public entry fun deposit(
            user: &signer,
            vault_addr: address,
            amount: u64,
        ) acquires Vault {
            let vault = borrow_global_mut<Vault>(vault_addr);
            
            // If someone donated APT directly: total_assets is wrong
            let shares = if (vault.total_shares == 0) {
                amount
            } else {
                amount * vault.total_shares / vault.total_assets
            };
            
            vault.total_shares = vault.total_shares + shares;
            vault.total_assets = vault.total_assets + amount;  // manual tracking
            
            move_to(user, UserPosition { shares });
        }
    }
    
    // SECURE VERSION
    module case_study::vault_v2 {
        struct Vault has key {
            total_shares: u64,
            // Don't track assets manually - read actual balance
        }
        
        struct UserPosition has key {
            shares: u64,
        }
        
        // FIX: Read actual balance from coin store
        public fun total_assets(vault_addr: address): u64 {
            aptos_framework::coin::balance<AptosCoin>(vault_addr)
        }
        
        // FIX 1: Minimum shares prevents rounding to zero
        // FIX 2: Virtual offset prevents share price manipulation
        const VIRTUAL_SHARES: u64 = 1000;
        const VIRTUAL_ASSETS: u64 = 1000;
        
        public entry fun deposit(
            user: &signer,
            vault_addr: address,
            amount: u64,
        ) acquires Vault {
            let vault = borrow_global_mut<Vault>(vault_addr);
            
            // Use virtual offset (ERC4626 trick)
            let current_assets = total_assets(vault_addr) + VIRTUAL_ASSETS;
            let current_shares = vault.total_shares + VIRTUAL_SHARES;
            
            // shares = amount * total_shares / total_assets
            let shares = (amount as u128) * (current_shares as u128) 
                        / (current_assets as u128);
            let shares = shares as u64;
            
            // Minimum shares check
            assert!(shares > 0, 1);
            
            vault.total_shares = vault.total_shares + shares;
            
            // Transfer APT to vault (actual balance tracked by coin module)
            let coins = aptos_framework::coin::withdraw<AptosCoin>(user, amount);
            aptos_framework::coin::deposit<AptosCoin>(vault_addr, coins);
            
            move_to(user, UserPosition { shares });
        }
        
        public entry fun withdraw(
            user: &signer,
            vault_addr: address,
            shares: u64,
        ) acquires Vault, UserPosition {
            let user_addr = std::signer::address_of(user);
            let pos = borrow_global_mut<UserPosition>(user_addr);
            assert!(pos.shares >= shares, 1);
            
            let vault = borrow_global_mut<Vault>(vault_addr);
            
            let current_assets = total_assets(vault_addr) + VIRTUAL_ASSETS;
            let current_shares = vault.total_shares + VIRTUAL_SHARES;
            
            // assets_out = shares * total_assets / total_shares
            let assets_out = (shares as u128) * (current_assets as u128)
                            / (current_shares as u128);
            let assets_out = assets_out as u64;
            
            pos.shares = pos.shares - shares;
            vault.total_shares = vault.total_shares - shares;
            
            // Transfer from vault to user
            // (requires vault signer - needs resource account)
            // coin::transfer<AptosCoin>(&vault_signer, user_addr, assets_out);
        }
        
        struct AptosCoin {}
    }
    
    // ============================================
    // Case Study: Flash Loan Defense
    // ============================================
    
    module case_study::flash_defense {
        use aptos_framework::timestamp;
        
        struct OraclePrice has key {
            price: u64,
            last_update: u64,
        }
        
        // VULNERABLE: Uses price updated in same tx
        public entry fun update_and_liquidate_bad(
            operator: &signer,
            new_price: u64,
            borrower: address,
        ) acquires OraclePrice {
            let oracle = borrow_global_mut<OraclePrice>(@oracle);
            oracle.price = new_price;  // Update price
            oracle.last_update = timestamp::now_seconds();
            
            // Then immediately use new price for liquidation
            // Attacker can manipulate price before calling this!
            do_liquidation(borrower, oracle.price);
        }
        
        // SECURE: Time delay between price update and usage
        public entry fun update_price(
            operator: &signer,
            new_price: u64,
        ) acquires OraclePrice {
            let oracle = borrow_global_mut<OraclePrice>(@oracle);
            oracle.price = new_price;
            oracle.last_update = timestamp::now_seconds();
        }
        
        public entry fun liquidate(
            liquidator: &signer,
            borrower: address,
        ) acquires OraclePrice {
            let oracle = borrow_global<OraclePrice>(@oracle);
            
            // Price must be fresh but not JUST updated (at least 1 block old)
            let now = timestamp::now_seconds();
            assert!(now >= oracle.last_update, 1);  // Not from future
            assert!(now - oracle.last_update <= 3600, 2);  // Not stale (< 1 hour)
            // In Aptos: one block ≈ 2 seconds, so check if at least N blocks old
            
            do_liquidation(borrower, oracle.price);
        }
        
        fun do_liquidation(_borrower: address, _price: u64) {}
    }
}
```

---

## สรุป Security Audit Priorities

```
Top 10 Move Security Issues:

1. Integer overflow in fee/reward calculations
   Fix: Use u128 for intermediate values

2. Missing access control on admin functions
   Fix: Capability pattern or signer check

3. Spot price oracle manipulation
   Fix: TWAP + staleness checks + deviation check

4. Share price inflation (first depositor attack)
   Fix: Virtual offset (VIRTUAL_SHARES/ASSETS)

5. Flash loan attacks
   Fix: Time delay between update and use

6. Missing slippage/deadline protection
   Fix: min_out + deadline params on all swaps

7. Arithmetic underflow in reward claims
   Fix: if (earned >= claimed) checks

8. Incorrect resource lifecycle
   Fix: Test resource creation/destruction paths

9. Cross-module logical reentrancy
   Fix: Update state before external calls

10. Unchecked upgrade effects on stored data
    Fix: Data migration tests + prover invariants

Before Audit:
  - Run: aptos move prove
  - Run: aptos move test --coverage
  - Check coverage: >90% required
  - Review all #[friend] and public(friend)
  - Document all admin capabilities
```

---

**ก่อนหน้า**: [Part 45 - Liquid Staking ←](part-45-liquid-staking.md)
**ต่อไป**: [Part 47 - Game Theory in DeFi →](part-47-game-theory.md)
