# Part 73: Token Launch & Vesting Systems

## สารบัญ
- [Token Launch Strategies](#token-launch-strategies)
- [Fair Launch Implementation](#fair-launch-implementation)
- [Initial DEX Offering (IDO)](#initial-dex-offering-ido)
- [Vesting & Lock Schedules](#vesting--lock-schedules)
- [Token Distribution System](#token-distribution-system)
- [Airdrop with Merkle Proof](#airdrop-with-merkle-proof)

---

## Token Launch Strategies

```
Token Launch Methods:

1. FAIR LAUNCH
   - No pre-mine, no VC allocation
   - Tokens available to everyone simultaneously
   - Example: Early Bitcoin, Compound's COMP retroactive
   - Pros: Community trust, no insider advantage
   - Cons: Hard to raise development capital
   
2. IDO (Initial DEX Offering)
   - Token sold directly via DEX at fixed/bonding price
   - Anyone can buy during public sale period
   - Example: Most DeFi tokens (early 2021)
   - Pros: Decentralized, accessible, immediate trading
   - Cons: Front-running, bot sniping, gas wars
   
3. LBP (Liquidity Bootstrapping Pool)
   - Copperlaunch, Fjord Foundry
   - Price starts high, decreases over time
   - Prevents front-running (buying early = expensive)
   - Pros: Fair price discovery, anti-bot
   - Cons: Complex for users, requires understanding
   
4. RETROACTIVE AIRDROP
   - Reward existing users of protocol
   - Based on on-chain activity snapshot
   - Example: Uniswap (400 UNI), ENS (100+ ENS)
   - Pros: Rewards loyal users, generates goodwill
   - Cons: Airdrop farmers, Sybil attacks
   
5. DUTCH AUCTION
   - Price starts high, decreases until fully subscribed
   - All winners pay same final clearing price
   - Example: Google IPO, Gnosis LBP variation
   - Pros: True price discovery, fair allocation
   - Cons: Complexity, uncertain for participants
   
Aptos/Sui Launch Considerations:
  - Lower gas than Ethereum → less gas war pressure
  - Faster finality → less front-running window
  - Aptos: Use coin standard or FA v2
  - Sui: Use coin with fixed supply object
```

---

## Fair Launch Implementation

```move
module launch::fair_launch {
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::timestamp;
    
    // ============================================
    // FAIR LAUNCH: WHITELIST → PUBLIC
    //
    // Phase 1: Whitelist (24h)
    //   - Registered users can buy at base price
    //   - Per-wallet cap prevents whales
    //   
    // Phase 2: Public (48h)
    //   - Open to all
    //   - Same price (simple launch)
    //   
    // Phase 3: Close
    //   - Remaining tokens distributed to liquidity
    // ============================================
    
    const PHASE_WHITELIST: u8 = 0;
    const PHASE_PUBLIC: u8 = 1;
    const PHASE_CLOSED: u8 = 2;
    
    struct LaunchConfig has key {
        // Token supply for launch
        tokens_for_sale: u64,
        tokens_sold: u64,
        
        // Pricing (tokens per USDC, scaled)
        price_per_token: u64,  // e.g., 100_000 = $0.10 (6 decimal USDC)
        
        // Phase timing
        whitelist_start: u64,
        whitelist_end: u64,
        public_start: u64,
        public_end: u64,
        
        // Caps
        whitelist_cap_per_wallet: u64,  // Max per whitelisted wallet
        public_cap_per_wallet: u64,     // Max per wallet in public phase
        
        // Whitelist (address → committed)
        whitelist: aptos_std::smart_table::SmartTable<address, bool>,
        
        // Purchases tracking
        purchased: aptos_std::smart_table::SmartTable<address, u64>,
        
        // Treasury
        raise_recipient: address,
    }
    
    // Admin: Add to whitelist
    public entry fun add_to_whitelist(
        admin: &signer,
        config_addr: address,
        users: vector<address>,
    ) acquires LaunchConfig {
        let config = borrow_global_mut<LaunchConfig>(config_addr);
        let len = std::vector::length(&users);
        let mut i = 0u64;
        while (i < len) {
            let user = *std::vector::borrow(&users, i);
            if (!aptos_std::smart_table::contains(&config.whitelist, user)) {
                aptos_std::smart_table::add(&mut config.whitelist, user, true);
            };
            i = i + 1;
        };
    }
    
    // User: Buy tokens
    public entry fun buy<TOKEN, PAYMENT>(
        buyer: &signer,
        config_addr: address,
        usdc_amount: u64,  // Amount of USDC to spend
    ) acquires LaunchConfig {
        let config = borrow_global_mut<LaunchConfig>(config_addr);
        let buyer_addr = std::signer::address_of(buyer);
        let now = timestamp::now_microseconds();
        
        // Determine current phase
        let phase = get_phase(config, now);
        assert!(phase != PHASE_CLOSED, 1);
        
        // Whitelist phase: check eligibility
        if (phase == PHASE_WHITELIST) {
            assert!(aptos_std::smart_table::contains(&config.whitelist, buyer_addr), 2);
        };
        
        // Calculate token amount
        let token_amount = usdc_amount * 1_000_000 / config.price_per_token;
        
        // Check per-wallet cap
        let already_purchased = get_purchased(config, buyer_addr);
        let cap = if (phase == PHASE_WHITELIST) {
            config.whitelist_cap_per_wallet
        } else {
            config.public_cap_per_wallet
        };
        
        assert!(already_purchased + token_amount <= cap, 3);
        
        // Check remaining supply
        assert!(config.tokens_sold + token_amount <= config.tokens_for_sale, 4);
        
        // Take payment
        let payment = coin::withdraw<PAYMENT>(buyer, usdc_amount);
        coin::deposit(config.raise_recipient, payment);
        
        // Send tokens immediately (or record for claim later)
        config.tokens_sold = config.tokens_sold + token_amount;
        
        // Record purchase
        if (aptos_std::smart_table::contains(&config.purchased, buyer_addr)) {
            *aptos_std::smart_table::borrow_mut(&mut config.purchased, buyer_addr) 
                = already_purchased + token_amount;
        } else {
            aptos_std::smart_table::add(&mut config.purchased, buyer_addr, token_amount);
        };
        
        // Emit event
        aptos_framework::event::emit(TokenPurchased {
            buyer: buyer_addr,
            payment_amount: usdc_amount,
            token_amount,
            phase,
        });
    }
    
    fun get_phase(config: &LaunchConfig, now: u64): u8 {
        if (now < config.whitelist_start) PHASE_CLOSED
        else if (now <= config.whitelist_end) PHASE_WHITELIST
        else if (now >= config.public_start && now <= config.public_end) PHASE_PUBLIC
        else PHASE_CLOSED
    }
    
    fun get_purchased(config: &LaunchConfig, user: address): u64 {
        if (aptos_std::smart_table::contains(&config.purchased, user)) {
            *aptos_std::smart_table::borrow(&config.purchased, user)
        } else {
            0
        }
    }
    
    #[event]
    struct TokenPurchased has drop, store {
        buyer: address,
        payment_amount: u64,
        token_amount: u64,
        phase: u8,
    }
}
```

---

## Vesting & Lock Schedules

```move
module launch::vesting {
    use aptos_framework::timestamp;
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // COMPREHENSIVE VESTING SYSTEM
    //
    // Supports:
    // - Linear vesting (equal daily/monthly)
    // - Cliff vesting (nothing until date, then linear)
    // - Step vesting (unlock at specific intervals)
    // - Immediate (no lock)
    //
    // Common schedules:
    // Team:      6-month cliff, 3-year linear total
    // Investors: 3-month cliff, 2-year linear
    // Advisors:  3-month cliff, 1-year linear
    // Community: No cliff, 4-year linear
    // Treasury:  No cliff, 5-year linear (governance controlled)
    // ============================================
    
    const VESTING_TYPE_LINEAR: u8 = 0;
    const VESTING_TYPE_CLIFF_LINEAR: u8 = 1;
    const VESTING_TYPE_STEP: u8 = 2;
    
    struct VestingSchedule has store {
        // Beneficiary
        beneficiary: address,
        
        // Total allocation
        total_amount: u64,
        claimed_amount: u64,
        
        // Timing (all in microseconds)
        start_time: u64,
        cliff_time: u64,    // Before cliff: nothing vests
        end_time: u64,      // Full vest time
        
        // Type-specific
        vesting_type: u8,
        step_period: u64,   // For step vesting: unlock every N seconds
        
        // Revocable by admin (for employee grants)
        is_revocable: bool,
        is_revoked: bool,
        revoked_at: u64,
    }
    
    struct VestingVault has key {
        schedules: SmartTable<u64, VestingSchedule>,
        next_id: u64,
        total_allocated: u64,
        admin: address,
    }
    
    // Create a vesting schedule
    public fun create_schedule(
        admin: &signer,
        vault_addr: address,
        beneficiary: address,
        amount: u64,
        vesting_type: u8,
        cliff_months: u64,
        total_months: u64,
        is_revocable: bool,
    ): u64 acquires VestingVault {
        let vault = borrow_global_mut<VestingVault>(vault_addr);
        assert!(std::signer::address_of(admin) == vault.admin, 1);
        
        let now = timestamp::now_microseconds();
        let month_us = 30 * 24 * 3_600 * 1_000_000u64;
        
        let id = vault.next_id;
        vault.next_id = id + 1;
        
        let schedule = VestingSchedule {
            beneficiary,
            total_amount: amount,
            claimed_amount: 0,
            start_time: now,
            cliff_time: now + cliff_months * month_us,
            end_time: now + total_months * month_us,
            vesting_type,
            step_period: month_us, // Monthly steps
            is_revocable,
            is_revoked: false,
            revoked_at: 0,
        };
        
        smart_table::add(&mut vault.schedules, id, schedule);
        vault.total_allocated = vault.total_allocated + amount;
        
        id
    }
    
    // Calculate vested amount at current time
    public fun vested_amount(schedule: &VestingSchedule): u64 {
        let now = timestamp::now_microseconds();
        
        // Revoked: only vested up to revocation time
        let effective_now = if (schedule.is_revoked) {
            std::u64::min(now, schedule.revoked_at)
        } else {
            now
        };
        
        // Before cliff: nothing
        if (effective_now < schedule.cliff_time) return 0;
        
        // After end: fully vested
        if (effective_now >= schedule.end_time) return schedule.total_amount;
        
        // During vesting
        match_vesting_type(schedule, effective_now)
    }
    
    fun match_vesting_type(schedule: &VestingSchedule, now: u64): u64 {
        if (schedule.vesting_type == VESTING_TYPE_LINEAR || 
            schedule.vesting_type == VESTING_TYPE_CLIFF_LINEAR) {
            // Linear from cliff to end
            let vesting_duration = schedule.end_time - schedule.cliff_time;
            let elapsed = now - schedule.cliff_time;
            (schedule.total_amount as u128) * (elapsed as u128) / (vesting_duration as u128) as u64
        } else {
            // Step vesting: unlock every step_period
            let periods_elapsed = (now - schedule.cliff_time) / schedule.step_period;
            let total_periods = (schedule.end_time - schedule.cliff_time) / schedule.step_period;
            if (total_periods == 0) return 0;
            
            let min_periods = if (periods_elapsed < total_periods) periods_elapsed else total_periods;
            schedule.total_amount * min_periods / total_periods
        }
    }
    
    // Claim vested tokens
    public fun claim<TOKEN>(
        beneficiary: &signer,
        vault_addr: address,
        schedule_id: u64,
    ) acquires VestingVault {
        let vault = borrow_global_mut<VestingVault>(vault_addr);
        let beneficiary_addr = std::signer::address_of(beneficiary);
        
        let schedule = smart_table::borrow_mut(&mut vault.schedules, schedule_id);
        assert!(schedule.beneficiary == beneficiary_addr, 1);
        
        let total_vested = vested_amount(schedule);
        let claimable = total_vested - schedule.claimed_amount;
        
        assert!(claimable > 0, 2);
        
        schedule.claimed_amount = schedule.claimed_amount + claimable;
        
        // Transfer tokens to beneficiary
        // coin::deposit(beneficiary_addr, claimable);
        
        aptos_framework::event::emit(VestingClaimed {
            beneficiary: beneficiary_addr,
            schedule_id,
            amount: claimable,
        });
    }
    
    // Admin: Revoke vesting (for employee departures)
    public fun revoke(
        admin: &signer,
        vault_addr: address,
        schedule_id: u64,
    ) acquires VestingVault {
        let vault = borrow_global_mut<VestingVault>(vault_addr);
        assert!(std::signer::address_of(admin) == vault.admin, 1);
        
        let schedule = smart_table::borrow_mut(&mut vault.schedules, schedule_id);
        assert!(schedule.is_revocable, 2);
        assert!(!schedule.is_revoked, 3);
        
        schedule.is_revoked = true;
        schedule.revoked_at = timestamp::now_microseconds();
        
        // Calculate unvested amount to return to treasury
        let total_vested = vested_amount(schedule);
        let unvested = schedule.total_amount - total_vested;
        
        // Return unvested tokens to admin/treasury
        // (Simplified: in production transfer unvested back)
    }
    
    #[event]
    struct VestingClaimed has drop, store {
        beneficiary: address,
        schedule_id: u64,
        amount: u64,
    }
}
```

---

## Airdrop with Merkle Proof

```move
module launch::airdrop {
    use std::hash;
    
    // ============================================
    // MERKLE AIRDROP
    // Efficient: Only store root hash on-chain
    // Users claim with Merkle proof (off-chain computed)
    //
    // Off-chain: Build Merkle tree from (address, amount) pairs
    // On-chain: Verify proof and distribute
    //
    // Gas: O(log n) per claim vs O(n) for all on-chain
    // ============================================
    
    struct AirdropConfig has key {
        merkle_root: vector<u8>,    // Root of Merkle tree
        total_tokens: u64,
        claimed_tokens: u64,
        claim_deadline: u64,        // After deadline: unclaimed tokens go to treasury
        
        // Claimed tracking (prevent double claims)
        claimed: aptos_std::simple_map::SimpleMap<address, bool>,
    }
    
    // Claim airdrop with Merkle proof
    public entry fun claim<TOKEN>(
        claimer: &signer,
        config_addr: address,
        amount: u64,
        proof: vector<vector<u8>>,  // Merkle proof nodes
    ) acquires AirdropConfig {
        let config = borrow_global_mut<AirdropConfig>(config_addr);
        let claimer_addr = std::signer::address_of(claimer);
        
        // Check deadline
        assert!(aptos_framework::timestamp::now_microseconds() <= config.claim_deadline, 1);
        
        // Check not already claimed
        assert!(
            !aptos_std::simple_map::contains_key(&config.claimed, &claimer_addr),
            2  // Already claimed
        );
        
        // Build leaf: hash(address || amount)
        let leaf = compute_leaf(claimer_addr, amount);
        
        // Verify Merkle proof
        assert!(verify_proof(&proof, &config.merkle_root, &leaf), 3);
        
        // Mark as claimed
        aptos_std::simple_map::add(&mut config.claimed, claimer_addr, true);
        config.claimed_tokens = config.claimed_tokens + amount;
        
        // Distribute tokens
        // coin::deposit(claimer_addr, amount);
        
        aptos_framework::event::emit(AirdropClaimed {
            claimer: claimer_addr,
            amount,
        });
    }
    
    // Compute leaf node hash: sha3(address || amount)
    public fun compute_leaf(addr: address, amount: u64): vector<u8> {
        let mut preimage = std::bcs::to_bytes(&addr);
        std::vector::append(&mut preimage, std::bcs::to_bytes(&amount));
        hash::sha3_256(preimage)
    }
    
    // Verify Merkle proof
    public fun verify_proof(
        proof: &vector<vector<u8>>,
        root: &vector<u8>,
        leaf: &vector<u8>,
    ): bool {
        let mut current = *leaf;
        let len = std::vector::length(proof);
        let mut i = 0u64;
        
        while (i < len) {
            let sibling = std::vector::borrow(proof, i);
            
            // Combine: hash(smaller ++ larger) for canonical ordering
            current = hash_pair(&current, sibling);
            
            i = i + 1;
        };
        
        current == *root
    }
    
    fun hash_pair(a: &vector<u8>, b: &vector<u8>): vector<u8> {
        // Canonical ordering: smaller hash first
        let (first, second) = if (compare_bytes(a, b)) {
            (a, b)
        } else {
            (b, a)
        };
        
        let mut combined = *first;
        std::vector::append(&mut combined, *second);
        hash::sha3_256(combined)
    }
    
    // Returns true if a < b (lexicographic)
    fun compare_bytes(a: &vector<u8>, b: &vector<u8>): bool {
        let len = std::u64::min(std::vector::length(a), std::vector::length(b));
        let mut i = 0u64;
        while (i < len) {
            let va = *std::vector::borrow(a, i);
            let vb = *std::vector::borrow(b, i);
            if (va < vb) return true;
            if (va > vb) return false;
            i = i + 1;
        };
        std::vector::length(a) < std::vector::length(b)
    }
    
    #[event]
    struct AirdropClaimed has drop, store {
        claimer: address,
        amount: u64,
    }
}
```

---

## สรุป Token Launch & Vesting

```
Token Launch Best Practices:

LAUNCH STRATEGY SELECTION
  Small team/community project:
    → Fair launch or retroactive airdrop
    
  VC-backed startup:
    → IDO with lockup for VCs
    → Different round prices with documentation
    
  Established protocol:
    → Retroactive airdrop to users
    → Ongoing emissions for liquidity
    
VESTING SCHEDULES (Typical)
  Founders:   12-month cliff, 36-month vest (4 years total)
  Team:       6-month cliff, 30-month vest (3 years total)
  Seed VCs:   6-month cliff, 18-month vest (2 years total)
  Series A:   3-month cliff, 15-month vest (1.5 years total)
  Advisors:   3-month cliff, 12-month vest (1 year total)
  Community:  No cliff, 48-month vest (4 years linear)
  Treasury:   No cliff, 60-month (5 years, governance controlled)
  
ANTI-SYBIL FOR AIRDROPS
  - Minimum transaction count (e.g., 10+ txs)
  - Minimum TVL (e.g., $100 deposited)
  - Protocol-specific activity
  - Gitcoin passport or similar proof-of-humanity
  - Cluster analysis (related addresses share allocation)
  
MERKLE AIRDROP GAS SAVINGS
  1000 recipients:
    On-chain list: 1000 writes × $0.01 = $10
    Merkle proof:  1000 claims × log2(1000) × $0.001 ≈ $10
                   BUT: Unclaimed tokens never cost gas!
  
  100,000 recipients:
    On-chain list: $1,000 upfront
    Merkle proof:  ~$0 upfront, claimers pay per claim
    Unclaimed (estimate 50%): Save 50% × $1,000 = $500

TOKEN ALLOCATION (Typical DeFi)
  Team + Founders:  15-20%
  Investors:        10-15%
  Community/Ecosystem: 40-50%
  Treasury:         10-20%
  Liquidity:        5-10%
  
  Key: Team + Investors must be LESS than Community
  Bad optics: More than 30% to insiders
```

---

**ก่อนหน้า**: [Part 72 - Lending Protocols ←](part-72-lending-protocols.md)
**ต่อไป**: [Part 74 - DAO & Community Governance →](part-74-dao-governance.md)
