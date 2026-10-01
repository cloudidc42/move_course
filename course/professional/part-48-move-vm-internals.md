# Part 48: Move VM Internals

## สารบัญ
- [Move VM Architecture](#move-vm-architecture)
- [Bytecode & Execution](#bytecode--execution)
- [Memory Model & Borrow Checker](#memory-model--borrow-checker)
- [Gas Metering](#gas-metering)
- [Type System Internals](#type-system-internals)
- [Writing VM-Aware Code](#writing-vm-aware-code)

---

## Move VM Architecture

```
Move VM Stack:

┌─────────────────────────────┐
│  Move Source Code (.move)    │
├─────────────────────────────┤
│  Move Compiler              │  Type checking, borrow checking
├─────────────────────────────┤
│  Move Bytecode              │  Stack-based instructions
├─────────────────────────────┤
│  Move VM (Interpreter)      │  Execute bytecode
├─────────────────────────────┤
│  Aptos Framework / Sui Framework │
├─────────────────────────────┤
│  MoveVM Runtime             │  Memory, gas, effects
├─────────────────────────────┤
│  Blockchain Node            │  Consensus, storage
└─────────────────────────────┘

Key Components:
  1. Loader: loads bytecode, verifies, links modules
  2. Verifier: static checks (safety, types, borrow)
  3. Interpreter: stack-based execution
  4. Memory Manager: resource allocation, move semantics
  5. Gas Meter: tracks and limits computation
  6. Storage Interface: reads/writes global state

Execution Model:
  - Stack machine (not register-based like EVM)
  - Values pushed/popped from operand stack
  - Local variables in frame
  - Functions = frames on call stack
```

---

## Bytecode & Execution

```
Move Bytecode Instructions:

Load/Store:
  LdU8, LdU16, LdU32, LdU64, LdU128, LdU256  - push constant
  LdTrue, LdFalse                              - boolean constants
  CopyLoc(idx)                                 - copy local variable
  MoveLoc(idx)                                 - move (consume) local
  StLoc(idx)                                   - store to local
  
Arithmetic:
  Add, Sub, Mul, Div, Mod      - u64/u128 arithmetic
  BitAnd, BitOr, Xor, Shl, Shr - bitwise
  Lt, Le, Gt, Ge, Eq, Neq     - comparison
  
Control Flow:
  Branch(offset)               - unconditional jump
  BrTrue(offset)               - jump if top of stack is true
  BrFalse(offset)              - jump if top of stack is false
  Ret                          - return (all returns values on stack)
  
Function Calls:
  Call(func_idx)               - call function, pop args, push results
  CallGeneric(func_idx, types) - generic call
  
Resources:
  MoveTo(struct_idx)           - store resource to global storage
  MoveFrom(struct_idx)         - load resource from global storage
  BorrowGlobal(struct_idx)     - borrow reference to resource
  BorrowGlobalMut(struct_idx)  - mutable borrow
  Exists(struct_idx)           - check if resource exists
  
Vectors:
  VecLen(type)     - vector length
  VecImm(type)     - immutable element borrow
  VecMut(type)     - mutable element borrow
  VecPushBack(type)
  VecPopBack(type)
  VecUnpack(type)
  
Struct:
  Pack(struct_idx)             - create struct from stack values
  Unpack(struct_idx)           - destructure struct to stack
  BorrowField(field_idx)       - borrow field reference
  BorrowFieldMut(field_idx)    - mutable field borrow
```

```
Example: Simple add function

Move source:
  public fun add(a: u64, b: u64): u64 { a + b }

Bytecode (simplified):
  CopyLoc(0)    // push 'a' (local 0)
  CopyLoc(1)    // push 'b' (local 1)
  Add           // pop two, push sum
  Ret           // return top of stack

Execution trace:
  Frame: locals = [a=5, b=3]
  Stack: []
  
  CopyLoc(0) → Stack: [5]
  CopyLoc(1) → Stack: [5, 3]
  Add        → Stack: [8]
  Ret        → return 8
```

---

## Memory Model & Borrow Checker

```
Move Memory Model:

Values in Move:
  Primitive:  u8, u16, u32, u64, u128, u256, bool, address
  Struct:     Collection of fields (can have abilities)
  Vector:     Dynamic array of same-type values
  Reference:  Pointer to value (& or &mut)
  
Key Invariants enforced by borrow checker:
  1. Single owner: each value has exactly one owner
  2. Move semantics: assigning/passing moves ownership
  3. Borrowing rules:
     - Multiple immutable borrows (&T) at once: OK
     - Only one mutable borrow (&mut T) at a time: OK
     - Cannot mix: if &mut T exists, no other borrows valid
  4. References don't outlive their referent
  
Abilities (control what VM allows):
  copy:  value can be duplicated (bytecode CopyLoc)
  drop:  value can be discarded (goes out of scope safely)
  store: value can be stored in global storage or other structs
  key:   value can be top-level resource (MoveFrom/MoveTo)
  
Without copy: must move (consume) value
Without drop: must explicitly handle/move_to
Without store: can't put in global storage or other structs
Without key: can't be top-level global resource
```

```move
module vm::memory_demo {
    
    // Demonstrates borrow checker rules
    
    struct MyStruct has copy, drop {
        value: u64,
    }
    
    struct UniqueResource has key {
        data: u64,
    }
    
    public fun borrow_rules_demo(s: MyStruct) {
        // Immutable borrows: multiple OK
        let ref1: &MyStruct = &s;
        let ref2: &MyStruct = &s;  // OK: two & refs
        let _v1 = ref1.value;
        let _v2 = ref2.value;
        
        // Drop ref1 and ref2 here implicitly
        
        // Mutable borrow: only one
        let ref_mut: &mut MyStruct = &mut s;  // Wait: s is not mut!
        // Actually needs: let mut s = s;
    }
    
    public fun move_semantics_demo(): UniqueResource {
        // Create on stack
        let r = UniqueResource { data: 42 };
        
        // Move into function: r is consumed here
        let r2 = consume_resource(r);  // r moved into fn
        // r is now invalid (can't use it)
        
        r2
    }
    
    fun consume_resource(r: UniqueResource): UniqueResource {
        // r is now owned by this function
        // We must either:
        // 1. Return it (transfer ownership back)
        // 2. move_to it to global storage
        // 3. Destructure it (only if fields are droppable)
        
        // Return: transfer ownership
        UniqueResource { data: r.data + 1 }
        // r is dropped here (wait: no drop ability!)
        // Actually: UniqueResource has key not drop
        // So we MUST return it or move_to it
    }
    
    // Phantom types: carry type info, no runtime overhead
    struct Token<phantom T> has key {
        amount: u64,
    }
    
    // T doesn't need to implement any ability
    // phantom T is purely compile-time label
    
    public fun mint_token<T>(amount: u64): Token<T> {
        Token { amount }
    }
    
    // T ensures type safety:
    // Token<APT> and Token<USDC> are distinct types
    // Can't mix them up at compile time
    public fun add_tokens<T>(a: Token<T>, b: Token<T>): Token<T> {
        let Token { amount: amt_a } = a;
        let Token { amount: amt_b } = b;
        Token { amount: amt_a + amt_b }
    }
}
```

---

## Gas Metering

```
Aptos Gas Model (v2):

Gas = Execution Gas + IO Gas + Storage Gas

Execution Gas: computation cost
  - Each bytecode instruction has a cost
  - Higher for complex operations (crypto, hashing)
  - Simple arithmetic: ~1 unit
  - SHA3-256: ~14 units
  - Table operations: ~10-50 units

IO Gas: reading/writing data
  - Read resource: ~cost per byte read
  - Write resource: ~cost per byte written
  - Event emission: ~cost per byte

Storage Gas: long-term storage
  - Per-byte cost for data stored on-chain
  - Ongoing cost (each transaction pays for storage it uses)
  - Refund when resources deleted

Transaction Fees:
  fee = max(min_price, gas_price) * gas_used
  
Gas optimizations:
  1. Minimize storage reads (cache in locals)
  2. Use smaller data types
  3. Batch operations to amortize fixed costs
  4. Delete resources when done (gas refund)
  5. Use events instead of storage for logs
  6. Lazy initialization (only create resources when needed)
```

```move
module vm::gas_optimization {
    use aptos_std::smart_table::{Self, SmartTable};
    
    // ============================================
    // Gas-aware code patterns
    // ============================================
    
    struct UserData has key {
        score: u64,
        last_update: u64,
        streak: u32,
        flags: u8,  // Pack multiple bool fields into u8
    }
    
    // BAD: Multiple storage reads (expensive)
    public fun update_user_bad(
        user_addr: address,
        data_addr: address,
    ) acquires UserData {
        // 3 separate storage reads!
        let score = borrow_global<UserData>(user_addr).score;
        let last = borrow_global<UserData>(user_addr).last_update;
        let streak = borrow_global<UserData>(user_addr).streak;
        
        let data = borrow_global_mut<UserData>(user_addr);
        data.score = score + 10;
        data.last_update = last + 1;
        data.streak = streak + 1;
    }
    
    // GOOD: Single storage read
    public fun update_user_good(
        user_addr: address,
    ) acquires UserData {
        let data = borrow_global_mut<UserData>(user_addr);
        // Read all fields from single reference
        data.score = data.score + 10;
        data.last_update = aptos_framework::timestamp::now_seconds();
        data.streak = data.streak + 1;
    }
    
    // ============================================
    // Bit packing: multiple fields in one u64
    // Saves storage slots → lower gas
    // ============================================
    
    // Pack: is_active (bit 0), is_verified (bit 1), tier (bits 2-3), flags (bits 4-7)
    public fun pack_flags(
        is_active: bool,
        is_verified: bool,
        tier: u8,       // 0-3
        extra: u8,      // 4 bits
    ): u8 {
        let mut flags = 0u8;
        if (is_active) flags = flags | 1;
        if (is_verified) flags = flags | 2;
        flags = flags | ((tier & 3) << 2);
        flags = flags | ((extra & 0xF) << 4);
        flags
    }
    
    public fun is_active(flags: u8): bool { (flags & 1) != 0 }
    public fun is_verified(flags: u8): bool { (flags & 2) != 0 }
    public fun get_tier(flags: u8): u8 { (flags >> 2) & 3 }
    
    // ============================================
    // Lazy initialization: create resource only when needed
    // ============================================
    
    struct LazyUserProfile has key {
        preferences: SmartTable<vector<u8>, vector<u8>>,
        created_at: u64,
    }
    
    fun ensure_profile_exists(user: &signer) {
        let addr = std::signer::address_of(user);
        if (!exists<LazyUserProfile>(addr)) {
            move_to(user, LazyUserProfile {
                preferences: smart_table::new(),
                created_at: aptos_framework::timestamp::now_seconds(),
            });
        };
    }
    
    public entry fun set_preference(
        user: &signer,
        key: vector<u8>,
        value: vector<u8>,
    ) acquires LazyUserProfile {
        // Only create profile on first use
        ensure_profile_exists(user);
        
        let addr = std::signer::address_of(user);
        let profile = borrow_global_mut<LazyUserProfile>(addr);
        smart_table::upsert(&mut profile.preferences, key, value);
    }
    
    // ============================================
    // Delete resources to get storage refund
    // ============================================
    
    public entry fun cleanup_profile(
        user: &signer,
    ) acquires LazyUserProfile {
        let addr = std::signer::address_of(user);
        if (exists<LazyUserProfile>(addr)) {
            let LazyUserProfile { preferences, created_at: _ } 
                = move_from<LazyUserProfile>(addr);
            smart_table::destroy_empty(preferences);
            // Gas refund for freed storage!
        };
    }
    
    // ============================================
    // Batch operations: amortize fixed costs
    // ============================================
    
    // BAD: N transactions for N updates
    // entry fun update_one(user: &signer, value: u64) { ... }
    
    // GOOD: Single transaction for N updates
    public entry fun batch_update(
        admin: &signer,
        users: vector<address>,
        values: vector<u64>,
    ) acquires UserData {
        let n = std::vector::length(&users);
        assert!(n == std::vector::length(&values), 1);
        
        let mut i = 0u64;
        while (i < n) {
            let user = *std::vector::borrow(&users, i);
            let value = *std::vector::borrow(&values, i);
            
            if (exists<UserData>(user)) {
                let data = borrow_global_mut<UserData>(user);
                data.score = value;
            };
            i = i + 1;
        };
    }
}
```

---

## Type System Internals

```
Move Type System:

Base Types:
  u8, u16, u32, u64, u128, u256  - unsigned integers
  bool                            - boolean
  address                         - 32-byte blockchain address
  
Composite Types:
  struct { field: Type, ... }     - named collection
  vector<T>                       - dynamic array
  
Reference Types (not storable):
  &T                              - immutable reference
  &mut T                          - mutable reference
  
Generic Types:
  struct Foo<T> { value: T }     - parameterized type
  fun bar<T: store>(x: T): T    - constrained generics
  
Ability Constraints:
  T: copy    - T must have copy ability
  T: drop    - T must have drop ability
  T: store   - T must have store ability
  T: key     - T must have key ability
  Multiple: T: copy + drop + store
  
Type Inference:
  Move infers types in most contexts
  Explicit needed for: function type params, some literals
  
Phantom Type Parameters:
  struct Token<phantom T> { amount: u64 }
  - T not used in fields
  - phantom avoids ability constraints on T
  - T still enforces type safety at compile time
  
Nominal vs Structural Typing:
  Move uses nominal typing:
  struct A { x: u64 } and struct B { x: u64 } are DIFFERENT types
  Not interchangeable even with same structure
```

```move
module vm::type_system_demo {
    
    // ============================================
    // Demonstrating generics and constraints
    // ============================================
    
    // Generic container with store constraint
    struct Box<T: store> has key, store {
        value: T,
    }
    
    // Can store Box<u64> as resource (T=u64 has store)
    // Cannot Box<&u64> (references don't have store)
    
    public fun box_value<T: store + drop>(value: T): Box<T> {
        Box { value }
    }
    
    public fun unbox<T: store + drop>(b: Box<T>): T {
        let Box { value } = b;
        value
    }
    
    // ============================================
    // Phantom types for safe token math
    // ============================================
    
    struct Amount<phantom Currency> has copy, drop, store {
        raw: u64,
    }
    
    struct USD {}
    struct EUR {}
    struct APT {}
    
    public fun add_amounts<C>(a: Amount<C>, b: Amount<C>): Amount<C> {
        Amount { raw: a.raw + b.raw }
    }
    
    // Compile error: can't add USD + EUR
    // public fun bad() {
    //     let a = Amount<USD> { raw: 100 };
    //     let b = Amount<EUR> { raw: 100 };
    //     add_amounts(a, b);  // TYPE ERROR!
    // }
    
    // ============================================
    // Type-level state machine
    // ============================================
    
    // States as phantom types
    struct Locked {}
    struct Unlocked {}
    struct Expired {}
    
    struct Vault<phantom State> has key {
        amount: u64,
        unlock_time: u64,
    }
    
    // Only locked vaults can be unlocked
    public fun unlock(v: Vault<Locked>): Vault<Unlocked> {
        let now = aptos_framework::timestamp::now_seconds();
        assert!(now >= v.unlock_time, 1);
        let Vault { amount, unlock_time } = v;
        Vault<Unlocked> { amount, unlock_time }
    }
    
    // Only unlocked vaults can withdraw
    public fun withdraw(v: Vault<Unlocked>): u64 {
        let Vault { amount, unlock_time: _ } = v;
        amount
    }
    
    // Can't withdraw from Locked vault at compile time!
    // public fun try_bad_withdraw(v: Vault<Locked>): u64 {
    //     withdraw(v)  // TYPE ERROR: expected Vault<Unlocked>
    // }
    
    // ============================================
    // Recursive types (via vectors)
    // ============================================
    
    // Move doesn't allow direct recursive structs,
    // but can use vectors for tree-like structures
    
    struct TreeNode has store, drop {
        value: u64,
        children: vector<TreeNode>,  // OK: via vector
    }
    
    public fun tree_sum(node: &TreeNode): u64 {
        let mut sum = node.value;
        let n = std::vector::length(&node.children);
        let mut i = 0u64;
        while (i < n) {
            sum = sum + tree_sum(std::vector::borrow(&node.children, i));
            i = i + 1;
        };
        sum
    }
}
```

---

## Writing VM-Aware Code

```move
module vm::vm_aware_patterns {
    
    // ============================================
    // 1. Avoid deep recursion (stack overflow)
    // Move has fixed call stack depth
    // ============================================
    
    // BAD: Recursive fibonacci (stack overflow for large n)
    public fun fib_recursive(n: u64): u64 {
        if (n <= 1) return n;
        fib_recursive(n - 1) + fib_recursive(n - 2)  // Stack overflow for n > ~15
    }
    
    // GOOD: Iterative fibonacci
    public fun fib_iterative(n: u64): u64 {
        if (n == 0) return 0;
        let mut a = 0u64;
        let mut b = 1u64;
        let mut i = 1u64;
        while (i < n) {
            let c = a + b;
            a = b;
            b = c;
            i = i + 1;
        };
        b
    }
    
    // ============================================
    // 2. Understand copy costs
    // Copying large structs is expensive
    // ============================================
    
    struct LargeData has copy, drop {
        data: vector<u8>,  // Copying copies entire vector!
    }
    
    // BAD: Passes by value = copy entire struct
    public fun process_bad(data: LargeData): u64 {
        std::vector::length(&data.data)
    }
    
    // GOOD: Pass by reference = no copy
    public fun process_good(data: &LargeData): u64 {
        std::vector::length(&data.data)
    }
    
    // ============================================
    // 3. String operations (bytes, not char)
    // ============================================
    
    // Move strings are vector<u8> (UTF-8 bytes)
    public fun string_length(s: &std::string::String): u64 {
        std::string::length(s)  // Returns byte length, not char count!
    }
    
    public fun concat_strings(a: std::string::String, b: &std::string::String): std::string::String {
        std::string::append(&mut a, *b);  // In-place modification
        a
    }
    
    // ============================================
    // 4. Error handling patterns
    // ============================================
    
    // Move uses abort codes, not exceptions
    // BEST PRACTICE: Define error codes as constants
    
    const E_NOT_INITIALIZED: u64 = 1;
    const E_ALREADY_EXISTS: u64 = 2;
    const E_OVERFLOW: u64 = 3;
    const E_UNAUTHORIZED: u64 = 4;
    
    // Detailed error codes with category encoding:
    // High 4 bits: module (0-15)
    // Low 28 bits: specific error
    const MODULE_ID: u64 = 1;  // This module's ID
    
    public fun make_error(local_code: u64): u64 {
        (MODULE_ID << 28) | (local_code & 0x0FFFFFFF)
    }
    
    // ============================================
    // 5. Test infrastructure awareness
    // ============================================
    
    #[test_only]
    struct TestState has key {
        initialized: bool,
    }
    
    #[test]
    fun test_with_proper_setup() {
        // Create test accounts (test framework only)
        let admin = aptos_framework::account::create_account_for_test(@0xADMIN);
        
        // Setup timestamp for tests
        let framework = aptos_framework::account::create_account_for_test(@0x1);
        aptos_framework::timestamp::set_time_has_started_for_testing(&framework);
        aptos_framework::timestamp::update_global_time_for_test_secs(1_000_000);
        
        // Run test
        assert!(aptos_framework::timestamp::now_seconds() == 1_000_000, 1);
    }
    
    // ============================================
    // 6. Module initialization pattern
    // ============================================
    
    // init_module: called automatically once when module published
    // (Aptos-specific feature)
    
    struct ModuleState has key {
        admin: address,
        version: u64,
    }
    
    // This runs automatically at publish time
    fun init_module(deployer: &signer) {
        move_to(deployer, ModuleState {
            admin: std::signer::address_of(deployer),
            version: 1,
        });
    }
    
    // ============================================
    // 7. Friend functions for internal APIs
    // ============================================
    
    // friend module vm::other_module;
    
    // public(friend) functions: callable only by friend modules
    // More restrictive than public, less restrictive than private
    
    public(friend) fun internal_transfer(amount: u64): bool {
        // Only friend modules can call this
        amount > 0
    }
}
```

---

## สรุป Move VM Internals

```
Key Takeaways:

1. Stack Machine
   - Values pushed/popped during execution
   - Locals stored in frame
   - Deep recursion → stack overflow → use iteration

2. Linear Types = Safety
   - Each value owned exactly once
   - Borrow checker proves safety at compile time
   - No garbage collection needed
   - No use-after-free, no double-free possible

3. Abilities = Expressive Constraints
   - copy/drop/store/key control what VM permits
   - No ability = must explicitly handle resource
   - phantom types = type safety without runtime overhead

4. Gas Model = Cost Awareness
   - Read/write storage is expensive
   - Computation relatively cheap
   - Batch operations, lazy init, reference passing = savings

5. Formal Verification Integration
   - Move Prover uses same type info
   - Spec functions can reference any field
   - Invariants checked at function boundaries
   
6. Module System
   - Modules are namespaced, not hierarchical
   - friend relationships for limited access
   - init_module for automatic initialization

Understanding the VM makes you write:
  - More efficient code (avoid unnecessary copies/reads)
  - Safer code (exploit ability system correctly)
  - More testable code (proper test setup)
```

---

**ก่อนหน้า**: [Part 47 - Game Theory ←](part-47-game-theory.md)
**ต่อไป**: [Part 49 - Sui Object Runtime Deep Dive →](part-49-sui-object-runtime.md)
