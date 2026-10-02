# Part 93: Advanced Governance Systems

## สารบัญ
- [Governance Architecture](#governance-architecture)
- [Token-Weighted Voting](#token-weighted-voting)
- [Quadratic Voting](#quadratic-voting)
- [Optimistic Governance](#optimistic-governance)
- [Governance Security](#governance-security)
- [DAO Treasury Management](#dao-treasury-management)

---

## Governance Architecture

```
GOVERNANCE SYSTEM LAYERS

Layer 1: EMERGENCY (2-of-3 multisig)
  ├── Pause/unpause protocol
  ├── Emergency fund recovery
  └── 10-minute timelock

Layer 2: GUARDIAN (5-of-9 multisig)
  ├── Protocol parameter changes
  ├── Strategy additions/removals
  └── 48-hour timelock

Layer 3: GOVERNANCE (Token vote)
  ├── Major protocol upgrades
  ├── Treasury allocations
  ├── New product launches
  └── 7-day vote + 48h timelock

Layer 4: DAO (Full decentralization)
  ├── All protocol decisions
  ├── On-chain execution
  └── Optimistic with veto

PROPOSAL LIFECYCLE
  1. Create Proposal
     └── Stake PROT tokens (anti-spam)
  2. Review Period (24-48h)
     └── Community discussion
  3. Voting Period (5-7 days)
     └── Token holders vote FOR/AGAINST/ABSTAIN
  4. Execution Queue (48h timelock)
     └── If passed, waits in queue
  5. Execute
     └── Call on-chain action(s)
  6. Post-Execution Monitoring

QUORUM REQUIREMENTS
  Quorum: 4% of circulating supply must vote
  Passing: >50% of votes must be FOR
  Fast-track: 10% quorum + 67% FOR (for urgent fixes)
```

---

## Token-Weighted Voting

```move
// ============================================
// ON-CHAIN GOVERNANCE WITH TIMELOCK
// ============================================

module protocol::governance {
    use aptos_framework::timestamp;
    use aptos_framework::coin;
    
    const ERR_NOT_PROPOSER: u64 = 1;
    const ERR_VOTING_NOT_OPEN: u64 = 2;
    const ERR_ALREADY_VOTED: u64 = 3;
    const ERR_PROPOSAL_FAILED: u64 = 4;
    const ERR_TIMELOCK_NOT_PASSED: u64 = 5;
    const ERR_ALREADY_EXECUTED: u64 = 6;
    const ERR_QUORUM_NOT_MET: u64 = 7;
    
    /// Governance configuration
    struct GovConfig has key {
        // Token requirements
        proposal_threshold: u64,   // Min tokens to create proposal
        quorum_votes: u64,         // Min total votes for quorum
        
        // Timing (in seconds)
        voting_delay: u64,         // Wait after proposal before voting starts
        voting_period: u64,        // How long voting lasts
        timelock_delay: u64,       // Wait after passing before execution
        
        // Counters
        proposal_count: u64,
    }
    
    /// A governance proposal
    struct Proposal has key {
        id: u64,
        proposer: address,
        description: vector<u8>,
        
        // Actions to execute (encoded as bytes)
        target_functions: vector<vector<u8>>,
        
        // Timing
        start_time: u64,     // When voting begins
        end_time: u64,       // When voting ends
        eta: u64,            // Earliest execution time (after timelock)
        
        // Vote tallies
        votes_for: u64,
        votes_against: u64,
        votes_abstain: u64,
        
        // State
        canceled: bool,
        executed: bool,
        
        // Proposer's stake (returned if proposal passes, slashed if spam)
        stake_amount: u64,
    }
    
    /// Voter's receipt for a specific proposal
    struct VoteReceipt has key {
        proposal_id: u64,
        voter: address,
        support: u8,      // 0=against, 1=for, 2=abstain
        weight: u64,      // Tokens at snapshot
        reason: vector<u8>,
    }
    
    /// Snapshot of voting power at proposal creation
    struct VotingPowerSnapshot has key {
        proposal_id: u64,
        snapshots: aptos_std::table::Table<address, u64>,
    }
    
    public entry fun create_proposal(
        proposer: &signer,
        gov_addr: address,
        description: vector<u8>,
        stake_amount: u64,
    ) {
        let config = borrow_global_mut<GovConfig>(gov_addr);
        let proposer_addr = std::signer::address_of(proposer);
        
        // Check proposer has enough tokens
        // let proposer_balance = coin::balance<GovToken>(proposer_addr);
        // assert!(proposer_balance >= config.proposal_threshold, 1);
        
        // Stake tokens as anti-spam
        // coin::transfer<GovToken>(proposer, @gov_escrow, stake_amount);
        
        let now = timestamp::now_microseconds();
        let start_time = now + config.voting_delay * 1_000_000;
        let end_time = start_time + config.voting_period * 1_000_000;
        
        let proposal_id = config.proposal_count;
        config.proposal_count = config.proposal_count + 1;
        
        move_to(proposer, Proposal {
            id: proposal_id,
            proposer: proposer_addr,
            description,
            target_functions: vector::empty(),
            start_time,
            end_time,
            eta: 0,
            votes_for: 0,
            votes_against: 0,
            votes_abstain: 0,
            canceled: false,
            executed: false,
            stake_amount,
        });
    }
    
    /// Cast a vote on a proposal
    public entry fun cast_vote(
        voter: &signer,
        proposal_addr: address,
        support: u8,         // 0=against, 1=for, 2=abstain
        reason: vector<u8>,
    ) {
        let voter_addr = std::signer::address_of(voter);
        let proposal = borrow_global_mut<Proposal>(proposal_addr);
        
        let now = timestamp::now_microseconds();
        assert!(
            now >= proposal.start_time && now <= proposal.end_time,
            ERR_VOTING_NOT_OPEN,
        );
        
        // Check voter hasn't already voted
        assert!(!has_voted(voter_addr, proposal.id), ERR_ALREADY_VOTED);
        
        // Get voting weight (from snapshot or current balance)
        let weight = get_voting_power(voter_addr);
        
        // Record vote
        if (support == 1) {
            proposal.votes_for = proposal.votes_for + weight;
        } else if (support == 0) {
            proposal.votes_against = proposal.votes_against + weight;
        } else {
            proposal.votes_abstain = proposal.votes_abstain + weight;
        };
        
        // Store receipt
        move_to(voter, VoteReceipt {
            proposal_id: proposal.id,
            voter: voter_addr,
            support,
            weight,
            reason,
        });
    }
    
    /// Queue a passed proposal for execution
    public entry fun queue_proposal(
        anyone: &signer,
        proposal_addr: address,
        gov_addr: address,
    ) {
        let config = borrow_global<GovConfig>(gov_addr);
        let proposal = borrow_global_mut<Proposal>(proposal_addr);
        
        let now = timestamp::now_microseconds();
        assert!(now > proposal.end_time, ERR_VOTING_NOT_OPEN);
        assert!(proposal.votes_for > proposal.votes_against, ERR_PROPOSAL_FAILED);
        
        // Check quorum
        let total_votes = proposal.votes_for + proposal.votes_against + proposal.votes_abstain;
        assert!(total_votes >= config.quorum_votes, ERR_QUORUM_NOT_MET);
        
        // Set execution time (after timelock)
        proposal.eta = now + config.timelock_delay * 1_000_000;
        
        let _ = anyone;
    }
    
    /// Execute a queued proposal
    public entry fun execute_proposal(
        anyone: &signer,
        proposal_addr: address,
    ) {
        let proposal = borrow_global_mut<Proposal>(proposal_addr);
        
        assert!(!proposal.executed, ERR_ALREADY_EXECUTED);
        assert!(proposal.eta > 0, ERR_PROPOSAL_FAILED);
        assert!(
            timestamp::now_microseconds() >= proposal.eta,
            ERR_TIMELOCK_NOT_PASSED,
        );
        
        proposal.executed = true;
        
        // Execute each action in the proposal
        // For each function in target_functions:
        //   dispatch(function_bytes)
        
        let _ = anyone;
    }
    
    /// Cancel a proposal (only proposer or guardian can cancel)
    public entry fun cancel_proposal(
        canceler: &signer,
        proposal_addr: address,
        gov_addr: address,
    ) {
        let proposal = borrow_global_mut<Proposal>(proposal_addr);
        let canceler_addr = std::signer::address_of(canceler);
        
        // Only proposer or guardian can cancel
        let is_proposer = canceler_addr == proposal.proposer;
        // let is_guardian = guardian::is_guardian(canceler_addr);
        assert!(is_proposer, ERR_NOT_PROPOSER);
        
        proposal.canceled = true;
        
        // Return stake to proposer
        // coin::transfer<GovToken>(gov_signer, proposal.proposer, proposal.stake_amount);
        let _ = gov_addr;
    }
    
    fun has_voted(_voter: address, _proposal_id: u64): bool {
        false  // Check VoteReceipt exists at voter address
    }
    
    fun get_voting_power(voter: address): u64 {
        // Return veToken balance or governance token balance
        let _ = voter;
        0  // Placeholder
    }
    
    #[view]
    public fun get_proposal_state(proposal_addr: address): u8 {
        let proposal = borrow_global<Proposal>(proposal_addr);
        let now = timestamp::now_microseconds();
        
        if (proposal.canceled) return 6;      // CANCELED
        if (proposal.executed) return 5;      // EXECUTED
        if (now < proposal.start_time) return 1;  // PENDING
        if (now <= proposal.end_time) return 2;   // ACTIVE
        if (proposal.votes_for <= proposal.votes_against) return 3;  // DEFEATED
        if (proposal.eta == 0) return 4;      // SUCCEEDED (not queued)
        if (now < proposal.eta) return 7;     // QUEUED
        return 8;                             // READY TO EXECUTE
    }
}
```

---

## Quadratic Voting

```move
// ============================================
// QUADRATIC VOTING: sqrt(tokens) = votes
// Reduces whale dominance
// ============================================

module protocol::quadratic_gov {
    
    struct QVProposal has key {
        id: u64,
        description: vector<u8>,
        votes_for: u128,      // Sum of sqrt(tokens) for FOR
        votes_against: u128,
        total_voters: u64,
        credits_used: aptos_std::table::Table<address, u64>,
        end_time: u64,
        executed: bool,
    }
    
    /// Integer square root (Babylonian method)
    fun isqrt(n: u128): u128 {
        if (n == 0) return 0;
        let x = n;
        let y = (x + 1) / 2;
        let result = x;
        loop {
            if (y >= result) return result;
            result = y;
            y = (result + n / result) / 2;
        }
    }
    
    /// Vote with quadratic weight
    /// voter spends credit_tokens, gets sqrt(credit_tokens) vote power
    public fun cast_qv_vote(
        voter_addr: address,
        proposal: &mut QVProposal,
        credit_tokens: u64,
        support: bool,
    ) {
        assert!(
            aptos_framework::timestamp::now_microseconds() < proposal.end_time,
            1,
        );
        
        // Check voter hasn't voted (one vote per address in QV)
        assert!(
            !aptos_std::table::contains(&proposal.credits_used, voter_addr),
            2,
        );
        
        // Quadratic weight: sqrt(tokens spent)
        let vote_weight = isqrt(credit_tokens as u128);
        
        if (support) {
            proposal.votes_for = proposal.votes_for + vote_weight;
        } else {
            proposal.votes_against = proposal.votes_against + vote_weight;
        };
        
        proposal.total_voters = proposal.total_voters + 1;
        aptos_std::table::add(
            &mut proposal.credits_used,
            voter_addr,
            credit_tokens,
        );
    }
    
    #[view]
    public fun is_qv_passed(proposal: &QVProposal): bool {
        proposal.votes_for > proposal.votes_against
    }
}
```

---

## Optimistic Governance

```move
// ============================================
// OPTIMISTIC GOVERNANCE
// Any action passes unless vetoed within window
// Suitable for frequent low-stakes decisions
// ============================================

module protocol::optimistic_gov {
    use aptos_framework::timestamp;
    
    const VETO_WINDOW_SECS: u64 = 48 * 3600;   // 48 hours to veto
    const VETO_THRESHOLD_BPS: u64 = 1000;       // 10% of supply to veto
    
    struct OptimisticAction has key {
        id: u64,
        proposer: address,
        description: vector<u8>,
        action_data: vector<u8>,
        created_at: u64,
        
        // Veto tracking
        veto_power: u64,     // Total governance tokens staked to veto
        vetoed: bool,
        executed: bool,
    }
    
    struct VetoStake has key {
        action_id: u64,
        staker: address,
        amount: u64,
    }
    
    public entry fun propose_action(
        proposer: &signer,
        description: vector<u8>,
        action_data: vector<u8>,
    ) {
        // Anyone can propose (or require minimum token balance)
        let now = timestamp::now_microseconds();
        
        move_to(proposer, OptimisticAction {
            id: 0,  // Increment from counter
            proposer: std::signer::address_of(proposer),
            description,
            action_data,
            created_at: now,
            veto_power: 0,
            vetoed: false,
            executed: false,
        });
    }
    
    /// Stake tokens to veto a proposed action
    public entry fun veto(
        vetoer: &signer,
        action_addr: address,
        stake_amount: u64,
    ) {
        let action = borrow_global_mut<OptimisticAction>(action_addr);
        let now = timestamp::now_microseconds();
        
        // Must be within veto window
        assert!(
            now <= action.created_at + VETO_WINDOW_SECS * 1_000_000,
            1,
        );
        assert!(!action.executed, 2);
        
        // Stake tokens for veto
        // coin::transfer<GovToken>(vetoer, @veto_escrow, stake_amount);
        
        action.veto_power = action.veto_power + stake_amount;
        
        // Check if veto threshold met
        let total_supply = 100_000_000u64;  // Get from token info
        let veto_threshold = total_supply * VETO_THRESHOLD_BPS / 10_000;
        
        if (action.veto_power >= veto_threshold) {
            action.vetoed = true;
        };
        
        let _ = std::signer::address_of(vetoer);
    }
    
    /// Execute action after veto window (if not vetoed)
    public entry fun execute_action(
        anyone: &signer,
        action_addr: address,
    ) {
        let action = borrow_global_mut<OptimisticAction>(action_addr);
        let now = timestamp::now_microseconds();
        
        assert!(!action.vetoed, 3);
        assert!(!action.executed, 4);
        assert!(
            now > action.created_at + VETO_WINDOW_SECS * 1_000_000,
            5,
        );
        
        action.executed = true;
        
        // Execute action_data
        // dispatch(action.action_data);
        
        let _ = anyone;
    }
}
```

---

## DAO Treasury Management

```move
// ============================================
// DAO TREASURY
// Multi-asset treasury with governance control
// ============================================

module protocol::treasury {
    use aptos_framework::coin;
    use aptos_framework::timestamp;
    use std::signer;
    
    struct Treasury has key {
        admin: address,            // Governance contract address
        monthly_budget: u64,       // Max spend per month
        spent_this_month: u64,
        month_start: u64,          // Timestamp of month start
        
        // Approved spenders (e.g., contributors)
        approved_spenders: aptos_std::table::Table<address, u64>,  // addr → monthly limit
    }
    
    struct SpendingRequest has key {
        requester: address,
        amount: u64,
        recipient: address,
        description: vector<u8>,
        proposal_id: u64,   // Must be approved via governance
        executed: bool,
    }
    
    public entry fun initialize(
        admin: &signer,
        monthly_budget: u64,
    ) {
        move_to(admin, Treasury {
            admin: signer::address_of(admin),
            monthly_budget,
            spent_this_month: 0,
            month_start: timestamp::now_microseconds(),
            approved_spenders: aptos_std::table::new(),
        });
    }
    
    /// Execute a treasury transfer (must have governance approval via proposal_id)
    public entry fun execute_transfer<CoinType>(
        executor: &signer,
        treasury_addr: address,
        request_addr: address,
    ) {
        let treasury = borrow_global_mut<Treasury>(treasury_addr);
        let request = borrow_global_mut<SpendingRequest>(request_addr);
        
        assert!(!request.executed, 1);
        assert!(signer::address_of(executor) == treasury.admin, 2);
        
        // Reset monthly budget if new month
        let now = timestamp::now_microseconds();
        let secs_per_month = 30u64 * 24 * 3600 * 1_000_000;
        if (now - treasury.month_start > secs_per_month) {
            treasury.spent_this_month = 0;
            treasury.month_start = now;
        };
        
        // Check budget
        assert!(
            treasury.spent_this_month + request.amount <= treasury.monthly_budget,
            3,
        );
        
        request.executed = true;
        treasury.spent_this_month = treasury.spent_this_month + request.amount;
        
        // Transfer funds
        // coin::transfer<CoinType>(treasury_signer, request.recipient, request.amount);
    }
    
    /// Diversify treasury: swap excess native token for stablecoins
    public entry fun rebalance_treasury<NativeToken, StableCoin>(
        admin: &signer,
        treasury_addr: address,
        swap_amount: u64,
        min_stable_out: u64,
    ) {
        let treasury = borrow_global<Treasury>(treasury_addr);
        assert!(signer::address_of(admin) == treasury.admin, 1);
        
        // Only governance can trigger rebalance
        // dex::swap<NativeToken, StableCoin>(swap_amount, min_stable_out);
        let _ = (swap_amount, min_stable_out);
    }
    
    #[view]
    public fun get_treasury_stats(treasury_addr: address): (u64, u64, u64) {
        let treasury = borrow_global<Treasury>(treasury_addr);
        let budget_remaining = treasury.monthly_budget - treasury.spent_this_month;
        (treasury.monthly_budget, treasury.spent_this_month, budget_remaining)
    }
}
```

---

## Governance Security

```
GOVERNANCE ATTACK VECTORS & MITIGATIONS

1. GOVERNANCE TAKEOVER (Flash Loan Attack)
   Attack: Borrow 51% of tokens, vote, return
   Fix:    Snapshot voting power at proposal creation
           Use veToken (locked, can't borrow)
           Add timelock between proposal + voting

2. VOTE BUYING
   Attack: Pay others to vote your way
   Fix:    Quadratic voting (expensive to buy influence)
           Anonymous voting (ZK proofs)
           Conviction voting (weight increases over time)

3. SPAM PROPOSALS
   Attack: Create thousands of proposals to clog governance
   Fix:    Proposal stake (slashed if spam, returned if passes)
           Minimum token balance to propose

4. SHORT-CIRCUIT ATTACKS
   Attack: Pass malicious proposal before community notices
   Fix:    Mandatory 5-7 day voting period (no fast-track)
           Emergency guardian veto (limited use)
           Timelock guardian (48h delay on execution)

5. LOW PARTICIPATION ATTACKS
   Attack: Pass proposal with 5% of supply (below quorum)
   Fix:    Quorum requirements (4-10% of circulating supply)
           Quorum rising over time if participation is low

6. GRIEFING (PERPETUAL VETO)
   Attack: Veto every proposal using whale position
   Fix:    Veto requires staking tokens (cost to attack)
           Veto only available for limited actions

SECURITY CHECKLIST FOR GOVERNANCE
  □ Snapshot voting power at proposal creation time
  □ Timelock on all executable proposals (min 48h)
  □ Quorum requirement with automatic checks
  □ Proposal stake to prevent spam
  □ Guardian emergency pause (limited scope)
  □ On-chain action encoding (no off-chain trust)
  □ Maximum proposal frequency per address
  □ Veto mechanism for community defense
  □ Post-execution verification (assert expected state)
  □ Governance health monitoring (turnout, diversity)
```

---

## สรุป Advanced Governance

```
GOVERNANCE DESIGN SPACE

Simple ←────────────────────────────────→ Complex
  │                                          │
Multisig                              Full On-Chain
(Centralized)                         Token Governance

Recommended for each stage:

Stage 1 (Launch):
  - 2/3 multisig for emergency
  - 5/9 guardian for parameters
  Rationale: Move fast, still safe

Stage 2 (Growth, >$10M TVL):
  - Governance token with timelock
  - 7-day voting, 48h timelock
  - Multisig only for emergencies
  Rationale: Community legitimacy

Stage 3 (Maturity, >$100M TVL):
  - Full DAO governance
  - veToken for long-term alignment
  - Quadratic voting for fairness
  - Optimistic for routine ops
  Rationale: True decentralization

KEY GOVERNANCE METRICS
  Voter Turnout: target >10% of supply
  Proposal Pass Rate: target 50-80%
  Time to Execution: total of voting + timelock
  Governance Token Concentration: top 10 < 30%
```

---

**ก่อนหน้า**: [Part 92 - NFT Marketplace ←](part-92-nft-marketplace.md)
**ต่อไป**: [Part 94 - Multi-Chain Strategy →](part-94-multichain.md)
