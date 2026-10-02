# Part 99: Contributing to the Move Ecosystem

## สารบัญ
- [Understanding the Ecosystem](#understanding-the-ecosystem)
- [Contributing to Move Core](#contributing-to-move-core)
- [Building Developer Tools](#building-developer-tools)
- [Writing Standards and MIPs](#writing-standards-and-mips)
- [Community Building](#community-building)
- [Open Source Protocol Development](#open-source-protocol-development)
- [Career in Move Development](#career-in-move-development)

---

## Understanding the Ecosystem

```
THE MOVE ECOSYSTEM MAP

CORE LANGUAGE & VM
  ┌─────────────────────────────────────────────────────────┐
  │  Move Language (github.com/move-language/move)          │
  │  - Compiler, VM, standard library                       │
  │  - Move Prover (formal verification)                    │
  │  - bytecode-verifier, disassembler                      │
  └─────────────────────────────────────────────────────────┘
                        ↓ Fork + Customize
  ┌─────────────────────┐         ┌────────────────────────┐
  │  Aptos Move         │         │  Sui Move              │
  │  aptos-core/        │         │  sui/                  │
  │  + aptos_std        │         │  + sui_framework       │
  │  + aptos_framework  │         │  + object model        │
  │  + aptos_token      │         │  + kiosk               │
  └─────────────────────┘         └────────────────────────┘

TOOLING LAYER
  Editors:         VS Code Move extension, IntelliJ plugin
  Linting:         Move Lint, custom Prover specs
  Testing:         Move Test, property-based, fork testing
  Deployment:      Aptos CLI, Sui CLI
  Indexing:        Aptos Indexer SDK, Mysten's indexer

PROTOCOL LAYER
  AMMs:            PancakeSwap, Thala, Turbos, Cetus
  Lending:         Aries Markets, Navi, Scallop
  Stablecoins:     MOD (Thala), BUCK (Bucket Protocol)
  Bridges:         LayerZero, Wormhole, Axelar
  Oracles:         Pyth, Switchboard, Supra

APPLICATION LAYER
  NFT:             Wapal, Tradeport, Bluemove
  Gaming:          Blocto, Aptus, various on Sui
  Wallets:         Petra, Pontem, Martian (Aptos)
                   Sui Wallet, Suiet, Phantom (Sui)

YOUR CONTRIBUTION OPPORTUNITIES
  Any layer above is open for contribution
  Highest leverage: tooling, standards, education
```

---

## Contributing to Move Core

```
HOW TO CONTRIBUTE TO MOVE LANGUAGE

STEP 1: UNDERSTAND THE CODEBASE

git clone https://github.com/move-language/move
cd move

# Key directories:
language/move-compiler/     # Lexer, parser, IR builder
language/move-vm/           # Stack machine, interpreter
language/move-stdlib/       # core, vector, hash modules
language/move-prover/       # Formal verification backend

# Build and test:
cargo build
cargo test --workspace

STEP 2: FIND AN ISSUE TO WORK ON

Good first issues:
  label: "good first issue" on GitHub
  Examples:
  - Improve error messages
  - Add new Move standard library function
  - Fix documentation
  - Improve performance of bytecode verifier

Medium issues:
  label: "help wanted"
  Examples:
  - Add new bytecode instruction
  - Improve Move Prover precision
  - Add linting checks

Advanced:
  - Move 2.0 features (macros, lambdas)
  - Cross-compilation targets
  - New VM optimizations

STEP 3: DEVELOPMENT WORKFLOW

# Fork the repository first
# Then:
git checkout -b feature/my-improvement

# Make your change
# Add tests
# Run the full test suite
cargo test --workspace

# Move Prover tests:
cd language/move-prover
cargo test

# Submit PR following contribution guidelines

STEP 4: CODE STYLE REQUIREMENTS

  - Rust: follow rustfmt style (cargo fmt)
  - All new code must have tests
  - All public APIs must have doc comments
  - Breaking changes need RFC
  - Performance changes need benchmarks

CONTRIBUTING TO APTOS MOVE

git clone https://github.com/aptos-labs/aptos-core
cd aptos-core/aptos-move

# Key areas for contribution:
aptos-framework/           # Core framework modules
move-examples/             # Example code (great for beginners)
aptos-stdlib/              # Standard library extensions

# Testing:
cd aptos-move/aptos-framework
aptos move test

# Specific contribution opportunities:
- aptos_std::math new functions
- aptos_token_v2 improvements
- New framework features (discussion first)

CONTRIBUTING TO SUI MOVE

git clone https://github.com/MystenLabs/sui
cd sui

# Key areas:
sui-framework/             # Core Sui framework
sui-move-examples/         # Example dApps
crates/sui-framework/      # Rust side of framework

# Testing:
cd sui-framework
sui move test

# Contribution opportunities:
- New kiosk rules
- sui::math extensions
- Random module improvements
- Documentation and examples
```

---

## Building Developer Tools

```
HIGH-IMPACT TOOLS TO BUILD

1. MOVE LINTER
   Analyzes Move code for common mistakes
   
   Checks to implement:
   ✓ Unused variables (compiler already catches)
   ✓ Magic numbers (should be named constants)
   ✓ Missing abort codes (use error::* not raw numbers)
   ✓ Unbounded loops
   ✓ Missing access control assertions
   ✓ Unsafe division (no div-by-zero check)
   
   Implementation approach:
   - Parse Move AST using move-compiler crate
   - Traverse AST with visitor pattern
   - Report findings with file:line:column
   
   Reference: move-lint in some forks

2. MOVE FORMATTER (moveformat)
   Consistent code style for Move projects
   
   Rules:
   - Indentation: 4 spaces
   - Max line length: 100 chars
   - Brace style: same-line
   - Import grouping: stdlib, framework, local
   
   Implementation:
   - Parse → AST → pretty-print with constraints
   - Integrate with VS Code as format-on-save

3. MOVE DEBUGGER
   Step-through debugging for Move tests
   
   Features needed:
   - Breakpoints in test functions
   - Inspect local variables at each step
   - View memory layout of structs
   - Step into/over calls
   
   Implementation:
   - Extend move-vm with debug hooks
   - DAP protocol (Debug Adapter Protocol)
   - VS Code extension for UI

4. MOVE COVERAGE TOOL
   Which lines are covered by tests?
   
   Current state: basic coverage in move-cli
   Enhancement: branch coverage, not just line
   
   Output format:
   - HTML report with colored lines
   - CI-friendly JSON with percentage
   - LCOV format for integration with tools

5. MOVE DOCUMENTATION GENERATOR
   Like rustdoc for Move
   
   Input: Move source with doc comments
   Output: HTML docs with:
   - Module overview
   - Function signatures
   - struct layouts
   - Ability constraints visualized
   - Examples from doc tests
   
   Current: basic doc generation in some tools
   Gap: interactive examples, search

6. MOVE PLAYGROUND (Web)
   Browser-based Move IDE for learning
   
   Features:
   - Write Move code in browser
   - Run tests instantly (WASM-compiled VM)
   - Share code via URL
   - Pre-loaded examples for each concept
   
   Implementation:
   - Compile Move VM to WASM
   - CodeMirror or Monaco editor
   - Serverless backend for sharing

7. MOVE UPGRADE TOOL
   Safely prepare and execute protocol upgrades
   
   Features:
   - Compare old vs new ABI (breaking changes?)
   - Simulate upgrade in local fork
   - Generate migration scripts
   - Roll back if migration fails
   
   Implementation:
   - ABI diff engine
   - Aptos/Sui testnet fork mode
   - Transaction simulation

TOOL SUBMISSION PATHS
  Submit to Aptos ecosystem:  github.com/aptos-labs/ecosystem-projects
  Submit to Sui ecosystem:    github.com/MystenLabs/sui/discussions
  Announce:                   Discord #developer-tools channels
  Blog post:                  medium.com/aptoslabs or blog.sui.io guest posts
```

---

## Writing Standards and MIPs

```
MOVE IMPROVEMENT PROPOSALS

WHAT ARE MIPs?
  Similar to EIPs (Ethereum), BIPs (Bitcoin)
  Formal proposals for Move language/framework changes
  
  Types:
  - Core: VM changes, new instructions
  - Framework: aptos_std / sui_framework changes
  - Interface: token standards, NFT standards
  - Informational: best practices, design patterns

HOW TO WRITE A GOOD MIP

STRUCTURE (follow this template):

---
MIP-XXX: Title
Author: Your Name <email>
Status: Draft
Type: Interface
Created: 2026-10-01
---

## Abstract
One paragraph summary of the proposal.
What problem does it solve? What is the solution?

## Motivation
Why is this necessary?
What are the use cases?
What is wrong with existing approaches?

## Specification

### Overview
[Technical description of the change]

### New Interfaces

```move
// New module to add to aptos_std
module aptos_std::your_proposal {
    /// Does X
    public fun new_function(param: Type): ReturnType {
        ...
    }
}
```

### State Changes
[What changes in global storage, if any]

### Events
[New events emitted, if any]

### Error Codes
[New error codes, if any]

## Reference Implementation
Link to working implementation:
github.com/your-username/aptos-core/tree/feature/mip-xxx

## Test Cases
[Unit tests that verify the specification]

## Security Considerations
[Attack vectors considered, mitigations]

## Backwards Compatibility
[Breaking changes? Migration path?]

## Copyright
Licensed under Apache 2.0

---

REAL EXAMPLES OF COMMUNITY-DRIVEN STANDARDS

TOKEN STANDARD (Aptos):
  aptos_token (v1): NFT standard, early ecosystem
  aptos_token_v2 (v2): more flexible, FA-compatible
  
  What community added:
  - Composable tokens (tokens on tokens)
  - Soulbound tokens (non-transferable)
  - Mutable metadata

OBJECT STANDARD (Aptos):
  aptos_framework::object module
  Community pushed for: deleteability, extensibility
  Result: objects now have LinearTransferRef, DeleteRef

KIOSK STANDARD (Sui):
  Sui team + community designed together
  Solves: royalty enforcement without trusted marketplace
  Community extensions: TransferPolicy rules (royalty, commission)

HOW TO SUBMIT

Aptos:
  1. Create GitHub issue in aptos-labs/aptos-core
  2. Label: "AIP" (Aptos Improvement Proposal)
  3. Discuss in Discord: #aip-discussion
  4. Final: submit PR to aptos-foundation/AIPs repo

Sui:
  1. Create forum post in forums.sui.io
  2. Category: "SIP" (Sui Improvement Proposal)
  3. Collect community feedback
  4. Final: submit to MystenLabs/sips repo
```

---

## Community Building

```
HOW TO BUILD A MOVE DEVELOPER COMMUNITY

1. CONTENT CREATION

Technical Blog Posts (High value):
  Topics that perform well:
  - "Move vs Solidity: X Key Differences"
  - "How I built X in Move in Y hours"
  - "Exploits that Move prevents by design"
  - "Understanding Move's ownership model"
  
  Platforms:
  - mirror.xyz (crypto-native, token gating possible)
  - medium.com/your-publication
  - dev.to (general developer audience)
  - Substack (newsletter, paid subscriptions)
  
  Cadence: Weekly or bi-weekly for consistency
  Quality > Quantity: One great post > five mediocre ones

Video Tutorials:
  Topics that work:
  - "Build a DEX in Move (from scratch)"
  - "Move Prover tutorial: formal verification"
  - "Aptos vs Sui: technical comparison"
  
  Platforms:
  - YouTube (long-form, discoverable)
  - Twitter/X Spaces (live, interactive)
  - Farcaster for crypto-native audience

2. OPEN SOURCE PROJECTS

Start a project that others can contribute to:
  - Move template library (like OpenZeppelin for Solidity)
  - Move security checklist (checklist + automation)
  - Move learning exercises (like rustlings)
  - Move testing framework extensions

How to attract contributors:
  ✅ Good README with clear problem statement
  ✅ CONTRIBUTING.md with easy first issues
  ✅ Respond to PRs within 24 hours
  ✅ Give credit generously (changelog, README contributors)
  ✅ Regular releases (semantic versioning)

3. EDUCATION

Workshops and Hackathons:
  Run a local Move workshop (in-person or virtual)
  Coordinate with Aptos/Sui teams:
    - Aptos: partnerships@aptoslabs.com
    - Sui: developer@mystenlabs.com
  They provide:
    - Marketing support
    - Prizes for winners
    - Technical support
    - Network access

Online Courses:
  Platforms: Udemy, Coursera, own website
  This curriculum can be adapted into a course
  Price: $29-$199 for a comprehensive Move course
  
  Monetization models:
  - One-time payment (Udemy)
  - Subscription (GitHub Sponsors for access)
  - Free course + paid consulting

Mentorship:
  Offer 1:1 mentoring for new Move developers
  Platforms: mentorcruise.com, topmate.io
  Value: You learn by teaching

4. GETTING RECOGNITION

Grants:
  Aptos Foundation: https://aptosfoundation.org/grants
  Sui Foundation: https://sui.io/grants
  Categories: ecosystem tools, DeFi protocols, education
  
  Application tips:
  - Concrete deliverables with dates
  - Prior work demonstrates execution ability
  - Budget breakdown with justification
  - Impact metrics (how many developers helped?)

Ambassador Programs:
  Aptos Ecosystem: ecosystem@aptoslabs.com
  Sui Builders: builders@mystenlabs.com
  
  What they offer:
  - Early access to new features
  - Technical support from core team
  - Conference speaker opportunities
  - Network with other builders

Speaking:
  Major conferences where Move is relevant:
  - ETHDenver / ETHGlobal events
  - Aptos Summit
  - Sui Base Camp
  - Token2049 (if you have a product)
  - EthCC (if pitching to EVM devs converting)
```

---

## Open Source Protocol Development

```
STARTING AN OPEN SOURCE PROTOCOL

WHY OPEN SOURCE?
  ✅ Trust: users can verify no backdoors
  ✅ Contributions: community improves your code
  ✅ Security: more eyes find more bugs
  ✅ Protocol Network Effects: integrations easier
  ✅ Credibility: serious protocols are open source
  ❌ Copying risk: competitors can fork (accept this)

REPOSITORY STRUCTURE

my-protocol/
├── CLAUDE.md              # AI assistant context
├── README.md              # Overview and quick start
├── SECURITY.md            # Vulnerability disclosure policy
├── CONTRIBUTING.md        # How to contribute
├── LICENSE                # Apache 2.0 or MIT
├── move/
│   ├── Move.toml
│   ├── sources/
│   │   ├── core.move       # Core protocol logic
│   │   ├── math.move       # Math utilities
│   │   └── events.move     # Event definitions
│   └── tests/
│       ├── core_tests.move
│       └── math_tests.move
├── sdk/
│   ├── typescript/         # TypeScript SDK
│   │   ├── src/
│   │   ├── tests/
│   │   └── package.json
│   └── python/             # Python SDK (analytics)
├── scripts/
│   ├── deploy_mainnet.sh
│   ├── deploy_testnet.sh
│   └── upgrade.sh
├── docs/
│   ├── architecture.md
│   ├── api.md
│   └── security-model.md
└── audits/
    ├── 2024-spearbit.pdf
    └── 2025-ottersec.pdf

LICENSING CONSIDERATIONS

Apache 2.0: Standard for blockchain protocols
  - Permissive: anyone can use, modify, distribute
  - Patent protection: explicit patent grant
  - Attribution required
  
MIT: Even simpler
  - Very permissive
  - Short license
  
BUSL (Business Source License):
  - Popular in DeFi (Uniswap V3)
  - Converts to permissive after X years (e.g., 4 years)
  - Prevents forks from competing commercially initially
  
  Considerations: alienates some open source advocates

SECURITY.md TEMPLATE

# Security Policy

## Reporting a Vulnerability

**DO NOT** create a public GitHub issue for security bugs.

Email: security@yourprotocol.com
PGP Key: [link to keyserver]

Expected response time: 24 hours for acknowledgment
Bug bounty: Yes — see https://immunefi.com/bounty/yourprotocol

## Scope

In-scope:
- Protocol smart contracts (all deployed versions)
- SDK libraries
- Deployed infrastructure (if applicable)

Out-of-scope:
- Frontend website
- Third-party dependencies (report to them)
- Known issues in this SECURITY.md

## Disclosure Policy

1. You report vulnerability
2. We acknowledge in 24 hours
3. We fix and test in 7-30 days (depends on severity)
4. We deploy fix to mainnet
5. We disclose publicly after 90 days or after fix

MAINTAINING A PROTOCOL LONG-TERM

Version management:
  - Semantic versioning: major.minor.patch
  - Major: breaking changes
  - Minor: new features (backward compatible)
  - Patch: bug fixes
  
Changelog:
  - CHANGELOG.md with every version
  - Format: Added, Changed, Deprecated, Removed, Fixed, Security

Community health:
  - Respond to issues within 48 hours
  - Monthly update blog posts
  - Annual roadmap announcement
  - Governance transition plan (when to DAO?)
```

---

## Career in Move Development

```
MOVE DEVELOPER CAREER PATHS

PATH 1: PROTOCOL DEVELOPER (In-house)

Typical companies: Aptos Labs, Mysten Labs, DeFi protocols
Salary range: $150k-$400k USD + tokens

Day-to-day:
  - Design and implement new protocol features
  - Write Move smart contracts
  - Code review other engineers' Move
  - Work with auditors during audit cycles
  - Write technical documentation

Skills needed:
  - Expert-level Move (this course covers this)
  - Rust (for tooling, VMs)
  - TypeScript (for SDKs and tests)
  - System design experience
  - Cryptography basics

How to get there:
  1. Complete this curriculum + projects
  2. Contribute to ecosystem open source
  3. Write blog posts demonstrating expertise
  4. Apply to Aptos/Sui ecosystem companies
  5. Network at conferences

PATH 2: INDEPENDENT PROTOCOL FOUNDER

Build and launch your own protocol
Income: Token allocation (% of supply), transaction fees

Day-to-day:
  - Lead technical architecture
  - Manage team of 2-10 engineers
  - Raise venture capital
  - Partner with other protocols
  - Community management

Skills needed:
  - Everything in Path 1
  - Business development
  - Tokenomics design
  - Community management
  - Fundraising

How to get there:
  1. Build a compelling MVP
  2. Launch on testnet, gather users
  3. Apply to accelerators (Aptos Foundation, Sui Builder Program)
  4. Raise seed round from crypto VCs

PATH 3: SECURITY AUDITOR

Audit protocols before launch
Income: $200-$500/hour or $50k-$500k per audit

Day-to-day:
  - Read protocol code looking for bugs
  - Write proof-of-concept exploits
  - Write detailed audit reports
  - Stay current on new vulnerability patterns

Skills needed:
  - Deep Move knowledge (adversarial mindset)
  - Experience with DeFi exploits
  - Formal verification knowledge
  - Report writing

How to get there:
  1. Study all known Move/DeFi exploits
  2. Participate in bug bounties (Immunefi, Code4rena)
  3. Write public security research
  4. Apply to audit firms (OtterSec, Halborn, Zellic)
  5. Or freelance through Immunefi

PATH 4: DEVELOPER ADVOCATE (DevRel)

Bridge between protocol teams and developers
Income: $120k-$250k + tokens

Day-to-day:
  - Write developer documentation
  - Create tutorials and example code
  - Give talks at conferences
  - Answer developer questions on Discord
  - Gather feedback from developers

Skills needed:
  - Move knowledge (intermediate+)
  - Excellent writing and speaking
  - Community building
  - Example code that actually works

How to get there:
  1. Build public profile (blog, Twitter, GitHub)
  2. Become known in the community
  3. Apply to DevRel roles at Aptos/Sui/protocols

PATH 5: TOOLING ENGINEER

Build infrastructure for Move developers
Income: $150k-$300k + tokens or open source/grants

Day-to-day:
  - Build and maintain IDEs, linters, debuggers
  - Improve Move compiler and toolchain
  - Integrate with CI/CD pipelines

Skills needed:
  - Rust (the Move toolchain is in Rust)
  - Compiler theory (AST, IR, code generation)
  - Developer experience design

How to get there:
  1. Contribute to move-language repo
  2. Build a popular tool (see Building Developer Tools section)
  3. Apply to Aptos/Mysten tooling teams

PORTFOLIO CHECKLIST
  For any path above, your portfolio should show:
  
  □ 3+ completed Move protocols on testnet
  □ 1+ protocols deployed on mainnet
  □ Open source contributions (PRs merged)
  □ Technical writing (blog posts or docs)
  □ 1+ audit report (participated in)
  □ Active in community (Discord, Twitter, GitHub)
  
  Additional for founders:
  □ Users who actually use your protocol
  □ TVL > $0 on mainnet

LEARNING RESOURCES BEYOND THIS COURSE
  Move Book:        https://move-language.github.io/move/
  Aptos Learn:      https://learn.aptoslabs.com
  Sui Dev Docs:     https://docs.sui.io
  Move Examples:    github.com/aptos-labs/aptos-core/tree/main/aptos-move/move-examples
  Sui Examples:     github.com/MystenLabs/sui/tree/main/examples
  Discord Servers:
    Aptos Dev:      discord.gg/aptosnetwork  
    Sui Dev:        discord.gg/sui
```

---

## สรุป Contributing to Move Ecosystem

```
YOUR IMPACT OPPORTUNITIES (ranked by leverage)

1. HIGHEST: Build tools that help 1000s of developers
   - Move linter, formatter, debugger
   - SDK improvements
   - Indexing infrastructure

2. HIGH: Launch a protocol with real users
   - Every successful protocol raises Move's profile
   - Open source protocols become reference implementations
   - TVL = credibility for the ecosystem

3. HIGH: Education and content
   - This course = leverage because it helps many
   - One great blog post → hundreds of new Move devs
   - Tutorials with working code are especially valuable

4. MEDIUM: Contribute to core
   - Direct impact on language/framework
   - Requires deeper Rust + compiler knowledge
   - Lower entry point: documentation, examples

5. MEDIUM: Standards and MIPs
   - Shapes how protocols are built
   - Requires deep ecosystem understanding
   - Network effects: widely adopted standards are very valuable

YOUR IMMEDIATE NEXT STEPS:
  Week 1:  Complete the curriculum (you're almost there!)
  Week 2:  Build a simple AMM on testnet
  Month 1: Deploy something on mainnet (even if small)
  Month 2: Write one blog post about what you built
  Month 3: Open a PR to aptos-core or sui (even docs)
  Month 6: Apply to protocols or launch your own

"The best time to plant a tree was 20 years ago.
 The second best time is now."
 
 Move is still early. The builders who ship in 2024-2026
 will be the pillars of this ecosystem in 2030.
```

---

**ก่อนหน้า**: [Part 98 - Real-World DeFi Case Studies ←](part-98-case-studies.md)
**ต่อไป**: [Part 100 - Final Capstone Project →](part-100-capstone.md)
