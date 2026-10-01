# Part 44: Zero-Knowledge Proofs Integration

## สารบัญ
- [ZK Basics for Blockchain](#zk-basics-for-blockchain)
- [zkLogin on Aptos](#zklogin-on-aptos)
- [Verifying ZK Proofs in Move](#verifying-zk-proofs-in-move)
- [Privacy-Preserving Patterns](#privacy-preserving-patterns)
- [ZK Rollup Concepts](#zk-rollup-concepts)
- [ตัวอย่าง: Private Voting System](#ตัวอย่าง-private-voting-system)

---

## ZK Basics for Blockchain

```
Zero-Knowledge Proof = พิสูจน์ความรู้โดยไม่เปิดเผยข้อมูล

Types:
  - zk-SNARK: small proof, fast verify, trusted setup
  - zk-STARK: no trusted setup, larger proof, quantum resistant
  - Groth16: popular SNARK (used in Zcash, zkLogin)
  - PLONK: universal trusted setup
  
Use Cases in DeFi:
  1. zkLogin: login with Google/Apple without revealing identity
  2. Private transactions: hide amounts/parties
  3. KYC without revealing personal data
  4. Airdrop eligibility without doxxing
  5. Range proofs: prove amount in range without revealing amount
  6. Membership proofs: prove you're in a set without revealing which
  
On-chain verification:
  Prover: generates proof (off-chain, expensive)
  Verifier: checks proof (on-chain, cheap)
```

---

## zkLogin on Aptos

```move
module zklogin::integration {
    use std::signer;
    use aptos_framework::account;
    
    // ============================================
    // zkLogin: ephemeral key + OpenID Connect proof
    // ============================================
    
    // zkLogin flow:
    // 1. User generates ephemeral key pair (short-lived)
    // 2. User logs in with Google (gets JWT)
    // 3. ZK circuit proves: I have valid JWT without revealing it
    // 4. Proof binds JWT subject to ephemeral public key
    // 5. User signs txs with ephemeral key
    // 6. On-chain verifier checks ZK proof
    
    struct ZKProof has drop {
        proof_bytes: vector<u8>,     // Groth16 proof
        public_inputs: vector<u8>,   // Hash(iss + sub + aud + nonce)
        ephemeral_pubkey: vector<u8>, // Short-lived key
        max_epoch: u64,             // Proof validity
    }
    
    struct ZKLoginConfig has key {
        // Verification key for zkLogin circuit
        vk: vector<u8>,
        
        // Trusted JWT issuers (Google, Apple, etc.)
        trusted_issuers: vector<vector<u8>>,
        
        // Nullifier storage (prevent proof replay)
        used_nullifiers: aptos_std::smart_table::SmartTable<vector<u8>, bool>,
    }
    
    const E_INVALID_PROOF: u64 = 1;
    const E_EXPIRED_PROOF: u64 = 2;
    const E_PROOF_REPLAYED: u64 = 3;
    const E_UNTRUSTED_ISSUER: u64 = 4;
    
    // ============================================
    // Verify zkLogin proof (simplified interface)
    // In production: Aptos framework handles this natively
    // ============================================
    
    public fun verify_zklogin(
        config_addr: address,
        proof: &ZKProof,
        current_epoch: u64,
    ): address acquires ZKLoginConfig {
        use aptos_std::smart_table;
        let config = borrow_global_mut<ZKLoginConfig>(config_addr);
        
        // Check epoch not expired
        assert!(current_epoch <= proof.max_epoch, E_EXPIRED_PROOF);
        
        // Calculate nullifier (unique per login session)
        let nullifier = aptos_std::aptos_hash::keccak256(proof.proof_bytes);
        
        // Check not replayed
        assert!(!smart_table::contains(&config.used_nullifiers, nullifier), E_PROOF_REPLAYED);
        
        // Verify ZK proof against verification key
        // In Aptos: uses native groth16 verification
        let valid = verify_groth16_proof(
            &config.vk,
            &proof.proof_bytes,
            &proof.public_inputs,
        );
        assert!(valid, E_INVALID_PROOF);
        
        // Mark nullifier as used
        smart_table::add(&mut config.used_nullifiers, nullifier, true);
        
        // Derive address from ephemeral pubkey
        derive_address_from_pubkey(&proof.ephemeral_pubkey)
    }
    
    fun verify_groth16_proof(
        vk: &vector<u8>,
        proof: &vector<u8>,
        public_inputs: &vector<u8>,
    ): bool {
        // In production: use aptos_std::crypto_algebra for BLS12-381 operations
        // or call native function
        // For demo: always return true (you'd use actual cryptography)
        true
    }
    
    fun derive_address_from_pubkey(pubkey: &vector<u8>): address {
        // SHA3-256 of pubkey → address
        let hash = aptos_std::aptos_hash::sha3_256(*pubkey);
        // Convert 32-byte hash to address
        // In Move: addresses are 32 bytes
        @0x0  // placeholder
    }
    
    // ============================================
    // Using zkLogin in a DeFi protocol
    // ============================================
    
    struct ZKAccount has key {
        proven_identity_hash: vector<u8>,  // Hash of Google subject
        linked_address: address,
        created_at: u64,
    }
    
    // User creates zkLogin account with social login
    public entry fun create_zk_account(
        ephemeral_signer: &signer,
        config_addr: address,
        proof: ZKProof,
        current_epoch: u64,
    ) acquires ZKLoginConfig {
        let user_addr = verify_zklogin(config_addr, &proof, current_epoch);
        
        move_to(ephemeral_signer, ZKAccount {
            proven_identity_hash: proof.public_inputs,
            linked_address: user_addr,
            created_at: aptos_framework::timestamp::now_seconds(),
        });
    }
}
```

---

## Verifying ZK Proofs in Move

```move
module zk::verifier {
    use aptos_std::crypto_algebra;
    use aptos_std::bn254_algebra;
    
    // ============================================
    // BN254 Pairing-based ZK Proof Verification
    // Used for Groth16 proofs
    // ============================================
    
    // Verification key (fixed for a given circuit)
    struct VerificationKey has store, drop {
        alpha_g1: vector<u8>,         // G1 point
        beta_g2: vector<u8>,          // G2 point
        gamma_g2: vector<u8>,         // G2 point
        delta_g2: vector<u8>,         // G2 point
        ic: vector<vector<u8>>,       // G1 points (one per public input + 1)
    }
    
    // Groth16 proof
    struct Proof has drop {
        a: vector<u8>,    // G1 point
        b: vector<u8>,    // G2 point
        c: vector<u8>,    // G1 point
    }
    
    // ============================================
    // Groth16 verify: 
    // e(A, B) = e(alpha, beta) * e(vk_x, gamma) * e(C, delta)
    // ============================================
    
    public fun verify_groth16(
        vk: &VerificationKey,
        proof: &Proof,
        public_inputs: &vector<u256>,
    ): bool {
        // In production: use aptos_std::crypto_algebra
        // which provides BN254 pairing operations
        
        // Step 1: Compute vk_x = IC[0] + sum(IC[i] * public_inputs[i-1])
        // Step 2: Verify pairing equation
        
        // Simplified (actual impl uses native crypto ops):
        // aptos_std::bn254_algebra operations
        
        true  // placeholder
    }
    
    // ============================================
    // Range Proof Verifier
    // Prove: min <= secret <= max without revealing secret
    // ============================================
    
    struct RangeProofConfig has key {
        min: u64,
        max: u64,
        vk: VerificationKey,
    }
    
    // Commitment = hash(secret, randomness) 
    // Proof shows secret is in [min, max] without revealing it
    
    public fun verify_range_proof(
        config_addr: address,
        commitment: vector<u8>,
        proof_bytes: vector<u8>,
    ): bool acquires RangeProofConfig {
        let config = borrow_global<RangeProofConfig>(config_addr);
        
        // Parse proof
        let proof = parse_proof(&proof_bytes);
        
        // Public inputs: [commitment, min, max]
        let mut public_inputs = std::vector::empty<u256>();
        // Add commitment hash as u256
        // Add min and max
        
        verify_groth16(&config.vk, &proof, &public_inputs)
    }
    
    fun parse_proof(bytes: &vector<u8>): Proof {
        // Parse bytes into proof components
        Proof {
            a: std::vector::empty(),
            b: std::vector::empty(),
            c: std::vector::empty(),
        }
    }
    
    // ============================================
    // Merkle Proof Verifier (for membership proofs)
    // ============================================
    
    // Prove element is in a Merkle tree without revealing which leaf
    
    struct MerkleRoot has key {
        root: vector<u8>,
        leaf_count: u64,
    }
    
    public fun verify_merkle_proof(
        root: &vector<u8>,
        leaf: &vector<u8>,
        proof_path: &vector<vector<u8>>,
        proof_bits: &vector<bool>,  // true=right, false=left sibling
    ): bool {
        let n = std::vector::length(proof_path);
        assert!(n == std::vector::length(proof_bits), 1);
        
        let mut current = *leaf;
        let mut i = 0u64;
        
        while (i < n) {
            let sibling = *std::vector::borrow(proof_path, i);
            let is_right = *std::vector::borrow(proof_bits, i);
            
            current = if (is_right) {
                // current is left, sibling is right
                hash_pair(&current, &sibling)
            } else {
                // sibling is left, current is right
                hash_pair(&sibling, &current)
            };
            
            i = i + 1;
        };
        
        current == *root
    }
    
    fun hash_pair(left: &vector<u8>, right: &vector<u8>): vector<u8> {
        let mut combined = *left;
        std::vector::append(&mut combined, *right);
        aptos_std::aptos_hash::keccak256(combined)
    }
}
```

---

## Privacy-Preserving Patterns

```move
module privacy::commitments {
    use aptos_framework::timestamp;
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // Commit-Reveal with Pedersen Commitments
    // Hide amount until reveal phase
    // ============================================
    
    // Pedersen commitment: C = g^v * h^r
    // (simplified to hash in this demo)
    
    struct PrivateBalance has key {
        commitment: vector<u8>,  // hash(amount, randomness)
        nullifier: vector<u8>,   // prevents double-spend
    }
    
    struct NullifierSet has key {
        spent: SmartTable<vector<u8>, bool>,
    }
    
    // Commit to a private balance
    public entry fun commit_balance(
        user: &signer,
        amount: u64,
        randomness: vector<u8>,
    ) {
        // Commitment = Hash(amount || randomness)
        let mut input = randomness;
        // Encode amount as bytes
        let mut i = 0u8;
        while (i < 8) {
            std::vector::push_back(
                &mut input,
                ((amount >> (i * 8)) & 0xFF) as u8,
            );
            i = i + 1;
        };
        
        let commitment = aptos_std::aptos_hash::keccak256(input);
        
        // Nullifier = Hash("nullifier" || randomness)
        let mut null_input = b"nullifier";
        std::vector::append(&mut null_input, randomness);
        let nullifier = aptos_std::aptos_hash::keccak256(null_input);
        
        move_to(std::signer::borrow(user), PrivateBalance {
            commitment,
            nullifier,
        });
    }
    
    // Spend private balance (reveal amount + nullifier)
    public entry fun spend_balance(
        user: &signer,
        nullifier_set_addr: address,
        amount: u64,
        randomness: vector<u8>,
    ) acquires PrivateBalance, NullifierSet {
        let user_addr = std::signer::address_of(user);
        let balance = borrow_global<PrivateBalance>(user_addr);
        let nullifier_set = borrow_global_mut<NullifierSet>(nullifier_set_addr);
        
        // Verify nullifier not used
        assert!(
            !smart_table::contains(&nullifier_set.spent, balance.nullifier),
            1  // Already spent
        );
        
        // Verify commitment matches
        let mut input = randomness;
        let mut i = 0u8;
        while (i < 8) {
            std::vector::push_back(
                &mut input,
                ((amount >> (i * 8)) & 0xFF) as u8,
            );
            i = i + 1;
        };
        let expected_commitment = aptos_std::aptos_hash::keccak256(input);
        assert!(expected_commitment == balance.commitment, 2);
        
        // Mark nullifier as spent
        smart_table::add(&mut nullifier_set.spent, balance.nullifier, true);
        
        // Send `amount` to user
    }
    
    // ============================================
    // Anonymous Airdrop
    // Claim airdrop without revealing which address got it
    // ============================================
    
    struct AirdropMerkleTree has key {
        root: vector<u8>,
        total_amount: u64,
        claimed: SmartTable<vector<u8>, bool>,  // nullifier -> claimed
    }
    
    // Claim using Merkle proof (proves you're in the list)
    // Without revealing which leaf you are
    
    public entry fun claim_anonymous_airdrop(
        claimer: &signer,
        airdrop_addr: address,
        // ZK proof that claimer knows a preimage for some leaf in the tree
        zk_proof: vector<u8>,
        // Nullifier (prevents double claiming)
        nullifier: vector<u8>,
        // Amount claimed
        amount: u64,
        // ZK public inputs
        public_inputs: vector<u8>,
    ) acquires AirdropMerkleTree {
        let airdrop = borrow_global_mut<AirdropMerkleTree>(airdrop_addr);
        
        // Check nullifier not used
        assert!(
            !smart_table::contains(&airdrop.claimed, nullifier),
            1
        );
        
        // Verify ZK proof
        // (proves: I know (leaf, path) such that:
        //   1. leaf is in the Merkle tree with root = airdrop.root
        //   2. leaf corresponds to amount
        //   3. nullifier is derived from leaf's secret)
        
        // In production: actual ZK verification here
        let valid = true;  // placeholder
        assert!(valid, 2);
        
        // Mark nullifier
        smart_table::add(&mut airdrop.claimed, nullifier, true);
        
        // Send airdrop
        // coin::deposit(signer::address_of(claimer), minted_amount);
    }
}
```

---

## ZK Rollup Concepts

```
ZK Rollup Architecture:
  Layer 2 (off-chain):
    - Process many transactions
    - Generate ZK proof of state transition
    - Submit compressed state + proof to L1
    
  Layer 1 (Aptos on-chain):
    - Store state roots
    - Verify ZK proofs
    - Process deposits/withdrawals
    
Benefits:
  - 100-1000x throughput
  - Inherited L1 security
  - Near-instant finality
  - Lower fees
  
Components:
  1. Sequencer: orders transactions
  2. Prover: generates ZK proofs
  3. Verifier Contract: checks proofs on L1
  4. Bridge: moves assets between L1 and L2

Move Implementation (L1 verifier):
```

```move
module zkrollup::verifier {
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::event;
    
    struct RollupState has key {
        admin: address,
        
        // State tracking
        current_batch_id: u64,
        state_root: vector<u8>,     // Merkle root of L2 state
        pending_deposits: SmartTable<u64, PendingDeposit>,
        next_deposit_id: u64,
        
        // Proof verification
        verifier_key: vector<u8>,
    }
    
    struct PendingDeposit has copy, drop, store {
        user: address,
        amount: u64,
        deposited_at: u64,
    }
    
    struct BatchProof has drop {
        batch_id: u64,
        prev_state_root: vector<u8>,
        new_state_root: vector<u8>,
        transactions_hash: vector<u8>,
        zk_proof: vector<u8>,
    }
    
    #[event]
    struct DepositQueued has drop, store {
        deposit_id: u64,
        user: address,
        amount: u64,
    }
    
    #[event]
    struct BatchProcessed has drop, store {
        batch_id: u64,
        prev_root: vector<u8>,
        new_root: vector<u8>,
    }
    
    // Queue L1 → L2 deposit
    public entry fun deposit<Token>(
        user: &signer,
        rollup_addr: address,
        coins: Coin<Token>,
    ) acquires RollupState {
        let rollup = borrow_global_mut<RollupState>(rollup_addr);
        let amount = coin::value(&coins);
        let user_addr = std::signer::address_of(user);
        
        // Lock tokens in rollup contract
        // (in production: merge into rollup's coin store)
        
        let deposit_id = rollup.next_deposit_id;
        rollup.next_deposit_id = deposit_id + 1;
        
        smart_table::add(&mut rollup.pending_deposits, deposit_id, PendingDeposit {
            user: user_addr,
            amount,
            deposited_at: aptos_framework::timestamp::now_seconds(),
        });
        
        event::emit(DepositQueued { deposit_id, user: user_addr, amount });
        
        coin::destroy_zero(coin::extract(&mut coins, 0));  // placeholder
    }
    
    // Submit and verify a batch of L2 transactions
    public entry fun submit_batch(
        sequencer: &signer,
        rollup_addr: address,
        batch: BatchProof,
    ) acquires RollupState {
        let rollup = borrow_global_mut<RollupState>(rollup_addr);
        
        // Verify batch is sequential
        assert!(batch.batch_id == rollup.current_batch_id + 1, 1);
        
        // Verify previous state root matches
        assert!(batch.prev_state_root == rollup.state_root, 2);
        
        // Verify ZK proof
        let valid = verify_batch_proof(
            &rollup.verifier_key,
            &batch.zk_proof,
            &batch.prev_state_root,
            &batch.new_state_root,
            &batch.transactions_hash,
        );
        assert!(valid, 3);
        
        // Update state
        rollup.current_batch_id = batch.batch_id;
        rollup.state_root = batch.new_state_root;
        
        event::emit(BatchProcessed {
            batch_id: batch.batch_id,
            prev_root: batch.prev_state_root,
            new_root: batch.new_state_root,
        });
    }
    
    fun verify_batch_proof(
        vk: &vector<u8>,
        proof: &vector<u8>,
        prev_root: &vector<u8>,
        new_root: &vector<u8>,
        tx_hash: &vector<u8>,
    ): bool {
        // In production: verify Groth16 proof
        // Public inputs: [prev_root, new_root, tx_hash]
        true  // placeholder
    }
    
    // L2 → L1 withdrawal (requires ZK proof of balance)
    public entry fun withdraw<Token>(
        user: &signer,
        rollup_addr: address,
        amount: u64,
        merkle_proof: vector<u8>,
        nullifier: vector<u8>,
    ) acquires RollupState {
        // Verify user has balance in current state root
        // (via Merkle proof against state_root)
        let rollup = borrow_global<RollupState>(rollup_addr);
        
        let valid = zk::verifier::verify_merkle_proof(
            &rollup.state_root,
            // hash(user || amount || nullifier),
            &nullifier,  // placeholder for leaf
            &std::vector::empty(),  // proof path
            &std::vector::empty(),  // proof bits
        );
        assert!(valid, 4);
        
        // Transfer tokens to user
    }
}
```

---

## ตัวอย่าง: Private Voting System

```move
module privacy::voting {
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // Anonymous voting using ZK proofs
    // Vote without revealing who you voted for
    // ============================================
    
    struct Ballot has key {
        id: u64,
        question: vector<u8>,
        options: vector<vector<u8>>,
        vote_counts: vector<u64>,  // tally per option
        nullifiers: SmartTable<vector<u8>, bool>,
        start_time: u64,
        end_time: u64,
        eligibility_root: vector<u8>,  // Merkle root of eligible voters
    }
    
    const E_NOT_ELIGIBLE: u64 = 1;
    const E_DOUBLE_VOTE: u64 = 2;
    const E_INVALID_PROOF: u64 = 3;
    const E_VOTING_CLOSED: u64 = 4;
    
    public entry fun cast_vote(
        ballot_addr: address,
        // ZK proof: I am in eligibility Merkle tree
        // AND I haven't voted before (nullifier unique)
        zk_proof: vector<u8>,
        nullifier: vector<u8>,
        vote_option: u64,
        // Public inputs for proof verification
        public_inputs: vector<u8>,
    ) acquires Ballot {
        let ballot = borrow_global_mut<Ballot>(ballot_addr);
        
        let now = aptos_framework::timestamp::now_seconds();
        assert!(now >= ballot.start_time && now <= ballot.end_time, E_VOTING_CLOSED);
        
        // Check nullifier not used
        assert!(!smart_table::contains(&ballot.nullifiers, nullifier), E_DOUBLE_VOTE);
        
        // Verify ZK proof (proves eligibility without revealing identity)
        // The proof shows: I know a secret s such that
        //   Hash(s) is a leaf in eligibility_root
        //   nullifier = Hash("vote", s)
        let valid = true;  // placeholder for actual ZK verification
        assert!(valid, E_INVALID_PROOF);
        
        // Record nullifier (prevent double voting)
        smart_table::add(&mut ballot.nullifiers, nullifier, true);
        
        // Count vote
        assert!(vote_option < std::vector::length(&ballot.vote_counts), 5);
        let count = std::vector::borrow_mut(&mut ballot.vote_counts, vote_option);
        *count = *count + 1;
    }
    
    #[view]
    public fun results(ballot_addr: address): vector<u64> acquires Ballot {
        borrow_global<Ballot>(ballot_addr).vote_counts
    }
    
    #[view]
    public fun winner(ballot_addr: address): u64 acquires Ballot {
        let ballot = borrow_global<Ballot>(ballot_addr);
        let counts = &ballot.vote_counts;
        let n = std::vector::length(counts);
        
        let mut max_votes = 0u64;
        let mut winner_idx = 0u64;
        let mut i = 0u64;
        
        while (i < n) {
            let votes = *std::vector::borrow(counts, i);
            if (votes > max_votes) {
                max_votes = votes;
                winner_idx = i;
            };
            i = i + 1;
        };
        
        winner_idx
    }
}
```

---

## สรุป ZK Integration

```
Use Case           | ZK Type         | Benefit
-------------------|-----------------|------------------
Login              | zkLogin/Groth16 | No private key exposure
Private balance    | Commitment      | Hide amounts
Anonymous airdrop  | Merkle + ZK     | No doxxing
Private voting     | Nullifier + ZK  | Secret ballot
KYC without reveal | ZK credential   | Prove age/country
L2 rollup          | ZK-SNARK        | 100x throughput

ZK in Move Reality:
  - Aptos has native BN254 crypto primitives
  - zkLogin is production-ready on Aptos mainnet
  - Full ZK systems need off-chain proof generation
  - Move verifies proofs cheaply on-chain
  - Growing ecosystem of ZK tools for Move
```

---

**ก่อนหน้า**: [Part 43 - Tokenomics ←](part-43-tokenomics.md)
**ต่อไป**: [Part 45 - Liquid Staking Protocol →](part-45-liquid-staking.md)
