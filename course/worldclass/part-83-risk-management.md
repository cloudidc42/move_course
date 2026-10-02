# Part 83: DeFi Risk Management

## สารบัญ
- [Risk Framework Overview](#risk-framework-overview)
- [Smart Contract Risk](#smart-contract-risk)
- [Liquidity Risk](#liquidity-risk)
- [Oracle Risk](#oracle-risk)
- [Counterparty & Systemic Risk](#counterparty--systemic-risk)
- [Risk-Adjusted Position Sizing](#risk-adjusted-position-sizing)

---

## Risk Framework Overview

```
DEFI RISK CATEGORIES

1. SMART CONTRACT RISK
   - Bugs in protocol code
   - Upgrade vulnerabilities
   - Dependency risks (libraries, oracles)
   - Mitigation: Audits, formal verification, bug bounties

2. ORACLE RISK
   - Price feed manipulation
   - Stale prices
   - Oracle centralization
   - Mitigation: TWAP, multiple sources, circuit breakers

3. LIQUIDITY RISK
   - Pool drains (impermanent loss)
   - Bank runs (everyone withdraws)
   - DEX liquidity fragmentation
   - Mitigation: Reserves, withdrawal delays, diversification

4. GOVERNANCE RISK
   - Malicious governance proposals
   - Voter apathy (low quorum attacks)
   - Admin key compromise
   - Mitigation: Timelock, veto power, gradual decentralization

5. SYSTEMIC RISK
   - Cascade liquidations
   - Stablecoin de-peg
   - Bridge failures
   - Black swan market events
   - Mitigation: Diversification, circuit breakers, isolation

RISK SCORING MODEL (1-10)

Protocol ABC Risk Assessment:
  Smart Contract:   6/10 (1 audit, no prover, no bug bounty)
  Oracle:           4/10 (TWAP + multiple sources, but single provider)
  Liquidity:        5/10 (deep pools, but concentrated LPs)
  Governance:       7/10 (DAO, timelock, but low participation)
  Systemic:         5/10 (ETH/APT collateral, diversified)
  
  OVERALL RISK SCORE: 27/50 (Medium)
  Max safe TVL: $50M
```

---

## Smart Contract Risk

```move
// ============================================
// RISK MITIGATION PATTERNS IN CODE
// ============================================

module risk::mitigations {
    use aptos_framework::timestamp;
    
    // ============================================
    // PATTERN 1: GRADUATED EXPOSURE
    // Limit TVL until system is proven
    // ============================================
    
    struct TvlCap has key {
        current_tvl: u64,
        max_tvl: u64,
        cap_admin: address,
        cap_raise_delay: u64,   // Min seconds between cap raises
        last_cap_raise: u64,
    }
    
    public fun check_tvl_cap(deposit: u64, cap: &mut TvlCap) {
        let new_tvl = cap.current_tvl + deposit;
        assert!(new_tvl <= cap.max_tvl, 1);  // TVL cap enforced
        cap.current_tvl = new_tvl;
    }
    
    public fun raise_tvl_cap(
        admin: &signer,
        new_cap: u64,
        cap_addr: address,
    ) acquires TvlCap {
        let cap = borrow_global_mut<TvlCap>(cap_addr);
        assert!(std::signer::address_of(admin) == cap.cap_admin, 1);
        assert!(new_cap > cap.max_tvl, 2);  // Must be increase
        assert!(new_cap <= cap.max_tvl * 2, 3);  // Max 2x per raise
        assert!(
            timestamp::now_seconds() >= cap.last_cap_raise + cap.cap_raise_delay,
            4,  // Too soon
        );
        
        cap.max_tvl = new_cap;
        cap.last_cap_raise = timestamp::now_seconds();
    }
    
    // ============================================
    // PATTERN 2: WITHDRAWAL DELAY
    // Prevent bank run / flash loan exploits
    // ============================================
    
    struct WithdrawalRequest has key {
        amount: u64,
        token_type: std::string::String,
        requested_at: u64,
        claimable_at: u64,  // After delay period
    }
    
    const WITHDRAWAL_DELAY_SECS: u64 = 300;  // 5 minutes
    
    public entry fun request_withdrawal(
        user: &signer,
        amount: u64,
        token_type: std::string::String,
    ) {
        let now = timestamp::now_seconds();
        
        // Lock funds immediately
        // coin::withdraw<T>(user, amount);
        
        move_to(user, WithdrawalRequest {
            amount,
            token_type,
            requested_at: now,
            claimable_at: now + WITHDRAWAL_DELAY_SECS,
        });
    }
    
    public entry fun complete_withdrawal(user: &signer) acquires WithdrawalRequest {
        let user_addr = std::signer::address_of(user);
        let request = borrow_global<WithdrawalRequest>(user_addr);
        
        assert!(timestamp::now_seconds() >= request.claimable_at, 1);  // Delay not met
        
        let WithdrawalRequest { amount, token_type: _, requested_at: _, claimable_at: _ } 
            = move_from<WithdrawalRequest>(user_addr);
        
        // Return funds
        // coin::deposit(user_addr, locked_amount);
    }
    
    // ============================================
    // PATTERN 3: INVARIANT CHECKS
    // On-chain assertions that must always hold
    // ============================================
    
    struct PoolInvariant has key {
        k: u128,  // Constant product: k = reserve_x * reserve_y
        reserve_x: u64,
        reserve_y: u64,
    }
    
    public fun assert_invariant(pool: &PoolInvariant) {
        let k_current = (pool.reserve_x as u128) * (pool.reserve_y as u128);
        // k must not decrease (accounting for fees, which increase it)
        assert!(k_current >= pool.k, 1);  // Invariant violated!
    }
    
    // ============================================
    // PATTERN 4: CIRCUIT BREAKERS
    // Automatically pause if anomaly detected
    // ============================================
    
    struct CircuitBreaker has key {
        max_price_change_bps: u64,  // Max % price change in one block
        last_price: u64,
        last_block: u64,
        triggered: bool,
    }
    
    public fun check_circuit_breaker(
        new_price: u64,
        breaker: &mut CircuitBreaker,
    ) {
        let block = aptos_framework::block::get_current_block_height();
        
        if (block == breaker.last_block && breaker.last_price > 0) {
            // Check price change in this block
            let change = if (new_price > breaker.last_price) {
                (new_price - breaker.last_price) * 10_000 / breaker.last_price
            } else {
                (breaker.last_price - new_price) * 10_000 / breaker.last_price
            };
            
            if (change > breaker.max_price_change_bps) {
                breaker.triggered = true;
                aptos_framework::event::emit(CircuitBreakerTriggered {
                    old_price: breaker.last_price,
                    new_price,
                    change_bps: change,
                });
            };
        };
        
        breaker.last_price = new_price;
        breaker.last_block = block;
    }
    
    #[event]
    struct CircuitBreakerTriggered has drop, store {
        old_price: u64,
        new_price: u64,
        change_bps: u64,
    }
}
```

---

## Liquidity Risk

```move
// ============================================
// LIQUIDITY RISK MANAGEMENT
// ============================================

module risk::liquidity {
    use aptos_framework::timestamp;
    
    // ============================================
    // BAD DEBT TRACKER
    // Track positions that couldn't be liquidated profitably
    // ============================================
    
    struct BadDebtTracker has key {
        total_bad_debt: u64,  // USD value (scaled by 1e6)
        bad_debt_events: u64,
        bad_debt_cap: u64,  // If bad_debt > cap, pause protocol
    }
    
    public fun record_bad_debt(
        amount: u64,
        tracker: &mut BadDebtTracker,
    ) {
        tracker.total_bad_debt = tracker.total_bad_debt + amount;
        tracker.bad_debt_events = tracker.bad_debt_events + 1;
        
        // Circuit breaker: too much bad debt
        // Pause protocol to prevent cascade
        if (tracker.total_bad_debt > tracker.bad_debt_cap) {
            // emergency_pause();
        };
        
        aptos_framework::event::emit(BadDebtRecorded { amount, total: tracker.total_bad_debt });
    }
    
    // ============================================
    // RESERVE FACTOR
    // Build protocol reserves from fees for bad debt coverage
    // ============================================
    
    struct ProtocolReserves has key {
        amount: u64,
        coverage_ratio: u64,  // % of bad debt that reserves cover
    }
    
    public fun add_to_reserves(
        fee_amount: u64,
        reserve_factor_bps: u64,  // e.g., 1000 = 10% of fees to reserves
        reserves: &mut ProtocolReserves,
    ): u64 {
        let reserve_portion = fee_amount * reserve_factor_bps / 10_000;
        reserves.amount = reserves.amount + reserve_portion;
        
        // Return amount not going to reserves (for LPs)
        fee_amount - reserve_portion
    }
    
    // Use reserves to cover bad debt
    public fun cover_bad_debt(
        bad_debt: u64,
        reserves: &mut ProtocolReserves,
        tracker: &mut BadDebtTracker,
    ) {
        let coverage = std::u64::min(bad_debt, reserves.amount);
        reserves.amount = reserves.amount - coverage;
        
        let uncovered = bad_debt - coverage;
        if (uncovered > 0) {
            // Socialize remaining bad debt among LPs (rare, last resort)
            record_bad_debt(uncovered, tracker);
        };
    }
    
    // ============================================
    // UTILIZATION MANAGEMENT
    // Prevent 100% utilization (liquidity trap)
    // ============================================
    
    struct UtilizationGuard has key {
        total_liquidity: u64,
        total_borrowed: u64,
        max_utilization_bps: u64,  // e.g., 9500 = 95%
    }
    
    public fun check_utilization(
        new_borrow: u64,
        guard: &UtilizationGuard,
    ) {
        let new_borrowed = guard.total_borrowed + new_borrow;
        let utilization = new_borrowed * 10_000 / guard.total_liquidity;
        assert!(utilization <= guard.max_utilization_bps, 1);  // Would exceed max
    }
    
    // ============================================
    // LIQUIDITY STRESS TEST (off-chain simulation)
    // ============================================
    
    #[event] struct BadDebtRecorded has drop, store { amount: u64, total: u64 }
}
```

---

## Oracle Risk

```move
// ============================================
// ORACLE RISK MITIGATION
// ============================================

module risk::oracle_protection {
    use aptos_framework::timestamp;
    
    // ============================================
    // MULTI-SOURCE ORACLE WITH DEVIATION CHECKS
    // ============================================
    
    struct OracleAggregator has key {
        sources: vector<OracleSource>,
        min_sources: u64,           // Min valid sources required
        max_deviation_bps: u64,     // Max spread across sources
        staleness_threshold: u64,   // Max age in seconds
        last_valid_price: u64,
        last_valid_time: u64,
    }
    
    struct OracleSource has store {
        name: std::string::String,
        price: u64,
        timestamp: u64,
        weight: u64,  // Confidence weight (bps)
    }
    
    public fun get_aggregate_price(oracle: &mut OracleAggregator): u64 {
        let now = timestamp::now_seconds();
        let mut valid_prices: vector<u64> = std::vector::empty();
        let n = std::vector::length(&oracle.sources);
        
        // Filter valid (non-stale) sources
        let mut i = 0u64;
        while (i < n) {
            let source = std::vector::borrow(&oracle.sources, i);
            if (now - source.timestamp <= oracle.staleness_threshold) {
                std::vector::push_back(&mut valid_prices, source.price);
            };
            i = i + 1;
        };
        
        // Require minimum sources
        assert!(std::vector::length(&valid_prices) >= oracle.min_sources, 1);
        
        // Check deviation: no source > max_deviation from median
        let median = get_median(&valid_prices);
        let m = std::vector::length(&valid_prices);
        let mut j = 0u64;
        while (j < m) {
            let price = *std::vector::borrow(&valid_prices, j);
            let deviation = if (price > median) {
                (price - median) * 10_000 / median
            } else {
                (median - price) * 10_000 / median
            };
            assert!(deviation <= oracle.max_deviation_bps, 2);  // Source too far from median
            j = j + 1;
        };
        
        // Return median (most robust to outliers)
        oracle.last_valid_price = median;
        oracle.last_valid_time = now;
        median
    }
    
    // Use last valid price if oracle temporarily unavailable
    public fun get_price_with_fallback(oracle: &OracleAggregator): u64 {
        let now = timestamp::now_seconds();
        assert!(
            now - oracle.last_valid_time <= oracle.staleness_threshold * 2,
            1,  // Price too stale even for fallback
        );
        oracle.last_valid_price
    }
    
    fun get_median(prices: &vector<u64>): u64 {
        // Simple insertion sort for small vectors (typical: 3-7 sources)
        let n = std::vector::length(prices);
        let mut sorted = *prices;
        
        let mut i = 1u64;
        while (i < n) {
            let key = *std::vector::borrow(&sorted, i);
            let mut j = i;
            while (j > 0 && *std::vector::borrow(&sorted, j - 1) > key) {
                let prev = *std::vector::borrow(&sorted, j - 1);
                *std::vector::borrow_mut(&mut sorted, j) = prev;
                j = j - 1;
            };
            *std::vector::borrow_mut(&mut sorted, j) = key;
            i = i + 1;
        };
        
        *std::vector::borrow(&sorted, n / 2)  // Middle element
    }
    
    // ============================================
    // ORACLE MANIPULATION DETECTION
    // ============================================
    
    struct OracleGuard has key {
        twap_price: u64,
        spot_price: u64,
        max_spot_twap_deviation_bps: u64,
        last_update: u64,
    }
    
    public fun validate_price_for_liquidation(
        spot_price: u64,
        guard: &OracleGuard,
    ) {
        // For liquidations: use TWAP (prevents flash loan manipulation)
        // If spot far from TWAP: possible manipulation, reject liquidation
        
        let deviation = if (spot_price > guard.twap_price) {
            (spot_price - guard.twap_price) * 10_000 / guard.twap_price
        } else {
            (guard.twap_price - spot_price) * 10_000 / guard.twap_price
        };
        
        assert!(
            deviation <= guard.max_spot_twap_deviation_bps,
            1,  // Spot too far from TWAP: possible manipulation
        );
    }
}
```

---

## Counterparty & Systemic Risk

```typescript
// ============================================
// RISK MONITORING DASHBOARD
// ============================================

interface ProtocolRisk {
  tvl: bigint;
  utilizationRate: number;
  collateralComposition: Map<string, number>;  // token → % of collateral
  topBorrowers: Array<{ address: string; debtUSD: bigint; hf: number }>;
  oracleStatus: Map<string, { lastUpdate: number; price: bigint; deviation: number }>;
}

class RiskMonitor {
  
  // ============================================
  // CONCENTRATION RISK
  // Check if any single collateral > 30% of protocol
  // ============================================
  
  assessConcentrationRisk(risk: ProtocolRisk): string[] {
    const warnings: string[] = [];
    
    for (const [token, percentage] of risk.collateralComposition) {
      if (percentage > 30) {
        warnings.push(
          `HIGH CONCENTRATION: ${token} is ${percentage.toFixed(1)}% of collateral. ` +
          `If ${token} price drops 50%, protocol faces ~$${(Number(risk.tvl) * percentage / 100 * 0.5 / 1e6).toFixed(1)}M bad debt risk.`
        );
      }
    }
    
    return warnings;
  }
  
  // ============================================
  // LIQUIDATION CASCADE SIMULATION
  // What happens if ETH drops 30%?
  // ============================================
  
  simulatePriceDrop(
    borrowers: Array<{ address: string; collateral: bigint; debt: bigint; ltv: number }>,
    tokenPriceDrop: number,  // 0-1 (e.g., 0.30 for 30% drop)
  ): {
    liquidatedCount: number;
    totalBadDebt: bigint;
    liquidatableValue: bigint;
    cascadeRisk: boolean;
  } {
    let liquidatedCount = 0;
    let totalBadDebt = 0n;
    let liquidatableValue = 0n;
    
    for (const borrower of borrowers) {
      const newCollateralValue = borrower.collateral * BigInt(Math.floor((1 - tokenPriceDrop) * 1000)) / 1000n;
      const weightedCollateral = newCollateralValue * BigInt(Math.floor(borrower.ltv * 100)) / 100n;
      
      const healthFactor = borrower.debt > 0n 
        ? Number(weightedCollateral) / Number(borrower.debt)
        : 999;
      
      if (healthFactor < 1.0) {
        liquidatedCount++;
        liquidatableValue += borrower.debt;
        
        // Bad debt = debt - (collateral * liquidation discount)
        const collateralReceived = newCollateralValue * 90n / 100n;  // 90% (10% discount)
        if (collateralReceived < borrower.debt) {
          totalBadDebt += borrower.debt - collateralReceived;
        }
      }
    }
    
    // Cascade risk: if liquidations > 20% of TVL, market impact could cause more liquidations
    const cascadeRisk = Number(liquidatableValue) > Number(borrowers.reduce((sum, b) => sum + b.collateral, 0n)) * 0.20;
    
    return { liquidatedCount, totalBadDebt, liquidatableValue, cascadeRisk };
  }
  
  // ============================================
  // DAILY RISK REPORT
  // ============================================
  
  generateRiskReport(risk: ProtocolRisk): void {
    console.log('\n====== DAILY RISK REPORT ======');
    console.log(`Date: ${new Date().toISOString()}`);
    console.log(`\nTVL: $${(Number(risk.tvl) / 1e6).toFixed(1)}M`);
    console.log(`Utilization: ${(risk.utilizationRate * 100).toFixed(1)}%`);
    
    console.log('\nCollateral Composition:');
    for (const [token, pct] of risk.collateralComposition) {
      const bar = '█'.repeat(Math.floor(pct / 5));
      console.log(`  ${token.padEnd(10)} ${bar} ${pct.toFixed(1)}%`);
    }
    
    console.log('\nAt-Risk Borrowers (HF < 1.2):');
    const atRisk = risk.topBorrowers.filter(b => b.hf < 1.2);
    for (const b of atRisk.slice(0, 5)) {
      console.log(`  ${b.address.slice(0, 8)}... $${(Number(b.debtUSD) / 1e6).toFixed(1)}M debt, HF=${b.hf.toFixed(2)}`);
    }
    
    console.log('\nOracle Status:');
    for (const [token, status] of risk.oracleStatus) {
      const age = Math.floor((Date.now() - status.lastUpdate) / 1000);
      const statusIcon = age > 300 ? '🔴' : age > 60 ? '🟡' : '🟢';
      console.log(`  ${statusIcon} ${token}: $${(Number(status.price) / 1e6).toFixed(4)} (${age}s ago, ±${status.deviation.toFixed(2)}%)`);
    }
    
    // Run stress test
    const mockBorrowers = risk.topBorrowers.map(b => ({
      address: b.address,
      collateral: b.debtUSD * BigInt(Math.floor(b.hf * 100)) / 100n,
      debt: b.debtUSD,
      ltv: 0.75,
    }));
    
    const stress30 = this.simulatePriceDrop(mockBorrowers, 0.30);
    const stress50 = this.simulatePriceDrop(mockBorrowers, 0.50);
    
    console.log('\nStress Test Results:');
    console.log(`  -30% scenario: ${stress30.liquidatedCount} liquidations, $${(Number(stress30.totalBadDebt) / 1e6).toFixed(2)}M bad debt${stress30.cascadeRisk ? ' ⚠️ CASCADE RISK' : ''}`);
    console.log(`  -50% scenario: ${stress50.liquidatedCount} liquidations, $${(Number(stress50.totalBadDebt) / 1e6).toFixed(2)}M bad debt${stress50.cascadeRisk ? ' ⚠️ CASCADE RISK' : ''}`);
    
    // Concentration warnings
    const warnings = this.assessConcentrationRisk(risk);
    if (warnings.length > 0) {
      console.log('\n⚠️ RISK WARNINGS:');
      for (const w of warnings) console.log(`  • ${w}`);
    }
    
    console.log('\n============================\n');
  }
}
```

---

## Risk-Adjusted Position Sizing

```typescript
// ============================================
// PROTOCOL PARAMETER RISK CALIBRATION
// How to set safe LTV, liquidation thresholds
// ============================================

interface AssetRiskProfile {
  symbol: string;
  volatility30d: number;    // Annualized volatility (e.g., 0.80 for 80%)
  liquidity: bigint;        // On-chain DEX liquidity in USD
  oracle: 'twap' | 'spot';
  chainlink: boolean;
}

class RiskCalibrator {
  
  // Calculate safe LTV based on asset risk
  // LTV = Loan-To-Value: max borrowable / collateral
  calculateSafeLtv(asset: AssetRiskProfile): {
    ltv: number;
    liquidationThreshold: number;
    liquidationBonus: number;
    reasoning: string;
  } {
    // Base LTV starts at 85% for stablecoins, lower for volatile assets
    
    // Volatility adjustment
    // Rule: asset should not lose value faster than liquidation can happen
    // Assume: liquidation takes ~5 minutes (3 blocks on Aptos)
    // Daily vol = annual vol / sqrt(365)
    const dailyVol = asset.volatility30d / Math.sqrt(365);
    const fiveMinVol = dailyVol * Math.sqrt(5 / (60 * 24));
    
    // LTV = 1 - (liquidation bonus + 5-min volatility buffer + spread)
    const liquidationBonus = 0.08;  // 8% for liquidators
    const volatilityBuffer = fiveMinVol * 3;  // 3 sigma event
    const spread = 0.02;  // 2% oracle/spread buffer
    
    const ltv = Math.max(0, 1 - liquidationBonus - volatilityBuffer - spread);
    const ltvBps = Math.floor(ltv * 10000);
    
    // Liquidation threshold slightly above LTV
    const liqThreshold = Math.min(0.90, ltv + 0.05);
    
    // Liquidity-based cap: can't have LTV too high for illiquid assets
    // If DEX liquidity < $10M, cap LTV at 65%
    const ltvCap = Number(asset.liquidity) > 10_000_000 * 1e6 ? 0.85 : 0.65;
    
    const finalLtv = Math.min(ltv, ltvCap);
    
    return {
      ltv: Math.floor(finalLtv * 10000),  // bps
      liquidationThreshold: Math.floor(liqThreshold * 10000),
      liquidationBonus: 800,  // 8% in bps
      reasoning: `30d vol=${(asset.volatility30d * 100).toFixed(0)}%, ` +
                 `5min volatility buffer=${(volatilityBuffer * 100).toFixed(1)}%, ` +
                 `final LTV=${(finalLtv * 100).toFixed(0)}%`,
    };
  }
  
  // Example calibrations
  calibrateProtocol(): void {
    const assets: AssetRiskProfile[] = [
      { symbol: 'APT',  volatility30d: 0.90, liquidity: 100_000_000_000_000n, oracle: 'twap', chainlink: true },
      { symbol: 'BTC',  volatility30d: 0.60, liquidity: 500_000_000_000_000n, oracle: 'twap', chainlink: true },
      { symbol: 'USDC', volatility30d: 0.01, liquidity: 200_000_000_000_000n, oracle: 'spot', chainlink: true },
      { symbol: 'WETH', volatility30d: 0.70, liquidity: 300_000_000_000_000n, oracle: 'twap', chainlink: true },
    ];
    
    console.log('\nRisk-Calibrated Protocol Parameters:');
    console.log('Asset  | LTV%  | LiqThresh% | LiqBonus% | Reasoning');
    console.log('-------|-------|------------|-----------|----------');
    
    for (const asset of assets) {
      const params = this.calculateSafeLtv(asset);
      console.log(
        `${asset.symbol.padEnd(6)} | ${(params.ltv / 100).toFixed(0).padStart(5)} | ` +
        `${(params.liquidationThreshold / 100).toFixed(0).padStart(10)} | ` +
        `${(params.liquidationBonus / 100).toFixed(0).padStart(9)} | ` +
        params.reasoning
      );
    }
  }
}

const calibrator = new RiskCalibrator();
calibrator.calibrateProtocol();
```

---

## สรุป DeFi Risk Management

```
RISK MANAGEMENT HIERARCHY

TIER 1: PROTOCOL DESIGN (prevent risks)
  ✅ TVL caps during launch (graduated exposure)
  ✅ Conservative LTV based on volatility math
  ✅ Circuit breakers on price movements
  ✅ Withdrawal delays prevent bank runs
  ✅ Reserve factor builds safety buffer
  
TIER 2: MONITORING (detect risks)
  ✅ Real-time health factor tracking
  ✅ Oracle freshness monitoring
  ✅ Concentration risk alerts
  ✅ Daily stress test simulations
  ✅ Bad debt tracking
  
TIER 3: RESPONSE (contain risks)
  ✅ Guardian emergency pause
  ✅ Freeze new borrows (not withdrawals)
  ✅ Reserve coverage for bad debt
  ✅ Governance vote for parameter changes
  
KEY FORMULAS
  Safe LTV = 1 - liquidation_bonus - volatility_buffer - spread
  volatility_buffer = 3σ × sqrt(liquidation_time / year_seconds)
  Health Factor = weighted_collateral / total_debt
  Bad Debt Risk = max(0, debt - liquidatable_collateral * (1 - bonus))
  
"Risk management is not about eliminating risk —
it's about ensuring that when bad things happen,
they don't sink the ship."
```

---

**ก่อนหน้า**: [Part 82 - MEV & Order Flow ←](part-82-mev-orderflow.md)
**ต่อไป**: [Part 84 - Real-World Asset Tokenization →](part-84-rwa-tokenization.md)
