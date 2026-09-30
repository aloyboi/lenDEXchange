# lenDEXchange

A decentralised exchange (DEX) with built-in lending, built as my Final Year Project for the Bachelor of Computer Science (Cyber Security) at Nanyang Technological University (Aug 2022 – May 2023).

lenDEXchange combines two DeFi building blocks in a single app:

- **An on-chain limit order book exchange** where users trade ERC-20 tokens and ETH by placing limit buy and sell orders.
- **A lending platform** built on **Aave V3**, where users supply DAI to earn interest and receive interest-bearing **aDAI** in return.

The two are linked through trading fees: users can deposit their aDAI into the exchange to **waive trading fees**, so the yield they earn from lending pays for their trading.

---

## Features

**Exchange**
- Deposit and withdraw ETH and verified ERC-20 tokens (LINK, DAI, USDC) into an on-chain wallet
- Create limit buy and limit sell orders for any verified token pair
- Partial and full order fills, with fully filled orders closed automatically
- Cancel open orders, releasing the locked funds
- Order history and filled-order history per user

**Lending**
- Supply DAI to Aave V3 and receive aDAI
- View the current DAI supply rate and your total collateral

**Fees**
- 0.1% trading fee, valued in USD using Chainlink price feeds
- Optional fee waiver paid in aDAI, when the user holds enough to cover it

**Wallet**
- MetaMask connection via ethers.js, including account-switch handling
- Every transaction is signed by the user in MetaMask

---

## Architecture

```
                 ┌──────────────────────────────┐
                 │   React front end (ethers.js) │
                 │   MetaMask signs every tx     │
                 └───────┬───────────────┬───────┘
                         │               │
          ┌──────────────▼───┐     ┌─────▼──────────────┐
          │  Exchange.sol    │     │  Aave V3 Pool       │
          │  order book      │     │  supply DAI → aDAI  │
          └───┬─────────┬────┘     └─────────────────────┘
              │         │
   ┌──────────▼──┐  ┌───▼────────────┐
   │ fillLogic   │  │ TradingFees    │──► PriceChecker ──► Chainlink feeds
   │ matching    │  │ 0.1%, aDAI     │
   └──────┬──────┘  └───────┬────────┘
          │                 │
          └────────┬────────┘
             ┌─────▼──────┐
             │ Wallet.sol │  custody: balances + locked funds
             └────────────┘
```

### Smart contracts (`src/contracts`)

| Contract | Role |
|---|---|
| `Wallet.sol` | Holds user funds. Tracks each user's balance per token, and **locked funds** reserved for open orders so they cannot be withdrawn. Normalises every token to 18 decimals. Guards withdrawals against reentrancy. |
| `Exchange.sol` | The order book, keyed by token pair and side (buy/sell). Creates and cancels limit orders, maintains the list of verified tokens, and records filled orders. |
| `fillLogic.sol` | Order matching. Fills a batch of orders (including partial fills), calculates fees, swaps balances between buyer and seller, and closes fully filled orders. Split out of `Exchange.sol` to stay under Ethereum's contract size limit. |
| `TradingFees.sol` | Calculates the 0.1% fee in USD and checks whether the user holds enough aDAI to waive it. |
| `PriceChecker.sol` | Registry mapping each token to its Chainlink USD price feed. |
| `ERC20.sol` | OpenZeppelin's standard ERC-20 implementation, used for token interactions. |

Deposits of ERC-20 tokens use the standard **approve → transferFrom** flow: the user first approves the Wallet contract for an amount, then the contract pulls the tokens in.

### Front end (`src`)

- React with Redux for app state
- ethers.js with `Web3Provider(window.ethereum)`, so MetaMask is the provider and signer
- `scripts/functions.js`: exchange calls (deposit, withdraw, orders, fills)
- `scripts/aaveLend.js`: Aave V3 calls (approve, supply, reserve rate, collateral)

---

