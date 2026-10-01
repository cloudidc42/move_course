# Part 59: Sui Move Deep Dive

## สารบัญ
- [Sui vs Aptos: Core Differences](#sui-vs-aptos-core-differences)
- [Sui Consensus: Narwhal & Bullshark](#sui-consensus-narwhal--bullshark)
- [Object-Centric Programming Model](#object-centric-programming-model)
- [Sui Programmable Transaction Blocks](#sui-programmable-transaction-blocks)
- [Sui Events & Indexing](#sui-events--indexing)
- [Advanced Sui Patterns](#advanced-sui-patterns)
- [Production Sui DeFi: Full DEX](#production-sui-defi-full-dex)

---

## Sui vs Aptos: Core Differences

```
Fundamental Architecture Difference:

Aptos (Move):
  - Account-based (like Ethereum)
  - Resources stored at account address
  - Global storage: address → TypedResource
  - Parallel execution via Block-STM
  - Token standard: Coin<T>

Sui (Move):
  - Object-based (unique!)
  - Objects identified by unique ID (UID)
  - Objects owned (not stored at address)
  - Parallel by default: independent objects = parallel
  - Token standard: Coin<T> is an object
  
Key Differences Table:
+------------------+------------------+------------------+
| Feature          | Aptos            | Sui              |
+------------------+------------------+------------------+
| Storage model    | Account-based    | Object-based     |
| Resource access  | address::module  | object ID        |
| Parallelism      | Block-STM        | Object ownership |
| Coin standard    | Coin<T> (single) | Coin<T> (object) |
| NFT standard     | Digital Asset    | Any object       |
| Composability    | Via resources    | Via objects      |
| Init function    | Via module       | init(ctx)        |
| Transaction type | Single function  | PTB (batch)      |
+------------------+------------------+------------------+

When to use Sui:
  ✓ Games (high-frequency object state changes)
  ✓ NFTs (objects ARE NFTs natively)
  ✓ Composable DeFi (objects compose naturally)
  ✓ Parallelizable apps (independent objects)

When to use Aptos:
  ✓ DeFi with shared state (AMM pools = shared object)
  ✓ Account-centric apps (user profile systems)
  ✓ EVM compatibility needs
  ✓ Familiar account model
```

---

## Sui Consensus: Narwhal & Bullshark

```
Sui Consensus Architecture:

Narwhal (DAG Mempool):
  - Ensures data availability before consensus
  - Workers receive transactions from clients
  - Build DAG of "certificates" (batches)
  - Certificates = 2/3+ validator signatures on batch
  
Bullshark (Consensus):
  - Totally orders the Narwhal DAG
  - Operates on DAG, not individual txs
  - Finalizes blocks in 1-2 rounds typically
  
Result:
  - Very high throughput (>100k TPS theoretical)
  - 400ms - 1s finality (object-owned txs)
  - Shared object txs slower (need sequencing)

Sui Transaction Types:
  1. Object-owned tx: Only sender's objects
     - Immediate finality (500ms)
     - High throughput
     - No sequencing needed
     
  2. Shared object tx: Touches shared objects
     - Needs consensus ordering (800ms-2s)
     - Lower throughput
     - Sequenced by Narwhal
     
Design Implication:
  - Use owned objects when possible (faster)
  - Minimize shared objects
  - Shard shared state into multiple objects
```

---

## Object-Centric Programming Model

```move
module sui_deep::objects {
    use sui::object::{Self, UID, ID};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};
    use sui::dynamic_field as df;
    use sui::dynamic_object_field as dof;
    
    // ============================================
    // Sui Object Lifecycle
    // ============================================
    
    // CREATE: new UID → object exists
    // TRANSFER: change owner
    // SHARE: make shared (public mutable)
    // FREEZE: make immutable (public read-only)
    // DELETE: destroy UID → object gone
    
    struct Asset has key, store {
        id: UID,
        value: u64,
        name: std::string::String,
    }
    
    // Create and transfer to sender
    public entry fun create_asset(
        name: std::string::String,
        value: u64,
        ctx: &mut TxContext,
    ) {
        let asset = Asset {
            id: object::new(ctx),
            value,
            name,
        };
        transfer::transfer(asset, tx_context::sender(ctx));
    }
    
    // Create as shared object (anyone can access)
    public entry fun create_shared_asset(
        name: std::string::String,
        value: u64,
        ctx: &mut TxContext,
    ) {
        let asset = Asset {
            id: object::new(ctx),
            value,
            name,
        };
        transfer::share_object(asset);  // Now accessible by anyone
    }
    
    // Create as frozen object (immutable reference)
    public entry fun create_config(
        name: std::string::String,
        value: u64,
        ctx: &mut TxContext,
    ) {
        let asset = Asset {
            id: object::new(ctx),
            value,
            name,
        };
        transfer::freeze_object(asset);  // Now immutable forever
    }
    
    // ============================================
    // Dynamic Fields: Extensible objects
    // ============================================
    
    struct Inventory has key {
        id: UID,
        item_count: u64,
    }
    
    // Add any type as dynamic field (not in struct definition)
    public entry fun add_item(
        inventory: &mut Inventory,
        slot: u64,
        item_value: u64,
        ctx: &mut TxContext,
    ) {
        // Dynamic field: any slot → any value
        df::add(&mut inventory.id, slot, item_value);
        inventory.item_count = inventory.item_count + 1;
    }
    
    public fun get_item(inventory: &Inventory, slot: u64): u64 {
        *df::borrow<u64, u64>(&inventory.id, slot)
    }
    
    public entry fun remove_item(
        inventory: &mut Inventory,
        slot: u64,
        ctx: &mut TxContext,
    ) {
        let _: u64 = df::remove(&mut inventory.id, slot);
        inventory.item_count = inventory.item_count - 1;
    }
    
    // ============================================
    // Dynamic Object Fields: Object contains objects
    // ============================================
    
    struct NFTCollection has key {
        id: UID,
        count: u64,
    }
    
    struct NFTItem has key, store {
        id: UID,
        name: std::string::String,
        rarity: u8,
    }
    
    // Store NFT inside collection (object owns object)
    public entry fun add_to_collection(
        collection: &mut NFTCollection,
        nft: NFTItem,
        slot_name: std::string::String,
        ctx: &TxContext,
    ) {
        dof::add(&mut collection.id, slot_name, nft);
        collection.count = collection.count + 1;
    }
    
    public fun get_nft_rarity(
        collection: &NFTCollection,
        slot_name: std::string::String,
    ): u8 {
        dof::borrow<std::string::String, NFTItem>(&collection.id, slot_name).rarity
    }
    
    // Remove NFT from collection and return to owner
    public entry fun remove_from_collection(
        collection: &mut NFTCollection,
        slot_name: std::string::String,
        ctx: &mut TxContext,
    ) {
        let nft: NFTItem = dof::remove(&mut collection.id, slot_name);
        collection.count = collection.count - 1;
        transfer::transfer(nft, tx_context::sender(ctx));
    }
    
    // ============================================
    // Object versioning pattern
    // ============================================
    
    // Problem: how to upgrade objects without breaking references?
    // Solution: Version field + migration entry points
    
    struct VersionedConfig has key {
        id: UID,
        version: u64,
        
        // V1 fields
        fee_bps: u64,
        
        // V2 fields (added later)
        max_swap: std::option::Option<u64>,
        
        // V3 fields
        whitelist_enabled: std::option::Option<bool>,
    }
    
    // Migrate V1 → V2 (add max_swap)
    public entry fun migrate_to_v2(
        config: &mut VersionedConfig,
        admin: &signer,
        max_swap: u64,
        ctx: &TxContext,
    ) {
        assert!(config.version == 1, 1);
        config.version = 2;
        config.max_swap = std::option::some(max_swap);
    }
}
```

---

## Sui Programmable Transaction Blocks

```typescript
// TypeScript: Build complex Sui PTBs
import { TransactionBlock } from '@mysten/sui.js/transactions';
import { SuiClient } from '@mysten/sui.js/client';

const client = new SuiClient({ url: 'https://fullnode.mainnet.sui.io' });

// ============================================
// PTB: Complex multi-step DeFi operation
// All steps atomic (all succeed or all revert)
// ============================================

async function leveragedArbitrage(
  keypair: any,
  poolA_id: string,   // Pool A: token prices lower
  poolB_id: string,   // Pool B: token prices higher
  flashLoanPool: string,
  arbitrageAmount: bigint,
) {
  const tx = new TransactionBlock();
  
  // Step 1: Flash loan (borrow without collateral)
  const [borrowedCoins, receipt] = tx.moveCall({
    target: `${PACKAGE_ID}::flash_loan::borrow`,
    arguments: [
      tx.object(flashLoanPool),
      tx.pure(arbitrageAmount),
    ],
    typeArguments: ['0x2::sui::SUI'],
  });
  
  // Step 2: Buy token at Pool A (cheaper)
  const [tokenReceived] = tx.moveCall({
    target: `${PACKAGE_ID}::dex::swap_exact_input`,
    arguments: [
      tx.object(poolA_id),
      borrowedCoins,
      tx.pure(0n),  // min out
    ],
    typeArguments: ['0x2::sui::SUI', `${PACKAGE_ID}::usdc::USDC`],
  });
  
  // Step 3: Sell token at Pool B (more expensive)
  const [profitCoins] = tx.moveCall({
    target: `${PACKAGE_ID}::dex::swap_exact_input`,
    arguments: [
      tx.object(poolB_id),
      tokenReceived,
      tx.pure(arbitrageAmount),  // min out = loan amount (must profit)
    ],
    typeArguments: [`${PACKAGE_ID}::usdc::USDC`, '0x2::sui::SUI'],
  });
  
  // Step 4: Split profit from repayment
  const [repaymentCoins, keepCoins] = tx.splitCoins(
    profitCoins,
    [tx.pure(arbitrageAmount)]
  );
  
  // Step 5: Repay flash loan
  tx.moveCall({
    target: `${PACKAGE_ID}::flash_loan::repay`,
    arguments: [
      tx.object(flashLoanPool),
      repaymentCoins,
      receipt,
    ],
    typeArguments: ['0x2::sui::SUI'],
  });
  
  // Step 6: Transfer profit to caller
  tx.transferObjects([keepCoins], keypair.getPublicKey().toSuiAddress());
  
  // Execute PTB (all steps atomic)
  const result = await client.signAndExecuteTransactionBlock({
    signer: keypair,
    transactionBlock: tx,
    options: { showEffects: true, showEvents: true },
  });
  
  return result;
}

// ============================================
// PTB: Batch NFT operations
// ============================================

async function batchMintNFTs(
  keypair: any,
  mintCapId: string,
  names: string[],
  recipients: string[],
) {
  const tx = new TransactionBlock();
  
  for (let i = 0; i < names.length; i++) {
    // Mint NFT (returns object)
    const [nft] = tx.moveCall({
      target: `${PACKAGE_ID}::nft::mint`,
      arguments: [
        tx.object(mintCapId),
        tx.pure(names[i]),
      ],
    });
    
    // Transfer to recipient
    tx.transferObjects([nft], recipients[i]);
  }
  
  // All mints in ONE transaction (gas efficient!)
  const result = await client.signAndExecuteTransactionBlock({
    signer: keypair,
    transactionBlock: tx,
    options: { showEffects: true },
  });
  
  return result;
}

// ============================================
// PTB: Gas optimization with mergeCoins
// ============================================

async function collectAndSwap(
  keypair: any,
  coinObjectIds: string[],  // Multiple small coin objects
  poolId: string,
  minOut: bigint,
) {
  const tx = new TransactionBlock();
  
  // Merge all coin objects into one (gas efficient)
  // Problem: Sui Coin<T> can be split into many objects
  // Solution: merge before using
  const [primaryCoin] = [tx.object(coinObjectIds[0])];
  
  if (coinObjectIds.length > 1) {
    tx.mergeCoins(
      primaryCoin,
      coinObjectIds.slice(1).map(id => tx.object(id))
    );
  }
  
  // Now swap the merged coin
  const [received] = tx.moveCall({
    target: `${PACKAGE_ID}::dex::swap_exact_input`,
    arguments: [
      tx.object(poolId),
      primaryCoin,
      tx.pure(minOut),
    ],
    typeArguments: ['0x2::sui::SUI', `${PACKAGE_ID}::usdc::USDC`],
  });
  
  tx.transferObjects(
    [received],
    keypair.getPublicKey().toSuiAddress()
  );
  
  return await client.signAndExecuteTransactionBlock({
    signer: keypair,
    transactionBlock: tx,
  });
}
```

---

## Sui Events & Indexing

```move
module sui_deep::events {
    use sui::event;
    use sui::object::ID;
    use std::string::String;
    
    // ============================================
    // Efficient event design for indexing
    // ============================================
    
    // Events are off-chain (not stored on-chain)
    // Used for indexing and notifications
    
    // Best practices:
    // 1. Include enough data to reconstruct state
    // 2. Include IDs for linking events
    // 3. Include timestamp (block time)
    // 4. Use consistent naming
    
    // Swap event: rich data for indexer
    struct SwapEvent has copy, drop {
        pool_id: ID,
        sender: address,
        token_in_type: String,
        token_out_type: String,
        amount_in: u64,
        amount_out: u64,
        fee_amount: u64,
        // Prices for OHLCV tracking
        price_before: u64,  // in token_out per token_in (scaled 1e9)
        price_after: u64,
        // Liquidity info
        reserve_in_after: u64,
        reserve_out_after: u64,
        // TX info
        epoch: u64,
    }
    
    // LP event
    struct LiquidityEvent has copy, drop {
        pool_id: ID,
        provider: address,
        action: u8,  // 0=add, 1=remove
        token_x_type: String,
        token_y_type: String,
        amount_x: u64,
        amount_y: u64,
        lp_minted_or_burned: u64,
        epoch: u64,
    }
    
    // Governance event
    struct ProposalEvent has copy, drop {
        proposal_id: u64,
        proposer: address,
        action: u8,  // 0=created, 1=voted, 2=executed, 3=cancelled
        votes_for: u64,
        votes_against: u64,
    }
    
    // Emit events in trading functions
    public fun emit_swap(
        pool_id: ID,
        sender: address,
        amount_in: u64,
        amount_out: u64,
        fee: u64,
        reserve_in: u64,
        reserve_out: u64,
        epoch: u64,
    ) {
        event::emit(SwapEvent {
            pool_id,
            sender,
            token_in_type: std::string::utf8(b"SUI"),
            token_out_type: std::string::utf8(b"USDC"),
            amount_in,
            amount_out,
            fee_amount: fee,
            price_before: 0,  // Calculate from reserves before
            price_after: reserve_out * 1_000_000_000 / reserve_in,
            reserve_in_after: reserve_in,
            reserve_out_after: reserve_out,
            epoch,
        });
    }
}
```

---

## Production Sui DeFi: Full DEX

```move
module sui_dex::pool {
    use sui::object::{Self, UID, ID};
    use sui::coin::{Self, Coin};
    use sui::balance::{Self, Balance};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};
    use sui::event;
    
    // ============================================
    // Full Sui AMM Implementation
    // CPMM: x * y = k
    // ============================================
    
    struct Pool<phantom X, phantom Y> has key {
        id: UID,
        
        // Reserves
        reserve_x: Balance<X>,
        reserve_y: Balance<Y>,
        
        // LP token supply
        lp_supply: u64,
        
        // Fee configuration
        fee_bps: u64,        // e.g., 30 = 0.30%
        protocol_fee_bps: u64,  // of fee_bps, e.g., 500 = 5% of fees
        
        // Protocol fee accumulation
        protocol_x: Balance<X>,
        protocol_y: Balance<Y>,
    }
    
    struct LP<phantom X, phantom Y> has key, store {
        id: UID,
        pool_id: ID,
        shares: u64,
    }
    
    // Admin capability
    struct AdminCap has key { id: UID }
    
    // Events
    struct SwapEvent has copy, drop {
        pool_id: ID,
        is_x_to_y: bool,
        amount_in: u64,
        amount_out: u64,
        fee: u64,
    }
    
    struct LiquidityEvent has copy, drop {
        pool_id: ID,
        is_add: bool,
        amount_x: u64,
        amount_y: u64,
        shares: u64,
    }
    
    const MINIMUM_LIQUIDITY: u64 = 1000;
    const E_ZERO_AMOUNT: u64 = 1;
    const E_INSUFFICIENT_OUTPUT: u64 = 2;
    const E_INSUFFICIENT_LIQUIDITY: u64 = 3;
    const E_SLIPPAGE: u64 = 4;
    
    // ============================================
    // Create pool
    // ============================================
    
    public entry fun create_pool<X, Y>(
        fee_bps: u64,
        ctx: &mut TxContext,
    ) {
        assert!(fee_bps <= 1000, 1);  // Max 10% fee
        
        let pool = Pool<X, Y> {
            id: object::new(ctx),
            reserve_x: balance::zero(),
            reserve_y: balance::zero(),
            lp_supply: 0,
            fee_bps,
            protocol_fee_bps: 500,  // 5% of fees
            protocol_x: balance::zero(),
            protocol_y: balance::zero(),
        };
        
        transfer::share_object(pool);
    }
    
    // ============================================
    // Add liquidity
    // ============================================
    
    public entry fun add_liquidity<X, Y>(
        pool: &mut Pool<X, Y>,
        coin_x: Coin<X>,
        coin_y: Coin<Y>,
        min_lp: u64,
        ctx: &mut TxContext,
    ) {
        let x_amount = coin::value(&coin_x);
        let y_amount = coin::value(&coin_y);
        
        assert!(x_amount > 0 && y_amount > 0, E_ZERO_AMOUNT);
        
        let rx = balance::value(&pool.reserve_x);
        let ry = balance::value(&pool.reserve_y);
        
        let (actual_x, actual_y, lp_shares) = if (pool.lp_supply == 0) {
            // Initial liquidity
            let lp = sqrt_u64(
                ((x_amount as u128) * (y_amount as u128)) as u64
            ) - MINIMUM_LIQUIDITY;
            (x_amount, y_amount, lp)
        } else {
            // Proportional add
            let lp_from_x = (x_amount as u128) * (pool.lp_supply as u128) / (rx as u128);
            let lp_from_y = (y_amount as u128) * (pool.lp_supply as u128) / (ry as u128);
            
            let shares = std::u128::min(lp_from_x, lp_from_y) as u64;
            
            let actual_x = shares * rx / pool.lp_supply;
            let actual_y = shares * ry / pool.lp_supply;
            
            (actual_x, actual_y, shares)
        };
        
        assert!(lp_shares >= min_lp, E_SLIPPAGE);
        
        // Handle excess tokens (return change)
        let mut x_in = coin_x;
        let mut y_in = coin_y;
        
        if (actual_x < x_amount) {
            let change = coin::split(&mut x_in, x_amount - actual_x, ctx);
            transfer::public_transfer(change, tx_context::sender(ctx));
        };
        if (actual_y < y_amount) {
            let change = coin::split(&mut y_in, y_amount - actual_y, ctx);
            transfer::public_transfer(change, tx_context::sender(ctx));
        };
        
        // Add to reserves
        balance::join(&mut pool.reserve_x, coin::into_balance(x_in));
        balance::join(&mut pool.reserve_y, coin::into_balance(y_in));
        pool.lp_supply = pool.lp_supply + lp_shares;
        
        // Mint LP token object
        let lp_token = LP<X, Y> {
            id: object::new(ctx),
            pool_id: object::id(pool),
            shares: lp_shares,
        };
        
        event::emit(LiquidityEvent {
            pool_id: object::id(pool),
            is_add: true,
            amount_x: actual_x,
            amount_y: actual_y,
            shares: lp_shares,
        });
        
        transfer::transfer(lp_token, tx_context::sender(ctx));
    }
    
    // ============================================
    // Remove liquidity
    // ============================================
    
    public entry fun remove_liquidity<X, Y>(
        pool: &mut Pool<X, Y>,
        lp_token: LP<X, Y>,
        min_x: u64,
        min_y: u64,
        ctx: &mut TxContext,
    ) {
        assert!(lp_token.pool_id == object::id(pool), 1);
        
        let shares = lp_token.shares;
        let LP { id, pool_id: _, shares: _ } = lp_token;
        object::delete(id);
        
        let rx = balance::value(&pool.reserve_x);
        let ry = balance::value(&pool.reserve_y);
        
        let x_out = (shares as u128) * (rx as u128) / (pool.lp_supply as u128);
        let y_out = (shares as u128) * (ry as u128) / (pool.lp_supply as u128);
        
        assert!(x_out as u64 >= min_x, E_SLIPPAGE);
        assert!(y_out as u64 >= min_y, E_SLIPPAGE);
        
        pool.lp_supply = pool.lp_supply - shares;
        
        let x_coin = coin::from_balance(balance::split(&mut pool.reserve_x, x_out as u64), ctx);
        let y_coin = coin::from_balance(balance::split(&mut pool.reserve_y, y_out as u64), ctx);
        
        event::emit(LiquidityEvent {
            pool_id: object::id(pool),
            is_add: false,
            amount_x: x_out as u64,
            amount_y: y_out as u64,
            shares,
        });
        
        let sender = tx_context::sender(ctx);
        transfer::public_transfer(x_coin, sender);
        transfer::public_transfer(y_coin, sender);
    }
    
    // ============================================
    // Swap X → Y
    // ============================================
    
    public entry fun swap_x_to_y<X, Y>(
        pool: &mut Pool<X, Y>,
        coin_in: Coin<X>,
        min_out: u64,
        ctx: &mut TxContext,
    ) {
        let amount_in = coin::value(&coin_in);
        assert!(amount_in > 0, E_ZERO_AMOUNT);
        
        let rx = balance::value(&pool.reserve_x);
        let ry = balance::value(&pool.reserve_y);
        
        let fee = amount_in * pool.fee_bps / 10_000;
        let protocol_fee = fee * pool.protocol_fee_bps / 10_000;
        let lp_fee = fee - protocol_fee;
        
        let amount_in_with_fee = amount_in - fee;
        
        // CPMM: dy = ry * dx / (rx + dx)
        let amount_out = (ry as u128) * (amount_in_with_fee as u128)
            / ((rx + amount_in_with_fee) as u128);
        
        assert!(amount_out as u64 >= min_out, E_INSUFFICIENT_OUTPUT);
        assert!(amount_out < ry as u128, E_INSUFFICIENT_LIQUIDITY);
        
        // Collect protocol fee
        let mut x_in = coin_in;
        if (protocol_fee > 0) {
            let pf = coin::split(&mut x_in, protocol_fee, ctx);
            balance::join(&mut pool.protocol_x, coin::into_balance(pf));
        };
        
        // Add to reserve
        balance::join(&mut pool.reserve_x, coin::into_balance(x_in));
        
        // Extract output
        let y_out = coin::from_balance(
            balance::split(&mut pool.reserve_y, amount_out as u64),
            ctx
        );
        
        event::emit(SwapEvent {
            pool_id: object::id(pool),
            is_x_to_y: true,
            amount_in,
            amount_out: amount_out as u64,
            fee,
        });
        
        transfer::public_transfer(y_out, tx_context::sender(ctx));
    }
    
    // ============================================
    // View functions
    // ============================================
    
    public fun get_reserves<X, Y>(pool: &Pool<X, Y>): (u64, u64) {
        (balance::value(&pool.reserve_x), balance::value(&pool.reserve_y))
    }
    
    public fun get_lp_supply<X, Y>(pool: &Pool<X, Y>): u64 {
        pool.lp_supply
    }
    
    // Price of X in terms of Y (scaled 1e9)
    public fun get_price<X, Y>(pool: &Pool<X, Y>): u64 {
        let rx = balance::value(&pool.reserve_x);
        let ry = balance::value(&pool.reserve_y);
        if (rx == 0) return 0;
        (ry as u128) * 1_000_000_000 / (rx as u128) as u64
    }
    
    fun sqrt_u64(n: u64): u64 {
        if (n == 0) return 0;
        let mut x = n;
        let mut y = (x + 1) / 2;
        while (y < x) {
            x = y;
            y = (x + n / x) / 2;
        };
        x
    }
}
```

---

## สรุป Sui Deep Dive

```
Sui Key Advantages:
  ✓ Object model = natural composability
  ✓ PTBs = atomic multi-step operations
  ✓ Kiosk = on-chain royalty enforcement
  ✓ Parallel execution for owned objects
  ✓ Dynamic fields = extensible objects
  
Sui Key Trade-offs:
  ✗ Shared objects slower (need sequencing)
  ✗ Object IDs harder to reason about than addresses
  ✗ Coin fragmentation (many small Coin objects)
  ✗ Smaller ecosystem than Aptos/Ethereum

Production Tips:
  1. Merge coins before large operations
  2. Use owned objects when possible (faster)
  3. Shard shared objects (e.g., pool per pair)
  4. PTBs for complex multi-step operations
  5. Dynamic fields for extensible state

Sui vs Ethereum Analogy:
  Ethereum: Bank accounts (address owns balance)
  Sui: Cash (coins are physical objects)
  
  In Sui, your APT is literally a Coin<SUI> object
  You hand it to the function, it gives you back change
  Much closer to physical cash semantics!
```

---

**ก่อนหน้า**: [Part 58 - Protocol Economics ←](part-58-protocol-economics.md)
**ต่อไป**: [Part 60 - Production Deployment & Monitoring →](part-60-production-monitoring.md)
