# Tax Model (V5)

How Clanker V5 splits trading taxes, and how to read the values back on chain.

## Policy

The factory `taxPolicy` sets the rules and the contracts enforce them. The SDK cannot change them. As of 6 November 2026 every chain used the same policy:

- Buy and sell taxes are each 1% to 10% of the trade.
- The protocol takes 10% of the tax, but at least 0.5% and at most 1% of the trade. At the default 1% tax, the protocol takes 0.5% of the trade, which is half the tax.
- Burn, dividend, and integrator shares divide what remains after the protocol share. The creator slots receive the rest.
- A buy in the first 20 seconds pays an anti-snipe fee that starts at 99% and falls to the normal tax. The extra fee goes to the protocol.
- Dividends require a holding of at least 10,000 tokens. While no holder qualifies, the dividend share goes to the protocol.
- Tax credits stay in the fee escrow until someone claims them.

## Tax Split Table

| Tax | Protocol share of the trade | Left for burn, dividends, integrator, and creator |
|-----|----------------------------|---------------------------------------------------|
| 1% | 0.5% | 0.5% |
| 2% | 0.5% | 1.5% |
| 5% | 0.5% | 4.5% |
| 7% | 0.7% | 6.3% |
| 10% | 1% | 9% |

Read the live policy with `quoteConfig` and the factory `taxPolicy` before you quote numbers to a user, because the factory owner can change the policy for new launches.

## Configuring the Split at Launch

```typescript
const { txHash, waitForTransaction, error } = await clanker.deploy({
  name: 'Example',
  symbol: 'EX',
  buyFee: 5,
  sellFee: 5,
  burn: 10,                       // percent of the tax burned
  dividends: {
    share: 20,                    // percent of the tax paid as holder dividends
    minimumHolding: 10_000n * 10n ** 18n,
  },
  integrator: {
    address: '0x4444444444444444444444444444444444444444',
    share: 5,                     // percent of the tax paid to the integrator
  },
});
```

- `burn`, `dividends.share`, and `integrator.share` must total at most 100%. The creator receives the remainder.
- `dividends.minimumHolding` uses token base units (18 decimals) and must meet the factory policy and supply bounds.
- The factory controls the initial anti-snipe fee. Its supply goes to the pool. Read `launchSupply` for the amount.

## Reading the Policy

```typescript
const policy = await clanker.taxPolicy();
// { taxFeeBounds, minimumBuyTaxRate, maximumBuyTaxRate, minimumSellTaxRate, maximumSellTaxRate, ... }

const open = await clanker.launchCreationEnabled();
const quote = await clanker.quoteConfig(QUOTE_TOKEN);
// [enabled, buyTaxRate, sellTaxRate]
```

## Reading a Token's Stored Launch

```typescript
const config = await clanker.getPoolConfig({ token: TOKEN_ADDRESS });
```

Returns the stored launch: quote, creator, fee recipients, dividend contract, taxes and shares, and the protocol fee rule frozen at launch.

## Reading Live Tax Rates

```typescript
const buy = await clanker.getTaxRates({ token: TOKEN_ADDRESS, isBuy: true });
const sell = await clanker.getTaxRates({ token: TOKEN_ADDRESS, isBuy: false });
```

The rates include the anti-snipe `surchargeRate`.

### Scale Warning

- `getTaxRates` uses a 1e24 scale: 100% is `10n ** 24n`, so 1% is `10n ** 22n`.
- The taxes and shares inside `getPoolConfig` use the launch scale, where 1% is `1_000_000`.

Do not mix the two scales.

## Burn Credits

Burn credits are launch tokens owed to the burn address. They reach it when anyone calls:

```typescript
const { txHash, error } = await clanker.claimRewards({
  token: TOKEN_ADDRESS,
  rewardRecipient: '0x000000000000000000000000000000000000dEaD',
});
```

## Anti-Snipe

- The factory controls the initial anti-snipe fee, and `antiSnipeEnabled` defaults to true.
- A buy in the first 20 seconds pays an anti-snipe fee that starts at 99% and falls to the normal tax. The extra fee goes to the protocol.
- The V4 `sniperFees` configuration (starting fee, ending fee, decay) is rejected in V5.
