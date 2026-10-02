# Part 94: Multi-Chain Strategy

## สารบัญ
- [Multi-Chain Architecture](#multi-chain-architecture)
- [Unified Liquidity Model](#unified-liquidity-model)
- [Shared State via LayerZero/Wormhole](#shared-state-via-layerzero--wormhole)
- [Cross-Chain Account Abstraction](#cross-chain-account-abstraction)
- [Arbitrage Across Chains](#arbitrage-across-chains)
- [Multi-Chain Deployment Automation](#multi-chain-deployment-automation)

---

## Multi-Chain Architecture

```
MULTI-CHAIN DEPLOYMENT STRATEGY

          ┌──────────────────────────────────────┐
          │         Canonical Hub Chain          │
          │  (Ethereum / Aptos / Sui)            │
          │  ┌────────────────────────────────┐  │
          │  │  Governance Token (canonical)  │  │
          │  │  Treasury                      │  │
          │  │  Cross-Chain Bridge Adapter    │  │
          │  └────────────────────────────────┘  │
          └───────────┬──────────┬───────────────┘
                      │          │
              ┌───────▼──┐  ┌───▼──────┐
              │  Aptos   │  │   Sui    │
              │ Satellite│  │Satellite │
              │ ┌──────┐ │  │ ┌──────┐ │
              │ │Bridge│ │  │ │Bridge│ │
              │ │Adapter│ │  │ │Adapt.│ │
              │ └──┬───┘ │  │ └──┬───┘ │
              │    │     │  │    │     │
              │ ┌──▼───┐ │  │ ┌──▼───┐ │
              │ │ AMM  │ │  │ │ AMM  │ │
              │ │Lend  │ │  │ │Lend  │ │
              │ │NFT   │ │  │ │NFT   │ │
              │ └──────┘ │  │ └──────┘ │
              └──────────┘  └──────────┘

CHAIN SELECTION RATIONALE
  Aptos:  High TPS (160k+), parallel execution, Rust-like Move
  Sui:    Object model, low latency, PTBs, gaming/NFT friendly
  Ethereum: Most liquidity, EVM ecosystem, composability
  
  Rule: Deploy where your users are + where your assets are

BRIDGING OPTIONS
  1. Native bridges (most trust-minimized)
  2. Canonical wrapped assets (Wormhole, LayerZero)
  3. Liquidity networks (fast but limited size)
  4. Optimistic bridges (cheap, 30min delay)

TOKEN ARCHITECTURE
  Hub:     Governance token (ERC-20 / native Move FA)
  Spokes:  Wrapped governance token (via bridge)
  Note:    Only ONE canonical token, others are wrapped
```

---

## Unified Liquidity Model

```move
// ============================================
// UNIFIED LIQUIDITY: Route across chains
// ============================================

module protocol::unified_liquidity {
    
    /// Cross-chain liquidity position
    struct CrossChainPosition has key {
        owner: address,
        
        // Positions per chain (encoded chain ID → amounts)
        chain_positions: aptos_std::table::Table<u16, ChainPosition>,
        
        // Total value in USD terms (oracle-priced)
        total_usd_value: u128,
        
        last_sync_time: u64,
    }
    
    struct ChainPosition has store, copy, drop {
        chain_id: u16,
        protocol_addr: vector<u8>,  // Address on that chain
        asset_type: vector<u8>,     // Asset identifier
        amount: u64,
        usd_value: u128,
    }
    
    /// Virtual LP token that represents cross-chain liquidity
    struct XChainLP has key {
        total_shares: u64,
        // Liquidity distributed across chains
        // chain_id → ratio (bps)
        distribution: aptos_std::table::Table<u16, u64>,
    }
    
    const CHAIN_APTOS: u16 = 22;
    const CHAIN_SUI: u16 = 21;
    const CHAIN_ETH: u16 = 2;
    
    public fun sync_position(
        position: &mut CrossChainPosition,
        chain_id: u16,
        new_amount: u64,
        usd_price: u128,  // 1e8 scaled
    ) {
        let usd_value = (new_amount as u128) * usd_price / 100_000_000;
        
        let chain_pos = ChainPosition {
            chain_id,
            protocol_addr: vector::empty(),
            asset_type: vector::empty(),
            amount: new_amount,
            usd_value,
        };
        
        if (aptos_std::table::contains(&position.chain_positions, chain_id)) {
            let old = aptos_std::table::borrow_mut(&mut position.chain_positions, chain_id);
            position.total_usd_value = position.total_usd_value - old.usd_value + usd_value;
            *old = chain_pos;
        } else {
            aptos_std::table::add(&mut position.chain_positions, chain_id, chain_pos);
            position.total_usd_value = position.total_usd_value + usd_value;
        };
        
        position.last_sync_time = aptos_framework::timestamp::now_microseconds();
    }
    
    /// Rebalance: move liquidity between chains
    /// Sends cross-chain message to move funds
    public fun rebalance(
        owner: &signer,
        position: &CrossChainPosition,
        from_chain: u16,
        to_chain: u16,
        amount: u64,
    ) {
        // 1. Verify owner
        assert!(std::signer::address_of(owner) == position.owner, 1);
        
        // 2. Check from_chain has enough
        let from_pos = aptos_std::table::borrow(&position.chain_positions, from_chain);
        assert!(from_pos.amount >= amount, 2);
        
        // 3. Send cross-chain message to withdraw from from_chain
        // bridge::send_message(from_chain, encode_withdraw(amount, to_chain));
        
        // 4. Cross-chain message will trigger deposit on to_chain
        // (handled by bridge relayer)
        let _ = (to_chain, amount, from_pos);
    }
}
```

---

## Shared State via LayerZero / Wormhole

```move
// ============================================
// CROSS-CHAIN MESSAGE PASSING
// Synchronize state across chains
// ============================================

module protocol::cross_chain_state {
    use aptos_framework::timestamp;
    
    /// A message sent to another chain
    struct OutboundMessage has drop {
        sequence: u64,
        destination_chain: u16,
        destination_addr: vector<u8>,
        payload: vector<u8>,
        created_at: u64,
    }
    
    /// A message received from another chain
    struct InboundMessage has key {
        sequence: u64,
        source_chain: u16,
        source_addr: vector<u8>,
        payload: vector<u8>,
        processed: bool,
    }
    
    /// Global state synchronized across chains
    struct SharedProtocolState has key {
        total_tvl: u128,          // Updated via cross-chain messages
        global_utilization: u64,  // Borrow/supply ratio
        last_sync: u64,
        sync_sequence: u64,
        
        // Per-chain TVL reports
        chain_tvls: aptos_std::table::Table<u16, ChainTVL>,
    }
    
    struct ChainTVL has store, copy, drop {
        chain_id: u16,
        tvl: u128,
        reported_at: u64,
    }
    
    /// Receive TVL report from another chain
    /// Called by bridge adapter when message arrives
    public fun receive_tvl_report(
        bridge_adapter: &signer,
        source_chain: u16,
        tvl: u128,
        report_time: u64,
    ) {
        // Verify caller is authorized bridge adapter
        // assert!(is_authorized_adapter(signer::address_of(bridge_adapter)), 1);
        
        let state = borrow_global_mut<SharedProtocolState>(@protocol);
        
        let chain_tvl = ChainTVL {
            chain_id: source_chain,
            tvl,
            reported_at: report_time,
        };
        
        // Update per-chain TVL
        if (aptos_std::table::contains(&state.chain_tvls, source_chain)) {
            let old = aptos_std::table::borrow_mut(&mut state.chain_tvls, source_chain);
            state.total_tvl = state.total_tvl - old.tvl + tvl;
            *old = chain_tvl;
        } else {
            aptos_std::table::add(&mut state.chain_tvls, source_chain, chain_tvl);
            state.total_tvl = state.total_tvl + tvl;
        };
        
        state.last_sync = timestamp::now_microseconds();
        state.sync_sequence = state.sync_sequence + 1;
        
        let _ = bridge_adapter;
    }
    
    /// Broadcast current chain's state to all other chains
    public fun broadcast_state(
        reporter: &signer,
        local_tvl: u128,
        destination_chains: vector<u16>,
    ) {
        let payload = encode_tvl_report(local_tvl);
        
        let n = vector::length(&destination_chains);
        let i = 0;
        while (i < n) {
            let chain_id = *vector::borrow(&destination_chains, i);
            // bridge::send(chain_id, @protocol_on_that_chain, payload);
            let _ = (chain_id, &payload);
            i = i + 1;
        };
        
        let _ = reporter;
    }
    
    fun encode_tvl_report(tvl: u128): vector<u8> {
        // BCS encode the TVL value
        let bytes = vector::empty<u8>();
        // Encode as 16 bytes big-endian
        let i = 0u8;
        while (i < 16) {
            let byte = ((tvl >> (i * 8)) & 0xFF) as u8;
            vector::push_back(&mut bytes, byte);
            i = i + 1;
        };
        bytes
    }
    
    #[view]
    public fun get_global_tvl(): u128 {
        borrow_global<SharedProtocolState>(@protocol).total_tvl
    }
    
    #[view]
    public fun get_chain_tvl(chain_id: u16): u128 {
        let state = borrow_global<SharedProtocolState>(@protocol);
        if (aptos_std::table::contains(&state.chain_tvls, chain_id)) {
            aptos_std::table::borrow(&state.chain_tvls, chain_id).tvl
        } else {
            0
        }
    }
}
```

---

## Cross-Chain Account Abstraction

```typescript
// ============================================
// CROSS-CHAIN ACCOUNT ABSTRACTION
// One user identity, actions across all chains
// ============================================

import { AptosClient, AptosAccount } from 'aptos';
import { SuiClient } from '@mysten/sui.js/client';
import { Ed25519Keypair } from '@mysten/sui.js/keypairs/ed25519';

interface ChainConfig {
  chainId: string;
  rpcUrl: string;
  protocolAddr: string;
}

interface CrossChainUser {
  // Same private key across all chains (Ed25519 compatible)
  privateKey: string;
  aptosAddress: string;
  suiAddress: string;
}

class CrossChainWallet {
  private aptosClient: AptosClient;
  private suiClient: SuiClient;
  private aptosAccount: AptosAccount;
  private suiKeypair: Ed25519Keypair;
  
  constructor(privateKeyHex: string) {
    this.aptosAccount = new AptosAccount(Buffer.from(privateKeyHex, 'hex'));
    this.suiKeypair = Ed25519Keypair.fromSecretKey(Buffer.from(privateKeyHex, 'hex'));
    
    this.aptosClient = new AptosClient('https://fullnode.mainnet.aptoslabs.com/v1');
    this.suiClient = new SuiClient({ url: 'https://fullnode.mainnet.sui.io' });
  }
  
  get aptosAddress(): string {
    return this.aptosAccount.address().toString();
  }
  
  get suiAddress(): string {
    return this.suiKeypair.getPublicKey().toSuiAddress();
  }
  
  // Get balances across all chains
  async getPortfolio(): Promise<Record<string, any>> {
    const [aptosBalances, suiBalances] = await Promise.all([
      this.getAptosBalances(),
      this.getSuiBalances(),
    ]);
    
    return {
      aptos: aptosBalances,
      sui: suiBalances,
      totalUsdValue: await this.computeTotalUsdValue(aptosBalances, suiBalances),
    };
  }
  
  private async getAptosBalances(): Promise<any[]> {
    const resources = await this.aptosClient.getAccountResources(this.aptosAddress);
    return resources
      .filter(r => r.type.includes('CoinStore'))
      .map(r => ({
        coin: r.type,
        amount: BigInt((r.data as any).coin.value),
        chain: 'aptos',
      }));
  }
  
  private async getSuiBalances(): Promise<any[]> {
    const balances = await this.suiClient.getAllBalances({ owner: this.suiAddress });
    return balances.map(b => ({
      coin: b.coinType,
      amount: BigInt(b.totalBalance),
      chain: 'sui',
    }));
  }
  
  // Execute cross-chain operation: bridge + swap in one flow
  async bridgeAndSwap(
    fromChain: 'aptos' | 'sui',
    toChain: 'aptos' | 'sui',
    bridgeAmount: bigint,
    swapParams: { tokenIn: string; tokenOut: string; minOut: bigint },
  ): Promise<void> {
    console.log(`Bridging ${bridgeAmount} from ${fromChain} to ${toChain}`);
    
    // Step 1: Bridge tokens
    const bridgeTxHash = await this.bridge(fromChain, toChain, bridgeAmount);
    console.log(`Bridge tx: ${bridgeTxHash}`);
    
    // Step 2: Wait for bridge confirmation (~15 mins for Wormhole)
    await this.waitForBridgeConfirmation(bridgeTxHash);
    
    // Step 3: Swap on destination chain
    const swapTxHash = await this.swap(toChain, swapParams);
    console.log(`Swap tx: ${swapTxHash}`);
  }
  
  private async bridge(
    fromChain: 'aptos' | 'sui',
    toChain: 'aptos' | 'sui',
    amount: bigint,
  ): Promise<string> {
    if (fromChain === 'aptos') {
      const payload = {
        function: '0xWORMHOLE_ADDR::bridge::transfer_tokens',
        type_arguments: ['0x1::aptos_coin::AptosCoin'],
        arguments: [
          amount.toString(),
          this.chainToWormholeId(toChain),
          this.suiAddress,
          '0',  // nonce
        ],
      };
      const txn = await this.aptosClient.generateTransaction(
        this.aptosAddress,
        payload,
      );
      const signed = await this.aptosClient.signTransaction(this.aptosAccount, txn);
      const result = await this.aptosClient.submitTransaction(signed);
      return result.hash;
    }
    
    throw new Error('Only Aptos→Sui bridge implemented');
  }
  
  private async swap(
    chain: 'aptos' | 'sui',
    params: { tokenIn: string; tokenOut: string; minOut: bigint },
  ): Promise<string> {
    if (chain === 'aptos') {
      const payload = {
        function: '0xPROTOCOL::pool::swap',
        type_arguments: [params.tokenIn, params.tokenOut],
        arguments: [params.minOut.toString()],
      };
      const txn = await this.aptosClient.generateTransaction(this.aptosAddress, payload);
      const signed = await this.aptosClient.signTransaction(this.aptosAccount, txn);
      const result = await this.aptosClient.submitTransaction(signed);
      return result.hash;
    }
    
    throw new Error('Sui swap not implemented');
  }
  
  private chainToWormholeId(chain: string): number {
    const ids: Record<string, number> = { aptos: 22, sui: 21, ethereum: 2 };
    return ids[chain] || 0;
  }
  
  private async waitForBridgeConfirmation(txHash: string): Promise<void> {
    console.log(`Waiting for bridge confirmation (txHash: ${txHash})...`);
    await new Promise(r => setTimeout(r, 15 * 60 * 1000));  // 15 min
  }
  
  private async computeTotalUsdValue(..._balances: any[]): Promise<number> {
    return 0;  // Simplified
  }
}
```

---

## Arbitrage Across Chains

```typescript
// ============================================
// CROSS-CHAIN ARBITRAGE BOT
// Find and exploit price differences between chains
// ============================================

interface PriceOnChain {
  chain: string;
  tokenPair: string;
  price: number;
  liquidity: bigint;
  timestamp: number;
}

class CrossChainArbitrageBot {
  private minProfitBps = 50;  // 0.5% minimum profit after all fees
  private bridgeFeesBps = 30;  // Wormhole fees ~0.3%
  private gasCost = 5;         // $5 gas per transaction
  
  async findArbitrageOpportunities(
    pairs: string[],
    chains: string[],
  ): Promise<any[]> {
    const prices: PriceOnChain[] = [];
    
    // Fetch prices from all chains simultaneously
    await Promise.all(
      pairs.flatMap(pair =>
        chains.map(async chain => {
          const price = await this.getPrice(chain, pair);
          if (price) prices.push(price);
        })
      )
    );
    
    // Find cross-chain arbitrage opportunities
    const opportunities = [];
    for (let i = 0; i < prices.length; i++) {
      for (let j = i + 1; j < prices.length; j++) {
        const p1 = prices[i];
        const p2 = prices[j];
        
        if (p1.tokenPair !== p2.tokenPair || p1.chain === p2.chain) continue;
        
        const priceDiff = Math.abs(p1.price - p2.price) / Math.min(p1.price, p2.price);
        const priceDiffBps = priceDiff * 10000;
        
        // Check if profitable after bridge fees
        const netProfitBps = priceDiffBps - this.bridgeFeesBps - 20;  // 0.2% slippage
        
        if (netProfitBps > this.minProfitBps) {
          const buyOnChain = p1.price < p2.price ? p1 : p2;
          const sellOnChain = p1.price < p2.price ? p2 : p1;
          
          opportunities.push({
            pair: p1.tokenPair,
            buyChain: buyOnChain.chain,
            sellChain: sellOnChain.chain,
            buyPrice: buyOnChain.price,
            sellPrice: sellOnChain.price,
            priceDiffBps,
            netProfitBps,
            liquidity: BigInt(Math.min(Number(buyOnChain.liquidity), Number(sellOnChain.liquidity))),
          });
        }
      }
    }
    
    return opportunities.sort((a, b) => b.netProfitBps - a.netProfitBps);
  }
  
  // Compute optimal trade size for cross-chain arb
  optimalTradeSize(
    opportunity: any,
    maxCapital: bigint,
  ): bigint {
    // Max out at 1% of liquidity to minimize price impact
    const maxFromLiquidity = opportunity.liquidity / 100n;
    
    // Optimal: solve for max profit considering price impact on both sides
    // Simplified: use golden-section search
    const optimal = this.goldenSectionSearch(
      (size: number) => this.estimateProfit(size, opportunity),
      0,
      Number(maxFromLiquidity),
    );
    
    return BigInt(Math.floor(optimal)) > maxCapital ? maxCapital : BigInt(Math.floor(optimal));
  }
  
  private estimateProfit(tradeSize: number, opp: any): number {
    // Price impact on buy side
    const priceImpactBps = tradeSize / Number(opp.liquidity) * 10000;
    const effectiveBuyPrice = opp.buyPrice * (1 + priceImpactBps / 10000);
    
    // Price impact on sell side
    const effectiveSellPrice = opp.sellPrice * (1 - priceImpactBps / 10000);
    
    const grossProfit = (effectiveSellPrice - effectiveBuyPrice) * tradeSize;
    const bridgeFee = (this.bridgeFeesBps / 10000) * tradeSize * opp.buyPrice;
    const gasCost = this.gasCost * 2;  // Two transactions
    
    return grossProfit - bridgeFee - gasCost;
  }
  
  private goldenSectionSearch(
    f: (x: number) => number,
    lo: number,
    hi: number,
    iterations: number = 50,
  ): number {
    const phi = (1 + Math.sqrt(5)) / 2;
    const resphi = 2 - phi;
    
    let x1 = lo + resphi * (hi - lo);
    let x2 = hi - resphi * (hi - lo);
    let f1 = f(x1);
    let f2 = f(x2);
    
    for (let i = 0; i < iterations; i++) {
      if (f1 < f2) {
        lo = x1;
        x1 = x2;
        f1 = f2;
        x2 = hi - resphi * (hi - lo);
        f2 = f(x2);
      } else {
        hi = x2;
        x2 = x1;
        f2 = f1;
        x1 = lo + resphi * (hi - lo);
        f1 = f(x1);
      }
    }
    
    return (lo + hi) / 2;
  }
  
  private async getPrice(chain: string, pair: string): Promise<PriceOnChain | null> {
    // Fetch from chain-specific DEX
    return null;  // Simplified
  }
}
```

---

## Multi-Chain Deployment Automation

```bash
#!/bin/bash
# ============================================
# MULTI-CHAIN DEPLOYMENT SCRIPT
# Deploy same protocol to Aptos + Sui
# ============================================

set -e

PROTOCOL_NAME="my_protocol"
APTOS_NETWORK=${1:-devnet}
SUI_NETWORK=${2:-devnet}

echo "=========================================="
echo "Deploying $PROTOCOL_NAME"
echo "Aptos: $APTOS_NETWORK | Sui: $SUI_NETWORK"
echo "=========================================="

# ---- APTOS DEPLOYMENT ----
echo ""
echo "[1/4] Building Aptos package..."
cd ./aptos
aptos move compile \
  --named-addresses protocol=default \
  --dev

echo "[2/4] Running Aptos tests..."
aptos move test \
  --named-addresses protocol=default

echo "[2/4] Deploying to Aptos $APTOS_NETWORK..."
APTOS_RESULT=$(aptos move publish \
  --profile $APTOS_NETWORK \
  --named-addresses protocol=default \
  --assume-yes 2>&1)

APTOS_ADDR=$(echo "$APTOS_RESULT" | grep "Account Address" | awk '{print $NF}')
echo "Aptos deployed at: $APTOS_ADDR"

# ---- SUI DEPLOYMENT ----
echo ""
echo "[3/4] Building Sui package..."
cd ../sui
sui move build

echo "[3/4] Running Sui tests..."
sui move test

echo "[4/4] Deploying to Sui $SUI_NETWORK..."
SUI_RESULT=$(sui client publish \
  --gas-budget 100000000 \
  --network $SUI_NETWORK 2>&1)

SUI_PKG=$(echo "$SUI_RESULT" | grep "PackageID" | awk '{print $NF}')
echo "Sui deployed at: $SUI_PKG"

# ---- SAVE DEPLOYMENT CONFIG ----
echo ""
echo "Saving deployment config..."
cat > ../deployment.json << EOF
{
  "protocol": "$PROTOCOL_NAME",
  "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "aptos": {
    "network": "$APTOS_NETWORK",
    "address": "$APTOS_ADDR"
  },
  "sui": {
    "network": "$SUI_NETWORK",
    "packageId": "$SUI_PKG"
  }
}
EOF

echo "Deployment complete!"
echo "Config saved to deployment.json"
cat ../deployment.json
```

---

## สรุป Multi-Chain Strategy

```
MULTI-CHAIN DECISION FRAMEWORK

WHEN TO GO MULTI-CHAIN
  ✅ Users on multiple chains (don't force migration)
  ✅ Protocol needs liquidity from multiple ecosystems
  ✅ Regulatory: spread risk across chains
  ✅ Features: use best-fit chain per use case
  ❌ Don't multi-chain just to appear decentralized
  ❌ If liquidity is fragmented, single chain is better

CHAIN SPECIALIZATION STRATEGY
  Aptos:  Institutional DeFi (high-performance, Move safety)
  Sui:    Gaming, NFTs, consumer apps (object model, UX)
  Ethereum: Institutional liquidity, L1 security
  
  Example: Protocol on Aptos + Sui:
    - Aptos: Perps, options, high-frequency trading
    - Sui: NFT marketplace, gaming items, social features

UNIFIED LIQUIDITY APPROACHES
  1. Bridge-based: Real assets on each chain
     + Simple, native
     - Fragmented liquidity, bridge risk
     
  2. Message-based: One canonical chain, states synced
     + Unified liquidity
     - Cross-chain latency, oracle dependency
     
  3. Intent-based: Users specify output, fillers route
     + Best UX
     - Requires solver network

SECURITY HIERARCHY
  Priority: Hub chain security > satellite security
  Never: Trust satellite chain to hold canonical assets
  Rule:  Hub can override satellites, never reverse

DEPLOYMENT PRINCIPLE
  Same business logic → both chains
  Chain-specific optimizations → separate modules
  Shared state → canonical hub + message sync
```

---

**ก่อนหน้า**: [Part 93 - Advanced Governance ←](part-93-governance.md)
**ต่อไป**: [Part 95 - Production Launch Checklist →](part-95-launch-checklist.md)
