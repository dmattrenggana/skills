---
name: clanker
description: Deploy ERC20 tax tokens with the Clanker SDK (v5) on Base, BNB Chain, Monad, X Layer, Robinhood Chain, Arc, and Elysium. Use when the user wants to launch a token, set buy/sell taxes, split taxes between burn, dividends and an integrator, configure a dev buy, distribute trading fees to reward recipients, pay holder dividends, transfer creator rights, or update token metadata. Also covers managing existing V3/V4 tokens (vault claims, airdrops, presales, LP fee claims).
---

# Clanker SDK

Launch ERC20 tokens with Uniswap V4 liquidity pools, built-in taxes, and fee distribution through the official Clanker TypeScript SDK.

## Overview

- **V5 (current):** Tax-token launches on Base, BNB Chain, Monad, X Layer, Robinhood Chain, Arc, and Elysium. Taxes are set at launch and split between the protocol, burn, dividends, an integrator, and the creator.
- **V3/V4 (legacy):** Existing tokens stay manageable. Claims, reward updates, metadata updates, vault claims, airdrops, and running presales still work.
- From SDK v5, V3/V4 `deploy`, `deploySimulate`, `getDeployTransaction`, and `startPresale` **throw**. New launches go through `clanker-sdk/v5` only.

## Installation

```bash
npm install clanker-sdk viem
# or
yarn add clanker-sdk viem
# or
pnpm add clanker-sdk viem
```

- SDK v5 requires **viem 2.49.3 or newer**. viem is a peer dependency, so the project always uses its own viem copy. A pinned older viem fails to install.
- Types were checked against TypeScript 5.9 and 6.0.

```bash
# Check the installed version
npm view clanker-sdk version
```

## Quick Start (V5)

```typescript
import { Clanker } from 'clanker-sdk/v5';
import { createPublicClient, createWalletClient, http, isHex, type PublicClient } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';
import { base } from 'viem/chains';

const PRIVATE_KEY = process.env.PRIVATE_KEY;
if (!PRIVATE_KEY || !isHex(PRIVATE_KEY)) throw new Error('PRIVATE_KEY is required.');

const account = privateKeyToAccount(PRIVATE_KEY);
const publicClient = createPublicClient({
  chain: base,
  transport: http(),
}) as PublicClient;

const wallet = createWalletClient({
  account,
  chain: base,
  transport: http(),
});

const clanker = new Clanker({ wallet, publicClient });

const { txHash, waitForTransaction, error } = await clanker.deploy({
  name: 'My Token',
  symbol: 'TKN',
  image: 'ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi',
  tokenAdmin: account.address,
  metadata: { description: 'My awesome token' },
  context: { interface: 'Clanker SDK' },
});

if (error) throw error;

const result = await waitForTransaction();
if (result.error) throw result.error;

console.log('Token deployed at:', result.address);
console.log('Pool id:', result.poolId);
```

`wallet` and `publicClient` must use the same chain.

## V5 vs V4 (read before writing code)

| Topic | V5 behavior |
|-------|-------------|
| Import | `import { Clanker } from 'clanker-sdk/v5'` |
| Fees | `buyFee` and `sellFee`, each 1% by default, protocol share included. `fee` and `fees` are rejected. |
| Vanity | `vanity: true` is accepted with no effect. V5 always mines the `cccc` suffix. |
| Salt | `salt` is rejected. Each preparation uses a fresh salt. |
| Rewards | 1 to 7 slots, positive `bps` totalling 10,000. Use `Paired` or omit `token`. |
| Market cap | No per-launch start price or market cap input. The factory sets the start price per quote. |
| Rejected inputs | Vaults, airdrops, presales, pool positions, presets, custom ticks, sniper fees, lockers, pool extensions, and `Both` or `Clanker` fee preferences. |
| Address book | `clankerConfigFor` and `CLANKERS` have no V5 entry. Use `TAX_ADDRESSES` from `clanker-sdk/v5`. |
| Token type | Import `ClankerTokenV5` from `clanker-sdk/v5`, not from the package root. |
| Dev buy | `minAmountOut` (final token base units). The V4 `amountOutMin` name is rejected except as `0`. |

