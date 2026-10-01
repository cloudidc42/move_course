# Part 64: Layer 2 & Rollup Concepts for Move

## สารบัญ
- [Scalability Trilemma](#scalability-trilemma)
- [State Channels](#state-channels)
- [Rollup Architecture](#rollup-architecture)
- [Optimistic Rollup Pattern](#optimistic-rollup-pattern)
- [ZK Rollup Fundamentals](#zk-rollup-fundamentals)
- [Move on Rollup](#move-on-rollup)
- [Practical: Off-chain Computation, On-chain Verification](#practical-off-chain-computation-on-chain-verification)

---

## Scalability Trilemma

```
Blockchain Trilemma:
  
       Security
       /\
      /  \
     /    \
    /      \
   /________\
Decentralization  Scalability

Can only optimize 2 of 3:
  Bitcoin: Security + Decentralization (low TPS)
  BSC: Security + Scalability (fewer validators)
  Some L1s: Scalability + Security (centralized)

Solutions:
  Layer 2 (L2): Scale on top of L1
    - Inherit L1 security
    - Much higher throughput
    
  Sharding: Split L1 into parallel chains
    - Complex cross-shard communication
    
  Better L1: Aptos/Sui approach
    - Parallel execution (Block-STM)
    - Object model (Sui)
    
Aptos/Sui Scale Without L2:
  Aptos: Block-STM parallel execution → 100k+ TPS
  Sui: Object-level parallelism → 100k+ TPS
  
But L2 concepts still useful:
  - State channels (off-chain game state)
  - Off-chain computation (complex math)
  - Batching (gas efficiency)
  - Privacy (ZK for hidden state)
```

---

## State Channels

```move
module l2::state_channel {
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::coin;
    use aptos_framework::aptos_coin::AptosCoin;
    
    // ============================================
    // State Channels: Bilateral off-chain state
    // ============================================
    
    // Use case: Payment channels, game state
    // Two parties deposit → transact off-chain → settle on-chain
    
    // Benefits:
    // - Unlimited off-chain txs (no gas per tx)
    // - Instant confirmation (no block time)
    // - Privacy (state not on-chain)
    
    const DISPUTE_PERIOD: u64 = 24 * 3600;  // 1 day
    
    struct Channel has key {
        party_a: address,
        party_b: address,
        
        balance_a: u64,
        balance_b: u64,
        
        // Current state version (counter)
        nonce: u64,
        
        // Channel status
        status: u8,  // 0=open, 1=closing, 2=closed
        
        // Closing state
        close_initiated_by: std::option::Option<address>,
        close_nonce: u64,
        close_balance_a: u64,
        close_balance_b: u64,
        close_deadline: u64,  // Time for counterparty to dispute
    }
    
    struct ChannelRegistry has key {
        channels: SmartTable<vector<u8>, address>,  // channel_id → channel_addr
    }
    
    // ============================================
    // Open channel
    // ============================================
    
    public entry fun open_channel(
        party_a: &signer,
        party_b_addr: address,
        deposit_a: u64,
    ) {
        let a_addr = std::signer::address_of(party_a);
        
        // Lock party_a's deposit
        let deposit = coin::withdraw<AptosCoin>(party_a, deposit_a);
        // Store in channel escrow
        
        // Create channel object
        // (both parties must co-sign in real implementation)
    }
    
    public entry fun join_channel(
        party_b: &signer,
        channel_addr: address,
        deposit_b: u64,
    ) acquires Channel {
        let channel = borrow_global_mut<Channel>(channel_addr);
        let b_addr = std::signer::address_of(party_b);
        
        assert!(channel.party_b == b_addr, 1);
        assert!(channel.balance_b == 0, 2);  // Not yet joined
        
        let deposit = coin::withdraw<AptosCoin>(party_b, deposit_b);
        channel.balance_b = deposit_b;
        // Store deposit
    }
    
    // ============================================
    // Off-chain state update (signed by both)
    // ============================================
    
    // Off-chain: Alice and Bob exchange signed states
    struct ChannelState has drop {
        channel_addr: address,
        nonce: u64,
        balance_a: u64,
        balance_b: u64,
        // Signature by party_a
        sig_a: vector<u8>,
        // Signature by party_b
        sig_b: vector<u8>,
    }
    
    // Verify and apply state update on-chain
    public entry fun cooperative_close(
        party_a: &signer,
        channel_addr: address,
        nonce: u64,
        final_balance_a: u64,
        final_balance_b: u64,
        sig_b: vector<u8>,  // Counterparty's signature on final state
    ) acquires Channel {
        let channel = borrow_global_mut<Channel>(channel_addr);
        let a_addr = std::signer::address_of(party_a);
        
        assert!(channel.party_a == a_addr, 1);
        assert!(nonce >= channel.nonce, 2);
        
        // Verify party_b's signature on the final state
        let state_hash = hash_channel_state(
            channel_addr,
            nonce,
            final_balance_a,
            final_balance_b,
        );
        
        assert!(
            verify_signature(channel.party_b, &state_hash, &sig_b),
            3
        );
        
        // Final balances must sum to total
        assert!(
            final_balance_a + final_balance_b == channel.balance_a + channel.balance_b,
            4
        );
        
        channel.status = 2;  // Closed
        
        // Pay out
        // coin::transfer<AptosCoin>(channel_signer, channel.party_a, final_balance_a);
        // coin::transfer<AptosCoin>(channel_signer, channel.party_b, final_balance_b);
    }
    
    // ============================================
    // Unilateral close (if counterparty unresponsive)
    // ============================================
    
    public entry fun initiate_close(
        party: &signer,
        channel_addr: address,
        nonce: u64,
        proposed_balance_a: u64,
        proposed_balance_b: u64,
    ) acquires Channel {
        let channel = borrow_global_mut<Channel>(channel_addr);
        let party_addr = std::signer::address_of(party);
        
        assert!(channel.status == 0, 1);  // Open
        
        channel.status = 1;  // Closing
        channel.close_initiated_by = std::option::some(party_addr);
        channel.close_nonce = nonce;
        channel.close_balance_a = proposed_balance_a;
        channel.close_balance_b = proposed_balance_b;
        channel.close_deadline = aptos_framework::timestamp::now_seconds() + DISPUTE_PERIOD;
    }
    
    // ============================================
    // Dispute: Submit higher-nonce state
    // ============================================
    
    public entry fun dispute_close(
        challenger: &signer,
        channel_addr: address,
        nonce: u64,           // Must be higher than submitted
        balance_a: u64,
        balance_b: u64,
        sig_other: vector<u8>,  // Other party's signature
    ) acquires Channel {
        let channel = borrow_global_mut<Channel>(channel_addr);
        let now = aptos_framework::timestamp::now_seconds();
        
        assert!(channel.status == 1, 1);  // Closing
        assert!(now < channel.close_deadline, 2);
        assert!(nonce > channel.close_nonce, 3);  // Higher nonce wins!
        
        // Verify the other party signed this state
        let state_hash = hash_channel_state(channel_addr, nonce, balance_a, balance_b);
        let other_party = if (std::signer::address_of(challenger) == channel.party_a) {
            channel.party_b
        } else {
            channel.party_a
        };
        
        assert!(verify_signature(other_party, &state_hash, &sig_other), 4);
        
        // Update to higher nonce state
        channel.close_nonce = nonce;
        channel.close_balance_a = balance_a;
        channel.close_balance_b = balance_b;
    }
    
    // Execute close after dispute period
    public entry fun execute_close(caller: &signer, channel_addr: address) acquires Channel {
        let channel = borrow_global_mut<Channel>(channel_addr);
        let now = aptos_framework::timestamp::now_seconds();
        
        assert!(channel.status == 1, 1);
        assert!(now >= channel.close_deadline, 2);
        
        channel.status = 2;  // Closed
        
        // Pay final balances
        // coin::transfer<AptosCoin>(channel_signer, channel.party_a, channel.close_balance_a);
        // coin::transfer<AptosCoin>(channel_signer, channel.party_b, channel.close_balance_b);
    }
    
    fun hash_channel_state(addr: address, nonce: u64, bal_a: u64, bal_b: u64): vector<u8> {
        let mut preimage = bcs::to_bytes(&addr);
        std::vector::append(&mut preimage, bcs::to_bytes(&nonce));
        std::vector::append(&mut preimage, bcs::to_bytes(&bal_a));
        std::vector::append(&mut preimage, bcs::to_bytes(&bal_b));
        aptos_std::aptos_hash::keccak256(preimage)
    }
    
    fun verify_signature(_signer_addr: address, _message: &vector<u8>, _sig: &vector<u8>): bool { true }
}
```

---

## Off-chain Computation, On-chain Verification

```move
module l2::offchain_compute {
    
    // ============================================
    // Pattern: Complex computation off-chain
    // Verify result on-chain (much cheaper!)
    // ============================================
    
    // Use cases:
    // - Sorting a large dataset
    // - Complex math (ML inference)
    // - Merkle proof verification
    // - ZK proof verification
    
    // ============================================
    // Example 1: Off-chain Sort, On-chain Verify
    // ============================================
    
    // Off-chain: Sort array of 10,000 items
    // On-chain: Verify sorted (O(n) vs O(n log n))
    
    public fun verify_sorted(sorted: &vector<u64>): bool {
        let len = std::vector::length(sorted);
        if (len <= 1) return true;
        
        let mut i = 0u64;
        while (i < len - 1) {
            let a = *std::vector::borrow(sorted, i);
            let b = *std::vector::borrow(sorted, i + 1);
            if (a > b) return false;
            i = i + 1;
        };
        true
    }
    
    // Verify sorted AND same elements as original (permutation check)
    public fun verify_sorted_permutation(
        original: &vector<u64>,
        sorted: &vector<u64>,
    ): bool {
        if (std::vector::length(original) != std::vector::length(sorted)) return false;
        
        // Check sorted order
        if (!verify_sorted(sorted)) return false;
        
        // Check same multiset (sum + product is not sufficient but demonstrates concept)
        // In production: use Merkle trees or polynomial hashing
        let orig_sum = vector_sum(original);
        let sort_sum = vector_sum(sorted);
        
        if (orig_sum != sort_sum) return false;
        
        true
    }
    
    // ============================================
    // Example 2: Merkle Proof Verification
    // ============================================
    
    // Off-chain: Build Merkle tree of 1M items
    // On-chain: Verify single item inclusion (O(log n))
    
    public fun verify_merkle_proof(
        root: vector<u8>,      // Merkle root (on-chain)
        leaf: vector<u8>,      // Item to prove
        proof: vector<vector<u8>>,  // Siblings from leaf to root
        leaf_index: u64,
    ): bool {
        let mut current_hash = aptos_std::aptos_hash::keccak256(leaf);
        let mut index = leaf_index;
        let proof_len = std::vector::length(&proof);
        
        let mut i = 0u64;
        while (i < proof_len) {
            let sibling = std::vector::borrow(&proof, i);
            
            // Hash order: left always first
            current_hash = if (index % 2 == 0) {
                // Current is left child
                let mut combined = current_hash;
                std::vector::append(&mut combined, *sibling);
                aptos_std::aptos_hash::keccak256(combined)
            } else {
                // Current is right child
                let mut combined = *sibling;
                std::vector::append(&mut combined, current_hash);
                aptos_std::aptos_hash::keccak256(combined)
            };
            
            index = index / 2;
            i = i + 1;
        };
        
        current_hash == root
    }
    
    // ============================================
    // Example 3: Batch Settlement with Merkle Root
    // ============================================
    
    struct BatchSettlement has key {
        // Instead of storing 1000s of transfers on-chain
        // Just store the Merkle root of all transfers
        // Receivers prove their transfer via Merkle proof
        
        batch_id: u64,
        merkle_root: vector<u8>,
        total_amount: u64,
        
        // Track claimed transfers
        claimed: aptos_std::smart_table::SmartTable<vector<u8>, bool>,  // leaf_hash → claimed
    }
    
    // Claim a specific transfer from a batch
    public entry fun claim_transfer(
        recipient: &signer,
        settlement_addr: address,
        amount: u64,
        proof: vector<vector<u8>>,
        leaf_index: u64,
    ) acquires BatchSettlement {
        let settlement = borrow_global_mut<BatchSettlement>(settlement_addr);
        let recipient_addr = std::signer::address_of(recipient);
        
        // Build leaf: hash(recipient, amount)
        let mut leaf_data = bcs::to_bytes(&recipient_addr);
        std::vector::append(&mut leaf_data, bcs::to_bytes(&amount));
        let leaf_hash = aptos_std::aptos_hash::keccak256(leaf_data);
        
        // Verify not already claimed
        assert!(
            !aptos_std::smart_table::contains(&settlement.claimed, leaf_hash),
            1
        );
        
        // Verify Merkle proof
        assert!(
            verify_merkle_proof(
                settlement.merkle_root,
                leaf_data,
                proof,
                leaf_index,
            ),
            2
        );
        
        // Mark claimed
        aptos_std::smart_table::add(&mut settlement.claimed, leaf_hash, true);
        
        // Transfer to recipient
        // coin::transfer<AptosCoin>(settlement_signer, recipient_addr, amount);
    }
    
    fun vector_sum(v: &vector<u64>): u128 {
        let mut sum = 0u128;
        let len = std::vector::length(v);
        let mut i = 0u64;
        while (i < len) {
            sum = sum + (*std::vector::borrow(v, i) as u128);
            i = i + 1;
        };
        sum
    }
}
```

---

## ZK Rollup Fundamentals

```
ZK Rollups: Cryptographic Proof of Execution

How ZK Rollup Works:
  1. Users submit txs to rollup operator
  2. Operator batches txs (1000s at once)
  3. Operator computes new state off-chain
  4. Operator generates ZK proof (SNARK/STARK)
     "I correctly executed all these transactions"
  5. Proof posted on-chain (tiny, ~200 bytes)
  6. Anyone verifies proof on-chain in O(1)
  
Properties:
  - Instant finality (once proof verified)
  - High throughput (batch processing)
  - Privacy optional (ZK can hide inputs)
  - Security: L1 enforces correctness
  
ZK Proof Systems:
  SNARK (Groth16): 
    - Trusted setup required
    - Tiny proof (200 bytes)
    - Fast verification
    - Used: Zcash, Loopring
    
  SNARK (PLONK):
    - Universal setup (no per-circuit trust)
    - Larger proof (700 bytes)
    - Used: Aztec, zkSync
    
  STARK:
    - No trusted setup
    - Large proof (100KB+)
    - Fast proving
    - Used: StarkNet, dYdX v4

Move + ZK:
  Sui has zkLogin: use ZK to verify Google auth
  Aptos has crypto libraries for ZK verification
  
  On-chain ZK verifier in Move:
  - Verify Groth16 proofs
  - Verify BLS signatures (threshold)
  - Verify Merkle proofs
```

```move
module l2::zk_verifier {
    
    // ============================================
    // ZK Proof Verification in Move
    // (Using aptos_std crypto libraries)
    // ============================================
    
    // Groth16 Proof structure:
    // - Proof: (π_A, π_B, π_C) = 3 elliptic curve points
    // - Public inputs: x1, x2, ... xn
    // - Verification key: (alpha, beta, gamma, delta, gamma_abc)
    
    // Simplified Groth16 verification:
    // e(π_A, π_B) = e(alpha, beta) * e(L, gamma) * e(π_C, delta)
    // where L = sum(xi * gamma_abc[i])
    
    struct VerificationKey has store {
        alpha_g1: vector<u8>,  // G1 point
        beta_g2: vector<u8>,   // G2 point
        gamma_g2: vector<u8>,  // G2 point
        delta_g2: vector<u8>,  // G2 point
        gamma_abc_g1: vector<vector<u8>>,  // G1 points (one per public input)
    }
    
    struct Proof has drop {
        a: vector<u8>,  // G1 point
        b: vector<u8>,  // G2 point
        c: vector<u8>,  // G1 point
    }
    
    // Verify a Groth16 proof against public inputs
    public fun verify_groth16(
        vk: &VerificationKey,
        proof: &Proof,
        public_inputs: &vector<vector<u8>>,
    ): bool {
        // In production: use aptos_std::crypto_algebra
        // aptos_std::crypto_algebra provides BLS12-381 operations:
        //   - scalar_mul, multi_scalar_mul
        //   - pairing
        //   - add, neg
        
        // Compute L = sum(xi * gamma_abc[i])
        let mut l_point = *std::vector::borrow(&vk.gamma_abc_g1, 0);  // gamma_abc[0]
        
        let n = std::vector::length(public_inputs);
        let mut i = 0u64;
        while (i < n) {
            let xi = std::vector::borrow(public_inputs, i);
            let gamma_abc_i = std::vector::borrow(&vk.gamma_abc_g1, i + 1);
            
            // l_point += xi * gamma_abc_i
            // (elliptic curve scalar multiplication)
            // In production: crypto_algebra::scalar_mul(gamma_abc_i, xi)
            
            i = i + 1;
        };
        
        // Check pairing equation
        // e(proof.a, proof.b) == e(alpha, beta) * e(l_point, gamma) * e(proof.c, delta)
        // All handled by aptos_std::crypto_algebra::pairing
        
        // Simplified: return true for demonstration
        true
    }
    
    // zkLogin-style: verify JWT proof
    // Used in Sui for social login
    public fun verify_jwt_proof(
        // ZK proof that user knows JWT from Google/Apple
        proof: &Proof,
        // Public inputs: ephemeral_key, user_address_salt
        ephemeral_pubkey: vector<u8>,
        address_seed: vector<u8>,
        // JWT header (public info)
        iss: vector<u8>,
        kid: vector<u8>,
    ): bool {
        let public_inputs = vector[ephemeral_pubkey, address_seed, iss, kid];
        // Verify ZK proof...
        true  // Simplified
    }
}
```

---

## สรุป Layer 2 & Rollups

```
L2 Concepts Applied to Move:

1. STATE CHANNELS (Most relevant for Aptos/Sui)
   - Game state channels (instant moves, settle final score)
   - Payment channels (micropayments, no gas per tx)
   - Dispute mechanism: submit higher-nonce state
   
2. BATCH SETTLEMENT WITH MERKLE ROOTS
   - Process 1000s of transfers off-chain
   - Post single Merkle root on-chain
   - Receivers claim with Merkle proofs
   - Used by: Airdrops, token distributions
   
3. OFF-CHAIN COMPUTE, ON-CHAIN VERIFY
   - Sort off-chain → verify on-chain O(n)
   - Compute ML inference off-chain → verify hash
   - Solve optimization → verify solution feasibility
   
4. ZK PROOFS
   - Aptos has crypto_algebra for BLS12-381
   - Groth16 verification implementable
   - zkLogin (Sui): Google/Apple → blockchain auth
   
Aptos/Sui Don't Need Traditional Rollups:
  Their L1 already does 100k+ TPS
  State channels still useful for:
    - Zero gas micropayments
    - Privacy (state not on-chain)
    - Instant finality (no block wait)
  
Most useful L2 pattern on Move:
  Merkle batch settlement for mass airdrops/distributions
  ZK for privacy-preserving applications
```

---

**ก่อนหน้า**: [Part 63 - Options & Derivatives ←](part-63-options-derivatives.md)
**ต่อไป**: [Part 65 - AI & ML On-Chain →](part-65-ai-ml-onchain.md)
