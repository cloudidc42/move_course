# Part 95: Production Launch Checklist

## สารบัญ
- [Pre-Launch Security Checklist](#pre-launch-security-checklist)
- [Smart Contract Audit Process](#smart-contract-audit-process)
- [Testnet Validation](#testnet-validation)
- [Infrastructure Readiness](#infrastructure-readiness)
- [Launch Day Runbook](#launch-day-runbook)
- [Post-Launch Monitoring](#post-launch-monitoring)

---

## Pre-Launch Security Checklist

```
MOVE SMART CONTRACT SECURITY CHECKLIST

AUTHENTICATION & ACCESS CONTROL
  □ All admin functions check caller (signer vs stored admin)
  □ No sensitive function callable by arbitrary address
  □ Multi-sig or governance for all privileged operations
  □ Emergency pause mechanism exists and tested
  □ Role separation: admin ≠ fee recipient ≠ guardian
  □ Capability objects not stored in user-accessible locations

ARITHMETIC SAFETY
  □ All u64 multiplications cast to u128 before multiplication
  □ No division before multiplication (precision loss)
  □ Explicit overflow checks on critical paths
  □ isqrt uses integer-safe algorithm (not floating point)
  □ All fee calculations: multiply first, divide last
  □ Price calculations: scaled integers (1e8 or 1e12), not floats
  □ Signed arithmetic (i64): verify sign handling in all branches

ASSET SAFETY
  □ Token transfers use coin::transfer (no manual amount tracking)
  □ No coins created outside authorized minting functions
  □ Escrow patterns: verify escrow balance before releasing
  □ Two-step pattern for sensitive operations (request + execute)
  □ Withdrawal limits enforced (daily/monthly caps)
  □ Circuit breaker on abnormal volume/price moves

REENTRANCY & ORDERING
  □ State updated BEFORE external calls
  □ No callbacks that can re-enter your functions
  □ Oracle reads protected from sandwich attacks (TWAP, not spot)
  □ Commit-reveal for sensitive user inputs (not required in Move's model, but consider in design)

LOGIC CORRECTNESS
  □ AMM k-invariant: verified via Move Prover
  □ Lending LTV caps: formal verification or extensive tests
  □ Liquidation: correctly computes health factor
  □ Reward distribution: sum of user rewards ≤ total rewards
  □ LP share math: proportionality property holds
  □ Fee accounting: sum of all fees = declared fee rate

STORAGE SAFETY
  □ No unbounded vectors (use Table for large sets)
  □ Ring buffers for historical data (fixed size)
  □ Struct fields documented (units, scale factors)
  □ No sensitive data in events (PII, private keys)
  □ Upgrade compatibility: no breaking struct changes

GOVERNANCE & UPGRADE
  □ Upgrade policy set correctly (COMPATIBLE for most)
  □ Timelock duration appropriate (≥48h for mainnet)
  □ Multi-sig quorum for upgrade authority
  □ Migration functions tested for all data states
  □ Rollback plan exists if upgrade fails

DEPENDENCIES
  □ All imported modules from trusted sources
  □ No circular dependencies between modules
  □ External oracle sources documented with fallback
  □ Bridge contracts audited and production-tested

TESTING
  □ >90% line coverage via unit tests
  □ Integration tests for all user-facing flows
  □ Property-based tests for mathematical invariants
  □ Move Prover passed on critical functions
  □ Fuzz testing on input boundaries
  □ Gas profiling: no operations that hit gas limit
```

---

## Smart Contract Audit Process

```
AUDIT PROCESS (12-16 weeks for major protocol)

WEEK 1-2: Internal Review
  □ Developer self-review using checklist above
  □ Peer code review (second developer)
  □ Run automated tools:
      aptos move prove (formal verification)
      aptos move test --coverage
      Custom property tests
  □ Threat modeling: document all attack surfaces
  □ Fix all internal findings before external audit

WEEK 3-6: External Audit (Firm 1)
  □ Select auditors with Move expertise:
      - Spearbit, OtterSec, Halborn, Trail of Bits, Zellic
      - Look for Move-specific experience (not just EVM)
  □ Provide audit package:
      - All source code + tests
      - Architecture documentation
      - Threat model
      - Known issues list
      - Deployment parameters
  □ Regular sync calls with auditors
  □ Respond to questions within 24h

WEEK 7-8: Remediation
  □ Categorize findings: Critical/High/Medium/Low/Info
  □ Fix all Critical and High findings
  □ Fix all Medium findings (or document why not)
  □ Decide on Low/Info: fix or acknowledge
  □ Re-test all fixed paths
  □ Run Move Prover again after fixes

WEEK 9-10: Second Audit (Optional, recommended for >$10M TVL)
  □ Different firm → different perspective
  □ Focus on areas Firm 1 spent least time on
  □ Verify Firm 1's fixes are correct

WEEK 11-12: Bug Bounty (Pre-launch)
  □ Private bug bounty on Immunefi/HackerOne
  □ Rewards: Critical=$100k, High=$50k, Medium=$10k
  □ 2-week pre-launch window for white hat hunters
  □ Provide testnet + all documentation to bounty hunters
  □ Fix any new Critical/High findings

WEEK 13-14: Final Preparation
  □ Auditor publishes final report (with fixes verified)
  □ Prepare public disclosure of report
  □ Launch public bug bounty (goes live with mainnet)
  □ Finalize monitoring and incident response plan

AUDIT FINDINGS TEMPLATE
  ID: AUD-001
  Severity: Critical / High / Medium / Low / Informational
  Title: Brief description
  Location: module::function:line
  Description: What the vulnerability is
  Attack Vector: How to exploit it
  Impact: What attacker gains
  Recommendation: How to fix
  Status: Fixed / Acknowledged / Won't Fix
  Fix Commit: 0xabc123
```

---

## Testnet Validation

```bash
#!/bin/bash
# ============================================
# TESTNET VALIDATION SCRIPT
# Run before every mainnet deployment
# ============================================

TESTNET="testnet"
PROTOCOL_ADDR="0xYOUR_TESTNET_ADDRESS"

echo "========================================"
echo "Running Testnet Validation Suite"
echo "Protocol: $PROTOCOL_ADDR"
echo "Network: $TESTNET"
echo "========================================"

run_test() {
    local name="$1"
    local command="$2"
    echo -n "[$name] "
    if eval "$command" > /dev/null 2>&1; then
        echo "✓ PASSED"
        return 0
    else
        echo "✗ FAILED"
        eval "$command"  # Show output on failure
        return 1
    fi
}

# 1. Deployment test
echo ""
echo "=== DEPLOYMENT ==="
run_test "Compile" "aptos move compile --named-addresses protocol=$PROTOCOL_ADDR"
run_test "Unit Tests" "aptos move test --named-addresses protocol=$PROTOCOL_ADDR"
run_test "Prover" "aptos move prove --named-addresses protocol=$PROTOCOL_ADDR"

# 2. Initialization
echo ""
echo "=== INITIALIZATION ==="
run_test "Initialize Pool" "aptos move run --function-id '$PROTOCOL_ADDR::pool::initialize' --profile $TESTNET --assume-yes"
run_test "Verify Pool Exists" "aptos account get-resource --account $PROTOCOL_ADDR --resource '$PROTOCOL_ADDR::pool::Pool'"

# 3. Happy path operations
echo ""
echo "=== HAPPY PATH ==="
run_test "Add Liquidity" "aptos move run --function-id '$PROTOCOL_ADDR::pool::add_liquidity' --args u64:1000000 u64:4000000 --profile $TESTNET --assume-yes"
run_test "Swap X→Y" "aptos move run --function-id '$PROTOCOL_ADDR::pool::swap' --args u64:10000 bool:true u64:9000 --profile $TESTNET --assume-yes"
run_test "Remove Liquidity" "aptos move run --function-id '$PROTOCOL_ADDR::pool::remove_liquidity' --args u64:100 --profile $TESTNET --assume-yes"

# 4. Security tests
echo ""
echo "=== SECURITY ==="
run_test "Zero amount rejected" "! aptos move run --function-id '$PROTOCOL_ADDR::pool::swap' --args u64:0 bool:true u64:0 --profile $TESTNET 2>&1 | grep -q 'success.*true'"
run_test "Slippage protection" "! aptos move run --function-id '$PROTOCOL_ADDR::pool::swap' --args u64:10000 bool:true u64:999999999 --profile $TESTNET 2>&1 | grep -q 'success.*true'"

# 5. View functions
echo ""
echo "=== VIEW FUNCTIONS ==="
run_test "Get Reserves" "aptos move view --function-id '$PROTOCOL_ADDR::pool::get_reserves'"
run_test "Get Price" "aptos move view --function-id '$PROTOCOL_ADDR::pool::get_price_x_in_y'"
run_test "Get LP Value" "aptos move view --function-id '$PROTOCOL_ADDR::pool::preview_withdraw' --args u64:100"

echo ""
echo "========================================"
echo "Testnet validation complete"
echo "========================================"
```

---

## Infrastructure Readiness

```yaml
# infrastructure-checklist.yml

compute:
  indexer:
    - Service deployed and healthy
    - Events synced to current block
    - No gaps in sequence numbers
    - Alert if lag > 50 blocks
  
  api:
    - Redundant instances (>= 2)
    - Load balancer configured
    - SSL certificates valid
    - Rate limiting enabled
    - CORS configured for frontend
  
  keeper_bot:
    - Running and submitting transactions
    - Alert on consecutive failures
    - Sufficient gas balance (> 100 APT)
    - Restart on crash (systemd / k8s)

database:
  postgresql:
    - Read replica for analytics queries
    - Automated daily backups
    - Point-in-time recovery tested
    - Connection pool (pgBouncer) configured
  
  redis:
    - Price cache layer
    - Session storage
    - Rate limiting counters

monitoring:
  metrics:
    - Protocol TVL (vs. target)
    - 24h volume (vs. baseline)
    - Number of active positions
    - Pool imbalance ratio
    - Oracle freshness (< 60 seconds)
    
  alerts: (PagerDuty/Opsgenie)
    - TVL drops > 20% in 1 hour → CRITICAL
    - Unusual withdrawal spike → HIGH
    - API latency > 2s (p99) → HIGH
    - Indexer lag > 100 blocks → HIGH
    - Keeper fails 3 consecutive times → MEDIUM
    - Oracle stale > 60 seconds → CRITICAL

security:
  - Firewall: restrict admin endpoints to known IPs
  - Secrets: all private keys in Vault (not env vars)
  - TLS: 1.3 minimum, strong cipher suites
  - Key rotation: admin keys rotated monthly
  - Bug bounty: Immunefi bounty active

frontend:
  - Domain: DNS configured, CDN active
  - SSL: Auto-renew enabled
  - Content Security Policy: strict
  - Rate limiting: per-IP limits on all endpoints
```

---

## Launch Day Runbook

```
LAUNCH DAY TIMELINE (all times UTC)

T-24h: Final Checks
  □ Final build compiled from tagged release commit
  □ All audit findings verified fixed
  □ Deployment script tested on devnet one last time
  □ All team on standby communication
  □ Incident response channels created (Discord, Telegram)
  □ War room video call link shared
  □ Monitoring dashboards live and healthy

T-2h: Soft Pre-Launch
  □ Deploy to mainnet (limited functions enabled)
  □ Initialize protocol state
  □ Verify deployment parameters on-chain (fee, oracle, admins)
  □ Run smoke tests on mainnet (tiny amounts)
  □ Confirm monitoring is receiving mainnet data

T-0: Public Launch
  □ Remove function pauses (if launch was paused)
  □ Announce on Twitter/Discord
  □ Post audit report link publicly
  □ Enable public bug bounty
  □ Team monitoring chat active

T+1h: First Checks
  □ First transactions successful?
  □ No anomalous events in monitoring?
  □ TVL tracking correctly?
  □ Frontend working for all flows?
  □ API response times normal?

T+24h: End of Day 1 Review
  □ Total TVL, volume, users
  □ Any security incidents?
  □ Any user-reported bugs?
  □ Gas costs as expected?
  □ All monitoring green?

T+7d: Week 1 Retrospective
  □ Performance vs. targets
  □ Any unexpected behaviors
  □ User feedback analysis
  □ Plan for first protocol update (if needed)

EMERGENCY CONTACTS
  Lead Dev: [name] [phone]
  Auditor: [firm] [security@firm.com]
  Infrastructure: [name] [phone]
  Multi-sig Signers: [list of names and backup contacts]
  Legal/Comms: [name] [contact]

INCIDENT SEVERITY LEVELS
  P0 (CRITICAL): Funds at risk, smart contract exploit suspected
      → All hands, immediate pause, emergency multisig
  P1 (HIGH): Major functionality broken, large user impact
      → Engineering lead response within 15 minutes
  P2 (MEDIUM): Feature degraded, some users affected
      → Fix within 4 hours
  P3 (LOW): Minor issue, cosmetic, documentation
      → Fix within 48 hours
```

---

## Post-Launch Monitoring

```typescript
// ============================================
// LAUNCH HEALTH MONITOR
// Real-time protocol health scoring
// ============================================

interface HealthMetric {
  name: string;
  value: number;
  threshold: number;
  status: 'green' | 'yellow' | 'red';
  weight: number;
}

class ProtocolHealthMonitor {
  private metrics: HealthMetric[] = [];
  
  async computeHealthScore(): Promise<number> {
    const checks = await Promise.all([
      this.checkTVL(),
      this.checkVolume(),
      this.checkOracleHealth(),
      this.checkLiquidityBalance(),
      this.checkSystemLatency(),
      this.checkErrorRate(),
    ]);
    
    this.metrics = checks;
    
    const totalWeight = checks.reduce((sum, m) => sum + m.weight, 0);
    const weightedScore = checks.reduce((sum, m) => {
      const score = m.status === 'green' ? 1 : m.status === 'yellow' ? 0.5 : 0;
      return sum + score * m.weight;
    }, 0);
    
    return (weightedScore / totalWeight) * 100;
  }
  
  private async checkTVL(): Promise<HealthMetric> {
    const tvl = await this.getTVL();
    const prevTVL = await this.getTVL24hAgo();
    const change = (tvl - prevTVL) / prevTVL;
    
    return {
      name: 'TVL Stability',
      value: change,
      threshold: -0.20,  // -20% is red
      status: change > -0.10 ? 'green' : change > -0.20 ? 'yellow' : 'red',
      weight: 3,
    };
  }
  
  private async checkOracleHealth(): Promise<HealthMetric> {
    const lastUpdate = await this.getOracleLastUpdate();
    const ageSeconds = (Date.now() - lastUpdate) / 1000;
    
    return {
      name: 'Oracle Freshness',
      value: ageSeconds,
      threshold: 120,  // 120 seconds is red
      status: ageSeconds < 60 ? 'green' : ageSeconds < 120 ? 'yellow' : 'red',
      weight: 5,
    };
  }
  
  private async checkLiquidityBalance(): Promise<HealthMetric> {
    const { reserveX, reserveY } = await this.getPoolReserves();
    const ratio = Math.min(reserveX, reserveY) / Math.max(reserveX, reserveY);
    
    return {
      name: 'Pool Balance',
      value: ratio,
      threshold: 0.5,
      status: ratio > 0.9 ? 'green' : ratio > 0.5 ? 'yellow' : 'red',
      weight: 2,
    };
  }
  
  private async checkSystemLatency(): Promise<HealthMetric> {
    const start = Date.now();
    await this.pingNode();
    const latencyMs = Date.now() - start;
    
    return {
      name: 'Node Latency',
      value: latencyMs,
      threshold: 2000,
      status: latencyMs < 500 ? 'green' : latencyMs < 2000 ? 'yellow' : 'red',
      weight: 1,
    };
  }
  
  private async checkVolume(): Promise<HealthMetric> {
    return { name: 'Volume', value: 1, threshold: 0, status: 'green', weight: 1 };
  }
  
  private async checkErrorRate(): Promise<HealthMetric> {
    return { name: 'Error Rate', value: 0, threshold: 0.05, status: 'green', weight: 2 };
  }
  
  printHealthReport(): void {
    console.log('\n=== PROTOCOL HEALTH REPORT ===');
    this.metrics.forEach(m => {
      const icon = m.status === 'green' ? '✓' : m.status === 'yellow' ? '⚠' : '✗';
      console.log(`${icon} ${m.name}: ${m.value.toFixed(2)} (threshold: ${m.threshold})`);
    });
  }
  
  private async getTVL(): Promise<number> { return 0; }
  private async getTVL24hAgo(): Promise<number> { return 0; }
  private async getOracleLastUpdate(): Promise<number> { return Date.now(); }
  private async getPoolReserves(): Promise<{reserveX: number, reserveY: number}> {
    return { reserveX: 1, reserveY: 1 };
  }
  private async pingNode(): Promise<void> {}
}
```

---

## สรุป Production Launch

```
LAUNCH READINESS SCORECARD

Security (40%)
  □ External audit complete (Critical/High all fixed)
  □ Move Prover passes on core functions
  □ Bug bounty program live
  □ Emergency pause mechanism tested
  □ Multi-sig operational (all signers tested)
  Score: ___ / 40

Testing (25%)
  □ >90% unit test coverage
  □ All integration flows tested on testnet
  □ Gas profiles within bounds
  □ Property tests for all invariants
  □ Edge cases: zero, max, first user all tested
  Score: ___ / 25

Infrastructure (20%)
  □ Indexer live and synced
  □ Monitoring with alerts configured
  □ Incident response documented
  □ On-call rotation scheduled
  □ Backup/recovery tested
  Score: ___ / 20

Governance (15%)
  □ Admin keys in multi-sig
  □ Upgrade policy configured
  □ Timelock set to appropriate duration
  □ Team documented admin procedures
  □ Community communication plan ready
  Score: ___ / 15

TOTAL: ___ / 100
  90-100: Ready to launch
  75-89:  Fix remaining gaps first
  <75:    Significant work needed, do not launch

MINIMUM REQUIREMENTS (ALL required)
  ✅ Zero unresolved Critical/High audit findings
  ✅ Emergency pause tested on mainnet
  ✅ Multi-sig operational
  ✅ Monitoring with PagerDuty integration
  ✅ Runbook written and team trained
```

---

**ก่อนหน้า**: [Part 94 - Multi-Chain Strategy ←](part-94-multichain.md)
**ต่อไป**: [Part 96 - Economic Modeling & Simulation →](part-96-economic-modeling.md)
