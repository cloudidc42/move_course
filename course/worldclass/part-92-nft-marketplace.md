# Part 92: NFT Marketplace Protocol

## สารบัญ
- [NFT Marketplace Architecture](#nft-marketplace-architecture)
- [Aptos NFT Marketplace](#aptos-nft-marketplace)
- [Sui Kiosk Marketplace](#sui-kiosk-marketplace)
- [Royalty Enforcement](#royalty-enforcement)
- [Auction Mechanism](#auction-mechanism)
- [Collection Analytics](#collection-analytics)

---

## NFT Marketplace Architecture

```
NFT MARKETPLACE COMPONENTS

                    Creator
                      │ mints
                      ▼
              ┌──────────────┐
              │  NFT / Token │
              └──────┬───────┘
                     │ lists on
                     ▼
         ┌────────────────────────┐
         │      Marketplace       │
         │                        │
         │  ┌──────────────────┐  │
         │  │  Listing Engine  │  │  ← Fixed price / Auction
         │  └──────────────────┘  │
         │  ┌──────────────────┐  │
         │  │ Royalty Registry │  │  ← On-chain enforced
         │  └──────────────────┘  │
         │  ┌──────────────────┐  │
         │  │  Offer System    │  │  ← Bids/offers
         │  └──────────────────┘  │
         └────────────────────────┘
                     │
                     │ buyer purchases
                     ▼
              ┌──────────────┐
              │    Buyer     │
              └──────────────┘

FEE FLOW (on purchase):
  Purchase Price = 1000 USDC
  ├── Marketplace Fee: 2.5% = 25 USDC
  ├── Royalty: 5% = 50 USDC → Creator
  └── Seller Proceeds: 92.5% = 925 USDC

APTOS vs SUI APPROACH
  Aptos: Custom marketplace with TokenV2 objects
  Sui:   Built-in Kiosk primitive with TransferPolicy
```

---

## Aptos NFT Marketplace

```move
// ============================================
// APTOS NFT MARKETPLACE
// Using Aptos TokenV2 (Digital Asset Standard)
// ============================================

module nft_market::marketplace {
    use aptos_framework::coin;
    use aptos_framework::timestamp;
    use aptos_token_objects::token;
    use std::signer;
    use std::string::String;
    
    const ERR_NOT_OWNER: u64 = 1;
    const ERR_ALREADY_LISTED: u64 = 2;
    const ERR_NOT_LISTED: u64 = 3;
    const ERR_WRONG_PRICE: u64 = 4;
    const ERR_EXPIRED: u64 = 5;
    const ERR_PAUSED: u64 = 6;
    
    const MARKETPLACE_FEE_BPS: u64 = 250;  // 2.5%
    const MAX_ROYALTY_BPS: u64 = 1000;     // 10% max royalty
    
    /// A listed NFT on the marketplace
    struct Listing has key {
        seller: address,
        token_addr: address,     // Address of the token object
        price: u64,              // In octa (APT) or token units
        payment_coin_type: u8,   // 0=APT, 1=USDC, etc.
        created_at: u64,
        expires_at: u64,         // 0 = no expiry
    }
    
    /// Royalty configuration per collection
    struct RoyaltyConfig has key {
        creator: address,
        royalty_bps: u64,
        royalty_recipient: address,
    }
    
    /// Marketplace global config
    struct MarketplaceConfig has key {
        admin: address,
        fee_recipient: address,
        fee_bps: u64,
        paused: bool,
        total_volume: u128,
        total_sales: u64,
    }
    
    /// Offer: buyer makes an offer on a specific NFT
    struct Offer has key {
        buyer: address,
        token_addr: address,
        offered_price: u64,
        created_at: u64,
        expires_at: u64,
        // Buyer's payment escrowed here
    }
    
    public entry fun initialize(admin: &signer) {
        move_to(admin, MarketplaceConfig {
            admin: signer::address_of(admin),
            fee_recipient: signer::address_of(admin),
            fee_bps: MARKETPLACE_FEE_BPS,
            paused: false,
            total_volume: 0,
            total_sales: 0,
        });
    }
    
    /// List an NFT for fixed-price sale
    public entry fun list_nft<TokenType: key>(
        seller: &signer,
        token_addr: address,
        price: u64,
        expires_hours: u64,
    ) {
        let seller_addr = signer::address_of(seller);
        
        // Verify seller owns the token
        // assert!(token::owner(token_addr) == seller_addr, ERR_NOT_OWNER);
        
        // Verify not already listed
        assert!(!exists<Listing>(token_addr), ERR_ALREADY_LISTED);
        
        let now = timestamp::now_microseconds();
        let expires_at = if (expires_hours == 0) {
            0  // No expiry
        } else {
            now + expires_hours * 3_600_000_000
        };
        
        // Transfer NFT to escrow (marketplace holds it)
        // object::transfer(seller, token_object, @marketplace);
        
        move_to(seller, Listing {
            seller: seller_addr,
            token_addr,
            price,
            payment_coin_type: 0,
            created_at: now,
            expires_at,
        });
    }
    
    /// Buy a listed NFT
    public entry fun buy_nft<CoinType>(
        buyer: &signer,
        listing_addr: address,
        max_price: u64,  // Slippage protection
    ) {
        let config = borrow_global_mut<MarketplaceConfig>(@nft_market);
        assert!(!config.paused, ERR_PAUSED);
        
        let listing = move_from<Listing>(listing_addr);
        
        // Validate
        assert!(listing.price <= max_price, ERR_WRONG_PRICE);
        let now = timestamp::now_microseconds();
        if (listing.expires_at > 0) {
            assert!(now < listing.expires_at, ERR_EXPIRED);
        };
        
        let buyer_addr = signer::address_of(buyer);
        
        // Calculate fee splits
        let market_fee = listing.price * config.fee_bps / 10_000;
        let royalty = get_royalty_amount(listing.token_addr, listing.price);
        let seller_proceeds = listing.price - market_fee - royalty;
        
        // Collect payment from buyer
        // coin::transfer<CoinType>(buyer, @marketplace_escrow, listing.price);
        
        // Distribute payments
        // coin::transfer<CoinType>(marketplace_signer, config.fee_recipient, market_fee);
        // coin::transfer<CoinType>(marketplace_signer, royalty_recipient, royalty);
        // coin::transfer<CoinType>(marketplace_signer, listing.seller, seller_proceeds);
        
        // Transfer NFT to buyer
        // object::transfer(marketplace_signer, token_object, buyer_addr);
        
        // Update stats
        config.total_volume = config.total_volume + (listing.price as u128);
        config.total_sales = config.total_sales + 1;
        
        let _ = (buyer_addr, seller_proceeds, royalty);
    }
    
    /// Cancel listing
    public entry fun cancel_listing(
        seller: &signer,
        listing_addr: address,
    ) {
        let listing = move_from<Listing>(listing_addr);
        assert!(listing.seller == signer::address_of(seller), ERR_NOT_OWNER);
        
        // Return NFT to seller
        // object::transfer(marketplace_signer, token_object, listing.seller);
        
        let Listing { seller: _, token_addr: _, price: _, payment_coin_type: _, created_at: _, expires_at: _ } = listing;
    }
    
    /// Make an offer on any NFT (not necessarily listed)
    public entry fun make_offer<CoinType>(
        buyer: &signer,
        token_addr: address,
        offer_price: u64,
        expires_hours: u64,
    ) {
        let buyer_addr = signer::address_of(buyer);
        let now = timestamp::now_microseconds();
        
        // Escrow buyer's payment
        // coin::transfer<CoinType>(buyer, @marketplace_escrow, offer_price);
        
        move_to(buyer, Offer {
            buyer: buyer_addr,
            token_addr,
            offered_price: offer_price,
            created_at: now,
            expires_at: now + expires_hours * 3_600_000_000,
        });
    }
    
    /// Seller accepts an offer
    public entry fun accept_offer<CoinType>(
        seller: &signer,
        offer_addr: address,
        token_addr: address,
    ) {
        let config = borrow_global_mut<MarketplaceConfig>(@nft_market);
        let offer = move_from<Offer>(offer_addr);
        
        assert!(offer.token_addr == token_addr, ERR_NOT_LISTED);
        assert!(timestamp::now_microseconds() < offer.expires_at, ERR_EXPIRED);
        
        let market_fee = offer.offered_price * config.fee_bps / 10_000;
        let royalty = get_royalty_amount(token_addr, offer.offered_price);
        let seller_proceeds = offer.offered_price - market_fee - royalty;
        
        // Transfer escrowed payment
        // Distribute: market fee, royalty, seller
        // Transfer NFT to buyer
        
        config.total_volume = config.total_volume + (offer.offered_price as u128);
        config.total_sales = config.total_sales + 1;
        
        let _ = (signer::address_of(seller), seller_proceeds, royalty);
        
        let Offer { buyer: _, token_addr: _, offered_price: _, created_at: _, expires_at: _ } = offer;
    }
    
    /// Set royalty for a collection
    public entry fun set_collection_royalty(
        creator: &signer,
        collection_addr: address,
        royalty_bps: u64,
        royalty_recipient: address,
    ) {
        assert!(royalty_bps <= MAX_ROYALTY_BPS, 1);
        
        move_to(creator, RoyaltyConfig {
            creator: signer::address_of(creator),
            royalty_bps,
            royalty_recipient,
        });
        
        let _ = collection_addr;
    }
    
    fun get_royalty_amount(token_addr: address, price: u64): u64 {
        // Look up royalty config for token's collection
        // If no config, 0 royalty
        let _ = token_addr;
        price * 500 / 10_000  // Default 5% royalty (simplified)
    }
    
    #[view]
    public fun get_listing(listing_addr: address): (address, u64, u64) {
        let listing = borrow_global<Listing>(listing_addr);
        (listing.seller, listing.price, listing.expires_at)
    }
}
```

---

## Sui Kiosk Marketplace

```move
// ============================================
// SUI KIOSK-BASED MARKETPLACE
// Uses Sui's native Kiosk primitive
// ============================================

module sui_nft_market::marketplace {
    use sui::kiosk::{Self, Kiosk, KioskOwnerCap};
    use sui::transfer_policy::{Self, TransferPolicy, TransferRequest};
    use sui::coin::{Self, Coin};
    use sui::sui::SUI;
    use sui::object::{Self, UID};
    use sui::tx_context::TxContext;
    use sui::event;
    
    /// Marketplace config: rules for all sales
    struct MarketConfig has key {
        id: UID,
        fee_bps: u64,
        fee_recipient: address,
    }
    
    /// Royalty rule: enforced at protocol level
    struct RoyaltyRule has drop {}
    
    struct RoyaltyConfig has key {
        id: UID,
        amount_bps: u64,
        recipient: address,
    }
    
    #[event]
    struct NFTSold has copy, drop {
        kiosk: address,
        item_id: address,
        price: u64,
        buyer: address,
    }
    
    /// Create a marketplace that enforces royalties
    public fun create_marketplace(
        fee_bps: u64,
        fee_recipient: address,
        ctx: &mut TxContext,
    ): MarketConfig {
        MarketConfig {
            id: object::new(ctx),
            fee_bps,
            fee_recipient,
        }
    }
    
    /// Add royalty rule to a TransferPolicy
    /// Called by collection creator when setting up their NFT
    public fun add_royalty_rule<NFT: key + store>(
        policy: &mut TransferPolicy<NFT>,
        policy_cap: &transfer_policy::TransferPolicyCap<NFT>,
        amount_bps: u64,
        recipient: address,
        ctx: &mut TxContext,
    ) {
        let config = RoyaltyConfig {
            id: object::new(ctx),
            amount_bps,
            recipient,
        };
        
        transfer_policy::add_rule(
            RoyaltyRule {},
            policy,
            policy_cap,
            config,
        );
    }
    
    /// Satisfy royalty rule: buyer pays royalty as part of purchase
    public fun pay_royalty<NFT: key + store>(
        policy: &mut TransferPolicy<NFT>,
        request: &mut TransferRequest<NFT>,
        payment: &mut Coin<SUI>,
        ctx: &mut TxContext,
    ) {
        let config: &RoyaltyConfig = transfer_policy::get_rule(RoyaltyRule {}, policy);
        let price = transfer_policy::paid(request);
        
        let royalty_amount = price * config.amount_bps / 10_000;
        let royalty_payment = coin::split(payment, royalty_amount, ctx);
        
        // Pay royalty to creator
        sui::transfer::public_transfer(royalty_payment, config.recipient);
        
        // Mark rule as satisfied
        transfer_policy::add_receipt(RoyaltyRule {}, request);
    }
    
    /// Full purchase flow: buy from kiosk + pay royalty
    public fun purchase_with_royalty<NFT: key + store>(
        kiosk: &mut Kiosk,
        kiosk_cap: &KioskOwnerCap,
        policy: &mut TransferPolicy<NFT>,
        item_id: object::ID,
        payment: &mut Coin<SUI>,
        ctx: &mut TxContext,
    ): NFT {
        // Buy from kiosk (creates a TransferRequest)
        let (item, mut transfer_request) = kiosk::purchase<NFT>(
            kiosk,
            item_id,
            coin::split(payment, kiosk::listed_price(kiosk, item_id), ctx),
        );
        
        // Satisfy royalty rule
        pay_royalty<NFT>(policy, &mut transfer_request, payment, ctx);
        
        // Confirm all rules satisfied (transaction fails if not)
        transfer_policy::confirm_request(policy, transfer_request);
        
        let _ = kiosk_cap;
        
        event::emit(NFTSold {
            kiosk: object::uid_to_address(kiosk::kiosk_uid(kiosk)),
            item_id: object::id_to_address(&item_id),
            price: kiosk::listed_price(kiosk, item_id),
            buyer: tx_context::sender(ctx),
        });
        
        item
    }
    
    /// Batch buy multiple NFTs from same kiosk in one PTB
    /// (Called via Programmable Transaction Block)
    public fun batch_purchase<NFT: key + store>(
        kiosk: &mut Kiosk,
        policy: &mut TransferPolicy<NFT>,
        item_ids: vector<object::ID>,
        payment: &mut Coin<SUI>,
        ctx: &mut TxContext,
    ): vector<NFT> {
        let items = vector::empty<NFT>();
        let n = vector::length(&item_ids);
        let i = 0;
        
        while (i < n) {
            let item_id = *vector::borrow(&item_ids, i);
            let price = kiosk::listed_price(kiosk, item_id);
            let item_payment = coin::split(payment, price, ctx);
            
            let (item, mut request) = kiosk::purchase<NFT>(kiosk, item_id, item_payment);
            pay_royalty<NFT>(policy, &mut request, payment, ctx);
            transfer_policy::confirm_request(policy, request);
            
            vector::push_back(&mut items, item);
            i = i + 1;
        };
        
        items
    }
}
```

---

## Auction Mechanism

```move
// ============================================
// ENGLISH AUCTION: Time-based ascending price
// ============================================

module nft_market::auction {
    use aptos_framework::timestamp;
    
    const ERR_AUCTION_ENDED: u64 = 1;
    const ERR_BID_TOO_LOW: u64 = 2;
    const ERR_AUCTION_LIVE: u64 = 3;
    const ERR_NOT_SELLER: u64 = 4;
    const ERR_NO_BIDS: u64 = 5;
    
    const MIN_BID_INCREMENT_BPS: u64 = 500;  // 5% minimum bid increase
    const EXTENSION_SECS: u64 = 300;         // Extend 5 min if bid in last 5 min
    
    struct Auction has key {
        seller: address,
        token_addr: address,
        
        // Auction parameters
        start_price: u64,
        reserve_price: u64,    // Secret min to close (0 = no reserve)
        end_time: u64,         // Unix timestamp (microseconds)
        
        // Current state
        highest_bid: u64,
        highest_bidder: address,
        bid_count: u64,
        
        // Escrowed funds
        // Highest bid held in escrow until outbid or auction ends
    }
    
    public entry fun create_auction(
        seller: &signer,
        token_addr: address,
        start_price: u64,
        reserve_price: u64,
        duration_hours: u64,
    ) {
        let seller_addr = std::signer::address_of(seller);
        let now = timestamp::now_microseconds();
        
        // Transfer NFT to escrow
        // object::transfer(seller, token, @auction_escrow);
        
        move_to(seller, Auction {
            seller: seller_addr,
            token_addr,
            start_price,
            reserve_price,
            end_time: now + duration_hours * 3_600_000_000,
            highest_bid: 0,
            highest_bidder: @0x0,
            bid_count: 0,
        });
    }
    
    public entry fun place_bid(
        bidder: &signer,
        auction_addr: address,
        bid_amount: u64,
    ) {
        let auction = borrow_global_mut<Auction>(auction_addr);
        let now = timestamp::now_microseconds();
        
        assert!(now < auction.end_time, ERR_AUCTION_ENDED);
        
        // Minimum bid: max(start_price, highest_bid * 1.05)
        let min_bid = if (auction.highest_bid == 0) {
            auction.start_price
        } else {
            auction.highest_bid + auction.highest_bid * MIN_BID_INCREMENT_BPS / 10_000
        };
        
        assert!(bid_amount >= min_bid, ERR_BID_TOO_LOW);
        
        // Return previous highest bidder's escrow
        if (auction.highest_bidder != @0x0) {
            // coin::transfer(escrow, auction.highest_bidder, auction.highest_bid);
        };
        
        // Escrow new bid
        // coin::transfer(bidder, @auction_escrow, bid_amount);
        
        // Anti-sniping: extend if bid in last 5 minutes
        let time_remaining = auction.end_time - now;
        if (time_remaining < EXTENSION_SECS * 1_000_000) {
            auction.end_time = auction.end_time + EXTENSION_SECS * 1_000_000;
        };
        
        let bidder_addr = std::signer::address_of(bidder);
        auction.highest_bid = bid_amount;
        auction.highest_bidder = bidder_addr;
        auction.bid_count = auction.bid_count + 1;
    }
    
    public entry fun settle_auction(
        anyone: &signer,
        auction_addr: address,
    ) {
        let auction = move_from<Auction>(auction_addr);
        let now = timestamp::now_microseconds();
        
        assert!(now >= auction.end_time, ERR_AUCTION_LIVE);
        
        if (auction.bid_count == 0 || auction.highest_bid < auction.reserve_price) {
            // No valid bids: return NFT to seller
            // object::transfer(escrow, token, auction.seller);
        } else {
            // Winner gets NFT
            // object::transfer(escrow, token, auction.highest_bidder);
            
            // Seller gets proceeds (minus fees)
            let market_fee = auction.highest_bid * 250 / 10_000;  // 2.5%
            let seller_proceeds = auction.highest_bid - market_fee;
            // coin::transfer(escrow, auction.seller, seller_proceeds);
            // coin::transfer(escrow, @fee_recipient, market_fee);
            let _ = seller_proceeds;
        };
        
        let _ = anyone;
        
        let Auction { seller: _, token_addr: _, start_price: _, reserve_price: _, 
                     end_time: _, highest_bid: _, highest_bidder: _, bid_count: _ } = auction;
    }
    
    #[view]
    public fun get_auction_state(auction_addr: address): (u64, u64, address, u64) {
        let auction = borrow_global<Auction>(auction_addr);
        let now = timestamp::now_microseconds();
        let time_remaining = if (now < auction.end_time) {
            auction.end_time - now
        } else { 0 };
        
        (auction.highest_bid, time_remaining, auction.highest_bidder, auction.bid_count)
    }
}
```

---

## Collection Analytics

```typescript
// ============================================
// NFT COLLECTION ANALYTICS
// Track floor price, volume, whale wallets
// ============================================

interface CollectionStats {
  collectionAddr: string;
  floorPrice: bigint;
  ceilingPrice: bigint;
  volume24h: bigint;
  volume7d: bigint;
  sales24h: number;
  uniqueHolders: number;
  totalSupply: number;
  listedCount: number;
  listingRate: number;    // % of supply listed
  averageHoldTime: number;  // Days
  whaleConcentration: number;  // % held by top 10 holders
}

class NFTCollectionAnalytics {
  constructor(private db: any) {}
  
  async getCollectionStats(collectionAddr: string): Promise<CollectionStats> {
    const [floorResult, volumeResult, holderResult] = await Promise.all([
      this.db.query(`
        SELECT MIN(price) as floor, MAX(price) as ceiling
        FROM listings
        WHERE collection_addr = $1 AND is_active = true
      `, [collectionAddr]),
      
      this.db.query(`
        SELECT 
          SUM(CASE WHEN created_at > NOW() - INTERVAL '24 hours' THEN price ELSE 0 END) as vol_24h,
          SUM(CASE WHEN created_at > NOW() - INTERVAL '7 days' THEN price ELSE 0 END) as vol_7d,
          COUNT(CASE WHEN created_at > NOW() - INTERVAL '24 hours' THEN 1 END) as sales_24h
        FROM sales
        WHERE collection_addr = $1
      `, [collectionAddr]),
      
      this.db.query(`
        SELECT 
          COUNT(DISTINCT owner_addr) as unique_holders,
          COUNT(*) as total_tokens,
          SUM(CASE WHEN is_listed THEN 1 ELSE 0 END) as listed_count
        FROM nft_ownership
        WHERE collection_addr = $1
      `, [collectionAddr]),
    ]);
    
    const holders = holderResult.rows[0];
    const volume = volumeResult.rows[0];
    const floor = floorResult.rows[0];
    
    // Whale concentration: % held by top 10 holders
    const whaleResult = await this.db.query(`
      SELECT SUM(token_count) as whale_count
      FROM (
        SELECT owner_addr, COUNT(*) as token_count
        FROM nft_ownership
        WHERE collection_addr = $1
        GROUP BY owner_addr
        ORDER BY token_count DESC
        LIMIT 10
      ) top_holders
    `, [collectionAddr]);
    
    const whaleConcentration = parseInt(holders.total_tokens) > 0
      ? parseInt(whaleResult.rows[0].whale_count) / parseInt(holders.total_tokens) * 100
      : 0;
    
    return {
      collectionAddr,
      floorPrice: BigInt(floor.floor || 0),
      ceilingPrice: BigInt(floor.ceiling || 0),
      volume24h: BigInt(volume.vol_24h || 0),
      volume7d: BigInt(volume.vol_7d || 0),
      sales24h: parseInt(volume.sales_24h || 0),
      uniqueHolders: parseInt(holders.unique_holders),
      totalSupply: parseInt(holders.total_tokens),
      listedCount: parseInt(holders.listed_count),
      listingRate: parseInt(holders.total_tokens) > 0
        ? parseInt(holders.listed_count) / parseInt(holders.total_tokens) * 100 : 0,
      averageHoldTime: 0,  // Compute from transfer history
      whaleConcentration,
    };
  }
  
  // Detect wash trading: same buyer/seller within 24h
  async detectWashTrading(collectionAddr: string): Promise<string[]> {
    const result = await this.db.query(`
      SELECT s1.buyer_addr, s1.seller_addr, COUNT(*) as trade_count
      FROM sales s1
      JOIN sales s2 ON s1.buyer_addr = s2.seller_addr 
                    AND s1.seller_addr = s2.buyer_addr
                    AND ABS(EXTRACT(EPOCH FROM (s1.created_at - s2.created_at))) < 86400
      WHERE s1.collection_addr = $1
      GROUP BY s1.buyer_addr, s1.seller_addr
      HAVING COUNT(*) >= 3
    `, [collectionAddr]);
    
    return result.rows.map((r: any) => r.buyer_addr);
  }
  
  // Rarity score: higher is rarer
  async computeRarityScores(collectionAddr: string): Promise<Map<string, number>> {
    // Get trait frequencies
    const traitsResult = await this.db.query(`
      SELECT trait_type, trait_value, COUNT(*) as count
      FROM nft_traits
      WHERE collection_addr = $1
      GROUP BY trait_type, trait_value
    `, [collectionAddr]);
    
    const totalSupply = await this.db.query(
      'SELECT COUNT(*) as count FROM nft_ownership WHERE collection_addr = $1',
      [collectionAddr]
    );
    const total = parseInt(totalSupply.rows[0].count);
    
    // Rarity formula: sum of -log2(trait_frequency) for each trait
    const traitFrequencies = new Map<string, number>();
    traitsResult.rows.forEach((row: any) => {
      const key = `${row.trait_type}:${row.trait_value}`;
      traitFrequencies.set(key, parseInt(row.count) / total);
    });
    
    // Score each token
    const tokenTraits = await this.db.query(
      'SELECT token_addr, trait_type, trait_value FROM nft_traits WHERE collection_addr = $1',
      [collectionAddr]
    );
    
    const scores = new Map<string, number>();
    tokenTraits.rows.forEach((row: any) => {
      const key = `${row.trait_type}:${row.trait_value}`;
      const freq = traitFrequencies.get(key) || 1;
      const rarityScore = -Math.log2(freq);
      scores.set(row.token_addr, (scores.get(row.token_addr) || 0) + rarityScore);
    });
    
    return scores;
  }
}
```

---

## สรุป NFT Marketplace Protocol

```
NFT MARKETPLACE DESIGN DECISIONS

1. ROYALTY ENFORCEMENT
   Aptos: Application-level (marketplace enforces)
          Risk: Marketplace can skip royalties
   Sui Kiosk: Protocol-level (TransferPolicy)
          Risk: Cannot bypass → 100% enforcement
   
   ✅ Prefer Sui Kiosk for creator royalties
   ✅ Aptos: Use transfer approval (object::enable_ungated_transfer = false)

2. LISTING MECHANISMS
   Fixed Price: Simplest, immediate sale
   Dutch Auction: Price decreases over time (sell faster)
   English Auction: Price increases, anti-snipe extension
   Sealed Bid: Private bids, reveals at end

3. ANTI-PATTERNS TO AVOID
   ❌ Floor sweep bots: Use min_price protection
   ❌ Wash trading: Detect buyer/seller cycles
   ❌ Fake bids: Escrow payment, verify funds
   ❌ Rug pulls: Lock creator access or use escrow
   ❌ Sniping: 5-minute extension on last-minute bids

4. ROYALTY CALCULATION
   royalty = price * royalty_bps / 10_000
   Always: market_fee + royalty + proceeds = price
   Enforce: proceeds cannot go negative

5. COLLECTION HEALTH METRICS
   Floor Price: Baseline NFT value
   Volume/TVL Ratio: Activity level
   Listing Rate: >30% = lots of sellers (bearish)
   Whale Concentration: >50% by top 10 = risky
   Wash Trade Ratio: >20% volume = manipulated

6. RARITY SCORING
   Rarity Score = Σ -log2(trait_frequency)
   Higher score = rarer token = premium price
   Normalize: z-score relative to collection
```

---

**ก่อนหน้า**: [Part 91 - Yield Aggregator ←](part-91-yield-aggregator.md)
**ต่อไป**: [Part 93 - Advanced Governance Systems →](part-93-governance.md)
