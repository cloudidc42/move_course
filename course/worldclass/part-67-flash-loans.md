# Part 67: Flash Loan Protocols

## สารบัญ
- [Flash Loan Fundamentals](#flash-loan-fundamentals)
- [Full Flash Loan Protocol](#full-flash-loan-protocol)
- [Flash Loan Use Cases](#flash-loan-use-cases)
- [Arbitrage Bot Implementation](#arbitrage-bot-implementation)
- [Collateral Swap Pattern](#collateral-swap-pattern)
- [Flash Loan Security](#flash-loan-security)

---

## Flash Loan Fundamentals

```
Flash Loans: Uncollateralized loans repaid in same transaction

How It Works:
  1. User calls flashLoan(1,000,000 USDC)
  2. Contract lends 1M USDC to user
  3. User does stuff (arbitrage, liquidation, etc.)
  4. User calls repay(1,000,000 USDC + 9 USDC fee)
  5. If step 4 fails → ENTIRE TRANSACTION REVERTS
     (User gets nothing, they just wasted gas)
  
Why This Is Safe:
  - Atomic transaction: all-or-nothing
  - If loan not repaid → revert → nothing happened
  - Lender always gets principal + fee, or nothing
  - No credit risk, no collateral needed
  
Revenue:
  - Aave: 0.09% fee → $100M daily volume = $90K/day
  - dYdX: 0% fee (used for trade volume)
  - Uniswap V2: 0.3% fee (same as trade)
  
Move-specific advantage:
  - LinearType Receipt: Must be consumed (no forgetting to repay)
  - Type system prevents silently dropping the loan obligation
```

---

## Full Flash Loan Protocol

```move
module flash::lending_pool {
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::timestamp;
    use aptos_std::event;
    
    // ============================================
    // COMPLETE FLASH LOAN PROTOCOL
    //
    // Features:
    // - Multiple asset support
    // - Fee collection
    // - Receiver interface
    // - Nested flash loans
    // ============================================
    
    // Flash loan fee: 9 basis points (0.09%)
    const FLASH_FEE_BPS: u64 = 9;
    const BPS_BASE: u64 = 10_000;
    
    const E_INSUFFICIENT_FUNDS: u64 = 1;
    const E_REPAYMENT_INSUFFICIENT: u64 = 2;
    const E_POOL_EMPTY: u64 = 3;
    const E_ZERO_AMOUNT: u64 = 4;
    
    // ============================================
    // POOL RESOURCE (per asset type)
    // ============================================
    
    struct Pool<phantom T> has key {
        reserves: Coin<T>,
        total_flash_volume: u128,
        total_fee_earned: u128,
        utilization_count: u64,
    }
    
    // ============================================
    // FLASH LOAN RECEIPT (must be consumed)
    // Linear type: no `drop` ability
    // ============================================
    
    struct FlashLoan<phantom T> {
        pool_addr: address,
        amount_borrowed: u64,
        fee_amount: u64,
        borrow_time: u64,
    }
    
    // ============================================
    // EVENTS
    // ============================================
    
    #[event]
    struct FlashLoanEvent has drop, store {
        borrower: address,
        token_type: std::string::String,
        amount: u64,
        fee: u64,
    }
    
    // ============================================
    // POOL MANAGEMENT
    // ============================================
    
    // Create a new flash loan pool for token T
    public fun create_pool<T>(admin: &signer) {
        move_to(admin, Pool<T> {
            reserves: coin::zero<T>(),
            total_flash_volume: 0,
            total_fee_earned: 0,
            utilization_count: 0,
        });
    }
    
    // Add liquidity to pool (earn flash loan fees)
    public fun deposit<T>(
        provider: &signer,
        pool_addr: address,
        amount: u64,
    ) acquires Pool {
        let coins = coin::withdraw<T>(provider, amount);
        let pool = borrow_global_mut<Pool<T>>(pool_addr);
        coin::merge(&mut pool.reserves, coins);
    }
    
    // Withdraw liquidity
    public fun withdraw<T>(
        provider: &signer,
        pool_addr: address,
        amount: u64,
    ) acquires Pool {
        let pool = borrow_global_mut<Pool<T>>(pool_addr);
        let coins = coin::extract(&mut pool.reserves, amount);
        coin::deposit(std::signer::address_of(provider), coins);
    }
    
    // ============================================
    // FLASH LOAN CORE
    // ============================================
    
    // Borrow tokens for flash loan
    // Returns: (borrowed_coins, receipt_that_must_be_repaid)
    public fun flash_borrow<T>(
        borrower: &signer,
        pool_addr: address,
        amount: u64,
    ): (Coin<T>, FlashLoan<T>) acquires Pool {
        assert!(amount > 0, E_ZERO_AMOUNT);
        
        let pool = borrow_global_mut<Pool<T>>(pool_addr);
        let pool_balance = coin::value(&pool.reserves);
        
        assert!(pool_balance >= amount, E_INSUFFICIENT_FUNDS);
        
        // Calculate fee (minimum 1 unit)
        let fee = max(1, amount * FLASH_FEE_BPS / BPS_BASE);
        
        // Extract coins from pool
        let borrowed = coin::extract(&mut pool.reserves, amount);
        
        // Update stats
        pool.total_flash_volume = pool.total_flash_volume + (amount as u128);
        pool.utilization_count = pool.utilization_count + 1;
        
        // Emit event
        event::emit(FlashLoanEvent {
            borrower: std::signer::address_of(borrower),
            token_type: std::string::utf8(b"T"),
            amount,
            fee,
        });
        
        let receipt = FlashLoan<T> {
            pool_addr,
            amount_borrowed: amount,
            fee_amount: fee,
            borrow_time: timestamp::now_microseconds(),
        };
        
        (borrowed, receipt)
    }
    
    // Repay flash loan (must be called, receipt consumed)
    public fun flash_repay<T>(
        repayment: Coin<T>,
        fee_payment: Coin<T>,
        receipt: FlashLoan<T>,
    ) acquires Pool {
        let FlashLoan { pool_addr, amount_borrowed, fee_amount, borrow_time: _ } = receipt;
        
        // Verify amounts
        assert!(coin::value(&repayment) == amount_borrowed, E_REPAYMENT_INSUFFICIENT);
        assert!(coin::value(&fee_payment) >= fee_amount, E_REPAYMENT_INSUFFICIENT);
        
        let pool = borrow_global_mut<Pool<T>>(pool_addr);
        
        // Deposit repayment + fee
        coin::merge(&mut pool.reserves, repayment);
        let fee_value = coin::value(&fee_payment);
        coin::merge(&mut pool.reserves, fee_payment);
        
        pool.total_fee_earned = pool.total_fee_earned + (fee_value as u128);
    }
    
    // View functions
    public fun pool_liquidity<T>(pool_addr: address): u64 acquires Pool {
        coin::value(&borrow_global<Pool<T>>(pool_addr).reserves)
    }
    
    public fun flash_fee<T>(pool_addr: address, amount: u64): u64 acquires Pool {
        let _ = pool_addr;
        max(1, amount * FLASH_FEE_BPS / BPS_BASE)
    }
    
    fun max(a: u64, b: u64): u64 { if (a > b) a else b }
}
```

---

## Flash Loan Use Cases

```move
// ============================================
// USE CASE 1: DEX ARBITRAGE
// Buy low on DEX A, sell high on DEX B
// ============================================

module flash::arbitrage {
    use flash::lending_pool::{Self, FlashLoan};
    use aptos_framework::coin::Coin;
    
    public entry fun arbitrage_aptos_usdc(
        user: &signer,
        pool_addr: address,
        borrow_amount: u64,     // e.g., 100,000 USDC
        min_profit: u64,        // Minimum profit to proceed
        dex_a_addr: address,    // Buying DEX (cheaper price)
        dex_b_addr: address,    // Selling DEX (higher price)
    ) acquires flash::lending_pool::Pool {
        // Step 1: Borrow USDC
        let (usdc, receipt) = lending_pool::flash_borrow<USDC>(user, pool_addr, borrow_amount);
        
        // Step 2: Buy APT on DEX A (e.g., Liquidswap)
        let apt = swap_usdc_to_apt(usdc, dex_a_addr);
        let apt_amount = aptos_framework::coin::value(&apt);
        
        // Step 3: Sell APT on DEX B (e.g., AuxExchange)
        let usdc_out = swap_apt_to_usdc(apt, dex_b_addr);
        let usdc_amount_out = aptos_framework::coin::value(&usdc_out);
        
        // Step 4: Calculate profit
        let fee = lending_pool::flash_fee<USDC>(pool_addr, borrow_amount);
        let total_cost = borrow_amount + fee;
        
        assert!(usdc_amount_out > total_cost + min_profit, 1); // Must be profitable
        
        // Step 5: Repay loan
        let repayment = aptos_framework::coin::extract(&mut usdc_out, borrow_amount);
        let fee_payment = aptos_framework::coin::extract(&mut usdc_out, fee);
        
        lending_pool::flash_repay(repayment, fee_payment, receipt);
        
        // Step 6: Keep profit
        let profit = aptos_framework::coin::value(&usdc_out);
        aptos_framework::coin::deposit(std::signer::address_of(user), usdc_out);
        
        // Emit profit event
    }
    
    fun swap_usdc_to_apt(_usdc: Coin<USDC>, _dex_addr: address): Coin<APT> {
        aptos_framework::coin::zero<APT>() // Placeholder
    }
    
    fun swap_apt_to_usdc(_apt: Coin<APT>, _dex_addr: address): Coin<USDC> {
        aptos_framework::coin::zero<USDC>() // Placeholder
    }
    
    struct USDC {}
    struct APT {}
}

// ============================================
// USE CASE 2: LIQUIDATION
// Liquidate underwater position, repay flash loan from proceeds
// ============================================

module flash::liquidation_bot {
    use flash::lending_pool;
    
    public entry fun flash_liquidate(
        liquidator: &signer,
        flash_pool_addr: address,
        lending_protocol_addr: address,
        debt_token: address,     // Token the borrower owes
        collateral_token: address, // Token we'll receive as reward
        borrower_to_liquidate: address,
        debt_to_repay: u64,
    ) {
        // Step 1: Borrow debt token via flash loan
        // let (debt_coins, receipt) = lending_pool::flash_borrow<DebtToken>(...);
        
        // Step 2: Repay borrower's debt, receive collateral + bonus
        // let collateral = lending_protocol::liquidate(borrower, debt_coins);
        
        // Step 3: Swap collateral back to debt token
        // let debt_repayment = dex::swap(collateral, debt_token);
        
        // Step 4: Repay flash loan (profit = collateral_value - debt - fee)
        // lending_pool::flash_repay(principal, fee, receipt);
        
        // Step 5: Keep profit
    }
}
```

---

## Collateral Swap Pattern

```move
// ============================================
// USE CASE 3: COLLATERAL SWAP
// Change collateral type without closing position
// 
// Before: ETH collateral, USDC borrowed
// After:  BTC collateral, USDC borrowed (same debt)
// ============================================

module flash::collateral_swap {
    use flash::lending_pool;
    
    // Swap from ETH collateral to BTC collateral
    public entry fun swap_collateral(
        user: &signer,
        flash_pool_addr: address,     // Flash loan source
        lending_protocol_addr: address,
        current_collateral_value: u64, // Value of current ETH collateral
        debt_amount: u64,              // Current USDC debt
        min_new_collateral: u64,       // Min BTC we expect to deposit
    ) {
        // The Problem:
        // User has Position: ETH collateral → USDC debt
        // User wants BTC collateral instead
        // But they can't withdraw ETH (it's backing their USDC debt)
        
        // Flash Loan Solution:
        // 1. Flash borrow USDC (= debt amount)
        // 2. Use USDC to repay existing USDC debt → ETH collateral freed
        // 3. Withdraw ETH collateral
        // 4. Swap ETH → BTC (via DEX)
        // 5. Deposit BTC as new collateral
        // 6. Borrow USDC again (same amount as before)
        // 7. Repay flash loan with borrowed USDC
        
        // Net result: ETH → BTC collateral, same USDC debt, paid flash loan fee
    }
}
```

---

## Flash Loan Security

```move
module flash::security {
    // ============================================
    // DEFENDING AGAINST FLASH LOAN ATTACKS
    //
    // Flash loans amplify economic attacks:
    // - Oracle manipulation (borrow huge, manipulate price, profit)
    // - Governance attacks (borrow voting tokens, vote, repay)
    // - Pool draining (complex exploits)
    // ============================================
    
    // DEFENSE 1: TWAP instead of spot prices
    // Already covered in Part 66 (oracle security)
    
    // DEFENSE 2: Snapshot voting (anti governance attack)
    // Snapshot at block N-1, not block N
    // Flash loan in same block can't affect past snapshot
    
    struct GovernanceVote has key {
        // Voting power based on balance at PREVIOUS block
        // Flash loans are same-tx (same block), so can't affect historical snapshot
        snapshot_block: u64,
    }
    
    // DEFENSE 3: Invariant checks
    // Check protocol state at start and end of complex operations
    
    struct ProtocolInvariant has key {
        total_supply: u64,
        total_collateral_value: u64,
        total_debt: u64,
        // Invariant: total_collateral_value * 150% >= total_debt (150% collateral ratio)
    }
    
    public fun check_invariant(protocol_addr: address) acquires ProtocolInvariant {
        let inv = borrow_global<ProtocolInvariant>(protocol_addr);
        // total_collateral * 150/100 >= total_debt
        // total_collateral * 3/2 >= total_debt
        // total_collateral * 3 >= total_debt * 2
        let weighted_collateral = inv.total_collateral_value * 3;
        let required = inv.total_debt * 2;
        assert!(weighted_collateral >= required, 1);
    }
    
    // DEFENSE 4: Price oracle with minimum liquidity requirement
    // Only use price if pool has sufficient liquidity
    // Flash-drained pools have temporarily low liquidity → price oracle rejects
    
    public fun get_price_with_liquidity_check(
        pool_addr: address,
        min_liquidity: u64,  // e.g., $1M
    ): u64 {
        let liquidity = 0u64; // get_pool_liquidity(pool_addr);
        assert!(liquidity >= min_liquidity, 1); // Pool drained → reject
        0 // get_spot_price(pool_addr)
    }
    
    // DEFENSE 5: Same-block protection
    // Prevent deposit and withdraw in same block (reentrancy via flash loan)
    
    struct LastAction has key {
        last_deposit_block: u64,
        last_withdraw_block: u64,
    }
    
    public fun check_same_block_protection(
        user_addr: address,
        is_withdrawal: bool,
    ) acquires LastAction {
        if (!exists<LastAction>(user_addr)) return;
        
        let action = borrow_global<LastAction>(user_addr);
        
        if (is_withdrawal) {
            // Can't withdraw in same block as deposit
            let current_block = aptos_framework::block::get_current_block_height();
            assert!(action.last_deposit_block < current_block, 1);
        };
    }
}
```

---

## สรุป Flash Loans

```
Flash Loan Summary:

PROTOCOL DESIGN CHECKLIST
  [ ] Linear type receipt (no `drop`) → forces repayment
  [ ] Verify principal + fee before accepting repayment
  [ ] Emit events for monitoring
  [ ] Pool liquidity gates (can't borrow > available)
  [ ] Min fee (at least 1 unit, no zero-fee flash loans)
  [ ] Pool utilization stats for analytics

ATTACK VECTORS TO PREVENT
  [ ] Oracle manipulation → use TWAP, min liquidity check
  [ ] Governance attack → snapshot at previous block
  [ ] Reentrancy → Move's acquires prevents same-module, use guard for cross-module
  [ ] Same-block attacks → track last action block
  [ ] Pool draining → invariant checks, circuit breakers

LEGITIMATE USE CASES
  Arbitrage:      Buy DEX-A, sell DEX-B (risk-free if profitable)
  Liquidation:    Liquidate positions without own capital
  Collateral swap: Change collateral type atomically
  Debt migration: Move debt from one protocol to another
  Self-liquidation: Borrow to repay self, save gas vs manual

ECONOMICS
  Fee revenue = volume × fee_bps / 10000
  LP return = fee_revenue / total_liquidity
  Typical fee: 0.05% to 0.3%
  Aave: $30M/day volume × 0.09% = $27K/day to LPs
  
APTOS vs SUI
  Aptos: Standard coin flash loans
  Sui: Use Balance<T> + receipt pattern (same concept)
       PTBs allow atomic flash operations naturally
```

---

**ก่อนหน้า**: [Part 66 - Advanced Security ←](part-66-advanced-security.md)
**ต่อไป**: [Part 68 - Insurance Protocols →](part-68-insurance-protocols.md)
