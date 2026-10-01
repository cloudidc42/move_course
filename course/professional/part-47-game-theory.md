# Part 47: Game Theory in DeFi

## สารบัญ
- [Game Theory Fundamentals](#game-theory-fundamentals)
- [Mechanism Design in DeFi](#mechanism-design-in-defi)
- [MEV & Sandwich Attack Defense](#mev--sandwich-attack-defense)
- [Auction Mechanisms](#auction-mechanisms)
- [Incentive-Compatible Protocols](#incentive-compatible-protocols)
- [ตัวอย่าง: Dutch Auction & Sealed Bid](#ตัวอย่าง-dutch-auction--sealed-bid)

---

## Game Theory Fundamentals

```
Game Theory: Study of strategic decision-making between rational agents

Key Concepts:
  Nash Equilibrium: No player can improve by changing strategy alone
  Dominant Strategy: Best regardless of others' choices
  Pareto Optimal: No one can improve without making someone worse off
  Mechanism Design: Design rules so desired outcome = Nash Equilibrium
  
In DeFi context:
  Players: Users, LPs, validators, bots, attackers
  Strategies: When/how to trade, stake, liquidate
  Payoffs: Token profits, yields, fees
  
Examples:
  AMM: Arbitrageurs + LPs → Nash Eq at fair price
  Lending: Borrowers + Liquidators → Liquidate at health < 1.0
  Governance: Whales + Small holders → Plutocracy risk
  Sequencer: Multiple validators competing → MEV capture
  
Incentive Design Principles:
  1. Align incentives with desired behavior
  2. Make honest behavior dominant strategy
  3. Penalize defection more than its reward
  4. Use commitment mechanisms (locks, bonds)
  5. Transparency: everyone can verify rules
```

---

## Mechanism Design in DeFi

```move
module game_theory::mechanism_design {
    
    // ============================================
    // Example 1: Schelling Point Oracle
    // Truth-telling is Nash Equilibrium
    // ============================================
    
    // Based on Schelling's focal point theory:
    // People coordinate on obvious/focal answers
    // When penalized for being outlier → truth-telling equilibrium
    
    struct SchellingOracle has key {
        round_id: u64,
        commit_deadline: u64,
        reveal_deadline: u64,
        
        // Committed values (before reveal)
        commits: aptos_std::smart_table::SmartTable<address, vector<u8>>,
        
        // Revealed values
        reveals: aptos_std::smart_table::SmartTable<address, u64>,
        
        // Stakes locked for this round
        stakes: aptos_std::smart_table::SmartTable<address, u64>,
        
        // Final result
        result: u64,
        resolved: bool,
        
        // Reward pool
        total_stake: u64,
    }
    
    const REWARD_BPS: u64 = 2000;     // 20% reward for honest reporters
    const SLASH_BPS: u64 = 5000;      // 50% slash for outliers
    const TOLERANCE_BPS: u64 = 500;   // ±5% from median is "correct"
    
    // Commit phase: Hash(price, salt) to prevent copying
    public entry fun commit_price(
        reporter: &signer,
        oracle_addr: address,
        commitment: vector<u8>,  // keccak256(price || salt)
        stake_amount: u64,
    ) acquires SchellingOracle {
        use aptos_std::smart_table;
        let oracle = borrow_global_mut<SchellingOracle>(oracle_addr);
        let now = aptos_framework::timestamp::now_seconds();
        assert!(now <= oracle.commit_deadline, 1);
        
        let addr = std::signer::address_of(reporter);
        
        // Lock stake
        // coin::withdraw + store in oracle
        smart_table::add(&mut oracle.commits, addr, commitment);
        smart_table::add(&mut oracle.stakes, addr, stake_amount);
        oracle.total_stake = oracle.total_stake + stake_amount;
    }
    
    // Reveal phase: Show actual price + salt
    public entry fun reveal_price(
        reporter: &signer,
        oracle_addr: address,
        price: u64,
        salt: vector<u8>,
    ) acquires SchellingOracle {
        use aptos_std::smart_table;
        let oracle = borrow_global_mut<SchellingOracle>(oracle_addr);
        let now = aptos_framework::timestamp::now_seconds();
        assert!(now > oracle.commit_deadline, 2);
        assert!(now <= oracle.reveal_deadline, 3);
        
        let addr = std::signer::address_of(reporter);
        
        // Verify commitment
        let stored_commit = smart_table::borrow(&oracle.commits, addr);
        let mut price_bytes = salt;
        // Encode price into bytes and append
        let mut i = 0u8;
        while (i < 8) {
            std::vector::push_back(&mut price_bytes, ((price >> (i * 8)) & 0xFF) as u8);
            i = i + 1;
        };
        let expected = aptos_std::aptos_hash::keccak256(price_bytes);
        assert!(expected == *stored_commit, 4);
        
        smart_table::add(&mut oracle.reveals, addr, price);
    }
    
    // Resolve: Find median, reward those near it, slash outliers
    public entry fun resolve(
        oracle_addr: address,
    ) acquires SchellingOracle {
        let oracle = borrow_global_mut<SchellingOracle>(oracle_addr);
        let now = aptos_framework::timestamp::now_seconds();
        assert!(now > oracle.reveal_deadline, 5);
        assert!(!oracle.resolved, 6);
        
        // Compute median of all reveals
        // (In production: sort reveals, take middle)
        let median = compute_median(&oracle.reveals);
        oracle.result = median;
        oracle.resolved = true;
        
        // Reward reporters within tolerance, slash outliers
        // reward = oracle.total_stake * SLASH_BPS / 10000 / num_honest
        // distributed to honest reporters proportional to their stake
    }
    
    fun compute_median(
        reveals: &aptos_std::smart_table::SmartTable<address, u64>,
    ): u64 {
        // Simplified: return 0 (real impl sorts and takes middle)
        0
    }
    
    // ============================================
    // Example 2: Commit-Reveal for Fair Launch
    // ============================================
    
    struct FairLaunch has key {
        commit_phase_end: u64,
        reveal_phase_end: u64,
        max_per_wallet: u64,
        price_per_token: u64,
        
        commits: aptos_std::smart_table::SmartTable<address, vector<u8>>,
        reveals: aptos_std::smart_table::SmartTable<address, u64>,  // amount
        
        total_committed: u64,
        total_allocation: u64,  // Total tokens available
    }
    
    // Phase 1: Commit (prevents front-running)
    public entry fun commit_allocation(
        user: &signer,
        launch_addr: address,
        commitment: vector<u8>,  // hash(amount, salt)
    ) acquires FairLaunch {
        let launch = borrow_global_mut<FairLaunch>(launch_addr);
        let now = aptos_framework::timestamp::now_seconds();
        assert!(now <= launch.commit_phase_end, 1);
        
        let addr = std::signer::address_of(user);
        assert!(
            !aptos_std::smart_table::contains(&launch.commits, addr),
            7  // Already committed
        );
        aptos_std::smart_table::add(&mut launch.commits, addr, commitment);
    }
    
    // Phase 2: Reveal actual amount
    public entry fun reveal_allocation(
        user: &signer,
        launch_addr: address,
        amount: u64,
        salt: vector<u8>,
    ) acquires FairLaunch {
        let launch = borrow_global_mut<FairLaunch>(launch_addr);
        let now = aptos_framework::timestamp::now_seconds();
        assert!(now > launch.commit_phase_end, 2);
        assert!(now <= launch.reveal_phase_end, 3);
        assert!(amount <= launch.max_per_wallet, 8);
        
        let addr = std::signer::address_of(user);
        
        // Verify commitment
        let commit = aptos_std::smart_table::borrow(&launch.commits, addr);
        let mut input = salt;
        let mut i = 0u8;
        while (i < 8) {
            std::vector::push_back(&mut input, ((amount >> (i * 8)) & 0xFF) as u8);
            i = i + 1;
        };
        let expected = aptos_std::aptos_hash::keccak256(input);
        assert!(expected == *commit, 9);
        
        aptos_std::smart_table::add(&mut launch.reveals, addr, amount);
        launch.total_committed = launch.total_committed + amount;
    }
    
    // Phase 3: Claim proportional allocation if oversubscribed
    public entry fun claim(
        user: &signer,
        launch_addr: address,
    ) acquires FairLaunch {
        let launch = borrow_global<FairLaunch>(launch_addr);
        let now = aptos_framework::timestamp::now_seconds();
        assert!(now > launch.reveal_phase_end, 10);
        
        let addr = std::signer::address_of(user);
        let committed = *aptos_std::smart_table::borrow(&launch.reveals, addr);
        
        // If oversubscribed: pro-rate allocation
        let allocation = if (launch.total_committed <= launch.total_allocation) {
            committed
        } else {
            committed * launch.total_allocation / launch.total_committed
        };
        
        // Transfer tokens and refund excess payment
    }
}
```

---

## MEV & Sandwich Attack Defense

```move
module game_theory::mev_defense {
    use aptos_framework::timestamp;
    
    // ============================================
    // MEV (Miner/Maximal Extractable Value) in Move
    // ============================================
    
    // Types of MEV on Aptos:
    // 1. Sandwich attacks on DEX swaps
    // 2. Liquidation racing (multiple liquidators)
    // 3. Arbitrage (price discrepancy across pools)
    // 4. NFT mint sniping
    
    // Sandwich Attack:
    // Attacker sees victim swap in mempool
    // 1. Attacker buys BEFORE victim (front-run)
    // 2. Victim executes at worse price
    // 3. Attacker sells AFTER (back-run)
    // Profit = price impact from victim's trade
    
    struct ProtectedPool has key {
        reserve_x: u64,
        reserve_y: u64,
        fee_bps: u64,
        
        // Anti-MEV: commitment scheme
        committed_txs: aptos_std::smart_table::SmartTable<vector<u8>, CommittedTx>,
        
        // Price impact limit
        max_price_impact_bps: u64,
        
        // Rate limiting per address per block
        block_swaps: aptos_std::smart_table::SmartTable<address, u64>,
        current_block: u64,
    }
    
    struct CommittedTx has copy, drop, store {
        commit_block: u64,
        committed_at: u64,
    }
    
    const MIN_BLOCKS_BETWEEN_COMMIT_EXECUTE: u64 = 2;
    const MAX_SWAPS_PER_BLOCK: u64 = 3;
    
    // DEFENSE 1: Price impact limit
    // Prevents single tx from moving price too much
    public entry fun swap_with_impact_limit(
        user: &signer,
        pool_addr: address,
        amount_in: u64,
        min_out: u64,
        deadline: u64,
    ) acquires ProtectedPool {
        let pool = borrow_global_mut<ProtectedPool>(pool_addr);
        
        // Deadline check
        assert!(timestamp::now_seconds() <= deadline, 1);
        
        // Calculate price impact
        let amount_out = compute_out(pool.reserve_x, pool.reserve_y, amount_in, pool.fee_bps);
        
        // Price before: reserve_y / reserve_x
        // Price after: (reserve_y - out) / (reserve_x + in)
        // Impact = |price_after - price_before| / price_before
        let price_before = pool.reserve_y * 10_000 / pool.reserve_x;
        let price_after = (pool.reserve_y - amount_out) * 10_000 
                         / (pool.reserve_x + amount_in);
        
        let impact = if (price_before > price_after) {
            (price_before - price_after) * 10_000 / price_before
        } else {
            (price_after - price_before) * 10_000 / price_before
        };
        
        // Reject if too much impact (prevents large sandwichable txs)
        assert!(impact <= pool.max_price_impact_bps, 2);
        
        // Slippage check
        assert!(amount_out >= min_out, 3);
        
        // Execute swap
        pool.reserve_x = pool.reserve_x + amount_in;
        pool.reserve_y = pool.reserve_y - amount_out;
    }
    
    // DEFENSE 2: Rate limiting
    // Prevents same address from making multiple swaps per block
    public entry fun rate_limited_swap(
        user: &signer,
        pool_addr: address,
        amount_in: u64,
        min_out: u64,
    ) acquires ProtectedPool {
        let pool = borrow_global_mut<ProtectedPool>(pool_addr);
        let addr = std::signer::address_of(user);
        let current_block = timestamp::now_microseconds() / 500_000;  // ~0.5s blocks
        
        // Reset counter if new block
        if (pool.current_block != current_block) {
            pool.current_block = current_block;
            // Reset all counters (simplified - real impl clears old entries)
        };
        
        let swap_count = if (aptos_std::smart_table::contains(&pool.block_swaps, addr)) {
            *aptos_std::smart_table::borrow(&pool.block_swaps, addr)
        } else { 0 };
        
        assert!(swap_count < MAX_SWAPS_PER_BLOCK, 4);
        
        if (swap_count == 0) {
            aptos_std::smart_table::add(&mut pool.block_swaps, addr, 1);
        } else {
            *aptos_std::smart_table::borrow_mut(&mut pool.block_swaps, addr) = swap_count + 1;
        };
        
        // Execute swap
        let amount_out = compute_out(pool.reserve_x, pool.reserve_y, amount_in, pool.fee_bps);
        assert!(amount_out >= min_out, 3);
        pool.reserve_x = pool.reserve_x + amount_in;
        pool.reserve_y = pool.reserve_y - amount_out;
    }
    
    // DEFENSE 3: Commit-reveal for large swaps
    // Large swaps must commit hash first, execute after N blocks
    public entry fun commit_large_swap(
        user: &signer,
        pool_addr: address,
        swap_hash: vector<u8>,  // hash(amount_in, min_out, salt, user)
    ) acquires ProtectedPool {
        let pool = borrow_global_mut<ProtectedPool>(pool_addr);
        let current_block = timestamp::now_microseconds() / 500_000;
        
        aptos_std::smart_table::add(&mut pool.committed_txs, swap_hash, CommittedTx {
            commit_block: current_block,
            committed_at: timestamp::now_seconds(),
        });
    }
    
    public entry fun execute_committed_swap(
        user: &signer,
        pool_addr: address,
        amount_in: u64,
        min_out: u64,
        salt: vector<u8>,
    ) acquires ProtectedPool {
        let pool = borrow_global_mut<ProtectedPool>(pool_addr);
        let addr = std::signer::address_of(user);
        let current_block = timestamp::now_microseconds() / 500_000;
        
        // Recompute hash to verify commitment
        let mut input = salt;
        // Encode amount_in, min_out, addr...
        let swap_hash = aptos_std::aptos_hash::keccak256(input);
        
        let committed = aptos_std::smart_table::borrow(&pool.committed_txs, swap_hash);
        
        // Must wait N blocks after commit
        assert!(
            current_block >= committed.commit_block + MIN_BLOCKS_BETWEEN_COMMIT_EXECUTE,
            5
        );
        
        // Remove commitment
        aptos_std::smart_table::remove(&mut pool.committed_txs, swap_hash);
        
        // Execute swap
        let amount_out = compute_out(pool.reserve_x, pool.reserve_y, amount_in, pool.fee_bps);
        assert!(amount_out >= min_out, 3);
        pool.reserve_x = pool.reserve_x + amount_in;
        pool.reserve_y = pool.reserve_y - amount_out;
    }
    
    fun compute_out(reserve_x: u64, reserve_y: u64, amount_in: u64, fee_bps: u64): u64 {
        let fee_factor = 10_000 - fee_bps;
        let in_with_fee = (amount_in as u128) * (fee_factor as u128);
        let numerator = (reserve_y as u128) * in_with_fee;
        let denominator = (reserve_x as u128) * 10_000u128 + in_with_fee;
        (numerator / denominator) as u64
    }
}
```

---

## Auction Mechanisms

```move
module game_theory::auctions {
    use aptos_framework::timestamp;
    use aptos_framework::coin;
    use aptos_framework::event;
    
    // ============================================
    // English Auction (ascending price)
    // ============================================
    
    struct EnglishAuction has key {
        item_id: u64,
        seller: address,
        min_bid: u64,
        min_increment_bps: u64,  // Minimum bid increment %
        end_time: u64,
        extension_time: u64,     // Extend by N seconds if bid near end
        
        highest_bid: u64,
        highest_bidder: address,
        
        // Pending refunds for outbid participants
        pending_refunds: aptos_std::smart_table::SmartTable<address, u64>,
    }
    
    #[event]
    struct NewBid has drop, store {
        auction_id: u64,
        bidder: address,
        amount: u64,
    }
    
    public entry fun place_bid(
        bidder: &signer,
        auction_addr: address,
        bid_amount: u64,
    ) acquires EnglishAuction {
        let auction = borrow_global_mut<EnglishAuction>(auction_addr);
        let now = timestamp::now_seconds();
        assert!(now < auction.end_time, 1);
        
        // Minimum bid check
        let min_required = if (auction.highest_bid == 0) {
            auction.min_bid
        } else {
            auction.highest_bid 
            + auction.highest_bid * auction.min_increment_bps / 10_000
        };
        assert!(bid_amount >= min_required, 2);
        
        let bidder_addr = std::signer::address_of(bidder);
        
        // Refund previous highest bidder
        if (auction.highest_bid > 0 && auction.highest_bidder != @0x0) {
            let prev = auction.highest_bidder;
            let prev_bid = auction.highest_bid;
            if (aptos_std::smart_table::contains(&auction.pending_refunds, prev)) {
                *aptos_std::smart_table::borrow_mut(&mut auction.pending_refunds, prev) 
                    += prev_bid;
            } else {
                aptos_std::smart_table::add(&mut auction.pending_refunds, prev, prev_bid);
            };
        };
        
        // Lock bid
        let bid_coins = coin::withdraw<aptos_coin::AptosCoin>(bidder, bid_amount);
        coin::deposit<aptos_coin::AptosCoin>(auction_addr, bid_coins);
        
        auction.highest_bid = bid_amount;
        auction.highest_bidder = bidder_addr;
        
        // Extend auction if bid near end (anti-sniping)
        if (auction.end_time - now <= 300) {  // Last 5 minutes
            auction.end_time = auction.end_time + auction.extension_time;
        };
        
        event::emit(NewBid {
            auction_id: auction.item_id,
            bidder: bidder_addr,
            amount: bid_amount,
        });
    }
    
    // ============================================
    // Dutch Auction (descending price)
    // Good for token sales: price discovery
    // ============================================
    
    struct DutchAuction has key {
        start_price: u64,
        end_price: u64,
        start_time: u64,
        end_time: u64,
        
        total_supply: u64,
        sold: u64,
        
        participants: aptos_std::smart_table::SmartTable<address, DutchBid>,
        
        // Clearing price: final price when sold out
        clearing_price: u64,
        concluded: bool,
    }
    
    struct DutchBid has copy, drop, store {
        amount: u64,    // Tokens purchased
        price_paid: u64, // Price at time of purchase
        refund_due: u64,  // Refund if clearing price < paid
    }
    
    // Current price = linear decline from start to end
    public fun current_price(auction: &DutchAuction): u64 {
        let now = timestamp::now_seconds();
        if (now >= auction.end_time) return auction.end_price;
        if (now <= auction.start_time) return auction.start_price;
        
        let elapsed = now - auction.start_time;
        let duration = auction.end_time - auction.start_time;
        let price_decline = auction.start_price - auction.end_price;
        
        auction.start_price - price_decline * elapsed / duration
    }
    
    public entry fun buy_dutch(
        buyer: &signer,
        auction_addr: address,
        token_amount: u64,
    ) acquires DutchAuction {
        let auction = borrow_global_mut<DutchAuction>(auction_addr);
        assert!(!auction.concluded, 1);
        assert!(auction.sold + token_amount <= auction.total_supply, 2);
        
        let price = current_price(auction);
        let total_cost = price * token_amount;
        
        let buyer_addr = std::signer::address_of(buyer);
        
        // Lock payment (at current high price; refund later)
        let payment = coin::withdraw<aptos_coin::AptosCoin>(buyer, total_cost);
        coin::deposit<aptos_coin::AptosCoin>(auction_addr, payment);
        
        if (!aptos_std::smart_table::contains(&auction.participants, buyer_addr)) {
            aptos_std::smart_table::add(&mut auction.participants, buyer_addr, DutchBid {
                amount: token_amount,
                price_paid: price,
                refund_due: 0,
            });
        } else {
            let bid = aptos_std::smart_table::borrow_mut(&mut auction.participants, buyer_addr);
            bid.amount = bid.amount + token_amount;
            bid.price_paid = price;  // Track last price
        };
        
        auction.sold = auction.sold + token_amount;
        
        // Check if sold out → conclude
        if (auction.sold == auction.total_supply) {
            auction.clearing_price = price;
            auction.concluded = true;
        };
    }
    
    // Claim refund: difference between paid price and clearing price
    public entry fun claim_dutch_refund(
        buyer: &signer,
        auction_addr: address,
    ) acquires DutchAuction {
        let auction = borrow_global<DutchAuction>(auction_addr);
        assert!(auction.concluded, 3);
        
        let buyer_addr = std::signer::address_of(buyer);
        let bid = aptos_std::smart_table::borrow(&auction.participants, buyer_addr);
        
        // Refund = (price_paid - clearing_price) * amount
        if (bid.price_paid > auction.clearing_price) {
            let refund = (bid.price_paid - auction.clearing_price) * bid.amount;
            // Transfer refund to buyer
        };
    }
}
```

---

## Incentive-Compatible Protocols

```move
module game_theory::incentive_design {
    
    // ============================================
    // Incentive-Compatible Liquidation
    // Design so liquidators want to liquidate promptly
    // ============================================
    
    // Problem: Without incentive, no one liquidates
    // Solution: Liquidation bonus that increases with risk
    
    struct LiquidationIncentive has key {
        // Base bonus: 5%
        base_bonus_bps: u64,
        
        // Additional bonus per % below health threshold
        // e.g., 1% extra per 5% below threshold
        urgency_bonus_bps_per_unit: u64,
        urgency_unit_bps: u64,  // e.g., 500 = 5%
        
        // Maximum bonus cap: 20%
        max_bonus_bps: u64,
        
        // Competing liquidator protection window
        // First liquidator gets exclusive window
        exclusive_window_secs: u64,
    }
    
    // Calculate liquidation bonus based on health factor
    public fun calculate_liquidation_bonus(
        incentive: &LiquidationIncentive,
        health_factor: u64,           // In basis points (10000 = 1.0)
        liquidation_threshold: u64,   // e.g., 8000 (80%)
    ): u64 {
        // Base bonus
        let mut bonus = incentive.base_bonus_bps;
        
        // Urgency bonus: the more underwater, the more bonus
        if (health_factor < liquidation_threshold) {
            let underwater_bps = liquidation_threshold - health_factor;
            let units_underwater = underwater_bps / incentive.urgency_unit_bps;
            let urgency_bonus = units_underwater * incentive.urgency_bonus_bps_per_unit;
            bonus = bonus + urgency_bonus;
        };
        
        // Cap at maximum
        std::math64::min(bonus, incentive.max_bonus_bps)
    }
    
    // ============================================
    // Staking Slashing: make attack unprofitable
    // ============================================
    
    // If attacker has S stake and gains G from attack:
    // Expected value = G - P(caught) * slash_amount
    // For slash to deter: slash_amount > G / P(caught)
    
    // With 50% detection probability:
    // slash_amount > 2 * G to make attack unprofitable
    
    struct SlashingConfig has key {
        // Minimum stake to be a validator
        min_stake: u64,
        
        // Slash % for double signing: 100% (full slash)
        double_sign_slash_bps: u64,
        
        // Slash % for downtime: 1% (minor)
        downtime_slash_bps: u64,
        
        // Slash goes to reporting validator
        reporter_reward_bps: u64,  // 5% of slash goes to reporter
    }
    
    public fun compute_slash(
        config: &SlashingConfig,
        validator_stake: u64,
        offense_type: u8,  // 0=downtime, 1=double_sign
    ): (u64, u64) {  // (slash_amount, reporter_reward)
        let slash_bps = if (offense_type == 0) {
            config.downtime_slash_bps
        } else {
            config.double_sign_slash_bps
        };
        
        let slash_amount = validator_stake * slash_bps / 10_000;
        let reporter_reward = slash_amount * config.reporter_reward_bps / 10_000;
        
        (slash_amount, reporter_reward)
    }
    
    // ============================================
    // Token Voting: Whale Mitigation
    // ============================================
    
    // Quadratic voting: sqrt(tokens) = votes
    // Reduces whale dominance
    
    public fun quadratic_votes(token_amount: u64): u64 {
        // Integer square root
        if (token_amount == 0) return 0;
        
        let mut z = (token_amount as u128);
        let mut x = (token_amount as u128 + 1u128) / 2u128;
        
        while (x < z) {
            z = x;
            x = (x + (token_amount as u128) / x) / 2u128;
        };
        
        z as u64
    }
    
    // Conviction Voting: time-weighted votes
    // Long-term holders have more influence
    // vote_weight = tokens * sqrt(lock_time)
    
    public fun conviction_votes(token_amount: u64, lock_days: u64): u64 {
        // Weight = tokens * sqrt(lock_days) / sqrt(max_lock_days)
        let max_lock_days = 365u64;
        let time_weight = quadratic_votes(lock_days * 1000) * 1000 
                         / quadratic_votes(max_lock_days * 1000);
        
        (token_amount as u128 * (time_weight as u128) / 1000u128) as u64
    }
}
```

---

## ตัวอย่าง: Dutch Auction & Sealed Bid

```move
module game_theory::sealed_bid_auction {
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::timestamp;
    
    // ============================================
    // Vickrey (Second-Price) Sealed Bid Auction
    // Truth-telling is dominant strategy:
    // You win by bidding your true value
    // You pay only second-highest bid (lower than your bid)
    // ============================================
    
    struct VickreyAuction has key {
        item_id: u64,
        seller: address,
        commit_deadline: u64,
        reveal_deadline: u64,
        
        // Phase 1: Sealed bids
        commits: SmartTable<address, vector<u8>>,
        
        // Phase 2: Revealed bids
        bids: SmartTable<address, u64>,
        
        // Outcome
        winner: address,
        second_price: u64,
        concluded: bool,
    }
    
    // Commit sealed bid
    public entry fun commit_bid(
        bidder: &signer,
        auction_addr: address,
        commitment: vector<u8>,  // hash(bid_amount, salt)
    ) acquires VickreyAuction {
        let auction = borrow_global_mut<VickreyAuction>(auction_addr);
        assert!(timestamp::now_seconds() <= auction.commit_deadline, 1);
        
        let addr = std::signer::address_of(bidder);
        smart_table::add(&mut auction.commits, addr, commitment);
    }
    
    // Reveal bid
    public entry fun reveal_bid(
        bidder: &signer,
        auction_addr: address,
        bid_amount: u64,
        salt: vector<u8>,
    ) acquires VickreyAuction {
        let auction = borrow_global_mut<VickreyAuction>(auction_addr);
        let now = timestamp::now_seconds();
        assert!(now > auction.commit_deadline, 2);
        assert!(now <= auction.reveal_deadline, 3);
        
        let addr = std::signer::address_of(bidder);
        
        // Verify commitment
        let commit = smart_table::borrow(&auction.commits, addr);
        let mut input = salt;
        let mut i = 0u8;
        while (i < 8) {
            std::vector::push_back(&mut input, ((bid_amount >> (i * 8)) & 0xFF) as u8);
            i = i + 1;
        };
        let expected = aptos_std::aptos_hash::keccak256(input);
        assert!(expected == *commit, 4);
        
        smart_table::add(&mut auction.bids, addr, bid_amount);
    }
    
    // Conclude: find winner (highest bid) and price (second highest)
    // In Move, we'd iterate through bids to find top 2
    
    // Why Vickrey is truthful:
    // Suppose your value = V, you bid B, others' top bid = H
    // 
    // Case 1: B > H (you win) → you pay H (< V if B=V)
    //   Better to bid V: you still win and pay H
    //
    // Case 2: B < H (you lose)  
    //   If V > H: bidding B=V would win at price H > V? No: H < V so bid V wins.
    //   Actually case: B < H and V > H → you should have bid higher
    //   Truthful bidding (B=V) would win at price H < V (profitable)
    //
    // Therefore: Always bid your true value (dominant strategy)
    
    #[view]
    public fun get_bid_after_reveal(
        auction_addr: address,
        bidder: address,
    ): u64 acquires VickreyAuction {
        let auction = borrow_global<VickreyAuction>(auction_addr);
        assert!(timestamp::now_seconds() > auction.reveal_deadline, 5);
        
        if (smart_table::contains(&auction.bids, bidder)) {
            *smart_table::borrow(&auction.bids, bidder)
        } else {
            0
        }
    }
}
```

---

## สรุป Game Theory in DeFi

```
Mechanism Design Principles Applied:

1. LIQUIDATION
   - Make liquidation profitable (5-20% bonus)
   - Increase bonus with urgency
   - Result: Liquidators compete → positions liquidated promptly

2. ORACLE REPORTING  
   - Penalize outliers (Schelling point)
   - Reward honest majority
   - Result: Truth-telling is Nash Equilibrium

3. GOVERNANCE VOTING
   - Quadratic/conviction voting to reduce whale power
   - Time locks to prevent governance attacks
   - Result: Long-term holders have more influence

4. TOKEN SALE (DUTCH AUCTION)
   - Price discovery through market
   - Everyone pays clearing price (fair)
   - Result: No FOMO, efficient allocation

5. MEV DEFENSE
   - Commit-reveal for large trades
   - Rate limiting per block
   - Price impact limits
   - Result: Reduce sandwich attack profitability

Key Insight:
  Good protocol design = honest behavior is MORE profitable than attacks
  This is mechanism design: make desired equilibrium the Nash Equilibrium
```

---

**ก่อนหน้า**: [Part 46 - Security Auditing ←](part-46-security-auditing.md)
**ต่อไป**: [Part 48 - Move VM Internals →](part-48-move-vm-internals.md)
