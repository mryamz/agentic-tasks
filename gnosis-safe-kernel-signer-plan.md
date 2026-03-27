# Gnosis Safe as Kernel Signer — Production Feature + Demo

## Context

Plural's entity wallets are Kernel v3.1 smart accounts using `WeightedECDSAValidator` with Privy embedded wallets + hot wallet as signers. This is custodial. We're replacing it with a Gnosis Safe multisig as the Kernel's validator, using ZeroDev's `ERC1271Validator` (already deployed at `0xBF997aC6751e3aC5Ed7c22e358A1D536680Bb8e3` on Base Sepolia). The demo shows: (a) Safe-backed Kernel wallet working, (b) recovery scenario where Safe owners move funds after original key is lost.

## Key Discovery

- **`@zerodev/smart-account-validator`** (v5.4.1) on npm provides `signerToSmartAccountValidator`
- **ERC1271Validator contract** already deployed on Base Sepolia (verified: 8350 bytes)
- **No Solidity deployment needed** — we use the existing contract
- The validator delegates signature checks to a Safe via EIP-1271
- The ZeroDev SDK's `changeSudoValidator` can swap an existing Kernel wallet's validator

## Repos & Branches

| Repo | Branch | What changes |
|------|--------|-------------|
| plural-server | `teddy/0327/safe-kernel-signer` off main | Express sidecar: Safe deployment, validator swap, recovery ops |
| plural-platform | `teddy/0327/safe-kernel-signer` off main | New `@zerodev/smart-account-validator` dep, Safe management UI, Playwright demo |

## Implementation Plan

### Phase 1: Backend — Safe deployment + Kernel validator swap (plural-server)

#### 1a. Express sidecar endpoints (`js/src/zerodev/index.ts`)

Add new methods to the ZeroDev service class:

```typescript
// Deploy a Gnosis Safe with given owners/threshold, return Safe address
async deploySafe({ owners, threshold, chainId }): Promise<{ safeAddress: Address }>

// Swap a Kernel wallet's sudo validator from WeightedECDSA to ERC1271Validator
// backed by the given Safe address
async swapToSafeValidator({ kernelAddress, safeAddress, chainId }): Promise<{ txHash: string }>

// Execute a recovery: Safe owners sign a UserOp that transfers funds out
// of the Kernel wallet to a recovery address
async executeRecovery({ kernelAddress, safeAddress, recoveryAddress, tokenAddress, amount, chainId }): Promise<{ txHash: string }>
```

These use:
- `@safe-global/protocol-kit` for Safe deployment
- `@zerodev/smart-account-validator` + `signerToSmartAccountValidator` for validator creation
- `changeSudoValidator` from `@zerodev/sdk` for the swap
- `createKernelAccountClient` with the ERC1271 validator for recovery tx execution

#### 1b. Django views (`plural/chain/views/`)

New endpoints:
- `POST /chain/safe/deploy/<unified_id>/` — deploy a Safe for an entity
- `POST /chain/safe/swap-validator/<unified_id>/` — swap Kernel to Safe-backed validator
- `POST /chain/safe/recovery/<unified_id>/` — execute emergency fund recovery

#### 1c. Django models (`plural/chain/models/`)

New model `SafeWallet`:
- `entity` FK → Entity
- `safe_address` — the Gnosis Safe contract address
- `chain` FK → Chain
- `owners` — JSON list of owner addresses
- `threshold` — int
- `is_active_validator` — bool (is this Safe the current Kernel validator?)

### Phase 2: Frontend — Safe management + recovery UI (plural-platform)

#### 2a. Install dependency

```bash
bun add @zerodev/smart-account-validator @safe-global/protocol-kit
```

#### 2b. New hooks

- `use-safe-wallet.tsx` — CRUD for entity Safe wallets (fetch, deploy, swap validator)
- `use-safe-recovery.tsx` — execute recovery transaction via Safe

#### 2c. New UI components

- `components/safe/safe-settings.tsx` — Safe management in profile page (similar to passkey-settings.tsx)
  - Shows Safe address, owners, threshold
  - "Deploy Safe" button → deploys and links to entity
  - "Activate as Validator" button → swaps Kernel validator to Safe-backed
  - "Recovery" section → move funds when key is lost

#### 2d. API proxy routes

- `app/api/safe/deploy/route.ts`
- `app/api/safe/swap-validator/route.ts`
- `app/api/safe/recovery/route.ts`

