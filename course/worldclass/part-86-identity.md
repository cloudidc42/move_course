# Part 86: Decentralized Identity & Verifiable Credentials

## สารบัญ
- [Decentralized Identity Overview](#decentralized-identity-overview)
- [DID (Decentralized Identifiers)](#did-decentralized-identifiers)
- [Verifiable Credentials on Aptos](#verifiable-credentials-on-aptos)
- [ZK-Proof Identity (Privacy-Preserving)](#zk-proof-identity-privacy-preserving)
- [Soul-Bound Tokens](#soul-bound-tokens)

---

## Decentralized Identity Overview

```
DECENTRALIZED IDENTITY (DID)

Problem with centralized identity:
  - Google/Facebook can revoke your account
  - Each service has its own siloed identity
  - Data breaches expose personal info
  - Users don't control their own credentials

Web3 Identity Model:
  - Self-sovereign: You control your identity
  - Portable: Same identity across all apps
  - Verifiable: Cryptographically provable
  - Privacy-preserving: Reveal only what's needed

W3C STANDARDS
  DID (Decentralized Identifier):
    did:aptos:0x1234...  (like an email address but cryptographic)
    Resolves to DID Document (public keys, service endpoints)
    
  Verifiable Credential (VC):
    JSON-LD document signed by issuer
    Claims: "Alice is KYC verified" / "Bob has a degree in CS"
    Can be verified without asking the issuer (self-contained proof)

COMPONENTS
  Holder: The person who has credentials (wallet)
  Issuer: Authority that issues credentials (Onfido, university, government)
  Verifier: App that checks credentials (DeFi protocol, employer)
  
  Flow:
  1. Issuer checks identity off-chain
  2. Issuer issues VC (signed credential) to holder
  3. Holder presents VC to verifier
  4. Verifier checks: (a) issuer is trusted, (b) signature valid, (c) not revoked

MOVE/APTOS ADVANTAGES FOR IDENTITY
  Soul-bound tokens: Can't be transferred (stuck to identity)
  Linear types: Credential can't be duplicated
  On-chain revocation: Instant, globally visible
  Composable: Use same identity across multiple DeFi protocols
```

---

## DID (Decentralized Identifiers)

```move
// ============================================
// DID DOCUMENT: On-chain identity document
// Stores verification methods and service endpoints
// ============================================

module identity::did_registry {
    use aptos_std::table::{Self, Table};
    use aptos_framework::timestamp;
    
    // Supported key types
    const KEY_TYPE_ED25519: u8 = 1;
    const KEY_TYPE_SECP256K1: u8 = 2;
    
    struct DIDDocument has key {
        did: std::string::String,    // e.g., "did:aptos:0x1234..."
        controller: address,         // Who controls this DID
        verification_methods: vector<VerificationMethod>,
        authentication: vector<u64>,   // Indices into verification_methods
        assertion_method: vector<u64>, // For signing credentials
        service_endpoints: vector<ServiceEndpoint>,
        created: u64,
        updated: u64,
        deactivated: bool,
    }
    
    struct VerificationMethod has store {
        id: std::string::String,       // "#key-1"
        key_type: u8,
        public_key_bytes: vector<u8>,
        is_revoked: bool,
    }
    
    struct ServiceEndpoint has store {
        id: std::string::String,           // "#service-1"
        service_type: std::string::String, // "LinkedDomains", "DIDCommMessaging"
        endpoint: std::string::String,     // URL or address
    }
    
    struct DIDRegistry has key {
        dids: Table<address, std::string::String>,  // address → DID string
        resolver: Table<std::string::String, address>,  // DID string → controller address
        total_dids: u64,
    }
    
    // Create a new DID
    public entry fun create_did(
        controller: &signer,
        public_key_bytes: vector<u8>,
        key_type: u8,
        registry_addr: address,
    ) acquires DIDRegistry {
        let controller_addr = std::signer::address_of(controller);
        let registry = borrow_global_mut<DIDRegistry>(registry_addr);
        
        assert!(!table::contains(&registry.dids, controller_addr), 1);  // DID exists
        
        // Generate DID string from address
        let did_string = format_did(controller_addr);
        
        let doc = DIDDocument {
            did: did_string,
            controller: controller_addr,
            verification_methods: std::vector::singleton(VerificationMethod {
                id: std::string::utf8(b"#key-1"),
                key_type,
                public_key_bytes,
                is_revoked: false,
            }),
            authentication: std::vector::singleton(0u64),     // key-1 can authenticate
            assertion_method: std::vector::singleton(0u64),   // key-1 can assert
            service_endpoints: std::vector::empty(),
            created: timestamp::now_seconds(),
            updated: timestamp::now_seconds(),
            deactivated: false,
        };
        
        move_to(controller, doc);
        
        table::add(&mut registry.dids, controller_addr, format_did(controller_addr));
        table::add(&mut registry.resolver, format_did(controller_addr), controller_addr);
        
        registry.total_dids = registry.total_dids + 1;
        
        aptos_framework::event::emit(DIDCreated {
            did: format_did(controller_addr),
            controller: controller_addr,
        });
    }
    
    // Add a new key to DID
    public entry fun add_key(
        controller: &signer,
        key_id: std::string::String,
        key_type: u8,
        public_key_bytes: vector<u8>,
        can_authenticate: bool,
        can_assert: bool,
    ) acquires DIDDocument {
        let controller_addr = std::signer::address_of(controller);
        let doc = borrow_global_mut<DIDDocument>(controller_addr);
        
        assert!(doc.controller == controller_addr, 1);
        assert!(!doc.deactivated, 2);
        
        let key_idx = std::vector::length(&doc.verification_methods);
        
        std::vector::push_back(&mut doc.verification_methods, VerificationMethod {
            id: key_id,
            key_type,
            public_key_bytes,
            is_revoked: false,
        });
        
        if (can_authenticate) {
            std::vector::push_back(&mut doc.authentication, key_idx);
        };
        if (can_assert) {
            std::vector::push_back(&mut doc.assertion_method, key_idx);
        };
        
        doc.updated = timestamp::now_seconds();
    }
    
    // Revoke a key
    public entry fun revoke_key(
        controller: &signer,
        key_index: u64,
    ) acquires DIDDocument {
        let controller_addr = std::signer::address_of(controller);
        let doc = borrow_global_mut<DIDDocument>(controller_addr);
        assert!(doc.controller == controller_addr, 1);
        
        let key = std::vector::borrow_mut(&mut doc.verification_methods, key_index);
        key.is_revoked = true;
        doc.updated = timestamp::now_seconds();
    }
    
    // Add service endpoint
    public entry fun add_service(
        controller: &signer,
        id: std::string::String,
        service_type: std::string::String,
        endpoint: std::string::String,
    ) acquires DIDDocument {
        let controller_addr = std::signer::address_of(controller);
        let doc = borrow_global_mut<DIDDocument>(controller_addr);
        assert!(doc.controller == controller_addr, 1);
        
        std::vector::push_back(&mut doc.service_endpoints, ServiceEndpoint {
            id, service_type, endpoint,
        });
        doc.updated = timestamp::now_seconds();
    }
    
    // Deactivate DID (irreversible)
    public entry fun deactivate_did(controller: &signer) acquires DIDDocument {
        let controller_addr = std::signer::address_of(controller);
        let doc = borrow_global_mut<DIDDocument>(controller_addr);
        assert!(doc.controller == controller_addr, 1);
        doc.deactivated = true;
        doc.updated = timestamp::now_seconds();
    }
    
    fun format_did(addr: address): std::string::String {
        // did:aptos:0x{address}
        std::string::utf8(b"did:aptos:0x")
        // In practice: append hex encoding of address
    }
    
    #[event] struct DIDCreated has drop, store { did: std::string::String, controller: address }
}
```

---

## Verifiable Credentials on Aptos

```move
// ============================================
// VERIFIABLE CREDENTIALS
// Issued by trusted authorities, stored by users
// ============================================

module identity::credentials {
    use aptos_framework::timestamp;
    use aptos_std::table::{Self, Table};
    
    // Credential types
    const CRED_KYC_BASIC: u8 = 1;
    const CRED_KYC_ACCREDITED: u8 = 2;
    const CRED_AGE_18_PLUS: u8 = 3;
    const CRED_JURISDICTION: u8 = 4;
    const CRED_EMPLOYMENT: u8 = 5;
    
    struct Credential has key {
        id: std::string::String,             // Unique credential ID
        cred_type: u8,
        subject: address,                    // Who this credential is about
        issuer: address,                     // Who issued it
        issuance_date: u64,
        expiration_date: u64,                // 0 = no expiry
        
        // Credential claims (key-value pairs)
        claims: Table<std::string::String, std::string::String>,
        
        // Revocation
        is_revoked: bool,
        revocation_reason: std::string::String,
        
        // Signature (issuer's signature over credential hash)
        signature: vector<u8>,
    }
    
    struct IssuerRegistry has key {
        trusted_issuers: Table<address, IssuerInfo>,
        admin: address,
    }
    
    struct IssuerInfo has store {
        name: std::string::String,
        credential_types: vector<u8>,  // What types they can issue
        is_active: bool,
        issued_count: u64,
    }
    
    struct RevocationRegistry has key {
        revoked: Table<std::string::String, RevocationEntry>,  // cred_id → revocation
    }
    
    struct RevocationEntry has store {
        reason: std::string::String,
        revoked_at: u64,
        revoked_by: address,
    }
    
    // Register as a trusted issuer
    public entry fun register_issuer(
        admin: &signer,
        issuer: address,
        name: std::string::String,
        credential_types: vector<u8>,
        registry_addr: address,
    ) acquires IssuerRegistry {
        let registry = borrow_global_mut<IssuerRegistry>(registry_addr);
        assert!(std::signer::address_of(admin) == registry.admin, 1);
        
        table::upsert(&mut registry.trusted_issuers, issuer, IssuerInfo {
            name,
            credential_types,
            is_active: true,
            issued_count: 0,
        });
    }
    
    // Issue a credential to a subject
    public entry fun issue_credential(
        issuer: &signer,
        cred_id: std::string::String,
        cred_type: u8,
        subject: address,
        expiration_date: u64,
        claim_keys: vector<std::string::String>,
        claim_values: vector<std::string::String>,
        signature: vector<u8>,
        registry_addr: address,
    ) acquires IssuerRegistry {
        let issuer_addr = std::signer::address_of(issuer);
        let registry = borrow_global_mut<IssuerRegistry>(registry_addr);
        
        // Verify issuer is authorized
        assert!(table::contains(&registry.trusted_issuers, issuer_addr), 1);
        let issuer_info = table::borrow_mut(&mut registry.trusted_issuers, issuer_addr);
        assert!(issuer_info.is_active, 2);
        assert!(std::vector::contains(&issuer_info.credential_types, &cred_type), 3);
        
        issuer_info.issued_count = issuer_info.issued_count + 1;
        
        // Build claims table
        let n = std::vector::length(&claim_keys);
        let mut claims = table::new();
        let mut i = 0u64;
        while (i < n) {
            table::add(
                &mut claims,
                *std::vector::borrow(&claim_keys, i),
                *std::vector::borrow(&claim_values, i),
            );
            i = i + 1;
        };
        
        // Store credential at subject's address
        // (In practice, holder stores it themselves after receiving from issuer)
        let cred = Credential {
            id: cred_id,
            cred_type,
            subject,
            issuer: issuer_addr,
            issuance_date: timestamp::now_seconds(),
            expiration_date,
            claims,
            is_revoked: false,
            revocation_reason: std::string::utf8(b""),
            signature,
        };
        
        // Store at a deterministic address for the subject
        move_to(issuer, cred);  // Simplified; real impl uses object model
        
        aptos_framework::event::emit(CredentialIssued {
            cred_id: cred_id,
            cred_type,
            subject,
            issuer: issuer_addr,
        });
    }
    
    // Verify a credential is valid
    public fun verify_credential(
        cred: &Credential,
        required_type: u8,
        required_issuer: address,
        registry_addr: address,
    ): bool acquires IssuerRegistry, RevocationRegistry {
        // 1. Check type
        if (cred.cred_type != required_type) return false;
        
        // 2. Check issuer
        if (cred.issuer != required_issuer) return false;
        
        // 3. Check expiry
        if (cred.expiration_date > 0 && timestamp::now_seconds() > cred.expiration_date) {
            return false
        };
        
        // 4. Check not revoked
        if (cred.is_revoked) return false;
        
        // 5. Check issuer still trusted
        let registry = borrow_global<IssuerRegistry>(registry_addr);
        if (!table::contains(&registry.trusted_issuers, cred.issuer)) return false;
        let issuer_info = table::borrow(&registry.trusted_issuers, cred.issuer);
        if (!issuer_info.is_active) return false;
        
        // 6. Signature verification (simplified)
        // verify_signature(cred.issuer, compute_credential_hash(cred), cred.signature)
        
        true
    }
    
    // Revoke a credential
    public entry fun revoke_credential(
        issuer: &signer,
        cred_id: std::string::String,
        reason: std::string::String,
        revocation_addr: address,
    ) acquires RevocationRegistry {
        let registry = borrow_global_mut<RevocationRegistry>(revocation_addr);
        
        table::add(&mut registry.revoked, cred_id, RevocationEntry {
            reason,
            revoked_at: timestamp::now_seconds(),
            revoked_by: std::signer::address_of(issuer),
        });
    }
    
    #[event] struct CredentialIssued has drop, store { cred_id: std::string::String, cred_type: u8, subject: address, issuer: address }
}
```

---

## ZK-Proof Identity (Privacy-Preserving)

```
ZK-PROOF IDENTITY: Prove claims without revealing data

Problem:
  Protocol needs: "User is 18+"
  User doesn't want to reveal: exact birthdate, name, passport number
  
Solution: Zero-Knowledge Proofs
  User proves: "I know a credential issued by TrustCo that proves I'm 18+"
  Without revealing: Which credential, what the exact age is
  
HOW IT WORKS (simplified)

  1. User has credential: { birthdate: "1990-01-15", issuer: TrustCo, sig: 0x... }
  
  2. User generates ZK proof:
     Input (private): credential, birthdate
     Statement: birthdate makes me 18+ today
     Output: proof (public, ~200 bytes)
     
  3. Verifier checks:
     proof.verify(statement: "18+ today", trusted_issuer: TrustCo)
     
  Result: Verified 18+, without knowing when user was born

CIRCOM/SNARK IN MOVE CONTEXT

  Verification happens on-chain:
  The proof is submitted to a Move function
  Move verifies the SNARK proof (pairing check)
  This is computationally expensive but being optimized
  
  Current state (2024):
  - Groth16 verification: ~3M gas (expensive but feasible)
  - PLONK verification: ~2M gas
  - STARKs: cheaper to verify, larger proofs
  
APTOS ZK INTEGRATION
  Aptos Keyless: Uses ZK proofs for account authentication
  No private key needed - use Google/Apple login
  ZK proof that "I own this Google account" = on-chain auth
  
  This is LIVE on Aptos mainnet:
  - Ephemeral key pair (rotated periodically)
  - ZK proof binds Google auth to ephemeral key
  - On-chain verifier checks the proof
  - User: no seed phrase, no private key management!

USE CASES
  ✅ KYC: Prove verified without revealing identity
  ✅ Age verification: Prove 18+ without revealing age
  ✅ Accredited investor: Prove income threshold without revealing income
  ✅ Jurisdiction: Prove NOT in US without revealing country
  ✅ Credit score: Prove score > 700 without revealing exact score
```

---

## Soul-Bound Tokens

```move
// ============================================
// SOUL-BOUND TOKENS (SBT)
// Non-transferable tokens for identity/reputation
// ============================================

module identity::soul_bound {
    use aptos_framework::fungible_asset::{Self, MintRef, TransferRef, BurnRef};
    use aptos_framework::object;
    
    // SBTs represent achievements, credentials, reputation
    // They CANNOT be transferred (stuck to the person)
    
    struct SoulBoundBadge has key {
        id: u64,
        badge_type: u8,      // Type of achievement
        name: std::string::String,
        description: std::string::String,
        issuer: address,
        issued_to: address,
        issued_at: u64,
        metadata_uri: std::string::String,  // Link to detailed metadata
    }
    
    // Badge types
    const BADGE_KYC_VERIFIED: u8 = 1;
    const BADGE_CONTRIBUTOR: u8 = 2;       // Protocol contributor
    const BADGE_EARLY_ADOPTER: u8 = 3;     // Was here from the beginning
    const BADGE_GOVERNANCE_VOTER: u8 = 4;  // Active governance participant
    const BADGE_LIQUIDATOR: u8 = 5;         // Successfully liquidated positions
    
    struct BadgeAuthority has key {
        admin: address,
        next_badge_id: u64,
        badge_issuers: aptos_std::table::Table<address, vector<u8>>,  // issuer → allowed badge types
    }
    
    // Issue a soul-bound badge
    public entry fun issue_badge(
        issuer: &signer,
        recipient: address,
        badge_type: u8,
        name: std::string::String,
        description: std::string::String,
        metadata_uri: std::string::String,
        authority_addr: address,
    ) acquires BadgeAuthority {
        let issuer_addr = std::signer::address_of(issuer);
        let authority = borrow_global_mut<BadgeAuthority>(authority_addr);
        
        // Check issuer is authorized for this badge type
        assert!(
            aptos_std::table::contains(&authority.badge_issuers, issuer_addr),
            1,
        );
        let allowed_types = aptos_std::table::borrow(&authority.badge_issuers, issuer_addr);
        assert!(std::vector::contains(allowed_types, &badge_type), 2);
        
        let badge_id = authority.next_badge_id;
        authority.next_badge_id = badge_id + 1;
        
        let badge = SoulBoundBadge {
            id: badge_id,
            badge_type,
            name,
            description,
            issuer: issuer_addr,
            issued_to: recipient,
            issued_at: aptos_framework::timestamp::now_seconds(),
            metadata_uri,
        };
        
        // Badges stored at recipient address
        // Note: We need to use a different mechanism since move_to requires signer
        // In practice: Use object model or recipient calls a "receive" function
        // For this example, we emit an event and recipient calls receive_badge
        
        aptos_framework::event::emit(BadgeIssued {
            badge_id,
            badge_type,
            recipient,
            issuer: issuer_addr,
        });
        
        // Store temporarily at issuer, recipient must claim
        // (Or use Aptos object model with transfer restrictions)
        move_to(issuer, badge);
    }
    
    // Burn (revoke) a badge
    public entry fun revoke_badge(
        issuer: &signer,
        badge_holder: address,
        badge_id: u64,
    ) {
        // In real implementation: access badge by ID and burn
        // The key constraint: ONLY issuer can revoke, holder CANNOT transfer
        aptos_framework::event::emit(BadgeRevoked { badge_id, holder: badge_holder });
    }
    
    // ============================================
    // ON-CHAIN REPUTATION SYSTEM
    // Aggregates badges into a reputation score
    // ============================================
    
    struct ReputationScore has key {
        address: address,
        total_score: u64,
        breakdown: aptos_std::table::Table<u8, u64>,  // badge_type → score contribution
        badge_count: u64,
        last_updated: u64,
    }
    
    // Scoring weights per badge type
    fun get_badge_weight(badge_type: u8): u64 {
        if (badge_type == BADGE_KYC_VERIFIED) { 100 }
        else if (badge_type == BADGE_CONTRIBUTOR) { 50 }
        else if (badge_type == BADGE_EARLY_ADOPTER) { 30 }
        else if (badge_type == BADGE_GOVERNANCE_VOTER) { 20 }
        else if (badge_type == BADGE_LIQUIDATOR) { 10 }
        else { 5 }
    }
    
    // Compute on-chain reputation for lending/governance eligibility
    public fun get_reputation(addr: address): u64 acquires ReputationScore {
        if (!exists<ReputationScore>(addr)) return 0;
        borrow_global<ReputationScore>(addr).total_score
    }
    
    // Use reputation in protocol (e.g., reduced fees for high-rep users)
    public fun apply_reputation_discount(
        base_fee_bps: u64,
        user: address,
        max_discount_bps: u64,
    ): u64 acquires ReputationScore {
        let score = get_reputation(user);
        
        // 1% discount per 100 reputation points, capped at max_discount
        let discount_bps = (score / 100) * 100;
        let discount = std::u64::min(discount_bps, max_discount_bps);
        
        if (discount >= base_fee_bps) { 0 } else { base_fee_bps - discount }
    }
    
    #[event] struct BadgeIssued has drop, store { badge_id: u64, badge_type: u8, recipient: address, issuer: address }
    #[event] struct BadgeRevoked has drop, store { badge_id: u64, holder: address }
}
```

---

## สรุป Decentralized Identity

```
IDENTITY ON MOVE: KEY TAKEAWAYS

DID STANDARDS
  W3C DID: did:aptos:{address} resolves to DID Document
  Keys: Ed25519 primary, secp256k1 for EVM compatibility
  Services: Link off-chain endpoints (messaging, API)
  Rotation: Add new keys, revoke old ones
  
VERIFIABLE CREDENTIALS
  Issuer signs credential → Holder stores → Verifier checks
  On-chain revocation: Instant, globally visible
  Expiry: Automatic TTL prevents stale credentials
  Composable: Same VC works across multiple DeFi apps
  
PRIVACY SPECTRUM
  HIGH PRIVACY (ZK Proofs):
    Prove "18+" without revealing age
    Prove "KYC" without revealing identity
    Gas cost: Higher (SNARK verification)
    
  MEDIUM PRIVACY (Selective disclosure):
    Reveal some claims, hide others
    ZK-selective disclosure schemes
    
  LOW PRIVACY (On-chain VC):
    All credential data visible
    Simple but identity exposed
    OK for non-sensitive credentials (badges, achievements)
    
SOUL-BOUND TOKENS (SBTs)
  Use cases:
    - Protocol reputation (can't be bought/sold)
    - KYC verification (tied to verified person)
    - Contribution acknowledgment (non-financial)
    - Sybil resistance (1 person = 1 badge)
    
  Implementation:
    FA with transfer_ref NEVER exposed (or zero-ref)
    Or: require specific signer in every "transfer" (impossible)
    
APTOS KEYLESS (LIVE FEATURE)
  No private key needed for users
  Google/Apple auth → ZK proof → on-chain auth
  Revolutionary UX: Web2 onboarding for Web3
  Keys rotated automatically (no seed phrase risk)
```

---

**ก่อนหน้า**: [Part 85 - Perpetual Futures ←](part-85-perpetuals.md)
**ต่อไป**: [Part 87 - Advanced Testing Strategies →](part-87-testing.md)
