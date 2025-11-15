# Solana Development Environment Setup

**Document ID:** TECH-006  
**Version:** 1.0  
**Status:** Active  
**Owner:** Technical Lead  
**Category:** Core Technical / Foundation  
**Last Updated:** 2025-11-15

---

## Table of Contents

1. [Prerequisites & System Requirements](#1-prerequisites--system-requirements)
2. [Rust Toolchain Installation](#2-rust-toolchain-installation)
3. [Solana CLI Installation](#3-solana-cli-installation)
4. [Anchor Framework Setup](#4-anchor-framework-setup)
5. [Local Validator Configuration](#5-local-validator-configuration)
6. [Wallet Setup for Development](#6-wallet-setup-for-development)
7. [RPC Provider Configuration](#7-rpc-provider-configuration)
8. [IDE Setup (VS Code)](#8-ide-setup-vs-code)
9. [Testing Frameworks](#9-testing-frameworks)
10. [Project Dependencies](#10-project-dependencies)
11. [Environment Variable Management](#11-environment-variable-management)
12. [Git Workflow for Solana Projects](#12-git-workflow-for-solana-projects)
13. [Solana Explorer Usage](#13-solana-explorer-usage)
14. [Common Troubleshooting](#14-common-troubleshooting)
15. [Quick Start Verification](#15-quick-start-verification)

---

## Overview

This guide provides a production-ready setup for developing **DetourCoin**, a Solana-based SPL token with custom programs built using the Anchor framework. Follow these steps to configure your development environment for building, testing, and deploying Solana programs.

**Time Estimate:** 2-3 hours for complete setup

---

## 1. Prerequisites & System Requirements

### Operating System Compatibility

**Supported:**
- **Linux:** Ubuntu 20.04+, Debian 11+, Fedora 35+
- **macOS:** 12.0 (Monterey) or later
- **Windows:** WSL2 (Ubuntu 20.04+ recommended)

**Not Supported:**
- Native Windows (use WSL2 instead)
- macOS versions older than 12.0

### Hardware Requirements

**Minimum:**
- 8 GB RAM
- 50 GB free disk space
- Dual-core processor

**Recommended:**
- 16 GB RAM (local validator can be memory-intensive)
- 100 GB SSD storage
- Quad-core processor or better
- Stable internet connection (50+ Mbps)

### Required Prior Knowledge

- Basic Rust syntax and concepts (ownership, borrowing, traits)
- Command-line interface (CLI) proficiency
- Blockchain fundamentals (accounts, transactions, signatures)
- Basic cryptography concepts (public/private keys)
- Git version control

### Time Estimates by Section

- Rust installation: 15-20 minutes
- Solana CLI installation: 10-15 minutes
- Anchor framework: 10-15 minutes
- Local validator setup: 15-20 minutes
- Wallet configuration: 10 minutes
- RPC provider setup: 15 minutes
- IDE configuration: 20-30 minutes
- Testing and verification: 30-45 minutes

---

## 2. Rust Toolchain Installation

Solana programs are written in Rust. You'll need **Rust 1.70.0 or later** (recommend 1.75+).

### Installation via rustup

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

When prompted, select option `1` (default installation).

After installation, configure your shell:

```bash
source $HOME/.cargo/env
```

For permanent configuration, add to your shell profile (`~/.bashrc`, `~/.zshrc`):

```bash
export PATH="$HOME/.cargo/bin:$PATH"
```

### Verify Installation

```bash
rustc --version
cargo --version
rustup --version
```

**Expected output:**
```
rustc 1.75.0 (or higher)
cargo 1.75.0 (or higher)
rustup 1.26.0 (or higher)
```

### Install Required Components

```bash
rustup component add rustfmt clippy
```

**Verify components:**

```bash
rustfmt --version
cargo clippy --version
```

### Set Default Toolchain

```bash
rustup default stable
rustup update
```

### Common Installation Issues

**Issue:** `curl` not found
```bash
# Ubuntu/Debian
sudo apt-get update && sudo apt-get install curl

# macOS
# curl is pre-installed
```

**Issue:** Permission denied during installation
```bash
# Ensure you're NOT using sudo with rustup
# Run without sudo and install to user directory
```

**Issue:** `rustc` command not found after installation
```bash
# Reload shell or run:
source $HOME/.cargo/env
```

### Updating Rust Versions

```bash
rustup update stable
rustup default stable
```

⚠️ **Warning:** DetourCoin programs require Rust 1.70+ for Anchor 0.28+ compatibility.

💡 **Tip:** Run `rustup update` monthly to stay current with security patches.

---

## 3. Solana CLI Installation

The Solana CLI is essential for deploying programs, managing accounts, and interacting with the blockchain.

### Required Version

**Solana CLI 1.16.0 or later** (recommend 1.18+)

### Installation via Official Script

```bash
sh -c "$(curl -sSfL https://release.solana.com/v1.18.8/install)"
```

Add Solana to your PATH (add to `~/.bashrc` or `~/.zshrc`):

```bash
export PATH="$HOME/.local/share/solana/install/active_release/bin:$PATH"
```

Reload your shell:

```bash
source ~/.bashrc  # or source ~/.zshrc
```

### Alternative Installation (Package Managers)

**macOS (Homebrew):**
```bash
brew install solana
```

**Ubuntu/Debian (APT):**
```bash
wget -O - https://release.solana.com/v1.18.8/solana-release-x86_64-unknown-linux-gnu.tar.bz2 | tar jxf -
cd solana-release/
export PATH="$PWD/bin:$PATH"
```

### Verify Installation

```bash
solana --version
```

**Expected output:**
```
solana-cli 1.18.8
```

### Configure Network Endpoints

Set default network to devnet for development:

```bash
solana config set --url https://api.devnet.solana.com
```

**Available networks:**
- Devnet: `https://api.devnet.solana.com`
- Testnet: `https://api.testnet.solana.com`
- Mainnet-beta: `https://api.mainnet-beta.solana.com`
- Localhost: `http://localhost:8899`

Check current configuration:

```bash
solana config get
```

**Expected output:**
```
Config File: /home/user/.config/solana/cli/config.yml
RPC URL: https://api.devnet.solana.com
WebSocket URL: wss://api.devnet.solana.com/ (computed)
Keypair Path: /home/user/.config/solana/id.json
Commitment: confirmed
```

### Keypair Generation and Management

Generate a new keypair for development:

```bash
solana-keygen new --outfile ~/.config/solana/id.json
```

⚠️ **Warning:** This creates an **unencrypted** keypair file. Never use this for mainnet production funds.

**Display public key:**

```bash
solana-keygen pubkey ~/.config/solana/id.json
```

**Check account balance:**

```bash
solana balance
```

### SOL Airdrop for Devnet/Testnet

Request devnet SOL for testing:

```bash
solana airdrop 2
```

⚠️ **Note:** Airdrops are rate-limited. If you encounter errors, wait 5-10 minutes or use a faucet:
- Devnet: https://faucet.solana.com
- Testnet: https://faucet.solana.com

**Verify balance:**

```bash
solana balance
```

**Expected output:**
```
2 SOL
```

### Common Installation Issues

**Issue:** `solana: command not found`
```bash
# Ensure PATH is set correctly
echo $PATH | grep solana
# Should show: /home/user/.local/share/solana/install/active_release/bin

# If missing, add to ~/.bashrc:
export PATH="$HOME/.local/share/solana/install/active_release/bin:$PATH"
source ~/.bashrc
```

**Issue:** Airdrop fails with "Too many requests"
```bash
# Use web faucet: https://faucet.solana.com
# Or wait 10 minutes and retry
```

**Issue:** RPC connection timeout
```bash
# Try alternative RPC endpoints:
solana config set --url https://solana-devnet.g.alchemy.com/v2/demo
```

💡 **Tip:** Create separate keypairs for different environments (dev, test, prod).

---

## 4. Anchor Framework Setup

Anchor is a framework for Solana program development that simplifies common tasks.

### Required Version

**Anchor 0.28.0 or later** (recommend 0.29+)

### Installation via Cargo

Install Anchor version manager (avm):

```bash
cargo install --git https://github.com/coral-xyz/anchor avm --locked --force
```

Use avm to install Anchor:

```bash
avm install 0.29.0
avm use 0.29.0
```

### Verify Installation

```bash
anchor --version
```

**Expected output:**
```
anchor-cli 0.29.0
```

### Initialize Test Project

Verify Anchor works by creating a test project:

```bash
anchor init test_project
cd test_project
anchor build
```

**Expected output:**
```
...
Finished release [optimized] target(s) in 2m 30s
```

✅ If build succeeds, Anchor is correctly installed.

### Anchor.toml Configuration

Example `Anchor.toml` for DetourCoin:

```toml
[toolchain]

[features]
resolution = true
skip-lint = false

[programs.devnet]
detour_token = "DetourTokenProgramIDHere"
emission_controller = "EmissionControllerIDHere"
rbac_manager = "RBACManagerIDHere"
loyalty_program = "LoyaltyProgramIDHere"

[programs.testnet]
detour_token = "DetourTokenProgramIDHere"

[programs.mainnet-beta]
detour_token = "DetourTokenProgramIDHere"

[registry]
url = "https://api.apr.dev"

[provider]
cluster = "devnet"
wallet = "~/.config/solana/id.json"

[scripts]
test = "yarn run ts-mocha -p ./tsconfig.json -t 1000000 tests/**/*.ts"
```

### Common Installation Issues

**Issue:** `cargo install` fails with compilation errors
```bash
# Update Rust toolchain
rustup update stable
rustup default stable

# Retry installation
cargo install --git https://github.com/coral-xyz/anchor avm --locked --force
```

**Issue:** `avm: command not found`
```bash
# Ensure Cargo bin is in PATH
export PATH="$HOME/.cargo/bin:$PATH"
source ~/.bashrc
```

**Issue:** Anchor version mismatch errors
```bash
# Check installed version
avm list

# Switch to correct version
avm use 0.29.0

# Verify
anchor --version
```

💡 **Tip:** Use `avm` to manage multiple Anchor versions for different projects.

---

## 5. Local Validator Configuration

Running a local validator allows fast, free testing without network latency.

### Starting Local Validator

```bash
solana-test-validator
```

**Expected output:**
```
Ledger location: test-ledger
Log: test-ledger/validator.log
Identity: [PUBLIC_KEY]
Genesis Hash: [HASH]
Version: 1.18.8
Shred Version: [VERSION]
Gossip Address: 127.0.0.1:1024
TPU Address: 127.0.0.1:1027
JSON RPC URL: http://127.0.0.1:8899
...
```

⚠️ **Note:** This command blocks your terminal. Run in a separate terminal or use `&` to background:

```bash
solana-test-validator > validator.log 2>&1 &
```

### Configuration Options

**Reset state on restart:**
```bash
solana-test-validator --reset
```

**Custom slot time (faster blocks):**
```bash
solana-test-validator --slots-per-epoch 100
```

**Custom faucet limits:**
```bash
solana-test-validator --faucet-sol 1000
```

**Clone programs from devnet/mainnet:**
```bash
solana-test-validator \
  --clone TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA \
  --clone metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s
```

### Port Configuration and Conflicts

Default ports:
- **RPC:** 8899
- **WebSocket:** 8900
- **Gossip:** 1024

**Check if ports are in use:**
```bash
lsof -i :8899
```

**Kill existing validator:**
```bash
pkill solana-test-validator
```

### Configure Solana CLI for Local Validator

```bash
solana config set --url http://localhost:8899
```

**Verify connection:**

```bash
solana cluster-version
```

**Expected output:**
```
1.18.8
```

### Deploying to Local Validator

With Anchor:

```bash
anchor build
anchor deploy --provider.cluster localnet
```

With Solana CLI:

```bash
solana program deploy /path/to/program.so
```

### Stopping and Resetting Validator

**Stop validator:**
```bash
pkill solana-test-validator
```

**Reset ledger (fresh state):**
```bash
rm -rf test-ledger
solana-test-validator
```

### Log Locations and Debugging

**Log file location:**
```
test-ledger/validator.log
```

**Tail logs in real-time:**
```bash
tail -f test-ledger/validator.log
```

**View program logs:**
```bash
solana logs
```

💡 **Tip:** Use `solana logs | grep "Program log:"` to filter application logs.

---

## 6. Wallet Setup for Development

You'll need browser wallets to test frontend integrations and sign transactions.

### Phantom Wallet

**Installation:**
1. Visit https://phantom.app
2. Install browser extension (Chrome, Firefox, Brave, Edge)
3. Create new wallet or import existing

**Connect to Devnet:**
1. Click settings (gear icon)
2. Select "Change Network"
3. Choose "Devnet"

**Fund wallet:**
- Use https://faucet.solana.com with your Phantom public address

### Solflare Wallet

**Installation:**
1. Visit https://solflare.com
2. Install browser extension
3. Create new wallet or import

**Connect to Devnet:**
1. Click network dropdown (top right)
2. Select "Devnet"

### Backpack Wallet

**Installation:**
1. Visit https://backpack.app
2. Install browser extension
3. Create new wallet

**Connect to Devnet:**
1. Settings → Network
2. Select "Devnet"

### Managing Multiple Wallets for Testing

💡 **Best Practice:** Create separate wallets for different roles:
- **Admin wallet:** Program authority, upgrades
- **Merchant wallet:** Payment receiver testing
- **Customer wallet:** Token holder, spender
- **Test wallet:** Disposable for error scenarios

### Hardware Wallet Considerations

⚠️ **Mainnet Only:** Use hardware wallets (Ledger) for mainnet deployments.

**Ledger setup:**
1. Install Solana app on Ledger device
2. Use Phantom or Solflare with Ledger connection
3. Never use hardware wallet private keys in devnet/testnet

💡 **Tip:** Export Phantom private key to Solana CLI for testing:
```bash
# Phantom → Settings → Export Private Key
# Then import:
solana-keygen recover 'prompt:' --outfile ~/.config/solana/phantom.json
```

---

## 7. RPC Provider Configuration

Public RPC endpoints have rate limits. Use dedicated providers for production-like testing.

### Why External RPC Providers?

**Public RPC limitations:**
- Rate limits: 100-200 requests/minute
- Unreliable during high traffic
- No guaranteed uptime
- Slow historical data queries

**Provider benefits:**
- Higher rate limits (300K+ requests/day free tier)
- Better reliability and uptime
- Enhanced APIs (webhooks, NFT APIs)
- Archive node access

### QuickNode Setup

**1. Sign up:**
- Visit https://www.quicknode.com
- Create free account (300K requests/day)

**2. Create endpoint:**
- Dashboard → "Create an endpoint"
- Select "Solana"
- Choose network (devnet, mainnet-beta)
- Copy HTTP and WebSocket URLs

**3. Configure Solana CLI:**
```bash
solana config set --url https://your-endpoint.solana-devnet.quiknode.pro/YOUR_API_KEY/
```

**4. Test connection:**
```bash
solana cluster-version
```

### Helius Setup

**1. Sign up:**
- Visit https://www.helius.dev
- Create free account (100K requests/day)

**2. Create API key:**
- Dashboard → "API Keys" → "Create"
- Note your API key

**3. RPC URL format:**
```
Devnet: https://devnet.helius-rpc.com/?api-key=YOUR_API_KEY
Mainnet: https://mainnet.helius-rpc.com/?api-key=YOUR_API_KEY
```

**4. Configure:**
```bash
solana config set --url https://devnet.helius-rpc.com/?api-key=YOUR_API_KEY
```

### Alchemy (Solana)

**1. Sign up:**
- Visit https://www.alchemy.com
- Create free account (300M compute units/month)

**2. Create app:**
- Dashboard → "Create App"
- Select "Solana" → Choose network
- Copy HTTP URL

**3. Configure:**
```bash
solana config set --url https://solana-devnet.g.alchemy.com/v2/YOUR_API_KEY
```

### Free Tier Limitations

| Provider | Requests/Day (Free) | Networks | Notable Features |
|----------|---------------------|----------|------------------|
| QuickNode | 300,000 | All | Fastest response times |
| Helius | 100,000 | All | Enhanced transaction parsing |
| Alchemy | ~300M compute units | Mainnet, Devnet | NFT APIs included |

### RPC URL Configuration in Projects

**Anchor.toml:**
```toml
[provider]
cluster = "https://devnet.helius-rpc.com/?api-key=YOUR_API_KEY"
```

**JavaScript/TypeScript:**
```typescript
import { Connection } from '@solana/web3.js';

const connection = new Connection(
  process.env.RPC_URL || 'https://api.devnet.solana.com',
  'confirmed'
);
```

### Fallback RPC Strategies

```typescript
const RPC_ENDPOINTS = [
  'https://devnet.helius-rpc.com/?api-key=YOUR_API_KEY',
  'https://your-quicknode.solana-devnet.quiknode.pro/YOUR_API_KEY/',
  'https://api.devnet.solana.com', // Fallback to public
];

async function getConnectionWithFallback() {
  for (const endpoint of RPC_ENDPOINTS) {
    try {
      const connection = new Connection(endpoint, 'confirmed');
      await connection.getVersion();
      return connection;
    } catch (error) {
      continue;
    }
  }
  throw new Error('All RPC endpoints failed');
}
```

### Rate Limiting Best Practices

- **Implement exponential backoff** for failed requests
- **Cache responses** for frequently accessed data
- **Batch requests** when possible using `connection.getMultipleAccountsInfo()`
- **Monitor usage** via provider dashboards

💡 **Tip:** Use local validator for unit tests, devnet RPC for integration tests.

---

## 8. IDE Setup (VS Code)

VS Code provides the best Rust and Solana development experience.

### VS Code Installation

Download from https://code.visualstudio.com

### Required Extensions

Install via Extensions panel (`Ctrl+Shift+X`) or command line:

```bash
code --install-extension rust-lang.rust-analyzer
code --install-extension JScearcy.rust-doc-viewer
code --install-extension serayuzgur.crates
```

**Essential extensions:**
- **rust-analyzer** (`rust-lang.rust-analyzer`): Rust language server
- **Rust Doc Viewer** (`JScearcy.rust-doc-viewer`): View docs in editor
- **crates** (`serayuzgur.crates`): Cargo.toml dependency versions

**Recommended extensions:**
- **Better TOML** (`bungcip.better-toml`): Syntax for Anchor.toml
- **Error Lens** (`usernamehw.errorlens`): Inline error messages
- **GitLens** (`eamodio.gitlens`): Enhanced Git integration

### Rust-analyzer Configuration

Add to `.vscode/settings.json`:

```json
{
  "rust-analyzer.checkOnSave.command": "clippy",
  "rust-analyzer.cargo.features": "all",
  "rust-analyzer.procMacro.enable": true,
  "rust-analyzer.procMacro.attributes.enable": true,
  "rust-analyzer.completion.autoimport.enable": true,
  "rust-analyzer.inlayHints.typeHints.enable": true,
  "rust-analyzer.inlayHints.parameterHints.enable": true
}
```

### Workspace Settings

Create `.vscode/settings.json` in project root:

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "rust-lang.rust-analyzer",
  "[rust]": {
    "editor.formatOnSave": true,
    "editor.defaultFormatter": "rust-lang.rust-analyzer"
  },
  "[toml]": {
    "editor.formatOnSave": true,
    "editor.defaultFormatter": "tamasfe.even-better-toml"
  },
  "files.exclude": {
    "**/.anchor": true,
    "**/target": true,
    "**/node_modules": true
  },
  "rust-analyzer.linkedProjects": ["./Cargo.toml"],
  "rust-analyzer.cargo.target": "bpfel-unknown-unknown"
}
```

### Code Formatting Setup

**Auto-format on save** (already enabled above).

**Manual format:**
```bash
cargo fmt
```

**Lint with Clippy:**
```bash
cargo clippy -- -D warnings
```

### Debugging Configuration

Create `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "lldb",
      "request": "launch",
      "name": "Debug Anchor Tests",
      "cargo": {
        "args": ["test", "--no-fail-fast"],
        "filter": {
          "name": "detour_token",
          "kind": "lib"
        }
      },
      "args": [],
      "cwd": "${workspaceFolder}"
    }
  ]
}
```

⚠️ **Note:** Requires CodeLLDB extension for debugging.

### Terminal Setup

**Integrated terminal:**
- View → Terminal (`Ctrl+\``)
- Supports multiple terminals (one for validator, one for commands)

**Recommended terminal settings:**

```json
{
  "terminal.integrated.defaultProfile.linux": "bash",
  "terminal.integrated.scrollback": 10000,
  "terminal.integrated.fontSize": 14
}
```

💡 **Tip:** Create workspace tasks for common commands. See `.vscode/tasks.json`:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Anchor Build",
      "type": "shell",
      "command": "anchor build",
      "problemMatcher": [],
      "group": {
        "kind": "build",
        "isDefault": true
      }
    },
    {
      "label": "Anchor Test",
      "type": "shell",
      "command": "anchor test",
      "problemMatcher": []
    }
  ]
}
```

---

## 9. Testing Frameworks

### Anchor Built-in Testing

Anchor includes TypeScript-based testing with Mocha.

**Test file location:**
```
tests/detour_token.ts
```

**Run tests:**
```bash
anchor test
```

**Run specific test:**
```bash
anchor test --skip-local-validator -- --grep "initialize"
```

### solana-program-test

For Rust-native testing within program crates.

**Add to `programs/detour_token/Cargo.toml`:**

```toml
[dev-dependencies]
solana-program-test = "1.16"
solana-sdk = "1.16"
tokio = { version = "1.28", features = ["full"] }
```

**Example test:**

```rust
#[cfg(test)]
mod tests {
    use solana_program_test::*;
    use solana_sdk::{signature::Keypair, signer::Signer, transaction::Transaction};

    #[tokio::test]
    async fn test_initialize() {
        let program_id = Pubkey::new_unique();
        let mut program_test = ProgramTest::new(
            "detour_token",
            program_id,
            processor!(process_instruction),
        );
        
        let (mut banks_client, payer, recent_blockhash) = program_test.start().await;
        
        // Test logic here
    }
}
```

**Run Rust tests:**
```bash
cargo test --package detour_token
```

### Writing Tests for Solana Programs

**Key testing scenarios:**
- ✅ Successful instruction execution
- ✅ Account validation errors
- ✅ Authority checks
- ✅ Arithmetic overflow/underflow
- ✅ State transitions
- ✅ Cross-program invocations

### Running Tests Locally vs Devnet

**Local (faster, isolated):**
```bash
anchor test
```

**Devnet (realistic, slower):**
```bash
anchor test --skip-local-validator --provider.cluster devnet
```

### Test Coverage Tools

```bash
# Install cargo-tarpaulin (Linux only)
cargo install cargo-tarpaulin

# Generate coverage report
cargo tarpaulin --out Html --output-dir coverage
```

💡 **Tip:** Aim for 80%+ code coverage on critical paths (token transfers, authority checks).

---

## 10. Project Dependencies

### SPL Token Program Integration

**Program ID:** `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`

**Add to `Cargo.toml`:**

```toml
[dependencies]
spl-token = "4.0.0"
spl-associated-token-account = "2.2.0"
```

**TypeScript:**

```bash
npm install @solana/spl-token
```

### Token Metadata Program (Metaplex)

**Program ID:** `metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s`

**Add to `Cargo.toml`:**

```toml
[dependencies]
mpl-token-metadata = "4.1.0"
```

**TypeScript:**

```bash
npm install @metaplex-foundation/mpl-token-metadata
```

### Solana web3.js

```bash
npm install @solana/web3.js
```

### @solana/spl-token (npm)

```bash
npm install @solana/spl-token
```

### @coral-xyz/anchor (npm)

```bash
npm install @coral-xyz/anchor
```

### Version Compatibility Matrix

| Anchor | Solana CLI | Rust | SPL Token | Metaplex |
|--------|-----------|------|-----------|----------|
| 0.29.0 | 1.18.x    | 1.75+| 4.0.0     | 4.1.0    |
| 0.28.0 | 1.16.x    | 1.70+| 3.5.0     | 3.2.0    |

⚠️ **Warning:** Version mismatches can cause deployment failures or runtime errors.

### Installing Project Dependencies

**For new Anchor project:**

```bash
# Rust dependencies (auto-managed by Cargo)
cargo build

# TypeScript dependencies
npm install
```

💡 **Tip:** Lock dependency versions in `Cargo.toml` (use `=` instead of `^`):

```toml
spl-token = "=4.0.0"
```

---

## 11. Environment Variable Management

### .env File Structure

Create `.env` in project root:

```env
# Network configuration
SOLANA_NETWORK=devnet
ANCHOR_PROVIDER_URL=https://api.devnet.solana.com
ANCHOR_WALLET=/home/user/.config/solana/id.json

# RPC Configuration
RPC_URL=https://devnet.helius-rpc.com/?api-key=YOUR_API_KEY
RPC_WEBSOCKET_URL=wss://devnet.helius-rpc.com/?api-key=YOUR_API_KEY

# Program IDs (update after deployment)
DETOUR_TOKEN_PROGRAM_ID=
EMISSION_CONTROLLER_PROGRAM_ID=
RBAC_MANAGER_PROGRAM_ID=
LOYALTY_PROGRAM_PROGRAM_ID=

# Token Configuration
DETOUR_TOKEN_MINT=
DETOUR_TOKEN_DECIMALS=9

# Authority Keys
PROGRAM_AUTHORITY=
EMISSION_AUTHORITY=
RBAC_ADMIN=

# Frontend Configuration
NEXT_PUBLIC_RPC_URL=https://api.devnet.solana.com
NEXT_PUBLIC_NETWORK=devnet
```

### Required Environment Variables

**Core variables:**
- `ANCHOR_PROVIDER_URL`: RPC endpoint for Anchor CLI
- `ANCHOR_WALLET`: Path to keypair file
- `SOLANA_NETWORK`: Target network (devnet/testnet/mainnet-beta)

**Program identifiers:**
- `DETOUR_TOKEN_PROGRAM_ID`: Core token program
- `EMISSION_CONTROLLER_PROGRAM_ID`: Emission logic
- `RBAC_MANAGER_PROGRAM_ID`: Role-based access
- `LOYALTY_PROGRAM_PROGRAM_ID`: Loyalty mechanics

**RPC configuration:**
- `RPC_URL`: HTTP RPC endpoint
- `RPC_WEBSOCKET_URL`: WebSocket endpoint for subscriptions

### Using .env.example Templates

Create `.env.example` (commit to Git):

```env
# Copy this file to .env and fill in values
SOLANA_NETWORK=devnet
ANCHOR_PROVIDER_URL=https://api.devnet.solana.com
ANCHOR_WALLET=/path/to/keypair.json
RPC_URL=YOUR_RPC_URL_HERE
# ... (sensitive values removed)
```

**Setup for new developers:**

```bash
cp .env.example .env
# Edit .env with actual values
```

### Loading Environment Variables

**JavaScript/TypeScript (using dotenv):**

```bash
npm install dotenv
```

```typescript
import dotenv from 'dotenv';
dotenv.config();

const connection = new Connection(process.env.RPC_URL!, 'confirmed');
```

**Rust (using dotenvy):**

```toml
[dev-dependencies]
dotenvy = "0.15"
```

```rust
use dotenvy::dotenv;

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_with_env() {
        dotenv().ok();
        let rpc_url = std::env::var("RPC_URL").unwrap();
        // ...
    }
}
```

### Secrets Management Best Practices

⚠️ **Never commit to Git:**
- Private keys (`.json` keypair files)
- RPC API keys
- Mnemonics/seed phrases
- Production `.env` files

✅ **Do commit:**
- `.env.example` (template with dummy values)
- Public keys / program IDs
- Network configuration (devnet/testnet)

💡 **Production secrets management:**
- Use secret managers (AWS Secrets Manager, HashiCorp Vault)
- Environment variables in CI/CD (GitHub Secrets, Vercel env)
- Hardware wallets for signing authority

### Never Committing Private Keys

Add to `.gitignore`:

```gitignore
# Keypairs and secrets
*.json
!tsconfig.json
!package.json
.env
.env.local
.env.*.local
keypairs/
```

---

## 12. Git Workflow for Solana Projects

### .gitignore Template for Anchor Projects

Create `.gitignore` in project root:

```gitignore
# Anchor
.anchor
target/
**/*.rs.bk
test-ledger/

# IDL
*.json
!tsconfig.json
!package.json
!package-lock.json

# Environment
.env
.env.local
.env.*.local

# Node
node_modules/
dist/
.npm
.yarn

# Keypairs (NEVER commit)
keypairs/
*.json
id.json

# Logs
*.log
validator.log

# IDEs
.vscode/
.idea/
*.swp
*.swo
.DS_Store

# Build artifacts
build/
out/
```

### What to Commit vs Exclude

**✅ Commit:**
- Source code (`programs/`, `tests/`)
- Configuration templates (`.env.example`, `Anchor.toml`)
- Package manifests (`Cargo.toml`, `package.json`)
- Documentation (`README.md`, `docs/`)
- CI/CD configs (`.github/workflows/`)
- Public program IDs (after deployment)

**❌ Exclude:**
- Build artifacts (`target/`, `.anchor/`)
- Keypair files (`*.json` wallets)
- Environment variables (`.env`)
- Test ledgers (`test-ledger/`)
- Dependencies (`node_modules/`)
- Logs (`*.log`)

### Branch Strategy Recommendations

**Recommended workflow:**

```
main (production mainnet)
├── staging (testnet)
│   ├── develop (devnet)
│   │   ├── feature/token-emission
│   │   ├── feature/loyalty-system
│   │   └── bugfix/transfer-validation
```

**Branch naming:**
- `feature/description` - New features
- `bugfix/description` - Bug fixes
- `hotfix/description` - Critical production fixes
- `release/v1.2.0` - Release preparation

### Commit Message Conventions

Use conventional commits:

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `test`: Test additions/changes
- `refactor`: Code refactoring
- `chore`: Build/tooling changes

**Examples:**

```bash
git commit -m "feat(token): implement emission controller instruction"
git commit -m "fix(rbac): validate role assignment authority"
git commit -m "test(loyalty): add integration tests for reward claims"
```

### Pre-commit Hooks

Install pre-commit hooks to enforce formatting:

Create `.git/hooks/pre-commit`:

```bash
#!/bin/bash

echo "Running pre-commit checks..."

# Format Rust code
cargo fmt --all -- --check
if [ $? -ne 0 ]; then
  echo "❌ Rust formatting check failed. Run 'cargo fmt' to fix."
  exit 1
fi

# Lint Rust code
cargo clippy --all-targets --all-features -- -D warnings
if [ $? -ne 0 ]; then
  echo "❌ Clippy linting failed. Fix warnings before committing."
  exit 1
fi

# Check for private keys in staged files
if git diff --cached --name-only | grep -E '\\.json$' | grep -v -E '(package|tsconfig|package-lock)\\.json$'; then
  echo "⚠️  WARNING: JSON files detected (possible keypairs). Verify before committing."
  read -p "Continue? (y/n) " -n 1 -r
  echo
  if [[ ! $REPLY =~ ^[Yy]$ ]]; then
    exit 1
  fi
fi

echo "✅ Pre-commit checks passed"
```

Make executable:

```bash
chmod +x .git/hooks/pre-commit
```

💡 **Tip:** Use `husky` and `lint-staged` for Node.js projects to automate pre-commit hooks.

---

## 13. Solana Explorer Usage

### Navigating Solana Explorer

Visit: https://explorer.solana.com

**Switch networks:**
- Click network dropdown (top right)
- Select: Mainnet Beta / Devnet / Testnet / Custom RPC

### Viewing Transactions

**Search by:**
- Transaction signature (base58 string)
- Account address
- Block number
- Program ID

**Transaction details:**
- Status (success/failed)
- Block confirmation
- Fee paid (SOL)
- Instructions executed
- Logs output

### Viewing Programs

**Search program ID:**
- Enter program address in search bar
- View executable data account
- See deployment slot and authority

**Deployed programs:**
- Authority: Who can upgrade
- Executable: Whether program is deployed
- Data length: Program binary size

### Debugging Failed Transactions

**1. Find transaction signature:**
```bash
anchor deploy
# Copy transaction signature from output
```

**2. Open in Explorer:**
```
https://explorer.solana.com/tx/[SIGNATURE]?cluster=devnet
```

**3. Analyze failure:**
- Check "Program Log" section
- Look for custom error codes
- Identify failing instruction

**Common errors:**
- `custom program error: 0x1` → See program error enum
- `insufficient funds` → Not enough SOL for transaction
- `invalid account data` → Account not initialized or wrong type

### Reading Program Logs

**Log format:**

```
Program log: Instruction: Initialize
Program log: Token mint: [ADDRESS]
Program 11111111111111111111111111111111 invoke [1]
Program 11111111111111111111111111111111 success
```

**Filter logs by program:**
```bash
solana logs --url devnet | grep "Program DetourToken"
```

### Monitoring Account Changes

**Watch account in real-time:**

```bash
solana account [ADDRESS] --url devnet --output json-compact
```

**Subscribe to account changes:**

```typescript
connection.onAccountChange(
  publicKey,
  (accountInfo) => {
    console.log('Account updated:', accountInfo);
  },
  'confirmed'
);
```

💡 **Tip:** Bookmark frequently used accounts (token mint, program IDs) in Explorer.

---

## 14. Common Troubleshooting

### "Insufficient funds" Errors

**Symptoms:**
```
Error: insufficient lamports [AMOUNT], need [REQUIRED]
```

**Solutions:**

```bash
# Check balance
solana balance

# Airdrop more SOL (devnet/testnet)
solana airdrop 2

# Use web faucet if rate-limited
# https://faucet.solana.com
```

### RPC Connection Timeouts

**Symptoms:**
```
Error: 504 Gateway Timeout
Error: FetchError: request to https://api.devnet.solana.com failed
```

**Solutions:**

```bash
# Switch to alternative RPC
solana config set --url https://devnet.helius-rpc.com/?api-key=YOUR_KEY

# Or use QuickNode/Alchemy
# Check RPC provider dashboard for outages
```

### Program Compilation Errors

**Symptoms:**
```
error[E0308]: mismatched types
error[E0433]: failed to resolve: use of undeclared crate or module
```

**Solutions:**

```bash
# Update dependencies
cargo update

# Clean and rebuild
cargo clean
anchor build

# Check Anchor version compatibility
anchor --version
# Should match project requirements (0.28+)

# Verify Rust toolchain
rustc --version
# Should be 1.70+
```

### Anchor Version Mismatches

**Symptoms:**
```
Error: Anchor version mismatch
Expected: 0.29.0
Found: 0.28.0
```

**Solutions:**

```bash
# Check installed versions
avm list

# Install correct version
avm install 0.29.0
avm use 0.29.0

# Verify
anchor --version

# Update Anchor.toml if needed
```

### Local Validator Issues

**Symptoms:**
```
Error: unable to connect to validator
Error: port 8899 already in use
```

**Solutions:**

```bash
# Kill existing validator
pkill solana-test-validator

# Check port availability
lsof -i :8899

# Reset ledger state
rm -rf test-ledger
solana-test-validator

# Verify connection
solana cluster-version --url http://localhost:8899
```

### Wallet Connection Problems

**Symptoms:**
- Wallet doesn't appear in browser
- "User rejected request"
- Incorrect network

**Solutions:**

1. **Verify wallet is on correct network:**
   - Phantom: Settings → Change Network → Devnet
   - Solflare: Network dropdown → Devnet

2. **Clear browser extension data:**
   - Right-click extension → Manage → Clear storage

3. **Reconnect wallet in dApp:**
   - Disconnect → Refresh page → Reconnect

4. **Check browser console for errors:**
   - F12 → Console tab

### Transaction Simulation Failures

**Symptoms:**
```
Error: Transaction simulation failed: Blockhash not found
Error: Attempt to debit an account but found no record of a prior credit.
```

**Solutions:**

```bash
# Issue: Blockhash expired
# Use recent blockhash with shorter validity

# Issue: Account not funded
# Ensure all accounts have rent-exempt balance

# Issue: Wrong account owner
# Verify account is owned by correct program

# Debug with simulation logs
solana simulate [TRANSACTION_FILE] --url devnet
```

### Deployment Failures

**Symptoms:**
```
Error: Account allocation failed: insufficient lamports
Error: Program buffer [ADDRESS] has insufficient funds
```

**Solutions:**

```bash
# Ensure deployer wallet has enough SOL
solana balance
# Need ~10-20 SOL for program deployment

# Increase program size limit (if applicable)
solana program deploy program.so --max-len 200000

# Close unused buffers to reclaim SOL
solana program close [BUFFER_ADDRESS]
```

💡 **General debugging tips:**
- Check `solana logs` in real-time during testing
- Use `--verbose` flag for detailed error messages
- Review Anchor generated IDL for type mismatches
- Compare account discriminators (first 8 bytes)

---

## 15. Quick Start Verification

Use this checklist to verify your environment is correctly configured.

### Complete Setup Verification Checklist

#### ✅ Rust Toolchain

```bash
rustc --version
# Expected: rustc 1.75.0 (or higher)

cargo --version
# Expected: cargo 1.75.0 (or higher)

rustfmt --version
# Expected: rustfmt 1.x.x

cargo clippy --version
# Expected: clippy 0.1.x
```

#### ✅ Solana CLI

```bash
solana --version
# Expected: solana-cli 1.18.8 (or higher)

solana config get
# Expected: Config with valid RPC URL and keypair path

solana balance
# Expected: Balance (after airdrop) - e.g., 2 SOL
```

#### ✅ Anchor Framework

```bash
anchor --version
# Expected: anchor-cli 0.29.0 (or higher)

avm list
# Expected: List of installed versions with active marker (*)
```

#### ✅ Local Validator

```bash
# Terminal 1: Start validator
solana-test-validator

# Terminal 2: Check connection
solana cluster-version --url http://localhost:8899
# Expected: Version number (e.g., 1.18.8)
```

#### ✅ RPC Provider

```bash
solana config set --url [YOUR_RPC_URL]
solana cluster-version
# Expected: Should not timeout or error
```

#### ✅ Wallets

- [ ] Phantom installed and connected to devnet
- [ ] Funded with devnet SOL (check balance in wallet)
- [ ] Can sign test transactions

#### ✅ IDE (VS Code)

- [ ] rust-analyzer extension installed
- [ ] Code formatting works (`Shift+Alt+F`)
- [ ] No errors in extension output

### "Hello World" Anchor Program Test

**Step 1: Initialize project**

```bash
anchor init hello_detour
cd hello_detour
```

**Step 2: Build project**

```bash
anchor build
```

**Expected output:**
```
Compiling hello_detour v0.1.0
Finished release [optimized] target(s) in 2m 30s
```

**Step 3: Test locally**

```bash
anchor test
```

**Expected output:**
```
  hello_detour
    ✔ Is initialized! (250ms)

  1 passing (300ms)
```

✅ **Success!** Your environment is correctly configured.

### Step-by-Step Verification Commands

Run these commands in sequence:

```bash
# 1. Verify Rust
rustc --version && cargo --version

# 2. Verify Solana CLI
solana --version && solana config get

# 3. Verify Anchor
anchor --version

# 4. Check balance (should have SOL from airdrop)
solana balance

# 5. Create test project
anchor init verification_test
cd verification_test

# 6. Build
anchor build

# 7. Test
anchor test

# 8. Cleanup
cd ..
rm -rf verification_test
```

**If all commands succeed without errors:** ✅ Environment is ready!

### Expected Output Summary

| Step | Command | Expected Output |
|------|---------|-----------------|
| 1 | `rustc --version` | `rustc 1.75.0 (or higher)` |
| 2 | `solana --version` | `solana-cli 1.18.8 (or higher)` |
| 3 | `anchor --version` | `anchor-cli 0.29.0 (or higher)` |
| 4 | `solana balance` | `2 SOL` (or higher) |
| 5 | `anchor build` | `Finished release [optimized]` |
| 6 | `anchor test` | `1 passing` |

### Next Steps After Successful Setup

1. **Clone DetourCoin repository:**
   ```bash
   git clone [REPO_URL]
   cd detourpay_coin
   ```

2. **Install project dependencies:**
   ```bash
   npm install
   cargo build
   ```

3. **Configure environment:**
   ```bash
   cp .env.example .env
   # Edit .env with your RPC URLs and keypairs
   ```

4. **Build DetourCoin programs:**
   ```bash
   anchor build
   ```

5. **Run DetourCoin tests:**
   ```bash
   anchor test
   ```

6. **Read related documentation:**
   - [TECH-001: Core Token Program](./TECH-001-core-token-program.md)
   - [TECH-005: Architecture Map](./TECH-005-architecture-map.md) *(if available)*
   - [TECH-007: Deployment Procedures](./TECH-007-deployment-procedures.md) *(if available)*

---

## Additional Resources

### Official Documentation

- **Solana Docs:** https://docs.solana.com
- **Anchor Book:** https://book.anchor-lang.com
- **SPL Token Docs:** https://spl.solana.com/token
- **Solana Cookbook:** https://solanacookbook.com

### Development Tools

- **Solana Explorer:** https://explorer.solana.com
- **Solana FM:** https://solana.fm (alternative explorer)
- **SolScan:** https://solscan.io (analytics and tracking)
- **Anchor Playground:** https://beta.solpg.io

### Community Resources

- **Solana StackExchange:** https://solana.stackexchange.com
- **Anchor Discord:** https://discord.gg/anchorlang
- **Solana Discord:** https://discord.com/invite/solana

### Learning Paths

- **Solana Bootcamp:** https://www.soldev.app/course
- **Buildspace Solana Course:** https://buildspace.so/solana
- **QuickNode Guides:** https://www.quicknode.com/guides/solana-development

---

## Related Documents

- **[TECH-001: Core Token Program](../01-core_technical/001-core-token-program.md)** - DetourCoin token implementation
- **[TECH-005: Architecture Map](../01-core_technical/005-architecture-map.md)** - System architecture overview *(if available)*
- **[TECH-007: Deployment Procedures](../02-deployment_ops/007-deployment-procedures.md)** - Deployment guide *(if available)*

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-11-15 | Technical Lead | Initial comprehensive setup guide |

---

## Feedback & Improvements

If you encounter issues not covered in this guide or have suggestions for improvements, please:

1. Open an issue in the repository
2. Contact the Technical Lead
3. Submit a pull request with corrections

---

**End of Document**
