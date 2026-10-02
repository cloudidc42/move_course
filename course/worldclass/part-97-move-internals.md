# Part 97: Move Language Internals & VM Deep Dive

## สารบัญ
- [Move VM Architecture](#move-vm-architecture)
- [Bytecode & Execution Model](#bytecode--execution-model)
- [Type System Deep Dive](#type-system-deep-dive)
- [Memory Model & Ownership](#memory-model--ownership)
- [Aptos vs Sui VM Differences](#aptos-vs-sui-vm-differences)
- [Advanced Language Features](#advanced-language-features)

---

## Move VM Architecture

```
MOVE VIRTUAL MACHINE ARCHITECTURE

     Move Source Code (.move)
            │
            ▼ aptos move compile / sui move build
     Move Bytecode (.mv)
            │
            ▼ Verification (before execution)
     ┌──────────────────────────────────────┐
     │ BYTECODE VERIFIER                    │
     │ 1. Type Safety:  no type confusion   │
     │ 2. Resource Safety: no copies/drops  │
     │ 3. Reference Safety: no dangling refs│
     │ 4. Stack Safety: balanced push/pop   │
     └──────────────┬───────────────────────┘
                    │ verified
                    ▼
     ┌──────────────────────────────────────┐
     │ MOVE VM INTERPRETER                  │
     │                                      │
     │  ┌──────────┐  ┌────────────────┐    │
     │  │  Stack   │  │  Call Stack    │    │
     │  │          │  │  [frame1]      │    │
     │  │  [value] │  │  [frame2] ← current│
     │  │  [value] │  │    locals[]    │    │
     │  │  [value] │  │    module_id   │    │
     │  └──────────┘  │    function    │    │
     │                └────────────────┘    │
     │                                      │
     │  ┌─────────────────────────────────┐ │
     │  │ MODULE CACHE                    │ │
     │  │ {module_id → compiled module}   │ │
     │  └─────────────────────────────────┘ │
     └──────────────────────────────────────┘
                    │
                    ▼
     ┌──────────────────────────────────────┐
     │ GLOBAL STORAGE (via blockchain)      │
     │ {address → {module_id → struct data}}│
     └──────────────────────────────────────┘

KEY INVARIANTS (enforced at bytecode level)
  1. Resources never copied (only moved)
  2. Resources never dropped (must be stored or destroyed)
  3. References bounded to containing scope
  4. Globals accessed only via borrow_global / move_to / move_from
  5. Type parameters instantiated correctly (generics)
```

---

## Bytecode & Execution Model

```
MOVE BYTECODE INSTRUCTION SET (key instructions)

STACK OPERATIONS
  LdU8 n      → push u8 literal n
  LdU64 n     → push u64 literal n
  LdBool b    → push bool
  LdConst i   → push constant from constant pool
  CopyLoc i   → copy local i to stack (for non-resource types)
  MoveLoc i   → move local i to stack (for resources or others)
  StLoc i     → pop stack, store in local i

ARITHMETIC
  Add, Sub, Mul, Div, Mod
  BitAnd, BitOr, BitXor
  Shl, Shr
  Lt, Le, Gt, Ge, Eq, Neq

CONTROL FLOW
  Branch offset      → unconditional jump
  BrTrue offset      → pop bool, jump if true
  BrFalse offset     → pop bool, jump if false
  Call module::func  → call function, pass args from stack
  Ret                → return top of stack

STRUCT OPERATIONS
  Pack struct_id     → pop N fields, create struct, push result
  Unpack struct_id   → pop struct, push N fields
  
REFERENCE OPERATIONS
  ImmBorrowLoc i     → borrow local i immutably (&T)
  MutBorrowLoc i     → borrow local i mutably (&mut T)
  ImmBorrowField fid → get immutable reference to field
  MutBorrowField fid → get mutable reference to field
  ReadRef             → dereference &T → T (copies value)
  WriteRef            → write T through &mut T

GLOBAL OPERATIONS
  MoveTo              → move resource to global storage
  MoveFrom addr       → take resource from global storage
  BorrowGlobal addr   → borrow global resource immutably
  BorrowGlobalMut addr→ borrow global resource mutably
  Exists addr         → check if global resource exists

EXAMPLE: What happens when you write:
  let x = 5u64;
  let y = x + 3;
  
  Bytecode:
    LdU64 5        → stack: [5]
    StLoc 0        → stack: []   local[0] = 5
    CopyLoc 0      → stack: [5]  (copies x since u64 has Copy)
    LdU64 3        → stack: [5, 3]
    Add            → stack: [8]
    StLoc 1        → stack: []   local[1] = 8
```

---

## Type System Deep Dive

```move
// ============================================
// MOVE ABILITIES SYSTEM
// ============================================

// Ability: copy
// The type can be duplicated implicitly
// Without copy: value is moved (ownership transferred)

struct WithCopy has copy, drop {
    x: u64,
}

struct NoCopy has drop {
    resource: u64,  // Pretend this is a coin
}

module ability_demo {
    fun copy_demo() {
        let a = WithCopy { x: 1 };
        let b = a;          // Copy (a still accessible)
        let c = a;          // Another copy (a still accessible)
        let _ = (a, b, c);  // All accessible
    }
    
    fun move_demo() {
        let a = NoCopy { resource: 100 };
        let b = a;          // Move! a is no longer accessible
        // let c = a;       // ❌ COMPILE ERROR: a was moved
        let _ = b;
    }
    
    // ============================================
    // Ability: drop
    // Value can be discarded (goes out of scope)
    // Without drop: must be explicitly consumed
    
    struct MustConsume has key {
        coin: u64,
    }
    
    // This function MUST use the MustConsume value
    fun use_resource(res: MustConsume) {
        let MustConsume { coin } = res;  // Destructure (consumes it)
        // Do something with coin
        let _ = coin;
    }
    // Without the destructure, compiler error: "cannot drop MustConsume"
    
    // ============================================
    // Ability: key
    // Value can be stored in global storage
    
    struct GlobalConfig has key {
        admin: address,
    }
    
    // Only structs with 'key' can use borrow_global, move_to, move_from
    public fun get_admin(addr: address): address {
        borrow_global<GlobalConfig>(addr).admin
    }
    
    // ============================================
    // Ability: store
    // Value can be stored inside another struct
    
    struct Inner has store, copy, drop {
        value: u64,
    }
    
    struct Outer has key {
        inner: Inner,  // Works because Inner has store
    }
    
    // Abilities propagate: Outer's abilities depend on all its fields
    // If Inner didn't have store, Outer couldn't have key
}

// ============================================
// PHANTOM TYPE PARAMETERS
// Type parameter used only for type safety, not in data
// ============================================

// Coin<T> is parameterized by T (the currency type)
// but T doesn't appear in the struct's data
module phantom_demo {
    struct Coin<phantom CoinType> has key, store {
        value: u64,  // T doesn't appear here
    }
    
    // APT and USDC are separate types even though storage is just u64
    struct APT {}
    struct USDC {}
    
    public fun merge_coins(
        coin1: Coin<APT>,
        coin2: Coin<APT>,  // Same type! Can't accidentally merge APT + USDC
    ): Coin<APT> {
        let Coin { value: v1 } = coin1;
        let Coin { value: v2 } = coin2;
        Coin<APT> { value: v1 + v2 }
    }
    
    // This won't compile:
    // public fun wrong_merge(apt: Coin<APT>, usdc: Coin<USDC>): Coin<APT> {
    //     let Coin { value: v1 } = apt;
    //     let Coin { value: v2 } = usdc;  // ❌ type mismatch
    //     Coin<APT> { value: v1 + v2 }
    // }
}

// ============================================
// GENERICS WITH ABILITY CONSTRAINTS
// ============================================

module generics_demo {
    // Generic over any type that can be stored
    struct Wrapper<T: store> has key {
        inner: T,
    }
    
    // Generic function that only accepts resources (key + store)
    public fun wrap_resource<T: key + store>(resource: T): Wrapper<T> {
        Wrapper { inner: resource }
    }
    
    // Multiple type parameters
    struct Pair<T: copy + drop, U: copy + drop> has copy, drop {
        first: T,
        second: U,
    }
    
    public fun swap<T: copy + drop, U: copy + drop>(pair: Pair<T, U>): Pair<U, T> {
        Pair { first: pair.second, second: pair.first }
    }
}
```

---

## Memory Model & Ownership

```
MOVE MEMORY MODEL

OWNERSHIP RULES
  1. Every value has exactly one owner
  2. Ownership transfers on assignment (move semantics)
  3. Types with 'copy' ability can be duplicated
  4. Types without 'drop' must be explicitly consumed

REFERENCE RULES (Borrow Checker)
  1. At any time, either:
     - ONE mutable reference to a value, OR
     - ANY number of immutable references
     (But not both simultaneously)
  
  2. References cannot outlive their source
  
  3. Global borrows:
     - Can borrow_global and borrow_global_mut
     - Must release before another call to same address+type
     - Cannot have both immutable and mutable at same time

LIFETIMES IN MOVE
  
  struct A { x: u64 }
  
  fun demo() {
      let a = A { x: 1 };
      
      // Lifetime of ref_a: until reassignment of a
      let ref_a: &A = &a;     // a borrowed immutably
      
      // Cannot modify a while ref_a exists:
      // a.x = 2;  ❌ ERROR: a is borrowed
      
      let _value = ref_a.x;   // OK: read through ref
      
      // After ref_a is no longer used, can modify again:
      a.x = 2;  // ✓ OK: ref_a lifetime ended
  }

GLOBAL MEMORY LAYOUT (Aptos)

  Address Space: 256-bit (32 bytes)
  
  ┌─────────────────────────────────────┐
  │ 0x1 (Framework address)            │
  │   coin::CoinStore<AptosCoin>       │
  │   aptos_governance::GovernanceData  │
  │   ...                              │
  ├─────────────────────────────────────┤
  │ 0x3 (Token address)                │
  │   token::Collections               │
  │   ...                              │
  ├─────────────────────────────────────┤
  │ 0xUSER (User addresses)            │
  │   coin::CoinStore<...>             │
  │   YourModule::YourStruct           │
  │   ...                              │
  └─────────────────────────────────────┘

MOVE_TO vs MOVE_FROM
  move_to(account, resource)
    → Adds resource to account's address
    → Fails if resource already exists there
    
  move_from<T>(addr): T
    → Removes resource from addr
    → Fails if no resource of type T there
    → Returns the resource (ownership transferred to caller)
    
  borrow_global<T>(addr): &T
    → Immutable reference to resource at addr
    → Resource stays in global storage
    
  borrow_global_mut<T>(addr): &mut T
    → Mutable reference to resource at addr
    → Can modify resource through reference
```

---

## Aptos vs Sui VM Differences

```
APTOS vs SUI: KEY ARCHITECTURAL DIFFERENCES

STORAGE MODEL
  Aptos:
    Resources stored at addresses (account-based)
    address → StructType → Value
    One struct per type per address
    Access: borrow_global<T>(addr), move_to(signer, T)
    
  Sui:
    Objects have unique IDs (object-centric)
    object_id → (owner, version, type, value)
    Multiple objects of same type can exist
    Access: via transaction input, no borrow_global

OWNERSHIP TYPES
  Aptos:
    Resources at address (owned by that address)
    Shared: not built into language, app-level
    
  Sui:
    Owned:   one owner (address or object)
    Shared:  accessible by anyone (needs consensus)
    Frozen:  immutable, shareable without consensus
    Wrapped: embedded in parent object

CONSENSUS IMPLICATIONS
  Aptos:
    All transactions touch shared state → serial ordering
    Parallel execution: transactions touching different addresses
    BlockSTM: optimistic parallel execution with conflict detection
    
  Sui:
    Owned objects: bypass consensus (fastpath, ~100ms)
    Shared objects: require consensus (slower, ~500ms)
    Design advice: prefer owned objects for low-latency ops

PROGRAMMABLE TRANSACTIONS
  Aptos:
    One function call per transaction
    Multiple messages = multiple transactions
    
  Sui:
    Programmable Transaction Blocks (PTB):
      Chain multiple operations in one atomic transaction
      Pass results between operations
      Batch operations on multiple objects
      Rich composition without new smart contracts

OBJECT IDs (Sui)
  Every object has unique UID:
    struct MyObject has key {
        id: UID,  // Required for key ability in Sui
        ...
    }
    
  UID created via object::new(ctx):
    let id = object::new(ctx);
    
  Objects referenced by ID in transactions (not by address)

MODULE ADDRESSES
  Aptos:
    Module at: {address}::{module_name}::{function}
    Example: 0x1::coin::transfer
    
  Sui:
    Module at: {package_id}::{module_name}::{function}
    Package = collection of modules (published together)
    Example: 0xPKG::pool::swap

COIN STANDARDS
  Aptos:
    coin::Coin<T>: older standard
    fungible_asset::FungibleAsset: newer standard
    
  Sui:
    coin::Coin<T>: unified standard
    Coin is an object with UID + balance

ERROR HANDLING
  Aptos: abort(code) → integer error code
  Sui:   abort(code) + assert!() → same mechanism
  Both: no try/catch, transaction aborts on any error
  
EVENTS
  Aptos: emit with event handle objects
         event::EventHandle<T> stored in struct
  Sui:   event::emit<T>(event_data) → simpler
```

---

## Advanced Language Features

```move
// ============================================
// ADVANCED MOVE PATTERNS
// ============================================

// 1. FUNCTION ENTRY vs PUBLIC vs PUBLIC(FRIEND)
module advanced {
    
    // entry: Can be called in transactions
    // Public users can call this directly from a transaction
    public entry fun user_action(user: &signer, amount: u64) {
        let _ = (user, amount);
    }
    
    // public: Other modules can call this
    // Cannot be used as transaction entry point without entry modifier
    public fun compute_result(x: u64): u64 {
        x * 2
    }
    
    // public(friend): Only friend modules can call
    public(friend) fun internal_action(amount: u64) {
        let _ = amount;
    }
    
    // private: Only within this module
    fun helper(x: u64): u64 {
        x + 1
    }
    
    // friend declaration: who can use public(friend)
    friend protocol::orchestrator;
    
    // ============================================
    // 2. CLOSURES (not in Move, use structs instead)
    
    // Move doesn't have first-class functions or closures
    // Pattern: encode "function" as struct + call function with struct
    
    struct Filter has drop {
        min_amount: u64,
        max_amount: u64,
    }
    
    public fun apply_filter(filter: &Filter, amount: u64): bool {
        amount >= filter.min_amount && amount <= filter.max_amount
    }
    
    // Usage: pass Filter as "callback" parameter
    public fun process_many(
        amounts: &vector<u64>,
        filter: &Filter,
    ): vector<u64> {
        let result = vector::empty<u64>();
        let n = vector::length(amounts);
        let i = 0;
        while (i < n) {
            let amount = *vector::borrow(amounts, i);
            if (apply_filter(filter, amount)) {
                vector::push_back(&mut result, amount);
            };
            i = i + 1;
        };
        result
    }
    
    // ============================================
    // 3. INLINE FUNCTIONS (available in Move 2.0)
    
    // inline: Function body inlined at call site (no call overhead)
    // Useful for tiny functions called in tight loops
    inline fun clamp(x: u64, lo: u64, hi: u64): u64 {
        if (x < lo) lo
        else if (x > hi) hi
        else x
    }
    
    public fun safe_bps(bps: u64): u64 {
        clamp(bps, 0, 10_000)  // Inlined: no function call overhead
    }
    
    // ============================================
    // 4. DESTRUCTURING & PATTERN MATCHING
    
    struct Config has drop {
        fee: u64,
        admin: address,
        paused: bool,
    }
    
    // Named destructuring
    public fun use_config(config: Config): u64 {
        let Config { fee, admin: _, paused } = config;
        if (paused) 0 else fee
    }
    
    // Tuple returns
    public fun compute(): (u64, u64, bool) {
        (100, 200, true)
    }
    
    public fun use_tuple() {
        let (x, y, flag) = compute();
        let _ = (x, y, flag);
    }
    
    // ============================================
    // 5. SPECS AND FORMAL VERIFICATION
    
    spec module {
        // Module-level invariant
        // invariant total_fees_collected >= 0;
    }
    
    public fun add_safe(a: u64, b: u64): u64 {
        assert!(b <= (18_446_744_073_709_551_615u64 - a), 1);
        a + b
    }
    
    spec add_safe {
        pragma aborts_if_is_strict;
        aborts_if b > 18446744073709551615 - a;
        ensures result == a + b;
    }
    
    // ============================================
    // 6. MACROS (Move 2.0)
    
    // Move 2.0 introduces macro functions for code generation
    // Syntax: macro fun name($param: type): rettype { body }
    
    // Example: loop macro (conceptual, syntax may vary)
    // macro fun repeat($n: u64, $f: || ()) {
    //     let i = 0;
    //     while (i < $n) { $f(); i = i + 1; };
    // }
}

// ============================================
// STRUCT UPDATE SYNTAX
// ============================================

module struct_update {
    struct Config has copy, drop {
        fee: u64,
        admin: address,
        limit: u64,
    }
    
    public fun update_fee(config: Config, new_fee: u64): Config {
        // Update only fee, keep other fields
        Config { fee: new_fee, ..config }  // Struct update syntax
    }
}
```

---

## สรุป Move Language Internals

```
MOVE VM KEY INSIGHTS

1. SAFETY BY CONSTRUCTION (not by convention)
   - Resource safety enforced by VM, not programmer discipline
   - Type safety: no integer overflow on u256 (wrapping is explicit)
   - No null pointers: Option<T> is explicit
   - No use-after-free: ownership rules prevent it

2. ABILITY SYSTEM SUMMARY
   copy:  Can duplicate (= stack types in Rust)
   drop:  Can discard silently (= no destructor required)
   store: Can embed in other structs or global storage
   key:   Can be stored at top-level in global storage
   
   Combinations:
   copy+drop: Value types (u64, bool, address)
   copy+drop+store: Struct values (can go in tables)
   key+store: Resources in global storage
   store only: Fields of resources (can't live at top level)

3. OWNERSHIP vs REFERENCE
   Ownership moves when assigned (no implicit copy)
   References (&T, &mut T) allow temporary aliasing
   Borrow checker enforces: one writer XOR many readers

4. APTOS vs SUI CORE DIFFERENCE
   Aptos: Account-centric (resources at addresses)
          All code ultimately modifies account state
   Sui:   Object-centric (objects have IDs, owners)
          Owned objects bypass consensus for low latency

5. VERIFICATION LAYERS
   Source → Bytecode: Type checking, ability verification
   Bytecode → Execute: Stack safety, reference safety
   Prover (optional): Formal specification verification
   
   Three independent layers = defense in depth
```

---

**ก่อนหน้า**: [Part 96 - Economic Modeling ←](part-96-economic-modeling.md)
**ต่อไป**: [Part 98 - Real-World DeFi Case Studies →](part-98-case-studies.md)