## Token Configuration (V5)

```typescript
{
  chainId: 8453,                    // omit to follow the client chain
  name: 'My Token',
  symbol: 'TKN',
  image: 'ipfs://...',
  tokenAdmin: account.address,      // creator; omit or zero to use the caller

  metadata: {
    description: 'Token description',
    socialMediaUrls: [
      { platform: 'twitter', url: 'https://twitter.com/mytoken' },
      { platform: 'telegram', url: 'https://t.me/mytoken' },
    ],
    auditUrls: ['https://example.com/audit'],
  },

  context: {
    interface: 'Bankr',             // defaults to { interface: 'SDK' }
    platform: 'farcaster',
    messageId: '',
    id: '',
  },

  buyFee: 1,                       // percent, default 1
  sellFee: 1,                      // percent, default 1

  pool: {
    pairedToken: 'WETH',           // or the native currency; other quotes need factory approval
  },

  rewards: {
    recipients: [
      { recipient: account.address, admin: account.address, bps: 8000, token: 'Paired' },
      { recipient: INTEGRATOR, admin: INTEGRATOR, bps: 2000, token: 'Paired' },
    ],
  },

  devBuy: {
    ethAmount: 0.01,               // native currency units; decimal strings for small amounts
    recipient: account.address,    // defaults to tokenAdmin, then the caller
  },

  burn: 10,                        // percent of the tax sent to the burn address
  dividends: {
    share: 20,                     // percent of the tax paid as holder dividends
    minimumHolding: 10_000n * 10n ** 18n,
  },
  integrator: {
    address: '0x4444444444444444444444444444444444444444',
    share: 5,                      // percent of the tax paid to the integrator
  },

  contractURI: '',                 // default empty text
  metadataEditable: true,          // default true
  antiSnipeEnabled: true,          // default true
}
```

- `burn`, `dividends.share`, and `integrator.share` divide the tax after the protocol share and must total at most 100%. The creator receives the remainder.
- The first reward slot with the creator as admin becomes the creator slot. If no slot has the creator as admin, the given slots share the whole creator remainder and the creator receives none of it.

**Reference:** [references/deployment.md](references/deployment.md)

## Tax Model

- Buy and sell taxes are each 1% to 10% of the trade.
- The protocol takes 10% of the tax, but at least 0.5% and at most 1% of the trade. At the default 1% tax, the protocol takes 0.5% of the trade, which is half the tax.
- Burn, dividend, and integrator shares divide what remains after the protocol share. The creator slots receive the rest.
- A buy in the first 20 seconds pays an anti-snipe fee that starts at 99% and falls to the normal tax. The extra fee goes to the protocol.
- Dividends require a holding of at least 10,000 tokens. While no holder qualifies, the dividend share goes to the protocol.
- Read the live policy with `quoteConfig` and the factory `taxPolicy` before quoting numbers, because the factory owner can change the policy for new launches.

| Tax | Protocol share of the trade | Left for burn, dividends, integrator, and creator |
|-----|----------------------------|---------------------------------------------------|
| 1% | 0.5% | 0.5% |
| 2% | 0.5% | 1.5% |
| 5% | 0.5% | 4.5% |
| 7% | 0.7% | 6.3% |
| 10% | 1% | 9% |

**Reference:** [references/tax.md](references/tax.md)

## Dev Buy

```typescript
const { txHash, waitForTransaction, error } = await clanker.deploy({
  name: 'Example',
  symbol: 'EX',
  devBuy: { ethAmount: 0.01 },
});
```

