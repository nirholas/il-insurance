# ILInsurance

<!-- three.ws:badges -->
[![GitHub stars](https://img.shields.io/github/stars/nirholas/il-insurance?style=flat&logo=github)](https://github.com/nirholas/il-insurance/stargazers) [![License](https://img.shields.io/github/license/nirholas/il-insurance?style=flat)](https://github.com/nirholas/il-insurance/blob/HEAD/LICENSE) [![Last commit](https://img.shields.io/github/last-commit/nirholas/il-insurance?style=flat)](https://github.com/nirholas/il-insurance/commits) [![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat)](https://github.com/nirholas/il-insurance/pulls) [![AI agent friendly](https://img.shields.io/badge/AI%20agents-AGENTS.md%20%2B%20llms.txt-6d5dfc?style=flat)](https://github.com/nirholas/il-insurance/blob/HEAD/AGENTS.md)
<!-- /three.ws:badges -->


**Pays liquidity providers back for divergence loss, out of a fund the pool's own trading fills.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://il-insurance.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/ILInsuranceHook.sol`](src/hooks/ILInsuranceHook.sol)
- **Licence:** Apache-2.0

## How it works

Impermanent loss is the reason most people who try providing liquidity once do not do it twice. It is also badly named: there is nothing impermanent about it once you withdraw, and the number is not small. A provider who deposits into a pair that then moves has less than they would have had holding, and the fees they earned may or may not have covered it.

Nobody tells them which until they leave. The existing answers all come from outside the pool. Bancor underwrote it from a treasury and stopped when the treasury could not take it.

Options-based cover needs an options market for the pair. Both are somebody else's balance sheet promising to make a provider whole. This makes the pool underwrite itself.

A slice of every swap goes into a fund, and when a provider withdraws, the hook compares what their position is worth against what the same deposit would have been worth held, and pays the difference from the fund up to a cap. Traders pay for the cover through a slightly worse fee, which is the correct party: the cover is what keeps liquidity in the pool they are trading against. The comparison is measured rather than modelled.

At deposit the hook records what actually went in; at withdrawal it records what actually came out and values both at the current price. There is no closed-form IL formula here, because the formula assumes a constant-product position held across a single price move, and a v4 position is a range that may have been crossed repeatedly. What went in and what came out are facts, and the difference between them at one price is the loss.

Three bounds keep this solvent, and none of them is a promise. A claim never exceeds `coverageBps` of the loss. A claim never exceeds what the fund holds, so the fund cannot go negative and there is no lender of last resort.

And cover vests: a position withdrawn before `fullCoverAfter` is covered pro rata to how long it stayed, because insuring a position that arrives, watches one candle and leaves is not insurance, it is a free option on the fund.

## Prior art

Bancor v2.1 and v3 underwrote impermanent loss from protocol reserves and suspended it when they could not. Thorchain does the same with a vesting schedule this borrows from. Options-based cover exists where an options market for the pair does. Funding the cover from the pool's own trading flow, inside the pool, with a fund that can only pay what it holds, is the contribution here.

## Where it does not help

The fund is finite and claims are paid first-come. A pool that suffers a large correlated move after a quiet period will pay early withdrawers in full and later ones partially, which is the honest behaviour of a fund rather than a defect, but it is not the guarantee the word insurance implies. Cover is also valued in currency1 terms at the moment of withdrawal, so a provider withdrawing into a spike is measured against that spike.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
// This hook needs no configuration.

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

This hook takes no per-pool configuration.

## What it reverts with

| Error | Meaning |
| --- | --- |
| `CoverageTooLarge()` | Coverage above 100% would pay a provider more than they lost. |
| `HookFeeTooLarge()` | Fee is higher than the maximum allowed fee. |
| `PayoutNotPoolManager()` | Only the `PoolManager` may drive the callback. Named distinctly because `BaseHook` declares its own. |
| `PremiumTooLarge()` | The premium must leave the swap worth doing. |
| `WrongPool()` | This hook serves one pool, bound at its first initialization. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 5 of the fourteen:

- `afterInitialize`
- `afterAddLiquidity`
- `afterRemoveLiquidity`
- `afterSwap`
- `afterSwapReturnsDelta`

Mask: `0x1544`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # ILInsurance
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # risk, impermanent-loss, lp-economics, insurance, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/il-insurance
cd il-insurance
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/il-insurance&type=Date)](https://www.star-history.com/#nirholas/il-insurance&Date)

<!-- three.ws:growth -->
## Support the project

If il-insurance saves you time, **[star it on GitHub](https://github.com/nirholas/il-insurance)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=il-insurance%3A%20Pays%20liquidity%20providers%20back%20for%20divergence%20loss%2C%20out%20of%20a%20fund%20the%20pool's%20own%20trading%20fills&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fil-insurance) · [Share on Bluesky](https://bsky.app/intent/compose?text=il-insurance%3A%20Pays%20liquidity%20providers%20back%20for%20divergence%20loss%2C%20out%20of%20a%20fund%20the%20pool's%20own%20trading%20fills%20https%3A%2F%2Fgithub.com%2Fnirholas%2Fil-insurance) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fil-insurance) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Fil-insurance&t=il-insurance%3A%20Pays%20liquidity%20providers%20back%20for%20divergence%20loss%2C%20out%20of%20a%20fund%20the%20pool's%20own%20trading%20fills) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fil-insurance&title=il-insurance%3A%20Pays%20liquidity%20providers%20back%20for%20divergence%20loss%2C%20out%20of%20a%20fund%20the%20pool's%20own%20trading%20fills)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/il-insurance` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/il-insurance/issues) or [start a discussion](https://github.com/nirholas/il-insurance/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/il-insurance)](https://github.com/nirholas/il-insurance/graphs/contributors)

<!-- /three.ws:growth -->