## Tech stack

- **Smart contracts:** Solidity (0.8.x), OpenZeppelin, Aave V3, Chainlink
- **Development and testing:** Hardhat, Mocha/Chai, hardhat-gas-reporter, hardhat-contract-sizer
- **Front end:** React, Redux, ethers.js, MetaMask, Material UI, styled-components
- **Network:** Goerli testnet, accessed through Alchemy as the RPC provider

---

## Testing

The contracts are covered by **21 Mocha/Chai unit tests** (`src/test`), covering:

- creating limit buy and sell orders
- cancelling orders
- filling orders, including partial fills
- trading fee calculation
- wallet deposits and withdrawals

Tests run on a **local Hardhat fork of Goerli**, so they execute against the real deployed Aave V3 and Chainlink contracts without spending testnet gas. Gas usage per function is logged with hardhat-gas-reporter (see `src/gas-report.txt`).

---

## Running locally

> **Note:** This project was built in 2023 against the **Goerli** testnet, which has since been shut down. To run it today, point the config at **Sepolia** and update the Aave, Chainlink and token addresses in `src/helper-hardhat-config.js` and the deploy scripts.

**Prerequisites:** Node.js, npm, the MetaMask browser extension, and an RPC endpoint (for example from Alchemy).

1. Install dependencies from the project root:

   ```bash
   npm install
   ```

2. Create a `.env` file in `src/` with the following values:

   ```bash
   # Hardhat
   GOERLI_RPC_URL=         # your RPC endpoint, e.g. from Alchemy
   PRIVATE_KEY=            # deployer account (testnet only)
   PRIVATE_KEY_2=          # second test account
   ETHERSCAN_API_KEY=      # optional, for contract verification

   # Front end: deployed contract addresses
   REACT_APP_EXCHANGE_ADDRESS=
   REACT_APP_WALLET_ADDRESS=
   REACT_APP_FILLLOGIC_ADDRESS=
   REACT_APP_PRICECHECKER_ADDRESS=
   REACT_APP_ADMIN_ADDRESS=
   REACT_APP_LENDINGPOOLADDRESSPROVIDER=
   REACT_APP_LENDINGPOOLADDRESSV3=      # Aave V3 Pool
   ```

   Never commit this file or use a private key that holds real funds.

3. Compile, test, and deploy the contracts (Hardhat is configured in `src/`):

   ```bash
   cd src
   npx hardhat compile
   npx hardhat test
   npx hardhat run scripts/deploy/deploy-PriceChecker.js --network goerli
   npx hardhat run scripts/deploy/deploy-TradingFees.js  --network goerli
   npx hardhat run scripts/deploy/deploy-Wallet.js       --network goerli
   npx hardhat run scripts/deploy/deploy-Exchange.js     --network goerli
   ```

   After deploying, link the contracts through their owner-only setters (for example `setWalletAddress`, `updateExchangeAddress`, `setTradingFees`), register tokens with `addToken`, and add price feeds with `addPriceFeed`.

4. Start the front end from the project root:

   ```bash
   npm start
   ```

   Open the app, connect MetaMask, and switch to the network you deployed to.

---

## What I'd do differently today

- **Gas cost:** storing and matching the order book on-chain is expensive. Production DEXs usually use an AMM, or match orders off-chain and settle on-chain.
- **Safer building blocks:** use OpenZeppelin's `ReentrancyGuard` instead of a hand-rolled lock, and drop SafeMath, since Solidity 0.8 already checks for overflow.
- **Better UX with Account Abstraction:** every action currently needs its own MetaMask signature and gas paid in ETH (for example, approve then supply). With ERC-4337 smart accounts, these steps could be batched into one transaction and gas could be sponsored by a paymaster.
- **Oracle safety:** add staleness checks on Chainlink prices before using them for fees.

---

## Author

**Aloysius Seow** · [GitHub](https://github.com/aloyboi)
