# Part 54: Cross-Chain DeFi Architecture

## สารบัญ
- [Cross-Chain Landscape](#cross-chain-landscape)
- [Bridge Security Models](#bridge-security-models)
- [Wormhole Integration](#wormhole-integration)
- [LayerZero OFT V2 Advanced](#layerzero-oft-v2-advanced)
- [Intent-Based Cross-Chain Swaps](#intent-based-cross-chain-swaps)
- [ตัวอย่าง: Unified Liquidity Protocol](#ตัวอย่าง-unified-liquidity-protocol)

---

## Cross-Chain Landscape

```
Cross-Chain Ecosystem (2024-2025):

Bridging Methods:
  1. Lock & Mint: Lock on chain A, mint wrapper on chain B
     - Wormhole: "Wormhole-wrapped" tokens
     - Risk: Bridge hack = wrapped tokens worthless
     
  2. Burn & Mint (OFT): Burn on A, native mint on B
     - LayerZero OFT: single canonical token
     - Circle CCTP: native USDC on each chain
     - More secure: no wrapped tokens
     
  3. Atomic Swaps (HTLCs): Hash-time-locked contracts
     - No trusted bridge
     - Slow, requires matching parties
     
  4. Intent-based: User states desired output, solver executes
     - Across Protocol, deBridge
     - Fast: solver fronts liquidity, reimbursed later

Aptos Cross-Chain Bridges:
  - LayerZero: deployed on Aptos (OFT for tokens)
  - Wormhole: deployed on Aptos (VAA-based)
  - Stargate: LayerZero-based DEX bridge
  
Sui Cross-Chain:
  - Wormhole: major bridge on Sui
  - LayerZero: OFT for tokens
  - Celer: cBridge
  
Security Spectrum:
  Most Secure:                          Least Secure:
  Native → CCTP → Trusted Committee → MPC → PoA → Multisig
  
Bridge Hacks (historical lessons):
  - Ronin: $625M (trusted multisig compromised)
  - Wormhole: $320M (Solana signature verification bug)
  - Nomad: $190M (incorrect root validation)
  - Horizon: $100M (trusted multisig)
  
Lesson: Use established bridges with multiple audits
```

---

## Bridge Security Models

```move
module xchain::security_models {
    
    // ============================================
    // Security Pattern 1: Message Verification
    // ============================================
    
    // All cross-chain messages need cryptographic proof:
    // - VAA (Wormhole): signed by Guardian network
    // - DVN (LayerZero): signed by Decentralized Verifier Network  
    // - Header (IBC): Tendermint light client proof
    
    struct VerifiedMessage has drop {
        source_chain: u16,
        source_address: vector<u8>,
        sequence: u64,
        payload: vector<u8>,
        verified: bool,  // Only true if sig check passed
    }
    
    // Pattern: Never process unverified messages
    public fun process_cross_chain_message(
        message: VerifiedMessage,
        expected_source: vector<u8>,
        expected_chain: u16,
    ) {
        assert!(message.verified, 1);  // Sig verified
        assert!(message.source_chain == expected_chain, 2);  // Right chain
        assert!(message.source_address == expected_source, 3);  // Right sender
        assert!(!is_replayed(message.sequence), 4);  // Not replayed
        
        mark_processed(message.sequence);
        
        // Process payload
    }
    
    // ============================================
    // Security Pattern 2: Rate Limiting
    // ============================================
    
    struct BridgeRateLimit has key {
        // Per-hour limit (anti-drain attack)
        hourly_limit: u64,
        // Rolling window tracking
        window_amounts: vector<u64>,  // Last 24 hourly buckets
        window_start: u64,
        
        // Emergency: if >X% outflow in 24h → pause
        emergency_threshold_bps: u64,
        total_locked: u64,
    }
    
    public fun check_and_update_rate_limit(
        rl: &mut BridgeRateLimit,
        amount: u64,
    ) {
        let current_hour = aptos_framework::timestamp::now_seconds() / 3600;
        let bucket = current_hour % 24;
        
        // Sum last 24 hourly buckets
        let hourly_total = *std::vector::borrow(&rl.window_amounts, bucket);
        assert!(hourly_total + amount <= rl.hourly_limit, 1);
        
        // Emergency check: if >10% outflow in 24h → pause
        let total_24h: u64 = {
            let mut sum = 0u64;
            let mut i = 0u64;
            while (i < 24) {
                sum = sum + *std::vector::borrow(&rl.window_amounts, i);
                i = i + 1;
            };
            sum
        };
        
        let outflow_bps = total_24h * 10_000 / rl.total_locked;
        if (outflow_bps > rl.emergency_threshold_bps) {
            // Emit emergency event, pause bridge
            // (actual pause logic here)
        };
        
        // Update bucket
        *std::vector::borrow_mut(&mut rl.window_amounts, bucket) 
            = hourly_total + amount;
    }
    
    // ============================================
    // Security Pattern 3: Delayed Withdrawals
    // ============================================
    
    // Large withdrawals delayed for guardian review
    
    const LARGE_WITHDRAWAL_THRESHOLD: u64 = 100_000_000_000;  // $100k in microUSD
    const DELAY_PERIOD: u64 = 24 * 3600;  // 24 hour delay
    
    struct PendingWithdrawal has key {
        amount: u64,
        recipient: address,
        source_chain: u16,
        available_at: u64,
        executed: bool,
    }
    
    public fun queue_large_withdrawal(
        amount: u64,
        recipient: address,
        source_chain: u16,
        ctx: &signer,
    ) {
        let available_at = if (amount >= LARGE_WITHDRAWAL_THRESHOLD) {
            aptos_framework::timestamp::now_seconds() + DELAY_PERIOD
        } else {
            aptos_framework::timestamp::now_seconds()  // Immediate
        };
        
        move_to(ctx, PendingWithdrawal {
            amount,
            recipient,
            source_chain,
            available_at,
            executed: false,
        });
    }
    
    fun is_replayed(_seq: u64): bool { false }
    fun mark_processed(_seq: u64) {}
}
```

---

## Wormhole Integration

```move
module xchain::wormhole_v2 {
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // Advanced Wormhole Integration
    // ============================================
    
    // Wormhole VAA (Verified Action Approval):
    // - Signed by 19 of 19 Guardians (or 13/19 threshold)
    // - Contains: source_chain, source_address, sequence, payload
    // - Unique per message
    
    struct WormholeState has key {
        guardian_set_index: u32,
        // Note: actual guardian verification handled by Wormhole framework
        
        // Message tracking
        processed_vaas: SmartTable<vector<u8>, bool>,  // VAA hash → processed
        
        // Token registry: wormhole_address → local_token_type
        token_registry: SmartTable<vector<u8>, address>,
        
        // Chain-specific settings
        finality_threshold: u8,  // How many confirmations before accepting
    }
    
    struct TokenTransfer has drop {
        amount: u64,
        token_address: vector<u8>,   // On source chain
        token_chain: u16,
        recipient: vector<u8>,        // On Aptos (32 bytes = address)
        recipient_chain: u16,
        fee: u64,
    }
    
    // ============================================
    // Receive cross-chain token transfer
    // ============================================
    
    public entry fun receive_transfer(
        relayer: &signer,
        vaa_bytes: vector<u8>,
        wormhole_addr: address,
    ) acquires WormholeState {
        let state = borrow_global_mut<WormholeState>(wormhole_addr);
        
        // Parse and verify VAA
        let vaa_hash = aptos_std::aptos_hash::keccak256(vaa_bytes);
        
        // Check not already processed (replay protection)
        assert!(
            !smart_table::contains(&state.processed_vaas, vaa_hash),
            1
        );
        
        // Verify VAA signatures (simplified - Wormhole framework handles this)
        let transfer = parse_transfer_payload(&vaa_bytes);
        
        // Verify source
        assert!(transfer.recipient_chain == 22, 2);  // Aptos chain ID = 22
        
        // Mark as processed
        smart_table::add(&mut state.processed_vaas, vaa_hash, true);
        
        // Get local token
        let token_key = encode_token_key(transfer.token_chain, &transfer.token_address);
        assert!(smart_table::contains(&state.token_registry, token_key), 3);
        
        // Decode recipient address (32 bytes → address)
        let recipient = bytes32_to_address(&transfer.recipient);
        
        // Mint/release tokens to recipient
        // (actual token handling here)
    }
    
    // ============================================
    // Send cross-chain token transfer
    // ============================================
    
    public entry fun send_transfer<Token>(
        sender: &signer,
        wormhole_addr: address,
        amount: u64,
        target_chain: u16,
        recipient: vector<u8>,  // 32-byte address on target chain
        wormhole_fee: u64,       // Pay guardian network
    ) acquires WormholeState {
        let sender_addr = std::signer::address_of(sender);
        
        // Lock tokens
        let coins = aptos_framework::coin::withdraw<Token>(sender, amount);
        // Store in wormhole escrow or burn if native token
        
        // Publish message to Wormhole (off-chain guardians sign it)
        // In production: call wormhole::publish_message
        let payload = encode_transfer_payload(
            amount,
            get_token_address<Token>(),
            22,           // Aptos = source chain 22
            recipient,
            target_chain,
            0,            // fee
        );
        
        // Emit event (Wormhole relayer picks it up)
        // wormhole_core::publish_message(payload, wormhole_fee);
    }
    
    // ============================================
    // Token registration
    // ============================================
    
    // Register a token from another chain with its local equivalent
    public entry fun register_token(
        admin: &signer,
        wormhole_addr: address,
        source_chain: u16,
        source_address: vector<u8>,
        local_type_hash: address,  // Simplified: type info as address
    ) acquires WormholeState {
        let state = borrow_global_mut<WormholeState>(wormhole_addr);
        let key = encode_token_key(source_chain, &source_address);
        smart_table::add(&mut state.token_registry, key, local_type_hash);
    }
    
    fun parse_transfer_payload(_vaa: &vector<u8>): TokenTransfer {
        TokenTransfer {
            amount: 0,
            token_address: std::vector::empty(),
            token_chain: 0,
            recipient: std::vector::empty(),
            recipient_chain: 0,
            fee: 0,
        }
    }
    
    fun encode_transfer_payload(
        _amount: u64, _token: vector<u8>, _src: u16,
        _recipient: vector<u8>, _dst: u16, _fee: u64
    ): vector<u8> { std::vector::empty() }
    
    fun encode_token_key(chain: u16, addr: &vector<u8>): vector<u8> {
        let mut key = std::vector::empty<u8>();
        std::vector::push_back(&mut key, (chain >> 8) as u8);
        std::vector::push_back(&mut key, (chain & 0xFF) as u8);
        std::vector::append(&mut key, *addr);
        key
    }
    
    fun bytes32_to_address(bytes: &vector<u8>): address { @0x0 }
    fun get_token_address<Token>(): vector<u8> { std::vector::empty() }
}
```

---

## LayerZero OFT V2 Advanced

```move
module xchain::lz_oft_v2 {
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // LayerZero OFT V2: Omnichain Fungible Token
    // ============================================
    
    // OFT (Omnichain Fungible Token):
    // - Single token exists across multiple chains
    // - Burn on source, mint on destination
    // - No wrapped tokens
    
    struct OFTConfig has key {
        // Endpoint address (LayerZero endpoint contract)
        endpoint_addr: address,
        
        // Trusted remotes: chain_id → trusted source
        peers: SmartTable<u32, vector<u8>>,
        
        // Rate limiting
        rate_limiter: RateLimiter,
        
        // DVN (Decentralized Verifier Network) config
        required_dvns: vector<address>,
        optional_dvns: vector<address>,
        confirmations: u64,
        
        // Token info
        token_decimals: u8,
        shared_decimals: u8,  // Decimals used in cross-chain messages
    }
    
    struct RateLimiter has copy, drop, store {
        limit: u64,
        window: u64,
        current_amount: u64,
        window_start: u64,
    }
    
    // Shared decimal conversion (prevent precision loss between chains)
    // Different chains may have different decimals for same token
    // Shared decimals = minimum precision across all chains
    
    // local → shared (before sending)
    public fun ld2sd(local_amount: u64, local_dec: u8, shared_dec: u8): u64 {
        if (local_dec <= shared_dec) {
            local_amount * pow10((shared_dec - local_dec) as u64)
        } else {
            local_amount / pow10((local_dec - shared_dec) as u64)
        }
    }
    
    // shared → local (after receiving)
    public fun sd2ld(shared_amount: u64, local_dec: u8, shared_dec: u8): u64 {
        if (local_dec >= shared_dec) {
            shared_amount * pow10((local_dec - shared_dec) as u64)
        } else {
            shared_amount / pow10((shared_dec - local_dec) as u64)
        }
    }
    
    fun pow10(exp: u64): u64 {
        let mut result = 1u64;
        let mut i = 0u64;
        while (i < exp) {
            result = result * 10;
            i = i + 1;
        };
        result
    }
    
    // ============================================
    // Send OFT tokens cross-chain
    // ============================================
    
    public entry fun send_oft<Token>(
        sender: &signer,
        oft_config_addr: address,
        dst_chain_id: u32,
        dst_recipient: vector<u8>,  // Recipient on destination chain
        amount_ld: u64,             // Amount in local decimals
        min_amount_ld: u64,         // Min received (slippage protection)
        extra_options: vector<u8>,  // Gas options for destination tx
    ) acquires OFTConfig {
        let config = borrow_global_mut<OFTConfig>(oft_config_addr);
        
        // Verify destination is trusted
        assert!(smart_table::contains(&config.peers, dst_chain_id), 1);
        
        // Check rate limit
        check_rate_limit(&mut config.rate_limiter, amount_ld);
        
        // Convert to shared decimals
        let amount_sd = ld2sd(amount_ld, config.token_decimals, config.shared_decimals);
        let min_amount_sd = ld2sd(min_amount_ld, config.token_decimals, config.shared_decimals);
        
        // Dust removal: convert back to get actual amount (removes dust)
        let amount_received_ld = sd2ld(amount_sd, config.token_decimals, config.shared_decimals);
        
        // Burn tokens from sender
        let coins = aptos_framework::coin::withdraw<Token>(
            sender, 
            amount_received_ld
        );
        // In OFT: burn (no escrow)
        // aptos_framework::coin::burn(coins, &burn_cap);
        
        // Build LayerZero message
        let msg = build_oft_message(
            dst_recipient,
            amount_sd,
            min_amount_sd,
        );
        
        // Send via LayerZero endpoint
        // lz_endpoint::send(config.endpoint_addr, dst_chain_id, msg, extra_options);
        
        aptos_framework::coin::destroy_zero(
            aptos_framework::coin::extract(&mut coins, 0)
        );
    }
    
    // ============================================
    // Receive OFT tokens from cross-chain
    // ============================================
    
    // Called by LayerZero endpoint on delivery
    public entry fun lz_receive(
        endpoint: &signer,  // Must be LZ endpoint
        oft_config_addr: address,
        src_chain_id: u32,
        src_address: vector<u8>,
        nonce: u64,
        payload: vector<u8>,
    ) acquires OFTConfig {
        let config = borrow_global<OFTConfig>(oft_config_addr);
        
        // Verify caller is LayerZero endpoint
        assert!(
            std::signer::address_of(endpoint) == config.endpoint_addr,
            2
        );
        
        // Verify source is trusted peer
        let trusted_remote = smart_table::borrow(&config.peers, src_chain_id);
        assert!(*trusted_remote == src_address, 3);
        
        // Parse OFT message
        let (recipient, amount_sd, min_amount_sd) = parse_oft_message(&payload);
        
        // Convert from shared to local decimals
        let amount_ld = sd2ld(amount_sd, config.token_decimals, config.shared_decimals);
        
        assert!(amount_ld >= sd2ld(min_amount_sd, config.token_decimals, config.shared_decimals), 4);
        
        // Mint tokens to recipient
        // coin::mint(amount_ld, &mint_cap) → deposit to recipient
        
        let recipient_addr = bytes_to_address(&recipient);
        // coin::deposit<Token>(recipient_addr, minted_coins);
    }
    
    fun check_rate_limit(rl: &mut RateLimiter, amount: u64) {
        let now = aptos_framework::timestamp::now_seconds();
        if (now >= rl.window_start + rl.window) {
            rl.current_amount = 0;
            rl.window_start = now;
        };
        rl.current_amount = rl.current_amount + amount;
        assert!(rl.current_amount <= rl.limit, 5);
    }
    
    fun build_oft_message(_recipient: vector<u8>, _amount: u64, _min: u64): vector<u8> {
        std::vector::empty()
    }
    
    fun parse_oft_message(_payload: &vector<u8>): (vector<u8>, u64, u64) {
        (std::vector::empty(), 0, 0)
    }
    
    fun bytes_to_address(_bytes: &vector<u8>): address { @0x0 }
}
```

---

## Intent-Based Cross-Chain Swaps

```
Intent-Based Bridging:
  User expresses: "I want 100 USDC on Sui, starting with APT on Aptos"
  
  Traditional: User bridges APT→USDC, then sends USDC
  Intent-based: Solver network finds optimal path
  
  Flow:
  1. User signs intent (not transaction)
  2. Solver network sees intent
  3. Best solver:
     - Fronts 100 USDC on Sui to user (immediate!)
     - Collects APT from user on Aptos
     - Bridges APT back to their Sui balance
  4. User gets funds in seconds (not waiting for bridge)
  
  Benefits:
  - Fast: seconds (not bridge confirmation time)
  - Optimal routing (solver finds best path)
  - No slippage for user (solver takes risk)
  
  Protocols: Across Protocol, deBridge, Stargate
  
Intent Structure:
```

```move
module xchain::intents {
    
    struct CrossChainIntent has drop {
        // What user wants to sell (source)
        src_chain: u16,
        src_token: vector<u8>,
        src_amount: u64,
        
        // What user wants to receive (destination)
        dst_chain: u16,
        dst_token: vector<u8>,
        min_dst_amount: u64,
        dst_recipient: address,
        
        // Constraints
        deadline: u64,
        
        // Signature
        nonce: u64,
        signature: vector<u8>,
    }
    
    struct IntentRegistry has key {
        pending_intents: aptos_std::smart_table::SmartTable<vector<u8>, CrossChainIntent>,
        filled_intents: aptos_std::smart_table::SmartTable<vector<u8>, bool>,
    }
    
    // User signs and submits intent
    public entry fun submit_intent(
        user: &signer,
        registry_addr: address,
        intent: CrossChainIntent,
    ) acquires IntentRegistry {
        let registry = borrow_global_mut<IntentRegistry>(registry_addr);
        
        // Verify signature
        // verify_intent_signature(user, &intent);
        
        // Verify deadline
        assert!(intent.deadline > aptos_framework::timestamp::now_seconds(), 1);
        
        // Lock source tokens
        // coin::withdraw<SrcToken>(user, intent.src_amount) → store in registry
        
        let intent_hash = hash_intent(&intent);
        aptos_std::smart_table::add(&mut registry.pending_intents, intent_hash, intent);
    }
    
    // Solver fills intent (after delivering dst tokens on target chain)
    // Proof of delivery comes via cross-chain message
    public entry fun fill_intent(
        solver: &signer,
        registry_addr: address,
        intent_hash: vector<u8>,
        fill_proof: vector<u8>,  // Proof of delivery on dst chain
    ) acquires IntentRegistry {
        let registry = borrow_global_mut<IntentRegistry>(registry_addr);
        
        assert!(
            aptos_std::smart_table::contains(&registry.pending_intents, intent_hash),
            2
        );
        assert!(
            !aptos_std::smart_table::contains(&registry.filled_intents, intent_hash),
            3
        );
        
        // Verify fill proof (Wormhole VAA or other cross-chain proof)
        // verify_fill_proof(&fill_proof, &intent);
        
        // Mark intent as filled
        aptos_std::smart_table::add(&mut registry.filled_intents, intent_hash, true);
        
        // Release source tokens to solver
        let intent = aptos_std::smart_table::remove(&mut registry.pending_intents, intent_hash);
        // coin::transfer<SrcToken>(solver, intent.src_amount);
    }
    
    fun hash_intent(_intent: &CrossChainIntent): vector<u8> {
        std::vector::empty()
    }
}
```

---

## ตัวอย่าง: Unified Liquidity Protocol

```move
module xchain::unified_liquidity {
    
    // ============================================
    // Unified Liquidity: Same LP pool across chains
    // ============================================
    
    // Concept: LPs deposit on Aptos, but liquidity is accessible from Sui and EVM chains
    // Cross-chain message updates utilization rates
    
    struct UnifiedPool has key {
        // Local reserves
        local_balance: u64,
        
        // Remote reserves (reported by cross-chain messages)
        remote_balances: aptos_std::smart_table::SmartTable<u16, u64>,  // chain → balance
        
        // Total utilization across all chains
        total_utilization: u64,
        
        // Interest rate model
        base_rate: u64,      // Minimum rate
        slope1: u64,         // Rate increase below optimal utilization
        slope2: u64,         // Rate increase above optimal utilization
        optimal_utilization: u64,
        
        // Sync tracking
        last_sync: aptos_std::smart_table::SmartTable<u16, u64>,
    }
    
    // Calculate interest rate based on cross-chain utilization
    public fun get_borrow_rate(pool: &UnifiedPool): u64 {
        let total_deposits = get_total_deposits(pool);
        
        if (total_deposits == 0) return pool.base_rate;
        
        let utilization = pool.total_utilization * 10_000 / total_deposits;
        
        if (utilization <= pool.optimal_utilization) {
            // Below optimal: base_rate + slope1 * utilization / optimal
            pool.base_rate + pool.slope1 * utilization / pool.optimal_utilization
        } else {
            // Above optimal: sharp rate increase
            let excess = utilization - pool.optimal_utilization;
            let max_excess = 10_000 - pool.optimal_utilization;
            
            pool.base_rate + pool.slope1 + pool.slope2 * excess / max_excess
        }
    }
    
    // Update remote chain balance via bridge message
    public entry fun sync_remote_balance(
        relayer: &signer,
        pool_addr: address,
        source_chain: u16,
        new_balance: u64,
        timestamp_: u64,
        proof: vector<u8>,  // Cross-chain message proof
    ) acquires UnifiedPool {
        let pool = borrow_global_mut<UnifiedPool>(pool_addr);
        
        // Verify proof (simplified)
        // verify_cross_chain_proof(source_chain, proof);
        
        let old_balance = if (aptos_std::smart_table::contains(&pool.remote_balances, source_chain)) {
            *aptos_std::smart_table::borrow(&pool.remote_balances, source_chain)
        } else { 0 };
        
        // Update total deposits
        if (new_balance >= old_balance) {
            pool.total_utilization = pool.total_utilization + (new_balance - old_balance);
        } else {
            pool.total_utilization = pool.total_utilization - (old_balance - new_balance);
        };
        
        aptos_std::smart_table::upsert(&mut pool.remote_balances, source_chain, new_balance);
        aptos_std::smart_table::upsert(&mut pool.last_sync, source_chain, timestamp_);
    }
    
    fun get_total_deposits(pool: &UnifiedPool): u64 {
        // Sum local + all remote balances
        let mut total = pool.local_balance;
        // Iterate remote_balances (simplified)
        total
    }
}
```

---

## สรุป Cross-Chain DeFi

```
Cross-Chain Design Principles:

1. SECURITY FIRST
   - Use battle-tested bridges (Wormhole, LayerZero)
   - Multiple audits required
   - Rate limiting prevents drain attacks
   - Delay large withdrawals
   - Emergency pause mechanism

2. MESSAGE RELIABILITY
   - Every message needs cryptographic proof
   - Replay protection (sequence numbers)
   - Source verification (trusted remote list)
   - Timeout handling (if message lost)

3. DECIMAL PRECISION
   - Different chains have different decimal conventions
   - Shared decimals prevent precision loss
   - Always test boundary cases

4. UX OPTIMIZATION
   - Optimistic fast paths (accept risk for speed)
   - Intent-based for complex operations
   - Aggregate liquidity across chains

5. ECONOMIC SECURITY
   - Bridge validators have economic stake
   - Slashing for malicious behavior
   - Insurance pools for hacks

Top Cross-Chain Protocols on Aptos/Sui:
  Bridges: Wormhole, LayerZero, Celer
  DEX: Stargate (cross-chain swaps)
  Lending: Cross-chain collateral protocols
  Intent: Across Protocol (coming to Aptos)

The future: chains become invisible to users
  User just says "I want X on chain Y"
  Solvers and bridges handle everything
```

---

**ก่อนหน้า**: [Part 53 - Stablecoin Design ←](part-53-stablecoin-design.md)
**ต่อไป**: [Part 55 - NFT Infrastructure & Royalties →](part-55-nft-infrastructure.md)
