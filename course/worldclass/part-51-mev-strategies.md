# Part 51: MEV Strategies & Searcher Infrastructure

## สารบัญ
- [MEV Landscape on Aptos & Sui](#mev-landscape-on-aptos--sui)
- [Arbitrage Bot Architecture](#arbitrage-bot-architecture)
- [JIT Liquidity Provision](#jit-liquidity-provision)
- [Liquidation Racing](#liquidation-racing)
- [Cross-Protocol MEV](#cross-protocol-mev)
- [ตัวอย่าง: On-chain MEV Defense Protocol](#ตัวอย่าง-on-chain-mev-defense-protocol)

---

## MEV Landscape on Aptos & Sui

```
MEV (Maximal Extractable Value): Profit from tx ordering

Aptos MEV Types:
  1. DEX Arbitrage: price discrepancy across AMMs
  2. Liquidation: race to liquidate undercollateralized positions
  3. Sandwich attacks: front+back-run large swaps
  4. NFT sniping: buy underpriced NFTs instantly
  5. Governance arbitrage: vote to unlock value

Sui MEV Types:
  1. Similar to Aptos but with parallel execution nuances
  2. Shared object contention = ordering matters
  3. PTBs enable complex atomic strategies

Aptos Transaction Ordering:
  - Block proposer can order transactions
  - No public mempool → less front-running risk than Ethereum
  - But proposers can see pending txs
  - Solutions: private mempools, encrypted txs (future)

MEV Defense vs Capture:
  Defense: design protocols to minimize extractable value
    → TWAP oracles, slippage limits, commit-reveal
  Capture: build on-chain or off-chain systems to earn MEV
    → arbitrage bots, liquidation bots
  Redistributed: MEV goes back to protocol users
    → MEV auction with proceeds to LPs/stakers

World-class MEV understanding:
  You need both sides:
  1. How to defend your protocol against MEV attacks
  2. How to capture legitimate MEV (arbitrage, liquidation)
  Legitimate MEV = makes markets efficient
```

---

## Arbitrage Bot Architecture

```typescript
// off-chain/src/arbitrage/bot.ts
// TypeScript arbitrage bot for Aptos

import { Aptos, AptosConfig, Network, Account } from "@aptos-labs/ts-sdk";
import { createClient } from "redis";

interface PoolState {
  address: string;
  reserveX: bigint;
  reserveY: bigint;
  feeBps: number;
  tokenX: string;
  tokenY: string;
}

interface ArbitrageOpportunity {
  pools: [PoolState, PoolState];
  amountIn: bigint;
  expectedProfit: bigint;
  path: string[];
}

class AptosArbBot {
  private client: Aptos;
  private account: Account;
  private redis: ReturnType<typeof createClient>;
  
  // Known DEX pools to monitor
  private watchedPools: Map<string, PoolState> = new Map();
  
  constructor(privateKey: string, rpcUrl: string) {
    this.client = new Aptos(new AptosConfig({ fullnode: rpcUrl }));
    this.account = Account.fromPrivateKey({ privateKey });
    this.redis = createClient();
  }
  
  async start() {
    console.log("Starting arbitrage bot...");
    
    // Subscribe to events from all watched pools
    await this.subscribeToPoolEvents();
    
    // Main loop: scan for opportunities
    while (true) {
      try {
        await this.scanOpportunities();
      } catch (err) {
        console.error("Error scanning:", err);
      }
      await sleep(100);  // 100ms scan interval
    }
  }
  
  private async scanOpportunities() {
    const opportunities = await this.findArbitrageOpportunities();
    
    for (const opp of opportunities) {
      if (opp.expectedProfit > MIN_PROFIT_THRESHOLD) {
        await this.executeArbitrage(opp);
      }
    }
  }
  
  private async findArbitrageOpportunities(): Promise<ArbitrageOpportunity[]> {
    const opportunities: ArbitrageOpportunity[] = [];
    const pools = Array.from(this.watchedPools.values());
    
    // Find pairs of pools with same token pair but different prices
    for (let i = 0; i < pools.length; i++) {
      for (let j = i + 1; j < pools.length; j++) {
        const poolA = pools[i];
        const poolB = pools[j];
        
        // Check if same token pair (possibly reversed)
        if (!sameTokenPair(poolA, poolB)) continue;
        
        // Calculate price difference
        const priceA = Number(poolA.reserveY) / Number(poolA.reserveX);
        const priceB = Number(poolB.reserveY) / Number(poolB.reserveX);
        
        const priceDiff = Math.abs(priceA - priceB) / Math.min(priceA, priceB);
        
        // Only worth it if > combined fees
        const combinedFee = (poolA.feeBps + poolB.feeBps) / 10000;
        if (priceDiff <= combinedFee * 1.5) continue;
        
        // Find optimal arbitrage amount
        const { amountIn, profit } = computeOptimalArb(poolA, poolB);
        
        if (profit > 0n) {
          opportunities.push({
            pools: [poolA, poolB],
            amountIn,
            expectedProfit: profit,
            path: [poolA.tokenX, poolA.tokenY],
          });
        }
      }
    }
    
    // Sort by profit descending
    return opportunities.sort((a, b) => 
      b.expectedProfit > a.expectedProfit ? 1 : -1
    );
  }
  
  private async executeArbitrage(opp: ArbitrageOpportunity) {
    const [poolA, poolB] = opp.pools;
    const priceA = Number(poolA.reserveY) / Number(poolA.reserveX);
    const priceB = Number(poolB.reserveY) / Number(poolB.reserveX);
    
    // Buy from cheaper pool, sell to expensive pool
    const [buyPool, sellPool] = priceA < priceB 
      ? [poolA, poolB] 
      : [poolB, poolA];
    
    // Build transaction: buy X in poolA, sell X in poolB
    const transaction = await this.client.transaction.build.simple({
      sender: this.account.accountAddress,
      data: {
        function: `${PROTOCOL_ADDR}::arbitrage::execute_arb`,
        functionArguments: [
          buyPool.address,
          sellPool.address,
          opp.amountIn.toString(),
          (opp.expectedProfit * 90n / 100n).toString(),  // 90% of expected profit as min
        ],
      },
    });
    
    const signed = await this.client.transaction.sign({ 
      signer: this.account, 
      transaction 
    });
    
    try {
      const result = await this.client.transaction.submit.simple({ 
        transaction: signed 
      });
      
      console.log(`Arbitrage executed! TxHash: ${result.hash}, Expected profit: ${opp.expectedProfit}`);
    } catch (err) {
      // Race condition: someone else got there first
      console.log("Arbitrage failed (race condition?):", err);
    }
  }
  
  private async subscribeToPoolEvents() {
    // Subscribe to Swap events from all pools
    // Update pool state in real-time
    setInterval(async () => {
      for (const [addr, pool] of this.watchedPools) {
        const updated = await this.fetchPoolState(addr);
        this.watchedPools.set(addr, updated);
      }
    }, 200);  // 200ms refresh
  }
  
  private async fetchPoolState(addr: string): Promise<PoolState> {
    // Fetch current pool reserves from chain
    const [reserveX, reserveY] = await this.client.view({
      payload: {
        function: `${PROTOCOL_ADDR}::amm::get_reserves`,
        typeArguments: ["TOKEN_X_TYPE", "TOKEN_Y_TYPE"],
        functionArguments: [addr],
      }
    });
    
    return {
      ...this.watchedPools.get(addr)!,
      reserveX: BigInt(reserveX as string),
      reserveY: BigInt(reserveY as string),
    };
  }
}

// ============================================
// Math: Optimal arbitrage amount
// ============================================

function computeOptimalArb(
  poolBuy: PoolState,
  poolSell: PoolState,
): { amountIn: bigint; profit: bigint } {
  // Optimal arb amount for two constant product pools:
  // amountIn = sqrt(reserve_buy_x * reserve_sell_x * reserve_sell_y / reserve_buy_y) - reserve_buy_x
  // (ignoring fees for simplicity)
  
  const ra = poolBuy.reserveX;
  const rb = poolBuy.reserveY;
  const rc = poolSell.reserveX;
  const rd = poolSell.reserveY;
  
  // With 0.3% fee on both pools:
  const fa = BigInt(10000 - poolBuy.feeBps);
  const fb = BigInt(10000 - poolSell.feeBps);
  
  // Simplified: binary search for optimal amount
  let lo = 0n, hi = ra;
  let bestProfit = 0n;
  let bestAmount = 0n;
  
  for (let i = 0; i < 64; i++) {
    const mid = (lo + hi) / 2n;
    const profit = calcProfit(poolBuy, poolSell, mid);
    const profitPlus = calcProfit(poolBuy, poolSell, mid + 1n);
    
    if (profitPlus > profit) {
      lo = mid;
    } else {
      hi = mid;
      if (profit > bestProfit) {
        bestProfit = profit;
        bestAmount = mid;
      }
    }
  }
  
  return { amountIn: bestAmount, profit: bestProfit };
}

function calcProfit(poolBuy: PoolState, poolSell: PoolState, amountIn: bigint): bigint {
  // Buy X in poolBuy with amountIn Y
  const xBought = computeOut(
    poolBuy.reserveY,
    poolBuy.reserveX,
    amountIn,
    poolBuy.feeBps,
  );
  
  // Sell X in poolSell for Y
  const yReceived = computeOut(
    poolSell.reserveX,
    poolSell.reserveY,
    xBought,
    poolSell.feeBps,
  );
  
  return yReceived - amountIn;
}

function computeOut(
  reserveIn: bigint,
  reserveOut: bigint,
  amountIn: bigint,
  feeBps: number,
): bigint {
  const feeFactor = BigInt(10000 - feeBps);
  const inWithFee = amountIn * feeFactor;
  const numerator = reserveOut * inWithFee;
  const denominator = reserveIn * 10000n + inWithFee;
  return numerator / denominator;
}

function sameTokenPair(a: PoolState, b: PoolState): boolean {
  return (a.tokenX === b.tokenX && a.tokenY === b.tokenY) ||
         (a.tokenX === b.tokenY && a.tokenY === b.tokenX);
}

function sleep(ms: number) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

const MIN_PROFIT_THRESHOLD = 1_000_000n;  // 0.01 APT minimum profit
const PROTOCOL_ADDR = process.env.PROTOCOL_ADDR!;
```

---

## JIT Liquidity Provision

```
JIT (Just-In-Time) Liquidity:
  Strategy: Add liquidity just before a large swap, remove after
  Goal: Capture swap fees with minimal impermanent loss
  
  Timeline:
  Block N:   Attacker sees pending large swap
  Block N:   Attacker adds liquidity (front-run)
  Block N:   Victim's large swap executes (attacker earns fees)
  Block N:   Attacker removes liquidity (back-run)
  
  Risk: IL (impermanent loss) from price movement
  Reward: Swap fees from victim's trade
  
  Net: Fees > IL → profitable
  
  Defense for LPs (legitimate):
    → Protocol should penalize JIT with minimum LP duration
    → Fee boost for longer-term LPs
```

```move
module mev::anti_jit {
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::timestamp;
    
    // ============================================
    // Anti-JIT: minimum liquidity duration
    // ============================================
    
    const MIN_LP_DURATION: u64 = 3600;  // 1 hour minimum
    
    struct LPPosition has store {
        lp_amount: u64,
        entered_at: u64,
    }
    
    struct PoolLPTracker has key {
        positions: SmartTable<address, LPPosition>,
        
        // Fee boost for long-term LPs
        fee_boost_duration: u64,  // 7 days for max boost
        max_fee_boost_bps: u64,   // Up to 2x fees
    }
    
    // Get LP's fee multiplier based on age of position
    public fun get_fee_multiplier(
        tracker: &PoolLPTracker,
        lp_addr: address,
    ): u64 {  // In basis points: 10000 = 1x, 20000 = 2x
        if (!smart_table::contains(&tracker.positions, lp_addr)) {
            return 10_000
        };
        
        let pos = smart_table::borrow(&tracker.positions, lp_addr);
        let age = timestamp::now_seconds() - pos.entered_at;
        
        if (age < MIN_LP_DURATION) return 0;  // Can't withdraw yet!
        
        // Linear boost over fee_boost_duration
        let boost_pct = std::math64::min(
            age * 10_000 / tracker.fee_boost_duration,
            10_000,  // Cap at 100% boost
        );
        
        10_000 + tracker.max_fee_boost_bps * boost_pct / 10_000
    }
    
    // Penalize early removal
    public fun early_removal_penalty(
        tracker: &PoolLPTracker,
        lp_addr: address,
    ): u64 {  // Penalty in bps of LP value
        if (!smart_table::contains(&tracker.positions, lp_addr)) {
            return 0
        };
        
        let pos = smart_table::borrow(&tracker.positions, lp_addr);
        let age = timestamp::now_seconds() - pos.entered_at;
        
        if (age >= MIN_LP_DURATION) return 0;
        
        // 50% penalty for immediate removal
        // Decreases linearly over MIN_LP_DURATION
        let remaining_pct = (MIN_LP_DURATION - age) * 10_000 / MIN_LP_DURATION;
        5_000 * remaining_pct / 10_000  // Max 50% penalty
    }
}
```

---

## Liquidation Racing

```move
module mev::liquidation_bot {
    
    // ============================================
    // On-chain liquidation with flash loan
    // ============================================
    
    // Strategy:
    // 1. Detect undercollateralized position
    // 2. Flash loan the debt asset
    // 3. Repay debt, receive discounted collateral
    // 4. Sell collateral on DEX
    // 5. Repay flash loan
    // 6. Keep profit
    
    struct FlashLoanReceipt has drop {
        borrowed_amount: u64,
        fee: u64,
    }
    
    // Atomic flash liquidation: borrow → repay → profit
    public entry fun flash_liquidate<DebtAsset, CollateralAsset>(
        liquidator: &signer,
        lending_pool_addr: address,
        flash_pool_addr: address,
        amm_pool_addr: address,
        borrower: address,
        debt_to_repay: u64,
        min_profit: u64,
    ) {
        // Step 1: Flash borrow the debt asset
        // Hot potato: receipt must be returned
        let (debt_coins, receipt) = flash_borrow<DebtAsset>(
            flash_pool_addr,
            debt_to_repay,
        );
        
        // Step 2: Liquidate borrower
        // Receive collateral (at discount) + any bonus
        let collateral_received = lending_liquidate<DebtAsset, CollateralAsset>(
            liquidator,
            lending_pool_addr,
            borrower,
            debt_coins,
        );
        
        // Step 3: Sell collateral on AMM
        let repay_amount = receipt.borrowed_amount + receipt.fee;
        let proceeds = amm_swap<CollateralAsset, DebtAsset>(
            amm_pool_addr,
            collateral_received,
            repay_amount,  // Min out = need to repay flash loan
        );
        
        // Step 4: Verify profit
        let profit = proceeds - repay_amount;
        assert!(profit >= min_profit, 1);
        
        // Step 5: Repay flash loan
        flash_repay<DebtAsset>(flash_pool_addr, proceeds, receipt);
        
        // Remaining profit goes to liquidator automatically
    }
    
    // Placeholder functions (would call actual modules)
    fun flash_borrow<T>(_pool: address, _amount: u64): (u64, FlashLoanReceipt) {
        (0, FlashLoanReceipt { borrowed_amount: 0, fee: 0 })
    }
    
    fun lending_liquidate<D, C>(
        _liquidator: &signer, _pool: address, _borrower: address, _debt: u64
    ): u64 { 0 }
    
    fun amm_swap<In, Out>(_pool: address, _amount_in: u64, _min_out: u64): u64 { 0 }
    
    fun flash_repay<T>(_pool: address, _amount: u64, _receipt: FlashLoanReceipt) {}
}
```

---

## Cross-Protocol MEV

```
Cross-Protocol MEV opportunities:

1. ORACLE ARBITRAGE
   Pyth oracle reports price X
   AMM pool still at price Y (not yet arbitraged)
   → Arbitrage between oracle-linked lending and AMM

2. LIQUIDATION + AMM COMBO
   Position becomes liquidatable
   Liquidation bonus = 5%
   AMM can absorb collateral with 0.5% slippage
   Net: 4.5% profit
   → Flash loan + liquidate + swap

3. GOVERNANCE VOTE TIMING
   Governance passes proposal to change protocol params
   Before execution: trade to benefit from new params
   After execution: collect profit
   → Vote + trade + profit

4. RATE ARBITRAGE
   Lending protocol: borrow APT at 3% APY
   Staking: earn 7% APY
   → Borrow + stake = 4% risk-free (if overcollateralized)

5. LST ARBITRAGE
   lstAPT trades at 0.5% discount on DEX
   Can redeem 1:1 (after 30 days) from protocol
   → Buy lstAPT → redeem (if patient)
   OR
   → Buy lstAPT → lend as collateral → borrow APT → sell

On-chain Implementation:
  Complex MEV strategies need multi-step PTBs (Sui)
  or carefully ordered entry function calls (Aptos)
  Usually flash loans tie everything together atomically
```

---

## ตัวอย่าง: On-chain MEV Defense Protocol

```move
module mev_defense::protected_amm {
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::timestamp;
    
    // ============================================
    // MEV-resistant AMM design
    // Implements multiple defense mechanisms
    // ============================================
    
    struct ProtectedPool has key {
        reserve_x: u64,
        reserve_y: u64,
        fee_bps: u64,
        
        // Defense 1: TWAP for price protection
        price_twap: u64,           // Running TWAP
        twap_last_update: u64,
        max_twap_deviation_bps: u64,  // Max deviation from TWAP allowed
        
        // Defense 2: Rate limiting
        blocks_per_window: u64,
        max_volume_per_window: u64,
        current_window_volume: u64,
        current_window_start: u64,
        
        // Defense 3: MEV fee redistribution
        // MEV captured through fees goes to LPs
        accumulated_mev_fees: u64,
        
        // Defense 4: Min/max swap sizes
        min_swap_size: u64,
        max_swap_size: u64,
        
        // Defense 5: Address-based cooldown
        last_swap: SmartTable<address, u64>,
        cooldown_seconds: u64,
    }
    
    const E_TWAP_DEVIATION: u64 = 1;
    const E_VOLUME_LIMIT: u64 = 2;
    const E_COOLDOWN: u64 = 3;
    const E_SWAP_TOO_SMALL: u64 = 4;
    const E_SWAP_TOO_LARGE: u64 = 5;
    
    public entry fun protected_swap(
        user: &signer,
        pool_addr: address,
        amount_in: u64,
        min_out: u64,
        deadline: u64,
    ) acquires ProtectedPool {
        let pool = borrow_global_mut<ProtectedPool>(pool_addr);
        let now = timestamp::now_seconds();
        let user_addr = std::signer::address_of(user);
        
        // Check deadline
        assert!(now <= deadline, 6);
        
        // Check swap size limits
        assert!(amount_in >= pool.min_swap_size, E_SWAP_TOO_SMALL);
        assert!(amount_in <= pool.max_swap_size, E_SWAP_TOO_LARGE);
        
        // Check cooldown (prevent rapid sequential swaps)
        if (smart_table::contains(&pool.last_swap, user_addr)) {
            let last = *smart_table::borrow(&pool.last_swap, user_addr);
            assert!(now >= last + pool.cooldown_seconds, E_COOLDOWN);
        };
        
        // Compute output
        let amount_out = compute_out(pool.reserve_x, pool.reserve_y, amount_in, pool.fee_bps);
        
        // Update TWAP
        update_twap(pool, now);
        
        // Check output price vs TWAP (prevent sandwich via price manipulation)
        let execution_price = amount_out * 1_000_000 / amount_in;  // PRECISION
        let twap = pool.price_twap;
        let deviation = if (execution_price > twap) {
            (execution_price - twap) * 10_000 / twap
        } else {
            (twap - execution_price) * 10_000 / twap
        };
        assert!(deviation <= pool.max_twap_deviation_bps, E_TWAP_DEVIATION);
        
        // Check volume window
        let window_start = now / pool.blocks_per_window * pool.blocks_per_window;
        if (window_start > pool.current_window_start) {
            pool.current_window_volume = 0;
            pool.current_window_start = window_start;
        };
        pool.current_window_volume = pool.current_window_volume + amount_in;
        assert!(pool.current_window_volume <= pool.max_volume_per_window, E_VOLUME_LIMIT);
        
        // Check slippage
        assert!(amount_out >= min_out, 7);
        
        // Update state
        pool.reserve_x = pool.reserve_x + amount_in;
        pool.reserve_y = pool.reserve_y - amount_out;
        
        // Record last swap time for cooldown
        if (smart_table::contains(&pool.last_swap, user_addr)) {
            *smart_table::borrow_mut(&mut pool.last_swap, user_addr) = now;
        } else {
            smart_table::add(&mut pool.last_swap, user_addr, now);
        };
    }
    
    fun update_twap(pool: &mut ProtectedPool, now: u64) {
        let elapsed = now - pool.twap_last_update;
        if (elapsed == 0 || pool.reserve_x == 0) return;
        
        let spot_price = pool.reserve_y * 1_000_000 / pool.reserve_x;
        let alpha = std::math64::min(elapsed * 100, 10_000);  // Max weight 100%
        
        // Exponential moving average: TWAP = alpha * spot + (1-alpha) * TWAP
        pool.price_twap = (spot_price * alpha + pool.price_twap * (10_000 - alpha)) / 10_000;
        pool.twap_last_update = now;
    }
    
    fun compute_out(reserve_in: u64, reserve_out: u64, amount_in: u64, fee_bps: u64): u64 {
        let fee_factor = 10_000 - fee_bps;
        let in_with_fee = (amount_in as u128) * (fee_factor as u128);
        let numerator = (reserve_out as u128) * in_with_fee;
        let denominator = (reserve_in as u128) * 10_000u128 + in_with_fee;
        (numerator / denominator) as u64
    }
}
```

---

## สรุป MEV Ecosystem

```
MEV Summary for Move Protocols:

Legitimate MEV (market-making):
  ✓ Arbitrage: keeps prices aligned across venues
  ✓ Liquidation: keeps lending protocols solvent
  ✓ JIT liquidity: (controversial) provides depth
  
Harmful MEV:
  ✗ Sandwich attacks: direct cost to traders
  ✗ Front-running: steals value from users
  ✗ Back-running toxic flow: profits from oracle lag
  
Protocol Designer's Toolkit:
  1. TWAP oracle → price manipulation resistance
  2. Commit-reveal → prevents front-running
  3. Rate limiting → prevents rapid MEV extraction
  4. Slippage + deadline → sandwich attack protection
  5. Anti-JIT minimum hold → protects regular LPs
  6. MEV fee redistribution → turns MEV into LP revenue
  
Searcher's Toolkit:
  1. Off-chain simulation → find profitable opportunities
  2. Optimal amount calculation → maximize profit
  3. Flash loans → capital-efficient execution
  4. Atomic transactions → all-or-nothing safety
  5. Real-time event monitoring → fast reaction
  
World-class understanding:
  MEV is neither good nor bad
  It's a market efficiency mechanism
  The best protocols capture and redistribute it
  The best searchers understand protocol mechanics deeply
```

---

**ก่อนหน้า**: [Part 50 - Project Architecture ←](../professional/part-50-project-architecture.md)
**ต่อไป**: [Part 52 - Concentrated Liquidity AMM →](part-52-concentrated-liquidity.md)
