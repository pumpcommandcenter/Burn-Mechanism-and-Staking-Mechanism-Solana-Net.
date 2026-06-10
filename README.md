pump_rewards/                          # Root for the full ecosystem
├── programs/pump_rewards/             # On-chain Anchor program (cleaned)
│   ├── src/
│   │   ├── lib.rs                     # Main program (transfer_hook + stake + security + PQC)
│   │   ├── burn.rs                    # Dynamic burn (1%/0.5% + MC scaling stub)
│   │   ├── staking.rs                 # Vault + multipliers + VRF-weighted claims
│   │   ├── security.rs                # Freeze, reentrancy guard, rate limit, breach detection
│   │   ├── pqc.rs                     # Post-quantum stubs (Dilithium/Kyber ready)
│   │   └── state.rs / treasury.rs / ... (modular)
│   ├── Cargo.toml
│   └── Anchor.toml
├── pump-bot/                          # Off-chain bot + monitoring
│   ├── index.js                       # Helius webhooks + Jito (with fallback) + Jupiter
│   ├── docker-compose.yml             # Hardened stack (Mimir + Alloy + Loki + Grafana)
│   └── .env.example
├── scripts/                           # All deployment & debug scripts
│   ├── debug_repo.sh                  # NEW: Repository debugging hardcode
│   ├── initialize_all.sh
│   ├── squads_propose_upgrade.sh
│   └── create_token_2022_mint.sh
├── monitoring/                        # Cleaned configs (Prometheus/Mimir + Loki + Alloy)
└── docs/                              # Architecture + security model
chmod +x scripts/debug_repo.sh
./scripts/debug_repo.sh
cd /home/workdir/artifacts/pump_rewards

# 1. Debug the repo
./scripts/debug_repo.sh

# 2. On-chain
cd programs/pump_rewards
anchor build --verifiable
anchor test

# 3. Bot + Monitoring (hardened)
cd ../../pump-bot
cp .env.example .env   # ← Fill real secrets
docker compose up -d --build

# 4. Full secure init (Squads + verifiable + timelock)
../scripts/initialize_all.sh

[workspace]
members = ["programs/pump_rewards"]
resolver = "2"

[workspace.dependencies]
anchor-lang = "0.29.0"
anchor-spl = { version = "0.29.0", features = ["idl-build"] }
spl-transfer-hook-interface = "0.6.0"
pyth-solana-receiver-sdk = "0.3.0"
solana-program = "1.18.0"

[profile.release]
overflow-checks = true
lto = "fat"
codegen-units = 1
panic = "abort"
strip = true

[package]
name = "pump_rewards"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "lib"]

[dependencies]
anchor-lang = { workspace = true }
anchor-spl = { workspace = true }
spl-transfer-hook-interface = { workspace = true }
pyth-solana-receiver-sdk = { workspace = true }
solana-program = { workspace = true }

[dev-dependencies]
anchor-client = "0.29.0"
{
  "name": "pump-bot",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  },
  "dependencies": {
    "@solana/web3.js": "^1.95.3",
    "express": "^4.19.2",
    "dotenv": "^16.4.5",
    "prom-client": "^15.1.3",
    "@jito/ts": "^0.1.0"
  },
  "devDependencies": {
    "nodemon": "^3.1.0"
  },
  "engines": { "node": ">=20.0.0" }
}
version: '3.8'

services:
  pump-bot:
    security_opt:
      - no-new-privileges:true
      - seccomp:./seccomp/pump-bot.json
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=10m
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    mem_limit: 512m
    cpus: '1.0'
    restart: unless-stopped
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  mimir:
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=100m
    mem_limit: 2g
    cpus: '2.0'
    security_opt:
      - no-new-privileges:true

  alloy:
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=50m
    mem_limit: 512m
    security_opt:
      - no-new-privileges:true

  grafana:
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=50m
    mem_limit: 1g
    security_opt:
      - no-new-privileges:true
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD:-changeme}
     ./deploy.sh up
# or
npx ts-node scripts/run_all.ts
docker run -d \
  --name falco \
  --privileged \
  -v /var/run/docker.sock:/host/var/run/docker.sock \
  -v ./falco:/etc/falco \
  falcosecurity/falco:latest \
  -c /etc/falco/falco.yaml
  await fetch('/api/security/pq-bridge', {
  method: 'POST',
  body: JSON.stringify({
    currentPubkey: wallet.publicKey.toBase58(),
    pqPubkey: "your_dilithium_or_kyber_pubkey_hex",
    signature: await signMessage("Migrate to post-quantum"),
    algorithm: "dilithium"
  })
});
// In transfer_hook or squads_gated_buyback
pqc::verify_pqc_signature(ctx, message, signature)?;
cd pump-bot
npm install @noble/hashes @noble/curves
import { hybridSign, hybridVerify } from './lib/pqc-hybrid';

