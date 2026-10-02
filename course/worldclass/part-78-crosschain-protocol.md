# Part 78: Cross-Chain Protocol Design

## สารบัญ
- [Cross-Chain Architecture Overview](#cross-chain-architecture-overview)
- [Bridge Security Models](#bridge-security-models)
- [Wormhole Integration on Aptos](#wormhole-integration-on-aptos)
- [Message Passing Protocol](#message-passing-protocol)
- [Cross-Chain Liquidity Routing](#cross-chain-liquidity-routing)
- [TypeScript SDK for Cross-Chain](#typescript-sdk-for-cross-chain)

---

## Cross-Chain Architecture Overview

```
CROSS-CHAIN BRIDGE TAXONOMY

1. LOCK & MINT (Custodial)
   Source Chain: Lock native token in vault
   Bridge:       Relay proof to destination
   Dest Chain:   Mint wrapped token
   
   Risk: Central vault = single point of failure
   Examples: Wormhole, LayerZero, Axelar
   
2. BURN & MINT (Non-Custodial)
   Source Chain: Burn canonical token
   Bridge:       Verify burn event
   Dest Chain:   Mint canonical token
   
   Risk: Requires canonical token on both chains
   Examples: Circle CCTP (USDC), LayerZero OFT
   
3. ATOMIC SWAP (Trustless)
   Hash-Time Locked Contracts (HTLC):
   1. Alice locks funds with hash(secret) on chain A
   2. Bob locks funds with same hash on chain B
   3. Alice reveals secret → claims Bob's funds
   4. Bob uses revealed secret → claims Alice's funds
   
   Risk: Counterparty may not complete swap (funds temporarily locked)
   
4. LIQUIDITY NETWORK
   Chains have pools of assets
   User deposits A → Protocol moves B from destination pool
   Relayers earn fees for maintaining liquidity
   
   Examples: Stargate, Hop Protocol
   Risk: Liquidity pool depletion

SECURITY PROPERTIES TO ANALYZE
  Liveness:     Can bridge complete even if some validators offline?
  Safety:       Can bridge produce incorrect output?
  Censorship:   Can validators block specific users?
  
KNOWN BRIDGE VULNERABILITIES
  2022: Wormhole hack - $320M (signature verification bug)
  2022: Ronin Bridge - $625M (compromised validators)
  2022: Nomad Bridge - $190M (initialization bug)
  2022: Harmony Bridge - $100M (compromised private keys)
  
  Common pattern: Trust placed in small validator set
  Mitigations: Large validator sets, multi-sig, optimistic verification
```

---

## Bridge Security Models

```move
// ============================================
// MOVE: Wormhole-style Guardian Verification
// Guardians sign VAAs (Verified Action Approvals)
// ============================================

module bridge::guardian_set {
    use aptos_std::secp256k1;
    use aptos_std::aptos_hash;
    
    // A VAA (Verified Action Approval) from Wormhole
    struct VAA has drop {
        version: u8,
        guardian_set_index: u32,
        signatures: vector<GuardianSignature>,
        timestamp: u32,
        nonce: u32,
        emitter_chain: u16,
        emitter_address: vector<u8>,  // 32 bytes
        sequence: u64,
        consistency_level: u8,
        payload: vector<u8>,
    }
    
    struct GuardianSignature has drop {
        guardian_index: u8,
        r: vector<u8>,      // 32 bytes
        s: vector<u8>,      // 32 bytes
        v: u8,
    }
    
    struct GuardianSet has key {
        index: u32,
        keys: vector<vector<u8>>,   // secp256k1 public keys
        expiry: u32,                // 0 = never expires
    }
    
    // Verify a VAA against current guardian set
    // Requires 2/3 quorum of guardians to sign
    public fun verify_vaa(
        guardian_set: &GuardianSet,
        vaa: &VAA,
    ): bool {
        let n_guardians = std::vector::length(&guardian_set.keys);
        let required = (n_guardians * 2 / 3) + 1;  // 2/3 + 1 quorum
        
        // Hash the VAA body (everything after signatures)
        let body = encode_vaa_body(vaa);
        let hash = aptos_hash::keccak256(body);
        let hash2 = aptos_hash::keccak256(hash);  // Double hash like Ethereum
        
        let mut valid_sigs = 0u64;
        let n_sigs = std::vector::length(&vaa.signatures);
        
        let mut i = 0u64;
        while (i < n_sigs) {
            let sig = std::vector::borrow(&vaa.signatures, i);
            
            // Guardian index must be valid
            if ((sig.guardian_index as u64) >= n_guardians) {
                i = i + 1;
                continue
            };
            
            let guardian_key = std::vector::borrow(&guardian_set.keys, sig.guardian_index as u64);
            
            // Verify ECDSA signature
            let signature = secp256k1::ecdsa_signature_from_bytes(
                concat_bytes(&sig.r, &sig.s)
            );
            
            if (secp256k1::ecdsa_recover(hash2, sig.v, &signature) == *guardian_key) {
                valid_sigs = valid_sigs + 1;
            };
            
            i = i + 1;
        };
        
        valid_sigs >= required
    }
    
    fun encode_vaa_body(vaa: &VAA): vector<u8> {
        // Encode: timestamp + nonce + emitter_chain + emitter_address + sequence + consistency_level + payload
        let mut body = std::vector::empty<u8>();
        // ... encoding logic
        body
    }
    
    fun concat_bytes(a: &vector<u8>, b: &vector<u8>): vector<u8> {
        let mut result = *a;
        std::vector::append(&mut result, *b);
        result
    }
}

// ============================================
// Optimistic Bridge with Challenge Period
// Lower cost, longer finality
// ============================================

module bridge::optimistic_bridge {
    use aptos_framework::timestamp;
    
    const CHALLENGE_PERIOD_SECS: u64 = 1800;  // 30 minutes
    
    struct PendingMessage has key {
        id: u64,
        source_chain: u16,
        message_hash: vector<u8>,  // Hash of message content
        submitted_at: u64,
        challenged: bool,
        executed: bool,
    }
    
    struct BridgeState has key {
        next_message_id: u64,
        watchers: vector<address>,  // Fraud watchers
        messages: aptos_std::table::Table<u64, PendingMessage>,
    }
    
    // Submit a bridged message optimistically
    public fun submit_message(
        submitter: &signer,
        source_chain: u16,
        message_hash: vector<u8>,
        ctx_addr: address,
    ) acquires BridgeState {
        let state = borrow_global_mut<BridgeState>(ctx_addr);
        let id = state.next_message_id;
        state.next_message_id = id + 1;
        
        let pending = PendingMessage {
            id,
            source_chain,
            message_hash,
            submitted_at: timestamp::now_seconds(),
            challenged: false,
            executed: false,
        };
        
        aptos_std::table::add(&mut state.messages, id, pending);
    }
    
    // Challenge a fraudulent message during challenge period
    public fun challenge_message(
        challenger: &signer,
        message_id: u64,
        fraud_proof: vector<u8>,  // Merkle proof showing message is invalid
        ctx_addr: address,
    ) acquires BridgeState {
        let state = borrow_global_mut<BridgeState>(ctx_addr);
        let msg = aptos_std::table::borrow_mut(&mut state.messages, message_id);
        
        assert!(!msg.executed, 1);  // Not yet executed
        assert!(
            timestamp::now_seconds() <= msg.submitted_at + CHALLENGE_PERIOD_SECS,
            2,  // Challenge period expired
        );
        
        // Verify fraud proof (implementation depends on proof system)
        // verify_fraud_proof(&msg.message_hash, &fraud_proof);
        
        msg.challenged = true;
        // Slash submitter, reward challenger
    }
    
    // Execute message after challenge period
    public fun execute_message(
        message_id: u64,
        message_content: vector<u8>,
        ctx_addr: address,
    ) acquires BridgeState {
        let state = borrow_global_mut<BridgeState>(ctx_addr);
        let msg = aptos_std::table::borrow_mut(&mut state.messages, message_id);
        
        assert!(!msg.challenged, 1);  // Not challenged
        assert!(!msg.executed, 2);    // Not already executed
        assert!(
            timestamp::now_seconds() > msg.submitted_at + CHALLENGE_PERIOD_SECS,
            3,  // Still in challenge period
        );
        
        // Verify content matches submitted hash
        let content_hash = aptos_std::aptos_hash::keccak256(message_content);
        assert!(content_hash == msg.message_hash, 4);
        
        msg.executed = true;
        
        // Process the message content...
        process_message(message_content);
    }
    
    fun process_message(_content: vector<u8>) {
        // Decode and execute the bridged action
    }
}
```

---

## Wormhole Integration on Aptos

```move
// Real Wormhole integration patterns on Aptos
// Wormhole contracts deployed at: 0x5bc11445584a763c1e11cc26d1b9b52d8a3bce6b

module my_protocol::wormhole_bridge {
    // Wormhole core contract interface
    // use wormhole::wormhole;
    // use token_bridge::token_bridge;
    // use token_bridge::transfer_tokens;
    
    struct BridgeConfig has key {
        wormhole_chain_id: u16,     // Aptos = 22
        token_bridge_address: address,
        trusted_emitters: aptos_std::table::Table<u16, vector<u8>>,  // chain_id → emitter
    }
    
    struct TransferReceipt has key {
        sequence: u64,
        token_address: address,
        amount: u64,
        recipient_chain: u16,
        recipient: vector<u8>,  // 32-byte address on destination
        nonce: u32,
    }
    
    // Send tokens cross-chain via Wormhole
    // user_tokens: Coin<FA> to bridge
    // recipient_chain: Wormhole chain ID (1=Ethereum, 4=BSC, etc)
    // recipient: 32-byte address on destination chain
    public entry fun bridge_tokens<CoinType>(
        sender: &signer,
        amount: u64,
        relayer_fee: u64,
        recipient_chain: u16,
        recipient: vector<u8>,
        nonce: u32,
        config_addr: address,
    ) acquires BridgeConfig {
        let config = borrow_global<BridgeConfig>(config_addr);
        
        // Validate recipient chain is trusted
        assert!(
            aptos_std::table::contains(&config.trusted_emitters, recipient_chain),
            1,
        );
        
        // Withdraw from sender
        let coins = aptos_framework::coin::withdraw<CoinType>(sender, amount);
        
        // Call Wormhole token bridge (simplified - actual API varies)
        // let sequence = token_bridge::transfer_tokens_with_payload<CoinType>(
        //     sender,
        //     coins,
        //     relayer_fee,
        //     recipient_chain,
        //     recipient,
        //     nonce,
        //     encode_payload(sender),
        // );
        
        // Emit event for indexers
        aptos_framework::event::emit(BridgeInitiated {
            sender: std::signer::address_of(sender),
            token: std::type_info::type_name<CoinType>(),
            amount,
            recipient_chain,
            recipient,
        });
    }
    
    // Receive tokens from another chain (called by relayer with VAA)
    public entry fun complete_bridge<CoinType>(
        relayer: &signer,
        vaa_bytes: vector<u8>,
        config_addr: address,
    ) acquires BridgeConfig {
        let config = borrow_global<BridgeConfig>(config_addr);
        
        // Parse and verify VAA
        // let vaa = wormhole::parse_and_verify_vaa(&vaa_bytes);
        // let payload = token_bridge::complete_transfer_with_payload<CoinType>(vaa);
        
        // Decode recipient from payload
        // let recipient = decode_payload(payload);
        // let coins = get_bridged_coins<CoinType>(payload);
        
        // Transfer to recipient
        // aptos_framework::coin::deposit(recipient, coins);
        
        aptos_framework::event::emit(BridgeCompleted {
            relayer: std::signer::address_of(relayer),
            recipient: @0x0,  // decoded from payload
            amount: 0,
        });
    }
    
    #[event]
    struct BridgeInitiated has drop, store {
        sender: address,
        token: std::string::String,
        amount: u64,
        recipient_chain: u16,
        recipient: vector<u8>,
    }
    
    #[event]
    struct BridgeCompleted has drop, store {
        relayer: address,
        recipient: address,
        amount: u64,
    }
}
```

---

## Message Passing Protocol

```move
// Generic cross-chain message passing
// Pattern: Any message, not just token transfers

module xchain::message_protocol {
    use aptos_framework::timestamp;
    use aptos_std::table::{Self, Table};
    
    // Message types
    const MSG_TOKEN_TRANSFER: u8 = 1;
    const MSG_CONTRACT_CALL: u8 = 2;
    const MSG_GOVERNANCE: u8 = 3;
    
    struct MessageRouter has key {
        nonce: u64,
        sent_messages: Table<u64, SentMessage>,
        received_messages: Table<vector<u8>, bool>,  // hash → processed
        chain_endpoints: Table<u16, ChainEndpoint>,
    }
    
    struct ChainEndpoint has store {
        chain_id: u16,
        endpoint_address: vector<u8>,  // Contract on other chain
        is_active: bool,
    }
    
    struct SentMessage has store {
        nonce: u64,
        destination_chain: u16,
        payload: vector<u8>,
        sent_at: u64,
        delivered: bool,
    }
    
    struct CrossChainMessage has drop {
        version: u8,
        source_chain: u16,
        destination_chain: u16,
        nonce: u64,
        sender: vector<u8>,
        receiver: vector<u8>,
        message_type: u8,
        payload: vector<u8>,
    }
    
    // ENCODING: Pack message for cross-chain transmission
    public fun encode_message(msg: &CrossChainMessage): vector<u8> {
        let mut encoded = std::vector::empty<u8>();
        
        // Version (1 byte)
        std::vector::push_back(&mut encoded, msg.version);
        
        // Chain IDs (2 bytes each)
        append_u16(&mut encoded, msg.source_chain);
        append_u16(&mut encoded, msg.destination_chain);
        
        // Nonce (8 bytes, big-endian)
        append_u64(&mut encoded, msg.nonce);
        
        // Addresses (32 bytes each, zero-padded)
        append_padded_bytes(&mut encoded, &msg.sender, 32);
        append_padded_bytes(&mut encoded, &msg.receiver, 32);
        
        // Message type (1 byte)
        std::vector::push_back(&mut encoded, msg.message_type);
        
        // Payload length (4 bytes) + payload
        let payload_len = (std::vector::length(&msg.payload) as u32);
        append_u32(&mut encoded, payload_len);
        std::vector::append(&mut encoded, msg.payload);
        
        encoded
    }
    
    // DECODING: Unpack received message
    public fun decode_message(bytes: &vector<u8>): CrossChainMessage {
        let mut offset = 0u64;
        
        let version = *std::vector::borrow(bytes, offset);
        offset = offset + 1;
        
        let source_chain = read_u16(bytes, offset);
        offset = offset + 2;
        
        let destination_chain = read_u16(bytes, offset);
        offset = offset + 2;
        
        let nonce = read_u64(bytes, offset);
        offset = offset + 8;
        
        let sender = slice_bytes(bytes, offset, 32);
        offset = offset + 32;
        
        let receiver = slice_bytes(bytes, offset, 32);
        offset = offset + 32;
        
        let message_type = *std::vector::borrow(bytes, offset);
        offset = offset + 1;
        
        let payload_len = read_u32(bytes, offset) as u64;
        offset = offset + 4;
        
        let payload = slice_bytes(bytes, offset, payload_len);
        
        CrossChainMessage {
            version,
            source_chain,
            destination_chain,
            nonce,
            sender,
            receiver,
            message_type,
            payload,
        }
    }
    
    // Send message to another chain
    public fun send_message(
        sender: &signer,
        destination_chain: u16,
        receiver: vector<u8>,
        message_type: u8,
        payload: vector<u8>,
        router_addr: address,
    ): u64 acquires MessageRouter {
        let router = borrow_global_mut<MessageRouter>(router_addr);
        
        assert!(
            table::contains(&router.chain_endpoints, destination_chain),
            1,  // Unknown destination chain
        );
        
        let endpoint = table::borrow(&router.chain_endpoints, destination_chain);
        assert!(endpoint.is_active, 2);  // Endpoint disabled
        
        let nonce = router.nonce;
        router.nonce = nonce + 1;
        
        let msg = CrossChainMessage {
            version: 1,
            source_chain: 22,  // Aptos chain ID
            destination_chain,
            nonce,
            sender: address_to_bytes(std::signer::address_of(sender)),
            receiver,
            message_type,
            payload,
        };
        
        let encoded = encode_message(&msg);
        
        table::add(&mut router.sent_messages, nonce, SentMessage {
            nonce,
            destination_chain,
            payload: encoded,
            sent_at: timestamp::now_seconds(),
            delivered: false,
        });
        
        nonce
    }
    
    // Receive and process message from another chain
    public fun receive_message(
        relayer: &signer,
        encoded_message: vector<u8>,
        proof: vector<u8>,  // Validity proof (varies by bridge)
        router_addr: address,
    ) acquires MessageRouter {
        let router = borrow_global_mut<MessageRouter>(router_addr);
        
        // Compute message hash for deduplication
        let msg_hash = aptos_std::aptos_hash::keccak256(encoded_message);
        
        // Replay protection
        assert!(
            !table::contains(&router.received_messages, msg_hash),
            3,  // Message already processed
        );
        
        // Verify proof (bridge-specific)
        // verify_message_proof(&encoded_message, &proof);
        
        let msg = decode_message(&encoded_message);
        
        // Verify destination
        assert!(msg.destination_chain == 22, 4);  // Must be Aptos
        
        // Mark as processed (replay protection)
        table::add(&mut router.received_messages, msg_hash, true);
        
        // Route to appropriate handler
        if (msg.message_type == MSG_TOKEN_TRANSFER) {
            handle_token_transfer(&msg.payload);
        } else if (msg.message_type == MSG_CONTRACT_CALL) {
            handle_contract_call(&msg.payload);
        } else if (msg.message_type == MSG_GOVERNANCE) {
            handle_governance(&msg.payload);
        };
    }
    
    fun handle_token_transfer(_payload: &vector<u8>) { /* ... */ }
    fun handle_contract_call(_payload: &vector<u8>) { /* ... */ }
    fun handle_governance(_payload: &vector<u8>) { /* ... */ }
    
    // Utility functions for encoding
    fun append_u16(buf: &mut vector<u8>, val: u16) {
        std::vector::push_back(buf, (val >> 8) as u8);
        std::vector::push_back(buf, (val & 0xFF) as u8);
    }
    
    fun append_u32(buf: &mut vector<u8>, val: u32) {
        std::vector::push_back(buf, (val >> 24) as u8);
        std::vector::push_back(buf, ((val >> 16) & 0xFF) as u8);
        std::vector::push_back(buf, ((val >> 8) & 0xFF) as u8);
        std::vector::push_back(buf, (val & 0xFF) as u8);
    }
    
    fun append_u64(buf: &mut vector<u8>, val: u64) {
        let mut i = 7u8;
        loop {
            std::vector::push_back(buf, ((val >> ((i as u64) * 8)) & 0xFF) as u8);
            if (i == 0) break;
            i = i - 1;
        };
    }
    
    fun append_padded_bytes(buf: &mut vector<u8>, bytes: &vector<u8>, target_len: u64) {
        let len = std::vector::length(bytes);
        let mut padding = target_len - len;
        while (padding > 0) {
            std::vector::push_back(buf, 0u8);
            padding = padding - 1;
        };
        std::vector::append(buf, *bytes);
    }
    
    fun read_u16(bytes: &vector<u8>, offset: u64): u16 {
        ((*std::vector::borrow(bytes, offset) as u16) << 8) |
        (*std::vector::borrow(bytes, offset + 1) as u16)
    }
    
    fun read_u32(bytes: &vector<u8>, offset: u64): u32 {
        ((*std::vector::borrow(bytes, offset) as u32) << 24) |
        ((*std::vector::borrow(bytes, offset + 1) as u32) << 16) |
        ((*std::vector::borrow(bytes, offset + 2) as u32) << 8) |
        (*std::vector::borrow(bytes, offset + 3) as u32)
    }
    
    fun read_u64(bytes: &vector<u8>, offset: u64): u64 {
        let mut result = 0u64;
        let mut i = 0u64;
        while (i < 8) {
            result = (result << 8) | (*std::vector::borrow(bytes, offset + i) as u64);
            i = i + 1;
        };
        result
    }
    
    fun slice_bytes(bytes: &vector<u8>, start: u64, len: u64): vector<u8> {
        let mut result = std::vector::empty<u8>();
        let mut i = 0u64;
        while (i < len) {
            std::vector::push_back(&mut result, *std::vector::borrow(bytes, start + i));
            i = i + 1;
        };
        result
    }
    
    fun address_to_bytes(addr: address): vector<u8> {
        // Convert Aptos 32-byte address to bytes
        std::bcs::to_bytes(&addr)
    }
}
```

---

## Cross-Chain Liquidity Routing

```typescript
// ============================================
// TypeScript: Cross-chain swap aggregator
// Finds best route across chains
// ============================================

interface CrossChainRoute {
  sourceChain: string;
  destChain: string;
  sourceToken: string;
  destToken: string;
  bridges: BridgeStep[];
  swaps: SwapStep[];
  expectedOutput: bigint;
  estimatedTime: number;  // seconds
  totalFees: bigint;
}

interface BridgeStep {
  bridge: 'wormhole' | 'layerzero' | 'axelar' | 'stargate';
  sourceChain: string;
  destChain: string;
  token: string;
  fee: bigint;
  time: number;  // seconds
}

interface SwapStep {
  chain: string;
  dex: string;
  tokenIn: string;
  tokenOut: string;
  amountIn: bigint;
  amountOut: bigint;
}

class CrossChainRouter {
  private bridges: Map<string, BridgeAdapter>;
  private dexes: Map<string, DexAdapter>;
  
  constructor() {
    this.bridges = new Map([
      ['wormhole', new WormholeAdapter()],
      ['layerzero', new LayerZeroAdapter()],
      ['stargate', new StargateAdapter()],
    ]);
    
    this.dexes = new Map([
      ['aptos:liquidswap', new LiquidSwapAdapter()],
      ['aptos:pontem', new PontemAdapter()],
      ['sui:cetus', new CetusAdapter()],
      ['ethereum:uniswap', new UniswapAdapter()],
    ]);
  }
  
  // Find best cross-chain route
  async findBestRoute(
    sourceChain: string,
    destChain: string,
    sourceToken: string,
    destToken: string,
    amountIn: bigint,
  ): Promise<CrossChainRoute> {
    const routes = await this.findAllRoutes(
      sourceChain,
      destChain,
      sourceToken,
      destToken,
      amountIn,
    );
    
    // Score routes by: net output after fees, considering time value
    const scored = routes.map(route => ({
      route,
      score: this.scoreRoute(route, amountIn),
    }));
    
    scored.sort((a, b) => Number(b.score - a.score));
    return scored[0].route;
  }
  
  private async findAllRoutes(
    sourceChain: string,
    destChain: string,
    tokenIn: string,
    tokenOut: string,
    amount: bigint,
  ): Promise<CrossChainRoute[]> {
    const routes: CrossChainRoute[] = [];
    
    // Direct bridge routes
    for (const [bridgeName, bridge] of this.bridges) {
      if (bridge.supportsRoute(sourceChain, destChain, tokenIn, tokenOut)) {
        const quote = await bridge.getQuote(sourceChain, destChain, tokenIn, tokenOut, amount);
        routes.push({
          sourceChain,
          destChain,
          sourceToken: tokenIn,
          destToken: tokenOut,
          bridges: [{
            bridge: bridgeName as any,
            sourceChain,
            destChain,
            token: tokenIn,
            fee: quote.fee,
            time: quote.estimatedTime,
          }],
          swaps: [],
          expectedOutput: quote.outputAmount,
          estimatedTime: quote.estimatedTime,
          totalFees: quote.fee,
        });
      }
    }
    
    // Swap + bridge routes
    // e.g., APT→USDC on Aptos, then USDC→ETH on Ethereum
    const intermediateTokens = ['USDC', 'USDT', 'WETH'];
    
    for (const intermediate of intermediateTokens) {
      // Swap on source chain first
      const sourceSwap = await this.getBestSwap(sourceChain, tokenIn, intermediate, amount);
      if (!sourceSwap) continue;
      
      // Bridge the intermediate token
      for (const [bridgeName, bridge] of this.bridges) {
        if (!bridge.supportsRoute(sourceChain, destChain, intermediate, intermediate)) continue;
        
        const bridgeQuote = await bridge.getQuote(
          sourceChain, destChain, intermediate, intermediate, sourceSwap.amountOut
        );
        
        // Swap on destination chain
        const destSwap = await this.getBestSwap(
          destChain, intermediate, tokenOut, bridgeQuote.outputAmount
        );
        if (!destSwap) continue;
        
        const totalFees = sourceSwap.fee + bridgeQuote.fee + destSwap.fee;
        const totalTime = bridgeQuote.estimatedTime + 60;  // +1min for swaps
        
        routes.push({
          sourceChain,
          destChain,
          sourceToken: tokenIn,
          destToken: tokenOut,
          bridges: [{
            bridge: bridgeName as any,
            sourceChain,
            destChain,
            token: intermediate,
            fee: bridgeQuote.fee,
            time: bridgeQuote.estimatedTime,
          }],
          swaps: [
            { chain: sourceChain, dex: sourceSwap.dex, tokenIn, tokenOut: intermediate, amountIn: amount, amountOut: sourceSwap.amountOut },
            { chain: destChain, dex: destSwap.dex, tokenIn: intermediate, tokenOut, amountIn: bridgeQuote.outputAmount, amountOut: destSwap.amountOut },
          ],
          expectedOutput: destSwap.amountOut,
          estimatedTime: totalTime,
          totalFees,
        });
      }
    }
    
    return routes;
  }
  
  private scoreRoute(route: CrossChainRoute, amountIn: bigint): bigint {
    // Net output = expected output - total fees
    const netOutput = route.expectedOutput - route.totalFees;
    
    // Time penalty: 0.01% per minute (opportunity cost)
    const timePenaltyBps = BigInt(Math.floor(route.estimatedTime / 60));
    const timePenalty = (netOutput * timePenaltyBps) / 10_000n;
    
    return netOutput - timePenalty;
  }
  
  private async getBestSwap(
    chain: string,
    tokenIn: string,
    tokenOut: string,
    amount: bigint,
  ): Promise<{ dex: string; amountOut: bigint; fee: bigint } | null> {
    let best: { dex: string; amountOut: bigint; fee: bigint } | null = null;
    
    for (const [key, dex] of this.dexes) {
      if (!key.startsWith(chain + ':')) continue;
      if (!dex.supportsToken(tokenIn) || !dex.supportsToken(tokenOut)) continue;
      
      try {
        const quote = await dex.getQuote(tokenIn, tokenOut, amount);
        if (!best || quote.amountOut > best.amountOut) {
          best = { dex: key, amountOut: quote.amountOut, fee: quote.fee };
        }
      } catch {
        // Skip unavailable DEX
      }
    }
    
    return best;
  }
  
  // Execute the cross-chain route
  async executeRoute(
    route: CrossChainRoute,
    signer: any,  // Chain-specific signer
    slippageBps: number = 50,
  ): Promise<{ txHashes: string[]; estimatedCompletion: Date }> {
    const txHashes: string[] = [];
    
    // Step 1: Execute source chain swaps
    for (const swap of route.swaps.filter(s => s.chain === route.sourceChain)) {
      const minOut = swap.amountOut * BigInt(10_000 - slippageBps) / 10_000n;
      const dex = this.dexes.get(swap.chain + ':' + swap.dex.split(':')[1])!;
      const tx = await dex.swap(signer, swap.tokenIn, swap.tokenOut, swap.amountIn, minOut);
      txHashes.push(tx);
    }
    
    // Step 2: Execute bridge
    for (const bridge of route.bridges) {
      const bridgeAdapter = this.bridges.get(bridge.bridge)!;
      const amount = route.swaps.length > 0
        ? route.swaps.find(s => s.tokenOut === bridge.token)!.amountOut
        : BigInt(0);  // direct bridge from original token
      
      const tx = await bridgeAdapter.bridge(
        signer, bridge.sourceChain, bridge.destChain, bridge.token, amount
      );
      txHashes.push(tx);
    }
    
    return {
      txHashes,
      estimatedCompletion: new Date(Date.now() + route.estimatedTime * 1000),
    };
  }
}

// Stub interfaces for adapters
interface BridgeAdapter {
  supportsRoute(src: string, dst: string, tokenIn: string, tokenOut: string): boolean;
  getQuote(src: string, dst: string, tokenIn: string, tokenOut: string, amount: bigint): Promise<{ outputAmount: bigint; fee: bigint; estimatedTime: number }>;
  bridge(signer: any, src: string, dst: string, token: string, amount: bigint): Promise<string>;
}

interface DexAdapter {
  supportsToken(token: string): boolean;
  getQuote(tokenIn: string, tokenOut: string, amount: bigint): Promise<{ amountOut: bigint; fee: bigint }>;
  swap(signer: any, tokenIn: string, tokenOut: string, amountIn: bigint, minOut: bigint): Promise<string>;
}

// Placeholder implementations
class WormholeAdapter implements BridgeAdapter {
  supportsRoute() { return true; }
  async getQuote(_s: any, _d: any, _ti: any, _to: any, amount: bigint) {
    return { outputAmount: amount * 9970n / 10000n, fee: amount * 30n / 10000n, estimatedTime: 900 };
  }
  async bridge() { return '0x...'; }
}
class LayerZeroAdapter extends WormholeAdapter {}
class StargateAdapter extends WormholeAdapter {}
class LiquidSwapAdapter implements DexAdapter {
  supportsToken() { return true; }
  async getQuote(_ti: any, _to: any, amount: bigint) {
    return { amountOut: amount * 9970n / 10000n, fee: amount * 30n / 10000n };
  }
  async swap() { return '0x...'; }
}
class PontemAdapter extends LiquidSwapAdapter {}
class CetusAdapter extends LiquidSwapAdapter {}
class UniswapAdapter extends LiquidSwapAdapter {}
```

---

## สรุป Cross-Chain Protocol Design

```
Cross-Chain Security Hierarchy:

MOST SECURE (but slow/expensive)
  1. Optimistic bridges: 30-minute challenge period, anyone can fraud-proof
  2. ZK bridges: Cryptographic proof of source chain state (future)
  3. Guardian bridges: Distributed set of validators (Wormhole: 19 guardians)
  4. Multi-sig bridges: M-of-N committee (e.g., 5-of-8)

LEAST SECURE (but fast/cheap)
  5. Single relayer: Centralized, fastest, highest censorship risk

DESIGN CHECKLIST
  [ ] Replay protection: message hash deduplication
  [ ] Source chain verification: validate emitter address
  [ ] Amount validation: no rounding errors across decimal differences
  [ ] Slippage protection: min_out on all swaps
  [ ] Emergency pause: guardian can halt bridge
  [ ] Rate limiting: max transfer per block/day
  [ ] Multi-step atomicity: all steps must succeed or revert
  
COMMON PITFALLS
  ✗ Trust single validator (compromised → all funds lost)
  ✗ Spot price oracle for cross-chain arbitrage
  ✗ No deadline on pending messages
  ✗ Token decimal mismatch between chains
  ✗ Accepting VAAs for wrong chain ID
  ✗ Not validating emitter address (accepting messages from anyone)
  
PRODUCTION MONITORING
  - Alert: Bridge TVL change > 10% in 1 hour
  - Alert: Failed transactions > 5% rate
  - Alert: Guardian set changes
  - Alert: Large single transfers (> $1M)
  - Dashboard: Pending message queue depth
  - Dashboard: Bridge utilization by chain pair
```

---

**ก่อนหน้า**: [Part 77 - Sui Advanced Patterns ←](part-77-sui-advanced.md)
**ต่อไป**: [Part 79 - Move Prover & Formal Verification →](part-79-move-prover.md)