- `ethAmount` uses native currency units. Use a decimal string for small amounts.
- `minAmountOut` uses final token base units with 18 decimals.
- By default the SDK simulates the exact launch with a minimum of `1n`, checks the token address, and reads the third result value. It subtracts `devBuySlippagePercent` (default 5%, must be above 0 and below 50) with integer math and rounds down. A zero minimum produces an error. An explicit positive `minAmountOut` skips the quote simulation.
- With default addresses the SDK selects `directNativeSwapXRoute` for a native quote, `stableSwapXRoute(publicClient)` for a stable quote, and `directArcUsdcSwapXRoute` on Arc. Other quotes need `devBuy.routeData` or `buildBuyRoute`. A route builder returns the router calldata as a hex string.
- The effective recipient is `devBuy.recipient`, then a nonzero `tokenAdmin`, then the caller. When it differs from the caller, `deploy` waits for the launch and transfers the exact bought amount, returning `transferTxHash`.
- A dev buy uses an upgradeable swap router, so a low `minAmountOut` gives little price protection. A sandwich trade can lose more value on the native-to-stable step than the slippage percent covers.

**Reference:** [references/chains.md](references/chains.md)

## Supported Chains

| Chain | Chain ID | Native quote | Stable quote | V3 pool fee for a stable dev buy |
|-------|----------|--------------|--------------|----------------------------------|
| Base | 8453 | ETH | USDC (6 decimals) | 0.05% |
| BNB Chain | 56 | BNB | USDC (18 decimals) | 0.01% |
| Monad | 143 | MON | USDC (6 decimals) | 0.3% |
| X Layer | 196 | OKB | USDC (6 decimals) | 0.05% |
| Robinhood Chain | 4663 | ETH | USDG (6 decimals) | 0.01% |
| Arc | 5042 | none, the native coin is USDC | USDC (6 decimals), the default | none, no V3 step |
| Elysium | 99901 | HYPE | USDC (6 decimals) | none, no HYPE/USDC pool |

- Launches were open on all seven chains as of 6 November 2026.
- Clanker **no longer supports new launches** on Ethereum, Arbitrum, and Unichain. Their factories have launches turned off, so a launch there fails with `Launches are not open on chain N` before any send.
- Elysium (chain id 99901) is not in viem, so the SDK exports the `elysium` chain. It has no public RPC, so pass your own (`http('<your Elysium RPC URL>')`).
- `pool.pairedToken` defaults to native currency. The `WETH` string also means native currency. On Arc, both mean USDC. Other quote addresses require factory approval.

**Reference:** [references/chains.md](references/chains.md)

## Post-Deployment Operations (V5)

### Rewards

```typescript
// Read the reward slots
const rewards = await clanker.getTokenRewards({ token: TOKEN_ADDRESS });
// { token, poolId, rewardBps, rewardAdmins, rewardRecipients, rewardIsCreator }

// Check and claim
const available = await clanker.availableRewards({
  token: TOKEN_ADDRESS,
  rewardRecipient: FEE_OWNER_ADDRESS,
});

if (available > 0n) {
  const { txHash, error } = await clanker.claimRewards({
    token: TOKEN_ADDRESS,
    rewardRecipient: FEE_OWNER_ADDRESS,
  });
  if (error) throw error;
}
```

Anyone can send a claim; the escrow always pays the recipient. Burn credits reach the burn address when anyone calls `claimRewards` with `rewardRecipient: '0x000000000000000000000000000000000000dEaD'`.

### Dividends

```typescript
// What a holder can withdraw
const claimable = await clanker.withdrawableDividends({
  token: TOKEN_ADDRESS,
  holder: HOLDER_ADDRESS,
});

// Claim for up to 64 tokens in one transaction
const { txHash, error } = await clanker.claimDividends({
  tokens: [TOKEN_A, TOKEN_B],
  unwrapWrappedNative: true,   // false to receive the wrapped token
});
```

### Creator Rights

```typescript
await clanker.setCreatorFeeRecipient({ token, newCreatorFeeRecipient });
await clanker.proposeCreatorRightsTransfer({ token, newCreator });
await clanker.acceptCreatorRightsTransfer({ token });
await clanker.cancelCreatorRightsTransfer({ token });
await clanker.safeTransferCreatorRights({ token, receiver, callbackData, value });
```

`safeTransferCreatorRights` moves rights to a contract in one step; the contract must implement the creator-rights receiver callback.

### Metadata

