# Part 40: Production Deployment & DevOps

## สารบัญ
- [Deployment Checklist](#deployment-checklist)
- [Move.toml Configuration](#movetoml-configuration)
- [Scripts & Entry Functions](#scripts--entry-functions)
- [Testing Framework](#testing-framework)
- [CI/CD Pipeline](#cicd-pipeline)
- [ตัวอย่าง: Full Deployment Flow](#ตัวอย่าง-full-deployment-flow)

---

## Deployment Checklist

```
PRE-DEPLOYMENT:
  □ All unit tests pass (aptos move test)
  □ Move Prover specs pass (aptos move prove)
  □ Security audit completed
  □ Gas optimization done
  □ Testnet deployment verified
  □ Admin keys secured (hardware wallet / multisig)
  □ Upgrade policy decided (immutable/compatible/arbitrary)
  □ Emergency procedures documented
  □ Monitoring setup ready

DEPLOYMENT:
  □ Deploy to mainnet with correct addresses
  □ Initialize all modules in correct order
  □ Verify state post-initialization
  □ Grant initial permissions/roles
  □ Set initial parameters
  □ Verify contract on explorer

POST-DEPLOYMENT:
  □ Monitor events for 24h
  □ Test all critical paths
  □ Public announcement
  □ Bug bounty launched
  □ Incident response on standby
```

---

## Move.toml Configuration

```toml
# Move.toml - Production configuration
[package]
name = "MyProtocol"
version = "1.0.0"
authors = ["Team <team@myprotocol.xyz>"]

[addresses]
# Named addresses for different environments
my_protocol = "_"         # Set at deploy time
aptos_framework = "0x1"
aptos_std = "0x1"

[dev-addresses]
# Override addresses for testing
my_protocol = "0xCAFE"

[dependencies.AptosFramework]
git = "https://github.com/aptos-labs/aptos-core.git"
rev = "mainnet"
subdir = "aptos-move/framework/aptos-framework"

[dependencies.AptosStdlib]
git = "https://github.com/aptos-labs/aptos-core.git"
rev = "mainnet"
subdir = "aptos-move/framework/aptos-stdlib"

[dev-dependencies]
# Dependencies only for testing

[prover]
timeout = 120
backend = "z3"
```

---

## Scripts & Entry Functions

```move
module deployment::init_scripts {
    use std::signer;
    use aptos_framework::account;
    
    // ============================================
    // Initialization scripts (called once at deploy)
    // ============================================
    
    // The canonical init pattern: all in one function
    public entry fun initialize_protocol(
        deployer: &signer,
        // Configuration parameters
        fee_bps: u64,
        treasury: address,
        governance: address,
        initial_admin: address,
    ) {
        let deployer_addr = signer::address_of(deployer);
        
        // 1. Initialize core state
        // my_protocol::core::initialize(deployer, fee_bps);
        
        // 2. Initialize token
        // my_protocol::token::initialize(deployer);
        
        // 3. Setup roles
        // my_protocol::access::setup_roles(deployer, initial_admin);
        
        // 4. Set treasury
        // my_protocol::treasury::set(deployer, treasury);
        
        // 5. Set governance
        // my_protocol::governance::set(deployer, governance);
        
        // 6. Emit deployment event
        aptos_framework::event::emit(ProtocolInitialized {
            deployer: deployer_addr,
            fee_bps,
            treasury,
            governance,
            timestamp: aptos_framework::timestamp::now_seconds(),
        });
    }
    
    #[event]
    struct ProtocolInitialized has drop, store {
        deployer: address,
        fee_bps: u64,
        treasury: address,
        governance: address,
        timestamp: u64,
    }
    
    // ============================================
    // Post-deploy configuration
    // ============================================
    
    public entry fun configure_v1(
        admin: &signer,
        max_fee_bps: u64,
        min_liquidity: u64,
        oracle_addr: address,
    ) {
        // Configure after deploy
        // Called separately to stay under gas limits
    }
    
    // ============================================
    // Emergency procedures
    // ============================================
    
    public entry fun emergency_shutdown(
        admin: &signer,
        protocol_addr: address,
        reason: vector<u8>,
    ) {
        // Pause all operations
        // Emit emergency event
        // Notify monitoring systems (via events)
    }
    
    public entry fun emergency_drain(
        admin: &signer,
        protocol_addr: address,
        to: address,
    ) {
        // ONLY for critical vulnerabilities
        // Move all funds to safety address
        // Requires multisig + timelock override
    }
}
```

---

## Testing Framework

```move
module deployment::comprehensive_tests {
    use std::signer;
    use aptos_framework::account;
    use aptos_framework::timestamp;
    use aptos_framework::coin;
    
    // ============================================
    // Test setup helpers
    // ============================================
    
    #[test_only]
    public fun setup_test_env(scenario: &signer): (address, address, address) {
        // Create test accounts
        let admin = account::create_account_for_test(@0xADMIN);
        let alice = account::create_account_for_test(@0xALICE);
        let bob = account::create_account_for_test(@0xBOB);
        
        // Initialize timestamp
        timestamp::set_time_has_started_for_testing(scenario);
        timestamp::update_global_time_for_test_secs(1_000_000);
        
        (@0xADMIN, @0xALICE, @0xBOB)
    }
    
    // ============================================
    // Unit tests
    // ============================================
    
    #[test]
    fun test_basic_transfer() {
        // ...
    }
    
    #[test]
    #[expected_failure(abort_code = 1)]
    fun test_unauthorized_mint() {
        // Should abort with code 1
    }
    
    #[test]
    fun test_deposit_withdraw_cycle() {
        // Test full deposit -> withdraw cycle
        // Verify final state equals initial state
    }
    
    // ============================================
    // Integration tests (test multiple modules)
    // ============================================
    
    #[test]
    fun test_amm_integration() {
        // 1. Create pool
        // 2. Add liquidity
        // 3. Swap
        // 4. Remove liquidity
        // 5. Verify conservation
    }
    
    // ============================================
    // Fuzz-like tests (boundary conditions)
    // ============================================
    
    #[test]
    fun test_boundary_values() {
        // Test with u64::MAX
        let max = 18_446_744_073_709_551_615u64;
        
        // Test with 0
        // Test with 1
        // Test exact boundary (should succeed)
        // Test over boundary (should fail)
    }
    
    #[test]
    fun test_precision_math() {
        // Verify no precision loss in fee calculations
        // fee_bps = 30, amount = 1
        // expected fee = 0 (rounds down)
        let fee = 1u64 * 30 / 10_000;
        assert!(fee == 0, 1);
        
        // amount = 10_000 (fee = 3)
        let fee = 10_000u64 * 30 / 10_000;
        assert!(fee == 3, 2);
        
        // Verify minimum fee threshold
    }
    
    // ============================================
    // Time-based tests
    // ============================================
    
    #[test]
    fun test_vesting_schedule() {
        let start = 1_000_000u64;
        let duration = 86_400u64 * 365;  // 1 year
        
        // At start: 0 vested
        // At 6 months: 50% vested
        // At 1 year: 100% vested
        // After 1 year: no more
        
        let half_year = start + duration / 2;
        // timestamp::fast_forward_seconds(half_year);
        // let vested = calculate_vested(start, duration, total);
        // assert!(vested == total / 2, 1);
    }
    
    // ============================================
    // Gas benchmarking tests
    // ============================================
    
    #[test]
    fun benchmark_swap_gas() {
        // aptos move test shows gas used per test
        // Ensure swap stays under gas budget
        // Typical: swap should use < 1000 gas units
    }
}
```

---

## CI/CD Pipeline

```yaml
# .github/workflows/move_ci.yml
name: Move CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    name: Move Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Install Aptos CLI
        run: |
          curl -fsSL "https://aptos.dev/scripts/install_cli.py" | python3
          echo "$HOME/.local/bin" >> $GITHUB_PATH
      
      - name: Compile
        run: |
          aptos move compile \
            --named-addresses my_protocol=0xCAFE
      
      - name: Run Tests
        run: |
          aptos move test \
            --named-addresses my_protocol=0xCAFE \
            --coverage
      
      - name: Coverage Report
        run: |
          aptos move coverage summary \
            --named-addresses my_protocol=0xCAFE
      
      - name: Run Prover (on main only)
        if: github.ref == 'refs/heads/main'
        run: |
          aptos move prove \
            --named-addresses my_protocol=0xCAFE \
            --timeout 120
      
      - name: Security Checks
        run: |
          # Custom security linting
          grep -r "move_to" --include="*.move" | grep -v "#\[test" | wc -l
          # Check for proper error codes
          grep -r "assert!" --include="*.move" | grep -v "with [0-9]" | wc -l

  deploy-testnet:
    name: Deploy to Testnet
    needs: test
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Testnet
        env:
          TESTNET_PRIVATE_KEY: ${{ secrets.TESTNET_PRIVATE_KEY }}
        run: |
          aptos move publish \
            --named-addresses my_protocol=$TESTNET_ADDR \
            --private-key $TESTNET_PRIVATE_KEY \
            --url https://fullnode.testnet.aptoslabs.com/v1

  deploy-mainnet:
    name: Deploy to Mainnet
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    environment: mainnet  # Requires manual approval
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Mainnet
        env:
          MAINNET_PRIVATE_KEY: ${{ secrets.MAINNET_PRIVATE_KEY }}
        run: |
          aptos move publish \
            --named-addresses my_protocol=$MAINNET_ADDR \
            --private-key $MAINNET_PRIVATE_KEY \
            --url https://fullnode.mainnet.aptoslabs.com/v1 \
            --upgrade-policy compatible
```

---

## ตัวอย่าง: Full Deployment Flow

```bash
#!/bin/bash
# deploy.sh - Production deployment script

set -e  # Exit on any error

NETWORK="mainnet"
PRIVATE_KEY="${DEPLOY_PRIVATE_KEY}"  # From environment
PROTOCOL_ADDR="0xYOUR_PROTOCOL_ADDRESS"
TREASURY_ADDR="0xTREASURY_ADDRESS"
GOVERNANCE_ADDR="0xGOVERNANCE_ADDRESS"

echo "=== Move Protocol Deployment ==="
echo "Network: $NETWORK"
echo "Protocol: $PROTOCOL_ADDR"
echo ""

# Step 1: Compile
echo "Step 1: Compiling..."
aptos move compile \
  --named-addresses my_protocol=$PROTOCOL_ADDR \
  --save-metadata

# Step 2: Run tests one more time
echo "Step 2: Final test run..."
aptos move test \
  --named-addresses my_protocol=$PROTOCOL_ADDR

# Step 3: Run prover
echo "Step 3: Formal verification..."
aptos move prove \
  --named-addresses my_protocol=$PROTOCOL_ADDR

# Step 4: Deploy
echo "Step 4: Deploying to $NETWORK..."
aptos move publish \
  --named-addresses my_protocol=$PROTOCOL_ADDR \
  --private-key $PRIVATE_KEY \
  --url https://fullnode.mainnet.aptoslabs.com/v1 \
  --upgrade-policy compatible \
  --assume-yes

# Step 5: Initialize
echo "Step 5: Initializing protocol..."
aptos move run \
  --function-id "${PROTOCOL_ADDR}::init_scripts::initialize_protocol" \
  --args "u64:30" \        # fee_bps = 30 (0.3%)
         "address:$TREASURY_ADDR" \
         "address:$GOVERNANCE_ADDR" \
         "address:$PROTOCOL_ADDR" \
  --private-key $PRIVATE_KEY \
  --url https://fullnode.mainnet.aptoslabs.com/v1

# Step 6: Verify deployment
echo "Step 6: Verifying..."
aptos move view \
  --function-id "${PROTOCOL_ADDR}::init_scripts::get_version" \
  --url https://fullnode.mainnet.aptoslabs.com/v1

echo "=== Deployment Complete ==="
echo "Contract: $PROTOCOL_ADDR"
echo "Explorer: https://explorer.aptoslabs.com/account/$PROTOCOL_ADDR"
```

---

## Monitoring Setup

```typescript
// monitor.ts - Real-time event monitoring
import { AptosClient, AptosEventFetcher } from "@aptos-labs/ts-sdk";

const client = new AptosClient("https://fullnode.mainnet.aptoslabs.com/v1");

const PROTOCOL_ADDR = "0xYOUR_PROTOCOL_ADDRESS";
const ALERT_THRESHOLD = 1_000_000_000n;  // Alert on large transactions

async function monitorSwaps() {
  const lastSeq = await getLastProcessedSeq();
  
  while (true) {
    // Fetch new events
    const events = await client.getEventsByEventHandle({
      address: PROTOCOL_ADDR,
      eventHandleStruct: `${PROTOCOL_ADDR}::amm::SwapEvent`,
      fieldName: "swap_events",
      query: { start: lastSeq, limit: 100 }
    });
    
    for (const event of events) {
      const data = event.data;
      
      // Alert on large swaps
      if (BigInt(data.amount) >= ALERT_THRESHOLD) {
        await sendAlert({
          type: "LARGE_SWAP",
          amount: data.amount,
          user: data.user,
          tx: event.transaction_hash,
        });
      }
      
      // Check for abnormal price impact
      if (Number(data.price_impact_bps) > 500) {
        await sendAlert({
          type: "HIGH_PRICE_IMPACT",
          impact_bps: data.price_impact_bps,
          tx: event.transaction_hash,
        });
      }
    }
    
    await sleep(5000);  // Poll every 5 seconds
  }
}

async function monitorPoolHealth() {
  while (true) {
    // Check pool reserves
    const poolState = await client.view({
      function: `${PROTOCOL_ADDR}::amm::get_pool_state`,
      arguments: [POOL_ADDR],
    });
    
    const [reserveX, reserveY, k] = poolState;
    
    // Verify k invariant
    const expectedK = BigInt(reserveX) * BigInt(reserveY);
    if (BigInt(k) < expectedK * 99n / 100n) {  // k dropped more than 1%
      await sendAlert({
        type: "K_INVARIANT_VIOLATION",
        k_current: k,
        k_expected: expectedK.toString(),
      });
    }
    
    await sleep(30_000);  // Check every 30 seconds
  }
}
```

---

## Gas Optimization Tips

```
Move Gas Optimization:

1. Avoid unnecessary resource access
   BAD:  borrow_global<T>(addr).field
   GOOD: let r = borrow_global<T>(addr); r.field  [same thing]
   
2. Use SmartTable not Table for iteration needs

3. Batch operations in single transaction

4. Avoid vector operations in loops:
   BAD: vector::push_back in loop
   GOOD: construct final vector, push once

5. Use u64 for most calculations (cheaper than u128)
   Upgrade to u128 only when needed for precision

6. Avoid storing large data on-chain
   Use off-chain storage (IPFS) + on-chain hash

7. Move resources to cheaper locations:
   Resource at protocol addr: cheaper reads
   Resource at user addr: cheaper writes per user

8. Minimize event data (only essential fields)

Typical gas costs (approximate):
  Simple transfer: ~200 gas
  Swap with fee: ~500 gas  
  Complex DeFi op: ~1000-3000 gas
  NFT mint: ~500-800 gas
```

---

## สรุป Production Readiness

```
Level 1 - Basic:
  ✓ Tests pass
  ✓ Deploy script ready
  ✓ Admin keys secured

Level 2 - Production Ready:
  ✓ Formal verification (Move Prover)
  ✓ Security audit
  ✓ Monitoring setup
  ✓ Emergency procedures
  ✓ Multisig admin

Level 3 - Battle-hardened:
  ✓ Bug bounty program (6+ months)
  ✓ Multiple audits
  ✓ Insurance coverage
  ✓ Incident response drill
  ✓ Gradual rollout with TVL caps
```

---

**ก่อนหน้า**: [Part 39 - Move Prover ←](part-39-move-prover.md)
**ต่อไป**: [Part 41 - Professional: Performance Optimization →](../professional/part-41-performance.md)
