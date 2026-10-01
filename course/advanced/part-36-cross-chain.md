# Part 36: Cross-chain Bridges & Interoperability

## สารบัญ
- [Bridge Architecture](#bridge-architecture)
- [LayerZero on Aptos](#layerzero-on-aptos)
- [Wormhole Integration](#wormhole-integration)
- [Canonical Bridge Pattern](#canonical-bridge-pattern)
- [Cross-chain Messaging](#cross-chain-messaging)
- [ตัวอย่าง: Token Bridge](#ตัวอย่าง-token-bridge)

---

## Bridge Architecture

```
Cross-chain Bridge Types:

1. Lock & Mint (Canonical):
   Chain A: lock(token) → emit event
   Relayer: read event → submit proof
   Chain B: verify proof → mint(wrapped_token)

2. Burn & Mint (Native):
   Chain A: burn(token)
   Relayer: relay message
   Chain B: mint(token)  [shared token supply]

3. Liquidity Pool:
   User deposits Token A on Chain A
   Protocol provides Token B from pool on Chain B
   (no locking, just pool balancing)

Security Considerations:
  - Validator set size (more = more decentralized)
  - Fraud proof window vs optimistic vs ZK proofs
  - Message ordering and replay protection
  - Smart contract bugs (most hacks happen here)
```

---

## LayerZero on Aptos

```move
module bridge::layerzero_oft {
    use std::signer;
    use aptos_framework::coin;
    
    // ============================================
    // Omnichain Fungible Token (OFT) using LayerZero
    // ============================================
    
    // LayerZero Chain IDs
    const CHAIN_ETHEREUM: u16 = 101;
    const CHAIN_APTOS: u16 = 108;
    const CHAIN_ARBITRUM: u16 = 110;
    const CHAIN_POLYGON: u16 = 109;
    const CHAIN_SUI: u16 = 122;
    
    struct OFTConfig has key {
        admin: address,
        endpoint_addr: address,    // LayerZero endpoint contract
        local_decimals: u8,
        shared_decimals: u8,       // common decimals across chains (usually 6)
        
        // Trusted remote configs (chain_id -> remote OFT address)
        trusted_remotes: aptos_std::smart_table::SmartTable<u16, vector<u8>>,
        
        // Rate limiting
        outbound_limit_per_day: u64,
        outbound_today: u64,
        outbound_day_start: u64,
    }
    
    struct SendParam has drop {
        dst_chain_id: u16,
        to_address: vector<u8>,  // bytes32 on EVM
        amount: u64,
        min_amount: u64,         // slippage protection
        extra_options: vector<u8>,
    }
    
    struct LzReceiveParam has drop {
        src_chain_id: u16,
        src_address: vector<u8>,
        nonce: u64,
        payload: vector<u8>,
    }
    
    const E_UNTRUSTED_REMOTE: u64 = 1;
    const E_RATE_LIMIT_EXCEEDED: u64 = 2;
    const E_AMOUNT_TOO_SMALL: u64 = 3;
    const E_NOT_ENDPOINT: u64 = 4;
    
    // ============================================
    // Send tokens cross-chain
    // ============================================
    
    public entry fun send(
        sender: &signer,
        config_addr: address,
        dst_chain_id: u16,
        to_address: vector<u8>,  // recipient on destination chain
        amount: u64,
        native_fee: u64,         // LayerZero fee in native token
    ) acquires OFTConfig {
        use aptos_std::smart_table;
        let config = borrow_global_mut<OFTConfig>(config_addr);
        let sender_addr = signer::address_of(sender);
        
        // Check trusted remote configured
        assert!(smart_table::contains(&config.trusted_remotes, dst_chain_id), E_UNTRUSTED_REMOTE);
        
        // Rate limiting
        check_and_update_rate_limit(config, amount);
        
        // Convert to shared decimals (truncate dust)
        let (ld_amount, sd_amount) = ld2sd(amount, config.local_decimals, config.shared_decimals);
        assert!(sd_amount > 0, E_AMOUNT_TOO_SMALL);
        
        // Burn local tokens (or lock, depending on OFT type)
        // coin::burn or lock in escrow
        
        // Encode payload: (to_address, sd_amount)
        let payload = encode_send_payload(to_address, sd_amount);
        
        // Call LayerZero endpoint to send message
        // layerzero_endpoint::send(dst_chain_id, trusted_remote, payload, native_fee)
        // (actual call depends on LZ SDK)
    }
    
    // ============================================
    // Receive tokens from another chain (called by LZ endpoint)
    // ============================================
    
    public entry fun lz_receive(
        endpoint: &signer,
        config_addr: address,
        src_chain_id: u16,
        src_address: vector<u8>,
        nonce: u64,
        payload: vector<u8>,
    ) acquires OFTConfig {
        use aptos_std::smart_table;
        let config = borrow_global<OFTConfig>(config_addr);
        
        // Verify caller is LayerZero endpoint
        assert!(signer::address_of(endpoint) == config.endpoint_addr, E_NOT_ENDPOINT);
        
        // Verify trusted remote
        assert!(smart_table::contains(&config.trusted_remotes, src_chain_id), E_UNTRUSTED_REMOTE);
        let expected_remote = smart_table::borrow(&config.trusted_remotes, src_chain_id);
        assert!(*expected_remote == src_address, E_UNTRUSTED_REMOTE);
        
        // Decode payload
        let (recipient, sd_amount) = decode_receive_payload(payload);
        
        // Convert to local decimals
        let ld_amount = sd2ld(sd_amount, config.local_decimals, config.shared_decimals);
        
        // Mint or unlock tokens to recipient
        // coin::mint or unlock from escrow
    }
    
    // ============================================
    // Decimal conversion
    // ============================================
    
    fun ld2sd(ld_amount: u64, local_dec: u8, shared_dec: u8): (u64, u64) {
        if (local_dec == shared_dec) return (ld_amount, ld_amount);
        
        if (local_dec > shared_dec) {
            let factor = pow10((local_dec - shared_dec) as u64);
            let sd = ld_amount / factor;
            let ld = sd * factor;  // remove dust
            (ld, sd)
        } else {
            // local has fewer decimals than shared (unusual)
            let factor = pow10((shared_dec - local_dec) as u64);
            (ld_amount, ld_amount * factor)
        }
    }
    
    fun sd2ld(sd_amount: u64, local_dec: u8, shared_dec: u8): u64 {
        if (local_dec == shared_dec) return sd_amount;
        
        if (local_dec > shared_dec) {
            let factor = pow10((local_dec - shared_dec) as u64);
            sd_amount * factor
        } else {
            let factor = pow10((shared_dec - local_dec) as u64);
            sd_amount / factor
        }
    }
    
    fun pow10(n: u64): u64 {
        let mut result = 1u64;
        let mut i = 0;
        while (i < n) { result = result * 10; i = i + 1; };
        result
    }
    
    fun check_and_update_rate_limit(config: &mut OFTConfig, amount: u64) {
        let now = aptos_framework::timestamp::now_seconds();
        let day = now / 86400;
        let today_start = day * 86400;
        
        if (today_start > config.outbound_day_start) {
            config.outbound_today = 0;
            config.outbound_day_start = today_start;
        };
        
        assert!(
            config.outbound_today + amount <= config.outbound_limit_per_day,
            E_RATE_LIMIT_EXCEEDED
        );
        config.outbound_today = config.outbound_today + amount;
    }
    
    fun encode_send_payload(recipient: vector<u8>, amount: u64): vector<u8> {
        let mut payload = std::vector::empty<u8>();
        std::vector::append(&mut payload, recipient);
        // Append amount as big-endian u64
        let mut i = 7u8;
        loop {
            std::vector::push_back(&mut payload, ((amount >> (i * 8)) & 0xFF) as u8);
            if (i == 0) break;
            i = i - 1;
        };
        payload
    }
    
    fun decode_receive_payload(payload: vector<u8>): (vector<u8>, u64) {
        // First 32 bytes: recipient address
        let mut recipient = std::vector::empty<u8>();
        let mut i = 0u64;
        while (i < 32) {
            std::vector::push_back(&mut recipient, *std::vector::borrow(&payload, i));
            i = i + 1;
        };
        
        // Next 8 bytes: amount
        let mut amount = 0u64;
        while (i < 40) {
            amount = (amount << 8) | (*std::vector::borrow(&payload, i) as u64);
            i = i + 1;
        };
        
        (recipient, amount)
    }
}
```

---

## Wormhole Integration

```move
module bridge::wormhole_bridge {
    use std::signer;
    
    // ============================================
    // Wormhole Message Passing
    // Guardian network validates VAAs (Verified Action Approvals)
    // ============================================
    
    struct WormholeConfig has key {
        admin: address,
        wormhole_core: address,      // Wormhole core contract
        token_bridge: address,       // Token bridge contract
        emitter_chain_id: u16,
        emitter_address: vector<u8>,
    }
    
    // VAA = Verified Action Approval (signed by guardians)
    struct VAA has drop {
        version: u8,
        guardian_set_index: u32,
        signatures: vector<vector<u8>>,
        timestamp: u32,
        nonce: u32,
        emitter_chain: u16,
        emitter_address: vector<u8>,
        sequence: u64,
        consistency_level: u8,
        payload: vector<u8>,
    }
    
    // Token Transfer payload format
    struct TokenTransferPayload has drop {
        amount: u64,
        token_address: vector<u8>,
        token_chain: u16,
        to: vector<u8>,
        to_chain: u16,
        fee: u64,
    }
    
    const E_NOT_WORMHOLE: u64 = 1;
    const E_VAA_ALREADY_CONSUMED: u64 = 2;
    const E_INVALID_EMITTER: u64 = 3;
    
    // ============================================
    // Send cross-chain message via Wormhole
    // ============================================
    
    public entry fun publish_message(
        sender: &signer,
        config_addr: address,
        nonce: u32,
        payload: vector<u8>,
        consistency_level: u8,  // 0=immediate, 1=finalized
    ) acquires WormholeConfig {
        let config = borrow_global<WormholeConfig>(config_addr);
        
        // Call Wormhole core to emit message
        // wormhole::publish_message(nonce, payload, consistency_level)
        // Returns sequence number (for tracking)
        
        // Off-chain: Wormhole guardians observe and sign the message
        // After threshold signatures: VAA is available
    }
    
    // ============================================
    // Receive and verify VAA
    // ============================================
    
    public entry fun complete_transfer(
        relayer: &signer,
        config_addr: address,
        vaa_bytes: vector<u8>,
    ) acquires WormholeConfig {
        let config = borrow_global<WormholeConfig>(config_addr);
        
        // Parse and verify VAA
        // wormhole::parse_and_verify_vaa(vaa_bytes)
        
        // Decode token transfer payload
        // let payload = decode_token_transfer(vaa.payload)
        
        // Verify emitter is trusted bridge on source chain
        
        // Release tokens to recipient
        // (mint wrapped or unlock native)
    }
    
    // ============================================
    // Replay Protection
    // ============================================
    
    struct ConsumedVAAs has key {
        vaas: aptos_std::smart_table::SmartTable<vector<u8>, bool>,
    }
    
    fun consume_vaa(
        consumed: &mut ConsumedVAAs,
        vaa_hash: vector<u8>,
    ) {
        use aptos_std::smart_table;
        assert!(!smart_table::contains(&consumed.vaas, vaa_hash), E_VAA_ALREADY_CONSUMED);
        smart_table::add(&mut consumed.vaas, vaa_hash, true);
    }
}
```

---

## Canonical Bridge Pattern

```move
module bridge::canonical_bridge {
    use std::signer;
    use aptos_framework::coin::{Self, Coin, MintCapability, BurnCapability};
    use aptos_framework::event;
    
    // ============================================
    // Lock on source chain, mint on destination
    // (or burn on source, mint on destination for native bridges)
    // ============================================
    
    // Wrapped ETH on Aptos
    struct WETH has key {}
    
    struct WrappedTokenConfig has key {
        mint_cap: MintCapability<WETH>,
        burn_cap: BurnCapability<WETH>,
        bridge_addr: address,
        total_bridged: u64,
        max_bridged: u64,
    }
    
    #[event]
    struct Deposit has drop, store {
        from_chain: u16,
        from_addr: vector<u8>,
        to_addr: address,
        amount: u64,
        nonce: u64,
    }
    
    #[event]
    struct Withdrawal has drop, store {
        to_chain: u16,
        from_addr: address,
        to_addr: vector<u8>,
        amount: u64,
        nonce: u64,
    }
    
    const E_NOT_BRIDGE: u64 = 1;
    const E_EXCEEDS_CAP: u64 = 2;
    const E_ZERO_AMOUNT: u64 = 3;
    
    // ============================================
    // Mint wrapped token (called by bridge)
    // ============================================
    
    public entry fun bridge_mint(
        bridge: &signer,
        config_addr: address,
        recipient: address,
        amount: u64,
        from_chain: u16,
        from_addr: vector<u8>,
        nonce: u64,
    ) acquires WrappedTokenConfig {
        let config = borrow_global_mut<WrappedTokenConfig>(config_addr);
        assert!(signer::address_of(bridge) == config.bridge_addr, E_NOT_BRIDGE);
        assert!(amount > 0, E_ZERO_AMOUNT);
        assert!(config.total_bridged + amount <= config.max_bridged, E_EXCEEDS_CAP);
        
        let coins = coin::mint(amount, &config.mint_cap);
        coin::deposit(recipient, coins);
        config.total_bridged = config.total_bridged + amount;
        
        event::emit(Deposit { from_chain, from_addr, to_addr: recipient, amount, nonce });
    }
    
    // ============================================
    // Burn to withdraw (user initiates)
    // ============================================
    
    public entry fun initiate_withdrawal(
        user: &signer,
        config_addr: address,
        amount: u64,
        to_chain: u16,
        to_addr: vector<u8>,
    ) acquires WrappedTokenConfig {
        let config = borrow_global_mut<WrappedTokenConfig>(config_addr);
        let user_addr = signer::address_of(user);
        
        // Burn user's wrapped tokens
        let to_burn = coin::withdraw<WETH>(user, amount);
        coin::burn(to_burn, &config.burn_cap);
        config.total_bridged = config.total_bridged - amount;
        
        // Nonce for tracking (simplified: use timestamp)
        let nonce = aptos_framework::timestamp::now_microseconds();
        
        event::emit(Withdrawal { to_chain, from_addr: user_addr, to_addr, amount, nonce });
        // Off-chain bridge picks up event and releases on destination
    }
    
    #[view]
    public fun total_bridged(config_addr: address): u64 acquires WrappedTokenConfig {
        borrow_global<WrappedTokenConfig>(config_addr).total_bridged
    }
}
```

---

## Cross-chain Messaging

```move
module bridge::cross_chain_messaging {
    use std::signer;
    use aptos_framework::event;
    
    // ============================================
    // Generic cross-chain message passing
    // for arbitrary data, not just tokens
    // ============================================
    
    struct MessageConfig has key {
        admin: address,
        message_router: address,
        nonce: u64,
        
        // Trusted message senders per chain
        trusted_senders: aptos_std::smart_table::SmartTable<u16, vector<u8>>,
    }
    
    struct ReceivedMessage has store {
        src_chain: u16,
        src_address: vector<u8>,
        nonce: u64,
        payload: vector<u8>,
        processed: bool,
    }
    
    struct MessageInbox has key {
        messages: aptos_std::smart_table::SmartTable<u64, ReceivedMessage>,
    }
    
    #[event]
    struct MessageSent has drop, store {
        nonce: u64,
        dst_chain: u16,
        payload: vector<u8>,
    }
    
    #[event]
    struct MessageReceived has drop, store {
        nonce: u64,
        src_chain: u16,
        payload: vector<u8>,
    }
    
    // ============================================
    // Send message
    // ============================================
    
    public entry fun send_message(
        sender: &signer,
        config_addr: address,
        dst_chain: u16,
        payload: vector<u8>,
    ) acquires MessageConfig {
        let config = borrow_global_mut<MessageConfig>(config_addr);
        config.nonce = config.nonce + 1;
        let nonce = config.nonce;
        
        event::emit(MessageSent { nonce, dst_chain, payload });
        // Off-chain relayer picks this up and delivers to dst_chain
    }
    
    // ============================================
    // Receive message (called by relayer with proof)
    // ============================================
    
    public entry fun receive_message(
        relayer: &signer,
        config_addr: address,
        inbox_addr: address,
        src_chain: u16,
        src_address: vector<u8>,
        nonce: u64,
        payload: vector<u8>,
        proof: vector<u8>,  // Merkle proof or guardian signatures
    ) acquires MessageConfig, MessageInbox {
        use aptos_std::smart_table;
        let config = borrow_global<MessageConfig>(config_addr);
        
        // Verify sender is trusted
        assert!(smart_table::contains(&config.trusted_senders, src_chain), 1);
        let expected = smart_table::borrow(&config.trusted_senders, src_chain);
        assert!(*expected == src_address, 2);
        
        // Verify proof (in production: verify against bridge contract)
        // verify_proof(proof, src_chain, nonce, payload);
        
        // Store message
        let inbox = borrow_global_mut<MessageInbox>(inbox_addr);
        smart_table::add(&mut inbox.messages, nonce, ReceivedMessage {
            src_chain,
            src_address,
            nonce,
            payload,
            processed: false,
        });
        
        event::emit(MessageReceived { nonce, src_chain, payload });
    }
    
    // ============================================
    // Process received message (application layer)
    // ============================================
    
    public entry fun process_message(
        processor: &signer,
        inbox_addr: address,
        nonce: u64,
    ) acquires MessageInbox {
        use aptos_std::smart_table;
        let inbox = borrow_global_mut<MessageInbox>(inbox_addr);
        let msg = smart_table::borrow_mut(&mut inbox.messages, nonce);
        assert!(!msg.processed, 3);
        
        // Process based on payload type
        let payload = &msg.payload;
        let msg_type = *std::vector::borrow(payload, 0);
        
        if (msg_type == 1) {
            // Token mint message
        } else if (msg_type == 2) {
            // Governance message
        } else if (msg_type == 3) {
            // Arbitrary call
        };
        
        msg.processed = true;
    }
}
```

---

## สรุป Cross-chain Patterns

| Bridge Type | Security | Speed | Decentralization |
|------------|---------|-------|-----------------|
| Multisig | Low | Fast | Low |
| Optimistic | Medium | Slow (7d) | Medium |
| ZK Proof | High | Medium | High |
| Light Client | Highest | Slow | Very High |
| LayerZero | Medium | Fast | Medium |
| Wormhole | Medium | Medium | Medium |

**Security Checklist:**
1. Replay protection (consume VAA/message nonce)
2. Rate limiting (max per day/tx)
3. Trusted remote validation
4. Circuit breaker on large amounts
5. Timelock on bridge admin operations
6. Multi-sig for guardian/validator set
7. Audit: bridge contracts are highest-value targets

---

**ก่อนหน้า**: [Part 35 - Advanced DeFi ←](part-35-advanced-defi.md)
**ต่อไป**: [Part 37 - Security & MEV Protection →](part-37-security-mev.md)
