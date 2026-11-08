# Pool Configuration (V5)

## V5: Quote Only

V5 has no per-launch start price or market cap input. The factory sets the start price for each quote, and the V4 market cap helpers do not apply.

```typescript
pool: {
  pairedToken: 'WETH',  // or a native currency address; 'WETH' means native currency
}
```

- The default quote is the native currency of the chain.
- On Arc, the native coin is USDC, so both the default and `WETH` mean USDC.
- Other quote addresses require factory approval.

The following V4 inputs are rejected in V5:

| Rejected | Reason |
|----------|--------|
| `pool.positions` | V5 creates a standard pool only |
| `pool.preset` | V5 creates a standard pool only |
| `pool.tickIfToken0IsClanker` | No custom ticks |
| `pool.tickSpacing` | No custom ticks |
| `poolPositions` | No custom positions |
| `poolPreset` | No custom positions |
| `poolExtension` | No pool extensions |

## Reading the Stored Pool

```typescript
const config = await clanker.getPoolConfig({ token: TOKEN_ADDRESS });
```

Returns the stored launch: quote, creator, fee recipients, dividend contract, taxes and shares, and the protocol fee rule frozen at launch.

## Legacy V4 Market Cap Helpers

These helpers still exist in the package root for V4 tokens. They do not apply to V5 launches.

```typescript
import { getTickFromMarketCap, getTickFromMarketCapStable, getTickFromMarketCapUSDC } from 'clanker-sdk';

const customPool = getTickFromMarketCap(20); // 20 ETH
```

A V4 launch with a custom market cap looked like this:

```typescript
const { txHash, waitForTransaction, error } = await clanker.deploy({
  // ... V4 config
  pool: {
    ...customPool,
    positions: [
      {
        tickLower: customPool.tickIfToken0IsClanker,
        tickUpper: -120000,
        positionBps: 10_000,
      },
    ],
  },
});
```

Use that shape only with `clanker-sdk/v4`. The V4 `deploy` throws from SDK v5 onward.

## Legacy V4 Fee Configs

```typescript
import { FEE_CONFIGS, FeeConfigs } from 'clanker-sdk';

const fees = FEE_CONFIGS[FeeConfigs.StaticBasic];
```

`FEE_CONFIGS` has no V5 entry. V5 uses `buyFee` and `sellFee` percentages instead of a fee config object.

## Legacy V4 Position Presets

```typescript
import { POOL_POSITIONS, PoolPositions } from 'clanker-sdk';

const positions = POOL_POSITIONS[PoolPositions.Standard];
```

V5 has no position presets.

## Contract Limits

| Parameter | Value |
|-----------|-------|
| Max reward slots | 7 |
| Reward bps total | 10,000 |
| Max LP fee | 10% |
| Max sniper fee | 80% |
| Sniper fee decay | 2 minutes max |
| Max LP positions | 7 |
| Max extensions | 10 |
| Max extension bps | 9000 (90%) |
| Token supply | 100 billion (18 decimals) |

The pool limits for extensions, positions, and sniper fees apply to V4 tokens.