### Phase 3: Playwright demo recording

#### 3a. Injected wallet provider

Build `ethereum-provider.bundle.js` — a self-contained EIP-1193 provider injected via `context.addInitScript()`:
- Signs with a private key (Safe owner)
- Forwards RPC to Alchemy
- Announces via EIP-6963 + `window.ethereum`
- Handles `eth_signTypedData_v4` for Safe's EIP-712 signatures
- Uses `@noble/curves` + `@noble/hashes` for browser-compatible signing

#### 3b. Demo spec (`gnosis-safe-demo.spec.ts`)

**Scene 1: Setup**
- Login as admin
- Navigate to profile
- Deploy a 2-of-2 Safe (hot wallet + demo owner)
- Activate Safe as Kernel validator

**Scene 2: Normal operation**
- Navigate to a transaction context
- Execute a transaction signed by the Safe (show it works)

**Scene 3: Recovery**
- Simulate key loss (the original Privy signer can no longer sign)
- Safe owners execute a fund recovery transaction
- Funds move to recovery address
- Show the tx on BaseScan

#### 3c. Output
- Video recorded at 1280x720, slowMo for readability
- ffmpeg → MP4 → `~/Desktop/gnosis-safe-demo.mp4`
- Push to `mryamz/agentic-tasks`

## Key Technical Details

**ERC1271Validator flow:**
```
UserOp → EntryPoint → Kernel.validateUserOp()
  → ERC1271Validator.validateUserOp(op, hash)
    → wrappedHash = EIP712(MessageHash(hash))
    → Safe.isValidSignature(wrappedHash, signature)
      → Safe checks owner ECDSA sigs against threshold
      → returns 0x1626ba7e (magic value)
    → returns SIG_VALIDATION_SUCCESS
```

**Validator swap:**
```typescript
// Current: WeightedECDSAValidator (Privy + hot wallet)
// New: ERC1271Validator → Safe multisig

const erc1271Validator = await signerToSmartAccountValidator(publicClient, {
  signer: safeOwnerAccount,     // EOA that owns the Safe
  smartAccountType: "SAFE",
  entryPoint: getEntryPoint("0.7"),
  kernelVersion: KERNEL_V3_1,
});

// Swap (requires current sudo access)
await changeSudoValidator(currentKernelClient, {
  sudoValidator: erc1271Validator,
});
```

**Recovery scenario:**
After the swap, even if the original Privy key is lost, the Safe owners can:
1. Create a new KernelAccountClient with the ERC1271Validator
2. Sign a UserOp that calls `kernel.execute(to, value, data)` to transfer funds
3. The Safe validates the signature via EIP-1271
4. Funds are moved to the recovery address

## Files to Create/Modify

### plural-server
- `js/src/zerodev/index.ts` — add Safe deploy, validator swap, recovery methods
- `js/package.json` — add `@safe-global/protocol-kit`, `@zerodev/smart-account-validator`
- `plural/chain/models/safe_models.py` — new SafeWallet model
- `plural/chain/models/__init__.py` — export
- `plural/chain/views/safe_views.py` — new API views
- `plural/chain/urls.py` — new URL patterns
- `plural/chain/migrations/0026_safewallet.py` — migration

### plural-platform
- `package.json` — add `@zerodev/smart-account-validator`
- `hooks/use-safe-wallet.tsx` — Safe CRUD hook
- `hooks/use-safe-recovery.tsx` — recovery hook
- `components/safe/safe-settings.tsx` — Safe management UI
- `app/api/safe/deploy/route.ts` — proxy
- `app/api/safe/swap-validator/route.ts` — proxy
- `app/api/safe/recovery/route.ts` — proxy
- `app/profile/page.tsx` — wire in SafeSettings component
- `.playwright/gnosis-safe/ethereum-provider.ts` — injected wallet
- `.playwright/gnosis-safe-demo.spec.ts` — demo spec
- `.playwright/gnosis-safe-demo.config.ts` — config

## Verification

1. Unit tests for Safe deployment + validator swap in Express sidecar
2. Django tests for new views
3. Playwright demo runs end-to-end and produces MP4
4. Safe visible at `app.safe.global/home?safe=base-sep:{address}`
5. Recovery tx visible on BaseScan
6. Original Privy signer confirmed unable to sign after swap (AA24 error)
