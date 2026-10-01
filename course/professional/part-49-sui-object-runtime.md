# Part 49: Sui Object Runtime Deep Dive

## สารบัญ
- [Sui Object Model](#sui-object-model)
- [Object Ownership Types](#object-ownership-types)
- [Programmable Transaction Blocks](#programmable-transaction-blocks)
- [Dynamic Object System](#dynamic-object-system)
- [Shared Object Contention & Performance](#shared-object-contention--performance)
- [ตัวอย่าง: Advanced Sui Patterns](#ตัวอย่าง-advanced-sui-patterns)

---

## Sui Object Model

```
Sui vs Aptos Memory Model:

Aptos (Account-based):
  Global State = Map<Address, Map<ResourceType, Resource>>
  - Resources stored at an address
  - Access: borrow_global<T>(addr)
  - All state at one address = sequential access

Sui (Object-based):
  Global State = Map<ObjectID, Object>
  - Objects are independent entities
  - Each has unique ID, version, owner
  - Parallelism: no shared state needed for owned objects
  
Sui Object Structure:
  struct Object {
    id: UID,             // Unique, never reused
    owner: Owner,        // Who controls this object
    version: VersionNumber,  // Incremented on each tx
    data: ObjectData,   // The actual Move struct
  }

UID Properties:
  - Generated deterministically from tx digest + creation order
  - Never reused even after deletion
  - ObjectID = first 20 bytes of UID hash
  
Benefits:
  - Parallel execution for non-overlapping objects
  - No address-based storage (objects move freely)
  - Better for complex ownership patterns
```

---

## Object Ownership Types

```move
module sui_advanced::ownership {
    use sui::object::{Self, UID};
    use sui::tx_context::{Self, TxContext};
    use sui::transfer;
    
    // ============================================
    // 1. Owned Objects (by address or object)
    // ============================================
    
    struct MyNFT has key {
        id: UID,
        name: std::string::String,
        value: u64,
    }
    
    // Transfer to user address (owned by user)
    public entry fun mint_nft(
        recipient: address,
        name: std::string::String,
        ctx: &mut TxContext,
    ) {
        let nft = MyNFT {
            id: object::new(ctx),
            name,
            value: 100,
        };
        transfer::transfer(nft, recipient);  // → user-owned
    }
    
    // Transfer ownership: user passes their NFT
    public entry fun gift_nft(nft: MyNFT, recipient: address) {
        transfer::transfer(nft, recipient);
    }
    
    // ============================================
    // 2. Shared Objects (accessible to all)
    // ============================================
    
    struct SharedPool has key {
        id: UID,
        total: u64,
    }
    
    public fun create_shared_pool(ctx: &mut TxContext) {
        let pool = SharedPool {
            id: object::new(ctx),
            total: 0,
        };
        transfer::share_object(pool);  // → shared (anyone can use)
    }
    
    // Shared objects require consensus → sequential
    public entry fun add_to_pool(pool: &mut SharedPool, amount: u64) {
        pool.total = pool.total + amount;
    }
    
    // ============================================
    // 3. Frozen/Immutable Objects (read-only forever)
    // ============================================
    
    struct ImmutableConfig has key {
        id: UID,
        max_fee: u64,
        protocol_version: u64,
    }
    
    public fun create_config(ctx: &mut TxContext) {
        let config = ImmutableConfig {
            id: object::new(ctx),
            max_fee: 100,
            protocol_version: 1,
        };
        transfer::freeze_object(config);  // → immutable forever
    }
    
    // Can only read frozen objects, not write
    public fun read_max_fee(config: &ImmutableConfig): u64 {
        config.max_fee
    }
    
    // ============================================
    // 4. Object-owned Objects (wrapped or owned by another object)
    // ============================================
    
    struct Wallet has key {
        id: UID,
        owner: address,
    }
    
    struct WalletItem has key {
        id: UID,
        value: u64,
    }
    
    // Transfer item to be owned by wallet object
    public entry fun put_item_in_wallet(
        wallet: &Wallet,
        item: WalletItem,
    ) {
        transfer::transfer(item, object::uid_to_address(&wallet.id));
        // item.owner = wallet's address
        // Now item is only accessible via wallet
    }
    
    // ============================================
    // 5. Wrapped Objects (stored inside another object)
    // ============================================
    
    struct Container has key {
        id: UID,
        item: Option<WalletItem>,  // Wrapped: object stored in field
    }
    
    public entry fun wrap_item(
        container: &mut Container,
        item: WalletItem,
    ) {
        // Item disappears from global state, lives inside container
        container.item = option::some(item);
    }
    
    public entry fun unwrap_item(
        container: &mut Container,
        ctx: &TxContext,
    ) {
        let item = option::extract(&mut container.item);
        transfer::transfer(item, tx_context::sender(ctx));
    }
}
```

---

## Programmable Transaction Blocks

```
PTB (Programmable Transaction Block):
  - Single Sui transaction = sequence of commands
  - Commands can use outputs from previous commands
  - All-or-nothing: entire PTB succeeds or fails
  - Extremely gas efficient
  - Enables complex DeFi compositions

PTB Commands:
  1. TransferObjects: transfer objects to address
  2. SplitCoins: split one coin into multiple
  3. MergeCoins: merge multiple coins into one
  4. MoveCall: call a Move function
  5. MakeMoveVec: create a vector of objects
  6. Publish: publish new module
  7. Upgrade: upgrade module

PTB Result Types:
  - Pure values (not objects): u64, bool, address, vector<u8>
  - Objects: created/transferred during PTB

Example PTB (TypeScript SDK):
```

```typescript
import { Transaction } from '@mysten/sui/transactions';

const tx = new Transaction();

// 1. Split coin for exact payment
const [payment, changeBack] = tx.splitCoins(tx.gas, [
  tx.pure.u64(100_000_000n),  // 0.1 SUI for payment
]);

// 2. Use payment in DeFi operation
const [lpTokens] = tx.moveCall({
  target: `${PACKAGE_ID}::amm::add_liquidity`,
  arguments: [
    tx.object(POOL_ID),
    payment,
    tx.pure.u64(50_000_000n),  // min LP out
  ],
});

// 3. Stake LP tokens
tx.moveCall({
  target: `${PACKAGE_ID}::farm::stake`,
  arguments: [
    tx.object(FARM_ID),
    lpTokens,   // Output from step 2!
  ],
});

// 4. Return change to sender
tx.transferObjects([changeBack], tx.pure.address(sender));

// Sign and execute
const result = await client.signAndExecuteTransaction({
  transaction: tx,
  signer: keypair,
});
```

```move
// Move side: PTB-friendly function design
module sui_advanced::ptb_design {
    use sui::coin::{Self, Coin};
    use sui::balance::{Self, Balance};
    use sui::object::{Self, UID};
    use sui::tx_context::TxContext;
    
    struct Pool has key {
        id: UID,
        balance_a: Balance<sui::SUI>,
        balance_b: Balance<SomeToken>,
    }
    
    struct SomeToken {}
    
    // PTB-friendly: takes coin, returns coin (composable)
    public fun swap_a_for_b(
        pool: &mut Pool,
        coin_in: Coin<sui::SUI>,
        min_out: u64,
        ctx: &mut TxContext,
    ): Coin<SomeToken> {
        let amount_in = coin::value(&coin_in);
        
        // Calculate output
        let amount_out = compute_out(
            balance::value(&pool.balance_a),
            balance::value(&pool.balance_b),
            amount_in,
        );
        assert!(amount_out >= min_out, 1);
        
        // Update pool state
        balance::join(&mut pool.balance_a, coin::into_balance(coin_in));
        
        // Return output coin (PTB can use this in next command)
        coin::from_balance(balance::split(&mut pool.balance_b, amount_out), ctx)
    }
    
    // This enables PTB chain:
    // swapResult = swap_a_for_b(pool, input, min) → Coin<SomeToken>
    // stakeResult = stake(farm, swapResult)         → StakeReceipt
    // All in one transaction!
    
    fun compute_out(reserve_a: u64, reserve_b: u64, amount_in: u64): u64 {
        let amount_in_with_fee = (amount_in as u128) * 997u128;
        let numerator = amount_in_with_fee * (reserve_b as u128);
        let denominator = (reserve_a as u128) * 1000u128 + amount_in_with_fee;
        (numerator / denominator) as u64
    }
}
```

---

## Dynamic Object System

```move
module sui_advanced::dynamic_objects {
    use sui::object::{Self, UID};
    use sui::dynamic_field as df;
    use sui::dynamic_object_field as dof;
    use sui::bag::{Self, Bag};
    use sui::table::{Self, Table};
    use sui::tx_context::TxContext;
    
    // ============================================
    // Dynamic Fields: attach any key-value to object
    // ============================================
    
    struct Inventory has key {
        id: UID,
        owner: address,
    }
    
    // Key type for dynamic fields
    struct ItemKey has copy, drop, store {
        item_type: vector<u8>,
    }
    
    public fun add_item(
        inventory: &mut Inventory,
        item_type: vector<u8>,
        quantity: u64,
    ) {
        let key = ItemKey { item_type };
        if (df::exists_(&inventory.id, key)) {
            let qty = df::borrow_mut<ItemKey, u64>(&mut inventory.id, key);
            *qty = *qty + quantity;
        } else {
            df::add(&mut inventory.id, key, quantity);
        };
    }
    
    public fun get_item_count(
        inventory: &Inventory,
        item_type: vector<u8>,
    ): u64 {
        let key = ItemKey { item_type };
        if (df::exists_(&inventory.id, key)) {
            *df::borrow<ItemKey, u64>(&inventory.id, key)
        } else {
            0
        }
    }
    
    // ============================================
    // Dynamic Object Fields: store objects as fields
    // Objects remain discoverable (not wrapped)
    // ============================================
    
    struct NFTVault has key {
        id: UID,
    }
    
    struct NFT has key {
        id: UID,
        name: std::string::String,
    }
    
    struct NFTKey has copy, drop, store {
        index: u64,
    }
    
    public entry fun deposit_nft(
        vault: &mut NFTVault,
        nft: NFT,
        index: u64,
    ) {
        // Object remains in global state, but owned by vault
        dof::add(&mut vault.id, NFTKey { index }, nft);
    }
    
    public entry fun withdraw_nft(
        vault: &mut NFTVault,
        index: u64,
        ctx: &sui::tx_context::TxContext,
    ) {
        let nft: NFT = dof::remove(&mut vault.id, NFTKey { index });
        sui::transfer::transfer(nft, sui::tx_context::sender(ctx));
    }
    
    // ============================================
    // Bag: heterogeneous key-value collection
    // ============================================
    
    struct MultiAssetPortfolio has key {
        id: UID,
        holdings: Bag,
        owner: address,
    }
    
    public fun create_portfolio(ctx: &mut TxContext): MultiAssetPortfolio {
        MultiAssetPortfolio {
            id: object::new(ctx),
            holdings: bag::new(ctx),
            owner: sui::tx_context::sender(ctx),
        }
    }
    
    public fun add_holding<T: store>(
        portfolio: &mut MultiAssetPortfolio,
        key: vector<u8>,
        asset: T,
    ) {
        bag::add(&mut portfolio.holdings, key, asset);
    }
    
    public fun remove_holding<T: store>(
        portfolio: &mut MultiAssetPortfolio,
        key: vector<u8>,
    ): T {
        bag::remove(&mut portfolio.holdings, key)
    }
    
    // ============================================
    // Table: typed key-value (all same type)
    // ============================================
    
    struct Leaderboard has key {
        id: UID,
        scores: Table<address, u64>,
    }
    
    public fun update_score(
        board: &mut Leaderboard,
        player: address,
        new_score: u64,
    ) {
        if (table::contains(&board.scores, player)) {
            *table::borrow_mut(&mut board.scores, player) = new_score;
        } else {
            table::add(&mut board.scores, player, new_score);
        };
    }
    
    public fun get_score(board: &Leaderboard, player: address): u64 {
        if (table::contains(&board.scores, player)) {
            *table::borrow(&board.scores, player)
        } else {
            0
        }
    }
}
```

---

## Shared Object Contention & Performance

```
Performance Considerations:

Owned Objects (User-owned):
  ✓ Parallel execution
  ✓ No consensus needed for read/write
  ✓ Very fast (single-validator can process)
  ✗ Only owner can use them
  
Shared Objects:
  ✓ Multiple users can access
  ✗ Requires consensus (slower)
  ✗ Sequential execution (bottleneck)
  ✗ Contention under high load

Optimization Strategies:

1. Use owned objects where possible
   Instead of one shared pool:
   → Each user has their own account object
   → Only merge/split during settlement

2. Epoch-based settlement
   Collect user actions off-chain
   Batch-process once per epoch using shared object
   → 1000 user actions = 1 shared object transaction

3. Hot potato pattern
   Shared object creates receipt (owned)
   User interacts with owned receipt
   Return receipt to settle with shared object
   → Minimizes shared object contention

4. Sharded state
   Instead of one SharedPool:
   → PoolShard[0..N] each independent
   → Users randomly assigned to shard
   → Merge shards periodically

5. Admin-mediated pattern
   Off-chain: users sign messages
   Admin: collects and batch-processes
   → Admin handles shared object, users submit intentions

Throughput Numbers (approximate):
  Owned objects: 100,000+ TPS
  Shared objects: ~1,000 TPS (with consensus)
  Real DeFi protocols: mix both for balance
```

```move
module sui_advanced::performance_patterns {
    use sui::object::{Self, UID};
    use sui::tx_context::TxContext;
    use sui::balance::{Self, Balance};
    use sui::coin::{Self, Coin};
    
    // ============================================
    // Pattern: Hot Potato for Atomic Settlements
    // ============================================
    
    // Shared pool (minimal state, only touched for settlement)
    struct Pool has key {
        id: UID,
        total_sui: Balance<sui::SUI>,
        total_shares: u64,
    }
    
    // Owned receipt (no consensus needed for individual ops)
    struct DepositReceipt {
        pool_id: sui::object::ID,
        sui_amount: u64,
        shares_to_mint: u64,
    }
    
    struct ShareToken has key {
        id: UID,
        amount: u64,
    }
    
    // Step 1: User initiates (reads shared pool, creates receipt)
    public fun initiate_deposit(
        pool: &Pool,
        payment: Coin<sui::SUI>,
        ctx: &mut TxContext,
    ): (DepositReceipt, Coin<sui::SUI>) {
        let sui_amount = coin::value(&payment);
        
        // Calculate shares (read-only from pool)
        let shares = if (pool.total_shares == 0) {
            sui_amount
        } else {
            (sui_amount as u128) * (pool.total_shares as u128)
                / (balance::value(&pool.total_sui) as u128) as u64
        };
        
        let receipt = DepositReceipt {
            pool_id: object::uid_to_inner(&pool.id),
            sui_amount,
            shares_to_mint: shares,
        };
        
        (receipt, payment)  // Return receipt + original payment
    }
    
    // Step 2: Finalize (writes to shared pool, returns share tokens)
    public fun finalize_deposit(
        pool: &mut Pool,
        receipt: DepositReceipt,  // Hot potato: must be consumed
        payment: Coin<sui::SUI>,
        ctx: &mut TxContext,
    ): ShareToken {
        let DepositReceipt { pool_id, sui_amount, shares_to_mint } = receipt;
        
        // Verify payment matches
        assert!(coin::value(&payment) == sui_amount, 1);
        assert!(pool_id == object::uid_to_inner(&pool.id), 2);
        
        // Update pool (write to shared object - only once!)
        balance::join(&mut pool.total_sui, coin::into_balance(payment));
        pool.total_shares = pool.total_shares + shares_to_mint;
        
        ShareToken {
            id: object::new(ctx),
            amount: shares_to_mint,
        }
    }
    
    // ============================================
    // Pattern: Sharded State to Reduce Contention
    // ============================================
    
    struct Shard has key {
        id: UID,
        shard_id: u64,
        balance: Balance<sui::SUI>,
    }
    
    struct ShardRouter has key {
        id: UID,
        num_shards: u64,
        shard_ids: vector<sui::object::ID>,
    }
    
    // User deposits to randomly assigned shard
    public entry fun deposit_to_shard(
        shard: &mut Shard,
        coins: Coin<sui::SUI>,
    ) {
        balance::join(&mut shard.balance, coin::into_balance(coins));
    }
    
    // Calculate which shard to use based on user address
    public fun get_shard_for_user(
        router: &ShardRouter,
        user: address,
    ): u64 {
        let addr_bytes = sui::address::to_bytes(user);
        let last_byte = *std::vector::borrow(&addr_bytes, 31);
        (last_byte as u64) % router.num_shards
    }
}
```

---

## ตัวอย่าง: Advanced Sui Patterns

```move
module sui_advanced::advanced_patterns {
    use sui::object::{Self, UID, ID};
    use sui::tx_context::{Self, TxContext};
    use sui::transfer;
    use sui::event;
    
    // ============================================
    // Capability-based access on Sui
    // ============================================
    
    struct AdminCap has key, store {
        id: UID,
        protocol: ID,
    }
    
    struct Protocol has key {
        id: UID,
        fee_bps: u64,
        paused: bool,
    }
    
    public fun initialize(ctx: &mut TxContext): (Protocol, AdminCap) {
        let protocol = Protocol {
            id: object::new(ctx),
            fee_bps: 30,
            paused: false,
        };
        let protocol_id = object::uid_to_inner(&protocol.id);
        
        let cap = AdminCap {
            id: object::new(ctx),
            protocol: protocol_id,
        };
        
        (protocol, cap)
    }
    
    public entry fun set_fee(
        cap: &AdminCap,
        protocol: &mut Protocol,
        new_fee: u64,
    ) {
        assert!(cap.protocol == object::uid_to_inner(&protocol.id), 1);
        assert!(new_fee <= 1000, 2);
        protocol.fee_bps = new_fee;
    }
    
    // ============================================
    // One-Time Witness (OTW) pattern
    // Ensures setup functions run exactly once
    // ============================================
    
    struct MYTOKEN has drop {}  // OTW: struct name matches module, has drop
    
    // init() receives OTW as first arg - called once at publish
    fun init(otw: MYTOKEN, ctx: &mut TxContext) {
        // Create currency using OTW (one-time witness proves this is init)
        // This cannot be called again because MYTOKEN is consumed
        let (treasury_cap, metadata) = sui::coin::create_currency(
            otw,
            9,
            b"MYTKN",
            b"My Token",
            b"An example token",
            std::option::none(),
            ctx,
        );
        
        transfer::public_freeze_object(metadata);
        transfer::transfer(treasury_cap, tx_context::sender(ctx));
    }
    
    // ============================================
    // Transfer Policy (custom transfer rules)
    // ============================================
    
    // With Kiosk + TransferPolicy:
    // Publisher sets rules that MUST be satisfied on purchase
    // E.g., royalty must be paid
    
    // See Part 38 for full Kiosk implementation
    
    // ============================================
    // Versioned shared objects (safe upgrades)
    // ============================================
    
    struct Versioned has key {
        id: UID,
        version: u64,
    }
    
    struct DataV1 {
        value: u64,
    }
    
    struct DataV2 {
        value: u64,
        extra: std::string::String,
    }
    
    const VERSION_1: u64 = 1;
    const VERSION_2: u64 = 2;
    
    public fun create_v1(ctx: &mut TxContext): Versioned {
        let mut obj = Versioned {
            id: object::new(ctx),
            version: VERSION_1,
        };
        sui::dynamic_field::add(&mut obj.id, VERSION_1, DataV1 { value: 0 });
        obj
    }
    
    public fun migrate_to_v2(obj: &mut Versioned) {
        assert!(obj.version == VERSION_1, 1);
        
        // Read old data
        let DataV1 { value } = sui::dynamic_field::remove(&mut obj.id, VERSION_1);
        
        // Write new data
        sui::dynamic_field::add(&mut obj.id, VERSION_2, DataV2 {
            value,
            extra: std::string::utf8(b"migrated"),
        });
        
        obj.version = VERSION_2;
    }
    
    // Functions check version before accessing data
    public fun get_value_v2(obj: &Versioned): u64 {
        assert!(obj.version == VERSION_2, 2);
        sui::dynamic_field::borrow<u64, DataV2>(&obj.id, VERSION_2).value
    }
}
```

---

## สรุป Sui Object Runtime

```
Sui Unique Characteristics:

1. Object Ownership (4 types)
   Owned by address: user controls, parallel
   Shared: anyone uses, sequential
   Frozen: everyone reads, no writes
   Object-owned: nested ownership

2. Programmable Transaction Blocks
   Multiple operations in one transaction
   Outputs from one op feed into next
   All-or-nothing atomicity
   Essential for DeFi composability

3. Dynamic Fields
   Attach arbitrary data to any object
   Key-value store attached to UID
   Dynamic Object Fields: objects remain discoverable
   Enables extensible patterns

4. Parallel Execution
   Owned objects: true parallelism
   Shared objects: sequential bottleneck
   Design to minimize shared object access

5. One-Time Witness
   Proves initialization ran exactly once
   Essential for token creation, publisher caps
   Consumed (has drop) so can't be reused

6. Versioned Objects
   Safe upgrades via dynamic fields
   Version field controls data access
   Gradual migration possible

Design Principle:
  "Owned objects = fast, shared objects = necessary evil"
  Design to push as much as possible to owned objects
```

---

**ก่อนหน้า**: [Part 48 - Move VM Internals ←](part-48-move-vm-internals.md)
**ต่อไป**: [Part 50 - Professional Project Architecture →](part-50-project-architecture.md)
