# Part 82: MEV & Order Flow Optimization

## สารบัญ
- [MEV on Move Blockchains](#mev-on-move-blockchains)
- [Frontrunning Protection](#frontrunning-protection)
- [Batch Auction Settlement](#batch-auction-settlement)
- [Searcher Bots](#searcher-bots)
- [Protocol Design for MEV Resistance](#protocol-design-for-mev-resistance)

---

## MEV on Move Blockchains

```
MEV (Maximal Extractable Value):
Value extracted by reordering, inserting, or censoring transactions

ETHEREUM MEV LANDSCAPE
  Frontrunning:      Copy pending tx, pay higher gas to go first
  Backrunning:       Follow specific tx (e.g., DEX trade), arbitrage
  Sandwich attack:   Buy before large trade, sell after
  Liquidation MEV:   Race to liquidate positions
  JIT Liquidity:     Add/remove LP around large trades
  
APTOS/SUI MEV DIFFERENCES
  No mempool in same sense:
    - Transactions go to validators directly
    - No public pending pool to copy from
    - Validators batch-process blocks
    
  Block structure:
    - Aptos: parallel execution (BlockSTM)
    - Sui: per-object parallelism
    - Both: deterministic ordering within a block
    
  What MEV still exists:
    ✓ Validator ordering within a block (validators see all txs)
    ✓ Arbitrage between DEXes (backrunning after price move)
    ✓ Liquidation racing (first-valid-wins at same price)
    ✓ Oracle frontrunning (if oracle updates are predictable)
    
  What's HARDER vs Ethereum:
    ✗ Classic frontrunning (no public mempool to watch)
    ✗ Sandwich attacks (harder without mempool)
    ✗ Gas auction wars (fixed fee structure)

MEV PROTECTION PRIORITY
  High priority (protect these):
    - Large DEX swaps (sandwich risk on high-volume paths)
    - Oracle updates (validator ordering risk)
    - Liquidations (should be first-come, not validator-preferred)
    
  Lower priority (less risk on Move chains):
    - Small swaps (sandwich not economical)
    - Deposits/withdrawals (no ordering benefit)
```

---

## Frontrunning Protection

```move
// ============================================
// COMMIT-REVEAL SWAP
// Prevents validator from knowing intent before ordering
// ============================================

module mev::commit_reveal_dex {
    use aptos_framework::timestamp;
    use aptos_std::aptos_hash;
    
    struct CommitRecord has key {
        commitment: vector<u8>,   // Hash of intent
        committed_at: u64,
        revealed: bool,
    }
    
    struct CommitConfig has key {
        reveal_window: u64,      // Seconds to reveal after commit (e.g., 30)
        min_commit_age: u64,     // Minimum seconds before reveal (e.g., 1 block)
    }
    
    // Step 1: Commit
    // User commits hash(token_in, token_out, amount, nonce)
    // Validator cannot see intent (only hash)
    public entry fun commit_swap(
        user: &signer,
        commitment: vector<u8>,
        config_addr: address,
    ) {
        let user_addr = std::signer::address_of(user);
        
        // Each user can only have one pending commit
        assert!(!exists<CommitRecord>(user_addr), 1);
        
        move_to(user, CommitRecord {
            commitment,
            committed_at: timestamp::now_seconds(),
            revealed: false,
        });
    }
    
    // Step 2: Reveal
    // User reveals actual parameters
    // These are executed at pre-committed price bounds
    public entry fun reveal_and_swap<TokenIn, TokenOut>(
        user: &signer,
        amount_in: u64,
        min_out: u64,
        nonce: u64,
        pool_addr: address,
        config_addr: address,
    ) acquires CommitRecord, CommitConfig {
        let user_addr = std::signer::address_of(user);
        let record = borrow_global_mut<CommitRecord>(user_addr);
        let config = borrow_global<CommitConfig>(config_addr);
        
        assert!(!record.revealed, 1);  // Not already revealed
        
        let now = timestamp::now_seconds();
        assert!(now >= record.committed_at + config.min_commit_age, 2);  // Old enough
        assert!(now <= record.committed_at + config.reveal_window, 3);   // Not expired
        
        // Verify commitment matches revealed parameters
        let expected_commitment = compute_commitment(
            user_addr,
            amount_in,
            min_out,
            nonce,
        );
        assert!(record.commitment == expected_commitment, 4);  // Reveal matches commit
        
        record.revealed = true;
        
        // Execute the actual swap
        // pool::swap<TokenIn, TokenOut>(user, amount_in, min_out, pool_addr);
    }
    
    // Compute the commitment hash
    public fun compute_commitment(
        user: address,
        amount_in: u64,
        min_out: u64,
        nonce: u64,
    ): vector<u8> {
        let mut data = std::bcs::to_bytes(&user);
        std::vector::append(&mut data, std::bcs::to_bytes(&amount_in));
        std::vector::append(&mut data, std::bcs::to_bytes(&min_out));
        std::vector::append(&mut data, std::bcs::to_bytes(&nonce));
        aptos_hash::keccak256(data)
    }
    
    // Cancel expired commitment
    public entry fun cancel_commit(user: &signer) acquires CommitRecord, CommitConfig {
        let user_addr = std::signer::address_of(user);
        let record = borrow_global<CommitRecord>(user_addr);
        // Can only cancel after reveal window expired
        // (prevents griefing: user can't cancel right before reveal)
        assert!(timestamp::now_seconds() > record.committed_at + 60, 1);
        
        let CommitRecord { commitment: _, committed_at: _, revealed: _ } = move_from<CommitRecord>(user_addr);
    }
    
    // ============================================
    // SLIPPAGE AS ANTI-SANDWICH
    // Strict min_out prevents profitable sandwiching
    // ============================================
    
    // Sandwich math: attacker only profits if:
    // profit_from_arb > gas_cost + slip_paid_by_user
    // 
    // If user sets min_out such that slip < 0.1%:
    //   Sandwich requires buying enough to move price 0.1%+
    //   On large pools: requires very large capital
    //   Gas cost > profit for small trades
    //
    // Rule: Always set min_out = expected_out * (1 - slippage_tolerance)
    
    public fun calculate_min_out(
        expected_out: u64,
        slippage_bps: u64,
    ): u64 {
        expected_out * (10_000 - slippage_bps) / 10_000
    }
    
    // ============================================
    // TWAP-BASED SLIPPAGE
    // Reject swaps that deviate too much from TWAP
    // ============================================
    
    public fun validate_swap_price(
        twap_price: u64,        // TWAP price (reserve_y / reserve_x, scaled)
        spot_price: u64,        // Current spot price
        max_deviation_bps: u64,
    ) {
        let deviation = if (spot_price > twap_price) {
            (spot_price - twap_price) * 10_000 / twap_price
        } else {
            (twap_price - spot_price) * 10_000 / twap_price
        };
        
        assert!(deviation <= max_deviation_bps, 1);  // Price too far from TWAP
    }
}
```

---

## Batch Auction Settlement

```move
// ============================================
// BATCH AUCTION: CoW Protocol style
// Orders are collected, then settled optimally
// Eliminates frontrunning by batching
// ============================================

module mev::batch_auction {
    use aptos_framework::timestamp;
    use aptos_std::table::{Self, Table};
    
    struct Order has store {
        user: address,
        token_in: std::string::String,
        token_out: std::string::String,
        amount_in: u64,
        min_out: u64,
        deadline: u64,
        filled: bool,
        fill_amount: u64,
    }
    
    struct BatchAuction has key {
        current_batch: u64,
        batch_duration: u64,      // Seconds per batch
        batch_start: u64,
        orders: Table<u64, Order>,  // order_id → Order
        next_order_id: u64,
        solver_address: address,    // Trusted solver (can be decentralized)
    }
    
    // Users submit orders (collected for batch)
    public entry fun submit_order(
        user: &signer,
        token_in: std::string::String,
        token_out: std::string::String,
        amount_in: u64,
        min_out: u64,
        deadline: u64,
        auction_addr: address,
    ) acquires BatchAuction {
        let auction = borrow_global_mut<BatchAuction>(auction_addr);
        
        let order_id = auction.next_order_id;
        auction.next_order_id = order_id + 1;
        
        table::add(&mut auction.orders, order_id, Order {
            user: std::signer::address_of(user),
            token_in,
            token_out,
            amount_in,
            min_out,
            deadline,
            filled: false,
            fill_amount: 0,
        });
        
        aptos_framework::event::emit(OrderSubmitted {
            order_id,
            user: std::signer::address_of(user),
            batch: auction.current_batch,
        });
    }
    
    // Solver settles the batch (off-chain compute, on-chain settlement)
    // Solver finds optimal matching (CoW: Coincidence of Wants)
    // Orders that match directly don't need AMM liquidity at all!
    public entry fun settle_batch(
        solver: &signer,
        order_ids: vector<u64>,
        fill_amounts: vector<u64>,
        prices: vector<u64>,
        auction_addr: address,
    ) acquires BatchAuction {
        let auction = borrow_global_mut<BatchAuction>(auction_addr);
        
        assert!(std::signer::address_of(solver) == auction.solver_address, 1);
        assert!(timestamp::now_seconds() >= auction.batch_start + auction.batch_duration, 2);
        
        let n = std::vector::length(&order_ids);
        let mut i = 0u64;
        
        while (i < n) {
            let order_id = *std::vector::borrow(&order_ids, i);
            let fill_amount = *std::vector::borrow(&fill_amounts, i);
            
            let order = table::borrow_mut(&mut auction.orders, order_id);
            
            assert!(!order.filled, 3);
            assert!(timestamp::now_seconds() <= order.deadline, 4);
            assert!(fill_amount >= order.min_out, 5);  // Slippage check
            
            order.filled = true;
            order.fill_amount = fill_amount;
            
            // Transfer tokens to user
            // (In practice: settle atomically using reserves)
            
            i = i + 1;
        };
        
        // Advance to next batch
        auction.current_batch = auction.current_batch + 1;
        auction.batch_start = timestamp::now_seconds();
    }
    
    // Batch clearing price benefits:
    // - All orders in same batch get same price
    // - No frontrunning (all submitted before execution)
    // - CoW matching: user A sells X for Y, user B sells Y for X = no AMM needed!
    
    #[event] struct OrderSubmitted has drop, store { order_id: u64, user: address, batch: u64 }
}
```

---

## Searcher Bots

```typescript
// ============================================
// ARBITRAGE BOT: Cross-DEX arbitrage on Aptos
// ============================================

import { AptosClient, AptosAccount } from 'aptos';

interface PoolState {
  dex: string;
  address: string;
  reserveX: bigint;
  reserveY: bigint;
  feeNumerator: bigint;
  feeDenominator: bigint;
}

class AptosArbitrageBot {
  private client: AptosClient;
  private account: AptosAccount;
  private pools: PoolState[] = [];
  
  constructor(nodeUrl: string, privateKey: string) {
    this.client = new AptosClient(nodeUrl);
    this.account = new AptosAccount(Buffer.from(privateKey, 'hex'));
  }
  
  // Load pool states from on-chain
  async loadPools(): Promise<void> {
    const poolConfigs = [
      { dex: 'liquidswap', address: '0xLIQUID...', type: 'liquidswap::Pool' },
      { dex: 'pontem', address: '0xPONTEM...', type: 'pontem::Pool' },
      { dex: 'econia', address: '0xECONIA...', type: 'econia::Pool' },
    ];
    
    this.pools = [];
    for (const config of poolConfigs) {
      try {
        const resource = await this.client.getAccountResource(config.address, config.type);
        const data = resource.data as any;
        
        this.pools.push({
          dex: config.dex,
          address: config.address,
          reserveX: BigInt(data.reserve_x),
          reserveY: BigInt(data.reserve_y),
          feeNumerator: BigInt(data.fee_numerator ?? 30),
          feeDenominator: BigInt(data.fee_denominator ?? 10000),
        });
      } catch {
        // Pool not found or different structure
      }
    }
  }
  
  // Calculate output for CPMM swap
  getAmountOut(
    amountIn: bigint,
    reserveIn: bigint,
    reserveOut: bigint,
    feeNum: bigint,
    feeDen: bigint,
  ): bigint {
    const amountInWithFee = amountIn * (feeDen - feeNum);
    const numerator = amountInWithFee * reserveOut;
    const denominator = reserveIn * feeDen + amountInWithFee;
    return numerator / denominator;
  }
  
  // Find arbitrage opportunities
  findArbitrage(amountIn: bigint): Array<{
    profit: bigint;
    buyPool: string;
    sellPool: string;
    amountIn: bigint;
    amountOut: bigint;
  }> {
    const opportunities = [];
    
    for (let i = 0; i < this.pools.length; i++) {
      for (let j = 0; j < this.pools.length; j++) {
        if (i === j) continue;
        
        const buyPool = this.pools[i];   // Buy token Y (spend X)
        const sellPool = this.pools[j];  // Sell token Y (receive X)
        
        // Buy Y from pool i
        const yReceived = this.getAmountOut(
          amountIn,
          buyPool.reserveX,
          buyPool.reserveY,
          buyPool.feeNumerator,
          buyPool.feeDenominator,
        );
        
        // Sell Y on pool j
        const xReceived = this.getAmountOut(
          yReceived,
          sellPool.reserveY,
          sellPool.reserveX,
          sellPool.feeNumerator,
          sellPool.feeDenominator,
        );
        
        const profit = xReceived - amountIn;
        
        if (profit > 0n) {
          opportunities.push({
            profit,
            buyPool: `${buyPool.dex}@${buyPool.address}`,
            sellPool: `${sellPool.dex}@${sellPool.address}`,
            amountIn,
            amountOut: xReceived,
          });
        }
      }
    }
    
    return opportunities.sort((a, b) => Number(b.profit - a.profit));
  }
  
  // Optimize trade size using golden section search
  optimizeTradeSize(
    buyPool: PoolState,
    sellPool: PoolState,
    maxIn: bigint,
  ): { optimalIn: bigint; expectedProfit: bigint } {
    let lo = 1n;
    let hi = maxIn;
    
    const profitAt = (amountIn: bigint): bigint => {
      const yOut = this.getAmountOut(amountIn, buyPool.reserveX, buyPool.reserveY, buyPool.feeNumerator, buyPool.feeDenominator);
      const xOut = this.getAmountOut(yOut, sellPool.reserveY, sellPool.reserveX, sellPool.feeNumerator, sellPool.feeDenominator);
      return xOut - amountIn;
    };
    
    // Golden section search (profit is concave in trade size)
    const PHI = 618n;  // ~0.618 * 1000
    
    for (let iter = 0; iter < 50; iter++) {
      const m1 = lo + (hi - lo) * PHI / 1000n;
      const m2 = hi - (hi - lo) * PHI / 1000n;
      
      if (profitAt(m1) < profitAt(m2)) {
        hi = m2;
      } else {
        lo = m1;
      }
    }
    
    const optimalIn = (lo + hi) / 2n;
    return { optimalIn, expectedProfit: profitAt(optimalIn) };
  }
  
  // Execute arbitrage trade
  async executeArbitrage(
    buyPoolAddr: string,
    sellPoolAddr: string,
    amountIn: bigint,
    minProfit: bigint,
  ): Promise<string | null> {
    const profitEstimate = this.findArbitrage(amountIn)[0]?.profit ?? 0n;
    
    if (profitEstimate < minProfit) {
      console.log(`Profit ${profitEstimate} < min ${minProfit}, skipping`);
      return null;
    }
    
    // Execute atomic arbitrage in single transaction
    const payload = {
      function: `0xARB_CONTRACT::arbitrage::execute_arb`,
      type_arguments: ['0xAPT::aptos_coin::AptosCoin', '0xUSDC::coin::USDC'],
      arguments: [
        buyPoolAddr,
        sellPoolAddr,
        amountIn.toString(),
        (amountIn + minProfit).toString(),  // min_out = at least break even + min profit
      ],
    };
    
    try {
      const tx = await this.client.generateTransaction(this.account.address().toString(), payload);
      const signed = await this.client.signTransaction(this.account, tx);
      const result = await this.client.submitTransaction(signed);
      await this.client.waitForTransaction(result.hash);
      console.log(`Arb executed: ${result.hash}`);
      return result.hash;
    } catch (error) {
      console.log(`Arb failed: ${error}`);
      return null;
    }
  }
  
  // Main loop
  async run(): Promise<void> {
    console.log('Starting arbitrage bot...');
    
    while (true) {
      await this.loadPools();
      
      const amountsToTry = [1_000_000n, 10_000_000n, 100_000_000n];  // APT in octas
      
      for (const amount of amountsToTry) {
        const opps = this.findArbitrage(amount);
        
        if (opps.length > 0) {
          const best = opps[0];
          console.log(`Found: profit=${best.profit} buy=${best.buyPool} sell=${best.sellPool}`);
          
          if (best.profit > 100_000n) {  // Min 0.001 APT profit
            await this.executeArbitrage(
              best.buyPool.split('@')[1],
              best.sellPool.split('@')[1],
              amount,
              best.profit / 2n,  // Accept half estimated profit (accounting for price impact)
            );
          }
        }
      }
      
      await new Promise(r => setTimeout(r, 1000));  // 1 second between checks
    }
  }
}
```

---

## Protocol Design for MEV Resistance

```move
// ============================================
// MEV-RESISTANT PROTOCOL PATTERNS
// ============================================

module mev::mev_resistant {
    use aptos_framework::timestamp;
    
    // ============================================
    // PATTERN 1: TIME-WEIGHTED EXECUTION
    // Large orders split across time
    // ============================================
    
    struct TWAPOrder has key {
        token_in: std::string::String,
        token_out: std::string::String,
        total_amount: u64,
        remaining: u64,
        rate_per_second: u64,   // How much to execute per second
        start_time: u64,
        end_time: u64,
        last_execution: u64,
        owner: address,
    }
    
    // Anyone can trigger execution (no MEV from who executes)
    public entry fun execute_twap_slice(
        executor: &signer,  // Gets small reward for executing
        order_addr: address,
        pool_addr: address,
    ) acquires TWAPOrder {
        let order = borrow_global_mut<TWAPOrder>(order_addr);
        let now = timestamp::now_seconds();
        
        assert!(now >= order.start_time, 1);
        assert!(now <= order.end_time, 2);
        assert!(order.remaining > 0, 3);
        
        // Calculate how much to execute
        let elapsed = now - order.last_execution;
        let to_execute = std::u64::min(
            order.rate_per_second * elapsed,
            order.remaining,
        );
        
        assert!(to_execute > 0, 4);
        
        order.last_execution = now;
        order.remaining = order.remaining - to_execute;
        
        // Execute the slice on pool
        // pool::swap(order_addr, to_execute, 0, pool_addr);
        
        // Small reward to executor (incentivizes execution)
        let reward = to_execute / 1000;  // 0.1% of executed amount
        // transfer_reward(std::signer::address_of(executor), reward);
    }
    
    // ============================================
    // PATTERN 2: MINIMUM SIZE THRESHOLD
    // Prevent tiny trades that exist only to manipulate
    // ============================================
    
    const MIN_SWAP_AMOUNT: u64 = 10_000_000;  // 0.01 APT minimum
    
    public fun validate_swap_size(amount: u64) {
        assert!(amount >= MIN_SWAP_AMOUNT, 1);
    }
    
    // ============================================
    // PATTERN 3: PER-BLOCK RATE LIMITS
    // Limit how much liquidity can move in one block
    // ============================================
    
    struct RateLimiter has key {
        max_per_block: u64,     // Max tokens swappable in one block
        current_block: u64,
        block_volume: u64,
    }
    
    public fun check_rate_limit(
        amount: u64,
        limiter: &mut RateLimiter,
    ) {
        let block_height = aptos_framework::block::get_current_block_height();
        
        if (block_height != limiter.current_block) {
            // New block: reset volume
            limiter.current_block = block_height;
            limiter.block_volume = 0;
        };
        
        limiter.block_volume = limiter.block_volume + amount;
        assert!(limiter.block_volume <= limiter.max_per_block, 1);
    }
    
    // ============================================
    // PATTERN 4: PRICE IMPACT LIMITS
    // Reject swaps that move price too much
    // ============================================
    
    public fun check_price_impact(
        amount_in: u64,
        reserve_in: u64,
        max_impact_bps: u64,
    ) {
        // Price impact ≈ amount_in / (2 * reserve_in)
        // Impact bps = amount_in * 10_000 / (2 * reserve_in)
        let impact_bps = (amount_in as u128) * 10_000 / (2 * reserve_in as u128);
        assert!(impact_bps as u64 <= max_impact_bps, 1);
    }
    
    // ============================================
    // PATTERN 5: SANDWICH DETECTION
    // Revert if sandwich is detected in same block
    // ============================================
    
    struct SandwichGuard has key {
        last_block: u64,
        block_swaps: vector<SwapRecord>,
    }
    
    struct SwapRecord has store, drop {
        user: address,
        amount: u64,
        direction: bool,  // true = x->y, false = y->x
    }
    
    public fun check_no_sandwich(
        user: address,
        direction: bool,
        amount: u64,
        guard: &mut SandwichGuard,
    ) {
        let block = aptos_framework::block::get_current_block_height();
        
        if (block != guard.last_block) {
            guard.last_block = block;
            guard.block_swaps = std::vector::empty();
        };
        
        // Check if someone already swapped in same direction this block
        // (simplified sandwich detection)
        let n = std::vector::length(&guard.block_swaps);
        if (n > 0) {
            let last = std::vector::borrow(&guard.block_swaps, n - 1);
            // If last swap was in same direction by different user: potential sandwich
            if (last.direction == direction && last.user != user && last.amount > amount * 2) {
                // This looks like a sandwich (large swap same direction before us)
                // Note: This is simplified; real detection is more complex
                // assert!(false, 1);  // Would reject but too many false positives
            };
        };
        
        std::vector::push_back(&mut guard.block_swaps, SwapRecord { user, amount, direction });
    }
}
```

---

## สรุป MEV & Order Flow

```
MEV PROTECTION TOOLBOX

PROTOCOL DESIGN
  ✅ TWAP orders: Split large trades over time
  ✅ Commit-reveal: Hide intent until execution
  ✅ Batch auctions: All orders same price, no ordering benefit
  ✅ Price impact limits: Reject price-moving trades
  ✅ Rate limits per block: Cap total flow
  ✅ Strict slippage: Make sandwich unprofitable

USER BEST PRACTICES
  ✅ Set tight slippage (0.1-0.5% for liquid pairs)
  ✅ Use private RPC for sensitive trades
  ✅ Split large trades manually or use TWAP
  ✅ Trade during high-volume periods (sandwiches harder)

APTOS ADVANTAGES
  ✅ No public mempool → frontrunning harder
  ✅ Parallel execution → less ordering manipulation
  ✅ Deterministic fees → no gas auction wars
  
  Remaining risk:
  ⚠️ Validator ordering power within blocks
  ⚠️ Cross-DEX arbitrage (beneficial MEV, keeps prices aligned)
  ⚠️ Liquidation competition (race to liquidate)

PROTOCOL REVENUE FROM MEV
  Not all MEV is bad:
  + Arbitrage: Keeps prices aligned across DEXes
  + Liquidations: Keeps lending protocols solvent
  
  Harmful MEV to prevent:
  - Sandwich attacks: Pure extraction, harms users
  - Oracle manipulation: Causes wrong liquidations
  - JIT liquidity: Dilutes honest LP fees without taking risk
```

---

**ก่อนหน้า**: [Part 81 - Advanced Tokenomics ←](part-81-tokenomics.md)
**ต่อไป**: [Part 83 - DeFi Risk Management →](part-83-risk-management.md)