```typescript
await clanker.updateImage({ token, newImage: 'ipfs://...' });
await clanker.updateMetadata({ token, metadata: JSON.stringify({...}) });
await clanker.updateTokenName({ token, newName: '...' });
await clanker.updateTokenSymbol({ token, newSymbol: '...' });
await clanker.updateTokenContractURI({ token, newContractURI: '...' });
await clanker.updateTokenExtraMetadata({ token, metadataKey: '...', metadataValue: '...' });
```

These work while `metadataEditable` is true. Every write has a matching `get…Transaction` and `…Simulate` method.

### Reads

```typescript
const policy = await clanker.taxPolicy();
const open = await clanker.launchCreationEnabled();
const config = await clanker.getPoolConfig({ token });
const rates = await clanker.getTaxRates({ token, isBuy: true });
const address = await clanker.predictToken(params, account);
const version = await clanker.configVersion();
```

`getTaxRates` returns a 1e24 scale, so 1% is `10n ** 22n`. The taxes and shares in `getPoolConfig` use the launch scale, where 1% is `1_000_000`.

**Reference:** [references/rewards.md](references/rewards.md)

## Result and Error Semantics

`deploy` returns `{ txHash, expectedAddress, waitForTransaction, error, transferTxHash?, failedStep?, transferError? }`.

- A top-level `error` means **nothing was sent**, so a retry is safe. After the launch is sent, `deploy` never returns a top-level `error`.
- `waitForTransaction()` returns `{ address, poolId }` or `{ error }`. An `error.data.type` of `state` means the launch failed on chain. A type of `unknown` means the SDK could not confirm the launch, so the launch can exist. Do not launch again before checking the transaction.
- `SEND_UNCERTAIN` (exported from `clanker-sdk/v5`) marks a failure during the send itself, where the wallet or RPC failed and the launch can exist. Check the wallet's transactions before a retry.
- `failedStep` names the failed step: `launchReceipt` means the SDK could not confirm the launch; `transfer` means the launch succeeded and the transfer failed.
- The deadline is 1,200 seconds, or 300 seconds with a dev buy. Set `deadlineSeconds` to change it. A launch signed after the deadline reverts with `LaunchExpired` and costs gas.
- The local salt search permits 1,000,000 attempts. `maxTries` limits attempts, `signal` cancels the search. Each preparation can produce a different token address.

```typescript
import { Clanker, SEND_UNCERTAIN } from 'clanker-sdk/v5';

const { txHash, waitForTransaction, error } = await clanker.deploy(config);

if (error) {
  if ((error as any).data?.rawName === SEND_UNCERTAIN) {
    // The launch can exist. Check the wallet's transactions before retrying.
  }
  throw error;
}

const result = await waitForTransaction();
if (result.error) throw result.error;
```

## Unsigned Deployment

```typescript
const tx = await clanker.getDeployTransaction(config, { account: callerAddress });
// Submit tx.address, tx.abi, tx.functionName, tx.args, tx.value yourself.
```

`getDeployTransaction` sends nothing. The caller must send any recipient transfer themselves, with `transferLaunchBuy` or an ERC-20 transfer.

## CLI

```bash
npm install -g clanker-sdk   # or: npx clanker-sdk ...
```

```sh
clanker-sdk deploy --name "Example" --symbol "EX" --chain base
clanker-sdk deploy --name "Example" --symbol "EX" --chain robinhood \
  --dev-buy-eth 0.05 \
  --burn 10 --dividend-share 20 --dividend-min-holding 10000 \
  --integrator 0x4444444444444444444444444444444444444444 --integrator-share 5
clanker-sdk dividends available --token <TOKEN_A>,<TOKEN_B>
clanker-sdk dividends claim --token <TOKEN_A> --token <TOKEN_B>
clanker-sdk rewards claim --token <V5_LAUNCH_TOKEN>
```

| Command | Description |
|---------|-------------|
| `setup` | Save default wallet and chain config |
| `deploy` | Deploy a V5 token |
| `rewards` | Claim rewards and update recipients or admins (add `--v5` for V5 currency claims) |
| `dividends` | Check and claim holder dividends of V5 tokens |
| `vault` | Claim vaulted V4 tokens |
| `airdrop` | Generate Merkle trees, register and claim V4 airdrops |
| `token` | Update the image and metadata for V4 and V5 tokens |

