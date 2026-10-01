# Part 69: Multi-Chain Token Standards

## สารบัญ
- [Cross-Chain Token Landscape](#cross-chain-token-landscape)
- [Aptos Fungible Asset Standard](#aptos-fungible-asset-standard)
- [Wrapped Token Pattern](#wrapped-token-pattern)
- [LayerZero OFT on Aptos/Sui](#layerzero-oft-on-aptos-sui)
- [Wormhole Token Bridge](#wormhole-token-bridge)
- [Native Multi-Chain Token Design](#native-multi-chain-token-design)

---

## Cross-Chain Token Landscape

```
Token Standards Across Chains:

Ethereum: ERC-20
  interface ERC20:
    totalSupply, balanceOf, transfer, approve, transferFrom
    
Aptos: 
  Legacy: 0x1::coin::Coin<T> (older standard)
  New:    0x1::fungible_asset (v2, preferred for new tokens)
  
Sui:
  sui::coin::Coin<T> (similar to Aptos legacy)
  + newer standards for advanced features

Cross-Chain Transfer Methods:
  1. LOCK & MINT (most common)
     Chain A: Lock token → mint wrapped token on Chain B
     Chain B: Burn wrapped token → unlock on Chain A
     
  2. BURN & MINT
     Canonical token can exist on multiple chains
     Burn on one chain, mint on another
     Requires trusted bridge contract
     
  3. ATOMIC SWAP
     No locking/minting, direct peer-to-peer
     Requires both parties online simultaneously
     
  4. INTENT-BASED
     User declares intent, solver finds route
     Solver takes bridge risk, user gets funds fast
     
Standards for Cross-Chain Tokens:
  Circle CCTP: USDC native burn/mint
  Wormhole Wrapped: Lock/mint with VAA
  LayerZero OFT v2: Burn/mint with omnichain
  Axelar ITS: Token service with message passing
```

---

## Aptos Fungible Asset Standard

```move
module token::fungible_asset_v2 {
    use aptos_framework::fungible_asset::{
        Self, MintRef, TransferRef, BurnRef, Metadata,
        FungibleAsset, FungibleStore
    };
    use aptos_framework::object::{Self, Object};
    use aptos_framework::primary_fungible_store;
    
    // ============================================
    // APTOS FUNGIBLE ASSET v2 (new standard)
    // More composable than coin::Coin
    // Supports extensions, hooks, and automation
    // ============================================
    
    #[resource_group_member(group = aptos_framework::object::ObjectGroup)]
    struct ManagedToken has key {
        mint_ref: MintRef,
        transfer_ref: TransferRef,
        burn_ref: BurnRef,
    }
    
    // Create a new fungible asset
    public fun create_token(
        creator: &signer,
        name: std::string::String,
        symbol: std::string::String,
        decimals: u8,
        icon_uri: std::string::String,
        project_uri: std::string::String,
    ): Object<Metadata> {
        let (constructor_ref, _) = aptos_framework::object::create_sticky_object(creator);
        
        primary_fungible_store::create_primary_store_enabled_fungible_asset(
            &constructor_ref,
            std::option::none(),  // No max supply
            name,
            symbol,
            decimals,
            icon_uri,
            project_uri,
        );
        
        let mint_ref = fungible_asset::generate_mint_ref(&constructor_ref);
        let transfer_ref = fungible_asset::generate_transfer_ref(&constructor_ref);
        let burn_ref = fungible_asset::generate_burn_ref(&constructor_ref);
        
        let token_signer = aptos_framework::object::generate_signer(&constructor_ref);
        move_to(&token_signer, ManagedToken {
            mint_ref,
            transfer_ref,
            burn_ref,
        });
        
        aptos_framework::object::object_from_constructor_ref::<Metadata>(&constructor_ref)
    }
    
    // Mint tokens
    public fun mint(
        admin: &signer,
        metadata: Object<Metadata>,
        to: address,
        amount: u64,
    ) acquires ManagedToken {
        let metadata_addr = aptos_framework::object::object_address(&metadata);
        let managed = borrow_global<ManagedToken>(metadata_addr);
        
        let fa = fungible_asset::mint(&managed.mint_ref, amount);
        primary_fungible_store::deposit(to, fa);
    }
    
    // Burn tokens
    public fun burn(
        admin: &signer,
        from: address,
        metadata: Object<Metadata>,
        amount: u64,
    ) acquires ManagedToken {
        let metadata_addr = aptos_framework::object::object_address(&metadata);
        let managed = borrow_global<ManagedToken>(metadata_addr);
        
        let fa = primary_fungible_store::withdraw(admin, metadata, amount);
        fungible_asset::burn(&managed.burn_ref, fa);
    }
    
    // Force transfer (for compliance/freeze scenarios)
    public fun force_transfer(
        admin: &signer,
        metadata: Object<Metadata>,
        from: address,
        to: address,
        amount: u64,
    ) acquires ManagedToken {
        let metadata_addr = aptos_framework::object::object_address(&metadata);
        let managed = borrow_global<ManagedToken>(metadata_addr);
        
        let from_store = primary_fungible_store::primary_store(from, metadata);
        let to_store = primary_fungible_store::ensure_primary_store_exists(to, metadata);
        
        let transfer_ref = &managed.transfer_ref;
        let linear_ref = fungible_asset::generate_linear_transfer_ref(transfer_ref);
        
        let fa = fungible_asset::withdraw_with_ref(linear_ref, from_store, amount);
        // fungible_asset::deposit_with_ref(&transfer_ref, to_store, fa);
    }
    
    // Get balance
    public fun balance_of(owner: address, metadata: Object<Metadata>): u64 {
        if (primary_fungible_store::primary_store_exists(owner, metadata)) {
            primary_fungible_store::balance(owner, metadata)
        } else {
            0
        }
    }
    
    // Total supply
    public fun total_supply(metadata: Object<Metadata>): u128 {
        std::option::get_with_default(
            &fungible_asset::supply(metadata),
            0
        )
    }
}
```

---

## Wrapped Token Pattern

```move
module bridge::wrapped_token {
    use aptos_framework::fungible_asset::{Self, MintRef, BurnRef, Metadata};
    use aptos_framework::primary_fungible_store;
    use aptos_framework::object;
    
    // ============================================
    // WRAPPED TOKEN
    // Lock native token on Chain A
    // Mint wrapped version on Chain B (Aptos)
    //
    // Example: Wrapped ETH on Aptos (WETH-Aptos)
    // ============================================
    
    struct WrappedToken has key {
        // Original chain info
        origin_chain_id: u64,      // e.g., 1 for Ethereum
        origin_token_address: vector<u8>,  // ERC20 address on Ethereum
        
        // Refs for this chain
        mint_ref: MintRef,
        burn_ref: BurnRef,
        
        // Bridge authority
        bridge_admin: address,
        
        // Supply tracking (must match bridge deposits)
        bridged_supply: u64,
    }
    
    // Called by bridge when tokens are locked on origin chain
    public fun bridge_mint(
        bridge: &signer,
        wrapped_token_addr: address,
        recipient: address,
        amount: u64,
        origin_tx_hash: vector<u8>,  // For verification/dedup
    ) acquires WrappedToken {
        let wt = borrow_global_mut<WrappedToken>(wrapped_token_addr);
        
        // Only bridge admin can mint
        assert!(std::signer::address_of(bridge) == wt.bridge_admin, 1);
        
        // Mint wrapped tokens
        let fa = fungible_asset::mint(&wt.mint_ref, amount);
        primary_fungible_store::deposit(recipient, fa);
        
        wt.bridged_supply = wt.bridged_supply + amount;
    }
    
    // Called by user to bridge back (burn wrapped, unlock origin)
    public entry fun bridge_burn(
        user: &signer,
        wrapped_token_addr: address,
        amount: u64,
        destination_address: vector<u8>,  // Address on origin chain
    ) acquires WrappedToken {
        let wt = borrow_global_mut<WrappedToken>(wrapped_token_addr);
        
        // Burn wrapped tokens
        // In production: get metadata from wrapped_token_addr
        // let fa = primary_fungible_store::withdraw(user, metadata, amount);
        // fungible_asset::burn(&wt.burn_ref, fa);
        
        wt.bridged_supply = wt.bridged_supply - amount;
        
        // Emit event for bridge relayer to pick up
        // Event includes: destination_address, amount, origin_chain_id
        aptos_framework::event::emit(BridgeBurnEvent {
            sender: std::signer::address_of(user),
            amount,
            destination_chain_id: wt.origin_chain_id,
            destination_address,
        });
    }
    
    #[event]
    struct BridgeBurnEvent has drop, store {
        sender: address,
        amount: u64,
        destination_chain_id: u64,
        destination_address: vector<u8>,
    }
    
    // Verify supply matches bridged amount
    public fun verify_supply(wrapped_token_addr: address): bool acquires WrappedToken {
        let wt = borrow_global<WrappedToken>(wrapped_token_addr);
        // In production: compare total_supply() with bridged_supply
        // total_supply should never exceed bridged_supply
        true
    }
}
```

---

## LayerZero OFT on Aptos/Sui

```move
module layerzero::oft_aptos {
    // ============================================
    // OMNICHAIN FUNGIBLE TOKEN (OFT) v2
    // LayerZero protocol for multi-chain tokens
    //
    // Model: Burn/Mint
    // - Send from Aptos: burn tokens + send LZ message
    // - Receive on Ethereum: mint tokens + verify LZ proof
    //
    // LayerZero message path:
    // User → OFT contract → LayerZero endpoint → DVN (verification)
    //      → Oracle reports → Receiver → Mint on destination
    // ============================================
    
    const CHAIN_ID_ETHEREUM: u64 = 101;
    const CHAIN_ID_BNB: u64 = 102;
    const CHAIN_ID_APTOS: u64 = 108;
    const CHAIN_ID_SUI: u64 = 149;
    
    // Shared decimals between chains (to avoid precision issues)
    // If token has 18 decimals on ETH and 8 on Aptos:
    // Use 6 shared decimals (ld2sd converts between)
    const LOCAL_DECIMALS: u8 = 8;    // Aptos
    const SHARED_DECIMALS: u8 = 6;   // Cross-chain standard
    
    struct OFTConfig has key {
        token_metadata: aptos_framework::object::Object<aptos_framework::fungible_asset::Metadata>,
        lz_endpoint: address,
        
        // Trusted remotes (peer OFT contracts on other chains)
        trusted_remotes: aptos_std::smart_table::SmartTable<u64, vector<u8>>,
        
        // Limits per chain per hour
        outbound_limits: aptos_std::smart_table::SmartTable<u64, u64>,
        inbound_limits: aptos_std::smart_table::SmartTable<u64, u64>,
        
        // Credit tracking (for rate limits)
        outbound_credits: aptos_std::smart_table::SmartTable<u64, u64>,
    }
    
    // Convert local amount to shared decimals (remove trailing precision)
    fun ld2sd(local_amount: u64): u64 {
        let decimal_diff = (LOCAL_DECIMALS - SHARED_DECIMALS) as u64;
        let conversion_factor = pow10(decimal_diff);
        local_amount / conversion_factor
    }
    
    // Convert shared decimals back to local
    fun sd2ld(shared_amount: u64): u64 {
        let decimal_diff = (LOCAL_DECIMALS - SHARED_DECIMALS) as u64;
        let conversion_factor = pow10(decimal_diff);
        shared_amount * conversion_factor
    }
    
    // Dust amount that gets lost in conversion
    fun ld2sd_dust(local_amount: u64): u64 {
        local_amount - sd2ld(ld2sd(local_amount))
    }
    
    // Send tokens to another chain
    public entry fun send<T>(
        sender: &signer,
        config_addr: address,
        dst_chain_id: u64,
        to_address: vector<u8>,   // Recipient address on destination chain
        amount_ld: u64,            // Amount in local decimals
        min_amount_ld: u64,        // Min after decimal conversion
        lz_fee: u64,               // LayerZero messaging fee in APT
    ) acquires OFTConfig {
        let config = borrow_global<OFTConfig>(config_addr);
        let sender_addr = std::signer::address_of(sender);
        
        // Verify trusted remote exists
        assert!(aptos_std::smart_table::contains(&config.trusted_remotes, dst_chain_id), 1);
        
        // Convert to shared decimals (precision reduction)
        let amount_sd = ld2sd(amount_ld);
        let amount_ld_actual = sd2ld(amount_sd); // Amount after precision loss
        let dust = amount_ld - amount_ld_actual; // Dust returned to sender
        
        assert!(amount_ld_actual >= min_amount_ld, 2);
        
        // Apply rate limit
        apply_outbound_limit(config_addr, dst_chain_id, amount_sd);
        
        // Burn tokens from sender
        // (In production: burn fungible asset)
        
        // Build LayerZero message payload
        let payload = encode_send_payload(to_address, amount_sd);
        
        // Send via LayerZero endpoint
        // lz_endpoint::send(dst_chain_id, trusted_remote, payload, lz_fee);
        
        // Return dust if any
        // if (dust > 0) { mint(sender_addr, dust); };
        
        aptos_framework::event::emit(OFTSentEvent {
            sender: sender_addr,
            dst_chain_id,
            to_address,
            amount_ld: amount_ld_actual,
            amount_sd,
        });
    }
    
    // Receive tokens from another chain (called by LayerZero receiver)
    public fun lz_receive(
        config_addr: address,
        src_chain_id: u64,
        src_address: vector<u8>,
        payload: vector<u8>,
    ) acquires OFTConfig {
        let config = borrow_global<OFTConfig>(config_addr);
        
        // Verify source is trusted
        assert!(aptos_std::smart_table::contains(&config.trusted_remotes, src_chain_id), 1);
        let trusted = aptos_std::smart_table::borrow(&config.trusted_remotes, src_chain_id);
        assert!(src_address == *trusted, 2);
        
        // Decode payload
        let (to_address, amount_sd) = decode_receive_payload(&payload);
        let amount_ld = sd2ld(amount_sd);
        
        // Apply inbound rate limit
        // apply_inbound_limit(config_addr, src_chain_id, amount_sd);
        
        // Mint tokens to recipient
        // fungible_asset::mint(to_address, amount_ld);
        
        aptos_framework::event::emit(OFTReceivedEvent {
            src_chain_id,
            to: to_address,
            amount_ld,
        });
    }
    
    fun apply_outbound_limit(_addr: address, _chain: u64, _amount: u64) acquires OFTConfig {
        // Rate limiting logic
    }
    
    fun encode_send_payload(to: vector<u8>, amount: u64): vector<u8> {
        let mut payload = to;
        // Encode amount as 8 bytes big-endian
        let mut i = 0u8;
        while (i < 8) {
            std::vector::push_back(&mut payload, ((amount >> ((7 - i) * 8)) & 0xFF) as u8);
            i = i + 1;
        };
        payload
    }
    
    fun decode_receive_payload(payload: &vector<u8>): (address, u64) {
        // Decode from bytes
        (@0x0, 0) // Placeholder
    }
    
    fun pow10(n: u64): u64 {
        let mut result = 1u64;
        let mut i = 0u64;
        while (i < n) {
            result = result * 10;
            i = i + 1;
        };
        result
    }
    
    #[event]
    struct OFTSentEvent has drop, store {
        sender: address,
        dst_chain_id: u64,
        to_address: vector<u8>,
        amount_ld: u64,
        amount_sd: u64,
    }
    
    #[event]
    struct OFTReceivedEvent has drop, store {
        src_chain_id: u64,
        to: address,
        amount_ld: u64,
    }
}
```

---

## Native Multi-Chain Token Design

```
Best Practices for Multi-Chain Token Design:

1. CHOOSE YOUR CANONICAL CHAIN
   - Where is the token primarily traded/held?
   - Where is the issuer based? (regulatory)
   - Ethereum often canonical for DeFi
   - Aptos/Sui for gaming/NFT projects
   
2. BRIDGE ARCHITECTURE DECISION
   Lock/Mint (Wormhole):
     + Simple to understand
     - Bridge holds ALL native tokens (honeypot)
     - Bridge hack = all wrapped tokens worthless
   
   Burn/Mint (LayerZero OFT, Circle CCTP):
     + No central honeypot
     + Token supply conserved across chains
     - Requires trust in message delivery
     - If message lost: tokens burned but not minted
   
   Third-Party Liquidity (Intent-based):
     + No bridge risk
     + Fast settlement
     - Solver must have liquidity
     - Higher fees for illiquid tokens

3. DECIMAL HANDLING
   Different chains have different decimal standards:
     Ethereum ERC-20: Usually 18 decimals
     Aptos Coin: Usually 8 decimals  
     Sui Coin: Usually 9 decimals
   
   LayerZero solution: SHARED DECIMALS (usually 6)
     ld2sd(): Local Decimals → Shared Decimals (precision loss)
     sd2ld(): Shared Decimals → Local Decimals
     
   Example: 1.23456789 ETH (18 decimals)
     Local:  1234567890000000000 (18 decimals)
     Shared: 1234567 (6 decimals) ← truncated!
     Dust:   0.000000890000000000 ETH (lost precision, returned to sender)

4. RATE LIMITING (Security)
   Max outbound/inbound per hour per chain
   Prevents: Bridge exploit draining destination chain
   
   e.g., Max 1M USDC per hour per chain
   If bridge hacked: attacker can only steal 1M/hr
   Alert system can catch and pause within 1 hour

5. PAUSE MECHANISM
   Bridge guardian can pause all operations
   Separate role from admin (faster response)
   
6. MONITORING
   Alert if: supply on chain A + chain B ≠ total supply
   Monitor bridge events in real-time
   Anomaly detection: large sudden flows

Token Standard Comparison:
  Aptos Coin (legacy): Simple, battle-tested
  Aptos FA v2 (new):   Extensible, hooks, automation
  Sui Coin:            Object-based, owned per tx
  
  For cross-chain: Use FA v2 on Aptos, standard Coin on Sui
  For new projects: FA v2 on Aptos preferred
```

---

**ก่อนหน้า**: [Part 68 - Insurance Protocols ←](part-68-insurance-protocols.md)
**ต่อไป**: [Part 70 - Real-World Protocol Architecture →](part-70-protocol-architecture.md)
