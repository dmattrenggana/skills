# Troubleshooting

Common issues and solutions when using the Clanker SDK.

## Setup Issues

### Peer dependency error on install

**Cause:** SDK v5 requires viem 2.49.3 or newer, and viem is a peer dependency, so the SDK always uses the project's viem copy.

**Solution:**
```bash
npm install viem@^2.49.3 clanker-sdk
```

### "Clanker is not exported from 'clanker-sdk'"

**Cause:** The root entry no longer exports the `Clanker` class. Versioned entry points exist per client version.

**Solution:**
```typescript
import { Clanker } from 'clanker-sdk/v5';           // new launches
import { Clanker } from 'clanker-sdk/v4';           // V4 token management
import { Clanker } from 'clanker-sdk/v3';           // V3 token management
import { createAirdrop, registerAirdrop } from 'clanker-sdk/v4/extensions';
import { ClankerTokenV5 } from 'clanker-sdk/v5';    // V5 token type
```

### "Missing PRIVATE_KEY env var"

**Cause:** Environment variable not set or not in correct format.

**Solution:**
```bash
export PRIVATE_KEY=0x...your_64_character_hex_key...
```

```typescript
import { isHex } from 'viem';

const PRIVATE_KEY = process.env.PRIVATE_KEY;
if (!PRIVATE_KEY || !isHex(PRIVATE_KEY)) {
  throw new Error('PRIVATE_KEY must be a hex string starting with 0x');
}
```

## Deployment Issues

### "Launches are not open on chain N"

**Cause:** New launches are closed on Ethereum, Arbitrum, and Unichain. Their factories have launches turned off.

**Solution:** Deploy on Base, BNB Chain, Monad, X Layer, Robinhood Chain, Arc, or Elysium. The error is raised before anything is sent, so a retry on another chain is safe.

### "Unsupported input" for a V4 field

**Cause:** V5 rejects vault, airdrop, presale, pool positions, presets, custom ticks, sniper fees, lockers, pool extensions, `salt`, `fee`, `fees`, and the `Both` or `Clanker` fee preferences.

**Solution:** Remove the field. Use `buyFee` and `sellFee` for fees, `rewards.recipients` for fee slots, `burn`, `dividends`, and `integrator` for the tax split. V5 always mines the `cccc` suffix, so `vanity` is accepted but has no effect.

### "Transaction reverted" with an unknown reason

**Cause:** Invalid configuration or contract state issue.

**Solution:**
1. Simulate first:
```typescript
const result = await clanker.deploySimulate(config);
```

2. Validate the configuration:
```typescript
if (!config.name || !config.symbol) throw new Error('Name and symbol required');

const totalBps = config.rewards?.recipients?.reduce((sum, r) => sum + r.bps, 0) ?? 0;
if (totalBps !== 10000) throw new Error('Rewards must total 10000 bps');

const taxSplit = (config.burn ?? 0) + (config.dividends?.share ?? 0) + (config.integrator?.share ?? 0);
if (taxSplit > 100) throw new Error('Burn, dividend, and integrator shares must total at most 100%');
```

### "Insufficient balance"

**Cause:** The wallet does not have enough native token for gas and the dev buy.

**Solution:**
```typescript
import { formatEther } from 'viem';

const balance = await publicClient.getBalance({ address: account.address });
console.log('Balance:', formatEther(balance), 'ETH');
```

### "Invalid image"

**Cause:** IPFS hash not accessible or invalid format.

**Solution:**
```typescript
const image = 'ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi';
// Test: https://ipfs.io/ipfs/bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi
```

### Wrong Chain

**Cause:** Wallet and publicClient configured for different chains.

**Solution:**
```typescript
const CHAIN = base;

const publicClient = createPublicClient({ chain: CHAIN, transport: http() });
const wallet = createWalletClient({ account, chain: CHAIN, transport: http() });

const { txHash } = await clanker.deploy({ chainId: CHAIN.id, name: 'My Token', symbol: 'TKN' });
```

## Result Handling Issues

### `error.data.rawName === SEND_UNCERTAIN`

**Cause:** The wallet or RPC failed during the send itself, after the checks passed. The launch can exist.

**Solution:** Check the wallet's transactions before retrying. Do not blindly retry, because a second call can launch a second token.

```typescript
import { SEND_UNCERTAIN } from 'clanker-sdk/v5';

if (error && (error as any).data?.rawName === SEND_UNCERTAIN) {
  // Inspect the wallet's pending transactions first.
}
```

### `result.error.data.type` is `unknown`

**Cause:** The SDK could not confirm the launch, for example because the receipt read failed. The launch can exist.

**Solution:** Check the transaction on the block explorer before launching again.

### `transferError` and `failedStep`

**Cause:** The launch succeeded but the recipient transfer failed.

