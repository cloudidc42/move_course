# Part 88: DeFi Indexing & Analytics

## สารบัญ
- [Event Indexing Architecture](#event-indexing-architecture)
- [Aptos Indexer Integration](#aptos-indexer-integration)
- [On-Chain Analytics Modules](#on-chain-analytics-modules)
- [Real-Time Dashboard](#real-time-dashboard)
- [TWAP & Price History](#twap--price-history)
- [Portfolio Tracker](#portfolio-tracker)

---

## Event Indexing Architecture

```
INDEXING PIPELINE

Aptos Fullnode
    │
    ├── REST API /v1/events (poll-based)
    ├── gRPC stream (push-based, low latency)
    └── Indexer Processor (official Aptos indexer)

Your Service
    │
    ├── Event Processor
    │     ├── Parse event types
    │     ├── Decode event data
    │     └── Store in DB
    │
    ├── PostgreSQL / TimescaleDB
    │     ├── events table (raw)
    │     ├── swaps table (decoded)
    │     ├── liquidity table
    │     ├── prices table (OHLCV)
    │     └── user_positions table
    │
    └── API Layer (GraphQL / REST)
          ├── Query historical data
          ├── Aggregate stats (TVL, volume)
          └── WebSocket for live updates

APTOS EVENT TYPES
  {module}::{struct_name}
  
  Examples:
  - protocol::pool::SwapEvent
  - protocol::pool::LiquidityEvent
  - protocol::lending::DepositEvent
  - protocol::lending::BorrowEvent
  - protocol::governance::ProposalCreatedEvent
```

---

## Aptos Indexer Integration

```typescript
// ============================================
// EVENT INDEXER SERVICE
// Polls Aptos events, decodes, stores in DB
// ============================================

import { AptosClient } from 'aptos';
import { Pool } from 'pg';

interface SwapEvent {
  user: string;
  amount_in: string;
  amount_out: string;
  is_x_to_y: boolean;
  reserve_x_after: string;
  reserve_y_after: string;
  timestamp: string;
  sequence_number: string;
}

interface LiquidityEvent {
  user: string;
  amount_x: string;
  amount_y: string;
  lp_tokens: string;
  is_add: boolean;
  timestamp: string;
  sequence_number: string;
}

class AptosEventIndexer {
  private client: AptosClient;
  private db: Pool;
  private contractAddr: string;
  
  constructor(nodeUrl: string, dbUrl: string, contractAddr: string) {
    this.client = new AptosClient(nodeUrl);
    this.db = new Pool({ connectionString: dbUrl });
    this.contractAddr = contractAddr;
  }
  
  async initialize(): Promise<void> {
    await this.db.query(`
      CREATE TABLE IF NOT EXISTS swap_events (
        id SERIAL PRIMARY KEY,
        sequence_number BIGINT UNIQUE,
        user_addr TEXT NOT NULL,
        amount_in NUMERIC NOT NULL,
        amount_out NUMERIC NOT NULL,
        is_x_to_y BOOLEAN NOT NULL,
        reserve_x_after NUMERIC NOT NULL,
        reserve_y_after NUMERIC NOT NULL,
        price_x_in_y NUMERIC GENERATED ALWAYS AS (
          CASE WHEN is_x_to_y THEN amount_out::NUMERIC / NULLIF(amount_in, 0)
               ELSE amount_in::NUMERIC / NULLIF(amount_out, 0) END
        ) STORED,
        event_timestamp BIGINT NOT NULL,
        indexed_at TIMESTAMPTZ DEFAULT NOW()
      );
      
      CREATE TABLE IF NOT EXISTS liquidity_events (
        id SERIAL PRIMARY KEY,
        sequence_number BIGINT UNIQUE,
        user_addr TEXT NOT NULL,
        amount_x NUMERIC NOT NULL,
        amount_y NUMERIC NOT NULL,
        lp_tokens NUMERIC NOT NULL,
        is_add BOOLEAN NOT NULL,
        event_timestamp BIGINT NOT NULL,
        indexed_at TIMESTAMPTZ DEFAULT NOW()
      );
      
      -- OHLCV for charts (1 row per minute)
      CREATE TABLE IF NOT EXISTS price_ohlcv (
        timestamp_minute BIGINT PRIMARY KEY,
        open NUMERIC,
        high NUMERIC,
        low NUMERIC,
        close NUMERIC,
        volume_x NUMERIC DEFAULT 0,
        volume_y NUMERIC DEFAULT 0,
        trade_count INTEGER DEFAULT 0
      );
      
      CREATE INDEX IF NOT EXISTS idx_swap_events_timestamp 
        ON swap_events(event_timestamp DESC);
      CREATE INDEX IF NOT EXISTS idx_swap_events_user 
        ON swap_events(user_addr);
    `);
  }
  
  async getLastIndexedSequence(): Promise<bigint> {
    const result = await this.db.query(
      'SELECT COALESCE(MAX(sequence_number), 0) as max_seq FROM swap_events'
    );
    return BigInt(result.rows[0].max_seq);
  }
  
  async fetchAndIndexEvents(): Promise<void> {
    const lastSeq = await this.getLastIndexedSequence();
    
    // Fetch swap events
    const swapEvents = await this.client.getEventsByEventHandle(
      this.contractAddr,
      `${this.contractAddr}::pool::Pool`,
      'swap_events',
      { start: Number(lastSeq) + 1, limit: 100 },
    );
    
    for (const event of swapEvents) {
      const data = event.data as SwapEvent;
      await this.db.query(`
        INSERT INTO swap_events (
          sequence_number, user_addr, amount_in, amount_out,
          is_x_to_y, reserve_x_after, reserve_y_after, event_timestamp
        ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
        ON CONFLICT (sequence_number) DO NOTHING
      `, [
        event.sequence_number,
        data.user,
        data.amount_in,
        data.amount_out,
        data.is_x_to_y,
        data.reserve_x_after,
        data.reserve_y_after,
        data.timestamp,
      ]);
      
      // Update OHLCV
      await this.updateOHLCV(data);
    }
    
    console.log(`Indexed ${swapEvents.length} new swap events`);
  }
  
  private async updateOHLCV(event: SwapEvent): Promise<void> {
    const timestamp = BigInt(event.timestamp);
    const minuteTs = (timestamp / 60_000_000n) * 60_000_000n;  // Round to minute (microseconds)
    
    // Calculate price (Y per X)
    const amountIn = BigInt(event.amount_in);
    const amountOut = BigInt(event.amount_out);
    const price = event.is_x_to_y
      ? Number(amountOut) / Number(amountIn)
      : Number(amountIn) / Number(amountOut);
    
    const volumeX = event.is_x_to_y ? Number(amountIn) : Number(amountOut);
    const volumeY = event.is_x_to_y ? Number(amountOut) : Number(amountIn);
    
    await this.db.query(`
      INSERT INTO price_ohlcv (timestamp_minute, open, high, low, close, volume_x, volume_y, trade_count)
      VALUES ($1, $2, $2, $2, $2, $3, $4, 1)
      ON CONFLICT (timestamp_minute) DO UPDATE SET
        high = GREATEST(price_ohlcv.high, EXCLUDED.high),
        low = LEAST(price_ohlcv.low, EXCLUDED.low),
        close = EXCLUDED.close,
        volume_x = price_ohlcv.volume_x + EXCLUDED.volume_x,
        volume_y = price_ohlcv.volume_y + EXCLUDED.volume_y,
        trade_count = price_ohlcv.trade_count + 1
    `, [minuteTs, price, volumeX, volumeY]);
  }
  
  async startIndexer(pollIntervalMs: number = 5000): Promise<void> {
    await this.initialize();
    console.log('Indexer started, polling every', pollIntervalMs, 'ms');
    
    while (true) {
      try {
        await this.fetchAndIndexEvents();
      } catch (error) {
        console.error('Indexer error:', error);
      }
      await new Promise(r => setTimeout(r, pollIntervalMs));
    }
  }
}
```

---

## On-Chain Analytics Modules

```move
// ============================================
// ON-CHAIN ANALYTICS: Track stats directly
// ============================================

module protocol::analytics {
    use aptos_framework::timestamp;
    
    /// Snapshot of protocol state at a point in time
    struct ProtocolSnapshot has store {
        timestamp: u64,
        tvl_x: u64,
        tvl_y: u64,
        volume_24h_x: u64,
        volume_24h_y: u64,
        fee_24h: u64,
        unique_users_24h: u64,
    }
    
    /// Rolling 24-hour stats
    struct RollingStats has key {
        // Volume buckets: 24 slots, one per hour
        hourly_volume_x: vector<u64>,
        hourly_volume_y: vector<u64>,
        hourly_fees: vector<u64>,
        hourly_trades: vector<u64>,
        
        // Current hour slot (0-23)
        current_hour_slot: u64,
        last_update_hour: u64,
        
        // All-time stats
        total_volume_x: u128,
        total_volume_y: u128,
        total_fees: u128,
        total_trades: u128,
    }
    
    /// Per-user stats
    struct UserStats has key {
        total_swaps: u64,
        total_volume_x: u128,
        total_volume_y: u128,
        total_lp_added: u128,
        total_lp_removed: u128,
        first_swap_ts: u64,
        last_swap_ts: u64,
    }
    
    fun get_hour_slot(): u64 {
        (timestamp::now_microseconds() / 3_600_000_000) % 24
    }
    
    fun get_current_hour(): u64 {
        timestamp::now_microseconds() / 3_600_000_000
    }
    
    public fun record_swap(
        stats: &mut RollingStats,
        volume_x: u64,
        volume_y: u64,
        fee: u64,
    ) {
        let current_hour = get_current_hour();
        let slot = get_hour_slot();
        
        // Clear old data when hour changes
        if (current_hour > stats.last_update_hour) {
            // How many hours passed? Clear those slots
            let hours_passed = current_hour - stats.last_update_hour;
            let clear_count = std::u64::min(hours_passed, 24);
            
            let i = 0u64;
            while (i < clear_count) {
                let clear_slot = (slot + 24 - i) % 24;
                *vector::borrow_mut(&mut stats.hourly_volume_x, clear_slot) = 0;
                *vector::borrow_mut(&mut stats.hourly_volume_y, clear_slot) = 0;
                *vector::borrow_mut(&mut stats.hourly_fees, clear_slot) = 0;
                *vector::borrow_mut(&mut stats.hourly_trades, clear_slot) = 0;
                i = i + 1;
            };
            
            stats.current_hour_slot = slot;
            stats.last_update_hour = current_hour;
        };
        
        // Add to current hour bucket
        let cur_vol_x = vector::borrow_mut(&mut stats.hourly_volume_x, slot);
        *cur_vol_x = *cur_vol_x + volume_x;
        
        let cur_vol_y = vector::borrow_mut(&mut stats.hourly_volume_y, slot);
        *cur_vol_y = *cur_vol_y + volume_y;
        
        let cur_fee = vector::borrow_mut(&mut stats.hourly_fees, slot);
        *cur_fee = *cur_fee + fee;
        
        let cur_trades = vector::borrow_mut(&mut stats.hourly_trades, slot);
        *cur_trades = *cur_trades + 1;
        
        // All-time totals
        stats.total_volume_x = stats.total_volume_x + (volume_x as u128);
        stats.total_volume_y = stats.total_volume_y + (volume_y as u128);
        stats.total_fees = stats.total_fees + (fee as u128);
        stats.total_trades = stats.total_trades + 1;
    }
    
    /// Get 24h rolling volume (sum all hourly buckets)
    public fun get_volume_24h(stats: &RollingStats): (u64, u64, u64) {
        let vol_x = 0u64;
        let vol_y = 0u64;
        let fees = 0u64;
        
        let i = 0u64;
        while (i < 24) {
            vol_x = vol_x + *vector::borrow(&stats.hourly_volume_x, i);
            vol_y = vol_y + *vector::borrow(&stats.hourly_volume_y, i);
            fees = fees + *vector::borrow(&stats.hourly_fees, i);
            i = i + 1;
        };
        
        (vol_x, vol_y, fees)
    }
    
    public fun record_user_swap(
        user_stats: &mut UserStats,
        volume_x: u64,
        volume_y: u64,
    ) {
        let now = timestamp::now_microseconds();
        
        user_stats.total_swaps = user_stats.total_swaps + 1;
        user_stats.total_volume_x = user_stats.total_volume_x + (volume_x as u128);
        user_stats.total_volume_y = user_stats.total_volume_y + (volume_y as u128);
        user_stats.last_swap_ts = now;
        
        if (user_stats.first_swap_ts == 0) {
            user_stats.first_swap_ts = now;
        };
    }
    
    #[view]
    public fun get_apr(
        stats: &RollingStats,
        tvl_x: u64,
        tvl_y: u64,
        price_y_in_x: u64,  // 1e8 scaled
    ): u64 {
        let (_, _, fees_24h) = get_volume_24h(stats);
        
        // TVL in X terms
        let tvl_y_in_x = (tvl_y as u128) * (price_y_in_x as u128) / 100_000_000;
        let total_tvl = (tvl_x as u128) + tvl_y_in_x;
        
        if (total_tvl == 0) return 0;
        
        // APR = (fees_24h / TVL) * 365 * 10000 (bps)
        let daily_rate = (fees_24h as u128) * 10_000 / total_tvl;
        (daily_rate * 365) as u64
    }
}
```

---

## Real-Time Dashboard

```typescript
// ============================================
// ANALYTICS API SERVER
// Serves data to dashboards
// ============================================

import express from 'express';
import { Pool } from 'pg';
import { WebSocketServer } from 'ws';

const app = express();
const db = new Pool({ connectionString: process.env.DATABASE_URL });

// GET /api/stats/overview
app.get('/api/stats/overview', async (req, res) => {
  const result = await db.query(`
    SELECT
      -- 24h volume
      SUM(CASE WHEN event_timestamp > (EXTRACT(EPOCH FROM NOW()) * 1e6 - 86400000000)
          THEN amount_in ELSE 0 END) as volume_24h,
      -- 24h fees (0.3%)
      SUM(CASE WHEN event_timestamp > (EXTRACT(EPOCH FROM NOW()) * 1e6 - 86400000000)
          THEN amount_in * 0.003 ELSE 0 END) as fees_24h,
      -- 24h trade count
      COUNT(CASE WHEN event_timestamp > (EXTRACT(EPOCH FROM NOW()) * 1e6 - 86400000000)
          THEN 1 END) as trades_24h,
      -- All-time volume
      SUM(amount_in) as volume_all_time,
      -- Unique traders 24h
      COUNT(DISTINCT CASE WHEN event_timestamp > (EXTRACT(EPOCH FROM NOW()) * 1e6 - 86400000000)
          THEN user_addr END) as unique_traders_24h
    FROM swap_events
  `);
  
  res.json(result.rows[0]);
});

// GET /api/stats/ohlcv?interval=1m&from=...&to=...
app.get('/api/stats/ohlcv', async (req, res) => {
  const { interval = '1m', from, to } = req.query;
  
  // Determine grouping
  const bucketMap: Record<string, string> = {
    '1m': '60000000',     // 1 minute in microseconds
    '5m': '300000000',
    '1h': '3600000000',
    '1d': '86400000000',
  };
  const bucket = bucketMap[interval as string] || '60000000';
  
  const result = await db.query(`
    SELECT
      (event_timestamp / $1) * $1 as timestamp_bucket,
      MIN(price_x_in_y) as low,
      MAX(price_x_in_y) as high,
      (ARRAY_AGG(price_x_in_y ORDER BY sequence_number ASC))[1] as open,
      (ARRAY_AGG(price_x_in_y ORDER BY sequence_number DESC))[1] as close,
      SUM(CASE WHEN is_x_to_y THEN amount_in ELSE amount_out END) as volume_x
    FROM swap_events
    WHERE event_timestamp >= $2 AND event_timestamp <= $3
    GROUP BY timestamp_bucket
    ORDER BY timestamp_bucket
  `, [bucket, from, to]);
  
  res.json(result.rows);
});

// GET /api/stats/top-traders?period=24h
app.get('/api/stats/top-traders', async (req, res) => {
  const { period = '24h', limit = 10 } = req.query;
  const cutoffHours = period === '24h' ? 24 : period === '7d' ? 168 : 720;
  
  const result = await db.query(`
    SELECT
      user_addr,
      COUNT(*) as trade_count,
      SUM(amount_in) as volume,
      COUNT(DISTINCT DATE_TRUNC('day', TO_TIMESTAMP(event_timestamp / 1e6))) as active_days
    FROM swap_events
    WHERE event_timestamp > (EXTRACT(EPOCH FROM NOW()) * 1e6 - $1 * 3600000000)
    GROUP BY user_addr
    ORDER BY volume DESC
    LIMIT $2
  `, [cutoffHours, limit]);
  
  res.json(result.rows);
});

// GET /api/user/:address/portfolio
app.get('/api/user/:address/portfolio', async (req, res) => {
  const { address } = req.params;
  
  const [swapStats, liquidityStats] = await Promise.all([
    db.query(`
      SELECT
        COUNT(*) as total_swaps,
        SUM(amount_in) as total_volume,
        MAX(event_timestamp) as last_activity
      FROM swap_events WHERE user_addr = $1
    `, [address]),
    db.query(`
      SELECT
        SUM(CASE WHEN is_add THEN lp_tokens ELSE -lp_tokens END) as net_lp_position,
        SUM(CASE WHEN is_add THEN amount_x ELSE 0 END) as total_x_added,
        SUM(CASE WHEN is_add THEN amount_y ELSE 0 END) as total_y_added
      FROM liquidity_events WHERE user_addr = $1
    `, [address]),
  ]);
  
  res.json({
    address,
    swaps: swapStats.rows[0],
    liquidity: liquidityStats.rows[0],
  });
});

// WebSocket: Stream live trades
const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (ws) => {
  console.log('Client connected for live feed');
  
  // Send last 10 trades on connection
  db.query(
    'SELECT * FROM swap_events ORDER BY sequence_number DESC LIMIT 10'
  ).then(result => {
    ws.send(JSON.stringify({ type: 'history', data: result.rows }));
  });
});

// Push new trades to all connected clients
async function broadcastNewTrades() {
  let lastSeq = 0n;
  
  setInterval(async () => {
    try {
      const result = await db.query(
        'SELECT * FROM swap_events WHERE sequence_number > $1 ORDER BY sequence_number',
        [lastSeq]
      );
      
      if (result.rows.length > 0) {
        lastSeq = BigInt(result.rows[result.rows.length - 1].sequence_number);
        
        const message = JSON.stringify({ type: 'trades', data: result.rows });
        wss.clients.forEach(client => {
          if (client.readyState === 1) client.send(message);
        });
      }
    } catch (error) {
      console.error('Broadcast error:', error);
    }
  }, 2000);
}

app.listen(3000, () => {
  console.log('Analytics API running on port 3000');
  broadcastNewTrades();
});
```

---

## TWAP & Price History

```move
// ============================================
// ON-CHAIN TWAP ORACLE
// Stores cumulative price for TWAP calculation
// ============================================

module protocol::twap_oracle {
    use aptos_framework::timestamp;
    
    const PRECISION: u128 = 1_000_000_000_000;  // 1e12 for fixed-point
    
    struct PriceAccumulator has key {
        // Cumulative prices (price * time_elapsed, fixed-point)
        cumulative_price_x: u256,
        cumulative_price_y: u256,
        
        // Last snapshot values
        last_price_x: u128,   // X in terms of Y (1e12 scaled)
        last_price_y: u128,   // Y in terms of X (1e12 scaled)
        last_timestamp: u64,
        
        // Snapshots for TWAP calculation (ring buffer, 60 slots = 60 minutes)
        snapshots: vector<PriceSnapshot>,
        snapshot_count: u64,
        snapshot_interval: u64,  // microseconds between snapshots
    }
    
    struct PriceSnapshot has store, copy, drop {
        cumulative_x: u256,
        cumulative_y: u256,
        timestamp: u64,
    }
    
    /// Called after every swap to update accumulator
    public fun update(
        acc: &mut PriceAccumulator,
        reserve_x: u64,
        reserve_y: u64,
    ) {
        let now = timestamp::now_microseconds();
        let elapsed = now - acc.last_timestamp;
        
        if (elapsed == 0) return;
        
        // Calculate spot prices
        let price_x = (reserve_y as u128) * PRECISION / (reserve_x as u128);
        let price_y = (reserve_x as u128) * PRECISION / (reserve_y as u128);
        
        // Accumulate: price * time_elapsed
        acc.cumulative_price_x = acc.cumulative_price_x 
            + (acc.last_price_x as u256) * (elapsed as u256);
        acc.cumulative_price_y = acc.cumulative_price_y 
            + (acc.last_price_y as u256) * (elapsed as u256);
        
        // Save new snapshot if enough time has passed
        if (elapsed >= acc.snapshot_interval) {
            let snap = PriceSnapshot {
                cumulative_x: acc.cumulative_price_x,
                cumulative_y: acc.cumulative_price_y,
                timestamp: now,
            };
            
            // Ring buffer: overwrite oldest
            let slot = (acc.snapshot_count % (vector::length(&acc.snapshots) as u64)) as u64;
            if (slot < (vector::length(&acc.snapshots) as u64)) {
                *vector::borrow_mut(&mut acc.snapshots, slot) = snap;
            } else {
                vector::push_back(&mut acc.snapshots, snap);
            };
            acc.snapshot_count = acc.snapshot_count + 1;
        };
        
        // Update last values
        acc.last_price_x = price_x;
        acc.last_price_y = price_y;
        acc.last_timestamp = now;
    }
    
    /// Calculate TWAP over last N seconds
    #[view]
    public fun get_twap(
        acc: &PriceAccumulator,
        period_seconds: u64,
    ): (u128, u128) {
        let now = timestamp::now_microseconds();
        let period_us = (period_seconds as u64) * 1_000_000;
        let target_ts = now - period_us;
        
        // Find oldest snapshot within the period
        let best_snap: Option<PriceSnapshot> = option::none();
        
        let i = 0u64;
        let len = vector::length(&acc.snapshots);
        while (i < len) {
            let snap = vector::borrow(&acc.snapshots, i);
            if (snap.timestamp >= target_ts) {
                if (option::is_none(&best_snap)) {
                    best_snap = option::some(*snap);
                } else {
                    let current = option::borrow(&best_snap);
                    if (snap.timestamp < current.timestamp) {
                        best_snap = option::some(*snap);
                    };
                };
            };
            i = i + 1;
        };
        
        if (option::is_none(&best_snap)) {
            // No snapshot, return current price
            return (acc.last_price_x, acc.last_price_y)
        };
        
        let snap = option::borrow(&best_snap);
        let time_delta = now - snap.timestamp;
        
        if (time_delta == 0) {
            return (acc.last_price_x, acc.last_price_y)
        };
        
        // TWAP = (cumulative_now - cumulative_then) / time_delta
        let cum_delta_x = acc.cumulative_price_x - snap.cumulative_x;
        let cum_delta_y = acc.cumulative_price_y - snap.cumulative_y;
        
        let twap_x = (cum_delta_x / (time_delta as u256)) as u128;
        let twap_y = (cum_delta_y / (time_delta as u256)) as u128;
        
        (twap_x, twap_y)
    }
}
```

---

## Portfolio Tracker

```typescript
// ============================================
// PORTFOLIO TRACKER
// Track user's positions across protocols
// ============================================

interface TokenBalance {
  coinType: string;
  symbol: string;
  amount: bigint;
  decimals: number;
  usdValue: number;
}

interface LPPosition {
  poolAddr: string;
  lpTokenAmount: bigint;
  underlyingX: bigint;
  underlyingY: bigint;
  usdValue: number;
  feesEarned: number;
  aprEstimate: number;
}

interface LendingPosition {
  protocol: string;
  asset: string;
  supplied: bigint;
  borrowed: bigint;
  netApy: number;
  healthFactor: number;
}

interface Portfolio {
  address: string;
  totalUsdValue: number;
  tokens: TokenBalance[];
  lpPositions: LPPosition[];
  lendingPositions: LendingPosition[];
  timestamp: number;
}

class PortfolioTracker {
  private client: any;
  private priceApi: string;
  
  constructor(nodeUrl: string, priceApi: string) {
    this.priceApi = priceApi;
  }
  
  async fetchPortfolio(address: string): Promise<Portfolio> {
    const [tokens, lpPositions, lendingPositions] = await Promise.all([
      this.fetchTokenBalances(address),
      this.fetchLPPositions(address),
      this.fetchLendingPositions(address),
    ]);
    
    const totalUsdValue = 
      tokens.reduce((sum, t) => sum + t.usdValue, 0) +
      lpPositions.reduce((sum, p) => sum + p.usdValue, 0) +
      lendingPositions.reduce((sum, p) => sum + (Number(p.supplied) - Number(p.borrowed)) / 1e6, 0);
    
    return {
      address,
      totalUsdValue,
      tokens,
      lpPositions,
      lendingPositions,
      timestamp: Date.now(),
    };
  }
  
  private async fetchTokenBalances(address: string): Promise<TokenBalance[]> {
    // Fetch all CoinStore resources for the address
    const resources = await this.client.getAccountResources(address);
    const coinResources = resources.filter((r: any) => r.type.includes('CoinStore'));
    
    const prices = await this.fetchPrices(
      coinResources.map((r: any) => this.extractCoinType(r.type))
    );
    
    return coinResources.map((r: any) => {
      const coinType = this.extractCoinType(r.type);
      const amount = BigInt(r.data.coin.value);
      const decimals = 8;  // Default, should fetch from CoinInfo
      const price = prices[coinType] || 0;
      const usdValue = Number(amount) / 10 ** decimals * price;
      
      return { coinType, symbol: this.coinTypeToSymbol(coinType), amount, decimals, usdValue };
    });
  }
  
  private async fetchLPPositions(address: string): Promise<LPPosition[]> {
    // Fetch LP token balances
    // Calculate underlying amounts from LP share
    // Estimate fees earned from indexer data
    return [];  // Simplified
  }
  
  private async fetchLendingPositions(address: string): Promise<LendingPosition[]> {
    return [];  // Simplified
  }
  
  private async fetchPrices(coinTypes: string[]): Promise<Record<string, number>> {
    const response = await fetch(`${this.priceApi}/prices?coins=${coinTypes.join(',')}`);
    return response.json();
  }
  
  private extractCoinType(resourceType: string): string {
    const match = resourceType.match(/CoinStore<(.+)>/);
    return match ? match[1] : '';
  }
  
  private coinTypeToSymbol(coinType: string): string {
    const symbolMap: Record<string, string> = {
      '0x1::aptos_coin::AptosCoin': 'APT',
    };
    return symbolMap[coinType] || coinType.split('::').pop() || 'UNKNOWN';
  }
}
```

---

## สรุป DeFi Indexing & Analytics

```
KEY INDEXING PATTERNS

1. EVENT-BASED INDEXING
   - Poll /v1/events or use gRPC stream
   - Store sequence_number for deduplication
   - Process events in order (sequence matters)
   - Handle gaps: sequence_number continuity

2. ON-CHAIN ANALYTICS
   - Rolling 24h stats via hourly ring buffer
   - Cheap to compute (no complex queries on-chain)
   - TWAP: cumulative price * time, snap-and-diff

3. OFF-CHAIN INDEXING (Recommended)
   - More flexibility, complex queries
   - TimescaleDB for time-series (OHLCV, volume)
   - PostgreSQL for user stats, positions
   - Redis for real-time aggregates (cache)

4. API DESIGN
   - REST for bulk queries (historical OHLCV)
   - WebSocket for live trade stream
   - GraphQL for flexible portfolio queries

5. PERFORMANCE TIPS
   - Index on (timestamp, user_addr) for common queries
   - Pre-aggregate 24h/7d stats periodically
   - Cache price data (1-5 second TTL)
   - Paginate all list endpoints

ANALYTICS SCHEMA CHECKLIST
  ✅ swap_events (raw, indexed by time + user)
  ✅ liquidity_events (add/remove LP)
  ✅ price_ohlcv (aggregated, 1m/5m/1h/1d)
  ✅ protocol_snapshots (daily TVL, fees)
  ✅ user_lifetime_stats (materialized view)
```

---

**ก่อนหน้า**: [Part 87 - Advanced Testing Strategies ←](part-87-testing.md)
**ต่อไป**: [Part 89 - Gas Optimization →](part-89-gas-optimization.md)
