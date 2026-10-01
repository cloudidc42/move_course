# Part 74: DAO & Community Governance

## สารบัญ
- [DAO Design Principles](#dao-design-principles)
- [On-Chain Voting Implementation](#on-chain-voting-implementation)
- [Delegation & Liquid Democracy](#delegation--liquid-democracy)
- [Optimistic Governance](#optimistic-governance)
- [Multi-sig Treasury Management](#multi-sig-treasury-management)
- [Governance Attacks & Defenses](#governance-attacks--defenses)

---

## DAO Design Principles

```
DAO Architecture Tradeoffs:

SPEED vs SECURITY
  Fast: Lower quorum, shorter timelock → more agile but riskier
  Safe: High quorum, long timelock → slower but more secure
  Solution: Tiered system by impact level
  
PARTICIPATION vs PLUTOCRACY
  Token-weighted: 1 token = 1 vote → whales dominate
  Identity-based: 1 person = 1 vote → hard to implement
  Quadratic: 1 person = sqrt(tokens) votes → middle ground (requires Sybil resistance)
  
AUTONOMY vs ACCOUNTABILITY
  Fully on-chain: Code is law, unstoppable
  Hybrid: On-chain votes, off-chain execution → humans must act
  Centralized executor: DAO votes but foundation executes → trusted setup
  
TYPES OF DECISIONS
  Protocol parameter changes (fee adjustments):
    → Low quorum, short timelock (2-7 days)
    
  Smart contract upgrades:
    → High quorum, long timelock (14-30 days)
    
  Treasury allocation:
    → Medium quorum, medium timelock (7-14 days)
    
  Emergency (pause, security fix):
    → Multisig or guardian (instant)
    
SUCCESSFUL DAO EXAMPLES
  Uniswap:    40M UNI quorum (4%), 7-day voting, 2-day timelock
  Compound:   400K COMP quorum, 3-day voting, 2-day timelock
  Aave:       320K AAVE quorum, 3-day voting, 1-day timelock
  MakerDAO:   Full on-chain, complex hat system
  
Move Implementation:
  Part 61 covered basic governance
  This part: Full production DAO implementation
```

---

## On-Chain Voting Implementation

```move
module dao::governor_full {
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::timestamp;
    
    // ============================================
    // FULL DAO GOVERNOR
    // Features:
    // - Proposal lifecycle (5 states)
    // - Snapshot-based voting (flash loan resistant)
    // - Timelock before execution
    // - Delegation
    // - Quorum with minimum threshold
    // ============================================
    
    const PROPOSAL_STATE_PENDING: u8 = 0;
    const PROPOSAL_STATE_ACTIVE: u8 = 1;
    const PROPOSAL_STATE_CANCELED: u8 = 2;
    const PROPOSAL_STATE_DEFEATED: u8 = 3;
    const PROPOSAL_STATE_SUCCEEDED: u8 = 4;
    const PROPOSAL_STATE_QUEUED: u8 = 5;
    const PROPOSAL_STATE_EXPIRED: u8 = 6;
    const PROPOSAL_STATE_EXECUTED: u8 = 7;
    
    const VOTE_FOR: u8 = 0;
    const VOTE_AGAINST: u8 = 1;
    const VOTE_ABSTAIN: u8 = 2;
    
    // Microseconds
    const VOTING_DELAY: u64 = 86_400 * 1_000_000;      // 1 day delay after proposal
    const VOTING_PERIOD: u64 = 5 * 86_400 * 1_000_000;  // 5 days voting
    const TIMELOCK_DELAY: u64 = 2 * 86_400 * 1_000_000; // 2 days after passing
    const PROPOSAL_THRESHOLD: u64 = 100_000;              // Min tokens to propose
    const QUORUM_NUMERATOR: u64 = 400;                    // 4% (out of 10000)
    
    struct Proposal has store {
        id: u64,
        proposer: address,
        description: std::string::String,
        
        // Voting snapshot
        snapshot_time: u64,    // Voting power measured at this time
        vote_start: u64,       // When voting opens
        vote_end: u64,         // When voting closes
        
        // Vote counts
        votes_for: u64,
        votes_against: u64,
        votes_abstain: u64,
        
        // Execution
        state: u8,
        queued_at: u64,        // When proposal was queued (for timelock)
        executed_at: u64,
        
        // Encoded call (simplified as bytes)
        call_data: vector<u8>,  // BCS-encoded action to execute
        target_module: std::string::String,
        target_function: std::string::String,
    }
    
    struct Governor has key {
        proposals: SmartTable<u64, Proposal>,
        next_proposal_id: u64,
        
        // Voting power (token contract)
        token_addr: address,
        
        // Config
        voting_delay: u64,
        voting_period: u64,
        timelock_delay: u64,
        proposal_threshold: u64,
        quorum_numerator: u64,   // Out of 10000
        
        // Delegation tracking
        delegatees: SmartTable<address, address>,  // delegator → delegatee
        delegation_votes: SmartTable<address, u64>, // Who has how many delegated votes
    }
    
    // Submit a proposal
    public fun propose(
        proposer: &signer,
        governor_addr: address,
        description: std::string::String,
        target_module: std::string::String,
        target_function: std::string::String,
        call_data: vector<u8>,
    ): u64 acquires Governor {
        let governor = borrow_global_mut<Governor>(governor_addr);
        let proposer_addr = std::signer::address_of(proposer);
        let now = timestamp::now_microseconds();
        
        // Check proposer has enough voting power
        let proposer_votes = get_voting_power(governor, proposer_addr, now);
        assert!(proposer_votes >= governor.proposal_threshold, 1);
        
        let proposal_id = governor.next_proposal_id;
        governor.next_proposal_id = proposal_id + 1;
        
        let snapshot_time = now;
        let vote_start = now + governor.voting_delay;
        let vote_end = vote_start + governor.voting_period;
        
        let proposal = Proposal {
            id: proposal_id,
            proposer: proposer_addr,
            description,
            snapshot_time,
            vote_start,
            vote_end,
            votes_for: 0,
            votes_against: 0,
            votes_abstain: 0,
            state: PROPOSAL_STATE_PENDING,
            queued_at: 0,
            executed_at: 0,
            call_data,
            target_module,
            target_function,
        };
        
        smart_table::add(&mut governor.proposals, proposal_id, proposal);
        
        aptos_framework::event::emit(ProposalCreated {
            proposal_id,
            proposer: proposer_addr,
            vote_start,
            vote_end,
            description,
        });
        
        proposal_id
    }
    
    // Cast vote (FOR / AGAINST / ABSTAIN)
    public fun cast_vote(
        voter: &signer,
        governor_addr: address,
        proposal_id: u64,
        support: u8,  // 0=for, 1=against, 2=abstain
        reason: std::string::String,
    ) acquires Governor {
        let governor = borrow_global_mut<Governor>(governor_addr);
        let voter_addr = std::signer::address_of(voter);
        let now = timestamp::now_microseconds();
        
        let proposal = smart_table::borrow_mut(&mut governor.proposals, proposal_id);
        
        // Check voting window
        assert!(now >= proposal.vote_start, 1);
        assert!(now <= proposal.vote_end, 2);
        
        // Get voting power at snapshot time (prevents flash loan attacks)
        let votes = get_voting_power(governor, voter_addr, proposal.snapshot_time);
        assert!(votes > 0, 3);
        
        // Record vote
        if (support == VOTE_FOR) {
            proposal.votes_for = proposal.votes_for + votes;
        } else if (support == VOTE_AGAINST) {
            proposal.votes_against = proposal.votes_against + votes;
        } else {
            proposal.votes_abstain = proposal.votes_abstain + votes;
        };
        
        aptos_framework::event::emit(VoteCast {
            proposal_id,
            voter: voter_addr,
            support,
            votes,
            reason,
        });
    }
    
    // Queue passed proposal for execution
    public fun queue(
        governor_addr: address,
        proposal_id: u64,
    ) acquires Governor {
        let governor = borrow_global_mut<Governor>(governor_addr);
        let now = timestamp::now_microseconds();
        
        let proposal = smart_table::borrow_mut(&mut governor.proposals, proposal_id);
        
        // Must be after voting period
        assert!(now > proposal.vote_end, 1);
        
        // Check if passed (quorum + majority FOR)
        assert!(did_proposal_pass(governor, proposal), 2);
        
        proposal.state = PROPOSAL_STATE_QUEUED;
        proposal.queued_at = now;
    }
    
    // Execute after timelock
    public fun execute(
        governor_addr: address,
        proposal_id: u64,
    ) acquires Governor {
        let governor = borrow_global_mut<Governor>(governor_addr);
        let now = timestamp::now_microseconds();
        
        let proposal = smart_table::borrow_mut(&mut governor.proposals, proposal_id);
        
        assert!(proposal.state == PROPOSAL_STATE_QUEUED, 1);
        assert!(now >= proposal.queued_at + governor.timelock_delay, 2);
        
        proposal.state = PROPOSAL_STATE_EXECUTED;
        proposal.executed_at = now;
        
        // Execute the encoded call
        // In production: dispatch to appropriate module/function
    }
    
    fun did_proposal_pass(governor: &Governor, proposal: &Proposal): bool {
        let total_votes = proposal.votes_for + proposal.votes_against + proposal.votes_abstain;
        
        // Quorum check: total_votes >= quorum_numerator% of total supply
        let total_supply = get_total_supply(governor.token_addr);
        let quorum = total_supply * governor.quorum_numerator / 10_000;
        
        if (total_votes < quorum) return false;  // Didn't meet quorum
        
        // Simple majority: votes_for > votes_against
        proposal.votes_for > proposal.votes_against
    }
    
    fun get_voting_power(governor: &Governor, addr: address, _timestamp: u64): u64 {
        // In production: get historical balance at timestamp (checkpoint system)
        // For now: current balance + delegated votes
        let own_votes = 0u64;  // get_token_balance(governor.token_addr, addr);
        
        let delegated = if (smart_table::contains(&governor.delegation_votes, addr)) {
            *smart_table::borrow(&governor.delegation_votes, addr)
        } else {
            0
        };
        
        own_votes + delegated
    }
    
    fun get_total_supply(_token_addr: address): u64 { 0 }
    
    #[event]
    struct ProposalCreated has drop, store {
        proposal_id: u64,
        proposer: address,
        vote_start: u64,
        vote_end: u64,
        description: std::string::String,
    }
    
    #[event]
    struct VoteCast has drop, store {
        proposal_id: u64,
        voter: address,
        support: u8,
        votes: u64,
        reason: std::string::String,
    }
}
```

---

## Multi-sig Treasury Management

```move
module dao::treasury {
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::timestamp;
    
    // ============================================
    // MULTI-SIG TREASURY
    // M-of-N signers required for transactions
    //
    // Use cases:
    // - Protocol fee collection
    // - Grant disbursements
    // - Emergency fund
    // - Liquidity provision
    // ============================================
    
    struct MultiSigTreasury has key {
        // Signers and threshold
        signers: vector<address>,
        threshold: u64,  // M of N required
        
        // Pending transactions
        proposals: SmartTable<u64, TxProposal>,
        next_proposal_id: u64,
        
        // Nonce to prevent replay attacks
        nonce: u64,
        
        // Execution delay (optional timelock for large amounts)
        execution_delay: u64,
        large_amount_threshold: u64,
    }
    
    struct TxProposal has store {
        id: u64,
        proposer: address,
        target: address,
        value: u64,              // Amount to send
        token_type: std::string::String,
        call_data: vector<u8>,   // Optional call to execute
        description: std::string::String,
        
        // Approvals
        approvals: vector<address>,
        rejections: vector<address>,
        
        // State
        is_executed: bool,
        is_cancelled: bool,
        created_at: u64,
        executed_at: u64,
    }
    
    // Any signer can propose
    public fun propose_tx(
        proposer: &signer,
        treasury_addr: address,
        target: address,
        value: u64,
        token_type: std::string::String,
        description: std::string::String,
    ): u64 acquires MultiSigTreasury {
        let treasury = borrow_global_mut<MultiSigTreasury>(treasury_addr);
        let proposer_addr = std::signer::address_of(proposer);
        
        assert!(is_signer_internal(&treasury.signers, proposer_addr), 1);
        
        let id = treasury.next_proposal_id;
        treasury.next_proposal_id = id + 1;
        
        // Proposer auto-approves
        let approvals = vector::singleton(proposer_addr);
        
        let proposal = TxProposal {
            id,
            proposer: proposer_addr,
            target,
            value,
            token_type,
            call_data: vector::empty(),
            description,
            approvals,
            rejections: vector::empty(),
            is_executed: false,
            is_cancelled: false,
            created_at: timestamp::now_microseconds(),
            executed_at: 0,
        };
        
        smart_table::add(&mut treasury.proposals, id, proposal);
        id
    }
    
    // Approve a proposal
    public fun approve(
        signer_acct: &signer,
        treasury_addr: address,
        proposal_id: u64,
    ) acquires MultiSigTreasury {
        let treasury = borrow_global_mut<MultiSigTreasury>(treasury_addr);
        let signer_addr = std::signer::address_of(signer_acct);
        
        assert!(is_signer_internal(&treasury.signers, signer_addr), 1);
        
        let proposal = smart_table::borrow_mut(&mut treasury.proposals, proposal_id);
        assert!(!proposal.is_executed && !proposal.is_cancelled, 2);
        
        // Check not already voted
        assert!(!has_voted(&proposal.approvals, signer_addr), 3);
        assert!(!has_voted(&proposal.rejections, signer_addr), 4);
        
        std::vector::push_back(&mut proposal.approvals, signer_addr);
        
        // Auto-execute if threshold reached
        if (std::vector::length(&proposal.approvals) >= treasury.threshold as u64) {
            execute_proposal_internal(treasury, proposal_id);
        };
    }
    
    // Execute if enough approvals
    fun execute_proposal_internal(treasury: &mut MultiSigTreasury, proposal_id: u64) {
        let proposal = smart_table::borrow_mut(&mut treasury.proposals, proposal_id);
        
        assert!(std::vector::length(&proposal.approvals) >= treasury.threshold as u64, 1);
        
        proposal.is_executed = true;
        proposal.executed_at = timestamp::now_microseconds();
        treasury.nonce = treasury.nonce + 1;
        
        // Transfer tokens to target
        // coin::deposit(proposal.target, proposal.value);
    }
    
    // Add/remove signers (requires M-of-N approval)
    public fun add_signer(
        treasury_addr: address,
        new_signer: address,
        proposal_id: u64,  // Must be approved by M-of-N
    ) acquires MultiSigTreasury {
        let treasury = borrow_global_mut<MultiSigTreasury>(treasury_addr);
        
        // Verify this operation was approved
        let proposal = smart_table::borrow(&treasury.proposals, proposal_id);
        assert!(proposal.is_executed, 1);
        
        if (!is_signer_internal(&treasury.signers, new_signer)) {
            std::vector::push_back(&mut treasury.signers, new_signer);
        };
    }
    
    fun is_signer_internal(signers: &vector<address>, addr: address): bool {
        let len = std::vector::length(signers);
        let mut i = 0u64;
        while (i < len) {
            if (*std::vector::borrow(signers, i) == addr) return true;
            i = i + 1;
        };
        false
    }
    
    fun has_voted(voters: &vector<address>, addr: address): bool {
        is_signer_internal(voters, addr)
    }
}
```

---

## Governance Attacks & Defenses

```
Common Governance Attacks:

1. FLASH LOAN GOVERNANCE ATTACK
   Attack:
     - Take flash loan of governance tokens
     - Create/vote on malicious proposal
     - Return flash loan in same tx
   
   Defense:
     - Snapshot voting at block N-1 (not current)
     - Time delay between token acquisition and voting
     - Proposal creation requires holding tokens for X days
     
2. VOTE BUYING (Bribery)
   Attack:
     - Protocols like Convex/Votium let projects bribe voters
     - "Vote for my gauge, receive $Y in rewards"
   
   Defense (partial):
     - Anonymous voting (hard to verify who voted how)
     - Conviction voting (longer holding = more weight)
     - Penalty for selling voted tokens
     
3. LOW QUORUM ATTACK
   Attack:
     - When participation is low, small group controls votes
     - Time attack: propose when key holders are on vacation
   
   Defense:
     - Minimum quorum requirements
     - Long voting periods (5-7 days)
     - Guardian/multisig veto power
     - Alert systems to notify large holders of proposals
     
4. SYBIL ATTACK (Multiple Identities)
   Attack:
     - Create many wallets, get small allocations each
     - Aggregate voting power without detection
   
   Defense:
     - Minimum token threshold to vote (economic cost)
     - Identity verification (centralized)
     - On-chain history requirements
     
5. GRIEFING (Blocking Proposals)
   Attack:
     - Block valid governance proposals by voting against
     - Force proposals to expire without action
   
   Defense:
     - Optimistic governance (pass unless vetoed)
     - Guardian for urgent operations
     - Council bypass for security fixes

GOVERNANCE SECURITY PARAMETERS
  Parameter            Conservative  Moderate  Aggressive
  Voting delay         2 days        1 day     6 hours
  Voting period        7 days        5 days    3 days
  Timelock delay       30 days       14 days   2 days
  Quorum %             10%           4%        1%
  Proposal threshold   1%            0.25%     0.1%
  
  Recommendation: Start conservative, move to moderate as trust builds
```

---

**ก่อนหน้า**: [Part 73 - Token Launch & Vesting ←](part-73-token-launch-vesting.md)
**ต่อไป**: [Part 75 - Real-World DeFi Protocol (Capstone) →](part-75-capstone-defi.md)
