# Part 89: Gas Optimization Techniques

## สารบัญ
- [Gas Model on Aptos & Sui](#gas-model-on-aptos--sui)
- [Storage Cost Optimization](#storage-cost-optimization)
- [Computation Optimization](#computation-optimization)
- [Batch Operations](#batch-operations)
- [Profiling & Benchmarking](#profiling--benchmarking)
- [Anti-Patterns to Avoid](#anti-patterns-to-avoid)

---

## Gas Model on Aptos & Sui

```
APTOS GAS MODEL
  
  Total Gas = Intrinsic Gas + Execution Gas + IO Gas + Storage Gas
  
  Intrinsic Gas:     ~100 gas (transaction overhead)
  Execution Gas:     Opcodes × gas_per_opcode
  IO Gas:            Read/Write × gas_per_byte
  Storage Gas:       Bytes allocated × price_per_byte (ongoing)
  
  Key Insight:
  - Storage is the most expensive in the long run
  - Every byte stored on-chain costs APT continuously
  - Read is cheaper than write
  - Emit events costs gas (avoid redundant events)

  GAS UNITS vs GAS PRICE
  - Gas units: amount of computation
  - Gas price: APT per gas unit (market-driven)
  - Transaction fee = gas_units × gas_price
  
  APTOS MAX GAS (default): 2,000,000 units
  Set higher with max_gas_amount in transaction


SUI GAS MODEL

  Total Gas = Computation + Storage
  
  Computation:    Proportional to instructions executed
  Storage:        Per-byte storage rebate system
  
  KEY DIFFERENCE FROM APTOS:
  - Sui gives storage rebates when you delete objects
  - Incentivizes cleanup of on-chain state
  - Objects deleted → gas refunded to sender
  
  Storage Cost = size_in_bytes × STORAGE_FUND_REWARD_RATE
  Storage Rebate = original_storage_cost × rebate_rate (95%)
  
  Net Cost = New_storage - Old_storage_rebate
```

---

## Storage Cost Optimization

```move
// ============================================
// STORAGE OPTIMIZATION TECHNIQUES
// ============================================

// ❌ BAD: Storing redundant data
module bad::pool {
    struct Pool has key {
        reserve_x: u64,
        reserve_y: u64,
        total_lp: u64,
        k_value: u64,         // ❌ Redundant: can be computed from reserve_x * reserve_y
        price_x: u64,         // ❌ Redundant: can be computed from reserves
        price_y: u64,         // ❌ Redundant
        last_update: u64,
    }
}

// ✅ GOOD: Only store what can't be derived
module good::pool {
    struct Pool has key {
        reserve_x: u64,
        reserve_y: u64,
        total_lp: u64,
        fee_bps: u64,
        // k, prices computed on-the-fly (view functions)
    }
    
    #[view]
    public fun get_k(pool: &Pool): u128 {
        (pool.reserve_x as u128) * (pool.reserve_y as u128)
    }
    
    #[view]
    public fun get_price_x_in_y(pool: &Pool): u128 {
        (pool.reserve_y as u128) * 1_000_000_000 / (pool.reserve_x as u128)
    }
}

// ============================================
// PACK DATA INTO FEWER SLOTS (Aptos/Move)
// ============================================

// ❌ BAD: Many separate fields, all 64 bits each
struct UserPositionBad has store {
    is_long: bool,        // Only needs 1 bit, wastes 63
    leverage: u8,         // 0-10, wastes 56 bits
    status: u8,           // 0-3, wastes 62 bits
    collateral: u64,
    position_size: u64,
    entry_price: u64,
    timestamp: u64,
}
// Storage: 7 fields = 7 × 8 bytes = 56 bytes

// ✅ GOOD: Pack small fields into a single u64
struct UserPositionGood has store {
    // Packed flags: | is_long (1) | leverage (4) | status (2) | padding (57) |
    packed_flags: u64,
    collateral: u64,
    position_size: u64,
    entry_price: u64,
    timestamp: u64,
}
// Storage: 5 fields = 40 bytes (28% reduction)

module packed_position {
    const IS_LONG_MASK: u64 = 1;
    const LEVERAGE_SHIFT: u8 = 1;
    const LEVERAGE_MASK: u64 = 0xF << 1;     // 4 bits
    const STATUS_SHIFT: u8 = 5;
    const STATUS_MASK: u64 = 0x3 << 5;       // 2 bits
    
    public fun pack_flags(is_long: bool, leverage: u8, status: u8): u64 {
        let flags = if (is_long) 1u64 else 0u64;
        flags = flags | ((leverage as u64) << 1);
        flags = flags | ((status as u64) << 5);
        flags
    }
    
    public fun unpack_is_long(flags: u64): bool { flags & IS_LONG_MASK != 0 }
    public fun unpack_leverage(flags: u64): u8 { ((flags & LEVERAGE_MASK) >> 1) as u8 }
    public fun unpack_status(flags: u64): u8 { ((flags & STATUS_MASK) >> 5) as u8 }
}

// ============================================
// AVOID OVER-INDEXING WITH TABLES
// ============================================

// ❌ BAD: Separate Table for each small config
struct ProtocolConfig has key {
    fee_table: Table<vector<u8>, u64>,  // 10 entries × overhead
    limit_table: Table<vector<u8>, u64>,
    flag_table: Table<vector<u8>, bool>,
}

// ✅ GOOD: Flat struct for static config
struct ProtocolConfig has key {
    fee_swap_bps: u64,
    fee_flash_bps: u64,
    max_slippage_bps: u64,
    max_leverage: u8,
    paused: bool,
    emergency_admin: address,
}

// ============================================
// SUI: DELETE OBJECTS TO GET REBATES
// ============================================

// ✅ SUI PATTERN: Clean up temporary objects
module sui_gas_opt {
    use sui::object::{Self, UID};
    use sui::tx_context::TxContext;
    
    struct TempReceipt has key {
        id: UID,
        amount: u64,
        expiry: u64,
    }
    
    // After consuming a receipt, delete it → storage rebate!
    public fun consume_and_delete_receipt(receipt: TempReceipt) {
        let TempReceipt { id, amount: _, expiry: _ } = receipt;
        object::delete(id);  // Returns storage gas to sender
    }
    
    // Merge small objects to reduce object count
    public fun merge_coins(
        target: &mut sui::coin::Coin<sui::sui::SUI>,
        source: sui::coin::Coin<sui::sui::SUI>,
    ) {
        sui::coin::join(target, source);  // source deleted, rebate issued
    }
}
```

---

## Computation Optimization

```move
// ============================================
// MATHEMATICAL COMPUTATION OPTIMIZATIONS
// ============================================

module math_opt {
    
    // ❌ BAD: Division before multiplication (loses precision AND slow)
    fun fee_amount_bad(amount: u64, fee_bps: u64): u64 {
        amount / 10_000 * fee_bps  // Division first = precision loss
    }
    
    // ✅ GOOD: Multiply first, then divide
    fun fee_amount_good(amount: u64, fee_bps: u64): u64 {
        (amount as u128 * fee_bps as u128 / 10_000) as u64  // Full precision
    }
    
    // ❌ BAD: Using pow() for squaring
    fun square_bad(x: u64): u128 {
        let x128 = x as u128;
        // pow() is expensive, sequential multiplications
        x128 * x128 * x128 * x128  // ← This is fine actually
    }
    
    // ✅ GOOD: Bit shifts for powers of 2
    fun mul_by_power_of_two(x: u64, power: u8): u64 {
        x << power  // 2^power times faster than multiplication
    }
    
    fun div_by_power_of_two(x: u64, power: u8): u64 {
        x >> power  // Integer division by 2^power
    }
    
    // ❌ BAD: Nested loops O(n²)
    fun sum_products_bad(a: &vector<u64>, b: &vector<u64>): u128 {
        let result = 0u128;
        let n = vector::length(a);
        let i = 0;
        while (i < n) {
            let j = 0;
            while (j < n) {
                result = result + (*vector::borrow(a, i) as u128) * (*vector::borrow(b, j) as u128);
                j = j + 1;
            };
            i = i + 1;
        };
        result
    }
    
    // ✅ GOOD: Precompute sums, reduce to O(n)
    fun sum_products_good(a: &vector<u64>, b: &vector<u64>): u128 {
        let sum_a = 0u128;
        let sum_b = 0u128;
        let n = vector::length(a);
        let i = 0;
        while (i < n) {
            sum_a = sum_a + (*vector::borrow(a, i) as u128);
            sum_b = sum_b + (*vector::borrow(b, i) as u128);
            i = i + 1;
        };
        sum_a * sum_b  // = (Σa)(Σb) — different semantics but illustrates optimization
    }
    
    // ✅ GOOD: Integer square root (Babylonian method)
    fun isqrt(n: u128): u128 {
        if (n == 0) return 0;
        if (n < 4) return 1;
        
        // Initial guess
        let x = n;
        let y = (x + 1) / 2;
        
        while (y < x) {
            x = y;
            y = (x + n / x) / 2;
        };
        x
    }
    
    // ✅ GOOD: Precomputed table for small values (avoid function call overhead)
    fun get_bps_multiplier(leverage: u8): u64 {
        // Instead of computing 10_000 / leverage each time
        if (leverage == 1) return 10_000;
        if (leverage == 2) return 5_000;
        if (leverage == 5) return 2_000;
        if (leverage == 10) return 1_000;
        10_000 / (leverage as u64)  // Fallback
    }
    
    // ============================================
    // EARLY TERMINATION OPTIMIZATIONS
    // ============================================
    
    // ✅ GOOD: Short-circuit expensive checks
    fun validate_swap(
        amount: u64,
        reserve: u64,
        paused: bool,
        emergency: bool,
    ) {
        // Check cheapest (bool flag) first
        assert!(!emergency, 1);       // Cheapest: bool check
        assert!(!paused, 2);          // Still cheap: bool check
        assert!(amount > 0, 3);       // Integer compare
        assert!(reserve > 0, 4);      // Integer compare
        // More expensive checks last
        assert!(amount < reserve / 2, 5);  // Division (avoid if possible)
    }
}

// ============================================
// LOOP OPTIMIZATION PATTERNS
// ============================================

module loop_opt {
    // ❌ BAD: Calling vector::length() in every iteration
    fun sum_bad(v: &vector<u64>): u64 {
        let total = 0u64;
        let i = 0;
        while (i < vector::length(v)) {  // ❌ length() called every iteration
            total = total + *vector::borrow(v, i);
            i = i + 1;
        };
        total
    }
    
    // ✅ GOOD: Cache length before loop
    fun sum_good(v: &vector<u64>): u64 {
        let total = 0u64;
        let n = vector::length(v);  // ✅ Computed once
        let i = 0;
        while (i < n) {
            total = total + *vector::borrow(v, i);
            i = i + 1;
        };
        total
    }
    
    // ✅ GOOD: Avoid unnecessary borrows in tight loops
    fun find_max(v: &vector<u64>): u64 {
        assert!(vector::length(v) > 0, 0);
        let max = *vector::borrow(v, 0);
        let n = vector::length(v);
        let i = 1;
        while (i < n) {
            let val = *vector::borrow(v, i);
            if (val > max) { max = val; };
            i = i + 1;
        };
        max
    }
}
```

---

## Batch Operations

```move
// ============================================
// BATCH OPERATIONS: Amortize fixed costs
// ============================================

module batch_ops {
    use aptos_framework::coin;
    
    struct BatchSwapParams has drop {
        amount_in: u64,
        min_amount_out: u64,
        is_x_to_y: bool,
    }
    
    // ❌ BAD: N separate transactions, N × overhead
    // User calls swap() 100 times = 100 transactions
    
    // ✅ GOOD: Batch in single transaction
    public entry fun batch_swap(
        user: &signer,
        swaps: vector<BatchSwapParams>,
    ) {
        let n = vector::length(&swaps);
        assert!(n <= 20, 100);  // Limit batch size for gas safety
        
        let i = 0;
        while (i < n) {
            let params = vector::borrow(&swaps, i);
            // Execute each swap
            execute_single_swap(user, params.amount_in, params.min_amount_out, params.is_x_to_y);
            i = i + 1;
        };
    }
    
    fun execute_single_swap(
        user: &signer,
        amount_in: u64,
        min_amount_out: u64,
        is_x_to_y: bool,
    ) {
        // Swap implementation
        let _ = (user, amount_in, min_amount_out, is_x_to_y);
    }
    
    // ============================================
    // BULK MINT NFT (single transaction)
    // ============================================
    
    struct NFTBatch has key {
        minted_count: u64,
        max_supply: u64,
    }
    
    public entry fun bulk_mint(
        admin: &signer,
        batch: &mut NFTBatch,
        recipients: vector<address>,
        count_each: u64,
    ) {
        let n = vector::length(&recipients);
        let total = n * (count_each as u64);
        
        assert!(batch.minted_count + total <= batch.max_supply, 1);
        
        let i = 0;
        while (i < n) {
            let recipient = *vector::borrow(&recipients, i);
            
            let j = 0;
            while (j < count_each) {
                // mint_one(admin, recipient, batch.minted_count + i * count_each + j);
                j = j + 1;
            };
            
            i = i + 1;
        };
        
        batch.minted_count = batch.minted_count + total;
    }
    
    // ============================================
    // LAZY EVALUATION: Defer computation
    // ============================================
    
    struct PendingRewards has key {
        user: address,
        // Store inputs, not computed result
        staked_amount: u64,
        start_epoch: u64,
        // Compute actual rewards only when claiming
    }
    
    // ❌ BAD: Update rewards for ALL users every epoch
    // Gas: O(n) where n = number of stakers
    
    // ✅ GOOD: Lazy claim - compute only when user requests
    public fun claim_rewards(
        rewards: &mut PendingRewards,
        current_epoch: u64,
        emission_per_epoch: u64,
    ): u64 {
        let epochs_staked = current_epoch - rewards.start_epoch;
        let earned = rewards.staked_amount * emission_per_epoch * epochs_staked / 1_000_000;
        
        // Reset checkpoint
        rewards.start_epoch = current_epoch;
        
        earned
    }
}
```

---

## Profiling & Benchmarking

```bash
#!/bin/bash
# Gas profiling script for Move modules

# 1. Run tests with gas measurement
aptos move test \
  --package-dir . \
  --gas-coverage \
  --named-addresses protocol=0x1

# 2. Simulate transaction to check gas usage
aptos move run \
  --function-id "0x1::pool::swap" \
  --args "u64:1000000" "bool:true" "u64:900000" \
  --gas-unit-price 100 \
  --max-gas 100000 \
  --simulate  # Dry run, don't submit

# 3. Profile specific function gas
aptos move run \
  --function-id "0x1::pool::add_liquidity" \
  --args "u64:1000000" "u64:4000000" \
  --profile-gas  # Extended gas profiling output

# Expected output format:
# Execution gas: 1250
# IO gas: 340
# Storage gas: 890
# Total: 2480 gas units
# At 100 gas/unit = 0.000248 APT
```

```typescript
// ============================================
// GAS BENCHMARKING SCRIPT
// Compare gas costs between implementations
// ============================================

import { AptosClient, AptosAccount, TxnBuilderTypes } from 'aptos';

class GasBenchmark {
  private client: AptosClient;
  private account: AptosAccount;
  
  constructor(nodeUrl: string, privateKey: string) {
    this.client = new AptosClient(nodeUrl);
    this.account = new AptosAccount(Buffer.from(privateKey, 'hex'));
  }
  
  async measureGas(
    functionId: string,
    args: any[],
    typeArgs: string[] = [],
  ): Promise<{ gasUsed: number; gasUnitPrice: number; totalCost: number }> {
    const payload = {
      function: functionId,
      type_arguments: typeArgs,
      arguments: args,
    };
    
    const txn = await this.client.generateTransaction(
      this.account.address().toString(),
      payload,
      { max_gas_amount: '100000' },
    );
    
    // Simulate (don't submit)
    const simulation = await this.client.simulateTransaction(this.account, txn);
    
    const gasUsed = parseInt(simulation[0].gas_used);
    const gasUnitPrice = parseInt(simulation[0].gas_unit_price);
    
    return {
      gasUsed,
      gasUnitPrice,
      totalCost: gasUsed * gasUnitPrice,
    };
  }
  
  async compareImplementations(cases: Array<{
    name: string;
    functionId: string;
    args: any[];
  }>): Promise<void> {
    console.log('Gas Benchmark Results:');
    console.log('='.repeat(60));
    
    const results = await Promise.all(
      cases.map(async c => ({
        ...c,
        ...(await this.measureGas(c.functionId, c.args)),
      }))
    );
    
    // Sort by gas used
    results.sort((a, b) => a.gasUsed - b.gasUsed);
    
    const baseline = results[0].gasUsed;
    
    results.forEach(r => {
      const overhead = ((r.gasUsed - baseline) / baseline * 100).toFixed(1);
      const sign = r.gasUsed > baseline ? '+' : '';
      console.log(`${r.name.padEnd(30)} ${r.gasUsed.toString().padStart(8)} gas  ${sign}${overhead}%`);
    });
    
    console.log('='.repeat(60));
    console.log(`Best: ${results[0].name} (${baseline} gas)`);
  }
}

// Usage example
async function runBenchmarks() {
  const bench = new GasBenchmark(
    'https://fullnode.testnet.aptoslabs.com/v1',
    process.env.BENCH_PRIVATE_KEY!,
  );
  
  await bench.compareImplementations([
    {
      name: 'Swap v1 (basic)',
      functionId: '0xADDR::pool_v1::swap',
      args: ['1000000', true, '900000'],
    },
    {
      name: 'Swap v2 (optimized)',
      functionId: '0xADDR::pool_v2::swap',
      args: ['1000000', true, '900000'],
    },
  ]);
}
```

---

## Anti-Patterns to Avoid

```move
// ============================================
// GAS ANTI-PATTERNS AND THEIR FIXES
// ============================================

module antipatterns {
    
    // ============================================
    // ANTI-PATTERN 1: Unbounded loops
    // ============================================
    
    // ❌ BAD: Loop over ALL users (gas bomb)
    public fun distribute_rewards_bad(
        pool: &mut Pool,
        users: &vector<address>,
    ) {
        let n = vector::length(users);  // Could be 10,000!
        let i = 0;
        while (i < n) {
            // process_user(pool, *vector::borrow(users, i));
            i = i + 1;
        };
        // Gas: O(n) - could hit gas limit
    }
    
    // ✅ GOOD: Lazy/pull-based rewards
    public fun claim_reward_lazy(user: &signer, pool: &Pool) {
        // Calculate only this user's rewards
        // Gas: O(1)
    }
    
    // ============================================
    // ANTI-PATTERN 2: Storing history on-chain
    // ============================================
    
    // ❌ BAD: Unlimited history vector
    struct PriceHistory has key {
        prices: vector<u64>,  // Grows forever! Storage bomb
    }
    
    // ✅ GOOD: Fixed-size ring buffer
    struct PriceHistory has key {
        prices: vector<u64>,   // Fixed 24 slots
        head: u64,             // Oldest entry index
    }
    
    public fun push_price(history: &mut PriceHistory, price: u64) {
        assert!(vector::length(&history.prices) == 24, 0);
        *vector::borrow_mut(&mut history.prices, (history.head % 24) as u64) = price;
        history.head = history.head + 1;
    }
    
    // ============================================
    // ANTI-PATTERN 3: Unnecessary event emission
    // ============================================
    
    // ❌ BAD: Events in every sub-step
    public fun complex_op(pool: &mut Pool, amount: u64) {
        // emit!(StepStartEvent { amount });     // ❌ unnecessary
        let fee = calculate_fee(amount);
        // emit!(FeeCalculatedEvent { fee });    // ❌ unnecessary
        let out = calculate_output(amount, fee);
        // emit!(OutputCalculatedEvent { out }); // ❌ unnecessary
        apply_swap(pool, amount, out);
        emit!(SwapEvent { amount, fee, out });   // ✅ one event with all data
    }
    
    // ============================================
    // ANTI-PATTERN 4: Redundant reads
    // ============================================
    
    // ❌ BAD: Read same resource multiple times
    public fun update_pool_bad(pool_addr: address, amount: u64) {
        let pool = borrow_global<Pool>(pool_addr);  // Read 1
        let fee = pool.fee_bps;
        
        let pool_again = borrow_global<Pool>(pool_addr);  // Read 2 (redundant!)
        let reserve = pool_again.reserve_x;
        
        let pool_mut = borrow_global_mut<Pool>(pool_addr);  // Read 3 (write)
        pool_mut.reserve_x = reserve + amount;
    }
    
    // ✅ GOOD: Single mutable borrow
    public fun update_pool_good(pool_addr: address, amount: u64) {
        let pool = borrow_global_mut<Pool>(pool_addr);  // Single read
        let fee = pool.fee_bps;
        let reserve = pool.reserve_x;
        pool.reserve_x = reserve + amount;
    }
    
    // ============================================
    // ANTI-PATTERN 5: Large function arguments
    // ============================================
    
    // ❌ BAD: Pass entire struct when only one field needed
    public fun check_fee_bad(config: ProtocolConfig): bool {
        config.fee_bps > 0
    }
    
    // ✅ GOOD: Pass only what's needed (or use &Config for references)
    public fun check_fee_good(config: &ProtocolConfig): bool {
        config.fee_bps > 0
    }
    
    struct ProtocolConfig has key, drop { fee_bps: u64 }
    struct Pool has key { fee_bps: u64, reserve_x: u64 }
    
    fun calculate_fee(amount: u64): u64 { amount * 30 / 10000 }
    fun calculate_output(amount: u64, _fee: u64): u64 { amount * 9970 / 10000 }
    fun apply_swap(_pool: &mut Pool, _amount: u64, _out: u64) {}
}
```

---

## สรุป Gas Optimization

```
GAS OPTIMIZATION PRIORITY (highest impact first)

1. STORAGE REDUCTION (biggest impact)
   - Remove redundant fields (compute from others)
   - Pack small fields into single u64
   - Use ring buffers instead of growing vectors
   - Delete Sui objects when done (rebate!)
   - Use lazy computation instead of precomputing

2. LOOP OPTIMIZATION
   - Avoid O(n) operations with n = user count
   - Use pull-based (lazy) patterns for rewards
   - Cache vector::length() before loops
   - Bound all loop lengths with assertions

3. COMPUTATION TRICKS
   - Multiply before divide (preserves precision)
   - Bit shifts for powers of 2
   - Short-circuit: cheap checks before expensive
   - Precompute lookup tables for repeated values

4. BATCHING
   - Single transaction for N operations
   - Amortize fixed overhead across more work
   - But: bound batch size to avoid gas limit

5. I/O REDUCTION
   - Single borrow_global_mut instead of read+write
   - Avoid reading same resource multiple times
   - Use references (&T) instead of copying structs

6. EVENTS
   - One event per operation with all data
   - Don't emit progress events in sub-steps
   - Events have non-zero gas cost!

TOOL FOR MEASUREMENT
  aptos move run --simulate     → See gas before paying
  aptos move test --gas-coverage → Per-function breakdown
  TypeScript simulate() loop    → Compare implementations
  
TARGETS (rough guidelines)
  Simple view call: 100-500 gas
  Basic transfer: 500-2000 gas
  AMM swap: 3000-8000 gas
  Complex DeFi operation: 10,000-50,000 gas
```

---

**ก่อนหน้า**: [Part 88 - DeFi Indexing & Analytics ←](part-88-indexing.md)
**ต่อไป**: [Part 90 - Advanced Protocol Architecture →](part-90-protocol-architecture.md)
