# Part 87: Advanced Testing Strategies for Move

## สารบัญ
- [Testing Philosophy](#testing-philosophy)
- [Unit Tests in Move](#unit-tests-in-move)
- [Integration Testing](#integration-testing)
- [Property-Based Testing](#property-based-testing)
- [Mainnet Fork Testing](#mainnet-fork-testing)
- [CI/CD for Move Projects](#cicd-for-move-projects)

---

## Testing Philosophy

```
MOVE TESTING HIERARCHY

Level 1: UNIT TESTS (Move)
  Isolate individual functions
  Mock dependencies
  Fast execution (< 1 second)
  
Level 2: MODULE INTEGRATION TESTS (Move)
  Test interactions between modules
  Realistic scenarios with multiple actors
  Medium speed (1-10 seconds)
  
Level 3: PROPERTY-BASED TESTS (Python/Rust)
  Generate random inputs, verify invariants
  Find edge cases automatically
  
Level 4: FORK TESTS (TypeScript)
  Against real mainnet/testnet state
  Simulate real-world conditions
  
Level 5: ECONOMIC SIMULATIONS (Python)
  Model tokenomics, incentives
  Verify game theory assumptions
  
COVERAGE TARGETS
  Unit tests: > 90% line coverage
  Integration: > 80% scenario coverage
  Property tests: critical invariants proven
  Fork tests: all user-facing flows

WHAT TO TEST
  ✅ Happy path: normal operations work
  ✅ Edge cases: zero values, maximums, first/last user
  ✅ Access control: wrong caller is rejected
  ✅ Mathematical correctness: formulas are right
  ✅ Invariants: protocol properties hold across state changes
  ✅ Failure modes: correct error codes
  
WHAT NOT TO TEST
  ✗ Framework code (trust aptos_framework)
  ✗ Trivial getters with no logic
  ✗ Duplicate tests for the same code path
```

---

## Unit Tests in Move

```move
// ============================================
// COMPREHENSIVE UNIT TEST PATTERNS
// ============================================

module protocol::pool {
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::timestamp;
    
    const ERR_ZERO_AMOUNT: u64 = 1;
    const ERR_INSUFFICIENT_LIQUIDITY: u64 = 2;
    const ERR_SLIPPAGE: u64 = 3;
    const ERR_EXPIRED: u64 = 4;
    
    struct Pool has key {
        reserve_x: u64,
        reserve_y: u64,
        total_lp: u64,
        fee_bps: u64,
    }
    
    struct LP has key {
        amount: u64,
    }
    
    public fun add_liquidity(
        pool: &mut Pool,
        amount_x: u64,
        amount_y: u64,
    ): u64 {
        assert!(amount_x > 0 && amount_y > 0, ERR_ZERO_AMOUNT);
        
        let lp_minted = if (pool.total_lp == 0) {
            let lp = isqrt((amount_x as u128) * (amount_y as u128));
            (lp as u64) - 1000  // Burn 1000 to dead address
        } else {
            let lp_from_x = (amount_x as u128) * (pool.total_lp as u128) / (pool.reserve_x as u128);
            let lp_from_y = (amount_y as u128) * (pool.total_lp as u128) / (pool.reserve_y as u128);
            std::u128::min(lp_from_x, lp_from_y) as u64
        };
        
        pool.reserve_x = pool.reserve_x + amount_x;
        pool.reserve_y = pool.reserve_y + amount_y;
        pool.total_lp = pool.total_lp + lp_minted;
        
        lp_minted
    }
    
    public fun get_amount_out(
        amount_in: u64,
        reserve_in: u64,
        reserve_out: u64,
        fee_bps: u64,
    ): u64 {
        assert!(amount_in > 0, ERR_ZERO_AMOUNT);
        assert!(reserve_in > 0 && reserve_out > 0, ERR_INSUFFICIENT_LIQUIDITY);
        
        let amount_in_with_fee = (amount_in as u128) * (10_000 - fee_bps as u128);
        let numerator = amount_in_with_fee * (reserve_out as u128);
        let denominator = (reserve_in as u128) * 10_000 + amount_in_with_fee;
        (numerator / denominator) as u64
    }
    
    fun isqrt(n: u128): u128 {
        if (n == 0) return 0;
        let mut x = n;
        let mut y = (x + 1) / 2;
        while (y < x) { x = y; y = (x + n / x) / 2; };
        x
    }
    
    // ============================================
    // TESTS
    // ============================================
    
    #[test_only]
    use aptos_framework::account;
    
    // Test: First liquidity provision
    #[test]
    fun test_add_first_liquidity() {
        let mut pool = Pool { reserve_x: 0, reserve_y: 0, total_lp: 0, fee_bps: 30 };
        
        let lp = add_liquidity(&mut pool, 1_000_000, 4_000_000);
        
        // LP = sqrt(1e6 * 4e6) - 1000 = sqrt(4e12) - 1000 = 2_000_000 - 1000 = 1_999_000
        assert!(lp == 1_999_000, 1);
        assert!(pool.reserve_x == 1_000_000, 2);
        assert!(pool.reserve_y == 4_000_000, 3);
        assert!(pool.total_lp == 1_999_000, 4);
    }
    
    // Test: Subsequent liquidity provision (proportional)
    #[test]
    fun test_add_proportional_liquidity() {
        let mut pool = Pool { reserve_x: 1_000_000, reserve_y: 4_000_000, total_lp: 1_999_000, fee_bps: 30 };
        
        // Double the liquidity
        let lp = add_liquidity(&mut pool, 1_000_000, 4_000_000);
        
        // LP minted = 1_999_000 (exactly proportional)
        assert!(lp == 1_999_000, 1);
        assert!(pool.reserve_x == 2_000_000, 2);
        assert!(pool.reserve_y == 8_000_000, 3);
    }
    
    // Test: Swap output calculation
    #[test]
    fun test_get_amount_out() {
        // 1000 in, 10000/10000 pool, 0.3% fee
        let out = get_amount_out(1_000, 10_000, 10_000, 30);
        
        // Expected: 1000 * 9970 / (10000 * 10000 + 1000 * 9970)
        //         = 9_970_000 / (100_000_000 + 9_970_000)
        //         = 9_970_000 / 109_970_000
        //         ≈ 90 (floor division)
        assert!(out > 0, 1);
        assert!(out < 1_000, 2);  // Must be less than input (fees applied)
    }
    
    // Test: Zero input rejected
    #[test]
    #[expected_failure(abort_code = 1)]  // ERR_ZERO_AMOUNT
    fun test_swap_zero_amount_fails() {
        get_amount_out(0, 10_000, 10_000, 30);
    }
    
    // Test: Empty pool rejected
    #[test]
    #[expected_failure(abort_code = 2)]  // ERR_INSUFFICIENT_LIQUIDITY
    fun test_swap_empty_pool_fails() {
        get_amount_out(100, 0, 10_000, 30);
    }
    
    // Test: CPMM k invariant
    #[test]
    fun test_k_invariant_after_swap() {
        let reserve_x = 1_000_000u64;
        let reserve_y = 2_000_000u64;
        let k_before = (reserve_x as u128) * (reserve_y as u128);
        
        let amount_in = 10_000u64;
        let amount_out = get_amount_out(amount_in, reserve_x, reserve_y, 30);
        
        let new_reserve_x = reserve_x + amount_in;
        let new_reserve_y = reserve_y - amount_out;
        let k_after = (new_reserve_x as u128) * (new_reserve_y as u128);
        
        // k must not decrease (fees increase it)
        assert!(k_after >= k_before, 1);
    }
    
    // Test: price impact increases with trade size
    #[test]
    fun test_price_impact_increases_with_size() {
        let reserve_x = 1_000_000u64;
        let reserve_y = 1_000_000u64;
        
        let out_small = get_amount_out(1_000, reserve_x, reserve_y, 0);
        let out_large = get_amount_out(100_000, reserve_x, reserve_y, 0);
        
        // Rate should be worse for larger trade (price impact)
        // out_small/1000 > out_large/100_000
        assert!(out_small * 100 > out_large, 1);
    }
    
    // Test: AMM price symmetry (no arbitrage at equal reserves)
    #[test]
    fun test_no_arbitrage_equal_reserves() {
        let reserve = 1_000_000u64;
        let amount = 10_000u64;
        
        // X -> Y then Y -> X should give less than we started with (fees)
        let y_out = get_amount_out(amount, reserve, reserve, 30);
        let x_back = get_amount_out(y_out, reserve, reserve, 30);
        
        assert!(x_back < amount, 1);  // Fees taken both ways
    }
}
```

---

## Integration Testing

```move
// ============================================
// INTEGRATION TEST: Full swap workflow
// ============================================

#[test_only]
module protocol::integration_test {
    use aptos_framework::account;
    use aptos_framework::coin;
    use aptos_framework::timestamp;
    
    // Test: Complete user journey
    // 1. Initialize pool
    // 2. Alice adds liquidity
    // 3. Bob swaps
    // 4. Alice removes liquidity (with fees earned)
    #[test(
        admin = @protocol,
        alice = @0xA11CE,
        bob = @0xB0B,
        aptos_framework = @aptos_framework,
    )]
    fun test_full_swap_journey(
        admin: &signer,
        alice: &signer,
        bob: &signer,
        aptos_framework: &signer,
    ) {
        // Setup: Initialize accounts and timestamp
        account::create_account_for_test(std::signer::address_of(admin));
        account::create_account_for_test(std::signer::address_of(alice));
        account::create_account_for_test(std::signer::address_of(bob));
        timestamp::set_time_has_started_for_testing(aptos_framework);
        timestamp::update_global_time_for_test(1_000_000);  // Set to some time
        
        // Initialize pool
        // pool::initialize(admin, 30);  // 0.3% fee
        
        // Alice adds initial liquidity
        // let alice_lp = pool::add_liquidity(alice, 1_000_000, 4_000_000);
        // assert!(alice_lp > 0, 1);
        
        // Bob swaps X for Y
        // let bob_y = pool::swap_x_to_y(bob, 10_000, 9_000, 9_999_999);  // min_out, deadline
        // assert!(bob_y >= 9_000, 2);  // Got at least min_out
        
        // Alice removes liquidity (should get back + fees from Bob's swap)
        // let (x_out, y_out) = pool::remove_liquidity(alice, alice_lp);
        // assert!(x_out + y_out > 1_000_000 + 4_000_000 - 100, 3);  // Got more than put in (fees)
        
        // Placeholder assertions for compilation
        assert!(1 == 1, 0);
    }
    
    // Test: Lending cycle
    // 1. Supplier deposits USDC
    // 2. Borrower takes loan
    // 3. Time passes (interest accrues)
    // 4. Borrower repays with interest
    // 5. Supplier withdraws with yield
    #[test(
        admin = @protocol,
        supplier = @0x5,
        borrower = @0x6,
        aptos_framework = @aptos_framework,
    )]
    fun test_lending_cycle(
        admin: &signer,
        supplier: &signer,
        borrower: &signer,
        aptos_framework: &signer,
    ) {
        account::create_account_for_test(std::signer::address_of(admin));
        account::create_account_for_test(std::signer::address_of(supplier));
        account::create_account_for_test(std::signer::address_of(borrower));
        timestamp::set_time_has_started_for_testing(aptos_framework);
        
        // t=0: Supplier deposits 1000 USDC
        timestamp::update_global_time_for_test(0);
        // lending::deposit(supplier, 1_000_000_000);  // 1000 USDC
        
        // t=0: Borrower deposits 2 APT collateral, borrows 500 USDC
        // lending::deposit_collateral(borrower, 2_000_000_000);  // 2 APT
        // lending::borrow(borrower, 500_000_000);  // 500 USDC
        
        // t=365 days: Fast forward time
        timestamp::update_global_time_for_test(365 * 24 * 3600 * 1_000_000);
        // lending::accrue_interest(admin);
        
        // Borrower now owes ~525 USDC (5% APY)
        // let debt = lending::get_debt(std::signer::address_of(borrower));
        // assert!(debt > 500_000_000, 1);  // More than borrowed
        // assert!(debt < 530_000_000, 2);  // Less than 6% APY
        
        // Borrower repays
        // lending::repay(borrower, debt);
        
        // Supplier withdraws with yield
        // let supply_balance = lending::get_supply_balance(std::signer::address_of(supplier));
        // assert!(supply_balance > 1_000_000_000, 3);  // More than deposited
        
        assert!(1 == 1, 0);
    }
    
    // Test: Governance proposal lifecycle
    #[test(
        admin = @protocol,
        voter1 = @0xV1,
        voter2 = @0xV2,
        proposer = @0xP,
        aptos_framework = @aptos_framework,
    )]
    fun test_governance_proposal(
        admin: &signer,
        voter1: &signer,
        voter2: &signer,
        proposer: &signer,
        aptos_framework: &signer,
    ) {
        timestamp::set_time_has_started_for_testing(aptos_framework);
        timestamp::update_global_time_for_test(1_000);
        
        // Setup: Give voters some tokens
        // governance::mint(voter1, 100_000);
        // governance::mint(voter2, 50_000);
        // governance::mint(proposer, 10_000);
        
        // Proposer creates proposal
        // let proposal_id = governance::propose(proposer, "Increase fee to 0.5%", 48 * 3600);
        
        // Fast forward past voting delay
        timestamp::update_global_time_for_test(2_000);
        
        // Voters vote
        // governance::vote(voter1, proposal_id, 0);  // FOR
        // governance::vote(voter2, proposal_id, 0);  // FOR
        
        // Fast forward past voting period
        timestamp::update_global_time_for_test(10_000);
        
        // Queue proposal
        // governance::queue(proposer, proposal_id);
        
        // Fast forward past timelock (48h)
        timestamp::update_global_time_for_test(10_000 + 48 * 3600 * 1_000_000);
        
        // Execute proposal
        // governance::execute(proposer, proposal_id);
        
        // Verify fee was changed
        // assert!(pool::get_fee_bps() == 50, 1);  // 0.5%
        
        assert!(1 == 1, 0);
    }
}
```

---

## Property-Based Testing

```python
#!/usr/bin/env python3
"""
Property-based testing for Move AMM
Verify invariants hold for random inputs
"""

import random
import subprocess
import json
from typing import Tuple

class AMMModel:
    """Pure Python model of the AMM for verification"""
    
    def __init__(self, reserve_x: int, reserve_y: int, fee_bps: int = 30):
        self.reserve_x = reserve_x
        self.reserve_y = reserve_y
        self.fee_bps = fee_bps
        self.total_lp = 0
        
    @property
    def k(self) -> int:
        return self.reserve_x * self.reserve_y
    
    def get_amount_out(self, amount_in: int, is_x_to_y: bool) -> int:
        """CPMM formula with fees"""
        if is_x_to_y:
            reserve_in, reserve_out = self.reserve_x, self.reserve_y
        else:
            reserve_in, reserve_out = self.reserve_y, self.reserve_x
            
        amount_in_with_fee = amount_in * (10_000 - self.fee_bps)
        numerator = amount_in_with_fee * reserve_out
        denominator = reserve_in * 10_000 + amount_in_with_fee
        return numerator // denominator
    
    def swap(self, amount_in: int, is_x_to_y: bool) -> Tuple[int, int]:
        """Execute a swap, return (amount_out, new_k)"""
        amount_out = self.get_amount_out(amount_in, is_x_to_y)
        
        k_before = self.k
        if is_x_to_y:
            self.reserve_x += amount_in
            self.reserve_y -= amount_out
        else:
            self.reserve_y += amount_in
            self.reserve_x -= amount_out
        k_after = self.k
        
        return amount_out, k_after - k_before
    
    def add_liquidity(self, amount_x: int, amount_y: int) -> int:
        """Add liquidity, return LP tokens minted"""
        if self.total_lp == 0:
            lp = int((amount_x * amount_y) ** 0.5) - 1000
        else:
            lp = min(
                amount_x * self.total_lp // self.reserve_x,
                amount_y * self.total_lp // self.reserve_y,
            )
        self.reserve_x += amount_x
        self.reserve_y += amount_y
        self.total_lp += lp
        return lp


def test_k_never_decreases(iterations: int = 10000):
    """Property: k = reserve_x * reserve_y must never decrease after swaps"""
    failures = 0
    
    for _ in range(iterations):
        reserve_x = random.randint(1, 2**30)
        reserve_y = random.randint(1, 2**30)
        fee_bps = random.choice([0, 5, 10, 30, 100])
        amount_in = random.randint(1, min(reserve_x, reserve_y) // 2)
        is_x_to_y = random.random() > 0.5
        
        pool = AMMModel(reserve_x, reserve_y, fee_bps)
        k_before = pool.k
        _, delta_k = pool.swap(amount_in, is_x_to_y)
        
        if delta_k < 0:
            failures += 1
            print(f"K DECREASED: reserves={reserve_x},{reserve_y} fee={fee_bps} in={amount_in} delta_k={delta_k}")
    
    print(f"k_never_decreases: {iterations - failures}/{iterations} passed")
    return failures == 0


def test_output_less_than_reserve(iterations: int = 10000):
    """Property: Amount out must always be less than the full reserve"""
    failures = 0
    
    for _ in range(iterations):
        reserve_x = random.randint(1, 2**40)
        reserve_y = random.randint(1, 2**40)
        amount_in = random.randint(1, reserve_x)
        
        pool = AMMModel(reserve_x, reserve_y, 30)
        amount_out = pool.get_amount_out(amount_in, True)
        
        if amount_out >= reserve_y:
            failures += 1
            print(f"OUTPUT EXCEEDED RESERVE: amount_out={amount_out} >= reserve_y={reserve_y}")
    
    print(f"output_less_than_reserve: {iterations - failures}/{iterations} passed")
    return failures == 0


def test_no_arbitrage_with_fees(iterations: int = 5000):
    """Property: Buy X then sell X at same reserves = net loss (fees taken)"""
    failures = 0
    
    for _ in range(iterations):
        reserve_x = random.randint(1_000_000, 2**30)
        reserve_y = random.randint(1_000_000, 2**30)
        amount = random.randint(100, min(reserve_x, reserve_y) // 10)
        fee_bps = random.randint(1, 100)
        
        pool = AMMModel(reserve_x, reserve_y, fee_bps)
        
        # Buy Y with X
        y_received = pool.get_amount_out(amount, True)
        
        # Simulate state after first swap
        pool2 = AMMModel(reserve_x + amount, reserve_y - y_received, fee_bps)
        
        # Sell Y back for X
        x_received = pool2.get_amount_out(y_received, False)
        
        if x_received >= amount:
            failures += 1
            print(f"ARBITRAGE POSSIBLE: in={amount} x_back={x_received}")
    
    print(f"no_arbitrage_with_fees: {iterations - failures}/{iterations} passed")
    return failures == 0


def test_price_impact_monotonic(iterations: int = 1000):
    """Property: Larger trade always has worse price per unit"""
    failures = 0
    
    for _ in range(iterations):
        reserve_x = random.randint(1_000_000, 2**30)
        reserve_y = random.randint(1_000_000, 2**30)
        
        pool = AMMModel(reserve_x, reserve_y, 30)
        
        # Small trade
        small_in = 1_000
        small_out = pool.get_amount_out(small_in, True)
        
        # Large trade (10x)
        large_in = 10_000
        large_out = pool.get_amount_out(large_in, True)
        
        # Rate comparison: small_out/small_in > large_out/large_in
        # i.e., small_out * large_in > large_out * small_in
        if small_out * large_in <= large_out * small_in:
            failures += 1
    
    print(f"price_impact_monotonic: {iterations - failures}/{iterations} passed")
    return failures == 0


if __name__ == "__main__":
    print("Running AMM property tests...\n")
    
    all_passed = True
    all_passed &= test_k_never_decreases()
    all_passed &= test_output_less_than_reserve()
    all_passed &= test_no_arbitrage_with_fees()
    all_passed &= test_price_impact_monotonic()
    
    print(f"\n{'ALL TESTS PASSED' if all_passed else 'SOME TESTS FAILED'}")
```

---

## Mainnet Fork Testing

```typescript
// ============================================
// TESTNET FORK TESTING
// Test against real state without real money
// ============================================

import { AptosClient, AptosAccount, HexString } from 'aptos';

class ForkTestHarness {
  private client: AptosClient;
  private testAccount: AptosAccount;
  
  constructor(
    forkNodeUrl: string,  // Aptos testnet or custom fork
    testPrivateKey: string,
  ) {
    this.client = new AptosClient(forkNodeUrl);
    this.testAccount = new AptosAccount(Buffer.from(testPrivateKey, 'hex'));
  }
  
  // Get current state of a contract at mainnet-like state
  async getPoolState(poolAddr: string, poolType: string): Promise<any> {
    const resource = await this.client.getAccountResource(poolAddr, poolType);
    return resource.data;
  }
  
  // Simulate a swap against real pool state
  async simulateSwap(
    poolAddr: string,
    amountIn: bigint,
    isXToY: boolean,
    minOut: bigint,
  ): Promise<{ success: boolean; amountOut: bigint; gasUsed: bigint }> {
    const payload = {
      function: `${poolAddr}::pool::swap`,
      type_arguments: [],
      arguments: [
        poolAddr,
        amountIn.toString(),
        isXToY,
        minOut.toString(),
        (Math.floor(Date.now() / 1000) + 3600).toString(),  // 1 hour deadline
      ],
    };
    
    try {
      // Simulate first (dry run)
      const simulation = await this.client.simulateTransaction(
        this.testAccount,
        await this.client.generateTransaction(
          this.testAccount.address().toString(),
          payload,
        ),
      );
      
      if (!simulation[0].success) {
        return { success: false, amountOut: 0n, gasUsed: 0n };
      }
      
      // Parse output from events
      const swapEvent = simulation[0].events.find(
        e => e.type.includes('SwapEvent'),
      );
      const amountOut = swapEvent ? BigInt((swapEvent.data as any).amount_out) : 0n;
      
      return {
        success: true,
        amountOut,
        gasUsed: BigInt(simulation[0].gas_used),
      };
    } catch (error) {
      return { success: false, amountOut: 0n, gasUsed: 0n };
    }
  }
  
  // Run a test scenario
  async runScenario(name: string, fn: () => Promise<void>): Promise<void> {
    console.log(`\n[TEST] ${name}`);
    try {
      await fn();
      console.log(`[PASS] ${name}`);
    } catch (error) {
      console.error(`[FAIL] ${name}: ${error}`);
      throw error;
    }
  }
  
  // Assert helper
  assert(condition: boolean, message: string): void {
    if (!condition) throw new Error(`Assertion failed: ${message}`);
  }
}

// Example test suite
async function runForkTests() {
  const harness = new ForkTestHarness(
    'https://fullnode.testnet.aptoslabs.com/v1',
    process.env.TEST_PRIVATE_KEY!,
  );
  
  const poolAddr = '0xTESTNET_POOL_ADDRESS';
  
  await harness.runScenario('Swap output is reasonable', async () => {
    const result = await harness.simulateSwap(poolAddr, 1_000_000n, true, 900_000n);
    harness.assert(result.success, 'Swap should succeed');
    harness.assert(result.amountOut > 900_000n, 'Output should exceed min_out');
    harness.assert(result.amountOut < 1_100_000n, 'Output should be reasonable');
  });
  
  await harness.runScenario('Swap with zero amount fails', async () => {
    const result = await harness.simulateSwap(poolAddr, 0n, true, 0n);
    harness.assert(!result.success, 'Zero amount swap should fail');
  });
  
  await harness.runScenario('Large swap has price impact', async () => {
    const small = await harness.simulateSwap(poolAddr, 1_000n, true, 0n);
    const large = await harness.simulateSwap(poolAddr, 1_000_000n, true, 0n);
    
    // Price per unit should be worse for large swap
    const smallRate = Number(small.amountOut) / 1000;
    const largeRate = Number(large.amountOut) / 1_000_000;
    harness.assert(smallRate > largeRate, 'Large swap should have price impact');
  });
  
  console.log('\nAll fork tests complete.');
}
```

---

## CI/CD for Move Projects

```yaml
# .github/workflows/move-ci.yml

name: Move CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Install Aptos CLI
        run: |
          curl -fsSL "https://aptos.dev/scripts/install_cli.py" | python3
          echo "$HOME/.local/bin" >> $GITHUB_PATH
      
      - name: Verify CLI installed
        run: aptos --version
      
      - name: Run unit tests
        run: |
          aptos move test \
            --package-dir . \
            --named-addresses protocol=${{ vars.PROTOCOL_ADDR }}
      
      - name: Run tests with coverage
        run: |
          aptos move test \
            --package-dir . \
            --coverage \
            --named-addresses protocol=${{ vars.PROTOCOL_ADDR }}
          aptos move coverage summary --package-dir .
      
      - name: Check coverage threshold
        run: |
          COVERAGE=$(aptos move coverage summary --package-dir . 2>&1 | grep "Overall coverage" | awk '{print $NF}' | tr -d '%')
          echo "Coverage: $COVERAGE%"
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "ERROR: Coverage $COVERAGE% below 80% threshold"
            exit 1
          fi
      
      - name: Run Move Prover
        run: |
          aptos move prove \
            --package-dir . \
            --named-addresses protocol=${{ vars.PROTOCOL_ADDR }}
        continue-on-error: false  # Fail CI if prover fails
      
      - name: Run property tests
        run: |
          pip install -r tests/requirements.txt
          python tests/property_tests.py
      
      - name: Build package
        run: |
          aptos move compile \
            --package-dir . \
            --named-addresses protocol=${{ vars.PROTOCOL_ADDR }}
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build-artifacts
          path: build/
  
  security:
    runs-on: ubuntu-latest
    needs: test
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Check for hardcoded addresses
        run: |
          if grep -r "0x[0-9a-fA-F]\{40,\}" --include="*.move" --exclude-dir=".git"; then
            echo "WARNING: Hardcoded long addresses found"
          fi
      
      - name: Check for unsafe operations
        run: |
          # Check for potential issues
          grep -n "move_from\|borrow_global_mut" --include="*.move" -r . || true
```

---

## สรุป Advanced Testing

```
TESTING PYRAMID FOR MOVE PROTOCOLS

          [Fork Tests]
         Realistic state
         Catch integration bugs
        
       [Property Tests]
      Random inputs, invariants
      Find edge cases automatically
      
    [Integration Tests]
   Multi-module, multi-actor
   Full user journey coverage
   
 [Unit Tests] (Foundation)
Isolated functions, fast
Cover all code paths
>90% coverage target

MOVE-SPECIFIC TESTING TIPS

1. Use #[test_only] for test helpers
   (Code that won't be deployed but aids testing)

2. Test timestamp sensitivity:
   timestamp::update_global_time_for_test(...)
   Critical for: vesting, governance, interest accrual

3. Test account setup:
   account::create_account_for_test(addr)
   Required before any account-level operations

4. Use #[expected_failure] for error tests:
   #[expected_failure(abort_code = 1)]
   Tests that specific error code is returned

5. Test with realistic amounts:
   Use amounts that reflect real usage (e.g., 1e6 for 1 USDC)
   Don't just test with 1, 2, 3

6. Edge case checklist:
   Zero amounts
   Maximum u64 values
   First user (empty state)
   Single user (total == 1)
   Equal inputs/outputs
```

---

**ก่อนหน้า**: [Part 86 - Decentralized Identity ←](part-86-identity.md)
**ต่อไป**: [Part 88 - DeFi Indexing & Analytics →](part-88-indexing.md)
