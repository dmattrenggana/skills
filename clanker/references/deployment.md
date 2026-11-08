# Token Deployment (V5)

Complete guide to launching a token with the Clanker SDK v5 client.

## Simple Deployment

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
  chainId: base.id,
  metadata: {
    description: 'My really cool token',
  },
  context: {
    interface: 'Clanker SDK',
  },
});

if (error) throw error;

console.log(`Deploying... ${base.blockExplorers.default.url}/tx/${txHash}`);
const result = await waitForTransaction();
if (result.error) throw result.error;
console.log(`Done! ${base.blockExplorers.default.url}/address/${result.address}`);
```

`wallet` and `publicClient` must use the same chain.

## Multi-Chain Deployment

```typescript
import { bsc, base, monad, robinhood } from 'viem/chains';

const CHAIN = robinhood; // base | bsc | monad | robinhood | xlayer | arc | elysium

const publicClient = createPublicClient({
  chain: CHAIN,
  transport: http(process.env.RPC_URL),
}) as PublicClient;

const wallet = createWalletClient({
  account,
  chain: CHAIN,
  transport: http(process.env.RPC_URL),
});

const clanker = new Clanker({ wallet, publicClient });

const { txHash, waitForTransaction, error } = await clanker.deploy({
  chainId: CHAIN.id,
  name: 'My Token',
  symbol: 'TKN',
});
```

`chainId` is optional. Omit it to follow the client chain.

Elysium is not in viem, so the SDK exports the `elysium` chain and it has no default RPC:

```typescript
import { elysium } from 'clanker-sdk';

const publicClient = createPublicClient({
  chain: elysium,
  transport: http(process.env.ELYSIUM_RPC_URL), // required
}) as PublicClient;
```

## Token Metadata

```typescript
metadata: {
  description: 'Token with custom configuration including taxes and rewards',
  socialMediaUrls: [
    { platform: 'twitter', url: 'https://twitter.com/mytoken' },
    { platform: 'telegram', url: 'https://t.me/mytoken' },
    { platform: 'discord', url: 'https://discord.gg/mytoken' },
  ],
  auditUrls: ['https://example.com/audit'],
}
```

## Context Configuration

Track the deployment source:

```typescript
context: {
  interface: 'Bankr',        // your app or interface name
  platform: 'farcaster',     // social platform
  messageId: '',             // cast hash, tweet URL, and so on
  id: '',                    // FID, X handle, and so on
}
```

`context` defaults to `{ interface: 'SDK' }`.

## Fees

```typescript
buyFee: 2,    // percent of the trade, 1 to 10
sellFee: 2,   // percent of the trade, 1 to 10
```

Each fee defaults to 1% and includes the protocol share. The V4 `fee` and `fees` fields are rejected.

## Quotes

```typescript
pool: {
  pairedToken: 'WETH',  // or a native currency address; 'WETH' means native currency
}
```

- The default quote is the native currency of the chain.
- On Arc, the native coin is USDC, so both the default and `WETH` mean USDC.
- Any other quote address requires factory approval.

## Rewards

```typescript
rewards: {
  recipients: [
    { recipient: account.address, admin: account.address, bps: 8000, token: 'Paired' },
    { recipient: INTEGRATOR, admin: INTEGRATOR, bps: 2000, token: 'Paired' },
  ],
}
```

- 1 to 7 slots, each with a positive `bps`, totalling exactly 10,000.
- `token: 'Paired'` or omit `token`. `Both` and `Clanker` are rejected in V5.
- The first slot whose admin is the creator becomes the creator slot.

## Tax Allocation

```typescript
burn: 10,                       // percent of the tax burned
dividends: {
  share: 20,                    // percent of the tax paid as holder dividends
  minimumHolding: 10_000n * 10n ** 18n,  // token base units
},
integrator: {
  address: '0x4444444444444444444444444444444444444444',
  share: 5,                     // percent of the tax paid to the integrator
},
```

`burn`, `dividends.share`, and `integrator.share` must total at most 100%. The creator receives the remainder.

## Dev Buy

```typescript
const { txHash, waitForTransaction, error } = await clanker.deploy({
  name: 'My Token',
  symbol: 'TKN',
  devBuy: {
    ethAmount: 0.01,            // native currency units
    recipient: account.address, // defaults to tokenAdmin, then the caller
  },
});
```

Set the minimum explicitly when you want price protection:

```typescript
devBuy: {
  ethAmount: '0.05',
  minAmountOut: 123456789012345n,  // final token base units, 18 decimals
}
```

Adjust the default slippage with the `devBuySlippagePercent` deploy option (default 5, must be above 0 and below 50):

```typescript
const { txHash, waitForTransaction, error } = await clanker.deploy(config, {
  devBuySlippagePercent: 2,
});
```

Custom quotes need a route. The builder returns the router calldata as a hex string:

```typescript
import { Clanker, type TaxBuyRouteBuilder } from 'clanker-sdk/v5';

const buildBuyRoute: TaxBuyRouteBuilder = async (context) => {
  // context: chainId, caller, token, hook, adapter, tickSpacing,
  //          launchParams, amountIn, minAmountOut, poolKey
  return '0x...';
};

const { txHash, waitForTransaction, error } = await clanker.deploy(config, { buildBuyRoute });
```

## Deploy Options

```typescript
const { txHash, waitForTransaction, error } = await clanker.deploy(config, {
  deadlineSeconds: 1800,        // default 1200, or 300 with a dev buy
  devBuySlippagePercent: 5,     // default 5
  maxTries: 100_000,            // salt search attempts, default limit 1,000,000
  signal: controller.signal,    // cancel the salt search
});
```

- `deadlineSeconds` starts when the SDK reads the block, before the wallet signs. A launch signed after the deadline reverts with `LaunchExpired` and costs gas. Raise it for slow signers such as multisigs.
- The SDK mines a fresh salt for every preparation, so each attempt can produce a different token address.

## Simulation Before Deployment

```typescript
const result = await clanker.deploySimulate(config);
```

`deploySimulate(token, accountOrOptions?)` accepts a V4 `Account` or V5 options and returns the viem simulation result, or `{ error }`.

## Unsigned Deployment

```typescript
const tx = await clanker.getDeployTransaction(config, { account: callerAddress });
```

Submit `tx.address`, `tx.abi`, `tx.functionName`, `tx.args`, and `tx.value` yourself with that caller. This method sends nothing. The caller must send any recipient transfer themselves, with `transferLaunchBuy` or an ERC-20 transfer.

## Error Handling

```typescript
import { Clanker, SEND_UNCERTAIN } from 'clanker-sdk/v5';

const { txHash, waitForTransaction, error, transferError, failedStep } = await clanker.deploy(config);

if (error) {
  // A top-level error means nothing was sent, so a retry is safe.
  if ((error as any).data?.rawName === SEND_UNCERTAIN) {
    // The send itself failed and the launch can exist. Check the wallet's transactions.
  }
  throw error;
}

if (transferError) {
  // The launch succeeded but the recipient transfer failed.
  // failedStep is 'transfer'; retry with transferLaunchBuy({ txHash, token, recipient }).
  console.error('Transfer failed:', transferError.message);
}

const result = await waitForTransaction();
if (result.error) {
  // result.error.data.type is 'state' (failed on chain) or 'unknown' (not confirmed).
  throw result.error;
}

console.log('Token deployed at:', result.address);
```
