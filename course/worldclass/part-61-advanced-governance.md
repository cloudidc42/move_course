# Part 61: Advanced Governance Systems

## สารบัญ
- [Governance Design Principles](#governance-design-principles)
- [On-Chain Governance Architecture](#on-chain-governance-architecture)
- [Timelock Controller](#timelock-controller)
- [veToken Governance (Vote-Escrow)](#vetoken-governance-vote-escrow)
- [Delegation & Liquid Governance](#delegation--liquid-governance)
- [Governance Attacks & Defense](#governance-attacks--defense)

---

## Governance Design Principles

```
Why On-Chain Governance Matters:
  - Protocol must evolve (market changes, hacks, new features)
  - Who controls upgrades = who controls the protocol
  - Decentralization = no single point of failure

Governance Design Spectrum:
  
  Centralized ←——————————→ Decentralized
  (team multisig)            (token holder vote)
  
  Fast                       Slow
  Efficient                  Secure
  Risky (team can rug)       Resilient (community decides)

Best Practice: Progressive Decentralization
  Phase 1: Team multi-sig (fast, controllable)
  Phase 2: Guardian + team (limited decentralization)
  Phase 3: Token governance + timelock (full on-chain)
  Phase 4: veToken + sub-DAOs (mature)

Governance Attack Vectors:
  1. Vote buying: "flash loan governance"
     - Borrow tokens → vote → return tokens
     - Defense: Snapshot voting (historical balance)
     
  2. Voter apathy: whales decide everything
     - Defense: quorum requirements, vote delegation
     
  3. Governance squatting: block legitimate proposals
     - Defense: minimum quorum, time limits
     
  4. Malicious proposal: fund transfer to attacker
     - Defense: timelock + guardian veto
```

---

## On-Chain Governance Architecture

```move
module governance::core {
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // Full On-Chain Governance
    // ============================================
    
    const VOTING_PERIOD: u64 = 3 * 24 * 3600;   // 3 days
    const QUEUE_PERIOD: u64 = 2 * 24 * 3600;    // 2 day timelock
    const EXECUTION_WINDOW: u64 = 7 * 24 * 3600; // 7 days to execute
    
    const MIN_PROPOSAL_THRESHOLD: u64 = 100_000_000_000;  // 1% of supply (100M tokens)
    const QUORUM_THRESHOLD_BPS: u64 = 400;  // 4% of total supply must vote
    
    const STATE_PENDING: u8 = 0;
    const STATE_ACTIVE: u8 = 1;
    const STATE_QUEUED: u8 = 2;
    const STATE_SUCCEEDED: u8 = 3;
    const STATE_EXECUTED: u8 = 4;
    const STATE_DEFEATED: u8 = 5;
    const STATE_EXPIRED: u8 = 6;
    const STATE_CANCELLED: u8 = 7;
    
    struct GovernorConfig has key {
        proposals: SmartTable<u64, Proposal>,
        next_proposal_id: u64,
        
        // Token info
        token_supply: u64,
        
        // Timelock
        timelock_addr: address,
        
        // Proposer requirements
        proposal_threshold: u64,
        
        // Guardian can veto/cancel
        guardian: address,
    }
    
    struct Proposal has store {
        id: u64,
        proposer: address,
        description: std::string::String,
        
        // Calldata (what to execute)
        targets: vector<address>,
        values: vector<u64>,
        calldatas: vector<vector<u8>>,
        
        // Timing
        vote_start: u64,
        vote_end: u64,
        
        // Votes (use snapshot at vote_start)
        votes_for: u64,
        votes_against: u64,
        votes_abstain: u64,
        
        // Voter tracking
        has_voted: SmartTable<address, bool>,
        
        // State
        state: u8,
        queued_at: std::option::Option<u64>,
        executed_at: std::option::Option<u64>,
        cancelled_at: std::option::Option<u64>,
    }
    
    // ============================================
    // Create proposal
    // ============================================
    
    public entry fun propose(
        proposer: &signer,
        governor_addr: address,
        description: std::string::String,
        targets: vector<address>,
        values: vector<u64>,
        calldatas: vector<vector<u8>>,
    ) acquires GovernorConfig {
        let governor = borrow_global_mut<GovernorConfig>(governor_addr);
        let proposer_addr = std::signer::address_of(proposer);
        
        // Check proposer has enough voting power
        let voting_power = get_voting_power(proposer_addr);
        assert!(voting_power >= governor.proposal_threshold, 1);
        
        // Validate calldata
        assert!(std::vector::length(&targets) == std::vector::length(&values), 2);
        assert!(std::vector::length(&targets) == std::vector::length(&calldatas), 3);
        assert!(std::vector::length(&targets) > 0, 4);
        assert!(std::vector::length(&targets) <= 10, 5);  // Max 10 actions per proposal
        
        let now = aptos_framework::timestamp::now_seconds();
        let proposal_id = governor.next_proposal_id;
        governor.next_proposal_id = proposal_id + 1;
        
        let proposal = Proposal {
            id: proposal_id,
            proposer: proposer_addr,
            description,
            targets,
            values,
            calldatas,
            vote_start: now + 86400,  // 1 day delay before voting
            vote_end: now + 86400 + VOTING_PERIOD,
            votes_for: 0,
            votes_against: 0,
            votes_abstain: 0,
            has_voted: smart_table::new(),
            state: STATE_PENDING,
            queued_at: std::option::none(),
            executed_at: std::option::none(),
            cancelled_at: std::option::none(),
        };
        
        smart_table::add(&mut governor.proposals, proposal_id, proposal);
    }
    
    // ============================================
    // Cast vote
    // ============================================
    
    public entry fun cast_vote(
        voter: &signer,
        governor_addr: address,
        proposal_id: u64,
        support: u8,  // 0=against, 1=for, 2=abstain
    ) acquires GovernorConfig {
        let governor = borrow_global_mut<GovernorConfig>(governor_addr);
        let voter_addr = std::signer::address_of(voter);
        
        let proposal = smart_table::borrow_mut(&mut governor.proposals, proposal_id);
        let now = aptos_framework::timestamp::now_seconds();
        
        assert!(now >= proposal.vote_start, 1);
        assert!(now <= proposal.vote_end, 2);
        assert!(!smart_table::contains(&proposal.has_voted, voter_addr), 3);
        
        // Get voting power (snapshot at vote_start to prevent flash loan attacks)
        let voting_power = get_historical_voting_power(voter_addr, proposal.vote_start);
        assert!(voting_power > 0, 4);
        
        smart_table::add(&mut proposal.has_voted, voter_addr, true);
        
        if (support == 0) {
            proposal.votes_against = proposal.votes_against + voting_power;
        } else if (support == 1) {
            proposal.votes_for = proposal.votes_for + voting_power;
        } else {
            proposal.votes_abstain = proposal.votes_abstain + voting_power;
        };
        
        // Update state
        proposal.state = STATE_ACTIVE;
    }
    
    // ============================================
    // Queue (after vote passes)
    // ============================================
    
    public entry fun queue(
        caller: &signer,
        governor_addr: address,
        proposal_id: u64,
    ) acquires GovernorConfig {
        let governor = borrow_global_mut<GovernorConfig>(governor_addr);
        let proposal = smart_table::borrow_mut(&mut governor.proposals, proposal_id);
        
        let now = aptos_framework::timestamp::now_seconds();
        assert!(now > proposal.vote_end, 1);
        
        // Check quorum
        let total_votes = proposal.votes_for + proposal.votes_against + proposal.votes_abstain;
        let quorum = governor.token_supply * QUORUM_THRESHOLD_BPS / 10_000;
        assert!(total_votes >= quorum, 2);
        
        // Check vote result
        assert!(proposal.votes_for > proposal.votes_against, 3);
        
        proposal.state = STATE_QUEUED;
        proposal.queued_at = std::option::some(now);
    }
    
    // ============================================
    // Execute (after timelock)
    // ============================================
    
    public entry fun execute(
        caller: &signer,
        governor_addr: address,
        proposal_id: u64,
    ) acquires GovernorConfig {
        let governor = borrow_global_mut<GovernorConfig>(governor_addr);
        let proposal = smart_table::borrow_mut(&mut governor.proposals, proposal_id);
        
        assert!(proposal.state == STATE_QUEUED, 1);
        
        let queued_at = *std::option::borrow(&proposal.queued_at);
        let now = aptos_framework::timestamp::now_seconds();
        
        // Timelock delay must have passed
        assert!(now >= queued_at + QUEUE_PERIOD, 2);
        
        // Execution window must not have expired
        assert!(now <= queued_at + QUEUE_PERIOD + EXECUTION_WINDOW, 3);
        
        proposal.state = STATE_EXECUTED;
        proposal.executed_at = std::option::some(now);
        
        // Execute each action (simplified - actual dispatch requires capability)
        // for each (target, value, calldata) in proposal:
        //   call target::function(calldata)
    }
    
    // ============================================
    // Guardian veto
    // ============================================
    
    public entry fun cancel(
        caller: &signer,
        governor_addr: address,
        proposal_id: u64,
        reason: std::string::String,
    ) acquires GovernorConfig {
        let governor = borrow_global_mut<GovernorConfig>(governor_addr);
        let caller_addr = std::signer::address_of(caller);
        
        let proposal = smart_table::borrow_mut(&mut governor.proposals, proposal_id);
        
        // Guardian can cancel any proposal, proposer can cancel their own
        assert!(
            caller_addr == governor.guardian || caller_addr == proposal.proposer,
            1
        );
        
        // Cannot cancel executed proposals
        assert!(proposal.state != STATE_EXECUTED, 2);
        
        proposal.state = STATE_CANCELLED;
        proposal.cancelled_at = std::option::some(aptos_framework::timestamp::now_seconds());
    }
    
    fun get_voting_power(_addr: address): u64 { 0 }
    fun get_historical_voting_power(_addr: address, _timestamp: u64): u64 { 0 }
}
```

---

## veToken Governance (Vote-Escrow)

```move
module governance::ve_token {
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::coin;
    
    // ============================================
    // veToken: Lock tokens for voting power
    // Inspired by Curve's veCRV
    // ============================================
    
    // Mechanics:
    //   Lock TOKEN for 1-4 years
    //   Receive veToken (non-transferable, decays)
    //   veToken = locked_amount * (time_remaining / MAX_LOCK)
    //   veToken gives: voting power + fee share boost + gauge weight
    
    const MAX_LOCK_TIME: u64 = 4 * 365 * 24 * 3600;  // 4 years
    const MIN_LOCK_TIME: u64 = 7 * 24 * 3600;         // 1 week
    const WEEK: u64 = 7 * 24 * 3600;
    
    struct VeTokenSystem has key {
        // User locks
        locks: SmartTable<address, Lock>,
        
        // Total supply checkpoints (for historical queries)
        epoch: u64,
        point_history: vector<Point>,
        slope_changes: SmartTable<u64, i64>,  // timestamp → slope change
        
        // Total locked
        total_locked: u64,
    }
    
    struct Lock has store, copy, drop {
        amount: u64,
        end_time: u64,  // Round to next week
    }
    
    struct Point has store, copy, drop {
        bias: i64,   // Current veToken balance = bias
        slope: i64,  // Rate of decay per second
        timestamp: u64,
        block: u64,
    }
    
    // Create lock or add to existing
    public entry fun create_lock(
        user: &signer,
        system_addr: address,
        amount: u64,
        lock_duration: u64,  // In seconds
    ) acquires VeTokenSystem {
        let user_addr = std::signer::address_of(user);
        let system = borrow_global_mut<VeTokenSystem>(system_addr);
        
        assert!(!smart_table::contains(&system.locks, user_addr), 1);
        assert!(lock_duration >= MIN_LOCK_TIME, 2);
        assert!(lock_duration <= MAX_LOCK_TIME, 3);
        assert!(amount > 0, 4);
        
        let now = aptos_framework::timestamp::now_seconds();
        
        // Round end to next week (prevents gaming)
        let end_time = round_to_week(now + lock_duration);
        
        // Pull tokens
        let tokens = coin::withdraw<ProtocolToken>(user, amount);
        coin::deposit(system_addr, tokens);
        
        system.total_locked = system.total_locked + amount;
        
        smart_table::add(&mut system.locks, user_addr, Lock { amount, end_time });
        
        checkpoint_user(system, user_addr, 0, 0, amount, end_time, now);
    }
    
    // Extend lock duration
    public entry fun extend_lock(
        user: &signer,
        system_addr: address,
        new_end_time: u64,
    ) acquires VeTokenSystem {
        let user_addr = std::signer::address_of(user);
        let system = borrow_global_mut<VeTokenSystem>(system_addr);
        let now = aptos_framework::timestamp::now_seconds();
        
        assert!(smart_table::contains(&system.locks, user_addr), 1);
        let lock = smart_table::borrow_mut(&mut system.locks, user_addr);
        
        assert!(now < lock.end_time, 2);  // Not expired
        let new_end_rounded = round_to_week(new_end_time);
        assert!(new_end_rounded > lock.end_time, 3);  // Only extend, not shorten
        assert!(new_end_rounded <= now + MAX_LOCK_TIME, 4);
        
        let old_end = lock.end_time;
        lock.end_time = new_end_rounded;
        
        checkpoint_user(system, user_addr, lock.amount, old_end, lock.amount, new_end_rounded, now);
    }
    
    // Get current veToken balance (decays over time)
    public fun balance_of(system: &VeTokenSystem, user_addr: address): u64 {
        if (!smart_table::contains(&system.locks, user_addr)) return 0;
        
        let lock = smart_table::borrow(&system.locks, user_addr);
        let now = aptos_framework::timestamp::now_seconds();
        
        if (now >= lock.end_time) return 0;
        
        // veBalance = amount * (remaining / max_lock)
        let remaining = lock.end_time - now;
        (lock.amount as u128) * (remaining as u128) / (MAX_LOCK_TIME as u128) as u64
    }
    
    // Withdraw after lock expires
    public entry fun withdraw(
        user: &signer,
        system_addr: address,
    ) acquires VeTokenSystem {
        let user_addr = std::signer::address_of(user);
        let system = borrow_global_mut<VeTokenSystem>(system_addr);
        
        let lock = smart_table::remove(&mut system.locks, user_addr);
        let now = aptos_framework::timestamp::now_seconds();
        
        assert!(now >= lock.end_time, 1);
        
        system.total_locked = system.total_locked - lock.amount;
        
        // Return tokens
        // coin::transfer<ProtocolToken>(system_signer, user_addr, lock.amount);
    }
    
    fun round_to_week(timestamp: u64): u64 {
        (timestamp / WEEK + 1) * WEEK
    }
    
    fun checkpoint_user(
        _system: &mut VeTokenSystem,
        _user: address,
        _old_amount: u64, _old_end: u64,
        _new_amount: u64, _new_end: u64,
        _now: u64,
    ) {
        // Update slope changes and point history for historical queries
    }
    
    struct ProtocolToken has store {}
}
```

---

## Governance Attacks & Defense

```move
module governance::defense {
    
    // ============================================
    // Governance Attack Mitigations
    // ============================================
    
    // 1. Flash Loan Attack Defense: Snapshot Voting
    //    Problem: Borrow tokens → vote → return
    //    Solution: Use balance at proposal creation time
    
    struct VotingSnapshot has key {
        // Historical balance checkpoints
        // user → sorted list of (timestamp, balance) pairs
        checkpoints: aptos_std::smart_table::SmartTable<address, vector<Checkpoint>>,
    }
    
    struct Checkpoint has store, copy, drop {
        timestamp: u64,
        balance: u64,
    }
    
    // Get balance at specific timestamp (for governance snapshots)
    public fun get_past_balance(
        snapshots: &VotingSnapshot,
        user: address,
        timestamp: u64,
    ): u64 {
        if (!aptos_std::smart_table::contains(&snapshots.checkpoints, user)) {
            return 0
        };
        
        let checkpoints = aptos_std::smart_table::borrow(&snapshots.checkpoints, user);
        let len = std::vector::length(checkpoints);
        
        if (len == 0) return 0;
        
        // Binary search for the checkpoint at/before timestamp
        let mut lo = 0u64;
        let mut hi = len;
        
        while (lo < hi) {
            let mid = (lo + hi) / 2;
            let cp = std::vector::borrow(checkpoints, mid);
            
            if (cp.timestamp <= timestamp) {
                lo = mid + 1;
            } else {
                hi = mid;
            };
        };
        
        if (lo == 0) {
            0  // No checkpoint before timestamp
        } else {
            std::vector::borrow(checkpoints, lo - 1).balance
        }
    }
    
    // 2. Quorum Defense: Prevent minority control
    //    Minimum 4% of total supply must participate
    
    // 3. Timelock Defense: Give community time to react
    //    48-hour delay between vote passing and execution
    //    Guardian can veto during timelock window
    
    // 4. Proposal Spam Defense: Require minimum tokens to propose
    
    // 5. Vote Delegation Attack:
    //    Problem: Delegate to self, then to another = double count
    //    Solution: Track delegation chain, prevent cycles
    
    struct DelegationGraph has key {
        // Who each address delegates to
        delegations: aptos_std::smart_table::SmartTable<address, address>,
        
        // Effective delegated power
        delegated_power: aptos_std::smart_table::SmartTable<address, u64>,
    }
    
    public entry fun delegate(
        delegator: &signer,
        graph_addr: address,
        delegatee: address,
    ) acquires DelegationGraph {
        let graph = borrow_global_mut<DelegationGraph>(graph_addr);
        let delegator_addr = std::signer::address_of(delegator);
        
        // Prevent self-delegation (cycles)
        assert!(delegator_addr != delegatee, 1);
        
        // Prevent circular delegation (A→B→A)
        let mut current = delegatee;
        let mut depth = 0u64;
        
        while (aptos_std::smart_table::contains(&graph.delegations, current)) {
            current = *aptos_std::smart_table::borrow(&graph.delegations, current);
            assert!(current != delegator_addr, 2);  // No cycle
            depth = depth + 1;
            assert!(depth < 10, 3);  // Max delegation depth
        };
        
        // Remove old delegation
        if (aptos_std::smart_table::contains(&graph.delegations, delegator_addr)) {
            let old_delegatee = *aptos_std::smart_table::borrow(&graph.delegations, delegator_addr);
            remove_delegation_power(&mut graph.delegated_power, old_delegatee, delegator_addr);
        };
        
        // Add new delegation
        aptos_std::smart_table::upsert(&mut graph.delegations, delegator_addr, delegatee);
        
        let voting_power = get_own_voting_power(delegator_addr);
        let delegatee_power = if (aptos_std::smart_table::contains(&graph.delegated_power, delegatee)) {
            *aptos_std::smart_table::borrow(&graph.delegated_power, delegatee)
        } else { 0 };
        
        aptos_std::smart_table::upsert(
            &mut graph.delegated_power,
            delegatee,
            delegatee_power + voting_power,
        );
    }
    
    fun remove_delegation_power(
        _delegated_power: &mut aptos_std::smart_table::SmartTable<address, u64>,
        _delegatee: address,
        _delegator: address,
    ) {}
    
    fun get_own_voting_power(_addr: address): u64 { 0 }
}
```

---

## สรุป Advanced Governance

```
Governance Best Practices:

1. START SIMPLE, INCREASE COMPLEXITY GRADUALLY
   - Team multisig → guardian + token → full on-chain
   - Don't start with complex veToken governance

2. PROPOSAL LIFECYCLE
   Pending → Active (voting) → Queued (timelock) → Executed
   
   Alternative paths: Defeated, Cancelled, Expired

3. ANTI-ATTACK MEASURES
   ✓ Snapshot voting (not current balance)
   ✓ 4% quorum minimum
   ✓ 48-72h timelock
   ✓ Guardian veto (emergency only)
   ✓ Max delegation depth
   ✓ Proposal expiry

4. veToken MECHANICS
   veBAL = BAL_locked * (lock_remaining / max_lock)
   
   Benefits:
   - Rewards long-term holders
   - Discourages short-term speculation
   - Aligns incentives with protocol
   
   Risks:
   - Less liquidity (locked tokens)
   - Whale accumulation
   - Death spiral if price drops

5. GOVERNANCE MINIMIZATION
   Less governance = fewer attack surfaces
   Only govern: fees, parameters, upgrades
   Don't govern: core math, security invariants
   
   "Governance minimization" is a feature!

Real World Examples:
  Uniswap: Simple token governance, works well
  Curve: veToken, complex but battle-tested
  Compound: Governor Bravo, industry standard
  MakerDAO: Complex, many incidents, but survived
```

---

**ก่อนหน้า**: [Part 60 - Production Monitoring ←](part-60-production-monitoring.md)
**ต่อไป**: [Part 62 - Perpetual DEX Architecture →](part-62-perpetual-dex.md)
