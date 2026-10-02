# Part 98: Real-World DeFi Case Studies

## สารบัญ
- [Case Study 1: Pancake on Aptos](#case-study-1-pancake-on-aptos)
- [Case Study 2: Turbos on Sui](#case-study-2-turbos-on-sui)
- [Case Study 3: Thala Protocol](#case-study-3-thala-protocol)
- [Exploit Post-Mortems](#exploit-post-mortems)
- [Protocol Architecture Lessons](#protocol-architecture-lessons)
- [From Code to Production](#from-code-to-production)

---

## Case Study 1: Pancake on Aptos

```
PANCAKESWAP ON APTOS
Architecture Analysis

BACKGROUND
  PancakeSwap launched on Aptos in Oct 2022
  Already battle-tested on BSC (Ethereum fork)
  Challenge: Rewrite Solidity → Move while maintaining parity

KEY ARCHITECTURAL DECISIONS

1. CONSTANT PRODUCT AMM (CPMM)
   Same formula as Uniswap V2: x * y = k
   
   Why not V3 (concentrated liquidity)?
   - Move doesn't have loops or tick iteration as gas-cheaply
   - V2 is simpler to audit (fewer code paths)
   - Sufficient for initial market validation
   
   Trade-off: Less capital efficiency vs. simpler codebase

2. DUAL POOL TYPES
   Stable pairs (low slippage for stablecoins):
     Uses Curve-style invariant: x³y + y³x = k
     More complex math, lower slippage for pegged assets
   
   Volatile pairs (CPMM):
     Standard x*y = k
     Works for all pairs

3. COIN STANDARD EVOLUTION
   Initially: CoinStore (aptos_std::coin)
   Later: Added FA (Fungible Asset) support
   Both: wrapped under same API via router

4. SWAP ROUTER
   Finds best path through multiple pools
   Multi-hop: APT → USDC → TOKEN in one transaction
   
   Implementation:
   graph BFS from tokenA → tokenB
   Return: (path[], amounts_out[])
   Execute: call swap on each pool in sequence

5. GOVERNANCE
   CAKE token: stake → xCAKE → governance votes
   veCAKE: time-locked CAKE, higher voting power
   Follows Curve's veToken model

LESSONS LEARNED
  ✅ Port contract-by-contract, not all at once
  ✅ Move's resource model prevents common Solidity bugs
  ✅ Gas optimization needed: Move reads are expensive
  ✅ User onboarding challenge: different from EVM wallets
  ❌ Underestimated: fee tier complexity in Move
  ❌ Event indexing: had to rebuild from Solidity assumptions
```

---

## Case Study 2: Turbos on Sui

```
TURBOS FINANCE ON SUI
Concentrated Liquidity AMM (CLMM)

BACKGROUND
  Built natively on Sui (not a port)
  Uniswap V3-style concentrated liquidity
  First production CLMM on Sui

KEY INNOVATIONS

1. SUI OBJECT MODEL FOR POOLS
   
   struct Pool has key {
       id: UID,
       token_x: Balance<X>,
       token_y: Balance<Y>,
       
       // Tick system for concentrated liquidity
       ticks: Table<i32, TickInfo>,
       current_tick: i32,
       current_sqrt_price: u128,
       
       // Bitmap of initialized ticks (u256 per word)
       tick_bitmap: Table<i32, u256>,
       
       // Liquidity
       liquidity: u128,
       protocol_fee: u64,
   }

2. POSITION AS OWNED OBJECT
   
   // Each LP position is a distinct Sui object
   struct Position has key {
       id: UID,
       pool: ID,
       tick_lower: i32,
       tick_upper: i32,
       liquidity: u128,
       fee_growth_checkpoint_x: u128,
       fee_growth_checkpoint_y: u128,
       tokens_owed_x: u64,
       tokens_owed_y: u64,
   }
   
   // Benefit: Position ownership is clear (owned object)
   // Transfer position = transfer ownership of LP

3. TICK MATH IN MOVE
   
   // sqrt(1.0001^tick) with Q64.64 fixed point
   // Pre-computed tick to sqrt_price mapping
   fun get_sqrt_price_at_tick(tick: i32): u128 {
       // Use bit manipulation for efficient computation
       // Based on Uniswap V3's tick math adapted for Move
       let abs_tick = if (tick < 0) (-tick as u32) else (tick as u32);
       
       let ratio = if (abs_tick & 0x1 != 0) {
           0xFFFCB933BD6FAD37AA2D162D1A594001u128
       } else {
           0x100000000000000000000000000000000u128
       };
       // ... continues with bit operations for each bit of abs_tick
       ratio
   }

4. PTB INTEGRATION
   
   // Turbos leverages Sui PTBs for atomic arbitrage
   // Single PTB: flash borrow → swap on external pool → swap back → repay
   // No re-entrancy risk (Sui's object model prevents it)

5. CONCENTRATED LIQUIDITY MATH
   
   // Amount of tokens for a position:
   // if current_tick < tick_lower:
   //   token_x = L * (sqrt(upper) - sqrt(lower)) / (sqrt(upper) * sqrt(lower))
   //   token_y = 0
   // if tick_lower <= current_tick < tick_upper:
   //   token_x = L * (sqrt(upper) - sqrt(current)) / (sqrt(upper) * sqrt(current))
   //   token_y = L * (sqrt(current) - sqrt(lower))
   // if current_tick >= tick_upper:
   //   token_x = 0
   //   token_y = L * (sqrt(upper) - sqrt(lower))

LESSONS
  ✅ Sui object model is perfect for LP positions (clear ownership)
  ✅ PTBs enable complex atomic operations without new contracts
  ✅ Tick bitmap: efficient O(1) tick finding
  ❌ Tick math: very precise fixed-point arithmetic, error-prone
  ❌ Position fee calculation: complex cumulative growth tracking
  ❌ Gas: tick crossing operations can be expensive
```

---

## Case Study 3: Thala Protocol

```
THALA PROTOCOL
Stablecoin + DEX on Aptos

BACKGROUND
  Thala built MOD (Move Dollar): CDP-backed stablecoin
  Similar to MakerDAO's DAI but on Aptos
  Also runs ThalaSwap AMM (composable liquidity)

CDP SYSTEM ARCHITECTURE

  User deposits APT/wBTC/wETH as collateral
  User mints MOD (stablecoin) up to collateral limit
  
  Health Factor = (Collateral Value * LTV) / Debt in MOD
  Liquidation when Health Factor < 1.0

STABILITY MECHANISMS

1. COLLATERALIZATION RATIO
   Each collateral type has max LTV:
   APT: 70% (more volatile)
   wBTC: 80% (less volatile)
   USDC: 90% (stablecoin collateral)

2. STABILITY FEE
   Annual interest on MOD borrowed
   Accrues per second: debt = debt * (1 + rate)^elapsed
   Used to buy back and burn THL token

3. LIQUIDATION ENGINE
   
   // Simplified liquidation logic
   fun liquidate(vault: &mut Vault, liquidator: &signer, debt_to_cover: u64) {
       let collateral_price = oracle.get_price(vault.collateral_type);
       
       let health = vault.collateral_amount * collateral_price * ltv
           / vault.debt_mod;
       
       assert!(health < 1_000_000, E_HEALTHY_VAULT);
       
       // Liquidator repays debt_to_cover MOD
       // Gets collateral_to_seize worth of collateral
       let collateral_to_seize = debt_to_cover * (1 + LIQUIDATION_BONUS) / collateral_price;
       
       vault.debt_mod -= debt_to_cover;
       vault.collateral_amount -= collateral_to_seize;
       // Transfer collateral to liquidator
   }

4. PRICE STABILITY MODULE (PSM)
   
   // Allows 1:1 swaps between MOD and USDC (within limits)
   // Maintains peg when market price deviates
   
   struct PSM has key {
       usdc_balance: u64,
       mod_cap: u64,       // Max USDC → MOD conversions
       swap_fee_bps: u64,  // 0.1% fee
   }
   
   // If MOD > $1.00: Users swap USDC → MOD via PSM (arbitrage brings price down)
   // If MOD < $1.00: Users swap MOD → USDC via PSM (arbitrage brings price up)

5. THEALASWAP INTEGRATION
   
   MOD/USDC pool: deep liquidity via liquidity mining
   LP incentives paid in THL (governance token)
   Creates natural demand for THL (stake to earn fees)
   Circle back: THL used for governance → controls stability fee

LESSONS
  ✅ CDP design requires extremely careful oracle security
  ✅ Stability mechanisms must be coordinated (PSM + CDP + AMM)
  ✅ Liquidation bots need 24/7 operation (keeper incentives matter)
  ❌ Black swan events: APT -80% in 24h = mass liquidations
  ❌ MOD depeg risk: liquidity must be deep enough to hold peg
  ❌ Oracle manipulation: use TWAP, multi-source, outlier filtering
```

---

## Exploit Post-Mortems

```
NOTABLE DeFi EXPLOITS & LESSONS FOR MOVE DEVELOPERS

EXPLOIT 1: FLASH LOAN ORACLE MANIPULATION
Protocol: Mango Markets (Solana, 2022, $114M)

How it worked:
  1. Attacker opens large MNGO position
  2. Flash loans + spot buys to pump MNGO price 10x
  3. Borrows against inflated collateral
  4. MNGO price collapses, leaves bad debt
  5. Attacker keeps borrowed assets

Move Mitigations:
  ✅ TWAP oracle: 30-min average resists single-block manipulation
  ✅ Deviation check: reject price if > X% from TWAP
  ✅ Circuit breaker: pause borrowing if utilization spikes
  ✅ Position concentration limits: max borrow per user

---

EXPLOIT 2: REENTRANCY IN CALLBACK
Protocol: Cream Finance (Ethereum, 2021, $130M)

How it worked:
  1. Flash loan from Protocol A
  2. Callback → attack Protocol B
  3. Use B's inflated position to borrow more from A
  4. Profits from inconsistent state between A's call and B's response

Move's Natural Protection:
  ✅ Move has no callbacks/fallback functions
  ✅ No dynamic dispatch to unknown contracts
  ✅ Transaction is linear: no re-entry possible
  ✅ Resource safety: can't "fake" having a coin

But Watch For:
  ⚠️ Cross-module calls can still have ordering issues
  ⚠️ State read before write (classic reentrancy pattern)
  ✅ Rule: update state BEFORE calling other modules

---

EXPLOIT 3: INTEGER OVERFLOW
Protocol: Multiple protocols (various)

Classic Solidity bug: uint256 + uint256 can overflow
Move's Protection:
  ✅ Arithmetic aborts on overflow (no silent wrapping)
  ✅ u256 is available for large intermediate calculations
  ✅ Explicit overflow with wrapping_add if needed

Move Best Practice:
  always: cast to u128/u256 before multiplying u64 values
  check:  assert!(b <= u64::MAX - a, overflow_error)
  use:    math::mul_div() for large multiplications

---

EXPLOIT 4: ACCESS CONTROL BYPASS
Pattern: admin_only function callable by anyone

Solidity bug:
  function setFee(uint256 fee) public {  // Missing onlyOwner!
      feeRate = fee;
  }

Move equivalent (would be a bug):
  public fun set_fee(amount: u64) acquires Config {  // Missing admin check!
      borrow_global_mut<Config>(@protocol).fee = amount;
  }

Move Prevention:
  ✅ Require signer parameter:
     public fun set_fee(admin: &signer, amount: u64) {
         assert!(signer::address_of(admin) == ADMIN_ADDR, E_NOT_ADMIN);
         ...
     }
  ✅ Use Capability pattern:
     public fun set_fee(_cap: &AdminCap, amount: u64) { ... }
  ✅ Move Prover can verify access control:
     spec set_fee {
         requires signer::address_of(admin) == @protocol;
     }

---

EXPLOIT 5: INCORRECT DECIMAL HANDLING
Protocol: Many DeFi protocols

Common bug: Mix up scaled and unscaled values
  price_usd = 1_500_00  // Is this $1500 or $150000?
  
  calculate_value(amount, price_usd) {
      // If amount is scaled 1e8 and price is 1e2 (cents):
      return amount * price_usd / 100;  // Wrong! Should be / 1e8
  }

Move Prevention:
  ✅ Comment every u64 with its scale:
     price_usd_1e8: u64,  // Price in USD, scaled by 1e8
  ✅ Named constants for scales:
     const PRICE_SCALE: u128 = 100_000_000;  // 1e8
  ✅ Prover assertions on expected ranges:
     ensures result <= EXPECTED_MAX;
```

---

## Protocol Architecture Lessons

```
ARCHITECTURAL LESSONS FROM PRODUCTION PROTOCOLS

1. START SIMPLE, ADD COMPLEXITY INCREMENTALLY
   
   Pancake V1: Basic CPMM
   Pancake V2: + stable pairs, router
   Pancake V3: + concentrated liquidity
   
   Reason: Each version can be audited independently
   Mistake: Launching V3 complexity without V1/V2 validation

2. ORACLE IS THE MOST CRITICAL EXTERNAL DEPENDENCY
   
   All financial protocols depend on price feeds
   One oracle failure = protocol failure
   
   Production checklist for oracles:
     Multiple sources (3+): Pyth, Switchboard, custom
     TWAP instead of spot: 30-minute window
     Deviation filter: reject if > 5% from median
     Staleness check: reject if > 30 seconds old
     Fallback: pause if all oracles fail

3. LIQUIDITY DEPTH DETERMINES STABILITY
   
   Protocol with $1M TVL cannot safely support $1M loan
   Rule: Borrow cap = f(liquidity depth, liquidation incentive)
   
   Safe formula:
     max_single_position = pool_liquidity × 5%
     total_debt_cap = collateral_tvl × 70%

4. LIQUIDATION MUST ALWAYS BE PROFITABLE
   
   If liquidation is unprofitable, bad debt accumulates
   Liquidation bonus must exceed:
     - Gas cost of liquidation transaction
     - Slippage of selling seized collateral
     - Bridge fees (for cross-chain liquidation)
   
   Monitor: liquidation bot profitability daily

5. GOVERNANCE ATTACKS ARE REAL
   
   Small protocols with concentrated token ownership
   are targets for governance attacks
   
   Protection levels:
   Level 1: Multisig admin (3/5)
   Level 2: Timelock (48h)
   Level 3: Community veto (72h veto period)
   Level 4: Full DAO with >10% quorum

6. TEST WITH REAL AMOUNTS FROM DAY ONE
   
   Testing with $1 works fine; $1M reveals problems
   Issues at scale:
     - Integer overflow (u64 caps at ~$18B with 1e9 scaling)
     - Gas: complex operations may hit limits
     - Oracle precision: rounding errors compound
   
   Always test with max expected position sizes

7. MONITORING IS PART OF THE PROTOCOL
   
   Smart contract is deployed; monitoring is ongoing
   Budget 20% of engineering for:
     - Event indexing
     - Alerting infrastructure
     - Keeper bots
     - Dashboard

8. DOCUMENTATION PREVENTS MISTAKES
   
   Document every number with its unit and scale
   Document every assumption in comments
   Document the threat model in the README
   
   Future developers (including you 6 months later)
   will make mistakes if context is missing
```

---

## From Code to Production

```
THE FULL JOURNEY: IDEA TO PRODUCTION

Phase 1: DESIGN (2-4 weeks)
  □ Write one-pager: what problem does this solve?
  □ Define user journeys (happy path + edge cases)
  □ Design token economics (if applicable)
  □ Economic simulation (Python agent model)
  □ Threat model: list all attack vectors
  □ Decide: Aptos or Sui or both?

Phase 2: PROTOTYPE (4-8 weeks)
  □ Core AMM/lending/staking logic in Move
  □ Move Prover specs for critical invariants
  □ Basic TypeScript tests against devnet
  □ Simple frontend to test UX
  □ Internal review: 2nd developer reads all code

Phase 3: AUDIT PREP (2-4 weeks)
  □ Clean up code: remove TODOs, add comments
  □ Write comprehensive test suite (>90% coverage)
  □ Document all design decisions
  □ Run all Move Prover proofs
  □ Fix all internal findings

Phase 4: AUDIT (8-12 weeks)
  □ Engage auditor (Spearbit, OtterSec, Halborn, etc.)
  □ Fix all Critical/High findings
  □ Fix all Medium findings
  □ Acknowledge Low/Info findings
  □ Final audit sign-off

Phase 5: TESTNET (4-8 weeks)
  □ Deploy to Aptos testnet / Sui testnet
  □ Public testnet with real users
  □ Private bug bounty (white hat hunters)
  □ Fix any new findings
  □ Stress test: simulate market conditions

Phase 6: MAINNET LAUNCH
  □ Seed liquidity (protocol-owned)
  □ TVL caps: start at $100k, raise weekly
  □ Monitoring active before launch
  □ Team on standby for 48h post-launch
  □ Public bug bounty live

Phase 7: GROWTH
  □ Liquidity mining incentives
  □ Strategic partnerships (other protocols)
  □ DEX aggregator integration
  □ Community governance transition

TOTAL TIME: 6-12 months for a well-executed protocol
TOTAL COST: 
  Audit: $50k-$500k (depends on size)
  Infrastructure: $2k-$20k/month
  Engineering: 4-8 engineers × 6-12 months
  Bug bounty: $50k-$500k ongoing
```

---

## สรุป Real-World Case Studies

```
KEY TAKEAWAYS FROM PRODUCTION PROTOCOLS

APTOS ECOSYSTEM
  Pancakeswap: CPMM AMM, multi-hop routing, veToken
  Thala: CDP stablecoin + PSM + AMM composability
  Aries Markets: lending protocol with liquidation bots
  
  Common patterns:
  - Start with CoinStore, migrate to FA
  - Use Pyth oracle with TWAP protection
  - veToken for governance + LP incentive alignment

SUI ECOSYSTEM  
  Turbos: CLMM (concentrated liquidity)
  Cetus: CLMM with Sui PTB flash arbitrage
  Aftermath: stablecoin AMM with deep liquidity
  
  Common patterns:
  - Objects as first-class positions (LP tokens, NFTs)
  - PTBs for complex atomic operations
  - kiosk for NFT marketplace with enforced royalties

UNIVERSAL TRUTHS (both chains)
  1. Oracle security is the foundation of all DeFi
  2. Liquidation must be profitable at any market price
  3. Audit + monitor = table stakes, not optional
  4. Start simple, earn trust, then add complexity
  5. Community trust takes years to build, seconds to lose
```

---

**ก่อนหน้า**: [Part 97 - Move Language Internals ←](part-97-move-internals.md)
**ต่อไป**: [Part 99 - Contributing to Move Ecosystem →](part-99-ecosystem-contribution.md)