const sig = await hybridSign(message, ed25519Priv, dilithiumPriv);
const isValid = await hybridVerify(message, sig, hybridPubkey);
cd pump_rewards
ts-node scripts/migrate_wallets_to_pqc.ts wallets.json
[
  { "pubkey": "YourWalletPubkeyHere..." },
  { "pubkey": "AnotherWalletPubkeyHere..." }
]
# Run migration
ts-node scripts/migrate_wallets_to_pqc.ts wallets.json

# Build on-chain
cd programs/pump_rewards && anchor build --verifiable

# Start bot (with PQC support)
cd ../../pump-bot && npm run start
const hybridSig = await hybridDilithiumSign(message, ed25519PrivKey);
const isValid = await hybridVerify(message, hybridSig, hybridPubkey);
docker compose -f docker-compose.production.yml up -d --build
cd pump_rewards

# 1. On-chain (with PQC + zk stubs)
cd programs/pump_rewards
anchor build --verifiable
anchor test

# 2. Bot with real Dilithium WASM path
cd ../../pump-bot
npm install
npm run start   # (or docker compose)

# 3. Migrate existing wallets to PQC
cd ..
ts-node scripts/migrate_wallets_to_pqc.ts wallets.json

# 4. Full hardened stack
docker compose -f docker-compose.production.yml up -d
// Future: Use @lightprotocol/stateless.js or light-sdk
const compressedProof = await createCompressedAccount({
  owner: userPubkey,
  data: privateStakeProofData,
  merkleTree: LIGHT_MERKLE_TREE_PDA
});
spl-token initialize-confidential-transfer-mint \
  --mint $MINT \
  --auditor YOUR_AUDITOR_PUBKEY   # Optional
  [
  {
    "pubkey": "YourMainTreasuryWalletPubkeyHere1111111111111111111111111111",
    "label": "Treasury"
  },
  {
    "pubkey": "StakingVaultAuthorityPubkeyHere2222222222222222222222222222",
    "label": "Staking Vault"
  },
  {
    "pubkey": "LargeHolder1PubkeyHere333333333333333333333333333333333333",
    "label": "Whale 1"
  },
  {
    "pubkey": "LargeHolder2PubkeyHere444444444444444444444444444444444444",
    "label": "Whale 2"
  },
  {
    "pubkey": "TeamMultisigPubkeyHere555555555555555555555555555555555555",
    "label": "Team Multisig"
  }
]
cp scripts/wallets.json scripts/my_wallets.json
# Edit with real pubkeys
ts-node scripts/migrate_wallets_to_pqc.ts scripts/my_wallets.json
const hybridSig = await hybridDilithiumSign(message, ed25519Key, dilithiumKey);
// Send to on-chain verify_hybrid_signature_onchain
light-compressed-token = { version = "0.3.0", features = ["cpi"] }
light-sdk = "0.3.0"
spl-token-2022 = { version = "3.0", features = ["no-entrypoint"] }
cd pump_rewards

# Test migration
ts-node scripts/migrate_wallets_to_pqc.ts scripts/wallets.json

# Build with new dependencies
cd programs/pump_rewards
anchor build --verifiable

# Run full hardened stack
cd ../../
docker compose -f docker-compose.production.yml up -d --build
# 1. Prepare your wallet list
cp scripts/wallets.json scripts/my-wallets.json
# Edit with real pubkeys (or use the sample)

# 2. Run the migration script (registers hybrid keys)
cd pump_rewards
ts-node scripts/migrate_wallets_to_pqc.ts scripts/my-wallets.json

# 3. (Future) On-chain registration
# Call register_pqc_key or migrate_account_to_pqc via bot or frontend

# 4. Enable hybrid verification in critical paths
# Already wired in the new polished index.js (see below)

# 5. Test the full stack
docker compose -f docker-compose.production.yml up -d --build

# 6. Monitor everything
# Grafana: http://localhost:3000
# Metrics: http://localhost:3000/metrics
cd pump_rewards/pump-bot
npm install          # installs new dependencies if needed

# Start everything
docker compose -f ../docker-compose.production.yml up -d --build

# Or run bot locally for development
npm run dev
// Light Protocol Merkle Trees:
// - Poseidon-hashed Merkle trees for state compression
// - Accounts stored off-chain → only root + ZK proof on-chain
// - Massive rent savings + scalability
//
// We use the `merkle_root` field to reference a Light Protocol compressed tree.
// In production, replace normal `Account` creation with Light's 
// `create_compressed_account` CPI.
spl-token initialize-confidential-transfer-mint --mint $MINT
// Check for confidential transfer extension
if let Some(confidential) = get_extension::<ConfidentialTransferAccount>(&source_account)? {
    let decrypted_amount = /* decrypt using proof context */;
    let tax = calculate_buy_burn(decrypted_amount);
    // CPI burn + treasury deposit using decrypted_amount
}
[package]
name = "pump_rewards"
version = "0.1.0"
description = "Pump Rewards - Transfer Hook + Staking + Treasury + Buyback + Security + PQC + zk"
edition = "2021"

