# Numoen contest details
- Total Prize Pool: $60,500 USDC
  - HM awards: $42,500 USDC 
  - QA report awards: $5,000 USDC 
  - Gas report awards: $2,500 USDC 
  - Judge + presort awards: $10,000 USDC
  - Scout awards: $500 USDC 
- Join [C4 Discord](https://discord.gg/code4rena) to register
- Submit findings [using the C4 form](https://code4rena.com/contests/2023-01-numoen-contest/submit)
- [Read our guidelines for more details](https://docs.code4rena.com/roles/wardens)
- Starts January 26, 2023 20:00 UTC
- Ends February 1, 2023 20:00 UTC

## C4udit / Publicly Known Issues

The C4audit output for the contest can be found [here](add link to report) within an hour of contest opening.

*Note for C4 wardens: Anything included in the C4udit output is considered a publicly known issue and is ineligible for awards.*

# Overview

Numoen Core is a protocol for the permissionless creation of option-like leverage tokens called Power Tokens that are enabled by the borrowing and lending of automated market maker (AMM) shares. The protocol implements a capped power invariant, introduced in the paper `Replicating Monotonic Payoffs Without Oracles`, that allows lenders to provide two tokens to a pool of liquidity that always rebalances to a desired portfolio value via arbitrageurs. This portfolio value corresponds to a payoff that when inverted replicates the payoff of a power perpetual to some bound. Numoen Core achieves the power perpetual payoff through the AMM's LP shares that are lend out and used to mint Power Tokens. Borrowers provide collateral according to strict requirements and borrow the maximum amount of AMM shares from the pool. Funding rates are determined using the jump rate model with fixed parameters. The jump rate model is identical in structure to that of the Compound Protocol with changes made to the parameters so that it relates to the implied volitity of the LP share. Borrowers also pay interest by decreasing the overall size of their position and giving the collateral to lenders. Numoen allows for the permissionless creation of pairs using the factory model.

Numoen has [docs](https://numoen.gitbook.io/numoen/) but most information pertaining to the smart contracts are not relevant as the codebase documented is outdated. The newer version that is being audited through this contest is more robust and efficient. Therefore documentation on the smart contracts are most accurate here.

## Protocol functionality overview

### Factory

A new instance of a market is created using the factory. Token1 is the speculative token and token0 is the base token. The upper bound is the price at which the Power Token only holds the base token (token0) and the Power Token no longer has convexity. Token scales are meant to be decimals.

### Pair

Liquidity providers provide liquidity to an AMM with a custom invariant. The invariant is documented in the function `invariant` in `Pair.sol`. The typical `Mint`, `Burn`, and `Swap` functions are implemented. Swap is externally exposed so that accounts can swap between the underlying tokens of the pool with any trade that upholds the invariant. Callbacks are used to allow for flash swaps. Mint is not externally exposed and is called by a higher level function in `Lendgine.sol`. Mint also uses callbacks to receive the tokens that are deposited which enables liquidity to be minted before supplying the underlying tokens. Mint checks that the deposited tokens in addition to the requested liquidity still satisfies the invariant or else reverts. Burn removes liquidity and transfers the underlying tokens to the recipient, while performing an extra, potentially unnecessary, check that the invariant is satisfied after the outputs are removed.

### Providing Liquidity

Liquidity positions are recorded with a size, tokensOwed, and rewardPerPositionPaid in `Position.sol`. This is the same algorithm used by Synthetix `StakingRewards.sol`. Size is a different unit than shares of the AMM because size accounts for the dilution that liquidity providers undergo, explained further in the interest section. Liquidity is provided through the `deposit` function in `Lendgine.sol`, which calculates the size of the liquidity, updates the position struct, and calles the underlying `mint` function in `Pair.sol`. Withdraw performs the opposite function, calculating how many shares of the AMM are proportional to the size of the position being withdrawn, updating the position struct, and calls the underlying `burn` function in `Pair.sol`.

### Borrowing

Power Tokens are created by using token1 as collateral to borrow LP shares. Our invariant has the special property that underlying composition can be entirely token1 without needing an infinite amount of token0. This is similar to a bounded UniswapV3 position but dissimilar from UniswapV2. This means that we can determine an amount of token1 such that the value of that amount is greater than the value a LP share, no matter the exchange rate of the two underlying tokens. Thus, under collateralization is not possible with the correct amount of collateral. The `mint` function determines how many LP shares are to be borrowed for the specified amount of collateral and then calls the underlying `burn` function in `Pair.sol` to remove the borrowed liquidity. The collateral is passed in through a callback function, which allows for liquidity to be optimistically borrowed, then paid for. Again, the size of the position is not directly proportional to the amount of liquidity being borrowed, so the amount of shares that a minter receives must be calculated. Power Tokens are minted as an ERC20 representing collateral in token1 and debt in LP shares. Power Tokens can be burned by transferring them to the `Lendgine.sol` contract first, calculating the amount of liquidity owed, then calling the `mint` function in `Pair.sol` to payback the LP share debt owed and unlock the collateral of the position.

### Interest


The jump rate model is used to determine the interest rate. Interest is accrued from Power Token holders to liquidity providers in the form of token1. When interest is accrued, the amount of LP shares and speculative tokens that should be removed from Power Token holders is determined. The collateral and debt of the options holders are decreased simultaneously. The debt of Power Token holders is forgiven, meaning that liquidity providers are not expecting to be repaid. This is why the size of a liquidity position is not equivalent to the LP shares that originally were deposited. To makeup for slowly decreasing amount of LP shares, liquidity providers are given the collateral removed from the Power Token holders.

In other terms, liquidity providers are slowly exchanging their liquidity for the collateral of Power Token holders. A Power Token position is gradually worth less and less because it represents a claim to a smaller pool of collateral and debt. A liquidity provider position is gradually worth less and less because it represents a claim to a smaller pool of AMM shares but this is made up because over time it is rewarded with token1 from the collateral of Power Token holders.

There is a special case when all liquidity currently borrowed is accrued at once. As long as liquidity is accrued somewhat frequently this should not happen. When this does happen, all Power Token positions are worth nothing and all LP positions are worth only the token1 that is owed to it. There are special checks in the `mint` and `deposit` function in `Lendgine.sol` that disable opening new Power Tokens or LP positions because the amount to be rewards is not able to be determined. The market is effectively done at this point in time and would require a redeployment to be restarted from scratch.

### Liquidity Manager

The `LiquidityManager` contract provides some helpers to aid with entering, exiting, and managing a LP position in Numoen. This adds checks for stale transactions, slippage, handling permit functions and native tokens.

### Lendgine Router

The `LendgineRouter` contract provides help when entering or exiting an option position. Checks for staleness, slippage, handling permit functions, and native tokens are included. This can also perform the leveraging and deleveraging of Power Token positions with the help of external liquidity pools such as UniswapV2 style pools and UniswapV3 style pools. This is somewhat similar to looping through compound while trading the borrowed token for collateral and then borrowing more. `mint` takes the borrowed liquidity and transfers it entirely into token1 for collateral. A borrowed amount can be passed in such that liquidity can be optimistically borrowed for more collateral than the option depositor has at the moment, the underlying liquidity is then swapped entirely for token1 to use as collateral in combination with collateral from the user. `burn` optimistically mints a liquidity position and repays debt, then uses the unlocked collateral to come up with the underlying for the liquidity position that was minted.

# Scope

The following directories and implementations are considered in-scope for this audit.

For the Protocol Implementation, here's a brief description of each file.

|File|[SLOC](#nowhere "(nSLOC, SLOC, Lines)")|Description|Libraries|
|:-|:-:|:-|:-|
|_Contracts (4)_|
|[src/core/Factory.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/Factory.sol) [🧮](#nowhere "Uses Hash-Functions")|[50](#nowhere "(nSLOC:40, SLOC:50, Lines:89)")|Deploys lendgine markets||
|[src/core/Lendgine.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/Lendgine.sol)|[165](#nowhere "(nSLOC:139, SLOC:165, Lines:273)")|Lending and borrowing of AMM shares||
|[src/periphery/LiquidityManager.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/LiquidityManager.sol) [💰](#nowhere "Payable Functions")|[168](#nowhere "(nSLOC:168, SLOC:168, Lines:251)")|Aids with entry, exit, and management of liquidity positions||
|[src/periphery/LendgineRouter.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/LendgineRouter.sol) [💰](#nowhere "Payable Functions")|[206](#nowhere "(nSLOC:197, SLOC:206, Lines:287)")|Aids with entry and exit of options positions||
|_Abstracts (5)_|
|[src/core/ImmutableState.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/ImmutableState.sol)|[19](#nowhere "(nSLOC:19, SLOC:19, Lines:38)")|Immutables||
|[src/core/JumpRate.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/JumpRate.sol)|[34](#nowhere "(nSLOC:26, SLOC:34, Lines:44)")|Interest rate curve||
|[src/periphery/Payment.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/Payment.sol) [💰](#nowhere "Payable Functions")|[42](#nowhere "(nSLOC:42, SLOC:42, Lines:65)")|Functions to ease deposit and withdrawal of ETH||
|[src/periphery/SwapHelper.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/SwapHelper.sol)|[72](#nowhere "(nSLOC:72, SLOC:72, Lines:120)")|Facilitates swapping on external liquidity sources||
|[src/core/Pair.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/Pair.sol)|[81](#nowhere "(nSLOC:81, SLOC:81, Lines:140)")|Implements AMM with Capped Power Invariant||
|_Libraries (6)_|
|[src/core/libraries/PositionMath.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/libraries/PositionMath.sol)|[10](#nowhere "(nSLOC:10, SLOC:10, Lines:19)")|Math for liquidity positions||
|[src/libraries/Balance.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/libraries/Balance.sol) [🧮](#nowhere "Uses Hash-Functions")|[10](#nowhere "(nSLOC:10, SLOC:10, Lines:18)")|Reads token balances||
|[src/libraries/SafeCast.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/libraries/SafeCast.sol)|[10](#nowhere "(nSLOC:10, SLOC:10, Lines:19)")|Cast Solidity types||
|[src/periphery/libraries/LendgineAddress.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/libraries/LendgineAddress.sol) [🧮](#nowhere "Uses Hash-Functions")|[32](#nowhere "(nSLOC:21, SLOC:32, Lines:36)")|Computes Numoen Lendgine addresses||
|[src/core/libraries/Position.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/libraries/Position.sol)|[62](#nowhere "(nSLOC:39, SLOC:62, Lines:97)")|Liquidity position handler||
|[src/periphery/UniswapV2/libraries/UniswapV2Library.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/UniswapV2/libraries/UniswapV2Library.sol) [🧮](#nowhere "Uses Hash-Functions")|[70](#nowhere "(nSLOC:46, SLOC:70, Lines:84)")|Modified V2 Library for Solidity 0.8||
|Total (over 15 files):| [1031](#nowhere "(nSLOC:920, SLOC:1031, Lines:1580)") ||


## Out of scope

|File|[SLOC](#nowhere "(nSLOC, SLOC, Lines)")|Description|Libraries|
|:-|:-:|:-|:-|
|_Abstracts (4)_|
|[src/core/ReentrancyGuard.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/ReentrancyGuard.sol)|[10](#nowhere "(nSLOC:10, SLOC:10, Lines:20)")|||
|[src/periphery/Multicall.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/Multicall.sol) [🖥](#nowhere "Uses Assembly") [💰](#nowhere "Payable Functions") [👥](#nowhere "DelegateCall") [Σ](#nowhere "Unchecked Blocks")|[19](#nowhere "(nSLOC:19, SLOC:19, Lines:29)")|||
|[src/periphery/SelfPermit.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/SelfPermit.sol) [💰](#nowhere "Payable Functions")|[22](#nowhere "(nSLOC:12, SLOC:22, Lines:44)")|||
|[src/core/ERC20.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/ERC20.sol) [🧮](#nowhere "Uses Hash-Functions") [🔖](#nowhere "Handles Signatures: ecrecover") [Σ](#nowhere "Unchecked Blocks")|[98](#nowhere "(nSLOC:87, SLOC:98, Lines:190)")|||
|_Libraries (4)_|
|[src/periphery/UniswapV3/libraries/TickMath.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/UniswapV3/libraries/TickMath.sol)|[7](#nowhere "(nSLOC:7, SLOC:7, Lines:17)")|||
|[src/periphery/UniswapV3/libraries/PoolAddress.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/UniswapV3/libraries/PoolAddress.sol) [🧮](#nowhere "Uses Hash-Functions")|[27](#nowhere "(nSLOC:27, SLOC:27, Lines:43)")|||
|[src/libraries/FullMath.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/libraries/FullMath.sol) [🖥](#nowhere "Uses Assembly") [Σ](#nowhere "Unchecked Blocks")|[56](#nowhere "(nSLOC:56, SLOC:56, Lines:129)")|||
|[src/libraries/SafeTransferLib.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/libraries/SafeTransferLib.sol) [🖥](#nowhere "Uses Assembly")|[56](#nowhere "(nSLOC:56, SLOC:56, Lines:116)")|||
|_Interfaces (24)_|
|[src/core/interfaces/callback/IPairMintCallback.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/interfaces/callback/IPairMintCallback.sol)|[4](#nowhere "(nSLOC:4, SLOC:4, Lines:10)")|||
|[src/core/interfaces/callback/ISwapCallback.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/interfaces/callback/ISwapCallback.sol)|[4](#nowhere "(nSLOC:4, SLOC:4, Lines:10)")|||
|[src/periphery/UniswapV3/interfaces/callback/IUniswapV3SwapCallback.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/UniswapV3/interfaces/callback/IUniswapV3SwapCallback.sol)|[4](#nowhere "(nSLOC:4, SLOC:4, Lines:17)")|||
|[src/periphery/interfaces/IMulticall.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/interfaces/IMulticall.sol) [💰](#nowhere "Payable Functions")|[4](#nowhere "(nSLOC:4, SLOC:4, Lines:13)")|||
|[src/periphery/interfaces/external/IWETH9.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/interfaces/external/IWETH9.sol) [💰](#nowhere "Payable Functions")|[5](#nowhere "(nSLOC:5, SLOC:5, Lines:11)")|||
|[src/core/interfaces/IJumpRate.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/interfaces/IJumpRate.sol)|[8](#nowhere "(nSLOC:8, SLOC:8, Lines:18)")|||
|[src/core/interfaces/IImmutableState.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/interfaces/IImmutableState.sol)|[9](#nowhere "(nSLOC:9, SLOC:9, Lines:24)")|||
|[src/core/interfaces/IPair.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/interfaces/IPair.sol)|[9](#nowhere "(nSLOC:9, SLOC:9, Lines:26)")|||
|[src/periphery/UniswapV3/interfaces/pool/IUniswapV3PoolImmutables.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/UniswapV3/interfaces/pool/IUniswapV3PoolImmutables.sol)|[9](#nowhere "(nSLOC:9, SLOC:9, Lines:35)")|||
|[src/core/interfaces/callback/IMintCallback.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/interfaces/callback/IMintCallback.sol)|[11](#nowhere "(nSLOC:4, SLOC:11, Lines:17)")|||
|[src/periphery/UniswapV3/interfaces/pool/IUniswapV3PoolOwnerActions.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/UniswapV3/interfaces/pool/IUniswapV3PoolOwnerActions.sol)|[11](#nowhere "(nSLOC:5, SLOC:11, Lines:25)")|||
|[src/periphery/UniswapV2/interfaces/IUniswapV2Factory.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/UniswapV2/interfaces/IUniswapV2Factory.sol)|[14](#nowhere "(nSLOC:14, SLOC:14, Lines:26)")|||
|[src/periphery/UniswapV3/interfaces/IUniswapV3Factory.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/UniswapV3/interfaces/IUniswapV3Factory.sol)|[14](#nowhere "(nSLOC:14, SLOC:14, Lines:66)")|||
|[src/periphery/UniswapV3/interfaces/pool/IUniswapV3PoolDerivedState.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/UniswapV3/interfaces/pool/IUniswapV3PoolDerivedState.sol)|[14](#nowhere "(nSLOC:5, SLOC:14, Lines:43)")|||
|[src/periphery/interfaces/ISelfPermit.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/interfaces/ISelfPermit.sol) [💰](#nowhere "Payable Functions")|[14](#nowhere "(nSLOC:5, SLOC:14, Lines:33)")|||
|[src/periphery/interfaces/external/IERC20PermitAllowed.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/interfaces/external/IERC20PermitAllowed.sol)|[14](#nowhere "(nSLOC:4, SLOC:14, Lines:28)")|||
|[src/periphery/UniswapV3/interfaces/IUniswapV3Pool.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/UniswapV3/interfaces/IUniswapV3Pool.sol)|[15](#nowhere "(nSLOC:15, SLOC:15, Lines:22)")|||
|[src/periphery/interfaces/external/IERC20Permit.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/interfaces/external/IERC20Permit.sol)|[15](#nowhere "(nSLOC:6, SLOC:15, Lines:61)")|||
|[src/core/interfaces/ILendgine.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/interfaces/ILendgine.sol)|[20](#nowhere "(nSLOC:20, SLOC:20, Lines:81)")|||
|[src/core/interfaces/IFactory.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/interfaces/IFactory.sol)|[26](#nowhere "(nSLOC:6, SLOC:26, Lines:41)")|||
|[src/periphery/UniswapV3/interfaces/pool/IUniswapV3PoolActions.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/UniswapV3/interfaces/pool/IUniswapV3PoolActions.sol)|[34](#nowhere "(nSLOC:10, SLOC:34, Lines:103)")|||
|[src/periphery/UniswapV3/interfaces/pool/IUniswapV3PoolEvents.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/UniswapV3/interfaces/pool/IUniswapV3PoolEvents.sol)|[44](#nowhere "(nSLOC:44, SLOC:44, Lines:113)")|||
|[src/periphery/UniswapV3/interfaces/pool/IUniswapV3PoolState.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/UniswapV3/interfaces/pool/IUniswapV3PoolState.sol)|[47](#nowhere "(nSLOC:12, SLOC:47, Lines:118)")|||
|[src/periphery/UniswapV2/interfaces/IUniswapV2Pair.sol](https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/UniswapV2/interfaces/IUniswapV2Pair.sol)|[52](#nowhere "(nSLOC:43, SLOC:52, Lines:82)")|||
|Total (over 32 files):| [696](#nowhere "(nSLOC:537, SLOC:696, Lines:1611)") ||


## Scoping Details 
```
- If you have a public code repo, please share it here: N/A
- How many contracts are in scope?: 31  
- Total SLoC for these contracts?: 1,014
- How many external imports are there?: 2 
- How many separate interfaces and struct definitions are there for the contracts within scope?: 12
- Does most of your code generally use composition or inheritance?: Inheritance
- How many external calls?: 2  
- What is the overall line coverage percentage provided by your tests?: 0 
- Is there a need to understand a separate part of the codebase / get context in order to audit this part of the protocol?: No  
- Please describe required context: N/A 
- Does it use an oracle?: No  
- Does the token conform to the ERC20 standard?: The option perps conform to the ERC-20 standard. But we have no governance token.  
- Are there any novel or unique curve logic or mathematical models?: We use the "capped power" invariant from the Replicating Monotonic Payoffs paper: k = x − (p_0 + (-1/2) * y)^2
- Does it use a timelock function?: No
- Is it an NFT?: No
- Does it have an AMM?: Yes it's a fully autonomous AMM.  
- Is it a fork of a popular project?: No 
- Does it use rollups?: No
- Is it multi-chain?: No
- Does it use a side-chain?: No
```

# Tests

Numoen runs on [Foundry](https://github.com/foundry-rs/foundry). If you don't have it installed, follow the installation instructions [here](https://book.getfoundry.sh/getting-started/installation).

To install contract dependencies, run:

```sh
forge install
yarn
```

Some tests are reliant upon a fork of goerli. Add the goerli RPC to a .env file. See .env.example for an example. To run tests, run:

`forge test --gas-report`

The following test currently fails and has no workaround:

```
Encountered 1 failing test in src/core/Lendgine.sol:Lendgine
[FAIL. Reason: EvmError: Revert] setUp() (gas: 0)

Encountered a total of 1 failing tests, 103 tests succeeded
```


## Quickstart command

`export RPC_URL_GOERLI="<your-goerli-rpc-url-goes-here>" && rm -Rf 2023-01-numoen || true && git clone https://github.com/code-423n4/2023-01-numoen.git -j8 --recurse-submodules && cd 2023-01-numoen && echo "RPC_URL_GOERLI=$RPC_URL_GOERLI" > .env && forge install && yarn && forge test --gas-report`

