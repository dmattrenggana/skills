# Rewards, Dividends, and Creator Rights (V5)

How trading-fee credits, holder dividends, and creator rights work on V5 tokens.

## Reward Slots

Reward slots split the creator-side tax credits. A slot has a recipient, an admin, and a share in basis points.

```typescript
rewards: {
  recipients: [
    { recipient: account.address, admin: account.address, bps: 8000, token: 'Paired' },
    { recipient: INTEGRATOR, admin: INTEGRATOR, bps: 2000, token: 'Paired' },
  ],
}
```

- 1 to 7 slots.
- Each `bps` must be positive and the slots must total exactly 10,000.
- Use `token: 'Paired'` or omit `token`. `Both` and `Clanker` are rejected in V5.
- The first slot whose admin is the creator becomes the creator slot.
- If no slot has the creator as admin, the given slots share the whole creator remainder and the creator receives none of it.

### Basis Points

- 100 bps = 1%
- 1000 bps = 10%
- 5000 bps = 50%
- 10000 bps = 100%

## Reading the Slots

```typescript
const rewards = await clanker.getTokenRewards({ token: TOKEN_ADDRESS });
```

Returns `token`, `poolId`, `rewardBps`, `rewardAdmins`, `rewardRecipients`, and `rewardIsCreator`, which marks the creator slot. V5 has no V4 position identifiers.

## Checking and Claiming

```typescript
const available = await clanker.availableRewards({
  token: TOKEN_ADDRESS,
  rewardRecipient: FEE_OWNER_ADDRESS,
});
// bigint, the credit balance for that recipient and currency

if (available > 0n) {
  const { txHash, error } = await clanker.claimRewards({
    token: TOKEN_ADDRESS,
    rewardRecipient: FEE_OWNER_ADDRESS,
  });
  if (error) throw error;
}
```

- The currency is the pool quote for a launch token, and the launch token itself for the burn address `0x000000000000000000000000000000000000dEaD`.
- The wrapped token of the native coin maps to native currency. Other tokens keep their own currency.
- The escrow keeps one balance per recipient and currency. A claim pays that whole balance across all pools.
- Anyone can send a claim. The escrow always pays the recipient.
- Burn credits reach the burn address when anyone calls `claimRewards` with `rewardRecipient: '0x000000000000000000000000000000000000dEaD'`.

## Updating a Reward Slot

```typescript
await clanker.updateRewardRecipient({
  token: TOKEN_ADDRESS,
  rewardIndex: 0n,
  newRecipient: '0x...',
});

await clanker.updateRewardAdmin({
  token: TOKEN_ADDRESS,
  rewardIndex: 0n,
  newAdmin: '0x...',
});
```

- The hook pays `slot.recipient`. For the creator slot, `updateRewardRecipient` uses `setCreatorFeeRecipient`.
- A slot admin follows creator rights. `updateRewardAdmin` rejects the creator slot and names `proposeCreatorRightsTransfer`.
- Prior credits stay with the prior recipient.

## Moving Your Own Credits

```typescript
await clanker.claimTo({
  currency: CURRENCY_ADDRESS,
  destination: '0x...',
});
```

`claimTo` sends the caller's own escrow credits to another address.

## Holder Dividends

Dividends are the part of the tax reserved for holders, separate from reward slots.

### Minimum Holding

Dividends require a holding of at least 10,000 tokens. While no holder qualifies, the dividend share goes to the protocol.

```typescript
dividends: {
  share: 20,                              // percent of the tax
  minimumHolding: 10_000n * 10n ** 18n,   // token base units
}
```

### Checking and Claiming

```typescript
// One token
const claimable = await clanker.withdrawableDividends({
  token: TOKEN_ADDRESS,
  holder: HOLDER_ADDRESS,
});

// Or by dividend contract
const byContract = await clanker.withdrawableDividends({
  dividend: DIVIDEND_CONTRACT,
  holder: HOLDER_ADDRESS,
});
```

It returns 0 for an excluded holder.

```typescript
// Claim for up to 64 tokens in one transaction
const { txHash, error } = await clanker.claimDividends({
  tokens: [TOKEN_A, TOKEN_B],
  unwrapWrappedNative: true,  // false to receive the wrapped token
});

// Or claim from one dividend contract
const single = await clanker.withdrawDividends({ dividend: DIVIDEND_CONTRACT });
```

- Native currency is the default payout. Set `unwrapWrappedNative: false` to receive the wrapped token, for example in a contract without `receive`.
- Both calls return `{ error }` without a transaction when the simulation pays nothing.
- Only the factory owner or the platform creator admin manages dividend exclusions and receivers. Token creators cannot.

## Creator Rights

```typescript
// Change where creator fees go
await clanker.setCreatorFeeRecipient({ token: TOKEN_ADDRESS, newCreatorFeeRecipient: '0x...' });

// Move rights to a wallet in two steps
await clanker.proposeCreatorRightsTransfer({ token: TOKEN_ADDRESS, newCreator: '0x...' });
await clanker.acceptCreatorRightsTransfer({ token: TOKEN_ADDRESS });
await clanker.cancelCreatorRightsTransfer({ token: TOKEN_ADDRESS });

// Move rights to a contract in one step
await clanker.safeTransferCreatorRights({
  token: TOKEN_ADDRESS,
  receiver: CONTRACT_ADDRESS,
  callbackData: '0x',
  value: 0n,
});
```

The contract must implement the creator-rights receiver callback.

## Read-Only Client

Reads work without a wallet:

```typescript
import { createPublicClient, http, type PublicClient } from 'viem';
import { base } from 'viem/chains';
import { Clanker } from 'clanker-sdk/v5';

const publicClient = createPublicClient({ chain: base, transport: http() }) as PublicClient;
const clanker = new Clanker({ publicClient });

const rewards = await clanker.getTokenRewards({ token: TOKEN_ADDRESS });
```

## Every Write Has a Simulation

Each V5 write has a matching `get…Transaction` and `…Simulate` method. Simulations accept an optional V4 `Account`.

```typescript
const simulation = await clanker.claimRewardsSimulate({
  token: TOKEN_ADDRESS,
  rewardRecipient: FEE_OWNER_ADDRESS,
});

const tx = await clanker.getClaimRewardsTransaction({
  token: TOKEN_ADDRESS,
  rewardRecipient: FEE_OWNER_ADDRESS,
});
```

## Best Practices

1. **Secure admin addresses** — use a multisig for high-value tokens.
2. **Document the split** — be transparent about fee and dividend shares.
3. **Claim regularly** — credits sit in the escrow until claimed.
4. **Read the policy live** — `taxPolicy` and `quoteConfig` can change for new launches.
5. **Check the creator slot** — `rewardIsCreator` tells you which slot follows creator rights.
