# Repo setup

## ⭐️ Sponsor: Add code to this repo

- [ ] Create a PR to this repo with the below changes:
- [ ] Provide a self-contained repository with working commands that will build (at least) all in-scope contracts, and commands that will run tests producing gas reports for the relevant contracts.
- [ ] Make sure your code is thoroughly commented using the [NatSpec format](https://docs.soliditylang.org/en/v0.5.10/natspec-format.html#natspec-format).
- [ ] Please have final versions of contracts and documentation added/updated in this repo **no less than 24 hours prior to contest start time.**
- [ ] Be prepared for a 🚨code freeze🚨 for the duration of the contest — important because it establishes a level playing field. We want to ensure everyone's looking at the same code, no matter when they look during the contest. (Note: this includes your own repo, since a PR can leak alpha to our wardens!)


---

## ⭐️ Sponsor: Edit this README

Under "SPONSORS ADD INFO HERE" heading below, include the following:

- [ ] Modify the bottom of this `README.md` file to describe how your code is supposed to work with links to any relevent documentation and any other criteria/details that the C4 Wardens should keep in mind when reviewing. ([Here's a well-constructed example.](https://github.com/code-423n4/2022-08-foundation#readme))
  - [ ] When linking, please provide all links as full absolute links versus relative links
  - [ ] All information should be provided in markdown format (HTML does not render on Code4rena.com)
- [ ] Under the "Scope" heading, provide the name of each contract and:
  - [ ] source lines of code (excluding blank lines and comments) in each
  - [ ] external contracts called in each
  - [ ] libraries used in each
- [ ] Describe any novel or unique curve logic or mathematical models implemented in the contracts
- [ ] Does the token conform to the ERC-20 standard? In what specific ways does it differ?
- [ ] Describe anything else that adds any special logic that makes your approach unique
- [ ] Identify any areas of specific concern in reviewing the code
- [ ] Optional / nice to have: pre-record a high-level overview of your protocol (not just specific smart contract functions). This saves wardens a lot of time wading through documentation.
- [ ] See also: [this checklist in Notion](https://code4rena.notion.site/Key-info-for-Code4rena-sponsors-f60764c4c4574bbf8e7a6dbd72cc49b4#0cafa01e6201462e9f78677a39e09746)
- [ ] Delete this checklist and all text above the line below when you're ready.

---

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
- Starts January 24, 2023 20:00 UTC
- Ends January 30, 2023 20:00 UTC

## C4udit / Publicly Known Issues

The C4audit output for the contest can be found [here](add link to report) within an hour of contest opening.

*Note for C4 wardens: Anything included in the C4udit output is considered a publicly known issue and is ineligible for awards.*

[ ⭐️ SPONSORS ADD INFO HERE ]

# Overview

Numoen Core is a protocol for the permissionless creation of option-like leverage tokens that are enabled by the borrowing and lending of automated market maker (AMM) shares. The protocol implements a capped power invariant, introduced in the paper `Replicating Monotonic Payoffs Without Oracles`, that allows lenders to provide two tokens to a pool of liquidity that always rebalances to a desired portfolio value via arbitrageurs. This portfolio value corresponds to a payoff that when inverted replicates the payoff of a power perpetual. Numoen Core achieves the power perpetual payoff through the AMM's LP shares that are lend out and used to mint the leveraged tokens we call Power Tokens. Borrowers provide collateral according to strict requirements and borrow the maximum amount of AMM shares from the pool which results in a payoff that replicated a power perpetual up to some bound. Funding rates are determined using the jump rate model with fixed parameters. Borrowers pay interest by decreasing the overall size of their position and giving the collateral to lenders. Numoen allows for the permissionless creation of pairs using the factory model.

Numoen has [docs](https://numoen.gitbook.io/numoen/) but most information retaining to smart contracts in the documentation is not relevant because the codebase undergoing the contest is updated from the beta version of the contracts to be more robust and efficient. However the explanation of the core mechanism is still relevant.

## Protocol functionality overview

### Factory

A new instance of a market is created using the factory. Token1 is the speculative token and token0 is the base token or numeraire. The upper bound is the price at which the AMM underlying consists of entirely token0 or the option loses convexity. Token scales are meant to be decimals.

### Pair

Liquidity providers provide liquidity to an AMM with a custom invariant. The invariant is documented in the function `invariant` in `Pair.sol`. The typical `Mint`, `Burn`, and `Swap` functions are implemented. Swap is externally exposed so that accounts can swap between the underlying tokens of the pool with any trade that upholds the invariant. Callbacks are used to allow for flash swaps. Mint is not externally exposed and is called by a higher level function in `Lendgine.sol`. Mint also uses callbacks to receive the tokens that are deposited which enables liquidity to be minted before supplying the underlying tokens. Mint checks that the deposited tokens in addition to the requested liquidity still satisfies the invariant or else reverts. Burn removes liquidity and transfers the underlying tokens to the recipient, while performing an extra, potentially unnecessary, check that the invariant is satisfied after the outputs are removed.

### Providing Liquidity

Liquidity positions are recorded with a size, tokensOwed, and rewardPerPositionPaid in `Position.sol`. This is the same algorithm used by Synthetix `StakingRewards.sol`. Size is a different unit than shares of the AMM where size accounts for the dilution that liquidity providers undergo, explained further in the interest section. Liquidity is provided through the `deposit` function in `Lendgine.sol`, which calculated the size of the liquidity, updates the position struct, and called the underlying `mint` function in `Pair.sol`. Withdraw performs the opposite function, calculating how many shares of the AMM are proportional to the size of the position being withdrawn, updating the position struct, and calls the underlying `burn` function in `Pair.sol`.

### Borrowing 

Options positions are created by using token1 as collateral to borrow AMM shares. Our invariant has the special property that underlying composition can be entirely token1 without needing an infinite amount of token0. This is similar to a bounded UniswapV3 position but dissimilar from UniswapV2. This means that we can determine an amount of token1 such that the value of that amount is greater than the value an AMM share, no matter the exchange rate of the two underlying tokens. Thus, under collateralization is not possible with the correct amount of collateral. The `mint` function determines how many AMM shares are to be borrowed for the specified amount of collateral and then calls the underlying `burn` function in `Pair.sol` to remove the borrowed liquidity. The collateral is passed in through a callback function, which allows for liquidity to be optimistically borrowed, then paid for. Again, the size of the position is not directly proportional to the amount of liquidity being borrowed, so the amount of shares that a minter receives must be calculated. Options shares are minted as an ERC20. representing collateral in token1 and debt in AMM shares. Option shares can be burned by transferring them to the `Lendgine.sol` contract first, calculating the amount of liquidity owed, then calling the `mint` function in `Pair.sol` to payback the AMM share debt owed  and unlock the collateral of the position.

### Interest

Interest is accrued from options holders to liquidity providers in the form of token1. The jump rate model is used to determine the interest rate. When interest is accrued, the amount of AMM shares and speculative tokens that are removed from options holders is determined. The collateral and debt of the options holders simultaneously. The debt of options holders is forgiven, meaning that liquidity providers are not expecting to be repaid. This is why the size of a liquidity position is not equivalent to the AMM shares that originally were deposited. To makeup for slowly decreasing amount of AMM shares liquidity providers have, the collateral from the options holders that is to be removed is given to the liquidity providers.

In other terms, liquidity providers are slowly exchanging their liquidity for the collateral of options holders (liquidity borrowers). An option position is gradually over time worth less and less because it represents a claim to a smaller pool of collateral and debt. A liquidity provider position is gradually over time worth less and less because it represents a claim to a smaller pool of AMM shares but this is made up because over time it is rewarded with token1 from the collateral of options holders.

There is a special case when all liquidity currently borrowed is accrued at once. As long as liquidity is accrued somewhat frequently this should not happen. When this does happen, all options positions are worth nothing and all liquidity positions are worth only the token1 that is owed to it. There are special checks in the `mint` and `deposit` function in `Lendgine.sol` that disable opening new options or liquidity positions because the amount to be rewards is not able to be determined. The market is effectively done at this point in time and would require a redeployment to be restarted from scratch.

### Liquidity Manager

The `LiquidityManager` contract provides some helpers to aid with entering, exiting, and managing a liquidity provider position in Numoen. This adds checks for stale transactions, slippage, handling permit functions and native tokens.

### Lendgine Router

The `LendgineRouter` contract provides help when entering or exiting an option position. Checks for staleness, slippage, handling permit functions, and native tokens are included. This can also perform the leveraging and deleveraging of options positions with the help of external liquidity pools such as UniswapV2 style pools and UniswapV3 style pools. This is somewhat similar to looping through compound while trading the borrowed token for collateral and then borrowing more. `mint` takes the borrowed liquidity and transfers it entirely into token1 for collateral. A borrowed amount can be passed in such that liquidity can be optimistically borrowed for more collateral than the option depositor has at the moment, the underlying liquidity is then swapped entirely for token1 to use as collateral in combination with collateral from the user. `burn` optimistically mints a liquidity position and repays debt, then uses the unlocked collateral to come up with the underlying for the liquidity position that was minted. 

# Scope

*List all files in scope in the table below (along with hyperlinks) -- and feel free to add notes here to emphasize areas of focus.*

The following directories and implementations are considered in-scope for this audit.


For the Protocol Implementation, here's a brief description of each file.

| Contract              | SLOC | Purpose                                                      | Libraries Used    |
| --------------------- | ---- | ---------------------------------------------------------    | ----------------- |
| Pair.sol              | 81   | Implements AMM with Capped Power Invariant                   |      N/A          |
| Lendgine.sol          | 139  | Lending and borrowing of AMM shares                          |      N/A          |
| Position.sol          | 50   | Liquidity position handler                                   |      N/A          |
| PositionMath.sol      | 7    | Math for liquidity positions                                 |      N/A          |
| ImmutableState.sol    | 19   | Immutables                                                   |      N/A          |
| Factory.sol           | 40   | Deploys lendgine markets                                     |      N/A          |
| JumpRate.sol          | 26   | Interest rate curve                                          |      N/A          |
| Balance.sol           | 8    | Reads token balances                                         |      N/A          |
| SafeCast.sol          | 7    | Cast Solidity types                                          |      N/A          |
| LendgineRouter.sol    | 221  | Aids with entry and exit of options positions                |      N/A          |
| LiquidityManager.sol  | 168  | Aids with entry, exit, and management of liquidity positions |      N/A          |
| Payment.sol           | 32   | Functions to ease deposit and withdrawal of ETH              |      N/A          |
| SwapHelper.sol        | 63   | Facilitates swapping on external liquidity sources           |      N/A          |
| LendgineAddress.sol   | 23   | Computes Numoen Lendgine addresses                           |      N/A          |
| UniswapV2Library.sol  | 46   | Modified V2 Library for Solidity 0.8                         |      N/A          |


## Out of scope

| Contract              |
| --------------------- |
| core/ReentrancyGaurd.sol |
| core/ERC20.sol |
| libraries/SafeTransferLib.sol |
| libraries/FullMath.sol |
| periphery/Multicall.sol |
| periphery/SelfPermit.sol |
| periphery/UniswapV3/PoolAddress.sol |
| periphery/UniswapV3/TickMath.sol|
| test/* |


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

*Provide every step required to build the project from a fresh git clone, as well as steps to run the tests with a gas report.* 

*Note: Many wardens run Slither as a first pass for testing.  Please document any known errors with no workaround.* 
