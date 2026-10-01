# Part 34: Governance Systems & DAO

## สารบัญ
- [Governance Overview](#governance-overview)
- [Token-Based Voting](#token-based-voting)
- [Proposal Lifecycle](#proposal-lifecycle)
- [Timelock Controller](#timelock-controller)
- [Multi-sig Governance](#multi-sig-governance)
- [ตัวอย่าง: Complete DAO](#ตัวอย่าง-complete-dao)

---

## Governance Overview

```
DAO Architecture:
  Token Holders → Vote → Proposals → Timelock → Execution

Key Components:
  1. Governance Token (GOV) - สิทธิ์โหวต proportional to holdings
  2. Proposal System - สร้าง, โหวต, execute proposals
  3. Timelock - delay ก่อน execution (security buffer)
  4. Treasury - ควบคุม protocol funds

Voting Models:
  - Token-weighted (1 token = 1 vote)
  - Quadratic (sqrt of tokens = votes)  
  - veToken (time-locked tokens = more votes)
  - Conviction voting (accumulate over time)
```

---

## Token-Based Voting

```move
module dao::governance_token {
    use std::signer;
    use aptos_framework::coin::{Self, Coin, MintCapability, BurnCapability};
    use aptos_framework::timestamp;
    
    // ============================================
    // Governance Token with delegation support
    // ============================================
    
    struct GOV has key {}
    
    struct GovTokenState has key {
        mint_cap: MintCapability<GOV>,
        burn_cap: BurnCapability<GOV>,
        total_supply: u64,
        governance: address,
    }
    
    // Delegation: delegate voting power to another address
    struct Delegation has key {
        delegated_to: address,
        amount: u64,
        delegated_at: u64,
    }
    
    // Voting power snapshot (for governance attacks prevention)
    struct VotingPowerSnapshot has key {
        checkpoints: vector<Checkpoint>,
    }
    
    struct Checkpoint has copy, drop, store {
        block_time: u64,
        votes: u64,
    }
    
    public entry fun initialize(deployer: &signer, governance: address) {
        let (burn_cap, freeze_cap, mint_cap) = coin::initialize<GOV>(
            deployer,
            std::string::utf8(b"Governance Token"),
            std::string::utf8(b"GOV"),
            8,
            true,
        );
        coin::destroy_freeze_cap(freeze_cap);
        
        move_to(deployer, GovTokenState {
            mint_cap,
            burn_cap,
            total_supply: 0,
            governance,
        });
    }
    
    public entry fun mint_to(
        governance: &signer,
        state_addr: address,
        recipient: address,
        amount: u64,
    ) acquires GovTokenState {
        let state = borrow_global_mut<GovTokenState>(state_addr);
        assert!(signer::address_of(governance) == state.governance, 1);
        
        let coins = coin::mint(amount, &state.mint_cap);
        coin::deposit(recipient, coins);
        state.total_supply = state.total_supply + amount;
    }
    
    // ============================================
    // Delegation
    // ============================================
    
    public entry fun delegate(
        user: &signer,
        user_addr: address,
        to: address,
        amount: u64,
    ) {
        let now = timestamp::now_seconds();
        
        if (exists<Delegation>(user_addr)) {
            // Update existing delegation
            let del = borrow_global_mut<Delegation>(user_addr);
            del.delegated_to = to;
            del.amount = amount;
            del.delegated_at = now;
        } else {
            move_to(user, Delegation {
                delegated_to: to,
                amount,
                delegated_at: now,
            });
        };
    }
    
    // Get effective voting power (own balance + delegations received)
    #[view]
    public fun voting_power(
        state_addr: address,
        voter: address,
    ): u64 acquires Delegation {
        let own_balance = coin::balance<GOV>(voter);
        
        // Subtract delegated-out amount
        let delegated_out = if (exists<Delegation>(voter)) {
            borrow_global<Delegation>(voter).amount
        } else { 0 };
        
        let net = if (own_balance >= delegated_out) {
            own_balance - delegated_out
        } else { 0 };
        
        // In production: add delegated-in amounts (need index)
        net
    }
    
    // ============================================
    // Snapshot voting power at a point in time
    // Prevents flash loan attacks on governance
    // ============================================
    
    public fun snapshot_voting_power(
        voter: address,
        at_time: u64,
    ): u64 acquires VotingPowerSnapshot {
        if (!exists<VotingPowerSnapshot>(voter)) return 0;
        
        let snaps = borrow_global<VotingPowerSnapshot>(voter);
        let n = std::vector::length(&snaps.checkpoints);
        
        if (n == 0) return 0;
        
        // Binary search for checkpoint at or before at_time
        let mut lo = 0u64;
        let mut hi = n;
        
        while (lo < hi) {
            let mid = (lo + hi) / 2;
            let cp = std::vector::borrow(&snaps.checkpoints, mid);
            if (cp.block_time <= at_time) {
                lo = mid + 1;
            } else {
                hi = mid;
            };
        };
        
        if (lo == 0) return 0;
        std::vector::borrow(&snaps.checkpoints, lo - 1).votes
    }
}
```

---

## Proposal Lifecycle

```move
module dao::proposals {
    use std::signer;
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::timestamp;
    use aptos_framework::event;
    
    // ============================================
    // Proposal State Machine
    // Pending → Active → Succeeded/Defeated → Queued → Executed/Cancelled
    // ============================================
    
    const STATE_PENDING: u8 = 0;
    const STATE_ACTIVE: u8 = 1;
    const STATE_SUCCEEDED: u8 = 2;
    const STATE_DEFEATED: u8 = 3;
    const STATE_QUEUED: u8 = 4;
    const STATE_EXECUTED: u8 = 5;
    const STATE_CANCELLED: u8 = 6;
    const STATE_EXPIRED: u8 = 7;
    
    struct GovernanceConfig has key {
        admin: address,
        gov_token_addr: address,
        
        // Thresholds
        proposal_threshold: u64,   // min tokens to create proposal
        quorum_votes: u64,         // min votes needed for quorum
        
        // Timing (in seconds)
        voting_delay: u64,         // delay after proposal before voting starts
        voting_period: u64,        // how long voting lasts
        timelock_delay: u64,       // delay after succeeded before executable
        
        // Counters
        proposal_count: u64,
    }
    
    struct Proposal has store {
        id: u64,
        proposer: address,
        title: vector<u8>,
        description: vector<u8>,
        
        // Timing
        created_at: u64,
        vote_start: u64,
        vote_end: u64,
        
        // Votes
        votes_for: u64,
        votes_against: u64,
        votes_abstain: u64,
        
        // State
        state: u8,
        executed_at: u64,
        
        // Action (what to execute)
        target_module: vector<u8>,
        action_data: vector<u8>,
    }
    
    struct ProposalRegistry has key {
        proposals: SmartTable<u64, Proposal>,
        // track if address has voted on proposal
        votes: SmartTable<u64, SmartTable<address, u8>>,  // prop_id -> (voter -> choice)
    }
    
    #[event]
    struct ProposalCreated has drop, store {
        id: u64,
        proposer: address,
        title: vector<u8>,
        vote_start: u64,
        vote_end: u64,
    }
    
    #[event]
    struct VoteCast has drop, store {
        voter: address,
        proposal_id: u64,
        support: u8,      // 0=against, 1=for, 2=abstain
        votes: u64,
        reason: vector<u8>,
    }
    
    #[event]
    struct ProposalExecuted has drop, store {
        id: u64,
        executor: address,
        timestamp: u64,
    }
    
    const E_BELOW_THRESHOLD: u64 = 1;
    const E_WRONG_STATE: u64 = 2;
    const E_ALREADY_VOTED: u64 = 3;
    const E_QUORUM_NOT_MET: u64 = 4;
    const E_TIMELOCK_NOT_DONE: u64 = 5;
    
    // ============================================
    // Create Proposal
    // ============================================
    
    public entry fun create_proposal(
        proposer: &signer,
        registry_addr: address,
        config_addr: address,
        title: vector<u8>,
        description: vector<u8>,
        target_module: vector<u8>,
        action_data: vector<u8>,
    ) acquires GovernanceConfig, ProposalRegistry {
        let proposer_addr = signer::address_of(proposer);
        let config = borrow_global<GovernanceConfig>(config_addr);
        
        // Check proposer has enough tokens
        let voting_power = aptos_framework::coin::balance<dao::governance_token::GOV>(proposer_addr);
        assert!(voting_power >= config.proposal_threshold, E_BELOW_THRESHOLD);
        
        let now = timestamp::now_seconds();
        let vote_start = now + config.voting_delay;
        let vote_end = vote_start + config.voting_period;
        
        let config_mut = borrow_global_mut<GovernanceConfig>(config_addr);
        let proposal_id = config_mut.proposal_count + 1;
        config_mut.proposal_count = proposal_id;
        
        let registry = borrow_global_mut<ProposalRegistry>(registry_addr);
        smart_table::add(&mut registry.proposals, proposal_id, Proposal {
            id: proposal_id,
            proposer: proposer_addr,
            title,
            description,
            created_at: now,
            vote_start,
            vote_end,
            votes_for: 0,
            votes_against: 0,
            votes_abstain: 0,
            state: STATE_PENDING,
            executed_at: 0,
            target_module,
            action_data,
        });
        
        event::emit(ProposalCreated {
            id: proposal_id,
            proposer: proposer_addr,
            title,
            vote_start,
            vote_end,
        });
    }
    
    // ============================================
    // Cast Vote
    // ============================================
    
    public entry fun cast_vote(
        voter: &signer,
        registry_addr: address,
        config_addr: address,
        proposal_id: u64,
        support: u8,    // 0=against, 1=for, 2=abstain
        reason: vector<u8>,
    ) acquires GovernanceConfig, ProposalRegistry {
        let voter_addr = signer::address_of(voter);
        let registry = borrow_global_mut<ProposalRegistry>(registry_addr);
        
        // Check proposal is active
        let proposal = smart_table::borrow_mut(&mut registry.proposals, proposal_id);
        let now = timestamp::now_seconds();
        assert!(now >= proposal.vote_start && now <= proposal.vote_end, E_WRONG_STATE);
        assert!(proposal.state == STATE_PENDING || proposal.state == STATE_ACTIVE, E_WRONG_STATE);
        
        // Mark as active
        if (proposal.state == STATE_PENDING) {
            proposal.state = STATE_ACTIVE;
        };
        
        // Check not already voted
        let votes_for_prop = smart_table::borrow_mut(&mut registry.votes, proposal_id);
        assert!(!smart_table::contains(votes_for_prop, voter_addr), E_ALREADY_VOTED);
        
        // Get voting power (use snapshot from proposal creation for safety)
        let votes = aptos_framework::coin::balance<dao::governance_token::GOV>(voter_addr);
        
        // Record vote
        smart_table::add(votes_for_prop, voter_addr, support);
        
        if (support == 1) {
            proposal.votes_for = proposal.votes_for + votes;
        } else if (support == 0) {
            proposal.votes_against = proposal.votes_against + votes;
        } else {
            proposal.votes_abstain = proposal.votes_abstain + votes;
        };
        
        event::emit(VoteCast {
            voter: voter_addr,
            proposal_id,
            support,
            votes,
            reason,
        });
    }
    
    // ============================================
    // Queue (after voting succeeds)
    // ============================================
    
    public entry fun queue_proposal(
        registry_addr: address,
        config_addr: address,
        proposal_id: u64,
    ) acquires GovernanceConfig, ProposalRegistry {
        let config = borrow_global<GovernanceConfig>(config_addr);
        let registry = borrow_global_mut<ProposalRegistry>(registry_addr);
        let proposal = smart_table::borrow_mut(&mut registry.proposals, proposal_id);
        
        let now = timestamp::now_seconds();
        assert!(now > proposal.vote_end, E_WRONG_STATE);
        
        // Check quorum and majority
        let total_votes = proposal.votes_for + proposal.votes_against + proposal.votes_abstain;
        assert!(total_votes >= config.quorum_votes, E_QUORUM_NOT_MET);
        assert!(proposal.votes_for > proposal.votes_against, E_WRONG_STATE);
        
        proposal.state = STATE_QUEUED;
        // Timelock delay starts now
    }
    
    // ============================================
    // Execute
    // ============================================
    
    public entry fun execute_proposal(
        executor: &signer,
        registry_addr: address,
        config_addr: address,
        proposal_id: u64,
    ) acquires GovernanceConfig, ProposalRegistry {
        let config = borrow_global<GovernanceConfig>(config_addr);
        let registry = borrow_global_mut<ProposalRegistry>(registry_addr);
        let proposal = smart_table::borrow_mut(&mut registry.proposals, proposal_id);
        
        assert!(proposal.state == STATE_QUEUED, E_WRONG_STATE);
        
        let now = timestamp::now_seconds();
        // Timelock: must wait voting_period + timelock_delay after vote end
        let executable_at = proposal.vote_end + config.timelock_delay;
        assert!(now >= executable_at, E_TIMELOCK_NOT_DONE);
        
        proposal.state = STATE_EXECUTED;
        proposal.executed_at = now;
        
        // Execute the action
        // In production: call the target module with action_data
        // This requires dynamic dispatch which Move handles via entry functions
        
        event::emit(ProposalExecuted {
            id: proposal_id,
            executor: signer::address_of(executor),
            timestamp: now,
        });
    }
    
    // ============================================
    // Cancel proposal (by proposer)
    // ============================================
    
    public entry fun cancel_proposal(
        proposer: &signer,
        registry_addr: address,
        proposal_id: u64,
    ) acquires ProposalRegistry {
        let registry = borrow_global_mut<ProposalRegistry>(registry_addr);
        let proposal = smart_table::borrow_mut(&mut registry.proposals, proposal_id);
        
        assert!(signer::address_of(proposer) == proposal.proposer, 1);
        assert!(proposal.state < STATE_QUEUED, E_WRONG_STATE);
        
        proposal.state = STATE_CANCELLED;
    }
    
    // ============================================
    // View functions
    // ============================================
    
    #[view]
    public fun proposal_state(registry_addr: address, proposal_id: u64): u8 
    acquires ProposalRegistry {
        let registry = borrow_global<ProposalRegistry>(registry_addr);
        let now = timestamp::now_seconds();
        let p = smart_table::borrow(&registry.proposals, proposal_id);
        
        if (p.state == STATE_CANCELLED) return STATE_CANCELLED;
        if (p.state == STATE_EXECUTED) return STATE_EXECUTED;
        
        if (now < p.vote_start) return STATE_PENDING;
        if (now <= p.vote_end) return STATE_ACTIVE;
        
        // After voting
        if (p.votes_for > p.votes_against) STATE_SUCCEEDED
        else STATE_DEFEATED
    }
    
    #[view]
    public fun proposal_votes(registry_addr: address, id: u64): (u64, u64, u64) 
    acquires ProposalRegistry {
        let r = borrow_global<ProposalRegistry>(registry_addr);
        let p = smart_table::borrow(&r.proposals, id);
        (p.votes_for, p.votes_against, p.votes_abstain)
    }
}
```

---

## ตัวอย่าง: Complete DAO

```move
module dao::complete_dao {
    use std::signer;
    use aptos_framework::coin;
    use aptos_framework::timestamp;
    use aptos_framework::event;
    
    // ============================================
    // DAO Treasury management
    // ============================================
    
    struct Treasury has key {
        admin: address,  // governance contract
        budget_epoch: u64,
        epoch_budget: u64,
        spent_this_epoch: u64,
    }
    
    struct GrantProposal has store {
        recipient: address,
        amount: u64,
        purpose: vector<u8>,
        milestone_count: u64,
        milestones_completed: u64,
    }
    
    #[event]
    struct GrantPaid has drop, store {
        recipient: address,
        amount: u64,
        milestone: u64,
    }
    
    const EPOCH_DURATION: u64 = 2592000;  // 30 days
    const E_OVER_BUDGET: u64 = 1;
    const E_NOT_GOVERNANCE: u64 = 2;
    
    public entry fun execute_grant(
        governance: &signer,
        treasury_addr: address,
        recipient: address,
        amount: u64,
        milestone: u64,
    ) acquires Treasury {
        let treasury = borrow_global_mut<Treasury>(treasury_addr);
        assert!(signer::address_of(governance) == treasury.admin, E_NOT_GOVERNANCE);
        
        // Check epoch budget
        let now = timestamp::now_seconds();
        if (now > treasury.budget_epoch + EPOCH_DURATION) {
            treasury.budget_epoch = now;
            treasury.spent_this_epoch = 0;
        };
        
        assert!(
            treasury.spent_this_epoch + amount <= treasury.epoch_budget,
            E_OVER_BUDGET
        );
        treasury.spent_this_epoch = treasury.spent_this_epoch + amount;
        
        // Transfer APT from treasury to recipient
        // (treasury must hold APT coins)
        
        event::emit(GrantPaid { recipient, amount, milestone });
    }
    
    // ============================================
    // Parameter management via governance
    // ============================================
    
    struct ProtocolParams has key {
        governance: address,
        swap_fee_bps: u64,
        protocol_fee_bps: u64,
        treasury: address,
        max_price_impact_bps: u64,
    }
    
    // Called by governance after successful proposal
    public entry fun update_swap_fee(
        governance: &signer,
        params_addr: address,
        new_fee_bps: u64,
    ) acquires ProtocolParams {
        let params = borrow_global_mut<ProtocolParams>(params_addr);
        assert!(signer::address_of(governance) == params.governance, E_NOT_GOVERNANCE);
        assert!(new_fee_bps <= 1000, 100);  // Max 10%
        params.swap_fee_bps = new_fee_bps;
    }
    
    public entry fun update_treasury(
        governance: &signer,
        params_addr: address,
        new_treasury: address,
    ) acquires ProtocolParams {
        let params = borrow_global_mut<ProtocolParams>(params_addr);
        assert!(signer::address_of(governance) == params.governance, E_NOT_GOVERNANCE);
        params.treasury = new_treasury;
    }
    
    // ============================================
    // Emergency Guardian (can veto malicious proposals)
    // ============================================
    
    struct Guardian has key {
        address: address,
        can_veto_until: u64,  // guardian power expires
    }
    
    public entry fun veto_proposal(
        guardian: &signer,
        dao_addr: address,
        proposal_id: u64,
    ) acquires Guardian {
        let g = borrow_global<Guardian>(dao_addr);
        assert!(signer::address_of(guardian) == g.address, E_NOT_GOVERNANCE);
        assert!(timestamp::now_seconds() <= g.can_veto_until, 200);
        
        // Cancel the proposal
        // In production: call proposals::cancel with guardian authority
    }
    
    // ============================================
    // Participation incentives
    // ============================================
    
    struct VotingRewards has key {
        total_distributed: u64,
        reward_per_vote: u64,  // reward tokens per vote cast
    }
    
    struct UserVotingStats has key {
        total_votes_cast: u64,
        proposals_voted: vector<u64>,
        rewards_claimed: u64,
    }
    
    public entry fun claim_voting_rewards(
        user: &signer,
        rewards_addr: address,
    ) acquires VotingRewards, UserVotingStats {
        let user_addr = signer::address_of(user);
        let rewards = borrow_global<VotingRewards>(rewards_addr);
        let stats = borrow_global_mut<UserVotingStats>(user_addr);
        
        let claimable = stats.total_votes_cast * rewards.reward_per_vote 
            - stats.rewards_claimed;
        
        stats.rewards_claimed = stats.rewards_claimed + claimable;
        
        // Mint/transfer reward tokens to user
    }
    
    // ============================================
    // Quadratic voting (anti-whale)
    // ============================================
    
    public fun quadratic_votes(token_balance: u64): u64 {
        // votes = floor(sqrt(balance))
        // Using Newton's method for integer sqrt
        if (token_balance == 0) return 0;
        
        let mut x = token_balance;
        let mut y = (x + 1) / 2;
        while (y < x) {
            x = y;
            y = (x + token_balance / x) / 2;
        };
        x
    }
    
    #[test]
    fun test_quadratic() {
        assert!(quadratic_votes(100) == 10, 1);
        assert!(quadratic_votes(10000) == 100, 2);
        assert!(quadratic_votes(1000000) == 1000, 3);
    }
}
```

---

## สรุป Governance Patterns

| Component | Purpose | Key Parameters |
|-----------|---------|----------------|
| Governance Token | Voting rights | total_supply, distribution |
| Proposal System | Decision making | threshold, quorum, period |
| Timelock | Security delay | delay (48-72h typical) |
| Treasury | Fund management | budget_per_epoch |
| Guardian | Emergency veto | limited time, community-elected |

**DeFi Governance Timeline:**
```
Day 0: Proposal created (need 100K tokens)
Day 1-2: Voting delay (snapshot taken at Day 0)
Day 3-7: Voting period (5 days active)
Day 8-10: Timelock (48h delay)
Day 10: Execution window (7 days to execute)
```

---

**ก่อนหน้า**: [Part 33 - Oracle Integration ←](part-33-oracle-integration.md)
**ต่อไป**: [Part 35 - Advanced DeFi →](part-35-advanced-defi.md)
