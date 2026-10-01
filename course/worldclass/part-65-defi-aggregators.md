# Part 65: DeFi Aggregators & Routing

## สารบัญ
- [Aggregator Architecture](#aggregator-architecture)
- [Multi-hop Routing](#multi-hop-routing)
- [Split Route Execution](#split-route-execution)
- [Price Impact Optimization](#price-impact-optimization)
- [Gas-Optimal Routing](#gas-optimal-routing)
- [Full Implementation: Move Router](#full-implementation-move-router)

---

## Aggregator Architecture

```
DeFi Aggregators (1inch, Paraswap, Jupiter):

Problem: User wants to swap 1M USDC → BTC
  Pool A: BTC-USDC, only has 500K liquidity → huge slippage
  Pool B: BTC-USDC on another DEX, 300K liquidity
  Pool C: BTC-ETH, ETH-USDC (indirect route)
  
Aggregator Solution:
  1. Split order: 40% Pool A + 35% Pool B + 25% Pool C (via ETH)
  2. All execute in one transaction
  3. Better overall price than any single pool
  
Value of Aggregator:
  User gets: Best price possible
  DEXes get: More volume
  Aggregator gets: 0.05% cut of savings
  
On Aptos/Sui:
  Hippo (Aptos): Major aggregator
  BlueMove: Sui aggregator
  Aftermath Finance: Sui aggregator
  
Routing Algorithm:
  1. Enumerate all possible paths A→B
     (direct, A→C→B, A→C→D→B, etc.)
  2. For each path: calculate expected output
  3. For split routes: optimize allocation
  4. Select: max output - gas cost
```

---

## Multi-hop Routing

```move
module aggregator::router {
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // Multi-hop Router: A → B → C → ... → Z
    // ============================================
    
    struct Router has key {
        // Registered DEX adapters
        dex_adapters: SmartTable<std::string::String, address>,
        
        // Fee recipient
        protocol_fee_bps: u64,
        protocol_treasury: address,
    }
    
    struct RouteStep has drop {
        dex_name: std::string::String,  // "HippoSwap", "Liquidswap", etc.
        pool_addr: address,
        token_in: std::string::String,
        token_out: std::string::String,
        min_output: u64,                 // Slippage for this step
    }
    
    struct Route has drop {
        steps: vector<RouteStep>,
        total_input: u64,
        min_output: u64,  // Final minimum output
    }
    
    // Execute a multi-hop route
    public entry fun swap_multi_hop<
        TokenIn, 
        TokenMid1,
        TokenMid2,
        TokenOut,
    >(
        user: &signer,
        router_addr: address,
        amount_in: u64,
        min_amount_out: u64,
        route: vector<u8>,  // Encoded route (simplified as bytes)
    ) acquires Router {
        let router = borrow_global<Router>(router_addr);
        let user_addr = std::signer::address_of(user);
        
        // Take input from user
        let input_coins = aptos_framework::coin::withdraw<TokenIn>(user, amount_in);
        
        // Execute each hop
        // Hop 1: TokenIn → TokenMid1
        let after_hop1 = execute_dex_swap::<TokenIn, TokenMid1>(
            input_coins,
            &router.dex_adapters,
            std::string::utf8(b"HippoSwap"),
            0,
        );
        
        // Hop 2: TokenMid1 → TokenMid2
        let after_hop2 = execute_dex_swap::<TokenMid1, TokenMid2>(
            after_hop1,
            &router.dex_adapters,
            std::string::utf8(b"Liquidswap"),
            0,
        );
        
        // Hop 3: TokenMid2 → TokenOut
        let mut output = execute_dex_swap::<TokenMid2, TokenOut>(
            after_hop2,
            &router.dex_adapters,
            std::string::utf8(b"AuxExchange"),
            0,
        );
        
        let output_amount = aptos_framework::coin::value(&output);
        assert!(output_amount >= min_amount_out, 1);
        
        // Protocol fee on output
        let fee = output_amount * router.protocol_fee_bps / 10_000;
        if (fee > 0) {
            let fee_coins = aptos_framework::coin::extract(&mut output, fee);
            aptos_framework::coin::deposit(router.protocol_treasury, fee_coins);
        };
        
        // Return output to user
        aptos_framework::coin::deposit(user_addr, output);
    }
    
    fun execute_dex_swap<TokenIn, TokenOut>(
        coins_in: aptos_framework::coin::Coin<TokenIn>,
        adapters: &SmartTable<std::string::String, address>,
        dex_name: std::string::String,
        min_out: u64,
    ): aptos_framework::coin::Coin<TokenOut> {
        // In production: dispatch to registered DEX adapter
        // adapter.swap(coins_in, min_out)
        
        // Simplified: just return zero coins for example
        aptos_framework::coin::destroy_zero(
            aptos_framework::coin::extract(&mut {coins_in}, 0)
        );
        aptos_framework::coin::zero()
    }
}
```

---

## Split Route Execution

```move
module aggregator::split_router {
    use aptos_framework::coin::{Self, Coin};
    
    // ============================================
    // Split Route: Send different % to different DEXes
    // Better price for large orders
    // ============================================
    
    struct SplitPart has drop {
        pool_addr: address,
        pool_type: u8,  // 0=CPMM, 1=CLMM, 2=StableSwap
        share_bps: u64, // Share of input (in bps, must sum to 10000)
    }
    
    // Execute split swap: tokenIn → tokenOut via multiple pools simultaneously
    // All happens in one PTB on Sui, one tx on Aptos
    public fun swap_split<TokenIn, TokenOut>(
        coins_in: Coin<TokenIn>,
        splits: vector<SplitPart>,
        min_out: u64,
    ): Coin<TokenOut> {
        let total_in = coin::value(&coins_in);
        let mut coins_in_mut = coins_in;
        
        // Validate splits sum to 10000 bps
        let mut total_bps = 0u64;
        let len = std::vector::length(&splits);
        let mut i = 0u64;
        while (i < len) {
            total_bps = total_bps + std::vector::borrow(&splits, i).share_bps;
            i = i + 1;
        };
        assert!(total_bps == 10_000, 1);
        
        // Execute each split
        let mut accumulated_out: Coin<TokenOut> = coin::zero<TokenOut>();
        let mut remaining_in = total_in;
        
        let mut j = 0u64;
        while (j < len) {
            let split = std::vector::borrow(&splits, j);
            
            // Calculate how much to send to this pool
            let split_amount = if (j == len - 1) {
                // Last split: use remaining (avoid dust from rounding)
                remaining_in
            } else {
                total_in * split.share_bps / 10_000
            };
            
            remaining_in = remaining_in - split_amount;
            
            // Extract portion
            let split_coins = coin::extract(&mut coins_in_mut, split_amount);
            
            // Swap at this pool (dispatcher based on pool_type)
            let out = match_pool_swap<TokenIn, TokenOut>(
                split_coins,
                split.pool_addr,
                split.pool_type,
                0,  // min_out per split = 0, check at end
            );
            
            // Merge outputs
            coin::merge(&mut accumulated_out, out);
            
            j = j + 1;
        };
        
        // Check total output meets minimum
        let total_out = coin::value(&accumulated_out);
        assert!(total_out >= min_out, 2);
        
        accumulated_out
    }
    
    fun match_pool_swap<TokenIn, TokenOut>(
        coins_in: Coin<TokenIn>,
        pool_addr: address,
        pool_type: u8,
        min_out: u64,
    ): Coin<TokenOut> {
        // Route to appropriate DEX based on pool type
        // In production: call specific DEX modules
        coin::zero<TokenOut>()
    }
}
```

---

## Price Impact Optimization

```typescript
// off-chain routing optimizer (TypeScript)

interface Pool {
  address: string;
  reserveX: bigint;
  reserveY: bigint;
  feeBps: number;
  poolType: 'cpmm' | 'clmm' | 'stable';
}

interface Route {
  steps: RouteStep[];
  outputAmount: bigint;
  priceImpact: number;  // 0-1 (0% to 100%)
  gasEstimate: number;
}

interface RouteStep {
  poolAddress: string;
  tokenIn: string;
  tokenOut: string;
  amountIn: bigint;
  expectedOut: bigint;
}

// Find optimal route for token swap
class DexRouter {
  private pools: Map<string, Pool[]>;
  private tokenGraph: Map<string, Set<string>>;

  constructor(allPools: Pool[]) {
    this.buildGraph(allPools);
  }

  private buildGraph(pools: Pool[]) {
    this.pools = new Map();
    this.tokenGraph = new Map();

    for (const pool of pools) {
      // Add both directions
      this.addPoolToGraph(pool.address, 'X', 'Y');
      this.addPoolToGraph(pool.address, 'Y', 'X');
    }
  }

  // Find all routes from tokenIn to tokenOut (max 3 hops)
  findAllRoutes(tokenIn: string, tokenOut: string, maxHops: number = 3): string[][] {
    const routes: string[][] = [];
    const visited = new Set<string>();

    const dfs = (current: string, path: string[]) => {
      if (current === tokenOut) {
        routes.push([...path]);
        return;
      }
      if (path.length >= maxHops + 1) return;

      visited.add(current);
      const neighbors = this.tokenGraph.get(current) || new Set();

      for (const neighbor of neighbors) {
        if (!visited.has(neighbor)) {
          path.push(neighbor);
          dfs(neighbor, path);
          path.pop();
        }
      }
      visited.delete(current);
    };

    dfs(tokenIn, [tokenIn]);
    return routes;
  }

  // Calculate output for a specific route
  calculateRouteOutput(
    route: string[],
    amountIn: bigint,
  ): { output: bigint; priceImpact: number } {
    let currentAmount = amountIn;
    let totalPriceImpact = 1;

    for (let i = 0; i < route.length - 1; i++) {
      const tokenIn = route[i];
      const tokenOut = route[i + 1];
      
      const pool = this.findBestPool(tokenIn, tokenOut);
      if (!pool) return { output: 0n, priceImpact: 1 };

      const { output, priceImpact } = this.simulateSwap(pool, currentAmount, tokenIn);
      currentAmount = output;
      totalPriceImpact *= (1 - priceImpact);
    }

    return { 
      output: currentAmount, 
      priceImpact: 1 - totalPriceImpact 
    };
  }

  // Optimize split allocation between two routes
  optimizeSplit(
    route1: string[],
    route2: string[],
    totalAmountIn: bigint,
  ): { amount1: bigint; amount2: bigint; totalOutput: bigint } {
    // Binary search for optimal split
    let lo = 0n;
    let hi = totalAmountIn;
    
    const calcTotal = (amount1: bigint): bigint => {
      const amount2 = totalAmountIn - amount1;
      const { output: out1 } = this.calculateRouteOutput(route1, amount1);
      const { output: out2 } = this.calculateRouteOutput(route2, amount2);
      return out1 + out2;
    };

    // Ternary search for maximum
    for (let i = 0; i < 64; i++) {
      const m1 = lo + (hi - lo) / 3n;
      const m2 = hi - (hi - lo) / 3n;
      
      if (calcTotal(m1) < calcTotal(m2)) {
        lo = m1;
      } else {
        hi = m2;
      }
    }

    const optimalAmount1 = (lo + hi) / 2n;
    const optimalAmount2 = totalAmountIn - optimalAmount1;
    
    return {
      amount1: optimalAmount1,
      amount2: optimalAmount2,
      totalOutput: calcTotal(optimalAmount1),
    };
  }

  // Find best route considering gas costs
  findBestRoute(
    tokenIn: string,
    tokenOut: string,
    amountIn: bigint,
    gasPrice: bigint,  // Gas cost per hop in USD (scaled 1e6)
  ): Route {
    const allRoutes = this.findAllRoutes(tokenIn, tokenOut);
    
    let bestRoute: Route | null = null;
    let bestNetOutput = 0n;

    for (const tokenPath of allRoutes) {
      const { output, priceImpact } = this.calculateRouteOutput(tokenPath, amountIn);
      const hops = tokenPath.length - 1;
      const gasCost = gasPrice * BigInt(hops) * 100n;  // Rough estimate
      
      const netOutput = output - gasCost;
      
      if (netOutput > bestNetOutput) {
        bestNetOutput = netOutput;
        bestRoute = {
          steps: [],  // Build steps from tokenPath
          outputAmount: output,
          priceImpact,
          gasEstimate: hops,
        };
      }
    }

    return bestRoute!;
  }

  private simulateSwap(pool: Pool, amountIn: bigint, tokenIn: string): { output: bigint; priceImpact: number } {
    const { reserveX, reserveY, feeBps } = pool;
    const feeMultiplier = BigInt(10000 - feeBps);
    
    const amountWithFee = amountIn * feeMultiplier / 10000n;
    const output = reserveY * amountWithFee / (reserveX + amountWithFee);
    
    const idealOutput = reserveY * amountIn / reserveX;
    const priceImpact = Number(idealOutput - output) / Number(idealOutput);
    
    return { output, priceImpact };
  }

  private findBestPool(tokenIn: string, tokenOut: string): Pool | null {
    return null;  // Simplified
  }

  private addPoolToGraph(pool: string, tokenIn: string, tokenOut: string) {}
}

// Export optimized route as on-chain calldata
function encodeRoute(route: Route): Uint8Array {
  // Encode route for on-chain execution
  return new Uint8Array();
}
```

---

## สรุป DeFi Aggregators

```
Aggregator Key Concepts:

1. ROUTE DISCOVERY
   - BFS/DFS through token graph (max 3 hops)
   - Consider all pools for each pair
   - Early termination if path > max_hops
   
2. SPLIT OPTIMIZATION
   - Ternary search for optimal split
   - Convexity: price impact is convex
   - Optimal split makes marginal prices equal
   
3. GAS AWARENESS
   - Each hop costs ~$0.01-0.10 in gas
   - 1-hop with 2% slippage may beat 3-hop with 0.5%
   - Net output = output - gas_cost
   
4. SLIPPAGE PROTECTION
   - Per-step minimum outputs
   - Total minimum output
   - Deadline for execution
   
5. MEV PROTECTION
   - Commit-reveal for large swaps
   - Private mempool (Flashbots-style)
   - Slippage tolerance limits mev
   
Best Aggregators by Chain:
  Ethereum: 1inch, Paraswap, CoW Protocol
  Aptos: Hippo, Liquidswap aggregator
  Sui: BlueMove, Aftermath Finance
  
Building an Aggregator:
  1. Index all pools (GraphQL/RPC)
  2. Build token adjacency graph
  3. Find all paths (up to N hops)
  4. Simulate each path off-chain
  5. Execute best path on-chain
  6. Split if large order
```

---

**ก่อนหน้า**: [Part 64 - Layer 2 & Rollups ←](part-64-layer2-rollups.md)
**ต่อไป**: [Part 66 - Advanced Security Patterns →](part-66-advanced-security.md)