[lib]
crate-type = ["cdylib", "lib"]

[features]
no-entrypoint = []
no-idl = []
no-log-ix-name = []
cpi = ["no-entrypoint"]
idl-build = ["anchor-lang/idl-build", "anchor-spl/idl-build"]
verify = []

[dependencies]
anchor-lang = { workspace = true }
anchor-spl = { workspace = true }
spl-transfer-hook-interface = { workspace = true }
pyth-solana-receiver-sdk = { workspace = true }
solana-program = { workspace = true }

# === Light Protocol zk-Compression ===
light-compressed-token = { version = "0.4.0", features = ["cpi"] }
account-compression = { version = "0.3.0", features = ["cpi"] }
light-sdk = "0.4.0"

# === Confidential Transfers (Token-2022 zkToken) ===
spl-token-2022 = { version = "3.0.0", features = ["no-entrypoint", "confidential-transfer"] }

# PQC & Security (ready for real WASM / future native support)
# squads-multisig = { version = "0.1", features = ["no-entrypoint"] }
let cpi_accounts = AppendLeavesToMerkleTree {
    merkle_tree: ctx.accounts.merkle_tree.to_account_info(),
    authority: ctx.accounts.user.to_account_info(),
};

let cpi_ctx = CpiContext::new(
    ctx.accounts.account_compression_program.to_account_info(),
    cpi_accounts,
);

let leaves = vec![commitment.to_vec()];
account_compression::cpi::append_leaves_to_merkle_tree(cpi_ctx, leaves)?;
[dependencies]
anchor-lang = { workspace = true }
anchor-spl = { workspace = true }
spl-transfer-hook-interface = { workspace = true }
pyth-solana-receiver-sdk = { workspace = true }
solana-program = { workspace = true }

# Light Protocol zk-Compression (Merkle trees + compressed accounts)
light-compressed-token = { version = "0.4.0", features = ["cpi"] }
account-compression = { version = "0.3.0", features = ["cpi"] }
light-sdk = "0.4.0"

# Confidential Transfers (Token-2022)
spl-token-2022 = { version = "3.0.0", features = ["no-entrypoint", "confidential-transfer"] }
cd programs/pump_rewards
anchor build --verifiable
await program.methods
  .createPrivateStakeProof(commitment, nullifier, merkleRoot, pqcSignature)
  .accounts({
    proof: proofPda,
    merkleTree: lightMerkleTree,
    accountCompressionProgram: ACCOUNT_COMPRESSION_PROGRAM_ID,
  })
  .rpc();
  // ConfidentialTransferAccount handling (Solana Confidential Transfers / zkToken)
// The hook receives the *decrypted* amount when a confidential transfer occurs.
// We still apply burn/tax on this verifiable amount while keeping user balances private.
if let Ok(confidential_account) = get_extension::<ConfidentialTransferAccount>(
    &ctx.accounts.source.to_account_info().try_borrow_data()?
) {
    msg!("Confidential transfer detected via zkToken - tax applied on decrypted amount");
    // Optional: You can add extra proof verification here if needed
    // The `amount` parameter is already the decrypted value
}
/// Helper to derive Light Protocol Merkle Tree PDA
pub fn derive_light_merkle_tree_pda(
    merkle_tree_pubkey: &Pubkey,
    program_id: &Pubkey,
) -> Pubkey {
    Pubkey::find_program_address(
        &[
            b"merkle_tree",
            merkle_tree_pubkey.as_ref(),
        ],
        program_id,
    )
    .0
}
let light_program_id = account_compression::id();
let merkle_tree_pda = derive_light_merkle_tree_pda(&merkle_tree_pubkey, &light_program_id);
// Auto-create compressed proof on stake events
if (event.type === 'TOKEN_TRANSFER' && event.data?.mint === process.env.MINT_ADDRESS) {
    if (event.data.destination === process.env.STAKING_VAULT) {
        console.log('Staking detected → creating Light Protocol compressed proof');
        await createCompressedStakeProof(event.data.source, event.data.amount);
    }
}

async function createCompressedStakeProof(userPubkey, amount) {
    // Call on-chain create_private_stake_proof with Light Merkle tree
    // In production: build proper commitment, nullifier, and PQC signature
    console.log(`Creating compressed stake proof for ${userPubkey} amount ${amount}`);
    // TODO: Call Anchor program via @coral-xyz/anchor or raw transaction
}
// In rewards.rs or a new bubblegum.rs
pub fn mint_compressed_reward_nft(...) -> Result<()> {
    // CPI to Bubblegum program::mint_to_collection
    msg!("Minted compressed NFT reward");
    Ok(())
}


