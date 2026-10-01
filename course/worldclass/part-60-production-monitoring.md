# Part 60: Production Deployment & Monitoring

## สารบัญ
- [Production Checklist](#production-checklist)
- [Multi-Environment Deployment](#multi-environment-deployment)
- [On-Chain Monitoring](#on-chain-monitoring)
- [Alert Systems](#alert-systems)
- [Incident Response](#incident-response)
- [Emergency Controls](#emergency-controls)
- [Protocol Health Dashboard](#protocol-health-dashboard)

---

## Production Checklist

```
Pre-Deploy Checklist:

SECURITY
  □ 3+ independent audits (Certik, OtterSec, MoveBit, Halborn)
  □ Bug bounty active ($50k-$500k range)
  □ Move Prover specs for critical invariants
  □ Fuzz tested with 100k+ random inputs
  □ All admin functions require multi-sig
  □ Emergency pause implemented
  □ No hardcoded addresses (use config objects)
  □ No integer overflow opportunities
  □ All external calls validated
  □ Replay protection in place

DEPLOYMENT
  □ Testnet deployment (2+ weeks public testing)
  □ Mainnet deploy plan documented
  □ Rollback plan ready
  □ Upgrade strategy defined (proxy vs versioned)
  □ Module addresses documented
  □ Move.toml locked to exact versions

MONITORING
  □ On-chain event indexer running
  □ TVL monitoring in place
  □ Price feeds cross-validated
  □ Alert thresholds set
  □ On-call rotation defined
  □ Incident runbook ready

OPERATIONS
  □ Multi-sig signers (3/5 minimum)
  □ Hardware wallets for admin keys
  □ Cold storage for protocol treasury
  □ Key ceremony documented
  □ Signer rotation plan

LEGAL
  □ Terms of service
  □ Jurisdiction analysis
  □ OFAC compliance check
  □ Privacy policy
```

---

## Multi-Environment Deployment

```bash
#!/bin/bash
# deploy.sh: Multi-environment deployment script

set -e

ENVIRONMENT=${1:-testnet}
PACKAGE_NAME="defi_protocol"

# Environment-specific config
if [ "$ENVIRONMENT" == "testnet" ]; then
    PROFILE="testnet"
    NAMED_ADDRESS="defi=0x0"  # Let framework assign
    URL="https://fullnode.testnet.aptoslabs.com/v1"
elif [ "$ENVIRONMENT" == "mainnet" ]; then
    PROFILE="mainnet"
    NAMED_ADDRESS="defi=0x$(cat .addresses/mainnet_deployer)"
    URL="https://fullnode.mainnet.aptoslabs.com/v1"
else
    echo "Unknown environment: $ENVIRONMENT"
    exit 1
fi

echo "Deploying to $ENVIRONMENT..."

# Step 1: Validate code before deploying
echo "Running tests..."
aptos move test --named-addresses "$NAMED_ADDRESS"

echo "Running Move Prover..."
aptos move prove --named-addresses "$NAMED_ADDRESS" 2>&1 | tail -5

# Step 2: Compile and check bytecode size
echo "Compiling..."
aptos move compile --named-addresses "$NAMED_ADDRESS"

# Check compiled size
BYTECODE_SIZE=$(du -s .aptos/build/*/bytecode_modules/ 2>/dev/null | awk '{sum+=$1} END{print sum}')
echo "Bytecode size: ${BYTECODE_SIZE}KB"

if [ "$BYTECODE_SIZE" -gt 500 ]; then
    echo "Warning: Large bytecode size, check for code bloat"
fi

# Step 3: Deploy
echo "Publishing package..."
DEPLOY_OUTPUT=$(aptos move publish \
    --profile "$PROFILE" \
    --named-addresses "$NAMED_ADDRESS" \
    --max-gas 1000000 \
    --assume-yes \
    2>&1)

echo "$DEPLOY_OUTPUT"

# Step 4: Extract deployed address
PACKAGE_ADDRESS=$(echo "$DEPLOY_OUTPUT" | grep -oP '"address": "\K[^"]+' | head -1)

if [ -z "$PACKAGE_ADDRESS" ]; then
    echo "ERROR: Could not extract package address"
    exit 1
fi

echo "Package deployed at: $PACKAGE_ADDRESS"

# Step 5: Save deployment info
mkdir -p .deployments
cat > ".deployments/${ENVIRONMENT}_latest.json" << EOF
{
  "environment": "$ENVIRONMENT",
  "package_address": "$PACKAGE_ADDRESS",
  "deployed_at": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "git_commit": "$(git rev-parse HEAD)",
  "deployer": "$(aptos account list --profile $PROFILE 2>/dev/null | grep -oP 'Account Address: \K.*' | head -1)"
}
EOF

# Step 6: Initialize protocol (first deployment only)
if [ ! -f ".deployments/${ENVIRONMENT}_initialized" ]; then
    echo "Initializing protocol..."
    aptos move run \
        --profile "$PROFILE" \
        --function-id "${PACKAGE_ADDRESS}::protocol::initialize" \
        --args "u64:30" \
        --assume-yes
    
    touch ".deployments/${ENVIRONMENT}_initialized"
fi

echo "Deployment complete!"
echo "Package: $PACKAGE_ADDRESS"
```

---

## On-Chain Monitoring

```typescript
// monitor.ts: Real-time on-chain protocol monitoring

import { AptosClient, AptosAccount } from 'aptos';
import * as prometheus from 'prom-client';
import express from 'express';

const client = new AptosClient('https://fullnode.mainnet.aptoslabs.com/v1');

// ============================================
// Prometheus metrics
// ============================================

const metrics = {
  tvl: new prometheus.Gauge({
    name: 'protocol_tvl_usd',
    help: 'Total Value Locked in USD',
    labelNames: ['pool', 'token'],
  }),
  
  volume24h: new prometheus.Gauge({
    name: 'protocol_volume_24h_usd',
    help: '24-hour trading volume in USD',
    labelNames: ['pool'],
  }),
  
  swapCount: new prometheus.Counter({
    name: 'protocol_swap_total',
    help: 'Total number of swaps',
    labelNames: ['pool', 'direction'],
  }),
  
  errorRate: new prometheus.Gauge({
    name: 'protocol_error_rate',
    help: 'Transaction error rate (0-1)',
    labelNames: ['function'],
  }),
  
  latestBlock: new prometheus.Gauge({
    name: 'chain_latest_block',
    help: 'Latest processed block number',
  }),
  
  oracleFreshness: new prometheus.Gauge({
    name: 'oracle_price_age_seconds',
    help: 'Age of oracle price in seconds',
    labelNames: ['pair'],
  }),
  
  borrowRate: new prometheus.Gauge({
    name: 'lending_borrow_rate',
    help: 'Current borrow rate (annualized)',
    labelNames: ['asset'],
  }),
};

// ============================================
// Protocol state fetcher
// ============================================

interface PoolState {
  address: string;
  reserveX: bigint;
  reserveY: bigint;
  totalLp: bigint;
  feeBps: number;
}

async function fetchPoolState(poolAddress: string): Promise<PoolState> {
  const resource = await client.getAccountResource(
    poolAddress,
    `${PACKAGE_ADDRESS}::pool::Pool<${TOKEN_X_TYPE}, ${TOKEN_Y_TYPE}>`
  );
  
  const data = resource.data as any;
  
  return {
    address: poolAddress,
    reserveX: BigInt(data.reserve_x.value),
    reserveY: BigInt(data.reserve_y.value),
    totalLp: BigInt(data.lp_supply),
    feeBps: parseInt(data.fee_bps),
  };
}

// ============================================
// Event listener
// ============================================

async function listenToEvents() {
  let cursor = await getStoredCursor();  // Persist cursor to DB
  
  while (true) {
    try {
      const events = await client.getEventsByEventHandle(
        POOL_ADDRESS,
        `${PACKAGE_ADDRESS}::pool::Pool<${TOKEN_X_TYPE}, ${TOKEN_Y_TYPE}>`,
        'swap_events',
        {
          start: cursor,
          limit: 100,
        }
      );
      
      for (const event of events) {
        await processSwapEvent(event);
        cursor = BigInt(event.sequence_number) + 1n;
      }
      
      await saveStoredCursor(cursor);
      
      if (events.length < 100) {
        // Caught up, wait for new events
        await sleep(1000);
      }
      
    } catch (error) {
      console.error('Event listener error:', error);
      await sleep(5000);
    }
  }
}

async function processSwapEvent(event: any) {
  const { pool_id, is_x_to_y, amount_in, amount_out, fee } = event.data;
  
  // Update Prometheus metrics
  metrics.swapCount.inc({ 
    pool: pool_id, 
    direction: is_x_to_y ? 'x_to_y' : 'y_to_x' 
  });
  
  // Calculate USD value (assuming we have price feed)
  const usdValue = await calculateUsdValue(amount_in, is_x_to_y ? TOKEN_X_TYPE : TOKEN_Y_TYPE);
  
  metrics.volume24h.inc({ pool: pool_id }, usdValue);
  
  // Store in database for analytics
  await db.insert('swap_events', {
    pool_id,
    is_x_to_y,
    amount_in: amount_in.toString(),
    amount_out: amount_out.toString(),
    fee: fee.toString(),
    usd_value: usdValue,
    tx_hash: event.transaction_hash,
    timestamp: Date.now(),
  });
}

// ============================================
// Health checker
// ============================================

class ProtocolHealthChecker {
  private alerts: AlertManager;
  
  constructor() {
    this.alerts = new AlertManager();
  }
  
  async checkAll(): Promise<void> {
    await Promise.all([
      this.checkTVL(),
      this.checkOracleFreshness(),
      this.checkLiquidationHealth(),
      this.checkBridgeSolvency(),
    ]);
  }
  
  async checkTVL(): Promise<void> {
    const pool = await fetchPoolState(POOL_ADDRESS);
    const tvl = await calculatePoolTvlUsd(pool);
    
    metrics.tvl.set({ pool: POOL_ADDRESS, token: 'TOTAL' }, tvl);
    
    // Alert if TVL drops > 20% in 1 hour
    const previousTvl = await db.getMetric('tvl_1h_ago', POOL_ADDRESS);
    if (previousTvl && tvl < previousTvl * 0.8) {
      await this.alerts.send({
        severity: 'HIGH',
        title: 'TVL Drop Alert',
        message: `TVL dropped ${((1 - tvl/previousTvl) * 100).toFixed(1)}% in 1 hour`,
        value: tvl,
      });
    }
  }
  
  async checkOracleFreshness(): Promise<void> {
    const oracleResource = await client.getAccountResource(
      ORACLE_ADDRESS,
      `${ORACLE_PACKAGE}::oracle::PriceData`
    );
    
    const lastUpdate = parseInt((oracleResource.data as any).last_update);
    const age = Math.floor(Date.now() / 1000) - lastUpdate;
    
    metrics.oracleFreshness.set({ pair: 'ETH/USD' }, age);
    
    // Alert if oracle hasn't updated in 5 minutes
    if (age > 300) {
      await this.alerts.send({
        severity: 'CRITICAL',
        title: 'Oracle Stale',
        message: `Oracle price not updated for ${age}s`,
        pair: 'ETH/USD',
      });
    }
  }
  
  async checkLiquidationHealth(): Promise<void> {
    // Fetch all at-risk positions
    const atRiskPositions = await fetchPositionsBelowHealth(1.2);
    
    if (atRiskPositions.length > 0) {
      await this.alerts.send({
        severity: 'MEDIUM',
        title: 'Liquidatable Positions',
        message: `${atRiskPositions.length} positions at risk of liquidation`,
        positions: atRiskPositions.slice(0, 5),
      });
    }
  }
  
  async checkBridgeSolvency(): Promise<void> {
    const bridgeBalance = await fetchBridgeBalance();
    const pendingWithdrawals = await fetchPendingWithdrawals();
    
    if (bridgeBalance < pendingWithdrawals * BigInt(110) / BigInt(100)) {
      await this.alerts.send({
        severity: 'CRITICAL',
        title: 'Bridge Undercollateralized',
        message: `Bridge balance insufficient for pending withdrawals`,
      });
    }
  }
}

// ============================================
// Metrics server
// ============================================

const app = express();

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', prometheus.register.contentType);
  res.end(await prometheus.register.metrics());
});

app.get('/health', async (req, res) => {
  const block = await client.getLedgerInfo();
  res.json({ 
    status: 'ok', 
    block: block.ledger_version,
    timestamp: Date.now() 
  });
});

app.listen(8080, () => console.log('Metrics server on :8080'));

// Start monitoring loops
listenToEvents();
setInterval(() => new ProtocolHealthChecker().checkAll(), 30_000);

// Helpers
async function sleep(ms: number) { return new Promise(r => setTimeout(r, ms)); }
async function getStoredCursor(): Promise<bigint> { return 0n; }
async function saveStoredCursor(cursor: bigint): Promise<void> {}
async function calculateUsdValue(amount: any, tokenType: string): Promise<number> { return 0; }
async function calculatePoolTvlUsd(pool: PoolState): Promise<number> { return 0; }
async function fetchPositionsBelowHealth(healthFactor: number): Promise<any[]> { return []; }
async function fetchBridgeBalance(): Promise<bigint> { return 0n; }
async function fetchPendingWithdrawals(): Promise<bigint> { return 0n; }

class AlertManager {
  async send(alert: any): Promise<void> {
    // Send to PagerDuty, Slack, etc.
    console.error('ALERT:', JSON.stringify(alert));
  }
}

const db = {
  async insert(table: string, data: any) {},
  async getMetric(metric: string, label: string): Promise<number | null> { return null; },
};

const PACKAGE_ADDRESS = '0x1234';
const POOL_ADDRESS = '0x5678';
const ORACLE_ADDRESS = '0x9abc';
const ORACLE_PACKAGE = '0xdef0';
const TOKEN_X_TYPE = 'SUI';
const TOKEN_Y_TYPE = 'USDC';
```

---

## Emergency Controls

```move
module protocol::emergency {
    
    // ============================================
    // Emergency control system
    // ============================================
    
    struct EmergencyConfig has key {
        paused: bool,
        paused_functions: aptos_std::smart_table::SmartTable<std::string::String, bool>,
        
        // Who can pause (multiple guardians)
        guardians: vector<address>,
        
        // Who can unpause (higher threshold)
        council: vector<address>,
        council_threshold: u64,
        
        // Pending unpause (multi-sig)
        unpause_approvals: vector<address>,
        
        pause_timestamp: u64,
        pause_reason: std::string::String,
    }
    
    // Pause specific function (single guardian can do this)
    public entry fun pause_function(
        guardian: &signer,
        config_addr: address,
        function_name: std::string::String,
        reason: std::string::String,
    ) acquires EmergencyConfig {
        let config = borrow_global_mut<EmergencyConfig>(config_addr);
        let caller = std::signer::address_of(guardian);
        
        assert!(std::vector::contains(&config.guardians, &caller), 1);
        
        aptos_std::smart_table::upsert(&mut config.paused_functions, function_name, true);
        config.pause_timestamp = aptos_framework::timestamp::now_seconds();
        config.pause_reason = reason;
        
        // Emit event for monitoring
    }
    
    // Pause entire protocol (single guardian)
    public entry fun pause_all(
        guardian: &signer,
        config_addr: address,
        reason: std::string::String,
    ) acquires EmergencyConfig {
        let config = borrow_global_mut<EmergencyConfig>(config_addr);
        let caller = std::signer::address_of(guardian);
        
        assert!(std::vector::contains(&config.guardians, &caller), 1);
        
        config.paused = true;
        config.pause_timestamp = aptos_framework::timestamp::now_seconds();
        config.pause_reason = reason;
    }
    
    // Approve unpause (council member)
    public entry fun approve_unpause(
        council_member: &signer,
        config_addr: address,
    ) acquires EmergencyConfig {
        let config = borrow_global_mut<EmergencyConfig>(config_addr);
        let caller = std::signer::address_of(council_member);
        
        assert!(std::vector::contains(&config.council, &caller), 1);
        assert!(!std::vector::contains(&config.unpause_approvals, &caller), 2);
        
        std::vector::push_back(&mut config.unpause_approvals, caller);
    }
    
    // Execute unpause (after threshold)
    public entry fun execute_unpause(
        caller: &signer,
        config_addr: address,
    ) acquires EmergencyConfig {
        let config = borrow_global_mut<EmergencyConfig>(config_addr);
        
        assert!(
            std::vector::length(&config.unpause_approvals) >= config.council_threshold,
            1
        );
        
        config.paused = false;
        config.unpause_approvals = std::vector::empty();
    }
    
    // Check if function is paused (inline in every entry function)
    public fun assert_not_paused(config: &EmergencyConfig, function_name: std::string::String) {
        assert!(!config.paused, 100);
        assert!(
            !aptos_std::smart_table::contains(&config.paused_functions, function_name),
            101
        );
    }
    
    // Guard macro (use at start of every critical function)
    #[inline]
    public fun check_paused(config_addr: address, function_name: std::string::String) acquires EmergencyConfig {
        let config = borrow_global<EmergencyConfig>(config_addr);
        assert_not_paused(config, function_name);
    }
}
```

---

## สรุป Production Deployment

```
Production Operations Runbook:

DEPLOY PROCESS
  1. Code freeze → final audit
  2. Testnet stress test (7-14 days)
  3. Mainnet deploy (admin-only initially)
  4. Gradual limits increase
  5. Full public launch

MONITORING STACK
  Events → Indexer → TimescaleDB → Grafana
                   ↓
               Prometheus metrics
                   ↓
           PagerDuty alerts

KEY ALERT THRESHOLDS
  TVL drop >10% in 1h → WARN
  TVL drop >20% in 1h → CRITICAL (page on-call)
  Oracle stale >5min → CRITICAL
  Liquidatable position >$1M → HIGH
  Bridge insolvency risk → CRITICAL (auto-pause)
  Error rate >1% → HIGH

ON-CALL ROTATION
  Primary: 8am-8pm local time
  Secondary: 8pm-8am local time
  Escalation: Engineering lead

INCIDENT SEVERITY
  P0: Protocol paused, funds at risk → 15min response
  P1: Core function broken → 1h response
  P2: Degraded performance → 4h response
  P3: Non-critical issue → Next business day

EMERGENCY CONTACTS
  Audit firm hotlines (24/7)
  Key signers (≥3 available within 30min for P0)
  Bridge operators
  CEX contacts (for suspicious large withdrawals)
```

---

**ก่อนหน้า**: [Part 59 - Sui Deep Dive ←](part-59-sui-deep-dive.md)
**ต่อไป**: [Part 61 - Advanced Governance Systems →](part-61-advanced-governance.md)
