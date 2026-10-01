# Part 56: On-Chain Randomness & VRF

## สารบัญ
- [Why Randomness is Hard On-Chain](#why-randomness-is-hard-on-chain)
- [Aptos Native Randomness](#aptos-native-randomness)
- [Commit-Reveal Randomness](#commit-reveal-randomness)
- [VRF (Verifiable Random Function)](#vrf-verifiable-random-function)
- [Chainlink VRF Integration Pattern](#chainlink-vrf-integration-pattern)
- [ตัวอย่าง: On-Chain Lottery System](#ตัวอย่าง-on-chain-lottery-system)
- [ตัวอย่าง: NFT Reveal with Fairness](#ตัวอย่าง-nft-reveal-with-fairness)

---

## Why Randomness is Hard On-Chain

```
The On-Chain Randomness Problem:

Blockchain state is deterministic:
  - All nodes must reach same result
  - No "random" source available
  
Common bad approaches:
  ❌ block_hash as randomness → validators can manipulate
  ❌ timestamp as randomness → predictable, manipulable
  ❌ sender address → known in advance
  ❌ tx hash → miner/validator can re-try until favorable

Attack scenario:
  Lottery picks winner from block hash
  Validator: "I'm about to validate this block...
    if block_hash picks me → include this tx
    if not → exclude tx and re-try"
  = BROKEN: validator always wins

Solutions:
  1. Commit-Reveal: 2-phase, no MEV
     Phase 1: Commit hash(secret + nonce)
     Phase 2: Reveal secret, compute random
     
  2. VRF (Verifiable Random Function):
     - Chainlink VRF: off-chain oracle
     - drand: threshold BLS signature network
     - Sui built-in randomness (native)
     - Aptos built-in randomness (Move module)
     
  3. Multi-party computation:
     Multiple parties each contribute entropy
     Need all parties to collude to manipulate
     
Best: Native VRF from L1 (Aptos/Sui have this!)
```

---

## Aptos Native Randomness

```move
module randomness::aptos_vrf {
    use aptos_framework::randomness;
    
    // ============================================
    // Aptos has BUILT-IN randomness (since v1.7+)
    // Uses DKG (Distributed Key Generation) based VRF
    // Validators collectively generate randomness
    // Cannot be manipulated by any single validator
    // ============================================
    
    struct LootBox has key {
        total_opened: u64,
        legendary_count: u64,
    }
    
    // #[randomness] annotation: marks function as using randomness
    // Prevents transaction abort after randomness is revealed
    // (prevents "test-and-abort" attacks)
    
    #[randomness]
    entry fun open_loot_box(user: &signer, loot_box_addr: address) acquires LootBox {
        let loot_box = borrow_global_mut<LootBox>(loot_box_addr);
        
        // Generate a random u64 in [0, 10000)
        let roll = randomness::u64_range(0, 10000);
        
        let rarity = if (roll < 100) {
            // 1%: Legendary
            loot_box.legendary_count = loot_box.legendary_count + 1;
            b"Legendary"
        } else if (roll < 1000) {
            // 9%: Epic
            b"Epic"
        } else if (roll < 3000) {
            // 20%: Rare
            b"Rare"
        } else {
            // 70%: Common
            b"Common"
        };
        
        loot_box.total_opened = loot_box.total_opened + 1;
        
        // Emit event with drop result
        // event::emit(LootOpened { rarity, roll });
    }
    
    // ============================================
    // Shuffle a vector randomly
    // ============================================
    
    #[randomness]
    entry fun shuffle_deck(user: &signer) {
        let mut deck = vector[1u64, 2, 3, 4, 5, 6, 7, 8, 9, 10];
        let len = std::vector::length(&deck);
        
        // Fisher-Yates shuffle using aptos randomness
        let mut i = len - 1;
        while (i > 0) {
            let j = randomness::u64_range(0, i + 1);
            
            // Swap deck[i] and deck[j]
            let temp = *std::vector::borrow(&deck, i);
            let val_j = *std::vector::borrow(&deck, j);
            *std::vector::borrow_mut(&mut deck, i) = val_j;
            *std::vector::borrow_mut(&mut deck, j) = temp;
            
            i = i - 1;
        };
        
        // deck is now randomly shuffled
    }
    
    // ============================================
    // Random team assignment (fair distribution)
    // ============================================
    
    #[randomness]
    entry fun assign_teams(
        users: vector<address>,
        num_teams: u64,
    ) {
        let mut assignments = std::vector::empty<u64>();
        let user_count = std::vector::length(&users);
        
        // Assign each user to random team
        let mut i = 0u64;
        while (i < user_count) {
            let team = randomness::u64_range(0, num_teams);
            std::vector::push_back(&mut assignments, team);
            i = i + 1;
        };
        
        // assignments[i] = team for users[i]
    }
    
    // ============================================
    // Aptos Randomness API
    // ============================================
    
    // Available functions in aptos_framework::randomness:
    // - u8_integer(): random u8
    // - u16_integer(): random u16
    // - u32_integer(): random u32
    // - u64_integer(): random u64
    // - u128_integer(): random u128
    // - u256_integer(): random u256
    // - u64_range(min, max): random in [min, max)
    // - u256_range(min, max): random in [min, max)
    // - bytes(n): n random bytes
    // - permutation(n): random permutation of [0, n)
    
    #[randomness]
    entry fun demo_all_randomness() {
        let _byte: u8 = randomness::u8_integer();
        let _num: u64 = randomness::u64_integer();
        let _big: u128 = randomness::u128_integer();
        let _range: u64 = randomness::u64_range(1, 7);  // 1d6 dice roll
        let _bytes: vector<u8> = randomness::bytes(32);
        let _perm: vector<u64> = randomness::permutation(52);  // Shuffle 52-card deck
    }
}
```

---

## Commit-Reveal Randomness

```move
module randomness::commit_reveal {
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // Commit-Reveal: 2-phase fair randomness
    // Works on chains WITHOUT native randomness
    // ============================================
    
    // Phase 1: Commit (no one sees the secret yet)
    // Phase 2: Reveal (everyone reveals, combine for randomness)
    
    const COMMIT_PHASE_DURATION: u64 = 300;   // 5 minutes
    const REVEAL_PHASE_DURATION: u64 = 300;   // 5 minutes
    
    struct RandomnessGame has key {
        round: u64,
        phase: u8,    // 0=commit, 1=reveal, 2=complete
        
        phase_end: u64,
        
        // Commitments: participant → hash(secret || nonce)
        commitments: SmartTable<address, vector<u8>>,
        
        // Revealed secrets
        revealed: SmartTable<address, vector<u8>>,
        
        // Final random value (XOR of all secrets)
        final_randomness: vector<u8>,
        
        // Participants who committed but didn't reveal = slashed
        slash_amount: u64,
    }
    
    // ============================================
    // Phase 1: Commit
    // ============================================
    
    public entry fun commit(
        participant: &signer,
        game_addr: address,
        commitment: vector<u8>,  // keccak256(secret || nonce || address)
    ) acquires RandomnessGame {
        let game = borrow_global_mut<RandomnessGame>(game_addr);
        assert!(game.phase == 0, 1);
        assert!(aptos_framework::timestamp::now_seconds() < game.phase_end, 2);
        
        let addr = std::signer::address_of(participant);
        assert!(!smart_table::contains(&game.commitments, addr), 3);
        
        // Lock slash deposit
        let deposit = aptos_framework::coin::withdraw<aptos_framework::aptos_coin::AptosCoin>(
            participant, game.slash_amount
        );
        // Store deposit...
        
        smart_table::add(&mut game.commitments, addr, commitment);
    }
    
    // ============================================
    // Phase 2: Reveal
    // ============================================
    
    public entry fun reveal(
        participant: &signer,
        game_addr: address,
        secret: vector<u8>,
        nonce: vector<u8>,
    ) acquires RandomnessGame {
        let game = borrow_global_mut<RandomnessGame>(game_addr);
        assert!(game.phase == 1, 1);
        assert!(aptos_framework::timestamp::now_seconds() < game.phase_end, 2);
        
        let addr = std::signer::address_of(participant);
        assert!(smart_table::contains(&game.commitments, addr), 3);
        assert!(!smart_table::contains(&game.revealed, addr), 4);
        
        // Verify: commitment = keccak256(secret || nonce || addr)
        let mut preimage = secret;
        std::vector::append(&mut preimage, nonce);
        std::vector::append(&mut preimage, bcs::to_bytes(&addr));
        
        let expected_hash = aptos_std::aptos_hash::keccak256(preimage);
        let committed = *smart_table::borrow(&game.commitments, addr);
        
        assert!(expected_hash == committed, 5);
        
        // XOR into accumulated randomness
        xor_into(&mut game.final_randomness, &secret);
        
        smart_table::add(&mut game.revealed, addr, secret);
        
        // Return slash deposit on successful reveal
    }
    
    // ============================================
    // Finalize (after reveal phase ends)
    // ============================================
    
    public entry fun finalize(
        caller: &signer,
        game_addr: address,
    ) acquires RandomnessGame {
        let game = borrow_global_mut<RandomnessGame>(game_addr);
        assert!(game.phase == 1, 1);
        assert!(aptos_framework::timestamp::now_seconds() >= game.phase_end, 2);
        
        game.phase = 2;
        
        // Slash participants who committed but didn't reveal
        // (they tried to abort after seeing others' reveals)
    }
    
    // ============================================
    // Read final randomness
    // ============================================
    
    public fun get_randomness(game_addr: address): vector<u8> acquires RandomnessGame {
        let game = borrow_global<RandomnessGame>(game_addr);
        assert!(game.phase == 2, 1);
        game.final_randomness
    }
    
    // Derive a u64 in [0, max) from random bytes
    public fun derive_u64(random_bytes: &vector<u8>, max: u64): u64 {
        let mut num = 0u64;
        let mut i = 0u64;
        while (i < 8 && i < std::vector::length(random_bytes)) {
            num = (num << 8) | (*std::vector::borrow(random_bytes, i) as u64);
            i = i + 1;
        };
        num % max
    }
    
    fun xor_into(accumulator: &mut vector<u8>, value: &vector<u8>) {
        let len = std::vector::length(value);
        let acc_len = std::vector::length(accumulator);
        
        if (acc_len == 0) {
            *accumulator = *value;
            return
        };
        
        let mut i = 0u64;
        while (i < len && i < acc_len) {
            let a = std::vector::borrow_mut(accumulator, i);
            *a = *a ^ *std::vector::borrow(value, i);
            i = i + 1;
        };
    }
}
```

---

## VRF (Verifiable Random Function)

```
VRF Theory:

VRF = function that:
  1. Takes private key + input → (output, proof)
  2. Anyone can verify: verify(public_key, input, output, proof) = true
  3. Output is deterministic: same input → same output
  4. Output is unpredictable without private key

Used by:
  - Chainlink VRF: Oracle-based
  - drand: League of Entropy's randomness beacon
  - Aptos: BLS VRF from validator set
  - Sui: built-in randomness module

VRF Security:
  - Provably random (proof attached)
  - Cannot be manipulated by oracle (proof verifiable)
  - Cannot be predicted before reveal

BLS VRF (Aptos uses this):
  Private key: sk
  Public key: pk = sk * G
  
  Input: message m
  
  VRF compute:
    H = hash_to_curve(m)  // Map message to curve point
    Gamma = sk * H        // VRF output point
    output = hash(Gamma)  // Final random bytes
    proof = (c, s) where:
      c = hash(G, H, pk, Gamma, k*G, k*H)
      s = k - c*sk (mod q)
  
  VRF verify:
    U = s*G + c*pk
    V = s*H + c*Gamma
    check: hash(G, H, pk, Gamma, U, V) == c
```

---

## Chainlink VRF Integration Pattern

```move
module randomness::chainlink_vrf_pattern {
    
    // ============================================
    // Pattern: Request-Fulfill with Oracle
    // ============================================
    
    // Aptos doesn't have Chainlink VRF natively,
    // but here's the pattern for reference (EVM-style)
    // and how to build equivalent oracle-based VRF on Aptos
    
    struct VRFConsumer has key {
        // VRF coordinator address (oracle)
        vrf_coordinator: address,
        
        // Subscription ID for payment
        subscription_id: u64,
        
        // Callback function identifier
        // (which game to deliver randomness to)
        
        // Pending requests: request_id → game context
        pending_requests: aptos_std::smart_table::SmartTable<vector<u8>, GameContext>,
        
        // Fulfilled: request_id → random_words
        fulfilled: aptos_std::smart_table::SmartTable<vector<u8>, vector<u64>>,
    }
    
    struct GameContext has store, drop {
        game_type: u8,
        player: address,
        metadata: vector<u8>,
    }
    
    // ============================================
    // Step 1: Request randomness
    // ============================================
    
    public entry fun request_random_word(
        caller: &signer,
        consumer_addr: address,
        game_type: u8,
        metadata: vector<u8>,
    ) acquires VRFConsumer {
        let consumer = borrow_global_mut<VRFConsumer>(consumer_addr);
        let player = std::signer::address_of(caller);
        
        // Generate request ID
        let request_id = generate_request_id(player, game_type, &metadata);
        
        assert!(
            !aptos_std::smart_table::contains(&consumer.pending_requests, request_id),
            1
        );
        
        // Store context
        aptos_std::smart_table::add(&mut consumer.pending_requests, request_id, GameContext {
            game_type,
            player,
            metadata,
        });
        
        // Emit event for oracle to pick up
        // oracle sees event → generates VRF → calls fulfill_randomness
    }
    
    // ============================================
    // Step 2: Oracle fulfills randomness
    // ============================================
    
    public entry fun fulfill_randomness(
        oracle: &signer,  // Must be VRF coordinator
        consumer_addr: address,
        request_id: vector<u8>,
        random_words: vector<u64>,
        proof: vector<u8>,  // VRF proof (verifiable)
    ) acquires VRFConsumer {
        let consumer = borrow_global_mut<VRFConsumer>(consumer_addr);
        
        // Verify oracle is authorized
        assert!(
            std::signer::address_of(oracle) == consumer.vrf_coordinator,
            2
        );
        
        // Verify VRF proof (simplified - real impl would check BLS sig)
        assert!(verify_vrf_proof(&proof, &request_id, &random_words), 3);
        
        // Get pending context
        assert!(
            aptos_std::smart_table::contains(&consumer.pending_requests, request_id),
            4
        );
        let context = aptos_std::smart_table::remove(
            &mut consumer.pending_requests,
            request_id
        );
        
        // Store result
        aptos_std::smart_table::add(&mut consumer.fulfilled, request_id, random_words);
        
        // Process game result
        process_game_result(context, random_words);
    }
    
    fun process_game_result(context: GameContext, random_words: vector<u64>) {
        if (context.game_type == 1) {
            // Lottery result
        } else if (context.game_type == 2) {
            // NFT trait roll
        };
    }
    
    fun generate_request_id(player: address, game_type: u8, metadata: &vector<u8>): vector<u8> {
        let mut preimage = bcs::to_bytes(&player);
        std::vector::push_back(&mut preimage, game_type);
        std::vector::append(&mut preimage, *metadata);
        std::vector::append(&mut preimage, bcs::to_bytes(
            &aptos_framework::timestamp::now_microseconds()
        ));
        aptos_std::aptos_hash::keccak256(preimage)
    }
    
    fun verify_vrf_proof(_proof: &vector<u8>, _request_id: &vector<u8>, _words: &vector<u64>): bool {
        true  // Simplified
    }
}
```

---

## ตัวอย่าง: On-Chain Lottery System

```move
module randomness::lottery {
    use aptos_framework::randomness;
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::coin;
    use aptos_framework::aptos_coin::AptosCoin;
    
    const TICKET_PRICE: u64 = 1_000_000;     // 0.01 APT
    const LOTTERY_DURATION: u64 = 7 * 24 * 3600;  // 1 week
    const JACKPOT_SHARE_BPS: u64 = 8000;     // 80% jackpot
    const PROTOCOL_FEE_BPS: u64 = 500;       // 5% protocol
    const ROLLOVER_BPS: u64 = 1500;          // 15% rolls to next round
    
    struct LotteryRound has key {
        round_id: u64,
        jackpot: u64,
        start_time: u64,
        end_time: u64,
        
        // Tickets: ticket_number → owner
        tickets: SmartTable<u64, address>,
        next_ticket: u64,
        
        // Selected winner
        winning_ticket: std::option::Option<u64>,
        winner: std::option::Option<address>,
        
        // Payout tracking
        payout_claimed: bool,
        
        // Protocol
        protocol_treasury: address,
    }
    
    public entry fun buy_tickets(
        buyer: &signer,
        lottery_addr: address,
        count: u64,  // How many tickets to buy
    ) acquires LotteryRound {
        let lottery = borrow_global_mut<LotteryRound>(lottery_addr);
        let now = aptos_framework::timestamp::now_seconds();
        
        assert!(now >= lottery.start_time, 1);
        assert!(now < lottery.end_time, 2);
        assert!(lottery.winning_ticket == std::option::none(), 3);
        
        let buyer_addr = std::signer::address_of(buyer);
        let total_cost = TICKET_PRICE * count;
        
        // Collect payment
        let payment = coin::withdraw<AptosCoin>(buyer, total_cost);
        
        // Distribute payment
        let jackpot_contribution = total_cost * JACKPOT_SHARE_BPS / 10_000;
        let protocol_fee = total_cost * PROTOCOL_FEE_BPS / 10_000;
        let rollover = total_cost - jackpot_contribution - protocol_fee;
        
        lottery.jackpot = lottery.jackpot + jackpot_contribution;
        // Send protocol fee and rollover to respective addresses
        
        // Assign tickets
        let mut i = 0u64;
        while (i < count) {
            smart_table::add(&mut lottery.tickets, lottery.next_ticket, buyer_addr);
            lottery.next_ticket = lottery.next_ticket + 1;
            i = i + 1;
        };
        
        // Cleanup payment (simplified)
        coin::deposit(lottery.protocol_treasury, payment);
    }
    
    // Draw winner using Aptos native randomness
    #[randomness]
    entry fun draw_winner(
        lottery_addr: address,
    ) acquires LotteryRound {
        let lottery = borrow_global_mut<LotteryRound>(lottery_addr);
        let now = aptos_framework::timestamp::now_seconds();
        
        assert!(now >= lottery.end_time, 1);
        assert!(lottery.winning_ticket == std::option::none(), 2);
        assert!(lottery.next_ticket > 0, 3);  // At least one ticket sold
        
        // Use Aptos native randomness
        let winning_ticket = randomness::u64_range(0, lottery.next_ticket);
        
        let winner_addr = *smart_table::borrow(&lottery.tickets, winning_ticket);
        
        lottery.winning_ticket = std::option::some(winning_ticket);
        lottery.winner = std::option::some(winner_addr);
    }
    
    public entry fun claim_prize(
        winner: &signer,
        lottery_addr: address,
    ) acquires LotteryRound {
        let lottery = borrow_global_mut<LotteryRound>(lottery_addr);
        let caller = std::signer::address_of(winner);
        
        assert!(!lottery.payout_claimed, 1);
        assert!(lottery.winner == std::option::some(caller), 2);
        
        lottery.payout_claimed = true;
        
        let jackpot = lottery.jackpot;
        lottery.jackpot = 0;
        
        // Send jackpot to winner
        // coin::transfer<AptosCoin>(jackpot_pool, caller, jackpot);
    }
}
```

---

## ตัวอย่าง: NFT Reveal with Fairness

```move
module randomness::nft_reveal {
    use aptos_framework::randomness;
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // Blind Box NFT with Fair Reveal
    // ============================================
    
    // Problem: If mint reveals traits during mint,
    // users can simulate and cherry-pick
    
    // Solution: 
    //   1. Sell "blind boxes" (no traits revealed)
    //   2. After all sold, do ONE reveal (randomness picks all traits)
    //   3. Assign traits based on random seed
    
    const MAX_SUPPLY: u64 = 10_000;
    
    struct BlindBoxCollection has key {
        // Sold token IDs → owner
        sold_tokens: SmartTable<u64, address>,
        next_token_id: u64,
        
        // Revealed traits (after reveal event)
        token_traits: SmartTable<u64, TokenTraits>,
        
        // Reveal state
        reveal_done: bool,
        reveal_seed: vector<u8>,
        
        // Rarity tiers
        // 0-99: legendary (1%), 100-999: epic (9%), 1000-2999: rare (20%), 3000-9999: common (70%)
        legendary_count: u64,
        epic_count: u64,
        rare_count: u64,
    }
    
    struct TokenTraits has store, drop {
        background: u8,
        body: u8,
        head: u8,
        eyes: u8,
        mouth: u8,
        rarity: u8,  // 0=common, 1=rare, 2=epic, 3=legendary
    }
    
    // Mint blind box (before reveal)
    public entry fun mint_blind_box(
        buyer: &signer,
        collection_addr: address,
    ) acquires BlindBoxCollection {
        let collection = borrow_global_mut<BlindBoxCollection>(collection_addr);
        assert!(collection.next_token_id < MAX_SUPPLY, 1);
        assert!(!collection.reveal_done, 2);
        
        let buyer_addr = std::signer::address_of(buyer);
        let token_id = collection.next_token_id;
        
        // Pay mint price...
        
        smart_table::add(&mut collection.sold_tokens, token_id, buyer_addr);
        collection.next_token_id = token_id + 1;
    }
    
    // ============================================
    // Reveal all tokens at once (fair)
    // ============================================
    
    #[randomness]
    entry fun reveal_collection(
        admin: &signer,
        collection_addr: address,
    ) acquires BlindBoxCollection {
        let collection = borrow_global_mut<BlindBoxCollection>(collection_addr);
        assert!(!collection.reveal_done, 1);
        
        // Get reveal seed
        let seed = randomness::bytes(32);
        collection.reveal_seed = seed;
        collection.reveal_done = true;
        
        // Use permutation to fairly assign rarities
        // (no token gets preference)
        let perm = randomness::permutation(collection.next_token_id);
        
        // Distribute rarities based on permutation
        let total = collection.next_token_id;
        let legendary_count = total / 100;    // 1%
        let epic_count = total * 9 / 100;     // 9%
        let rare_count = total * 20 / 100;    // 20%
        
        let mut i = 0u64;
        while (i < total) {
            let token_id = *std::vector::borrow(&perm, i);
            
            let rarity = if (i < legendary_count) {
                3u8  // Legendary
            } else if (i < legendary_count + epic_count) {
                2u8  // Epic
            } else if (i < legendary_count + epic_count + rare_count) {
                1u8  // Rare
            } else {
                0u8  // Common
            };
            
            // Derive other traits from seed + token_id
            let traits = derive_traits(&seed, token_id, rarity);
            
            smart_table::add(&mut collection.token_traits, token_id, traits);
            
            i = i + 1;
        };
    }
    
    fun derive_traits(seed: &vector<u8>, token_id: u64, rarity: u8): TokenTraits {
        // Hash seed + token_id to get deterministic-but-unpredictable traits
        let mut preimage = *seed;
        std::vector::append(&mut preimage, bcs::to_bytes(&token_id));
        let hash = aptos_std::aptos_hash::keccak256(preimage);
        
        TokenTraits {
            background: *std::vector::borrow(&hash, 0) % 20,
            body: *std::vector::borrow(&hash, 1) % 15,
            head: *std::vector::borrow(&hash, 2) % 25,
            eyes: *std::vector::borrow(&hash, 3) % 30,
            mouth: *std::vector::borrow(&hash, 4) % 20,
            rarity,
        }
    }
}
```

---

## สรุป Randomness & VRF

```
Best Randomness Approach by Use Case:

Simple games / loot boxes / lottery:
  → Aptos #[randomness] module (native, easy)
  → Sui randomness module (native)
  Best: 1 transaction, instant, tamper-proof

Multi-party fairness (poker, competitive):
  → Commit-Reveal with multiple participants
  Each party contributes entropy
  Cannot manipulate without colluding ALL parties
  Cost: 2 transactions per round

Oracle-dependent (cross-chain, off-chain triggers):
  → VRF oracle pattern
  Cost: oracle fee + latency
  
Do NOT use:
  ❌ block_hash, timestamp, tx_hash
  ❌ Single-party commit-reveal (oracle can abort)
  ❌ block.number mod N

Security Notes:
  - #[randomness] prevents test-and-abort attacks
  - Commit-Reveal: slash non-revealers (economic incentive)
  - Never reveal randomness in same tx as consuming it
    (someone could simulate outcome and decide to abort)
  - Permutation-based reveals = mathematically fair

Aptos Native Randomness API:
  randomness::u64_range(min, max)
  randomness::bytes(n)
  randomness::permutation(n)
  
  Mark entry functions with #[randomness]
  Cannot be called from non-randomness context
```

---

**ก่อนหน้า**: [Part 55 - NFT Infrastructure ←](part-55-nft-infrastructure.md)
**ต่อไป**: [Part 57 - Advanced Testing & Fuzzing →](part-57-advanced-testing.md)
