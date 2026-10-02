# Part 77: Advanced Sui Move Patterns

## สารบัญ
- [Sui Object Model Deep Dive](#sui-object-model-deep-dive)
- [Programmable Transaction Blocks (PTB)](#programmable-transaction-blocks-ptb)
- [Dynamic Fields Advanced Patterns](#dynamic-fields-advanced-patterns)
- [Kiosk & Transfer Policy](#kiosk--transfer-policy)
- [Sui Clock & Randomness](#sui-clock--randomness)
- [Display Standard](#display-standard)

---

## Sui Object Model Deep Dive

```
Sui's Object Model:

4 OWNERSHIP TYPES
  1. OWNED (single owner):
     - One address or object owns it
     - Only owner can use it in transactions
     - Fast: no consensus needed (validator knows owner)
     
  2. SHARED (shared state):
     - Accessible to anyone
     - Requires consensus (sequencing)
     - Slower than owned but necessary for pools, DAOs
     
  3. FROZEN (immutable):
     - No one can modify
     - Anyone can read
     - Good for: configuration, constants, approved lists
     
  4. WRAPPED (inside another object):
     - Cannot be transferred or accessed directly
     - Only accessible through parent object's methods

OBJECT STRUCTURE
  Every Sui object has:
    id: UID          (unique identifier, non-copyable, non-droppable)
    version: u64     (increments on every modification)
    digest: [u8;32]  (hash of last modification)

When to use each ownership type:
  DEX Pool:        shared (anyone needs to swap)
  User balance:    owned (only user moves it)
  NFT:             owned (user's asset)
  Protocol config: frozen (read-only constants)
  Staking receipt: owned (tracks user's stake)
  Voting power:    owned or shared depending on design
  
Key Insight: Owned objects DON'T need global consensus!
  → Dramatically lower latency for personal assets
  → Sui can process owned-object txs in parallel
```

---

## Programmable Transaction Blocks (PTB)

```typescript
// ============================================
// PTB: Compose multiple operations atomically
// Like a mini-program in one transaction
// ============================================

import { 
  Transaction,
  SuiClient,
  Ed25519Keypair
} from '@mysten/sui/client';

const client = new SuiClient({ url: 'https://fullnode.mainnet.sui.io' });

// ============================================
// EXAMPLE 1: Complex DeFi Operation
// 1. Borrow flash loan
// 2. Swap on DEX A
// 3. Swap on DEX B  
// 4. Repay flash loan
// All in ONE transaction!
// ============================================

async function flashArbitrage(keypair: Ed25519Keypair) {
  const tx = new Transaction();
  
  // Split some SUI for gas
  const [gas] = tx.splitCoins(tx.gas, [100_000_000n]);
  
  // Step 1: Borrow USDC via flash loan
  const [borrowedUSDC, receipt] = tx.moveCall({
    target: '0xFlashLoan::pool::flash_borrow',
    typeArguments: ['0x2::sui::SUI', '0xUSDC::usdc::USDC'],
    arguments: [
      tx.object('0xFlashPoolAddr'),  // Pool object
      tx.pure.u64(1_000_000_000n),   // Borrow 1000 USDC
    ],
  });
  
  // Step 2: Swap USDC → APT on Cetus DEX
  const [suiOut] = tx.moveCall({
    target: '0xCetus::pool::swap',
    typeArguments: ['0xUSDC::usdc::USDC', '0x2::sui::SUI'],
    arguments: [
      tx.object('0xCetusPoolSUI_USDC'),
      borrowedUSDC,  // Pass borrowed USDC directly!
      tx.pure.bool(true),  // a2b direction
      tx.pure.u64(0n),     // min out
    ],
  });
  
  // Step 3: Sell SUI at higher price on another DEX
  const [usdcOut] = tx.moveCall({
    target: '0xTurbos::pool::swap',
    typeArguments: ['0x2::sui::SUI', '0xUSDC::usdc::USDC'],
    arguments: [
      tx.object('0xTurbosPoolSUI_USDC'),
      suiOut,
      tx.pure.bool(true),
      tx.pure.u64(0n),
    ],
  });
  
  // Step 4: Repay flash loan (must happen or whole tx reverts)
  tx.moveCall({
    target: '0xFlashLoan::pool::flash_repay',
    typeArguments: ['0xUSDC::usdc::USDC'],
    arguments: [
      usdcOut,
      receipt,  // Must consume receipt
    ],
  });
  
  // Execute
  const result = await client.signAndExecuteTransaction({
    signer: keypair,
    transaction: tx,
    options: { showEffects: true },
  });
  
  return result;
}

// ============================================
// EXAMPLE 2: Batch NFT Operations
// Mint 10 NFTs and list all on marketplace
// ============================================

async function batchMintAndList(
  keypair: Ed25519Keypair,
  collectionObj: string,
  count: number,
  price: bigint,
) {
  const tx = new Transaction();
  
  const nfts = [];
  for (let i = 0; i < count; i++) {
    const [nft] = tx.moveCall({
      target: '0xMyNFT::collection::mint',
      arguments: [
        tx.object(collectionObj),
        tx.pure.string(`NFT #${i}`),
        tx.pure.string(`https://.../${i}.json`),
      ],
    });
    nfts.push(nft);
  }
  
  // List each NFT on Kiosk
  const [kiosk, kioskCap] = tx.moveCall({
    target: '0x2::kiosk::new',
    arguments: [],
  });
  
  for (const nft of nfts) {
    tx.moveCall({
      target: '0x2::kiosk::place_and_list',
      typeArguments: ['0xMyNFT::collection::NFT'],
      arguments: [
        kiosk,
        kioskCap,
        nft,
        tx.pure.u64(price),
      ],
    });
  }
  
  // Share the kiosk
  tx.moveCall({
    target: '0x2::transfer::share_object',
    arguments: [kiosk],
  });
  
  // Transfer kiosk cap to sender
  tx.transferObjects([kioskCap], keypair.getPublicKey().toSuiAddress());
  
  const result = await client.signAndExecuteTransaction({
    signer: keypair,
    transaction: tx,
    options: { showEffects: true, showObjectChanges: true },
  });
  
  return result;
}

// ============================================
// EXAMPLE 3: Merge Coins + Send
// Aggregate multiple small SUI coins into one
// ============================================

async function mergeAndSend(
  keypair: Ed25519Keypair,
  coinObjectIds: string[],
  recipient: string,
  sendAmount: bigint,
) {
  const tx = new Transaction();
  
  // Merge all coins except the first
  const [primaryCoin, ...otherCoins] = coinObjectIds.map(id => tx.object(id));
  
  if (otherCoins.length > 0) {
    tx.mergeCoins(primaryCoin, otherCoins);
  }
  
  // Split exact amount to send
  const [toSend] = tx.splitCoins(primaryCoin, [sendAmount]);
  
  // Transfer to recipient
  tx.transferObjects([toSend], recipient);
  
  return client.signAndExecuteTransaction({
    signer: keypair,
    transaction: tx,
  });
}
```

---

## Dynamic Fields Advanced Patterns

```move
// Sui Move dynamic fields
module sui_patterns::dynamic_bag {
    use sui::dynamic_field as df;
    use sui::dynamic_object_field as dof;
    
    // ============================================
    // PATTERN: DYNAMIC REGISTRY
    // Store arbitrary objects by key
    // Useful for: plugin system, extensible protocols
    // ============================================
    
    struct Registry has key {
        id: sui::object::UID,
    }
    
    // Register any object type
    public fun register<K: copy + drop + store, V: key + store>(
        registry: &mut Registry,
        key: K,
        value: V,
    ) {
        // dof: preserves object identity (keeps object's ID)
        dof::add(&mut registry.id, key, value);
    }
    
    // Retrieve registered object
    public fun get<K: copy + drop + store, V: key + store>(
        registry: &Registry,
        key: K,
    ): &V {
        dof::borrow(&registry.id, key)
    }
    
    public fun get_mut<K: copy + drop + store, V: key + store>(
        registry: &mut Registry,
        key: K,
    ): &mut V {
        dof::borrow_mut(&mut registry.id, key)
    }
    
    // Remove from registry
    public fun remove<K: copy + drop + store, V: key + store>(
        registry: &mut Registry,
        key: K,
    ): V {
        dof::remove(&mut registry.id, key)
    }
    
    // Check existence
    public fun exists_<K: copy + drop + store>(
        registry: &Registry,
        key: K,
    ): bool {
        dof::exists_(&registry.id, key)
    }
    
    // ============================================
    // PATTERN: LINKED LIST using dynamic fields
    // ============================================
    
    struct LinkedList has key {
        id: sui::object::UID,
        head: u64,   // Key of first element
        tail: u64,   // Key of last element
        size: u64,
        next_key: u64,  // Auto-incrementing key
    }
    
    struct Node<T: store> has store {
        value: T,
        next: u64,   // 0 if tail
        prev: u64,   // 0 if head
    }
    
    public fun push_back<T: store>(list: &mut LinkedList, value: T): u64 {
        let key = list.next_key;
        list.next_key = list.next_key + 1;
        
        let node = Node { value, next: 0, prev: list.tail };
        
        if (list.size > 0) {
            // Update old tail's next pointer
            let old_tail = df::borrow_mut::<u64, Node<T>>(&mut list.id, list.tail);
            old_tail.next = key;
        } else {
            // First element
            list.head = key;
        };
        
        df::add(&mut list.id, key, node);
        list.tail = key;
        list.size = list.size + 1;
        
        key
    }
    
    public fun pop_front<T: store>(list: &mut LinkedList): T {
        assert!(list.size > 0, 1);
        
        let head_key = list.head;
        let Node { value, next, prev: _ } = df::remove::<u64, Node<T>>(&mut list.id, head_key);
        
        list.head = next;
        list.size = list.size - 1;
        
        if (list.size == 0) {
            list.tail = 0;
        } else {
            let new_head = df::borrow_mut::<u64, Node<T>>(&mut list.id, next);
            new_head.prev = 0;
        };
        
        value
    }
}
```

---

## Kiosk & Transfer Policy

```move
// Sui Kiosk: NFT marketplace primitive
// Handles: royalties, allowlisting, custom transfer logic

module sui_patterns::marketplace {
    use sui::kiosk::{Self, Kiosk, KioskOwnerCap};
    use sui::transfer_policy::{Self, TransferPolicy, TransferPolicyCap};
    use sui::coin::Coin;
    use sui::sui::SUI;
    
    // ============================================
    // SETUP ROYALTY POLICY
    // 5% royalty on every sale
    // ============================================
    
    struct NFT has key, store {
        id: sui::object::UID,
        name: std::string::String,
        image_url: std::string::String,
        creator: address,
    }
    
    // Publisher creates transfer policy (one-time setup)
    public fun setup_royalty_policy(
        publisher: &sui::package::Publisher,
        ctx: &mut sui::tx_context::TxContext,
    ): (TransferPolicy<NFT>, TransferPolicyCap<NFT>) {
        let (policy, cap) = transfer_policy::new<NFT>(publisher, ctx);
        
        // Add royalty rule: 5% goes to creator
        // (Using Sui's built-in royalty extension)
        // kiosk_ext::royalty_rule::add(&mut policy, &cap, 500, 0); // 5% min 0 SUI
        
        (policy, cap)
    }
    
    // ============================================
    // SELLER: List NFT in kiosk
    // ============================================
    
    public fun list_nft(
        kiosk: &mut Kiosk,
        cap: &KioskOwnerCap,
        nft: NFT,
        price: u64,
    ): sui::object::ID {
        let nft_id = sui::object::id(&nft);
        
        // Place and list in one call
        kiosk::place_and_list<NFT>(kiosk, cap, nft, price);
        
        nft_id
    }
    
    // ============================================
    // BUYER: Purchase from kiosk
    // ============================================
    
    public fun purchase_nft(
        kiosk: &mut Kiosk,
        nft_id: sui::object::ID,
        payment: Coin<SUI>,
        policy: &mut TransferPolicy<NFT>,
        ctx: &mut sui::tx_context::TxContext,
    ) {
        // Buy: takes coin, gives NFT + transfer request
        let (nft, mut request) = kiosk::purchase<NFT>(kiosk, nft_id, payment);
        
        // Satisfy all policy rules (royalties, allowlists, etc.)
        // For royalty rule: pay royalty
        // royalty_rule::pay(&mut policy, &mut request, royalty_payment);
        
        // Confirm all rules satisfied → policy allows transfer
        let (nft_out, _paid) = transfer_policy::confirm_request(policy, request);
        
        // Transfer NFT to buyer
        sui::transfer::public_transfer(nft_out, sui::tx_context::sender(ctx));
    }
    
    // ============================================
    // PROTOCOL: Take marketplace fee from kiosk
    // ============================================
    
    public fun take_profits(
        kiosk: &mut Kiosk,
        cap: &KioskOwnerCap,
        ctx: &mut sui::tx_context::TxContext,
    ) {
        let profits = kiosk::withdraw(kiosk, cap, std::option::none(), ctx);
        sui::transfer::public_transfer(profits, sui::tx_context::sender(ctx));
    }
}
```

---

## Sui Clock & Randomness

```move
// Sui-specific: Clock object and on-chain randomness

module sui_patterns::time_and_random {
    use sui::clock::{Self, Clock};
    use sui::random::{Self, Random, RandomGenerator};
    
    // ============================================
    // SUI CLOCK: Getting timestamps in Sui Move
    // Must pass Clock as an argument (not global state)
    // ============================================
    
    struct AuctionBid has key {
        id: sui::object::UID,
        bidder: address,
        amount: u64,
        bid_time: u64,
    }
    
    public fun place_bid(
        clock: &Clock,           // Pass clock object
        bid_amount: u64,
        deadline: u64,           // Unix timestamp in ms
        ctx: &mut sui::tx_context::TxContext,
    ) {
        let now = clock::timestamp_ms(clock);
        
        // Check bid is within auction window
        assert!(now <= deadline, 1);
        
        let bid = AuctionBid {
            id: sui::object::new(ctx),
            bidder: sui::tx_context::sender(ctx),
            amount: bid_amount,
            bid_time: now,
        };
        
        sui::transfer::share_object(bid);
    }
    
    // ============================================
    // SUI RANDOMNESS (Native, since Sui v1.15+)
    // Verifiable, unpredictable randomness
    // ============================================
    
    struct LotteryTicket has key {
        id: sui::object::UID,
        number: u32,
        owner: address,
    }
    
    // Buy lottery ticket with random number
    // Random object is a shared system object
    public fun buy_ticket(
        random: &Random,
        ctx: &mut sui::tx_context::TxContext,
    ): LotteryTicket {
        let mut generator = random::new_generator(random, ctx);
        
        let ticket_number: u32 = random::generate_u32(&mut generator);
        
        LotteryTicket {
            id: sui::object::new(ctx),
            number: ticket_number,
            owner: sui::tx_context::sender(ctx),
        }
    }
    
    // Draw winner (compare tickets)
    public fun draw_winner(
        random: &Random,
        tickets: vector<LotteryTicket>,
        ctx: &mut sui::tx_context::TxContext,
    ): address {
        let n = std::vector::length(&tickets);
        assert!(n > 0, 1);
        
        let mut generator = random::new_generator(random, ctx);
        
        // Pick random index
        let winner_idx = random::generate_u64_in_range(&mut generator, 0, n - 1);
        
        let winner_ticket = std::vector::borrow(&tickets, winner_idx);
        let winner = winner_ticket.owner;
        
        // Destroy all tickets
        let mut i = 0u64;
        while (i < n) {
            let LotteryTicket { id, number: _, owner: _ } = std::vector::pop_back(&mut tickets);
            sui::object::delete(id);
            i = i + 1;
        };
        std::vector::destroy_empty(tickets);
        
        winner
    }
    
    // Fisher-Yates shuffle using Sui randomness
    public fun shuffle<T>(
        random: &Random,
        items: &mut vector<T>,
        ctx: &mut sui::tx_context::TxContext,
    ) {
        let mut generator = random::new_generator(random, ctx);
        let n = std::vector::length(items);
        
        let mut i = n - 1;
        while (i > 0) {
            let j = random::generate_u64_in_range(&mut generator, 0, i);
            std::vector::swap(items, i, j);
            i = i - 1;
        };
    }
}
```

---

## Display Standard

```move
// Sui Display: On-chain NFT metadata standard
// Protocol to define how wallets/explorers show objects

module sui_patterns::display_nft {
    use sui::display;
    use sui::package;
    
    // ============================================
    // SUI DISPLAY STANDARD
    // Define how NFTs appear in wallets/marketplaces
    // ============================================
    
    struct GameCard has key, store {
        id: sui::object::UID,
        name: std::string::String,
        description: std::string::String,
        image_url: std::string::String,
        rarity: u8,         // 1=Common, 2=Rare, 3=Epic, 4=Legendary
        attack: u64,
        defense: u64,
        level: u64,
    }
    
    // OTW (One-Time Witness) for publisher
    struct DISPLAY_NFT has drop {}
    
    fun init(otw: DISPLAY_NFT, ctx: &mut sui::tx_context::TxContext) {
        // Create publisher from OTW
        let publisher = package::claim(otw, ctx);
        
        // Create display object
        let mut display = display::new_with_fields<GameCard>(
            &publisher,
            // Field names to expose
            vector[
                std::string::utf8(b"name"),
                std::string::utf8(b"description"),
                std::string::utf8(b"image_url"),
                std::string::utf8(b"rarity"),
                std::string::utf8(b"attack"),
                std::string::utf8(b"defense"),
                std::string::utf8(b"level"),
            ],
            // Field values (can use {field_name} templates!)
            vector[
                std::string::utf8(b"{name}"),       // Direct field value
                std::string::utf8(b"{description}"),
                std::string::utf8(b"https://cdn.mygame.io/cards/{id}.png"),  // Dynamic URL!
                std::string::utf8(b"{rarity}"),
                std::string::utf8(b"{attack}"),
                std::string::utf8(b"{defense}"),
                std::string::utf8(b"{level}"),
            ],
            ctx,
        );
        
        // Update version to publish
        display::update_version(&mut display);
        
        // Share display object
        sui::transfer::public_share_object(display);
        
        // Transfer publisher to deployer
        sui::transfer::public_transfer(publisher, sui::tx_context::sender(ctx));
    }
    
    // Mint a GameCard
    public fun mint(
        name: std::string::String,
        description: std::string::String,
        rarity: u8,
        attack: u64,
        defense: u64,
        ctx: &mut sui::tx_context::TxContext,
    ): GameCard {
        GameCard {
            id: sui::object::new(ctx),
            name,
            description,
            image_url: std::string::utf8(b""), // Constructed from display template
            rarity,
            attack,
            defense,
            level: 1,
        }
    }
    
    // Level up a card
    public fun level_up(card: &mut GameCard) {
        card.level = card.level + 1;
        card.attack = card.attack + 10;
        card.defense = card.defense + 5;
    }
}
```

---

## สรุป Sui Advanced Patterns

```
Sui vs Aptos Architecture Summary:

OBJECT MODEL
  Sui: Objects with UID, 4 ownership types
  Aptos: Resources stored at addresses, 2 types (exists/not)
  
  Sui advantage: Owned objects bypass consensus → very fast
  Aptos advantage: Simpler mental model, easier migration from Solidity

TRANSACTION COMPOSITION
  Sui PTBs: Rich composition within one tx (pass results between calls)
  Aptos Entry Functions: Limited composition (no inline result passing)
  
  Sui advantage: Flash loans, complex DeFi without helper contracts
  Aptos advantage: Simpler transaction model, easier tooling

DYNAMIC STORAGE
  Sui: Dynamic fields (df/dof) for arbitrary key-value in objects
  Aptos: SmartTable, Table for hash-map storage
  
  Both: Avoid unbounded vector iterations

RANDOMNESS
  Sui: Native Random object (BLS-based, verifiable)
  Aptos: aptos_framework::randomness (similar approach)
  
  Both: Better than Ethereum (which had no native randomness)

MARKETPLACE/NFT
  Sui: Kiosk standard (protocol-level royalties, policy enforcement)
  Aptos: Digital Asset standard (constructor refs, refs pattern)
  
  Sui advantage: Royalties enforced at protocol level (no workarounds)

WHEN TO USE SUI
  ✅ NFT projects (Kiosk royalties, Display standard)
  ✅ Games (fast owned-object txs, randomness)
  ✅ Complex DeFi (PTB composition)
  ✅ Real-world assets (fine-grained ownership)
  
WHEN TO USE APTOS
  ✅ DeFi protocols (simpler model)
  ✅ Identity/payments (account model familiar to users)
  ✅ Ecosystem integrations (more DeFi TVL currently)
  ✅ Enterprise (account-based familiar to institutions)
```

---

**ก่อนหน้า**: [Part 76 - Security Auditing ←](part-76-security-auditing.md)
**ต่อไป**: [Part 78 - Cross-Chain Protocol Design →](part-78-crosschain-protocol.md)
