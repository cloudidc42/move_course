# Part 76: Move Security Auditing

## สารบัญ
- [Auditing Methodology](#auditing-methodology)
- [Common Vulnerability Patterns](#common-vulnerability-patterns)
- [Move-Specific Security Checks](#move-specific-security-checks)
- [Audit Tooling](#audit-tooling)
- [Writing Audit Reports](#writing-audit-reports)
- [Bug Bounty Programs](#bug-bounty-programs)

---

## Auditing Methodology

```
Security Audit Process (Move Smart Contracts):

PHASE 1: UNDERSTANDING (2-3 days)
  1. Read all documentation (README, whitepaper, specs)
  2. Understand the protocol's intended behavior
  3. Map all user flows and state transitions
  4. Identify trust assumptions
  5. List all external dependencies (oracles, bridges)
  
PHASE 2: MANUAL REVIEW (5-10 days)
  1. Code walkthrough (all modules)
  2. Check each function against spec
  3. Look for logical errors in formulas
  4. Trace asset flows (where do tokens come from/go)
  5. Check all edge cases (zero, max values, empty state)
  6. Review access control for every function
  
PHASE 3: ADVERSARIAL THINKING (3-5 days)
  1. "How would I attack this protocol?"
  2. Economic attacks (MEV, oracle manipulation)
  3. Sequence attacks (specific order of transactions)
  4. State exhaustion (fill up data structures)
  5. Cross-module attacks
  
PHASE 4: AUTOMATED TOOLS (1-2 days)
  1. Move Prover (formal verification)
  2. Static analysis
  3. Fuzz testing
  4. Gas benchmarks
  
PHASE 5: REPORTING
  1. Categorize findings by severity
  2. Write reproduction steps
  3. Propose fixes
  4. Review fixes (1-2 rounds)

SEVERITY LEVELS
  Critical: Direct fund loss, protocol halt
  High:     Significant fund loss, major functionality broken
  Medium:   Partial fund loss, minor functionality broken
  Low:      Informational, best practice violations
  Info:     Gas optimizations, style issues
```

---

## Common Vulnerability Patterns

```move
// ============================================
// VULNERABILITY 1: INTEGER ARITHMETIC ERRORS
// ============================================

module vuln::bad_math {
    // BAD: Overflow/underflow
    public fun calculate_reward_BAD(
        staked: u64,
        duration: u64,
        rate_per_second: u64,
    ): u64 {
        // OVERFLOW: If staked=10^18, duration=31536000, rate=1000
        // staked * duration * rate > u64::MAX (18446744073709551615)
        staked * duration * rate_per_second  // ❌ Overflow!
    }
    
    // GOOD: Use u128 intermediate
    public fun calculate_reward_GOOD(
        staked: u64,
        duration: u64,
        rate_per_second: u64,
    ): u64 {
        let reward_u128 = (staked as u128) * (duration as u128) * (rate_per_second as u128);
        assert!(reward_u128 <= (std::u64::max_value() as u128), 1);
        reward_u128 as u64  // ✅ Safe
    }
    
    // BAD: Division before multiplication (precision loss)
    public fun calculate_fee_BAD(amount: u64, fee_bps: u64): u64 {
        amount / 10_000 * fee_bps  // ❌ Precision loss!
        // If amount = 100, fee_bps = 30: 100/10000 = 0, 0*30 = 0 (wrong!)
    }
    
    // GOOD: Multiply before divide
    public fun calculate_fee_GOOD(amount: u64, fee_bps: u64): u64 {
        amount * fee_bps / 10_000  // ✅ Correct
        // If amount = 100, fee_bps = 30: 100*30 = 3000, 3000/10000 = 0 (still 0 but expected)
        // Proper: use u128 for safety
    }
    
    // BAD: Wrong rounding direction for fees
    public fun fee_rounding_BAD(amount: u64): u64 {
        let fee = amount * 30 / 10_000;  // Floor rounding
        // User pays floor(fee), protocol loses dust over many txs
        fee
    }
    
    // GOOD: Ceiling rounding for fees charged to users
    public fun fee_rounding_GOOD(amount: u64): u64 {
        let fee = (amount * 30 + 9_999) / 10_000;  // Ceiling
        fee
    }
}

// ============================================
// VULNERABILITY 2: ACCESS CONTROL FAILURES
// ============================================

module vuln::bad_access {
    struct AdminConfig has key {
        admin: address,
        fee_bps: u64,
    }
    
    // BAD: No access check
    public entry fun set_fee_BAD(
        _caller: &signer,
        config_addr: address,
        new_fee: u64,
    ) acquires AdminConfig {
        // ❌ ANYONE can change fee!
        borrow_global_mut<AdminConfig>(config_addr).fee_bps = new_fee;
    }
    
    // BAD: Check the wrong address
    public entry fun set_fee_WRONG(
        caller: &signer,
        config_addr: address,
        new_fee: u64,
    ) acquires AdminConfig {
        let config = borrow_global<AdminConfig>(config_addr);
        // ❌ Bug: checking caller == config_addr (not caller == admin!)
        assert!(std::signer::address_of(caller) == config_addr, 1);
        borrow_global_mut<AdminConfig>(config_addr).fee_bps = new_fee;
    }
    
    // GOOD: Correct access check
    public entry fun set_fee_GOOD(
        caller: &signer,
        config_addr: address,
        new_fee: u64,
    ) acquires AdminConfig {
        let config = borrow_global<AdminConfig>(config_addr);
        // ✅ Check caller is admin
        assert!(std::signer::address_of(caller) == config.admin, 1);
        assert!(new_fee <= 1000, 2); // Bounds check too
        borrow_global_mut<AdminConfig>(config_addr).fee_bps = new_fee;
    }
}

// ============================================
// VULNERABILITY 3: MISSING CHECKS
// ============================================

module vuln::missing_checks {
    // BAD: No deadline check
    public fun swap_BAD(pool_addr: address, amount: u64, min_out: u64) {
        // ❌ No deadline: transaction can be pending for hours
        // Miner can delay execution until price moves unfavorably
    }
    
    // BAD: No slippage check
    public fun swap_no_slippage(pool_addr: address, amount: u64) {
        // ❌ No min_out: user can get 1 token even if expecting 1000
    }
    
    // BAD: No zero-amount check
    public fun deposit_BAD(amount: u64) {
        // ❌ If amount = 0: might still issue LP tokens
        // Division by zero if LP calculation uses amount
    }
    
    // GOOD: All checks present
    public fun swap_GOOD(
        pool_addr: address,
        amount: u64,
        min_out: u64,
        deadline: u64,
    ) {
        assert!(amount > 0, 1);           // Non-zero
        assert!(min_out > 0, 2);          // Slippage protection
        assert!(aptos_framework::timestamp::now_microseconds() <= deadline, 3); // Deadline
        // Execute swap...
    }
}

// ============================================
// VULNERABILITY 4: PRICE ORACLE MANIPULATION
// ============================================

module vuln::oracle_manipulation {
    struct Pool has key {
        reserve_x: u64,
        reserve_y: u64,
    }
    
    // BAD: Spot price as oracle
    public fun get_price_BAD(pool_addr: address): u64 acquires Pool {
        let pool = borrow_global<Pool>(pool_addr);
        // ❌ Spot price: manipulable by flash loan in same tx
        pool.reserve_y * 1_000_000 / pool.reserve_x
    }
    
    // GOOD: TWAP price
    public fun get_price_GOOD(oracle_addr: address): u64 {
        // ✅ Use 30-minute TWAP
        // oracle::get_twap_price(oracle_addr)
        0
    }
}
```

---

## Move-Specific Security Checks

```
MOVE-SPECIFIC VULNERABILITY CHECKLIST

1. ABILITY MISUSE
   [ ] Does struct have `drop` when it shouldn't? (receipt, loan obligation)
   [ ] Does struct have `copy` for unique resources? (NFTs, vouchers)
   [ ] Coins should NEVER have `drop` or `copy`
   
2. ACQUIRES ANNOTATION
   [ ] Every function that reads/writes global state has `acquires`
   [ ] `acquires` matches all resources actually accessed
   [ ] Cross-module calls may need to propagate `acquires`
   
3. SIGNER HANDLING
   [ ] Never store signer (you can only pass references)
   [ ] Signer cannot be created from address
   [ ] `signer::address_of()` gets address correctly
   
4. RESOURCE LIFECYCLE
   [ ] All resources are either stored (move_to) or destroyed properly
   [ ] No resource is accidentally dropped (would abort)
   [ ] `move_from` requires the resource to exist
   
5. VECTOR OPERATIONS
   [ ] Bounds checks before vector access
   [ ] Iterator indices correct (off-by-one)
   [ ] Empty vector edge cases handled
   
6. TYPE PHANTOM PARAMETERS
   [ ] Pool<X,Y> and Pool<Y,X> are different types (intentional?)
   [ ] Generic type constraints match intended use
   
7. FRIEND DECLARATIONS
   [ ] `friend` modules are minimal and necessary
   [ ] friend modules are trusted (attack surface analysis)
   
8. ENTRY FUNCTION SAFETY
   [ ] All entry functions have proper signer checks
   [ ] Arguments validated (length, bounds, type)
   [ ] Return values (entry functions can't return)
   
9. EVENT EMISSION
   [ ] Critical state changes emit events
   [ ] Events include enough info for monitoring
   [ ] No sensitive data in events
   
10. UPGRADE SAFETY
    [ ] Struct changes are backwards compatible
    [ ] Migration plan for incompatible upgrades
    [ ] Version tracking in key structs

APTOS-SPECIFIC CHECKS
  [ ] Coin withdrawals match expected amount (no gas deducted from value)
  [ ] Table/SmartTable doesn't grow unboundedly
  [ ] block::get_current_block_height used correctly
  [ ] timestamp::now_microseconds() vs now_seconds()
  
SUI-SPECIFIC CHECKS
  [ ] Shared vs owned object chosen correctly
  [ ] transfer vs public_transfer vs share_object
  [ ] Dynamic fields vs vector: right choice for use case
  [ ] Object wrapped correctly (can unwrap?)
  [ ] ID collision prevention in object creation
```

---

## Audit Tooling

```bash
# ============================================
# MOVE AUDIT TOOLING
# ============================================

# 1. MOVE PROVER (Formal Verification)
# Verify mathematical properties of code
aptos move prove \
  --package-dir . \
  --named-addresses mymod=0x1

# Prover can verify:
# - No arithmetic overflow (with spec)
# - Invariants hold across all calls
# - Access control correct (requires/ensures)

# 2. MOVE COVERAGE
# Check which code paths are tested
aptos move test \
  --coverage \
  --package-dir .

aptos move coverage summary \
  --package-dir .

# 3. GAS PROFILING
aptos move test \
  --gas-profiling \
  --package-dir .

# 4. STATIC ANALYSIS TOOLS
# External tools (as of 2024):
# - Move Static Analyzer (Mysten Labs internal)
# - Certora for Move (formal verifier)
# - OtterSec Move analysis
# - Verichains Move audit

# 5. MANUAL FUZZING SCRIPT
# (Run against testnet)

#!/usr/bin/env python3
"""Fuzzer for Move protocol functions"""
import random
import subprocess
import json

def fuzz_swap(pool_addr: str, iterations: int = 1000):
    """Generate random swap inputs and check for panics"""
    for i in range(iterations):
        amount = random.randint(1, 2**63 - 1)
        min_out = random.randint(0, amount)
        deadline = 9999999999999999
        
        cmd = [
            "aptos", "move", "run",
            "--function-id", f"{pool_addr}::pool::swap",
            "--args", f"u64:{amount}", f"u64:{min_out}", f"u64:{deadline}",
            "--network", "testnet",
        ]
        
        result = subprocess.run(cmd, capture_output=True, text=True)
        
        if "ARITHMETIC_ERROR" in result.stderr:
            print(f"OVERFLOW with amount={amount}, min_out={min_out}")
        elif "panic" in result.stderr.lower():
            print(f"PANIC with amount={amount}: {result.stderr[:200]}")

if __name__ == "__main__":
    fuzz_swap("0x1234567890abcdef")
```

---

## Writing Audit Reports

```markdown
# Security Audit Report Template

## [CRITICAL] Integer Overflow in calculate_reward()

**Severity**: Critical
**Location**: `staking.move:142`
**Category**: Arithmetic

### Description
The `calculate_reward()` function multiplies three u64 values without 
overflow protection, which can silently overflow and return incorrect 
(smaller) reward amounts, or in extreme cases return 0.

### Proof of Concept
```move
// Inputs that cause overflow:
staked = 1_000_000_000_000_000_000  // 1e18 tokens
duration = 31_536_000               // 1 year in seconds  
rate = 1_000                        // 0.001 per second

// result = staked * duration * rate
//        = 1e18 * 31536000 * 1000
//        = 3.15e28
//        > u64::MAX (1.84e19)  ← OVERFLOW!
```

### Impact
Users may receive incorrect (much lower) rewards. An attacker could 
time their staking to trigger overflow and steal from the reward pool.

### Recommendation
```move
// Fix: Use u128 intermediate
public fun calculate_reward(staked: u64, duration: u64, rate: u64): u64 {
    let reward = (staked as u128) * (duration as u128) * (rate as u128);
    assert!(reward <= (u64::max_value() as u128), ERR_OVERFLOW);
    reward as u64
}
```

### Status: Fixed in commit abc123
```

---

## Bug Bounty Programs

```
SETTING UP A BUG BOUNTY PROGRAM

PLATFORMS
  Immunefi: Largest DeFi bounties, good tooling
  HackerOne: General, good for non-DeFi
  Code4rena: Competitive audits (many researchers)
  Sherlock: Protocol-specific, auditors stake on coverage

BOUNTY STRUCTURE (Typical DeFi)
  Critical (direct fund loss):     $50,000 - $5,000,000
  High (significant impact):       $10,000 - $50,000
  Medium (partial impact):         $1,000 - $10,000
  Low (informational):             $100 - $1,000
  
WRITING GOOD SCOPE
  In scope:
    - All deployed smart contracts
    - Deployed SDKs that handle user funds
    - Admin/guardian systems
    
  Out of scope:
    - Theoretical economic attacks without PoC
    - Issues requiring physical access
    - Social engineering
    - Already-known issues
    
RESPONDING TO REPORTS
  1. Acknowledge within 24 hours
  2. Triage within 72 hours
  3. Fix within 30 days (critical: faster)
  4. Pay bounty promptly after fix
  5. Public disclosure after fix (with researcher credit)
  
IMMUNEFI SETUP
  1. Submit program to immunefi.com
  2. Fund bounty pool (escrow or on-chain)
  3. Define scope with contract addresses
  4. Specify triaging criteria
  5. Review team available (24/7 for critical)
  
MOVE PROTOCOL BOUNTY EXAMPLES
  Aptos Foundation: Has a security program
  Sui Foundation: security@sui.io for reports
  Major Aptos/Sui protocols: Check Immunefi
```

---

## สรุป Security Auditing

```
Auditor's Checklist (Move Protocol):

ARITHMETIC
  [ ] All multiplications: check u64 * u64 overflow
  [ ] Division before multiply: check precision loss
  [ ] Rounding direction: floor vs ceiling based on context
  [ ] Zero value edge cases
  
ACCESS CONTROL
  [ ] Every mutable function: who can call?
  [ ] Admin/owner checks correct address (not config_addr!)
  [ ] Role-based system covers all operations
  [ ] Two-step transfers for critical roles
  
ASSET SAFETY (Move-specific)
  [ ] Coin/Balance: properly extracted and deposited
  [ ] No accidental drops of non-droppable resources
  [ ] Flash loan receipts: no `drop` ability
  [ ] LP tokens: correct minting/burning math
  
ORACLE DEPENDENCIES
  [ ] TWAP not spot price
  [ ] Freshness checks (not stale)
  [ ] Min liquidity for price validity
  [ ] Multiple sources for critical prices
  
ECONOMIC MODELS
  [ ] Fee math correct (bps calculations)
  [ ] Incentive alignment (no perverse incentives)
  [ ] Game theory: can users profit by breaking protocol?
  [ ] Liquidation math: no bad debt scenario
  
OPERATIONAL
  [ ] Emergency pause works
  [ ] Pause doesn't lock funds permanently
  [ ] Upgrade path safe (data migration)
  [ ] Events sufficient for monitoring
```

---

**ก่อนหน้า**: [Part 75 - Capstone DeFi ←](part-75-capstone-defi.md)
**ต่อไป**: [Part 77 - Move on Sui: Advanced Patterns →](part-77-sui-advanced.md)
