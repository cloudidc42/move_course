# Part 84: Real-World Asset (RWA) Tokenization

## สารบัญ
- [RWA Overview](#rwa-overview)
- [Tokenized Treasury Bonds](#tokenized-treasury-bonds)
- [Real Estate Tokenization](#real-estate-tokenization)
- [Compliance & KYC/AML](#compliance--kycaml)
- [Fractional Ownership](#fractional-ownership)

---

## RWA Overview

```
REAL-WORLD ASSET TOKENIZATION

What it is:
  Representing ownership of real-world assets as on-chain tokens
  The token ↔ legal claim to the underlying asset

Why Move/Aptos/Sui for RWA:
  ✅ Linear types: Tokens can't be duplicated (ownership preserved)
  ✅ Formal verification: Prove correctness of distribution rules
  ✅ Object model: Rich metadata, composable
  ✅ Compliance hooks: KYC/AML at transfer level
  ✅ Low fees: Frequent distributions feasible

MAJOR RWA CATEGORIES IN 2024+

  TOKENIZED TREASURIES (largest)
  - US Treasury Bills, Bonds
  - Issuers: Franklin Templeton (BENJI), BlackRock (BUIDL)
  - Market size: $1.3B+ on-chain (2024)
  - Yield: 4-5% APY (risk-free rate)
  - Move: Perfect for daily interest accrual
  
  TOKENIZED MONEY MARKET FUNDS
  - Like USDC but with yield
  - Instant liquidity + T-bill yields
  - Composable with DeFi
  
  REAL ESTATE
  - Fractional property ownership
  - Rental income distribution
  - Secondary market trading
  
  PRIVATE CREDIT
  - Corporate loans tokenized
  - Monthly interest distributions
  - Borrower ratings on-chain
  
  COMMODITIES
  - Gold (PAXG model)
  - Oil, agricultural products
  - Physical redemption rights

LEGAL WRAPPER STRUCTURE
  SPV (Special Purpose Vehicle):
    - Separate legal entity holds the asset
    - Token represents ownership in SPV
    - SPV is legally bankruptcy remote
    
  Example:
    Property → SPV LLC → Membership units → ERC-20/FA token
    Tenant pays rent → LLC bank account → Smart contract distributes
```

---

## Tokenized Treasury Bonds

```move
// ============================================
// TOKENIZED T-BILL: Daily yield accrual model
// Users hold tokens, yield accrues automatically
// ============================================

module rwa::treasury_token {
    use aptos_framework::timestamp;
    use aptos_framework::fungible_asset::{Self, MintRef, TransferRef, BurnRef, Metadata};
    use aptos_framework::object;
    use aptos_std::table::{Self, Table};
    
    // REBASING MODEL:
    // Token balance increases each second with interest
    // user_balance = principal * (1 + daily_rate)^days
    // Implemented via index: balance = shares * price_per_share
    
    struct TreasuryToken has key {
        mint_ref: MintRef,
        transfer_ref: TransferRef,
        burn_ref: BurnRef,
        
        // Interest accrual
        price_per_share: u128,      // Scaled by 1e18; starts at 1e18 = $1.00
        annual_yield_bps: u64,      // e.g., 500 = 5% APY
        last_accrual: u64,          // Unix seconds
        
        // Compliance
        compliance_module: address,
        paused: bool,
        
        // Custodian (manages real assets)
        custodian: address,
        
        // Supply tracking
        total_shares: u64,         // Outstanding "shares"
        total_nav: u64,            // Total Net Asset Value in USD (1e6)
    }
    
    struct AllowList has key {
        allowed: Table<address, bool>,  // KYC-approved addresses
        admin: address,
    }
    
    // Initialize token with 5% APY
    public fun initialize(
        admin: &signer,
        custodian: address,
        annual_yield_bps: u64,
        ctx: &mut aptos_framework::object::ConstructorRef,
    ) {
        let admin_addr = std::signer::address_of(admin);
        
        // Create FA token
        let fa_ref = fungible_asset::create_primary_store_enabled_fungible_asset(
            ctx,
            std::option::none(),  // No max supply
            std::string::utf8(b"Tokenized T-Bill"),
            std::string::utf8(b"TBILL"),
            6,   // 6 decimals
            std::string::utf8(b"https://tbill.xyz/icon.png"),
            std::string::utf8(b"https://tbill.xyz"),
        );
        
        let mint_ref = fungible_asset::generate_mint_ref(ctx);
        let transfer_ref = fungible_asset::generate_transfer_ref(ctx);
        let burn_ref = fungible_asset::generate_burn_ref(ctx);
        
        move_to(admin, TreasuryToken {
            mint_ref,
            transfer_ref,
            burn_ref,
            price_per_share: 1_000_000_000_000_000_000,  // 1e18 = $1.00 per share
            annual_yield_bps,
            last_accrual: timestamp::now_seconds(),
            compliance_module: admin_addr,
            paused: false,
            custodian,
            total_shares: 0,
            total_nav: 0,
        });
        
        move_to(admin, AllowList {
            allowed: table::new(),
            admin: admin_addr,
        });
    }
    
    // YIELD ACCRUAL: Update price_per_share based on time elapsed
    // Called periodically (e.g., daily by automated keeper)
    public entry fun accrue_yield(
        keeper: &signer,
        token_addr: address,
    ) acquires TreasuryToken {
        let token = borrow_global_mut<TreasuryToken>(token_addr);
        
        let now = timestamp::now_seconds();
        let elapsed_secs = now - token.last_accrual;
        
        if (elapsed_secs == 0) return;
        
        // Compound interest: price *= (1 + rate_per_second)^elapsed
        // Approximation: price *= 1 + rate_per_second * elapsed (for small elapsed)
        // rate_per_second = annual_yield / 365 / 86400
        
        let annual_rate_scaled = token.annual_yield_bps as u128;  // bps
        let secs_per_year = 31_536_000u128;
        
        // interest = price * rate * elapsed / (10_000 * secs_per_year)
        let interest = token.price_per_share * annual_rate_scaled * (elapsed_secs as u128)
            / (10_000 * secs_per_year);
        
        token.price_per_share = token.price_per_share + interest;
        token.last_accrual = now;
        
        aptos_framework::event::emit(YieldAccrued {
            new_price_per_share: token.price_per_share,
            interest_accrued: interest,
            timestamp: now,
        });
    }
    
    // MINT: User deposits USD, receives shares
    // amount_usd: amount in USD (scaled by 1e6)
    public entry fun mint_tokens(
        user: &signer,
        amount_usd: u64,
        token_addr: address,
    ) acquires TreasuryToken, AllowList {
        let user_addr = std::signer::address_of(user);
        let allow_list = borrow_global<AllowList>(token_addr);
        
        // Compliance check
        assert!(
            table::contains(&allow_list.allowed, user_addr) &&
            *table::borrow(&allow_list.allowed, user_addr),
            1,  // Not KYC approved
        );
        
        let token = borrow_global_mut<TreasuryToken>(token_addr);
        assert!(!token.paused, 2);
        
        // Shares = amount_usd / price_per_share
        // price_per_share is in 1e18, amount_usd in 1e6
        // shares (in 1e6) = amount_usd * 1e18 / price_per_share / 1e6
        let shares = (amount_usd as u128) * 1_000_000_000_000_000_000 
            / token.price_per_share;
        let shares = shares as u64;
        
        token.total_shares = token.total_shares + shares;
        token.total_nav = token.total_nav + amount_usd;
        
        // Mint FA tokens to user
        // fungible_asset::mint_to(&token.mint_ref, user_primary_store, shares);
        
        aptos_framework::event::emit(TokensMinted {
            user: user_addr,
            amount_usd,
            shares,
            price_per_share: token.price_per_share,
        });
    }
    
    // REDEEM: User returns shares, receives USD (off-chain settlement typically)
    public entry fun redeem_tokens(
        user: &signer,
        shares: u64,
        token_addr: address,
    ) acquires TreasuryToken {
        let user_addr = std::signer::address_of(user);
        let token = borrow_global_mut<TreasuryToken>(token_addr);
        assert!(!token.paused, 1);
        
        // USD value = shares * price_per_share
        let usd_value = (shares as u128) * token.price_per_share 
            / 1_000_000_000_000_000_000;
        let usd_value = usd_value as u64;
        
        token.total_shares = token.total_shares - shares;
        token.total_nav = token.total_nav - usd_value;
        
        // Burn shares
        // fungible_asset::burn_from(&token.burn_ref, user_primary_store, shares);
        
        // USD transfer happens off-chain via custodian
        // Custodian receives redemption request event and transfers funds
        
        aptos_framework::event::emit(TokensRedeemed {
            user: user_addr,
            shares,
            usd_value,
            price_per_share: token.price_per_share,
        });
    }
    
    // Get current token price (includes accrued yield)
    public fun get_price_per_share(token_addr: address): u128 acquires TreasuryToken {
        borrow_global<TreasuryToken>(token_addr).price_per_share
    }
    
    // Get user's current USD value
    public fun get_user_value_usd(
        user: address,
        shares: u64,
        token_addr: address,
    ): u64 acquires TreasuryToken {
        let token = borrow_global<TreasuryToken>(token_addr);
        let value = (shares as u128) * token.price_per_share / 1_000_000_000_000_000_000;
        value as u64
    }
    
    #[event] struct YieldAccrued has drop, store { new_price_per_share: u128, interest_accrued: u128, timestamp: u64 }
    #[event] struct TokensMinted has drop, store { user: address, amount_usd: u64, shares: u64, price_per_share: u128 }
    #[event] struct TokensRedeemed has drop, store { user: address, shares: u64, usd_value: u64, price_per_share: u128 }
}
```

---

## Real Estate Tokenization

```move
// ============================================
// REAL ESTATE: Fractional ownership with rental yield
// ============================================

module rwa::property_token {
    use aptos_framework::timestamp;
    use aptos_std::table::{Self, Table};
    
    struct Property has key {
        // Legal identification
        address_hash: vector<u8>,   // Hash of legal address
        parcel_id: std::string::String,  // Legal parcel identifier
        jurisdiction: std::string::String,
        
        // Financial
        appraised_value_usd: u64,   // Scaled by 1e6
        total_tokens: u64,          // Total fractional tokens (e.g., 1,000,000)
        tokens_outstanding: u64,
        
        // Rental income
        monthly_rent_usd: u64,      // Current monthly rent
        last_distribution: u64,     // Timestamp
        accumulated_yield: u64,     // Yield not yet distributed
        
        // Management
        property_manager: address,
        spv_address: std::string::String,  // Legal entity address
        
        // Status
        is_rented: bool,
        maintenance_reserve: u64,   // Buffer for repairs
    }
    
    struct PropertyOwnership has key {
        property_id: address,
        tokens_held: u64,
        unclaimed_yield: u64,      // Yield earned but not claimed
        last_claim: u64,
    }
    
    // Distribute monthly rental income to token holders
    // Called by property manager after receiving rent
    public entry fun distribute_rent(
        manager: &signer,
        property_addr: address,
        rent_received: u64,  // USD received this month
    ) acquires Property {
        let property = borrow_global_mut<Property>(property_addr);
        assert!(std::signer::address_of(manager) == property.property_manager, 1);
        
        let now = timestamp::now_seconds();
        
        // Management fee: 10% to property manager
        let management_fee = rent_received / 10;
        let maintenance_portion = rent_received / 20;  // 5% to reserves
        let distributable = rent_received - management_fee - maintenance_portion;
        
        property.accumulated_yield = property.accumulated_yield + distributable;
        property.maintenance_reserve = property.maintenance_reserve + maintenance_portion;
        property.last_distribution = now;
        
        // Yield per token = distributable / tokens_outstanding
        let yield_per_token = distributable * 1_000_000 / property.tokens_outstanding;  // Scaled
        
        aptos_framework::event::emit(RentDistributed {
            property: property_addr,
            rent_received,
            distributable,
            yield_per_token,
            timestamp: now,
        });
    }
    
    // Claim accumulated yield for a specific holder
    public entry fun claim_yield(
        holder: &signer,
        property_addr: address,
    ) acquires PropertyOwnership, Property {
        let holder_addr = std::signer::address_of(holder);
        let ownership = borrow_global_mut<PropertyOwnership>(holder_addr);
        let property = borrow_global<Property>(property_addr);
        
        assert!(ownership.property_id == property_addr, 1);
        assert!(ownership.unclaimed_yield > 0, 2);
        
        let yield_amount = ownership.unclaimed_yield;
        ownership.unclaimed_yield = 0;
        ownership.last_claim = timestamp::now_seconds();
        
        // Transfer USDC to holder (simplified)
        // coin::transfer<USDC>(treasury, holder_addr, yield_amount);
        
        aptos_framework::event::emit(YieldClaimed {
            holder: holder_addr,
            property: property_addr,
            amount: yield_amount,
        });
    }
    
    // Update property value (quarterly appraisal)
    public entry fun update_appraisal(
        manager: &signer,
        property_addr: address,
        new_value_usd: u64,
        appraisal_hash: vector<u8>,  // Hash of appraisal document
    ) acquires Property {
        let property = borrow_global_mut<Property>(property_addr);
        assert!(std::signer::address_of(manager) == property.property_manager, 1);
        
        let old_value = property.appraised_value_usd;
        property.appraised_value_usd = new_value_usd;
        
        aptos_framework::event::emit(PropertyAppraised {
            property: property_addr,
            old_value,
            new_value: new_value_usd,
            appraisal_hash,
        });
    }
    
    #[event] struct RentDistributed has drop, store { property: address, rent_received: u64, distributable: u64, yield_per_token: u64, timestamp: u64 }
    #[event] struct YieldClaimed has drop, store { holder: address, property: address, amount: u64 }
    #[event] struct PropertyAppraised has drop, store { property: address, old_value: u64, new_value: u64, appraisal_hash: vector<u8> }
}
```

---

## Compliance & KYC/AML

```move
// ============================================
// COMPLIANCE MODULE
// Transfer restrictions for regulated assets
// ============================================

module rwa::compliance {
    use aptos_framework::timestamp;
    use aptos_std::table::{Self, Table};
    
    // Compliance levels (KYC tiers)
    const KYC_NONE: u8 = 0;
    const KYC_BASIC: u8 = 1;    // ID verification
    const KYC_ENHANCED: u8 = 2; // Income verification
    const KYC_ACCREDITED: u8 = 3; // Accredited investor (>$200K income or $1M net worth)
    
    struct ComplianceRegistry has key {
        admin: address,
        users: Table<address, UserCompliance>,
        blocked: Table<address, BlockReason>,
        jurisdiction_rules: Table<std::string::String, u8>,  // jurisdiction → min_kyc
    }
    
    struct UserCompliance has store {
        kyc_level: u8,
        kyc_expiry: u64,           // Must be re-verified periodically
        jurisdiction: std::string::String,
        provider_id: std::string::String,  // KYC provider reference ID
        approved_at: u64,
    }
    
    struct BlockReason has store {
        reason: std::string::String,
        blocked_at: u64,
        blocked_by: address,
    }
    
    // Register a verified user
    public entry fun register_user(
        admin: &signer,
        user: address,
        kyc_level: u8,
        kyc_expiry: u64,
        jurisdiction: std::string::String,
        provider_id: std::string::String,
        registry_addr: address,
    ) acquires ComplianceRegistry {
        let registry = borrow_global_mut<ComplianceRegistry>(registry_addr);
        assert!(std::signer::address_of(admin) == registry.admin, 1);
        
        table::upsert(&mut registry.users, user, UserCompliance {
            kyc_level,
            kyc_expiry,
            jurisdiction,
            provider_id,
            approved_at: timestamp::now_seconds(),
        });
        
        aptos_framework::event::emit(UserRegistered {
            user,
            kyc_level,
            jurisdiction,
        });
    }
    
    // Validate transfer (call from token transfer hook)
    public fun validate_transfer(
        from: address,
        to: address,
        required_kyc: u8,
        registry_addr: address,
    ) acquires ComplianceRegistry {
        let registry = borrow_global<ComplianceRegistry>(registry_addr);
        let now = timestamp::now_seconds();
        
        // Check from address
        assert!(!table::contains(&registry.blocked, from), 1);  // From is blocked
        assert!(table::contains(&registry.users, from), 2);      // From not verified
        let from_compliance = table::borrow(&registry.users, from);
        assert!(from_compliance.kyc_level >= required_kyc, 3);   // From: insufficient KYC
        assert!(from_compliance.kyc_expiry > now, 4);             // From: KYC expired
        
        // Check to address
        assert!(!table::contains(&registry.blocked, to), 5);    // To is blocked
        assert!(table::contains(&registry.users, to), 6);        // To not verified
        let to_compliance = table::borrow(&registry.users, to);
        assert!(to_compliance.kyc_level >= required_kyc, 7);     // To: insufficient KYC
        assert!(to_compliance.kyc_expiry > now, 8);               // To: KYC expired
        
        // Jurisdiction check
        let from_jur = &from_compliance.jurisdiction;
        if (table::contains(&registry.jurisdiction_rules, *from_jur)) {
            let min_kyc = *table::borrow(&registry.jurisdiction_rules, *from_jur);
            assert!(from_compliance.kyc_level >= min_kyc, 9);
        };
    }
    
    // Block a user (sanctions)
    public entry fun block_user(
        admin: &signer,
        user: address,
        reason: std::string::String,
        registry_addr: address,
    ) acquires ComplianceRegistry {
        let registry = borrow_global_mut<ComplianceRegistry>(registry_addr);
        assert!(std::signer::address_of(admin) == registry.admin, 1);
        
        table::upsert(&mut registry.blocked, user, BlockReason {
            reason,
            blocked_at: timestamp::now_seconds(),
            blocked_by: std::signer::address_of(admin),
        });
    }
    
    #[event] struct UserRegistered has drop, store { user: address, kyc_level: u8, jurisdiction: std::string::String }
}
```

---

## สรุป RWA Tokenization

```
RWA ON APTOS/SUI: KEY PATTERNS

TOKEN ARCHITECTURE
  Rebasing (T-bills, yield-bearing):
    price_per_share increases daily
    user_balance_usd = shares * price_per_share
    No rebasing: simpler wallets, composable with DeFi
    
  Fixed supply (real estate):
    Total tokens = 1,000,000 (100% ownership)
    Rental income → USDC distributed to holders
    Token price on secondary market = appraisal / total_supply
    
COMPLIANCE INTEGRATION
  Transfer hooks: Every FA transfer calls validate_transfer()
  KYC tiers: None, Basic, Enhanced, Accredited
  Jurisdictions: Different requirements per country
  Expiry: Annual re-verification
  
LEGAL STRUCTURE
  Token → On-chain (Move smart contract)
  SPV → Off-chain (LLC/trust in target jurisdiction)
  Asset → Real world (bank account, property title)
  
  Chain of legal claim:
    Token holder → SPV membership → Property ownership
    
ORACLES FOR RWA
  T-bills: Published daily NAV from custodian
  Real estate: Quarterly appraisals from certified appraisers
  Private credit: Monthly loan performance reports
  
RISKS
  ✗ Legal risk: Token may not convey actual ownership
  ✗ Custodian risk: Custodian defaults or is dishonest
  ✗ Oracle risk: NAV manipulation
  ✗ Regulatory risk: SEC, local regulations
  ✗ Liquidity risk: No secondary market buyers
  
OPPORTUNITIES
  ✅ $600T traditional finance market mostly off-chain
  ✅ Composable RWA + DeFi (borrow against T-bills)
  ✅ 24/7 settlement vs T+2 traditional
  ✅ Global access to previously restricted investments
```

---

**ก่อนหน้า**: [Part 83 - DeFi Risk Management ←](part-83-risk-management.md)
**ต่อไป**: [Part 85 - Perpetual Futures Protocol →](part-85-perpetuals.md)