Global options: `--chain <name>`, `--rpc <url>`, `--private-key <key>`, `--json`, `--dry-run`.

Chain names: `base`, `base-sepolia`, `arbitrum`, `ethereum`, `bsc`, `unichain`, `monad`, `robinhood`, `xlayer`, `arc`, `elysium`.

- Buy and sell fees default to 1%. Only name and symbol are required.
- Exit code 1 means the launch failed or nothing was sent. Exit code 2 means the launch exists or can exist and a later step failed. Do not re-run after exit code 2, because that can launch a second token.
- `vault` and `airdrop` are for V4 tokens only.

## Legacy V3/V4 Tokens

Management of existing tokens is unchanged, but the client comes from the versioned entry point:

```typescript
import { Clanker } from 'clanker-sdk/v4';           // V4 tokens
import { Clanker } from 'clanker-sdk/v3';           // V3 tokens
import { createAirdrop, registerAirdrop } from 'clanker-sdk/v4/extensions';
```

| Task | Entry point |
|------|-------------|
| Claim vaulted tokens | `clanker-sdk/v4` — `getVaultClaimableAmount`, `claimVaultedTokens` |
| Airdrops and presales | `clanker-sdk/v4/extensions` |
| LP fee claims, reward updates | `clanker-sdk/v4` or `clanker-sdk/v3` |
| Update image and metadata | `clanker-sdk/v4` or `clanker-sdk/v3` |
| Launch a new token | `clanker-sdk/v5` only |

**References:** [references/vesting.md](references/vesting.md), [references/airdrops.md](references/airdrops.md), [references/pool-config.md](references/pool-config.md)

## Common Workflows

### Simple Memecoin Launch

1. Upload the token image to IPFS.
2. Deploy with name, symbol, image, and `tokenAdmin`.
3. Share the token address and the pool id.

### Creator Token with Taxes and Dividends

1. Set `buyFee` and `sellFee` within the 1% to 10% range.
2. Split the tax with `burn`, `dividends.share`, and `integrator.share` (at most 100% in total).
3. Set `dividends.minimumHolding` in token base units.
4. Deploy and share the token address.

### Interface Fee Split

```typescript
rewards: {
  recipients: [
    { recipient: account.address, admin: account.address, bps: 8000, token: 'Paired' },
    { recipient: INTEGRATOR_ADDRESS, admin: INTEGRATOR_ADDRESS, bps: 2000, token: 'Paired' },
  ],
}
```

### Existing V4 Token With a Vault or Airdrop

Use the V4 entry point for claims and extensions. See the legacy references above.

## Best Practices

### Security

1. **Never expose private keys** — use environment variables and keep them out of git.
2. **Simulate first** — use `deploySimulate` or the CLI `--dry-run` before sending.
3. **Read the live tax policy** — `taxPolicy` and `quoteConfig` can change for new launches.
4. **Check the wallet's transactions before retrying** a `SEND_UNCERTAIN` failure.

### Cost and Timing

1. **Prefer Base, BNB Chain, Monad, or X Layer** for lower gas.
2. **Raise `deadlineSeconds`** for slow signers such as multisigs.
3. **Set `minAmountOut` explicitly** for large dev buys.

## Troubleshooting

**Reference:** [references/troubleshooting.md](references/troubleshooting.md)

## Resources

- **GitHub**: [github.com/clanker-devco/clanker-sdk](https://github.com/clanker-devco/clanker-sdk)
- **NPM**: [npmjs.com/package/clanker-sdk](https://www.npmjs.com/package/clanker-sdk)
- **V5 guide**: [docs/v5.md](https://github.com/clanker-devco/clanker-sdk/blob/main/docs/v5.md)
- **Examples**: [examples/v5](https://github.com/clanker-devco/clanker-sdk/tree/main/examples/v5)

---

**💡 Pro Tip**: Read `taxPolicy` and `quoteConfig` before you quote a tax or dividend number to a user. The factory owner can change the policy for new launches.

**⚠️ Security**: Never commit private keys. Use `.env` files and keep them in `.gitignore`.
