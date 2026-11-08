# Chains and Dev-Buy Routes (V5)

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

Launches were open on all seven chains as of 6 November 2026.

## Closed for New Launches

Clanker no longer supports new launches on Ethereum, Arbitrum, and Unichain. Their factories have launches turned off, so a launch there gives the error `Launches are not open on chain N` before any send.

Existing tokens on those chains stay manageable.

## Elysium

Elysium (chain id `ELYSIUM_CHAIN_ID`, 99901) is not in viem, so `clanker-sdk` exports the `elysium` chain. It has no public RPC, so the chain has no default RPC URL. Pass your own:

```typescript
import { elysium } from 'clanker-sdk';

const publicClient = createPublicClient({
  chain: elysium,
  transport: http(process.env.ELYSIUM_RPC_URL),
}) as PublicClient;
```

The CLI asks for `--rpc` on a chain without a public RPC instead of failing on its first request.

## Chain Clients

```typescript
import { base, bsc, monad, robinhood, xLayer, arc } from 'viem/chains';
import { elysium } from 'clanker-sdk';

const CHAIN = robinhood;
```

`chainId` in the token config is optional. Omit it to follow the client chain.

## Dev-Buy Route Selection

With the default addresses the SDK picks the route:

| Quote | Route | Notes |
|-------|-------|-------|
| Native currency | `directNativeSwapXRoute` | Default for native quotes |
| Stable quote | `stableSwapXRoute(publicClient)` | One V3 pool from the native coin to the stable quote, then the Tax pool |
| Arc USDC | `directArcUsdcSwapXRoute` | The adapter converts native USDC units from 18 decimals to 6 decimals |
| Any other quote | custom | Needs `devBuy.routeData` or `buildBuyRoute` |

Route rules:

- The helper reads the adapter limits and the factory pool. It does not search pools.
- A fee above `maxExternalPoolFee`, a missing pool, another pair, another tick spacing, or nonzero hooks produces an error.
- A native quote does not need a `poolKey`. Do not combine a nonzero key with `routeData` or `buildBuyRoute`.
- An all-zero `poolKey` means unset. Another key must pair the native coin or its wrapped token with the stable quote, in either order, with zero hooks.
- The helper uses the key's `fee` to select the V3 pool. Its `tickSpacing` must match that fee in the V3 factory.
- On Arc, use at most six decimals in `ethAmount`. The Arc router accepts deadlines no more than 300 seconds ahead.
- On Elysium, only the HYPE quote has an SDK route. A USDC dev buy needs a custom route, because there is no HYPE/USDC pool.

### Custom Route

```typescript
import { Clanker, type TaxBuyRouteBuilder } from 'clanker-sdk/v5';

const buildBuyRoute: TaxBuyRouteBuilder = async (context) => {
  // context: chainId, caller, token, hook, adapter, tickSpacing,
  //          launchParams, amountIn, minAmountOut, poolKey
  return '0x...';
};

const clanker = new Clanker({ wallet, publicClient, buildBuyRoute });
```

The builder returns the router calldata as a hex string. The callback receives the chain, caller, predicted token, hook, adapter, tick spacing, launch params, amount in, and the minimum. When the SDK sets the minimum, the callback receives `minAmountOut: 1n`. Do not use that value as a limit in the route.

## Addresses

`clankerConfigFor` and `CLANKERS` have no V5 entry. Use the V5 address book:

```typescript
import { TAX_ADDRESSES } from 'clanker-sdk/v5';

const addresses = TAX_ADDRESSES[4663];
// { factory, hook, escrow, tokenDeployer, dividendClaimHelper }
```

Omit `addresses` for the default deployment. Other deployments require the relevant `TaxAddresses` fields:

```typescript
const clanker = new Clanker({
  wallet,
  publicClient,
  addresses: {
    factory: '0x...',
    hook: '0x...',
    escrow: '0x...',
    tokenDeployer: '0x...',
    dividendClaimHelper: '0x...',
  },
});
```

## CLI Chain Names

`base`, `base-sepolia`, `arbitrum`, `ethereum`, `bsc`, `unichain`, `monad`, `robinhood`, `xlayer`, `arc`, `elysium`.

```sh
clanker-sdk deploy --name "Example" --symbol "EX" --chain robinhood --dev-buy-eth 0.05
```
