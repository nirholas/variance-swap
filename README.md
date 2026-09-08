# VarianceSwap

**Turns the exposure a liquidity provider already has into something they can actually sell.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://variance-swap.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/VarianceSwapHook.sol`](src/hooks/VarianceSwapHook.sol)
- **Licence:** Apache-2.0

## How it works

Everybody who has provided liquidity to an automated market maker is short volatility. It is not a choice they made and in most cases not one they were told about: the position loses to whoever rebalances against it, in proportion to how much the price moves, which is the payoff of a short variance position with extra steps. The literature calls it loss-versus-rebalancing, and the practical consequence is that a provider's real risk has no name in the interface and no way to hedge.

The exposure exists. What is missing is the other side of it. This hook measures the pool's realised variance from its own ticks, and lets anybody take either end: deposit collateral to be short variance and collect premiums, or pay a premium to be long it and get paid if the pool turns out to be wilder than the strike said.

A provider who wants to stop being short volatility can buy exactly enough of the long side to cancel it, in the same contract, denominated in the same units, against the same pool. The variance is the pool's own. It is the sum of squared tick moves the pool actually made, divided by the seconds it took, and it is not quoted by anybody, not signed by anybody, and not available to be reported wrongly.

Ticks are log prices, which is exactly what a variance calculation wants, so the pool's own data structure happens to be the correct input with no conversion at all. Every note is fully collateralised when it is written. The most it can ever pay is locked at that moment and released when it settles, so the short side cannot be surprised and the long side cannot be defaulted on.

That is a real constraint on how much can be written and it is the right one: an uncollateralised variance seller is a counterparty risk wearing a payoff diagram.

## Prior art

Variance swaps are standard over-the-counter equity derivatives. On-chain, Squeeth and this catalogue's own PowerPerp give quadratic price exposure, and Opyn, Volmex and Panoptic build volatility products on option or index machinery, all needing an external mark. Loss-versus-rebalancing is well described in the literature and universally left unhedged. Measuring realised variance from a pool's own ticks and settling fully collateralised notes on it inside that same pool, so the exposure and its hedge live in one contract, is the contribution here.

## Where it does not help

Realised variance is sampled per swap, so a pool that trades rarely reports a variance built from few observations and a pool that is quiet between two distant prints understates the path between them. The measure also cannot tell a real move from a manipulated one; on a shallow pool, buying a note and then pushing the price around is a strategy, and the cap on payout is the only thing bounding it. Notes settle in one collateral currency and pay nothing before expiry, so this is a held-to-maturity instrument, not a tradeable one. And the strike is chosen by whoever writes the note rather than discovered, so a badly struck note is simply a bad trade.

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
| `AlreadyBound()` | This hook serves one pool, bound the first time one initializes with it. |
| `AmountTooSmall()` | The deposit or withdrawal was too small to move any shares. |
| `CollateralLocked(uint256)` | Withdrawing this much would leave live notes uncollateralised. |
| `ERC20InsufficientAllowance(address,uint256,uint256)` | Indicates a failure with the `spender`’s `allowance`. Used in transfers. |
| `ERC20InsufficientBalance(address,uint256,uint256)` | Indicates an error related to the current `balance` of a `sender`. Used in transfers. |
| `ERC20InvalidApprover(address)` | Indicates a failure with the `approver` of a token to be approved. Used in approvals. |
| `ERC20InvalidReceiver(address)` | Indicates a failure with the token `receiver`. Used in transfers. |
| `ERC20InvalidSender(address)` | Indicates a failure with the token `sender`. Used in transfers. |
| `ERC20InvalidSpender(address)` | Indicates a failure with the `spender` to be approved. Used in approvals. |
| `InsufficientCollateral(uint256,uint256)` | The vault does not have enough free collateral to back this note. |
| `InsufficientInitialLiquidity()` | The first deposit must exceed the permanently locked minimum. |
| `InvalidStrike()` | A strike at or above the cap leaves no room for the note to pay anything. |
| `InvalidTerm()` | The term bounds are the wrong way round, or a term of zero was allowed. |
| `NoSuchNote()` | There is no note at that index, or it has already settled. |
| `NotExpired(uint64)` | The note has not reached its expiry. |
| `SafeCastOverflowedIntToUint(int256)` | An int value doesn't fit in a uint of `bits` size. |
| `SafeCastOverflowedUintDowncast(uint8,uint256)` | Value doesn't fit in a uint of `bits` size. |
| `SafeERC20FailedOperation(address)` | An operation with an ERC-20 token failed. |
| `TermOutOfRange(uint64,uint64)` | The requested term is outside what this hook writes. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 2 of the fourteen:

- `afterInitialize`
- `beforeSwap`

Mask: `0x1080`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # VarianceSwap
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # derivatives, volatility, variance-swap, oracle-free, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/variance-swap
cd variance-swap
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
