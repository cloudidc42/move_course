# Part 02: การติดตั้ง Development Environment

## สารบัญ
- [ภาพรวม Development Stack](#ภาพรวม-development-stack)
- [ติดตั้ง Aptos CLI](#ติดตั้ง-aptos-cli)
- [ติดตั้ง Sui CLI](#ติดตั้ง-sui-cli)
- [ติดตั้ง VS Code Extensions](#ติดตั้ง-vs-code-extensions)
- [สร้างโปรเจกต์แรก (Aptos)](#สร้างโปรเจกต์แรก-aptos)
- [สร้างโปรเจกต์แรก (Sui)](#สร้างโปรเจกต์แรก-sui)
- [ทดสอบ Environment](#ทดสอบ-environment)
- [Move Prover Setup](#move-prover-setup)
- [สรุป](#สรุป)

---

## ภาพรวม Development Stack

```
Development Environment
├── OS: Linux / macOS / Windows (WSL2)
├── Package Manager: Homebrew (macOS) / apt (Linux)
├── CLI Tools
│   ├── Aptos CLI - สำหรับ Aptos development
│   └── Sui CLI - สำหรับ Sui development
├── Editor
│   ├── VS Code (แนะนำ)
│   └── Extensions: Move Analyzer
├── Testing
│   ├── move test - unit tests
│   └── Move Prover - formal verification
└── Version Control: Git
```

---

## ติดตั้ง Aptos CLI

### macOS (Homebrew)

```bash
# ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง Aptos CLI
brew install aptos

# ตรวจสอบ version
aptos --version
# Output: aptos 3.x.x
```

### Linux (Binary)

```bash
# ดาวน์โหลด binary ล่าสุด
# ไปที่ https://github.com/aptos-labs/aptos-core/releases/latest
# แล้วดาวน์โหลดไฟล์ aptos-cli-VERSION-Ubuntu-x86_64.zip

# สำหรับ Ubuntu/Debian
wget https://github.com/aptos-labs/aptos-core/releases/download/aptos-cli-v3.5.0/aptos-cli-3.5.0-Ubuntu-22.04-x86_64.zip
unzip aptos-cli-3.5.0-Ubuntu-22.04-x86_64.zip
sudo mv aptos /usr/local/bin/
sudo chmod +x /usr/local/bin/aptos

# ตรวจสอบ
aptos --version
```

### Windows (WSL2 แนะนำ)

```powershell
# ติดตั้ง WSL2 ก่อน
wsl --install

# จากนั้นใช้ Ubuntu terminal และทำตาม Linux steps
```

### ทดสอบ Aptos CLI

```bash
# ดูคำสั่งทั้งหมด
aptos --help

# สร้าง account ใหม่ (สำหรับ testnet)
aptos init

# ตัวอย่าง output
# Configuring for profile default
# Choose network from [devnet, testnet, mainnet, local, custom | defaults to devnet]
devnet
# Enter your private key as a hex literal (0x...) [Current: None | No input: Generate new key (or keep one if present)]
# (กด Enter เพื่อ generate ใหม่)
# 
# Account <address> doesn't exist, creating it and funding it with 100000000 Octas
# Account <address> funded successfully
# ---
# Aptos CLI is now set up for account <address> as profile default!
```

---

## ติดตั้ง Sui CLI

### macOS (Homebrew)

```bash
# ติดตั้ง Sui
brew install sui

# ตรวจสอบ version
sui --version
# Output: sui x.xx.x
```

### Linux (Binary)

```bash
# ดาวน์โหลด binary
wget https://github.com/MystenLabs/sui/releases/download/mainnet-vX.X.X/sui-mainnet-vX.X.X-ubuntu-x86_64.tgz

# extract
tar -xvzf sui-mainnet-vX.X.X-ubuntu-x86_64.tgz

# ย้ายไปที่ PATH
sudo mv sui /usr/local/bin/

# ตรวจสอบ
sui --version
```

### Build from Source (ทุก OS)

```bash
# ต้องการ Rust ก่อน
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.bashrc

# Clone และ build
git clone https://github.com/MystenLabs/sui.git
cd sui
git checkout main
cargo install --locked --bin sui sui

# ตรวจสอบ
sui --version
```

### ทดสอบ Sui CLI

```bash
# สร้าง wallet ใหม่
sui client new-address ed25519

# เชื่อมต่อกับ testnet
sui client switch --env testnet

# ดู address ของเรา
sui client active-address

# ขอ testnet tokens
sui client faucet
```

---

## ติดตั้ง VS Code Extensions

### 1. Move Analyzer (สำหรับ Aptos)

```bash
# ติดตั้ง Move Analyzer binary ก่อน
# macOS
brew install move-analyzer

# Linux
cargo install --git https://github.com/move-language/move move-analyzer --locked

# จากนั้น Install extension ใน VS Code:
# Extensions → ค้นหา "Move" → ติดตั้ง "Move" by move-language
```

### 2. Move Syntax Highlighter (สำหรับ Sui)

```
VS Code Extensions ที่แนะนำ:
1. "Move" - Move Language Support
2. "Move Syntax" - Syntax highlighting
3. "Error Lens" - แสดง errors inline
4. "GitLens" - Git integration
5. "Thunder Client" - API testing
```

### การตั้งค่า VS Code สำหรับ Move

```json
// settings.json
{
    "[move]": {
        "editor.defaultFormatter": "move-language.move",
        "editor.formatOnSave": true,
        "editor.tabSize": 4
    },
    "move-analyzer.server.path": "/usr/local/bin/move-analyzer",
    "editor.rulers": [100],
    "files.associations": {
        "*.move": "move"
    }
}
```

---

## สร้างโปรเจกต์แรก (Aptos)

### Step 1: สร้างโปรเจกต์

```bash
# สร้าง directory
mkdir my_first_move_project
cd my_first_move_project

# Initialize Move project สำหรับ Aptos
aptos move init --name my_project

# โครงสร้างที่ได้:
# my_first_move_project/
# ├── Move.toml        ← configuration file
# └── sources/         ← โค้ด Move ของเรา
```

### Step 2: ดู Move.toml

```toml
# Move.toml - Configuration file
[package]
name = "my_project"
version = "1.0.0"
authors = ["Your Name"]

[addresses]
my_address = "0xCAFE"  # เปลี่ยนเป็น address จริงของเรา

[dependencies.AptosFramework]
git = "https://github.com/aptos-labs/aptos-core.git"
rev = "mainnet"
subdir = "aptos-move/framework/aptos-framework"
```

### Step 3: สร้าง Module แรก

```bash
# สร้างไฟล์ใหม่
touch sources/hello.move
```

```move
// sources/hello.move
module my_address::hello {
    use std::signer;
    use std::string::{Self, String};
    
    // Struct สำหรับเก็บ message
    struct MessageHolder has key {
        message: String,
    }
    
    // Error codes
    const E_NOT_INITIALIZED: u64 = 1;
    
    // Set message
    public entry fun set_message(
        account: &signer, 
        message: vector<u8>
    ) acquires MessageHolder {
        let addr = signer::address_of(account);
        let message_string = string::utf8(message);
        
        if (!exists<MessageHolder>(addr)) {
            move_to(account, MessageHolder {
                message: message_string
            });
        } else {
            let holder = borrow_global_mut<MessageHolder>(addr);
            holder.message = message_string;
        }
    }
    
    // Get message
    #[view]
    public fun get_message(addr: address): String acquires MessageHolder {
        assert!(exists<MessageHolder>(addr), E_NOT_INITIALIZED);
        borrow_global<MessageHolder>(addr).message
    }
}
```

### Step 4: Compile

```bash
# Compile โปรเจกต์
aptos move compile

# ถ้าสำเร็จจะเห็น:
# Compiling, may take a little while to download git dependencies...
# INCLUDING DEPENDENCY AptosFramework
# INCLUDING DEPENDENCY AptosStdlib
# INCLUDING DEPENDENCY MoveStdlib
# BUILDING my_project
# {
#   "Result": [
#     "my_address::hello"
#   ]
# }
```

### Step 5: Test

```bash
# รัน tests
aptos move test

# ถ้าไม่มี tests จะได้:
# BUILDING my_project
# Running Move unit tests
# Test result: OK. Total tests: 0; passed: 0; failed: 0
```

### Step 6: เพิ่ม Tests

```move
// sources/hello.move (เพิ่ม tests)
module my_address::hello {
    // ... code เดิม ...
    
    #[test(account = @my_address)]
    public fun test_set_message(account: &signer) acquires MessageHolder {
        set_message(account, b"Hello, Move!");
        let message = get_message(signer::address_of(account));
        assert!(message == string::utf8(b"Hello, Move!"), 0);
    }
    
    #[test(account = @my_address)]
    public fun test_update_message(account: &signer) acquires MessageHolder {
        set_message(account, b"First message");
        set_message(account, b"Updated message");
        let message = get_message(signer::address_of(account));
        assert!(message == string::utf8(b"Updated message"), 0);
    }
    
    #[test(account = @my_address)]
    #[expected_failure(abort_code = E_NOT_INITIALIZED)]
    public fun test_get_nonexistent(account: &signer) acquires MessageHolder {
        // ควร fail เพราะยังไม่ได้ set_message
        get_message(signer::address_of(account));
    }
}
```

```bash
# รัน tests อีกครั้ง
aptos move test

# ควรเห็น:
# Running Move unit tests
# [ PASS    ] my_address::hello::test_set_message
# [ PASS    ] my_address::hello::test_update_message
# [ PASS    ] my_address::hello::test_get_nonexistent
# Test result: OK. Total tests: 3; passed: 3; failed: 0
```

---

## สร้างโปรเจกต์แรก (Sui)

### Step 1: สร้างโปรเจกต์

```bash
# สร้าง Sui package
sui move new my_sui_project
cd my_sui_project

# โครงสร้าง:
# my_sui_project/
# ├── Move.toml
# ├── sources/
# │   └── my_sui_project.move  ← สร้างให้อัตโนมัติ
# └── tests/
```

### Step 2: ดู Move.toml สำหรับ Sui

```toml
# Move.toml สำหรับ Sui
[package]
name = "my_sui_project"
edition = "2024.beta"

[dependencies]
Sui = { git = "https://github.com/MystenLabs/sui.git", subdir = "crates/sui-framework/packages/sui-framework", rev = "framework/mainnet" }

[addresses]
my_sui_project = "0x0"  # จะถูกแทนที่เมื่อ deploy
```

### Step 3: แก้ไข Module

```move
// sources/my_sui_project.move
module my_sui_project::hello {
    use std::string::{Self, String};
    use sui::object::{Self, UID};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};
    
    // Object ใน Sui (แทน Resource ใน Aptos)
    public struct MessageObject has key, store {
        id: UID,
        message: String,
        owner: address,
    }
    
    // สร้าง message object และส่งให้ sender
    public entry fun create_message(
        message: vector<u8>,
        ctx: &mut TxContext
    ) {
        let sender = tx_context::sender(ctx);
        let message_obj = MessageObject {
            id: object::new(ctx),
            message: string::utf8(message),
            owner: sender,
        };
        
        // ส่ง object ให้ sender
        transfer::transfer(message_obj, sender);
    }
    
    // อัปเดต message
    public entry fun update_message(
        obj: &mut MessageObject,
        new_message: vector<u8>,
    ) {
        obj.message = string::utf8(new_message);
    }
    
    // ดู message
    public fun get_message(obj: &MessageObject): String {
        obj.message
    }
    
    // Getter สำหรับ owner
    public fun get_owner(obj: &MessageObject): address {
        obj.owner
    }
}
```

### Step 4: Build

```bash
# Build
sui move build

# Output:
# UPDATING GIT DEPENDENCY https://github.com/MystenLabs/sui.git
# INCLUDING DEPENDENCY Sui
# INCLUDING DEPENDENCY MoveStdlib
# BUILDING my_sui_project
# Build Successful
```

### Step 5: Test

```move
// tests/hello_tests.move
#[test_only]
module my_sui_project::hello_tests {
    use my_sui_project::hello;
    use sui::test_scenario::{Self as test, Scenario};
    use std::string;
    
    const USER: address = @0xCAFE;
    
    #[test]
    fun test_create_message() {
        let mut scenario = test::begin(USER);
        
        // สร้าง message
        test::next_tx(&mut scenario, USER);
        {
            let ctx = test::ctx(&mut scenario);
            hello::create_message(b"Hello, Sui!", ctx);
        };
        
        // ตรวจสอบว่า object ถูกสร้าง
        test::next_tx(&mut scenario, USER);
        {
            let obj = test::take_from_sender<hello::MessageObject>(&scenario);
            assert!(hello::get_message(&obj) == string::utf8(b"Hello, Sui!"), 0);
            assert!(hello::get_owner(&obj) == USER, 1);
            test::return_to_sender(&scenario, obj);
        };
        
        test::end(scenario);
    }
    
    #[test]
    fun test_update_message() {
        let mut scenario = test::begin(USER);
        
        test::next_tx(&mut scenario, USER);
        {
            let ctx = test::ctx(&mut scenario);
            hello::create_message(b"Original", ctx);
        };
        
        test::next_tx(&mut scenario, USER);
        {
            let mut obj = test::take_from_sender<hello::MessageObject>(&scenario);
            hello::update_message(&mut obj, b"Updated");
            assert!(hello::get_message(&obj) == string::utf8(b"Updated"), 0);
            test::return_to_sender(&scenario, obj);
        };
        
        test::end(scenario);
    }
}
```

```bash
# รัน tests
sui move test

# Output:
# Running Move unit tests
# [ PASS    ] my_sui_project::hello_tests::test_create_message
# [ PASS    ] my_sui_project::hello_tests::test_update_message
# Test result: OK. Total tests: 2; passed: 2; failed: 0
```

---

## ทดสอบ Environment

### Script ทดสอบ Aptos Environment

```bash
#!/bin/bash
# test_aptos_env.sh

echo "Testing Aptos Development Environment..."
echo "======================================="

# ตรวจสอบ Aptos CLI
if command -v aptos &> /dev/null; then
    echo "✅ Aptos CLI: $(aptos --version)"
else
    echo "❌ Aptos CLI: Not installed"
fi

# ตรวจสอบ Git
if command -v git &> /dev/null; then
    echo "✅ Git: $(git --version)"
else
    echo "❌ Git: Not installed"
fi

# ตรวจสอบ Profile
if aptos config show-profiles 2>/dev/null | grep -q "default"; then
    echo "✅ Aptos Profile: Configured"
else
    echo "⚠️  Aptos Profile: Not configured (run 'aptos init')"
fi

echo ""
echo "Environment check complete!"
```

```bash
chmod +x test_aptos_env.sh
./test_aptos_env.sh
```

### Script ทดสอบ Sui Environment

```bash
#!/bin/bash
# test_sui_env.sh

echo "Testing Sui Development Environment..."
echo "======================================"

# ตรวจสอบ Sui CLI
if command -v sui &> /dev/null; then
    echo "✅ Sui CLI: $(sui --version)"
else
    echo "❌ Sui CLI: Not installed"
fi

# ตรวจสอบ active address
if sui client active-address 2>/dev/null; then
    echo "✅ Sui Wallet: Configured"
else
    echo "⚠️  Sui Wallet: Not configured (run 'sui client new-address ed25519')"
fi

# ตรวจสอบ active network
NETWORK=$(sui client active-env 2>/dev/null)
if [ -n "$NETWORK" ]; then
    echo "✅ Active Network: $NETWORK"
else
    echo "⚠️  Network: Not configured"
fi

echo ""
echo "Environment check complete!"
```

---

## Move Prover Setup

Move Prover เป็นเครื่องมือสำหรับ formal verification ซึ่งต้องการ dependencies เพิ่มเติม

### ติดตั้ง Move Prover Dependencies

```bash
# macOS
brew install z3 cvc5

# Linux
# Z3
apt-get install z3
# หรือ build from source:
git clone https://github.com/Z3Prover/z3.git
cd z3
python3 scripts/mk_make.py
cd build && make -j$(nproc) && sudo make install

# ติดตั้ง boogie
dotnet tool install --global Boogie
```

### ทดสอบ Move Prover กับ Aptos

```move
// sources/verified_math.move
module my_address::verified_math {
    
    // Specification ที่ Prover จะตรวจสอบ
    spec module {
        pragma verify = true;
    }
    
    public fun add(a: u64, b: u64): u64 {
        a + b
    }
    
    // Specification สำหรับฟังก์ชัน add
    spec add {
        // ต้องไม่ overflow
        pragma aborts_if_is_strict;
        aborts_if a + b > MAX_U64;
        ensures result == a + b;
    }
    
    public fun safe_add(a: u64, b: u64): u64 {
        assert!(a <= 18446744073709551615 - b, 0);  // ป้องกัน overflow
        a + b
    }
    
    spec safe_add {
        pragma aborts_if_is_strict;
        aborts_if a > MAX_U64 - b;  // abort ถ้า overflow
        ensures result == a + b;    // result ต้องเท่ากับ a + b
    }
}
```

```bash
# รัน Prover
aptos move prove
```

---

## สรุป: Cheat Sheet สำหรับ Setup

### Aptos Commands

```bash
# Project management
aptos move init --name <name>    # สร้างโปรเจกต์ใหม่
aptos move compile               # compile code
aptos move test                  # รัน tests
aptos move prove                 # formal verification
aptos move publish               # deploy ไปยัง blockchain

# Account management
aptos init                       # setup account
aptos account list               # ดู accounts
aptos account balance            # ดู balance
aptos account fund-with-faucet   # ขอ test tokens

# Configuration
aptos config show-profiles       # ดู profiles ทั้งหมด
aptos config set-global-config   # ตั้งค่า global config
```

### Sui Commands

```bash
# Project management
sui move new <name>              # สร้างโปรเจกต์ใหม่
sui move build                   # build code
sui move test                    # รัน tests
sui client publish               # deploy ไปยัง blockchain

# Wallet management
sui client new-address ed25519   # สร้าง address ใหม่
sui client addresses             # ดู addresses
sui client balance               # ดู balance
sui client faucet                # ขอ test tokens

# Network
sui client switch --env testnet  # เปลี่ยน network
sui client active-env            # ดู network ปัจจุบัน
```

---

## Troubleshooting

### ปัญหาที่พบบ่อย

**1. Aptos CLI ไม่ทำงานหลัง install**
```bash
# แก้: เพิ่ม PATH
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

**2. Git dependency error ใน Move.toml**
```bash
# แก้: ตรวจสอบ internet connection และลอง อีกครั้ง
aptos move compile --skip-fetch-latest-git-deps

# หรือ cache locally
aptos move compile --dev
```

**3. Sui build fail - "package digest mismatch"**
```bash
# แก้: ลบ build cache
rm -rf build/
sui move build
```

**4. Test fail - "address not found"**
```toml
# แก้ Move.toml - ตรวจสอบ [addresses] section
[addresses]
my_address = "0xCAFE"  # ต้องตรงกับ module address
```

---

## แบบฝึกหัด

### Exercise 2.1: Setup Environment
1. ติดตั้ง Aptos CLI
2. รัน `aptos init` เพื่อสร้าง account บน devnet
3. ตรวจสอบ balance ด้วย `aptos account balance`

### Exercise 2.2: สร้าง Aptos Project
1. สร้างโปรเจกต์ใหม่ชื่อ `learning_move`
2. สร้าง module `basic_math` ที่มีฟังก์ชัน add, subtract, multiply, divide
3. เขียน tests สำหรับแต่ละฟังก์ชัน
4. รัน `aptos move test` และดูผลลัพธ์

### Exercise 2.3: สร้าง Sui Project
1. สร้าง Sui project ใหม่
2. แก้ไข module ให้สร้าง `Counter` object
3. เพิ่มฟังก์ชัน `increment`, `decrement`, `reset`
4. เขียน tests และรัน

### Exercise 2.4: ทดสอบ Deployment
1. Deploy โปรเจกต์ Aptos ไปยัง devnet
2. เรียก entry function ด้วย CLI
3. ตรวจสอบ transaction ใน Aptos Explorer

---

**ก่อนหน้า**: [Part 01 - แนะนำ Move Language](part-01-introduction.md)  
**ต่อไป**: [Part 03 - พื้นฐาน Syntax และ Types →](part-03-syntax-types.md)