**Solution:** Retry the transfer with the SDK:

```typescript
await clanker.transferLaunchBuy({ txHash, token: TOKEN_ADDRESS, recipient: RECIPIENT });
```

`failedStep` is `launchReceipt` when the launch could not be confirmed, and `transfer` when the launch succeeded and the transfer failed. A second transfer call sends the amount again if the wallet still holds the tokens, so check the recipient balance first.

### `LaunchExpired`

**Cause:** The launch was signed after the deadline. The deadline starts when the SDK reads the block, before the wallet signs.

**Solution:** Raise `deadlineSeconds` for slow signers such as a multisig.

```typescript
await clanker.deploy(config, { deadlineSeconds: 1800 });
```

## Dev Buy Issues

### "Zero minimum produces an error"

**Cause:** The SDK simulates the launch with a minimum of `1n`, subtracts `devBuySlippagePercent` (default 5), and rejects a zero result.

**Solution:** Set an explicit positive `minAmountOut`, or lower the slippage percent:

```typescript
await clanker.deploy(config, {
  devBuySlippagePercent: 2,
});
```

### "Other quotes or custom addresses require devBuy.routeData or buildBuyRoute"

**Cause:** The SDK only ships routes for the native quote, the stable quotes in its table, and Arc USDC.

**Solution:** Pass a custom route. Elysium USDC needs one, because there is no HYPE/USDC pool.

```typescript
import type { TaxBuyRouteBuilder } from 'clanker-sdk/v5';

const buildBuyRoute: TaxBuyRouteBuilder = async (context) => {
  // context: chainId, caller, token, hook, adapter, tickSpacing,
  //          launchParams, amountIn, minAmountOut, poolKey
  return '0x...';
};
```

### Dev buy bought too few tokens

**Cause:** A dev buy uses an upgradeable swap router, and a sandwich trade can lose more value on the native-to-stable step than the slippage percent covers.

**Solution:** Set an explicit `minAmountOut`, or lower `devBuySlippagePercent` for a large buy.

## Chain-Specific Issues

### Elysium: request fails immediately

**Cause:** Elysium has no public RPC.

**Solution:** Pass your own RPC URL.

```typescript
import { elysium } from 'clanker-sdk';

const publicClient = createPublicClient({
  chain: elysium,
  transport: http(process.env.ELYSIUM_RPC_URL),
}) as PublicClient;
```

In the CLI, pass `--rpc <url>`.

### Robinhood Chain: unsupported quote

**Cause:** The stable quote on Robinhood Chain is USDG, and other quotes require factory approval.

**Solution:** Use the default native quote, or a registered quote address.

## Post-Deployment Issues

### "Cannot claim rewards"

**Cause:** Wrong recipient, or no rewards accumulated yet.

**Solution:**
```typescript
const available = await clanker.availableRewards({
  token: TOKEN_ADDRESS,
  rewardRecipient: YOUR_ADDRESS,
});
console.log('Available:', available);
```

Anyone can send a claim; the escrow always pays the recipient.

### "updateRewardAdmin rejects the creator slot"

**Cause:** The creator slot follows creator rights.

**Solution:** Move creator rights instead:

```typescript
await clanker.proposeCreatorRightsTransfer({ token: TOKEN_ADDRESS, newCreator: '0x...' });
await clanker.acceptCreatorRightsTransfer({ token: TOKEN_ADDRESS });
```

### "No dividends available"

**Cause:** The holder does not meet `dividends.minimumHolding`, or the holder is excluded.

**Solution:**
```typescript
const claimable = await clanker.withdrawableDividends({
  token: TOKEN_ADDRESS,
  holder: HOLDER_ADDRESS,
});
```

Dividends require a holding of at least 10,000 tokens. While no holder qualifies, the dividend share goes to the protocol.

### "Cannot update metadata"

**Cause:** `metadataEditable` is false, or the caller is not the creator.

**Solution:** Metadata updates work while `metadataEditable` is true, which is the default at launch.

### Tax numbers look wrong

**Cause:** The factory owner can change the tax policy for new launches.

**Solution:** Read the live policy rather than assuming.

```typescript
const policy = await clanker.taxPolicy();
const quote = await clanker.quoteConfig(QUOTE_TOKEN);
```

## RPC Issues

### "Rate limited"

**Solution:**
```typescript
const publicClient = createPublicClient({
  chain: base,
  transport: http(process.env.RPC_URL_BASE),
});
```

## Error Reference

V5 revert names, taken from the SDK error ABI:

