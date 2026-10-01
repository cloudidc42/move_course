# Part 53: Advanced Stablecoin Design

## สารบัญ
- [Stablecoin Taxonomy](#stablecoin-taxonomy)
- [CDP (Collateralized Debt Position)](#cdp-collateralized-debt-position)
- [Algorithmic Stablecoin Mechanics](#algorithmic-stablecoin-mechanics)
- [Curve-style StableSwap AMM](#curve-style-stableswap-amm)
- [Peg Stability Module](#peg-stability-module)
- [ตัวอย่าง: MoveUSD Protocol](#ตัวอย่าง-moveusd-protocol)

---

## Stablecoin Taxonomy

```
Stablecoin Types by Collateral:

1. FIAT-BACKED (Centralized)
   Examples: USDC, USDT
   Collateral: USD in bank account
   Risk: Censorship, bank failure, regulatory
   Peg: Maintained by redemption 1:1

2. CRYPTO-BACKED (CDP/Overcollateralized)
   Examples: DAI, LUSD
   Collateral: ETH, BTC, APT
   Overcollateralized: 150%+ collateral for 100% stablecoin
   Risk: Smart contract bugs, oracle failure
   Peg: Arbitrage + liquidation

3. ALGORITHMIC (No Collateral)
   Examples: FRAX (partial), UST (failed)
   Collateral: Nothing (or protocol token)
   Risk: Death spiral (lost peg → panic → lost peg)
   Most failed: Luna/UST, Iron Finance
   
4. HYBRID (Partially Backed)
   Examples: FRAX v2, RAI
   Mix of collateral + algorithm
   More stable than pure algo
   More capital efficient than pure CDP
   
5. LST-BACKED
   Examples: eUSD (backed by lstETH)
   Collateral: yield-bearing assets
   Benefit: Stablecoin ALSO earns yield
   
On Aptos:
  aphUSD (hypothetical): backed by APT
  CDP model: most secure for DeFi
  Need: TWAP oracle + liquidation system
```

---

## CDP (Collateralized Debt Position)

```move
module stablecoin::cdp {
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::timestamp;
    use aptos_framework::event;
    
    // ============================================
    // CDP System: Mint stablecoin against APT collateral
    // ============================================
    
    struct StableToken {}  // The stablecoin
    
    struct CDPSystem has key {
        admin: address,
        
        // Token capabilities
        mint_cap: coin::MintCapability<StableToken>,
        burn_cap: coin::BurnCapability<StableToken>,
        
        // System parameters
        min_collateral_ratio: u64,    // 150% = 15000 bps
        liquidation_ratio: u64,       // 130% = 13000 bps
        liquidation_bonus: u64,       // 10% = 1000 bps
        stability_fee_per_year: u64,  // Annual fee: 200 = 2%
        
        // Oracle
        oracle_addr: address,
        
        // System state
        total_debt: u64,     // Total stablecoin minted
        total_collateral: u64,  // Total APT locked
        
        // Emergency params
        debt_ceiling: u64,    // Max total stablecoin
        paused: bool,
    }
    
    struct Vault has key {
        owner: address,
        
        // Collateral locked (in APT)
        collateral: u64,
        
        // Debt owed (in stablecoin)
        debt: u64,
        
        // Last time stability fee was charged
        last_fee_time: u64,
        
        // Accumulated stability fees (compounding)
        accumulated_fee: u64,
    }
    
    #[event]
    struct VaultOpened has drop, store {
        owner: address,
        collateral: u64,
        debt: u64,
    }
    
    #[event]
    struct Liquidated has drop, store {
        vault_owner: address,
        liquidator: address,
        collateral_seized: u64,
        debt_repaid: u64,
    }
    
    const E_BELOW_MIN_RATIO: u64 = 1;
    const E_ABOVE_DEBT_CEILING: u64 = 2;
    const E_NOT_LIQUIDATABLE: u64 = 3;
    const E_PAUSED: u64 = 4;
    const E_VAULT_NOT_FOUND: u64 = 5;
    
    // ============================================
    // Open vault: deposit APT, mint stablecoin
    // ============================================
    
    public entry fun open_vault(
        user: &signer,
        system_addr: address,
        collateral_amount: u64,
        debt_amount: u64,
    ) acquires CDPSystem {
        let system = borrow_global_mut<CDPSystem>(system_addr);
        assert!(!system.paused, E_PAUSED);
        assert!(
            system.total_debt + debt_amount <= system.debt_ceiling,
            E_ABOVE_DEBT_CEILING
        );
        
        // Get APT price from oracle (in USD, 6 decimal precision)
        let apt_price = get_oracle_price(system.oracle_addr);
        
        // Check collateral ratio
        let collateral_usd = (collateral_amount as u128) * (apt_price as u128) / 1_000_000u128;
        let required_collateral = (debt_amount as u128) * (system.min_collateral_ratio as u128) / 10_000u128;
        assert!(collateral_usd >= required_collateral, E_BELOW_MIN_RATIO);
        
        let user_addr = std::signer::address_of(user);
        
        // Lock collateral
        let collateral = coin::withdraw<aptos_coin::AptosCoin>(user, collateral_amount);
        coin::deposit<aptos_coin::AptosCoin>(system_addr, collateral);
        
        // Create vault
        move_to(user, Vault {
            owner: user_addr,
            collateral: collateral_amount,
            debt: debt_amount,
            last_fee_time: timestamp::now_seconds(),
            accumulated_fee: 0,
        });
        
        // Mint stablecoin to user
        let stable_coins = coin::mint<StableToken>(debt_amount, &system.mint_cap);
        coin::deposit<StableToken>(user_addr, stable_coins);
        
        system.total_debt = system.total_debt + debt_amount;
        system.total_collateral = system.total_collateral + collateral_amount;
        
        event::emit(VaultOpened {
            owner: user_addr,
            collateral: collateral_amount,
            debt: debt_amount,
        });
    }
    
    // ============================================
    // Add collateral or repay debt
    // ============================================
    
    public entry fun add_collateral(
        user: &signer,
        system_addr: address,
        amount: u64,
    ) acquires Vault, CDPSystem {
        let user_addr = std::signer::address_of(user);
        assert!(exists<Vault>(user_addr), E_VAULT_NOT_FOUND);
        
        let vault = borrow_global_mut<Vault>(user_addr);
        let system = borrow_global_mut<CDPSystem>(system_addr);
        
        let collateral = coin::withdraw<aptos_coin::AptosCoin>(user, amount);
        coin::deposit<aptos_coin::AptosCoin>(system_addr, collateral);
        
        vault.collateral = vault.collateral + amount;
        system.total_collateral = system.total_collateral + amount;
    }
    
    public entry fun repay_debt(
        user: &signer,
        system_addr: address,
        repay_amount: u64,
    ) acquires Vault, CDPSystem {
        let user_addr = std::signer::address_of(user);
        let vault = borrow_global_mut<Vault>(user_addr);
        let system = borrow_global_mut<CDPSystem>(system_addr);
        
        // Accrue stability fee first
        let fee = accrue_stability_fee(vault, system.stability_fee_per_year);
        vault.accumulated_fee = vault.accumulated_fee + fee;
        vault.last_fee_time = timestamp::now_seconds();
        
        // Burn stablecoin
        let stable_coins = coin::withdraw<StableToken>(user, repay_amount);
        coin::burn(stable_coins, &system.burn_cap);
        
        vault.debt = vault.debt - repay_amount;
        system.total_debt = system.total_debt - repay_amount;
    }
    
    // ============================================
    // Liquidation
    // ============================================
    
    public entry fun liquidate(
        liquidator: &signer,
        system_addr: address,
        vault_owner: address,
        debt_to_repay: u64,
    ) acquires Vault, CDPSystem {
        let system = borrow_global_mut<CDPSystem>(system_addr);
        let vault = borrow_global_mut<Vault>(vault_owner);
        
        // Get current APT price
        let apt_price = get_oracle_price(system.oracle_addr);
        
        // Compute current collateral ratio
        let collateral_usd = (vault.collateral as u128) * (apt_price as u128) / 1_000_000u128;
        let current_ratio = (collateral_usd * 10_000u128 / (vault.debt as u128)) as u64;
        
        // Check if liquidatable
        assert!(current_ratio < system.liquidation_ratio, E_NOT_LIQUIDATABLE);
        
        // Calculate collateral to seize (with bonus)
        let collateral_per_stable = 1_000_000u64 * 10_000 / apt_price;  // APT per stablecoin
        let base_collateral = debt_to_repay * collateral_per_stable / 1_000_000;
        let bonus = base_collateral * system.liquidation_bonus / 10_000;
        let collateral_to_seize = base_collateral + bonus;
        
        let collateral_seized = std::math64::min(collateral_to_seize, vault.collateral);
        
        // Burn liquidator's stablecoins (repay debt)
        let stable_coins = coin::withdraw<StableToken>(liquidator, debt_to_repay);
        coin::burn(stable_coins, &system.burn_cap);
        
        // Send collateral to liquidator
        let apt_out = coin::withdraw<aptos_coin::AptosCoin>(
            &create_protocol_signer(system_addr),
            collateral_seized,
        );
        coin::deposit<aptos_coin::AptosCoin>(std::signer::address_of(liquidator), apt_out);
        
        // Update vault
        vault.debt = vault.debt - debt_to_repay;
        vault.collateral = vault.collateral - collateral_seized;
        system.total_debt = system.total_debt - debt_to_repay;
        system.total_collateral = system.total_collateral - collateral_seized;
        
        event::emit(Liquidated {
            vault_owner,
            liquidator: std::signer::address_of(liquidator),
            collateral_seized,
            debt_repaid: debt_to_repay,
        });
    }
    
    // ============================================
    // Stability fee calculation
    // ============================================
    
    fun accrue_stability_fee(vault: &Vault, annual_fee_bps: u64): u64 {
        let elapsed = timestamp::now_seconds() - vault.last_fee_time;
        let annual_seconds = 365u64 * 86_400;
        
        // fee = debt * rate * elapsed / annual_seconds
        let fee = (vault.debt as u128) * (annual_fee_bps as u128) 
                  * (elapsed as u128) / (annual_seconds as u128) / 10_000u128;
        fee as u64
    }
    
    // ============================================
    // View functions
    // ============================================
    
    #[view]
    public fun get_vault_ratio(
        vault_owner: address,
        system_addr: address,
    ): u64 acquires Vault, CDPSystem {
        let vault = borrow_global<Vault>(vault_owner);
        let system = borrow_global<CDPSystem>(system_addr);
        let apt_price = get_oracle_price(system.oracle_addr);
        
        if (vault.debt == 0) return 9_999_999;  // Infinite ratio
        
        let collateral_usd = (vault.collateral as u128) * (apt_price as u128) / 1_000_000u128;
        ((collateral_usd * 10_000u128 / (vault.debt as u128)) as u64)
    }
    
    #[view]
    public fun is_liquidatable(
        vault_owner: address,
        system_addr: address,
    ): bool acquires Vault, CDPSystem {
        let ratio = get_vault_ratio(vault_owner, system_addr);
        let system = borrow_global<CDPSystem>(system_addr);
        ratio < system.liquidation_ratio
    }
    
    fun get_oracle_price(_oracle: address): u64 { 2_000_000 }  // $2.00 placeholder
    native fun create_protocol_signer(addr: address): signer;
}
```

---

## Curve-style StableSwap AMM

```move
module stablecoin::stableswap {
    
    // ============================================
    // StableSwap: optimized for same-peg assets
    // Curve Finance formula
    // ============================================
    
    // StableSwap invariant: 
    // A * n^n * sum(x_i) + D = A * D * n^n + D^(n+1) / (n^n * prod(x_i))
    // where:
    //   A = amplification coefficient (higher A = flatter curve)
    //   D = total liquidity (invariant)
    //   n = number of tokens (usually 2 or 3)
    //   x_i = token reserves
    
    // For 2 tokens: A*2^2*(x+y) + D = A*D*2^2 + D^3/(4*x*y)
    
    const N_COINS: u64 = 2;
    const PRECISION: u128 = 1_000_000_000_000_000_000u128;  // 1e18
    
    struct StablePool has key {
        balances: vector<u64>,       // [balance_a, balance_b]
        amplification: u64,          // Amplification coefficient (e.g., 100)
        fee_bps: u64,                // e.g., 4 (0.04%)
        admin_fee_bps: u64,          // Fee that goes to protocol (e.g., 50% of fee)
        lp_supply: u64,
    }
    
    // Compute D (total liquidity invariant) via Newton's method
    public fun compute_d(balances: &vector<u64>, amp: u64): u64 {
        let n = N_COINS;
        let sum: u128 = {
            let mut s = 0u128;
            let mut i = 0u64;
            while (i < n) {
                s = s + (*std::vector::borrow(balances, i) as u128);
                i = i + 1;
            };
            s
        };
        
        if (sum == 0) return 0;
        
        let mut d = sum;
        let ann = (amp as u128) * (n as u128);
        
        // Newton's method: iterate to convergence
        let mut i = 0u64;
        while (i < 255) {
            let mut d_prev = d;
            
            // d_p = D^(n+1) / (n^n * prod(x_i))
            let mut d_p = d;
            let mut j = 0u64;
            while (j < n) {
                let x = *std::vector::borrow(balances, j);
                d_p = d_p * d / ((x as u128) * (n as u128) + 1u128);  // +1 to prevent div/0
                j = j + 1;
            };
            
            d_prev = d;
            // d = (ann * sum + d_p * n) * d / ((ann - 1) * d + (n + 1) * d_p)
            d = (ann * sum + d_p * (n as u128)) * d
                / ((ann - 1u128) * d + ((n + 1) as u128) * d_p);
            
            // Check convergence
            if (d > d_prev) {
                if (d - d_prev <= 1u128) break;
            } else {
                if (d_prev - d <= 1u128) break;
            };
            i = i + 1;
        };
        
        d as u64
    }
    
    // Compute y given x and D: solve for y in invariant
    // Used for swaps: given new x, find new y
    public fun compute_y(
        x_new: u64,    // New balance of token x after deposit
        balances: &vector<u64>,
        amp: u64,
        d: u64,
    ): u64 {
        let n = N_COINS;
        let ann = (amp as u128) * (n as u128);
        let d_128 = d as u128;
        
        // Compute sum and product excluding y (token index 1)
        let x_128 = x_new as u128;
        let s = x_128;  // Sum of known balances
        let c = d_128 * d_128 / (x_128 * (n as u128));
        
        let c = c * d_128 / (ann * (n as u128));
        let b = s + d_128 / ann;
        
        // Newton's method for y
        let mut y = d_128;
        let mut i = 0u64;
        while (i < 255) {
            let y_prev = y;
            y = (y * y + c) / (2u128 * y + b - d_128);
            
            if (y > y_prev) {
                if (y - y_prev <= 1u128) break;
            } else {
                if (y_prev - y <= 1u128) break;
            };
            i = i + 1;
        };
        
        y as u64
    }
    
    // Execute stable swap
    public fun get_dy(
        pool: &StablePool,
        i: u64,  // Token index in
        j: u64,  // Token index out
        dx: u64, // Amount in
    ): u64 {
        let n = N_COINS;
        let amp = pool.amplification;
        
        // Current D
        let d = compute_d(&pool.balances, amp);
        
        // New balance of token i
        let x_new = *std::vector::borrow(&pool.balances, i) + dx;
        
        // Compute new y (balance of token j)
        let y_new = compute_y(x_new, &pool.balances, amp, d);
        
        let y_old = *std::vector::borrow(&pool.balances, j);
        let dy = y_old - y_new;
        
        // Apply fee
        let fee = dy * pool.fee_bps / 10_000;
        
        dy - fee
    }
    
    // Get stableswap price impact (much lower than constant product near peg)
    #[view]
    public fun price_impact_bps(
        pool: &StablePool,
        amount_in: u64,   // USDC
    ): u64 {
        let spot_price = get_dy(pool, 0, 1, 1_000_000);  // Price for 1 USDC
        let execution_price = get_dy(pool, 0, 1, amount_in) * 1_000_000 / amount_in;
        
        if (spot_price > execution_price) {
            (spot_price - execution_price) * 10_000 / spot_price
        } else {
            0
        }
    }
}
```

---

## Peg Stability Module

```move
module stablecoin::psm {
    use aptos_framework::coin::{Self, Coin};
    
    // ============================================
    // PSM (Peg Stability Module)
    // Based on MakerDAO's PSM design
    // ============================================
    
    // Allows 1:1 swap between stablecoin and USDC
    // Maintains the peg through arbitrage
    
    struct PSM has key {
        admin: address,
        
        // USDC reserves (the backing)
        usdc_reserves: u64,
        
        // Fee for buying stablecoin with USDC (tin)
        buy_fee_bps: u64,  // e.g., 1 = 0.01%
        
        // Fee for selling stablecoin for USDC (tout)
        sell_fee_bps: u64,  // e.g., 1 = 0.01%
        
        // Maximum USDC that PSM holds
        debt_ceiling: u64,
        
        // Fee recipient
        fee_recipient: address,
        
        // Accumulated fees
        accumulated_fees: u64,
    }
    
    // Buy stablecoin with USDC (when stablecoin > $1.00)
    // User pays: amount * (1 + buy_fee_bps/10000) USDC
    // User gets: amount stablecoin
    public entry fun buy_stable<USDC>(
        user: &signer,
        psm_addr: address,
        stable_amount: u64,  // Amount of stablecoin to buy
    ) acquires PSM {
        let psm = borrow_global_mut<PSM>(psm_addr);
        
        let usdc_needed = stable_amount 
            + stable_amount * psm.buy_fee_bps / 10_000;
        
        assert!(psm.usdc_reserves + usdc_needed <= psm.debt_ceiling, 1);
        
        // Take USDC from user
        let usdc = coin::withdraw<USDC>(user, usdc_needed);
        let fee_amount = stable_amount * psm.buy_fee_bps / 10_000;
        
        psm.usdc_reserves = psm.usdc_reserves + usdc_needed;
        psm.accumulated_fees = psm.accumulated_fees + fee_amount;
        
        // Mint stablecoin
        // (mint_cap required - simplified)
        
        coin::deposit<USDC>(psm_addr, usdc);
    }
    
    // Sell stablecoin for USDC (when stablecoin < $1.00)
    // User pays: amount stablecoin  
    // User gets: amount * (1 - sell_fee_bps/10000) USDC
    public entry fun sell_stable<USDC>(
        user: &signer,
        psm_addr: address,
        stable_amount: u64,
    ) acquires PSM {
        let psm = borrow_global_mut<PSM>(psm_addr);
        
        let usdc_out = stable_amount 
            - stable_amount * psm.sell_fee_bps / 10_000;
        
        assert!(psm.usdc_reserves >= usdc_out, 2);
        
        // Burn stablecoin
        // (burn_cap required - simplified)
        
        // Send USDC to user
        psm.usdc_reserves = psm.usdc_reserves - usdc_out;
        
        // Transfer usdc_out from psm to user
    }
    
    // Peg arbitrage explanation:
    // If stablecoin trades at $1.01 on DEX:
    //   → Buy from PSM at $1.001 (1 + 0.01% fee)
    //   → Sell on DEX at $1.01
    //   → Profit: $0.009 per stablecoin
    //   → This pressure brings price back to $1.00
    
    // If stablecoin trades at $0.99 on DEX:
    //   → Buy from DEX at $0.99
    //   → Sell to PSM for $0.999 (1 - 0.01% fee)
    //   → Profit: $0.009 per stablecoin
    //   → This pressure brings price back to $1.00
}
```

---

## ตัวอย่าง: MoveUSD Protocol

```move
module moveusd::protocol {
    
    // ============================================
    // MoveUSD: Full stablecoin protocol on Aptos
    // Architecture:
    //   - CDP module: APT collateral → MoveUSD
    //   - PSM module: USDC 1:1 swap
    //   - StableSwap AMM: MoveUSD/USDC trading
    //   - Governance: MOVE token holders set params
    // ============================================
    
    struct MoveUSD {}   // The stablecoin
    struct MOVE {}      // The governance token
    
    // ============================================
    // System Parameters (governance-controlled)
    // ============================================
    
    struct SystemParams has key {
        governance: address,
        
        // CDP params
        min_collateral_ratio: u64,     // 150%
        liquidation_ratio: u64,        // 130%
        liquidation_bonus: u64,        // 10%
        stability_fee_bps_annual: u64, // 2%
        debt_ceiling: u64,             // Max MoveUSD from CDP
        
        // PSM params
        psm_buy_fee_bps: u64,          // 0.01%
        psm_sell_fee_bps: u64,         // 0.01%
        psm_ceiling: u64,              // Max USDC in PSM
        
        // StableSwap params
        stableswap_amp: u64,           // Amplification coefficient
        stableswap_fee_bps: u64,       // 0.04%
        
        // Emergency
        paused: bool,
        emergency_admin: address,
    }
    
    // ============================================
    // Statistics
    // ============================================
    
    struct ProtocolStats has key {
        total_moveUSD_supply: u64,
        total_apt_collateral: u64,
        total_usdc_psm: u64,
        
        total_stability_fees_earned: u64,
        total_psm_fees_earned: u64,
        total_liquidations: u64,
        
        cdp_collateral_ratio: u64,  // Average CR of all CDPs
        
        last_update: u64,
    }
    
    // ============================================
    // Risk Management
    // ============================================
    
    // Circuit breakers:
    // - Pause if APT price drops >20% in 1 hour
    // - Pause if total CDP ratio < 120%
    // - Emergency shutdown if total ratio < 105%
    
    public entry fun check_circuit_breakers(
        keeper: &signer,
        system_addr: address,
    ) acquires SystemParams, ProtocolStats {
        let stats = borrow_global<ProtocolStats>(system_addr);
        let params = borrow_global_mut<SystemParams>(system_addr);
        
        // Check average collateral ratio
        if (stats.cdp_collateral_ratio < 12_000) {  // < 120%
            params.paused = true;
            // Emit emergency event
        };
    }
    
    // ============================================
    // Revenue distribution
    // ============================================
    
    // Revenue sources:
    // 1. CDP stability fees (2% annual on debt)
    // 2. PSM fees (0.01% per swap)
    // 3. StableSwap fees
    
    // Revenue distribution:
    // 50% → MOVE token stakers
    // 30% → Protocol surplus buffer (backstop)
    // 20% → MoveUSD peg defense fund
    
    struct SurplusBuffer has key {
        moveUSD_balance: u64,  // Backstop for bad debt
    }
    
    public entry fun distribute_revenue(
        system_addr: address,
        period_revenue: u64,
    ) {
        let to_stakers = period_revenue * 50 / 100;
        let to_buffer = period_revenue * 30 / 100;
        let to_peg_fund = period_revenue * 20 / 100;
        
        // Distribute to MOVE stakers (similar to Part 43 revenue distributor)
        // Add to surplus buffer
        // Add to peg defense fund
    }
    
    // ============================================
    // Price stability analysis
    // ============================================
    
    // Analyze peg stability mechanisms:
    // 
    // When MoveUSD > $1.00:
    //   1. CDP users mint more (more supply)
    //   2. PSM arbitrageurs sell USDC for MoveUSD → sell MoveUSD on DEX
    //   3. StableSwap attracts traders, more USDC flows in
    //   All = sell pressure on MoveUSD → price drops to $1.00
    //
    // When MoveUSD < $1.00:
    //   1. CDP users buy MoveUSD to repay (less supply)
    //   2. PSM arbitrageurs buy MoveUSD on DEX → sell to PSM for USDC
    //   3. StableSwap provides cheap conversion
    //   All = buy pressure on MoveUSD → price rises to $1.00
    //
    // Backstop (if both fail):
    //   Surplus buffer covers bad debt from undercollateralized CDPs
    //   Emergency shutdown redeems all MoveUSD at $1.00 face value
}
```

---

## สรุป Stablecoin Design Principles

```
Stablecoin Design Tradeoffs (Trilemma):
  You can optimize for 2 of 3:
  1. Stability (peg maintenance)
  2. Decentralization (trustless)
  3. Capital Efficiency (minimal collateral)
  
USDC: Stable + Capital Efficient (but centralized)
DAI/MoveUSD: Stable + Decentralized (but capital inefficient 150%)
UST: Efficient + Decentralized (but NOT stable → failed)

Key Mechanisms:
  CDP: Overcollateralization → reliable peg, capital inefficient
  PSM: 1:1 USDC swap → strong peg, centralizing
  Stableswap: Low slippage trading → good UX
  Surplus Buffer: Backstop for bad debt

Safety Principles:
  1. Always overcollateralized (never pure algo)
  2. TWAP oracles (not spot)
  3. Circuit breakers (pause if ratio drops)
  4. Surplus buffer (absorbs bad debt)
  5. Emergency shutdown (return $1 face value to holders)
  6. Gradual parameter changes (governance delay)
  7. Multiple collateral types (diversification)
  8. PSM as peg stabilizer (not primary mechanism)

Lessons from UST Failure:
  ✗ Purely algorithmic (no real collateral)
  ✗ Single point of failure (LUNA)
  ✗ Death spiral: lost peg → panic → lost more peg
  ✓ Real collateral can't go to zero
  ✓ Overcollateralization survives price drops
```

---

**ก่อนหน้า**: [Part 52 - Concentrated Liquidity ←](part-52-concentrated-liquidity.md)
**ต่อไป**: [Part 54 - Cross-chain DeFi Architecture →](part-54-crosschain-defi.md)
