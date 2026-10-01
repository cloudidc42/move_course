# Part 68: Insurance & Risk Protocols

## สารบัญ
- [DeFi Insurance Overview](#defi-insurance-overview)
- [Coverage Pool Architecture](#coverage-pool-architecture)
- [Claims Assessment System](#claims-assessment-system)
- [Risk Pricing (Actuarial Model)](#risk-pricing-actuarial-model)
- [Reinsurance & Risk Tranching](#reinsurance--risk-tranching)
- [Parametric Insurance](#parametric-insurance)
- [Full Implementation](#full-implementation)

---

## DeFi Insurance Overview

```
DeFi Insurance Problems:
  
  Traditional Insurance in Crypto:
    - Centralized (counterparty risk)
    - Slow claims process
    - Opaque pricing
    - Limited coverage for DeFi-specific risks
  
  DeFi Insurance Protocols:
    - Nexus Mutual: Cover token holders vote on claims
    - InsureAce: Multi-chain coverage
    - Sherlock: Security-focused, auditor staking
    - Bumper Finance: Downside protection for assets
  
  Covered Risks in DeFi:
    1. Smart contract bugs/hacks
    2. Oracle failures / manipulation
    3. Stablecoin de-peg events
    4. Bridge exploits
    5. Custodian insolvency
    6. Governance attacks
    7. Liquidation cascade failures
  
  Insurance Economics:
    - Premium income → Pool
    - Claims paid from Pool
    - Stakers earn premium income, bear claim risk
    - Actuarial pricing: Premium = Expected_Loss + Risk_Loading + Expenses
    
  Move Advantages:
    - Provable coverage terms (on-chain)
    - Automatic claim payout (no claims adjuster needed for parametric)
    - Transparent risk pool accounting
    - Composable with other DeFi protocols
```

---

## Coverage Pool Architecture

```move
module insurance::coverage_pool {
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::timestamp;
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // COVERAGE POOL
    // Stakers deposit capital, earn premiums
    // Pay out claims when events occur
    // ============================================
    
    // Coverage categories
    const COVER_SMART_CONTRACT: u8 = 1;
    const COVER_ORACLE_FAILURE: u8 = 2;
    const COVER_STABLECOIN_DEPEG: u8 = 3;
    const COVER_BRIDGE_EXPLOIT: u8 = 4;
    
    // Claim states
    const CLAIM_SUBMITTED: u8 = 0;
    const CLAIM_UNDER_REVIEW: u8 = 1;
    const CLAIM_APPROVED: u8 = 2;
    const CLAIM_REJECTED: u8 = 3;
    const CLAIM_PAID: u8 = 4;
    
    struct CoveragePool<phantom T> has key {
        // Capital pool
        staked_capital: Coin<T>,
        
        // Coverage tracking
        total_coverage_sold: u64,    // Total value currently covered
        max_coverage_ratio: u64,     // Max coverage / capital (e.g., 200% = 2x leverage)
        
        // Premium accounting
        premium_income: u128,        // Cumulative premiums collected
        claims_paid: u128,           // Cumulative claims paid
        
        // Staker shares
        total_shares: u64,
        staker_info: SmartTable<address, StakerInfo>,
        
        // Active policies
        policies: SmartTable<u64, Policy>,
        next_policy_id: u64,
        
        // Claims
        claims: SmartTable<u64, Claim>,
        next_claim_id: u64,
        
        // Assessors who vote on claims
        assessors: vector<address>,
        
        // Lock period (stakers can't withdraw during active claims)
        withdrawal_lock_period: u64,
    }
    
    struct StakerInfo has store {
        shares: u64,
        staked_amount: u64,
        staked_at: u64,
        last_claim_index: u64,  // For reward tracking
    }
    
    struct Policy has store, drop {
        id: u64,
        holder: address,
        covered_protocol: address,   // Protocol being insured
        coverage_type: u8,
        coverage_amount: u64,        // Max payout
        premium_per_second: u64,     // Continuous premium
        start_time: u64,
        end_time: u64,
        is_active: bool,
        last_premium_payment: u64,
    }
    
    struct Claim has store {
        id: u64,
        policy_id: u64,
        claimant: address,
        claimed_amount: u64,
        evidence_hash: vector<u8>,   // IPFS hash of evidence
        submitted_at: u64,
        status: u8,
        votes_for: u64,
        votes_against: u64,
        voted: SmartTable<address, bool>,
    }
    
    // ============================================
    // STAKING
    // ============================================
    
    public fun stake<T>(
        staker: &signer,
        pool_addr: address,
        amount: u64,
    ) acquires CoveragePool {
        let pool = borrow_global_mut<CoveragePool<T>>(pool_addr);
        let staker_addr = std::signer::address_of(staker);
        
        // Calculate shares to issue
        let capital = coin::value(&pool.staked_capital);
        let shares_to_issue = if (pool.total_shares == 0 || capital == 0) {
            amount  // 1:1 initial ratio
        } else {
            // shares = amount * total_shares / capital
            (amount as u128) * (pool.total_shares as u128) / (capital as u128) as u64
        };
        
        // Take capital
        let coins = coin::withdraw<T>(staker, amount);
        coin::merge(&mut pool.staked_capital, coins);
        
        // Issue shares
        pool.total_shares = pool.total_shares + shares_to_issue;
        
        if (smart_table::contains(&pool.staker_info, staker_addr)) {
            let info = smart_table::borrow_mut(&mut pool.staker_info, staker_addr);
            info.shares = info.shares + shares_to_issue;
            info.staked_amount = info.staked_amount + amount;
        } else {
            smart_table::add(&mut pool.staker_info, staker_addr, StakerInfo {
                shares: shares_to_issue,
                staked_amount: amount,
                staked_at: timestamp::now_microseconds(),
                last_claim_index: 0,
            });
        };
    }
    
    public fun unstake<T>(
        staker: &signer,
        pool_addr: address,
        shares: u64,
    ) acquires CoveragePool {
        let pool = borrow_global_mut<CoveragePool<T>>(pool_addr);
        let staker_addr = std::signer::address_of(staker);
        
        assert!(smart_table::contains(&pool.staker_info, staker_addr), 1);
        let info = smart_table::borrow_mut(&mut pool.staker_info, staker_addr);
        
        assert!(info.shares >= shares, 2);
        
        // Calculate capital to return
        let capital = coin::value(&pool.staked_capital);
        let capital_to_return = (shares as u128) * (capital as u128) / (pool.total_shares as u128);
        let capital_to_return = capital_to_return as u64;
        
        // Check sufficient free capital (not locked by active coverage)
        let locked_capital = pool.total_coverage_sold / 2; // Simplified
        let free_capital = if (capital > locked_capital) capital - locked_capital else 0;
        assert!(capital_to_return <= free_capital, 3);
        
        // Burn shares
        info.shares = info.shares - shares;
        pool.total_shares = pool.total_shares - shares;
        
        // Return capital
        let coins = coin::extract(&mut pool.staked_capital, capital_to_return);
        coin::deposit(staker_addr, coins);
    }
    
    // ============================================
    // POLICY PURCHASE
    // ============================================
    
    public fun buy_coverage<T>(
        buyer: &signer,
        pool_addr: address,
        covered_protocol: address,
        coverage_type: u8,
        coverage_amount: u64,
        duration_days: u64,
    ): u64 acquires CoveragePool {
        let pool = borrow_global_mut<CoveragePool<T>>(pool_addr);
        let buyer_addr = std::signer::address_of(buyer);
        
        // Check pool can support this coverage
        let capital = coin::value(&pool.staked_capital);
        let new_total_coverage = pool.total_coverage_sold + coverage_amount;
        let max_coverage = capital * pool.max_coverage_ratio / 100;
        assert!(new_total_coverage <= max_coverage, 4);
        
        // Calculate premium
        let annual_premium_rate = get_premium_rate(coverage_type, coverage_amount, capital);
        let duration_seconds = duration_days * 86_400 * 1_000_000; // microseconds
        let total_premium = (coverage_amount as u128) * (annual_premium_rate as u128) 
            * (duration_seconds as u128) / (365 * 86_400 * 1_000_000 * 10_000) as u128;
        let total_premium = total_premium as u64;
        
        // Collect premium
        let premium_coins = coin::withdraw<T>(buyer, total_premium);
        coin::merge(&mut pool.staked_capital, premium_coins);
        pool.premium_income = pool.premium_income + (total_premium as u128);
        
        // Issue policy
        let policy_id = pool.next_policy_id;
        pool.next_policy_id = policy_id + 1;
        
        let now = timestamp::now_microseconds();
        let policy = Policy {
            id: policy_id,
            holder: buyer_addr,
            covered_protocol,
            coverage_type,
            coverage_amount,
            premium_per_second: total_premium / duration_seconds * 1_000_000,
            start_time: now,
            end_time: now + duration_seconds,
            is_active: true,
            last_premium_payment: now,
        };
        
        smart_table::add(&mut pool.policies, policy_id, policy);
        pool.total_coverage_sold = pool.total_coverage_sold + coverage_amount;
        
        policy_id
    }
    
    // Calculate premium rate in basis points per year
    fun get_premium_rate(
        coverage_type: u8,
        coverage_amount: u64,
        pool_capital: u64,
    ): u64 {
        // Base rates by coverage type
        let base_rate = if (coverage_type == COVER_SMART_CONTRACT) {
            200  // 2% per year
        } else if (coverage_type == COVER_ORACLE_FAILURE) {
            100  // 1% per year  
        } else if (coverage_type == COVER_STABLECOIN_DEPEG) {
            300  // 3% per year
        } else {
            500  // 5% per year (bridge/other)
        };
        
        // Utilization loading: higher utilization → higher premium
        // utilization = coverage_sold / pool_capital
        let utilization_bps = if (pool_capital == 0) {
            10_000 // 100% if no capital
        } else {
            coverage_amount * 10_000 / pool_capital
        };
        
        // Loading factor: 1.0x to 2.0x based on utilization
        let loading = 100 + utilization_bps / 100; // 100% to 200%
        
        base_rate * loading / 100
    }
}
```

---

## Claims Assessment System

```move
module insurance::claims {
    use insurance::coverage_pool::{Self, CoveragePool};
    
    // ============================================
    // DECENTRALIZED CLAIMS ASSESSMENT
    // Assessors vote on claim validity
    // Majority wins
    // ============================================
    
    public fun submit_claim<T>(
        claimant: &signer,
        pool_addr: address,
        policy_id: u64,
        claimed_amount: u64,
        evidence_hash: vector<u8>,  // IPFS hash
    ): u64 acquires CoveragePool {
        let pool = borrow_global_mut<CoveragePool<T>>(pool_addr);
        let claimant_addr = std::signer::address_of(claimant);
        
        // Verify policy exists and is active
        assert!(aptos_std::smart_table::contains(&pool.policies, policy_id), 1);
        let policy = aptos_std::smart_table::borrow(&pool.policies, policy_id);
        assert!(policy.holder == claimant_addr, 2);
        assert!(policy.is_active, 3);
        assert!(claimed_amount <= policy.coverage_amount, 4);
        
        // Create claim
        let claim_id = pool.next_claim_id;
        pool.next_claim_id = claim_id + 1;
        
        let claim = insurance::coverage_pool::Claim {
            id: claim_id,
            policy_id,
            claimant: claimant_addr,
            claimed_amount,
            evidence_hash,
            submitted_at: aptos_framework::timestamp::now_microseconds(),
            status: insurance::coverage_pool::CLAIM_SUBMITTED,
            votes_for: 0,
            votes_against: 0,
            voted: aptos_std::smart_table::new(),
        };
        
        aptos_std::smart_table::add(&mut pool.claims, claim_id, claim);
        claim_id
    }
    
    // Assessor votes on claim
    public fun vote_on_claim<T>(
        assessor: &signer,
        pool_addr: address,
        claim_id: u64,
        approve: bool,
    ) acquires CoveragePool {
        let pool = borrow_global_mut<CoveragePool<T>>(pool_addr);
        let assessor_addr = std::signer::address_of(assessor);
        
        // Verify assessor
        assert!(is_assessor(&pool.assessors, assessor_addr), 1);
        
        let claim = aptos_std::smart_table::borrow_mut(&mut pool.claims, claim_id);
        
        // No double voting
        assert!(!aptos_std::smart_table::contains(&claim.voted, assessor_addr), 2);
        aptos_std::smart_table::add(&mut claim.voted, assessor_addr, true);
        
        if (approve) {
            claim.votes_for = claim.votes_for + 1;
        } else {
            claim.votes_against = claim.votes_against + 1;
        };
        
        // Check if threshold reached (majority of assessors)
        let total_assessors = std::vector::length(&pool.assessors);
        let threshold = total_assessors / 2 + 1; // Simple majority
        
        if (claim.votes_for >= threshold) {
            claim.status = insurance::coverage_pool::CLAIM_APPROVED;
        } else if (claim.votes_against >= threshold) {
            claim.status = insurance::coverage_pool::CLAIM_REJECTED;
        };
    }
    
    // Pay out approved claim
    public fun pay_claim<T>(
        pool_addr: address,
        claim_id: u64,
    ) acquires CoveragePool {
        let pool = borrow_global_mut<CoveragePool<T>>(pool_addr);
        
        let claim = aptos_std::smart_table::borrow_mut(&mut pool.claims, claim_id);
        assert!(claim.status == insurance::coverage_pool::CLAIM_APPROVED, 1);
        
        let payout = claim.claimed_amount;
        let claimant = claim.claimant;
        claim.status = insurance::coverage_pool::CLAIM_PAID;
        
        // Pay from pool
        assert!(aptos_framework::coin::value(&pool.staked_capital) >= payout, 2);
        let payout_coins = aptos_framework::coin::extract(&mut pool.staked_capital, payout);
        aptos_framework::coin::deposit(claimant, payout_coins);
        
        pool.claims_paid = pool.claims_paid + (payout as u128);
        
        // Mark policy as depleted if full coverage claimed
        // (simplified: in production handle partial claims)
    }
    
    fun is_assessor(assessors: &vector<address>, addr: address): bool {
        let len = std::vector::length(assessors);
        let mut i = 0u64;
        while (i < len) {
            if (*std::vector::borrow(assessors, i) == addr) return true;
            i = i + 1;
        };
        false
    }
}
```

---

## Parametric Insurance

```move
module insurance::parametric {
    use aptos_framework::timestamp;
    
    // ============================================
    // PARAMETRIC INSURANCE
    // Automatic payout when measurable event occurs
    // No claims assessment needed!
    //
    // Examples:
    // - Flight delay: Payout if flight >3h late
    // - Rainfall insurance: Payout if rain < X mm
    // - Stablecoin depeg: Payout if USDC < $0.95
    // - Protocol hack: Payout if TVL drops >50% in 1h
    // ============================================
    
    struct ParametricPolicy has key {
        id: u64,
        holder: address,
        
        // Trigger conditions
        trigger_type: u8,           // 0=price_depeg, 1=tvl_drop, 2=oracle_fail
        threshold: u64,             // Trigger threshold
        observation_window: u64,    // How long the condition must persist
        
        // Payout terms
        coverage_amount: u64,
        payout_percentage: u64,     // 0-100
        
        // Time bounds
        start_time: u64,
        end_time: u64,
        
        // State
        is_active: bool,
        triggered_at: u64,          // When condition first met (0 if not triggered)
        payout_processed: bool,
    }
    
    struct DepegInsurance has key {
        policies: aptos_std::smart_table::SmartTable<u64, ParametricPolicy>,
        next_id: u64,
        oracle_addr: address,       // Price feed for stablecoin
        stablecoin_addr: address,
        peg_price: u64,             // Expected price (e.g., 1_000_000 = $1.00 with 6 decimals)
        depeg_threshold_bps: u64,   // e.g., 500 = 5% depeg triggers payout
    }
    
    // Buy depeg insurance
    public entry fun buy_depeg_insurance<StableCoin, Premium>(
        buyer: &signer,
        insurance_addr: address,
        coverage_amount: u64,
        duration_days: u64,
        premium_amount: u64,
    ) acquires DepegInsurance {
        let insurance = borrow_global_mut<DepegInsurance>(insurance_addr);
        let buyer_addr = std::signer::address_of(buyer);
        let now = timestamp::now_microseconds();
        
        // Take premium
        let premium = aptos_framework::coin::withdraw<Premium>(buyer, premium_amount);
        // Deposit premium to pool...
        aptos_framework::coin::destroy_zero(
            aptos_framework::coin::extract(&mut premium, 0)
        );
        aptos_framework::coin::deposit(insurance_addr, premium);
        
        // Create policy
        let id = insurance.next_id;
        insurance.next_id = id + 1;
        
        let policy = ParametricPolicy {
            id,
            holder: buyer_addr,
            trigger_type: 0, // price_depeg
            threshold: insurance.peg_price * (10_000 - insurance.depeg_threshold_bps) / 10_000,
            observation_window: 3_600_000_000, // 1 hour in microseconds
            coverage_amount,
            payout_percentage: 100, // Full payout on trigger
            start_time: now,
            end_time: now + duration_days * 86_400 * 1_000_000,
            is_active: true,
            triggered_at: 0,
            payout_processed: false,
        };
        
        aptos_std::smart_table::add(&mut insurance.policies, id, policy);
    }
    
    // Check trigger condition (anyone can call)
    public fun check_trigger(
        insurance_addr: address,
        policy_id: u64,
    ) acquires DepegInsurance {
        let insurance = borrow_global_mut<DepegInsurance>(insurance_addr);
        let now = timestamp::now_microseconds();
        
        let policy = aptos_std::smart_table::borrow_mut(&mut insurance.policies, policy_id);
        
        assert!(policy.is_active, 1);
        assert!(now >= policy.start_time && now <= policy.end_time, 2);
        
        // Get current price from oracle
        let current_price = get_oracle_price(insurance.oracle_addr);
        
        if (current_price < policy.threshold) {
            // Price below threshold
            if (policy.triggered_at == 0) {
                // First time triggered
                policy.triggered_at = now;
            };
            
            // Check if observation window has passed
            let trigger_duration = now - policy.triggered_at;
            if (trigger_duration >= policy.observation_window && !policy.payout_processed) {
                // TRIGGER CONFIRMED - Process payout automatically
                process_payout_internal(policy);
            };
        } else {
            // Price recovered - reset trigger
            if (policy.triggered_at > 0) {
                policy.triggered_at = 0; // Reset
            };
        };
    }
    
    fun process_payout_internal(policy: &mut ParametricPolicy) {
        let payout = policy.coverage_amount * policy.payout_percentage / 100;
        policy.payout_processed = true;
        policy.is_active = false;
        
        // In production: transfer payout from pool to holder
        // pool::pay(policy.holder, payout);
        
        // Emit event
    }
    
    fun get_oracle_price(_oracle_addr: address): u64 {
        // Get price from oracle (Pyth, Chainlink, etc.)
        1_000_000 // Placeholder: $1.00
    }
}
```

---

## สรุป Insurance Protocols

```
Insurance Protocol Architecture:

CORE COMPONENTS
  1. Capital Pool
     - Stakers deposit → earn premiums
     - Shares represent pool ownership
     - Share value increases as premiums accrue
     
  2. Policy Management
     - Coverage terms on-chain (immutable)
     - Premium calculated actuarially
     - Continuous premiums (pay-as-you-go)
     
  3. Claims Assessment
     a. Discretionary: Assessors vote (Nexus Mutual model)
        - Pros: Handles complex cases
        - Cons: Slow, subjective, governance risk
     b. Parametric: Automatic trigger (event-based)
        - Pros: Fast, objective, trustless
        - Cons: Basis risk (event ≠ loss)
        
  4. Risk Pricing
     - Base rate by coverage type
     - Utilization loading (supply/demand)
     - Duration discount (longer = cheaper per day)
     - Protocol score adjustment (audited = cheaper)

PRICING FORMULA
  Annual Premium Rate = Base_Rate × Utilization_Loading × Protocol_Score
  
  Base rates (typical DeFi):
    Smart contract risk: 1-5% p.a.
    Oracle risk:         0.5-2% p.a.
    Stablecoin depeg:    1-10% p.a.
    Bridge exploit:      2-10% p.a.
    
POOL METRICS TO MONITOR
  Utilization = Coverage_Sold / Pool_Capital
  Target utilization: 30-70%
  
  Loss ratio = Claims_Paid / Premiums_Earned
  Healthy range: < 60%
  
  Combined ratio = Loss_ratio + Expense_ratio
  Break-even: < 100%
  
RISK TRANCHING
  Senior tranche: First loss protection (lower yield)
  Junior tranche: Last loss exposure (higher yield, first to lose)
  
  Allows institutional capital (senior) + retail (junior) to coexist
```

---

**ก่อนหน้า**: [Part 67 - Flash Loans ←](part-67-flash-loans.md)
**ต่อไป**: [Part 69 - Multi-Chain Token Standards →](part-69-multichain-tokens.md)