| Error | Cause | Solution |
|-------|-------|----------|
| `LaunchCreationDisabled` | Launches are off on that chain | Use a chain with launches open |
| `LaunchExpired` | Signed after the deadline | Raise `deadlineSeconds` |
| `LaunchSaltUsed` | The mined salt was already used | Retry, the SDK mines a fresh salt |
| `StaleConfig` | The factory config changed mid-launch | Retry, the SDK reads the config version again |
| `InvalidConfig` | Invalid launch configuration | Check the token fields |
| `EmptyTokenNameOrSymbol` | Missing name or symbol | Both are required |
| `InvalidTaxBounds` | Tax outside the allowed range | Keep each tax between 1% and 10% |
| `InvalidTaxConfiguration` | Tax shares do not add up | Burn, dividends, and integrator shares total at most 100% |
| `InvalidProtocolFeeConfiguration` | The protocol fee rule is invalid | Report it, the factory owns the policy |
| `InvalidFeeSchedule` | Invalid fee schedule | Use `buyFee` and `sellFee` percentages |
| `InvalidFeeSlots` | Reward slots invalid | 1 to 7 slots, positive bps, total 10000 |
| `InvalidMinimumDividendHoldingRequirement` | `minimumHolding` below the factory bound | Use at least 10,000 tokens in base units |
| `DividendsDisabled` | Dividends are off for the launch | Remove the dividend share |
| `InvalidQuoteToken` | Quote not registered | Use the native quote or a registered quote |
| `QuoteNotRegistered` | Quote missing from the factory | Register the quote first |
| `QuoteAlreadyRegistered` | Quote already registered | Pick another quote |
| `InvalidTokenAddressSuffix` | The mined suffix is invalid | Retry the launch |
| `MetadataNotEditable` | `metadataEditable` is false | Metadata updates are disabled for this token |
| `MetadataUnauthorized` | Caller is not the creator | Use the creator account |
| `NotCurrentCreator` | Caller is not the current creator | Transfer creator rights first |
| `NotPendingCreator` | Creator rights were not proposed | Call `proposeCreatorRightsTransfer` first |
| `NotPendingCreatorAdmin` | Caller is not the pending creator admin | Use the pending admin account |
| `CreatorSlotManagedByFactory` | The creator slot cannot be edited directly | Use creator rights transfers |
| `InvalidCreatorRightsReceiver` | Receiver does not implement the callback | Implement the creator-rights receiver callback |
| `IntegratorRejectedLaunch` | The integrator contract rejected the launch | Check the integrator contract |
| `InvalidLaunchBuyParams` | Dev buy parameters invalid | Check `ethAmount` and `minAmountOut` |
| `ZeroLaunchBuyOutput` | The dev buy would buy zero tokens | Lower the slippage or set a smaller minimum |
| `OutputBelowMinimum` | The swap output was below the minimum | Lower `minAmountOut` or raise slippage |
| `InsufficientGasForLaunchConsent` | Not enough gas for the launch consent | Fund the wallet |
| `AntiSnipeUnavailable` | Anti-snipe is unavailable for this launch | Retry, or set `antiSnipeEnabled: false` |
| `ExactOutputDisabledDuringAntiSnipe` | Exact-output swaps are blocked during anti-snipe | Use a normal swap |
| `InvalidExternalPool` | The V3 pool for a stable dev buy is invalid | Check the pool fee and tick spacing |
| `InvalidRouteData` | The custom route data is invalid | Check the router call |
| `UnsupportedRoute` | The custom route is not supported | Use a supported route shape |
| `NoEligibleDividendShares` | No holder qualifies for dividends | Wait for a qualifying holder |
| `NothingToClaim` | There is nothing to claim | Check the balance first |
| `DuplicateToken` | A duplicate token in a batch | Deduplicate the list |
| `InvalidBatch` | The batch is invalid | Check the batch size and order |

## Debug Checklist

1. ✅ viem 2.49.3 or newer installed
2. ✅ `Clanker` imported from `clanker-sdk/v5`
3. ✅ Private key set and valid (starts with 0x)
4. ✅ Wallet funded with native token
5. ✅ Wallet and publicClient on the same chain
6. ✅ Chain has launches open
7. ✅ Name and symbol provided
8. ✅ Rewards total exactly 10000 bps, at most 7 slots
9. ✅ Burn, dividend, and integrator shares total at most 100%
10. ✅ Quote is native or registered
11. ✅ Dev buy uses a supported route, or a custom route builder
12. ✅ `deadlineSeconds` high enough for the signer
13. ✅ Checked the wallet's transactions before any retry

## Getting Help

1. **Check SDK examples**: [github.com/clanker-devco/clanker-sdk/tree/main/examples](https://github.com/clanker-devco/clanker-sdk/tree/main/examples)
2. **Read the V5 guide**: [docs/v5.md](https://github.com/clanker-devco/clanker-sdk/blob/main/docs/v5.md)
3. **Review error messages**: they usually name the failing step
4. **Verify on the block explorer**: check transaction status and logs
