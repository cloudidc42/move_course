# Part 96: Economic Modeling & Simulation

## สารบัญ
- [Token Economic Design](#token-economic-design)
- [Agent-Based Simulation](#agent-based-simulation)
- [Game Theory in DeFi](#game-theory-in-defi)
- [Liquidity Mining Analysis](#liquidity-mining-analysis)
- [Protocol Revenue Modeling](#protocol-revenue-modeling)
- [Stress Testing Scenarios](#stress-testing-scenarios)

---

## Token Economic Design

```
TOKEN ECONOMIC FUNDAMENTALS

VALUE FLOWS IN A DeFi PROTOCOL

  User Activity
       │
       ├── Trading fees → Protocol Revenue
       ├── Lending spread → Protocol Revenue
       └── NFT royalties → Creator Revenue
                │
                ▼
         Protocol Revenue
                │
         ┌──────┴──────┐
         │             │
    Token Buyback   Distribute to
    & Burn          veToken holders
         │
         ▼
    Reduce supply → Appreciation

EQUILIBRIUM ANALYSIS

  Token Price = f(utility, speculation, supply/demand)
  
  Utility drivers:
    - Fee discounts (hold to save on fees)
    - Governance rights (hold to vote)
    - Yield (stake to earn protocol fees)
    - Collateral (use as collateral for borrowing)
  
  Supply drivers:
    - Emission rate (minting)
    - Vesting unlocks
    - Buyback & burn rate
    - Lock-up ratios (ve tokens reduce effective supply)
  
  Demand drivers:
    - Protocol TVL (more TVL → more fees → more demand)
    - Market sentiment
    - Competitive alternatives

TOKEN ECONOMIC KPIs
  P/E Ratio = Market Cap / Annual Revenue
  P/S Ratio = Market Cap / Annual Protocol Fees
  Token Velocity = Volume / Market Cap (lower = healthier)
  Lock-Up Ratio = veToken Supply / Total Supply (higher = healthier)
  Revenue Yield = Annual Revenue / Market Cap
```

---

## Agent-Based Simulation

```python
#!/usr/bin/env python3
"""
Agent-Based Economic Simulation for DeFi Protocol
Models user behavior, liquidity dynamics, and token economics
"""

import random
import math
from dataclasses import dataclass, field
from typing import List, Dict, Optional
from enum import Enum

class AgentType(Enum):
    ARBITRAGEUR = "arbitrageur"
    LONG_TERM_LP = "long_term_lp"
    YIELD_FARMER = "yield_farmer"
    TRADER = "trader"
    WHALE = "whale"

@dataclass
class Agent:
    id: int
    agent_type: AgentType
    capital_usd: float
    risk_tolerance: float    # 0 = risk averse, 1 = risk seeking
    time_horizon: int        # Days they plan to stay
    positions: Dict[str, float] = field(default_factory=dict)
    pnl: float = 0.0
    actions_taken: int = 0

@dataclass
class ProtocolState:
    # AMM pool
    reserve_x: float = 1_000_000.0   # Token X
    reserve_y: float = 2_000_000.0   # USDC
    total_lp: float = 1_414_213.0    # sqrt(1e6 * 2e6)
    
    # Token economics
    token_price: float = 1.0
    total_supply: float = 100_000_000.0
    circulating_supply: float = 30_000_000.0
    locked_supply: float = 0.0       # veToken locked
    
    # Protocol metrics
    fee_bps: int = 30
    tvl_usd: float = 3_000_000.0
    revenue_24h: float = 0.0
    volume_24h: float = 0.0
    emission_rate: float = 100_000.0  # tokens/day
    
    # Accumulated stats
    day: int = 0
    total_revenue: float = 0.0
    total_volume: float = 0.0

class DeFiSimulator:
    def __init__(
        self,
        n_agents: int = 100,
        simulation_days: int = 365,
    ):
        self.state = ProtocolState()
        self.agents = self._create_agents(n_agents)
        self.simulation_days = simulation_days
        self.history: List[Dict] = []
    
    def _create_agents(self, n: int) -> List[Agent]:
        agents = []
        agent_types = [
            (AgentType.ARBITRAGEUR, 0.10, 0.9, 1),
            (AgentType.LONG_TERM_LP, 0.25, 0.3, 180),
            (AgentType.YIELD_FARMER, 0.30, 0.6, 30),
            (AgentType.TRADER, 0.25, 0.5, 7),
            (AgentType.WHALE, 0.10, 0.4, 365),
        ]
        
        for i in range(n):
            # Pick agent type (weighted random)
            r = random.random()
            cumulative = 0
            agent_type, risk, horizon_mult = AgentType.TRADER, 0.5, 7
            for atype, weight, risk_tol, horizon in agent_types:
                cumulative += weight
                if r < cumulative:
                    agent_type = atype
                    risk = risk_tol
                    horizon_mult = horizon
                    break
            
            capital = random.lognormal(math.log(10_000), 1.5)  # Heavy-tailed distribution
            
            agents.append(Agent(
                id=i,
                agent_type=agent_type,
                capital_usd=min(capital, 10_000_000),  # Cap at $10M
                risk_tolerance=risk + random.gauss(0, 0.1),
                time_horizon=int(horizon_mult * random.uniform(0.5, 2.0)),
            ))
        
        return agents
    
    def simulate_day(self):
        self.state.day += 1
        self.state.revenue_24h = 0
        self.state.volume_24h = 0
        
        # Simulate each agent's actions
        random.shuffle(self.agents)  # Randomize order
        for agent in self.agents:
            self._agent_act(agent)
        
        # Protocol mechanics
        self._emit_tokens()
        self._update_token_price()
        self._record_history()
    
    def _agent_act(self, agent: Agent):
        if agent.agent_type == AgentType.ARBITRAGEUR:
            self._arbitrageur_strategy(agent)
        elif agent.agent_type == AgentType.LONG_TERM_LP:
            self._lp_strategy(agent)
        elif agent.agent_type == AgentType.YIELD_FARMER:
            self._yield_farmer_strategy(agent)
        elif agent.agent_type == AgentType.TRADER:
            self._trader_strategy(agent)
        elif agent.agent_type == AgentType.WHALE:
            self._whale_strategy(agent)
    
    def _arbitrageur_strategy(self, agent: Agent):
        """Arbitrageur: trade when price deviates from fair value"""
        fair_value = 1.0  # External market price
        pool_price = self.state.reserve_y / self.state.reserve_x
        
        deviation = abs(pool_price - fair_value) / fair_value
        
        if deviation > 0.005:  # 0.5% threshold
            # Trade to rebalance
            if pool_price > fair_value:
                # Pool overpriced X, sell X for Y
                trade_size = min(
                    agent.capital_usd * 0.3,
                    self.state.reserve_x * 0.02,
                )
                self._execute_swap(agent, trade_size, is_x_to_y=True)
            else:
                # Pool underpriced X, buy X
                trade_size = min(
                    agent.capital_usd * 0.3,
                    self.state.reserve_y * 0.02,
                )
                self._execute_swap(agent, trade_size, is_x_to_y=False)
    
    def _lp_strategy(self, agent: Agent):
        """LP: provide liquidity if APR is attractive"""
        if 'lp' not in agent.positions:
            # Check if APR is attractive enough
            apr = self._estimate_lp_apr()
            if apr > 0.05 + (1 - agent.risk_tolerance) * 0.10:  # 5-15% hurdle
                lp_amount = agent.capital_usd * 0.6
                self._add_liquidity(agent, lp_amount)
        else:
            # Remove if time horizon passed
            if self.state.day >= agent.time_horizon:
                self._remove_liquidity(agent)
    
    def _yield_farmer_strategy(self, agent: Agent):
        """Yield farmer: deposit to earn emission rewards"""
        emission_apr = (self.state.emission_rate * self.state.token_price * 365) / self.state.tvl_usd
        
        if emission_apr > 0.30 and 'farm' not in agent.positions:
            deposit = agent.capital_usd * 0.8
            agent.positions['farm'] = deposit
            agent.capital_usd -= deposit
        elif emission_apr < 0.10 and 'farm' in agent.positions:
            # Exit when yield drops too low
            amount = agent.positions.pop('farm')
            agent.capital_usd += amount  # Ignore IL for simplicity
    
    def _trader_strategy(self, agent: Agent):
        """Random trader: normal distribution of trade sizes"""
        if random.random() < 0.3:  # 30% chance of trading per day
            trade_size = abs(random.gauss(agent.capital_usd * 0.05, agent.capital_usd * 0.03))
            trade_size = min(trade_size, agent.capital_usd * 0.2)
            is_buy = random.random() > 0.5
            self._execute_swap(agent, trade_size, is_buy)
    
    def _whale_strategy(self, agent: Agent):
        """Whale: large infrequent moves, coordinated with market events"""
        if random.random() < 0.02:  # 2% chance per day
            trade_size = agent.capital_usd * random.uniform(0.1, 0.5)
            is_buy = random.random() > 0.5
            self._execute_swap(agent, trade_size, is_buy)
    
    def _execute_swap(self, agent: Agent, amount_usd: float, is_x_to_y: bool):
        """Execute AMM swap"""
        if amount_usd <= 0 or amount_usd > agent.capital_usd:
            return
        
        fee = amount_usd * self.state.fee_bps / 10_000
        amount_in = amount_usd - fee
        
        if is_x_to_y:
            amount_x = amount_in / (self.state.reserve_y / self.state.reserve_x)
            amount_out = self.state.reserve_y - (
                self.state.reserve_x * self.state.reserve_y / (self.state.reserve_x + amount_x)
            )
            self.state.reserve_x += amount_x
            self.state.reserve_y -= amount_out
        else:
            amount_out = self.state.reserve_x - (
                self.state.reserve_x * self.state.reserve_y / (self.state.reserve_y + amount_in)
            )
            self.state.reserve_y += amount_in
            self.state.reserve_x -= amount_out
        
        self.state.volume_24h += amount_usd
        self.state.revenue_24h += fee
        self.state.tvl_usd = (self.state.reserve_x + self.state.reserve_y)
        agent.actions_taken += 1
    
    def _add_liquidity(self, agent: Agent, amount_usd: float):
        lp_tokens = amount_usd / self.state.tvl_usd * self.state.total_lp
        agent.positions['lp'] = lp_tokens
        agent.capital_usd -= amount_usd
        self.state.total_lp += lp_tokens
        self.state.tvl_usd += amount_usd
    
    def _remove_liquidity(self, agent: Agent):
        lp_tokens = agent.positions.pop('lp', 0)
        value = lp_tokens / self.state.total_lp * self.state.tvl_usd
        agent.capital_usd += value
        self.state.total_lp -= lp_tokens
        self.state.tvl_usd -= value
    
    def _emit_tokens(self):
        """Distribute daily emissions to farmers"""
        self.state.circulating_supply += self.state.emission_rate
        # Buyback-and-burn with 20% of revenue
        burn_value = self.state.revenue_24h * 0.20
        tokens_burned = burn_value / self.state.token_price if self.state.token_price > 0 else 0
        self.state.total_supply -= tokens_burned
        self.state.circulating_supply -= tokens_burned
    
    def _estimate_lp_apr(self) -> float:
        daily_fees = self.state.revenue_24h
        apr = (daily_fees * 365) / self.state.tvl_usd if self.state.tvl_usd > 0 else 0
        return apr
    
    def _update_token_price(self):
        """Token price based on supply/demand"""
        demand = self.state.revenue_24h * 365 * 15  # 15x revenue multiple
        supply_factor = self.state.circulating_supply / 100_000_000
        self.state.token_price = (demand / self.state.circulating_supply) if self.state.circulating_supply > 0 else 0
        self.state.token_price = max(0.01, self.state.token_price)
    
    def _record_history(self):
        self.history.append({
            'day': self.state.day,
            'tvl': self.state.tvl_usd,
            'volume': self.state.volume_24h,
            'revenue': self.state.revenue_24h,
            'token_price': self.state.token_price,
            'circulating_supply': self.state.circulating_supply,
            'lp_apr': self._estimate_lp_apr(),
        })
    
    def run(self):
        for _ in range(self.simulation_days):
            self.simulate_day()
        return self.history
    
    def print_summary(self):
        last = self.history[-1]
        first = self.history[0]
        
        print(f"\n{'='*50}")
        print(f"SIMULATION SUMMARY ({self.simulation_days} days)")
        print(f"{'='*50}")
        print(f"TVL: ${first['tvl']:,.0f} → ${last['tvl']:,.0f}")
        print(f"Volume (avg daily): ${sum(h['volume'] for h in self.history) / len(self.history):,.0f}")
        print(f"Revenue (total): ${sum(h['revenue'] for h in self.history):,.0f}")
        print(f"Token Price: ${first['token_price']:.3f} → ${last['token_price']:.3f}")
        print(f"LP APR (avg): {sum(h['lp_apr'] for h in self.history) / len(self.history) * 100:.1f}%")

if __name__ == "__main__":
    sim = DeFiSimulator(n_agents=200, simulation_days=365)
    history = sim.run()
    sim.print_summary()
```

---

## Game Theory in DeFi

```
GAME THEORY ANALYSIS FOR DeFi PROTOCOLS

1. PRISONER'S DILEMMA IN LIQUIDITY PROVISION
   
   Each LP's strategy:
     Cooperate: Provide liquidity (good for ecosystem)
     Defect: Withdraw at first sign of trouble
   
   Nash Equilibrium problem: If all LPs fear others will leave,
   all leave (bank run). Protocol collapses despite being solvent.
   
   Solution: Withdrawal delays + cooldown periods
   Mechanism: Convert to gradual exit, not instant run

2. LIQUIDITY WARS (CURVE-STYLE)
   
   Protocol A and B both want deep USDC/USDT liquidity.
   
   Each can:
     - Offer high emissions (expensive)
     - Bribe veToken holders to direct emissions
     
   Nash Equilibrium: Bidding war drives emission costs up.
   Winner: Protocol with best unit economics.
   
   Optimal strategy: 
     Buy veTokens (lock), direct emissions, generate revenue,
     repeat. Self-reinforcing if revenue > emission cost.

3. ORACLE MANIPULATION
   
   Attacker's payoff matrix:
     Flash loan cost: F
     Manipulation profit: P
     Slashing risk: S
     
   Rational attack: if P > F + S
   
   Defense: Use TWAP (time average) instead of spot
     TWAP requires sustained price manipulation over N blocks
     Cost = N × F >> single-block manipulation cost
   
   Multi-source oracle: need to manipulate M of N oracles
     Cost scales as M × N × F
   
4. GOVERNANCE CAPTURE
   
   Token distribution matters for governance health:
     Gini coefficient → concentration risk
     Voter turnout → legitimacy
     Token velocity → speculation vs. governance value
   
   Attack threshold: acquire > 51% of voting power
   Defense: Quorum requirements + vote delegation
   
5. MEV EXTRACTION
   
   Arbitrageur extracts value from protocol users.
   This is a zero-sum game: arb gain = trader slippage loss.
   
   Protocol design to minimize MEV:
     Commit-reveal: hide intent until block confirmed
     Batch auctions: everyone gets same price
     TWAP execution: spread large orders over time
```

---

## Protocol Revenue Modeling

```python
#!/usr/bin/env python3
"""
Protocol revenue projection model
Sensitivity analysis for key parameters
"""

from dataclasses import dataclass
from typing import List

@dataclass
class RevenueProjection:
    year: int
    tvl: float
    daily_volume_to_tvl: float   # Volume/TVL ratio
    fee_bps: int
    lending_utilization: float
    lending_spread_bps: int
    
    def amm_revenue_annual(self) -> float:
        """Fee revenue from AMM trading"""
        daily_volume = self.tvl * self.daily_volume_to_tvl
        daily_fee = daily_volume * self.fee_bps / 10_000
        return daily_fee * 365
    
    def lending_revenue_annual(self) -> float:
        """Revenue from lending spread (supply rate vs. borrow rate)"""
        borrowed = self.tvl * self.lending_utilization * 0.7  # 70% of TVL in lending
        return borrowed * self.lending_spread_bps / 10_000
    
    def total_revenue(self) -> float:
        return self.amm_revenue_annual() + self.lending_revenue_annual()
    
    def token_value(self, pe_ratio: float = 20) -> float:
        """Estimate token market cap = revenue × P/E"""
        return self.total_revenue() * pe_ratio
    
    def print_report(self):
        print(f"\nYear {self.year} Projection:")
        print(f"  TVL: ${self.tvl:,.0f}")
        print(f"  AMM Revenue: ${self.amm_revenue_annual():,.0f}")
        print(f"  Lending Revenue: ${self.lending_revenue_annual():,.0f}")
        print(f"  Total Revenue: ${self.total_revenue():,.0f}")
        print(f"  Token Value (20x PE): ${self.token_value():,.0f}")


def sensitivity_analysis():
    """Run sensitivity analysis on key parameters"""
    base_tvl = 50_000_000  # $50M TVL
    
    print("TVL SENSITIVITY (Volume/TVL=0.3, fee=30bps)")
    for tvl_mult in [0.5, 1.0, 2.0, 5.0, 10.0]:
        p = RevenueProjection(
            year=1,
            tvl=base_tvl * tvl_mult,
            daily_volume_to_tvl=0.30,
            fee_bps=30,
            lending_utilization=0.60,
            lending_spread_bps=200,
        )
        print(f"  TVL ${p.tvl/1e6:.0f}M: Revenue ${p.total_revenue()/1e6:.2f}M, "
              f"Token Value ${p.token_value()/1e6:.1f}M")
    
    print("\nFEE SENSITIVITY (TVL=$50M, Volume/TVL=0.3)")
    for fee_bps in [5, 10, 20, 30, 50, 100]:
        p = RevenueProjection(
            year=1,
            tvl=base_tvl,
            daily_volume_to_tvl=0.30,
            fee_bps=fee_bps,
            lending_utilization=0.60,
            lending_spread_bps=200,
        )
        print(f"  Fee {fee_bps}bps: AMM Revenue ${p.amm_revenue_annual()/1e6:.2f}M")
    
    print("\n3-YEAR GROWTH SCENARIO (Base Case)")
    growth_scenarios = [
        (1, 10_000_000),   # Year 1: $10M TVL
        (2, 50_000_000),   # Year 2: $50M TVL
        (3, 150_000_000),  # Year 3: $150M TVL
    ]
    for year, tvl in growth_scenarios:
        p = RevenueProjection(
            year=year,
            tvl=tvl,
            daily_volume_to_tvl=0.40,
            fee_bps=30,
            lending_utilization=0.65,
            lending_spread_bps=250,
        )
        p.print_report()

if __name__ == "__main__":
    sensitivity_analysis()
```

---

## สรุป Economic Modeling

```
ECONOMIC MODELING TOOLKIT

1. SIMULATION APPROACHES
   Monte Carlo: Random market conditions, measure outcome distribution
   Agent-Based: Individual agents with behavior rules, emergent properties
   System Dynamics: Stock/flow model of protocol variables
   
   Use cases:
     Pre-launch: Validate token economics
     Post-launch: Calibrate emissions/fees
     Risk: Stress test under adversarial conditions

2. KEY METRICS TO MODEL
   TVL dynamics: Entry/exit based on yield and market conditions
   Volume: Function of TVL, market volatility, user count
   Revenue: Volume × fee rate (adjust for utilization)
   Token price: Revenue multiple (P/E) adjusted for growth and risk
   Token velocity: High velocity = low store-of-value = bearish

3. GAME THEORY APPLICATIONS
   Oracle design: TWAP makes manipulation expensive
   Liquidity wars: bribe market equilibrium
   Governance: Quorum + stake prevents capture
   MEV defense: Commit-reveal, batch auctions

4. CALIBRATION SOURCES
   Volume/TVL ratio: 0.1-0.5× for AMM, 0.05-0.2× for lending
   Fee rates: 0.05-0.3% for stable pairs, 0.2-1% for volatile
   P/E multiples: 10-30× for revenue-generating DeFi protocols
   Emission decay: Halve every 6-12 months to avoid hyperinflation

5. STRESS TEST SCENARIOS
   Bear market: TVL drops 80%, volume drops 90%
   Smart contract exploit: Sudden 100% TVL exit
   Oracle failure: Price manipulation attack
   Governance attack: 51% token accumulation
   Bridge exploit: 50% of bridged assets stolen
```

---

**ก่อนหน้า**: [Part 95 - Production Launch ←](part-95-launch-checklist.md)
**ต่อไป**: [Part 97 - Move Language Internals →](part-97-move-internals.md)
