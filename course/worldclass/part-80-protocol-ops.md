# Part 80: Protocol Operations & Incident Response

## สารบัญ
- [Production Operations Overview](#production-operations-overview)
- [Monitoring & Alerting](#monitoring--alerting)
- [Incident Response Playbooks](#incident-response-playbooks)
- [Upgrade Management](#upgrade-management)
- [Post-Incident Analysis](#post-incident-analysis)
- [TypeScript Operations Toolkit](#typescript-operations-toolkit)

---

## Production Operations Overview

```
PROTOCOL OPERATIONS LAYERS

1. PREVENTIVE (Before incidents)
   - Formal verification (Move Prover)
   - Security audits (external firms)
   - Bug bounty programs
   - Staging environment testing
   - Gradual rollout (TVL caps)

2. DETECTIVE (During incidents)
   - On-chain monitoring (event indexer)
   - Price oracle monitoring
   - Unusual activity alerts
   - Mempool watching (large pending txs)

3. RESPONSIVE (After detection)
   - Emergency pause (guardian role)
   - Communication (Discord, Twitter)
   - Root cause analysis
   - Fix development and audit
   - Patch deployment

4. RECOVERY
   - Unpause after fix
   - Compensate affected users
   - Post-mortem publication
   - Process improvement

ROLES & RESPONSIBILITIES
  Guardian: Technical team member; can pause instantly
  Council: 3-of-5 multisig; can unpause after deliberation
  Timelock Admin: DAO; controls parameter changes (48h delay)
  Emergency Council: 4-of-7; can bypass timelock for critical fixes
  
  On-Call Rotation:
  - 24/7 coverage across time zones
  - 15-minute acknowledgment SLA for Critical alerts
  - 1-hour response SLA for High alerts
```

---

## Monitoring & Alerting

```typescript
// ============================================
// PROTOCOL MONITORING SYSTEM
// ============================================

import { AptosClient, Types } from 'aptos';
import { EventEmitter } from 'events';

interface AlertConfig {
  severity: 'critical' | 'high' | 'medium' | 'low';
  channels: ('pagerduty' | 'slack' | 'discord' | 'telegram')[];
  threshold: number;
  cooldownMs: number;  // Don't spam: wait before re-alerting
}

class ProtocolMonitor extends EventEmitter {
  private client: AptosClient;
  private poolAddr: string;
  private lastAlerts: Map<string, number> = new Map();
  
  constructor(nodeUrl: string, poolAddr: string) {
    super();
    this.client = new AptosClient(nodeUrl);
    this.poolAddr = poolAddr;
  }
  
  // ============================================
  // HEALTH CHECKS
  // ============================================
  
  async checkPoolHealth(): Promise<void> {
    try {
      const pool = await this.client.getAccountResource(
        this.poolAddr,
        `${this.poolAddr}::pool::Pool`,
      );
      
      const data = pool.data as any;
      const reserveX = BigInt(data.reserve_x);
      const reserveY = BigInt(data.reserve_y);
      
      // Alert: reserve dropped significantly
      if (reserveX < 1_000_000n) {
        this.alert('LOW_RESERVE_X', 'Pool reserve X critically low', 'critical');
      }
      if (reserveY < 1_000_000n) {
        this.alert('LOW_RESERVE_Y', 'Pool reserve Y critically low', 'critical');
      }
      
      // Alert: reserve imbalance (price impact would be extreme)
      const ratio = Number(reserveX) / Number(reserveY);
      if (ratio > 1000 || ratio < 0.001) {
        this.alert('RESERVE_IMBALANCE', `Extreme reserve ratio: ${ratio}`, 'high');
      }
      
    } catch (error) {
      this.alert('POOL_READ_ERROR', `Failed to read pool: ${error}`, 'critical');
    }
  }
  
  async checkOracleHealth(): Promise<void> {
    try {
      const oracle = await this.client.getAccountResource(
        this.poolAddr,
        `${this.poolAddr}::oracle::TWAPOracle`,
      );
      
      const data = oracle.data as any;
      const lastUpdate = Number(data.last_update_time);
      const now = Math.floor(Date.now() / 1000);
      
      // Alert: stale oracle
      if (now - lastUpdate > 1800) {  // 30 minutes
        this.alert(
          'STALE_ORACLE',
          `Oracle not updated for ${Math.floor((now - lastUpdate) / 60)} minutes`,
          'high',
        );
      }
      
    } catch (error) {
      this.alert('ORACLE_READ_ERROR', `Failed to read oracle: ${error}`, 'high');
    }
  }
  
  // ============================================
  // EVENT MONITORING
  // ============================================
  
  async monitorSwapEvents(fromSequence: bigint): Promise<bigint> {
    const events = await this.client.getEventsByEventHandle(
      this.poolAddr,
      `${this.poolAddr}::pool::Pool`,
      'swap_events',
      { start: fromSequence, limit: 100 },
    );
    
    for (const event of events) {
      const data = event.data as any;
      const amountIn = BigInt(data.amount_in);
      const amountOut = BigInt(data.amount_out);
      
      // Alert: huge single swap (potential manipulation)
      const poolValue = await this.getPoolTVL();
      if (amountIn > poolValue / 10n) {  // > 10% of pool
        this.alert(
          'LARGE_SWAP',
          `Swap of ${amountIn.toString()} (>${10}% of pool TVL)`,
          'high',
        );
      }
      
      // Alert: price impact too high (oracle manipulation attempt)
      const priceImpact = Number(amountIn - amountOut) / Number(amountIn);
      if (priceImpact > 0.30) {  // >30% price impact
        this.alert(
          'HIGH_PRICE_IMPACT',
          `Suspicious swap with ${(priceImpact * 100).toFixed(1)}% price impact`,
          'high',
        );
      }
    }
    
    const lastSeq = events.length > 0
      ? BigInt(events[events.length - 1].sequence_number)
      : fromSequence;
    
    return lastSeq + 1n;
  }
  
  async monitorLiquidationEvents(fromSequence: bigint): Promise<bigint> {
    const events = await this.client.getEventsByEventHandle(
      this.poolAddr,
      `${this.poolAddr}::lending::LendingPool`,
      'liquidation_events',
      { start: fromSequence, limit: 100 },
    );
    
    // Count liquidations in last 5 minutes
    const fiveMinAgo = Date.now() - 300_000;
    const recentLiquidations = events.filter(
      e => (e as any).data.timestamp > fiveMinAgo / 1000,
    );
    
    if (recentLiquidations.length > 50) {
      this.alert(
        'LIQUIDATION_CASCADE',
        `${recentLiquidations.length} liquidations in 5 minutes - possible cascade!`,
        'critical',
      );
    }
    
    const lastSeq = events.length > 0
      ? BigInt(events[events.length - 1].sequence_number)
      : fromSequence;
    
    return lastSeq + 1n;
  }
  
  // ============================================
  // ALERT DISPATCHING
  // ============================================
  
  private alert(id: string, message: string, severity: string): void {
    const now = Date.now();
    const lastAlert = this.lastAlerts.get(id) ?? 0;
    const cooldown = severity === 'critical' ? 300_000 : 900_000;  // 5 or 15 min
    
    if (now - lastAlert < cooldown) return;  // Don't spam
    
    this.lastAlerts.set(id, now);
    this.emit('alert', { id, message, severity, timestamp: new Date() });
  }
  
  private async getPoolTVL(): Promise<bigint> {
    // Simplified: get total reserves in USD
    return 10_000_000_000_000n;  // $10M placeholder
  }
  
  // ============================================
  // MAIN MONITORING LOOP
  // ============================================
  
  async startMonitoring(): Promise<void> {
    let swapSeq = 0n;
    let liquidSeq = 0n;
    
    console.log('Starting protocol monitor...');
    
    while (true) {
      try {
        await this.checkPoolHealth();
        await this.checkOracleHealth();
        swapSeq = await this.monitorSwapEvents(swapSeq);
        liquidSeq = await this.monitorLiquidationEvents(liquidSeq);
      } catch (error) {
        console.error('Monitoring error:', error);
      }
      
      await new Promise(resolve => setTimeout(resolve, 15_000));  // 15 seconds
    }
  }
}

// ============================================
// ALERT DISPATCHER
// ============================================

class AlertDispatcher {
  async sendPagerDuty(severity: string, message: string): Promise<void> {
    const priority = severity === 'critical' ? 'P1' : severity === 'high' ? 'P2' : 'P3';
    
    await fetch('https://events.pagerduty.com/v2/enqueue', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        routing_key: process.env.PAGERDUTY_KEY,
        event_action: 'trigger',
        payload: {
          summary: `[${priority}] ${message}`,
          severity: severity === 'critical' ? 'critical' : 'error',
          source: 'move-protocol-monitor',
        },
      }),
    });
  }
  
  async sendSlack(channel: string, message: string, severity: string): Promise<void> {
    const emoji = { critical: ':red_circle:', high: ':orange_circle:', medium: ':yellow_circle:' }[severity] ?? ':white_circle:';
    
    await fetch(process.env.SLACK_WEBHOOK_URL!, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        channel,
        text: `${emoji} *Protocol Alert*\n${message}`,
      }),
    });
  }
}

// Usage
async function startMonitoring() {
  const monitor = new ProtocolMonitor(
    'https://fullnode.mainnet.aptoslabs.com/v1',
    '0xPROTOCOL_ADDRESS',
  );
  
  const dispatcher = new AlertDispatcher();
  
  monitor.on('alert', async (alert) => {
    console.error(`[${alert.severity.toUpperCase()}] ${alert.message}`);
    
    if (alert.severity === 'critical') {
      await dispatcher.sendPagerDuty(alert.severity, alert.message);
    }
    await dispatcher.sendSlack('#protocol-alerts', alert.message, alert.severity);
  });
  
  await monitor.startMonitoring();
}
```

---

## Incident Response Playbooks

```
PLAYBOOK 1: SUSPICIOUS LARGE WITHDRAWAL

Trigger: Single withdrawal > $5M from protocol

Step 1 - DETECT (0-5 minutes)
  Monitor alert: LARGE_WITHDRAWAL fires
  On-call engineer receives PagerDuty
  
Step 2 - ASSESS (5-15 minutes)
  Check: Is user whitelisted/known (whale)?
  Check: Is this a normal withdrawal or exploit?
  Check: Are funds still in protocol or already moved?
  Check: Is price oracle affected?
  Command: 
    aptos move run --function-id $ADDR::monitor::get_pool_state
    
  Decision matrix:
    If known user AND normal behavior → Log, continue monitoring
    If unknown user AND followed by unusual swaps → Emergency pause
    If price oracle affected → Emergency pause immediately

Step 3A - EMERGENCY PAUSE (if exploit detected, 15-30 minutes)
  Call guardian::emergency_pause() with level=3
  Announce in Discord: "Protocol paused for maintenance"
  Do NOT announce exploit details yet (don't help attacker)
  Command:
    aptos move run \
      --function-id $ADDR::emergency::emergency_pause \
      --args u8:3 \
      --private-key $GUARDIAN_KEY

Step 3B - COMMUNICATIONS (parallel with 3A)
  Post in #announcements: "Protocol temporarily paused. Funds are safe. Team investigating."
  DM security@firm.com for external audit support
  Brief legal team

Step 4 - ANALYSIS (30 minutes - 4 hours)
  Reproduce attack on testnet fork
  Identify root cause
  Draft fix (minimum change principle)
  Internal review of fix

Step 5 - FIX DEPLOYMENT (4-24 hours)
  External audit of fix (expedited, 24h)
  Deploy fix to testnet, verify
  Multi-sig approval for mainnet deploy
  Deploy fix
  Verify fix blocks attack vector

Step 6 - UNPAUSE (after fix verified)
  Council votes to unpause (3-of-5)
  Command:
    aptos move run \
      --function-id $ADDR::emergency::vote_unpause \
      --private-key $COUNCIL_KEY_1
  (Repeat for 3 council members)

Step 7 - POST-MORTEM (within 48 hours)
  Public post-mortem document
  Timeline of events
  Root cause
  Fix description
  User compensation plan (if funds lost)

---

PLAYBOOK 2: ORACLE MANIPULATION DETECTED

Trigger: Price deviation > 20% from TWAP

Step 1 - DETECT
  Oracle monitor fires PRICE_DEVIATION alert

Step 2 - ASSESS (immediate, 1-5 minutes)
  Check: Are there large swaps in same block?
  Check: Is deviation on single DEX or multiple?
  Check: Is any address borrowing against manipulated price?
  
  If borrowing is happening → PAUSE IMMEDIATELY (flash loan attack)

Step 3 - CONTAIN
  Pause new borrows (not withdrawals) via borrowing_paused flag
  Let existing positions close normally
  Oracle will self-correct after TWAP window passes (30 min)

Step 4 - VERIFY RESOLUTION
  Wait for TWAP to normalize
  Monitor: no unusual liquidations
  Unpause borrowing

---

PLAYBOOK 3: SMART CONTRACT BUG REPORT

Source: Bug bounty submission or community report

Step 1 - TRIAGE (within 4 hours)
  Security team reviews report
  Assess: Real or false positive?
  Assess: Severity (Critical/High/Medium/Low)

Step 2 - REPRODUCE (within 8 hours)
  Create PoC on testnet
  Confirm vulnerability exists
  Estimate impact (how much at risk)

Step 3 - DECIDE ACTION
  Critical (>$1M at risk): Emergency pause immediately
  High ($100K-$1M): Patch within 24h, consider pause
  Medium (<$100K): Patch within 7 days, enhanced monitoring
  Low: Patch in next scheduled upgrade

Step 4 - FIX & REWARD
  Develop fix
  Expedited audit
  Deploy
  Pay bounty as specified in program
  Coordinate responsible disclosure (90 days)
```

---

## Upgrade Management

```move
// ============================================
// UPGRADE PATTERN: Versioned Migrations
// ============================================

module protocol::upgrade_manager {
    use aptos_framework::timestamp;
    
    const TIMELOCK_SECS: u64 = 172_800;  // 48 hours
    
    struct UpgradeProposal has key, store {
        id: u64,
        description: std::string::String,
        new_code_hash: vector<u8>,  // Hash of upgrade package
        proposed_at: u64,
        execute_after: u64,
        approved_by: vector<address>,
        canceled: bool,
        executed: bool,
    }
    
    struct UpgradeManager has key {
        timelock_admin: address,
        emergency_admin: address,
        next_proposal_id: u64,
        min_approvals: u64,
        approvers: vector<address>,
        proposals: aptos_std::table::Table<u64, UpgradeProposal>,
    }
    
    // Propose an upgrade (starts timelock)
    public fun propose_upgrade(
        proposer: &signer,
        description: std::string::String,
        new_code_hash: vector<u8>,
        manager_addr: address,
    ): u64 acquires UpgradeManager {
        let manager = borrow_global_mut<UpgradeManager>(manager_addr);
        assert!(std::signer::address_of(proposer) == manager.timelock_admin, 1);
        
        let id = manager.next_proposal_id;
        manager.next_proposal_id = id + 1;
        
        let proposal = UpgradeProposal {
            id,
            description,
            new_code_hash,
            proposed_at: timestamp::now_seconds(),
            execute_after: timestamp::now_seconds() + TIMELOCK_SECS,
            approved_by: std::vector::empty(),
            canceled: false,
            executed: false,
        };
        
        aptos_std::table::add(&mut manager.proposals, id, proposal);
        
        aptos_framework::event::emit(UpgradeProposed {
            id,
            execute_after: proposal.execute_after,
        });
        
        id
    }
    
    // Multi-sig approval of upgrade
    public fun approve_upgrade(
        approver: &signer,
        proposal_id: u64,
        manager_addr: address,
    ) acquires UpgradeManager {
        let manager = borrow_global_mut<UpgradeManager>(manager_addr);
        let caller = std::signer::address_of(approver);
        
        // Must be an authorized approver
        assert!(std::vector::contains(&manager.approvers, &caller), 2);
        
        let proposal = aptos_std::table::borrow_mut(&mut manager.proposals, proposal_id);
        assert!(!proposal.canceled && !proposal.executed, 3);
        
        // Don't double-count
        if (!std::vector::contains(&proposal.approved_by, &caller)) {
            std::vector::push_back(&mut proposal.approved_by, caller);
        };
    }
    
    // Execute upgrade after timelock + sufficient approvals
    public fun execute_upgrade(
        executor: &signer,
        proposal_id: u64,
        manager_addr: address,
    ) acquires UpgradeManager {
        let manager = borrow_global_mut<UpgradeManager>(manager_addr);
        let proposal = aptos_std::table::borrow_mut(&mut manager.proposals, proposal_id);
        
        assert!(!proposal.canceled && !proposal.executed, 1);
        assert!(timestamp::now_seconds() >= proposal.execute_after, 2);  // Timelock expired
        assert!(
            std::vector::length(&proposal.approved_by) >= manager.min_approvals,
            3,  // Not enough approvals
        );
        
        proposal.executed = true;
        
        aptos_framework::event::emit(UpgradeExecuted { id: proposal_id });
        
        // Actual upgrade happens via Aptos upgrade mechanism
        // aptos_framework::code::publish_package_txn(...)
    }
    
    // Cancel upgrade (admin)
    public fun cancel_upgrade(
        admin: &signer,
        proposal_id: u64,
        manager_addr: address,
    ) acquires UpgradeManager {
        let manager = borrow_global_mut<UpgradeManager>(manager_addr);
        assert!(std::signer::address_of(admin) == manager.timelock_admin, 1);
        
        let proposal = aptos_std::table::borrow_mut(&mut manager.proposals, proposal_id);
        assert!(!proposal.executed, 2);
        
        proposal.canceled = true;
        
        aptos_framework::event::emit(UpgradeCanceled { id: proposal_id });
    }
    
    #[event] struct UpgradeProposed has drop, store { id: u64, execute_after: u64 }
    #[event] struct UpgradeExecuted has drop, store { id: u64 }
    #[event] struct UpgradeCanceled has drop, store { id: u64 }
}
```

---

## Post-Incident Analysis

```markdown
# Post-Incident Report Template

## Incident Summary
**Date**: 2024-xx-xx
**Duration**: X hours Y minutes
**Severity**: Critical / High / Medium
**Impact**: ~$X funds at risk, X users affected
**Status**: Resolved / Ongoing

## Timeline
| Time (UTC) | Event |
|------------|-------|
| 14:23 | Monitoring alert fires: LARGE_WITHDRAWAL |
| 14:24 | On-call engineer paged via PagerDuty |
| 14:27 | Engineer acknowledges, begins investigation |
| 14:31 | Exploit confirmed on testnet fork |
| 14:35 | Emergency pause executed (guardian key) |
| 14:38 | Announcement posted: "Protocol paused" |
| ... | ... |
| 18:45 | Fix deployed to testnet |
| 20:00 | External audit of fix complete |
| 20:15 | Fix deployed to mainnet |
| 20:20 | Protocol unpaused |

## Root Cause
The `calculate_reward()` function in `staking.move:142` multiplied three u64 
values without overflow protection. When `staked > 4.29B tokens` and 
`duration > 4.29B seconds`, the product overflowed u64 and returned a small 
number. Attacker staked maximum u64 tokens, waited for overflow, received 
disproportionately large rewards.

## Why Was This Not Caught?
- Unit tests covered normal values (up to 1M tokens)
- Move Prover was not run on this module
- Audit did not flag this specific code path
- No property-based testing with extreme values

## Impact Assessment
- Funds at risk: $2.3M in staking rewards
- Funds actually drained: $0 (caught before exploitation)
- Users affected: 0 (paused before any harm)

## Fix Description
Changed to u128 intermediate arithmetic:
```move
// Before (vulnerable)
staked * duration * rate  // u64 overflow

// After (fixed)  
let result = (staked as u128) * (duration as u128) * (rate as u128);
assert!(result <= u64::max_value() as u128, ERR_OVERFLOW);
result as u64
```

## Preventive Measures
1. Added Move Prover spec to all arithmetic functions
2. Added fuzz testing with extreme u64 values
3. Added CI step: `aptos move prove` must pass
4. Briefed team on u64 overflow patterns
5. Added monitoring for unusual reward amounts

## Lessons Learned
- Always use u128 intermediate for financial calculations
- Move Prover is effective but must be run
- Monitoring saved us: 4 minutes from alert to pause
- Need more extreme value test cases
```

---

## TypeScript Operations Toolkit

```typescript
// ============================================
// OPS TOOLKIT: Common protocol management tasks
// ============================================

import { AptosClient, AptosAccount, BCS, TxnBuilderTypes } from 'aptos';

class ProtocolOps {
  private client: AptosClient;
  private guardian: AptosAccount;
  private protocolAddr: string;
  
  constructor(nodeUrl: string, guardianPrivateKey: string, protocolAddr: string) {
    this.client = new AptosClient(nodeUrl);
    this.guardian = new AptosAccount(Buffer.from(guardianPrivateKey, 'hex'));
    this.protocolAddr = protocolAddr;
  }
  
  // EMERGENCY PAUSE
  async emergencyPause(level: 1 | 2 | 3): Promise<string> {
    console.log(`🚨 EMERGENCY PAUSE initiated at level ${level}`);
    
    const payload: Types.EntryFunctionPayload = {
      function: `${this.protocolAddr}::emergency::emergency_pause`,
      type_arguments: [],
      arguments: [level],
    };
    
    const txHash = await this.submitTx(payload);
    console.log(`✅ Pause tx: ${txHash}`);
    return txHash;
  }
  
  // GET PROTOCOL STATUS
  async getProtocolStatus(): Promise<void> {
    console.log('\n=== Protocol Status ===');
    
    try {
      const state = await this.client.getAccountResource(
        this.protocolAddr,
        `${this.protocolAddr}::emergency::EmergencyState`,
      );
      const data = state.data as any;
      
      console.log(`Pause Level: ${data.pause_level} (0=running, 1-3=paused)`);
      console.log(`Paused At: ${new Date(Number(data.paused_at) * 1000).toISOString()}`);
    } catch {
      console.log('Emergency state: Normal (not paused)');
    }
    
    try {
      const pool = await this.client.getAccountResource(
        this.protocolAddr,
        `${this.protocolAddr}::pool::Pool`,
      );
      const data = pool.data as any;
      
      console.log(`\nPool Status:`);
      console.log(`  Reserve X: ${BigInt(data.reserve_x).toLocaleString()}`);
      console.log(`  Reserve Y: ${BigInt(data.reserve_y).toLocaleString()}`);
      console.log(`  Total LP: ${BigInt(data.total_lp).toLocaleString()}`);
    } catch {
      console.log('Pool: not found');
    }
  }
  
  // SNAPSHOT: Save current state for comparison
  async snapshot(): Promise<object> {
    const state: Record<string, any> = {
      timestamp: new Date().toISOString(),
    };
    
    const resources = await this.client.getAccountResources(this.protocolAddr);
    for (const resource of resources) {
      state[resource.type] = resource.data;
    }
    
    return state;
  }
  
  // DIFF TWO SNAPSHOTS
  diffSnapshots(before: any, after: any): void {
    console.log('\n=== State Diff ===');
    
    for (const key of Object.keys(after)) {
      if (key === 'timestamp') continue;
      
      const b = JSON.stringify(before[key]);
      const a = JSON.stringify(after[key]);
      
      if (b !== a) {
        console.log(`\nChanged: ${key}`);
        console.log(`  Before: ${b}`);
        console.log(`  After:  ${a}`);
      }
    }
  }
  
  private async submitTx(payload: Types.EntryFunctionPayload): Promise<string> {
    const txRequest = await this.client.generateTransaction(
      this.guardian.address().toString(),
      payload,
    );
    const signedTx = await this.client.signTransaction(this.guardian, txRequest);
    const result = await this.client.submitTransaction(signedTx);
    await this.client.waitForTransaction(result.hash);
    return result.hash;
  }
}

// CLI usage
async function main() {
  const ops = new ProtocolOps(
    process.env.APTOS_NODE_URL!,
    process.env.GUARDIAN_PRIVATE_KEY!,
    process.env.PROTOCOL_ADDR!,
  );
  
  const command = process.argv[2];
  
  switch (command) {
    case 'status':
      await ops.getProtocolStatus();
      break;
    case 'pause':
      const level = parseInt(process.argv[3] ?? '2') as 1 | 2 | 3;
      await ops.emergencyPause(level);
      break;
    case 'snapshot':
      const snap = await ops.snapshot();
      console.log(JSON.stringify(snap, null, 2));
      break;
    default:
      console.log('Commands: status | pause <1|2|3> | snapshot');
  }
}

main().catch(console.error);
```

---

## สรุป Protocol Operations

```
OPERATIONS MATURITY MODEL

LEVEL 1 - BASIC
  ✅ Manual monitoring (check explorer daily)
  ✅ Guardian key exists
  ✅ Emergency pause function in code
  
LEVEL 2 - REACTIVE
  ✅ Automated monitoring with alerts
  ✅ PagerDuty on-call rotation
  ✅ Incident response documented
  ✅ Bug bounty program active
  
LEVEL 3 - PROACTIVE  
  ✅ Formal verification (Move Prover)
  ✅ Canary deployments (small TVL first)
  ✅ Chaos engineering (simulate failures)
  ✅ Regular security reviews
  
LEVEL 4 - ELITE
  ✅ Automated remediation (pause on anomaly)
  ✅ Full observability (metrics, traces, logs)
  ✅ Game theory modeling (adversarial simulation)
  ✅ Cross-protocol incident coordination
  
KEY METRICS TO TRACK
  Availability:          >99.9% uptime target
  MTTD (Mean Time to Detect): <5 minutes for critical
  MTTR (Mean Time to Respond): <30 minutes for critical  
  Incidents per quarter: Target <2 unplanned pauses
  Bug bounty submissions: >5/month shows healthy community
  
"The measure of a protocol's security is not
whether it has been attacked, but how quickly
it can detect and respond when it is."
```

---

**ก่อนหน้า**: [Part 79 - Move Prover ←](part-79-move-prover.md)
**ต่อไป**: [Part 81 - Advanced Tokenomics Design →](part-81-tokenomics.md)
