# Script Overview

This script automates interactions across several DeFi protocols, primarily focused on the following tasks:

1. **Token Approval**: Grants permission for the DEX to utilize USDC for a swap.
2. **Token Swap**: Converts USDC to LINK using a Uniswap V3-like pool.
3. **Aave Supply**: Deposits the acquired LINK tokens into the Aave protocol to earn interest.

## Workflow

1. **Contract Initialization**:
   - The script first establishes the necessary contract instances for interacting with the Uniswap V3-like pool, the swap router, and Aave's lending pool.

2. **USDC Approval**:
   - The `approveToken` function is invoked to allow the DEX's swap router to spend a specified amount of USDC.

3. **Pool Information Retrieval**:
   - The `getPoolInfo` function gathers details about the USDC/LINK token pair from the factory contract, including the pool address and token information.

4. **Swap Parameter Preparation**:
   - The script then uses the `prepareSwapParams` function to set up the parameters required for the swap, such as token addresses, fees, recipient, and the amount of USDC to be swapped.

5. **Executing the Swap**:
   - The swap operation is carried out using the `executeSwap` function, where the approved USDC is exchanged for LINK.

6. **Supplying LINK to Aave**:
   - After the swap, the acquired LINK tokens are deposited into Aave through the `supplyToAave` function.

## Diagram Illustration

```plaintext
+------------------+         +------------------+         +------------------+
|                  |         |                  |         |                  |
|      Wallet      |         |      Uniswap     |         |      Aave        |
|                  |         |                  |         |                  |
|    (Signer)      |         |    (DEX Pool)    |         | (Lending Pool)   |
|                  |         |                  |         |                  |
+------------------+         +------------------+         +------------------+
        |                           |                          |
        |                           |                          |
        v                           v                          v
+----------------+          +----------------+          +----------------+
|                |          |                |          |                |
| Approve USDC   |          | Swap USDC for  |          | Supply LINK to |
|  Token for     |          |    LINK        |          |    Aave        |
|   Spending     |          |                |          |                |
|                |          |                |          |                |
+----------------+          +----------------+          +----------------+
        |                           |                          |
        v                           v                          v
+-----------------+         +-----------------+         +-----------------+
|                 |         |                 |         |                 |
|   sendTransaction          |   sendTransaction        |   sendTransaction |
|                 |         |                 |         |                 |
|                 |         |                 |         |                 |
+-----------------+         +-----------------+         +-----------------+
```

Here's a detailed breakdown of the code, which interacts with DeFi protocols to swap tokens and then supply the swapped tokens to the Aave Lending Pool. This code involves multiple steps and integrates with various smart contracts on the Ethereum Sepolia testnet.

### Code Breakdown

#### **1. Imports and Setup**

```javascript
const ethers = require("ethers");
const FACTORY_ABI = require("./abis/factory.json");
const SWAP_ROUTER_ABI = require("./abis/swaprouter.json");
const POOL_ABI = require("./abis/pool.json");
const TOKEN_IN_ABI = require("./abis/token.json");
const AAVE_LENDING_POOL_ABI = require("./abis/aaveLendingPool.json");
const dotenv = require("dotenv");
dotenv.config();
```

- **ethers**: Library to interact with the Ethereum blockchain.
- **FACTORY_ABI, SWAP_ROUTER_ABI, POOL_ABI, TOKEN_IN_ABI, AAVE_LENDING_POOL_ABI**: ABI definitions for different smart contracts used in this script.
- **dotenv**: Loads environment variables from a `.env` file, like RPC URL and private key.

#### **2. Contract Addresses and Provider Setup**

```javascript
const POOL_FACTORY_CONTRACT_ADDRESS =
  "0x0227628f3F023bb0B980b67D528571c95c6DaC1c";
const SWAP_ROUTER_CONTRACT_ADDRESS =
  "0x3bFA4769FB09eefC5a80d6E87c3B9C650f7Ae48E";
const AAVE_LENDING_POOL_ADDRESS = "0x7d2768dE32b0b80b7a3454c06BdAc94A69DDc7A9";

const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
const factoryContract = new ethers.Contract(
  POOL_FACTORY_CONTRACT_ADDRESS,
  FACTORY_ABI,
  provider
);
const signer = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
```

- **POOL_FACTORY_CONTRACT_ADDRESS**: Address of the pool factory contract used to get pool information.
- **SWAP_ROUTER_CONTRACT_ADDRESS**: Address of the swap router contract used for token swaps.
- **AAVE_LENDING_POOL_ADDRESS**: Address of the Aave Lending Pool contract where tokens will be supplied.
- **provider**: Connects to the Ethereum Sepolia testnet using the provided RPC URL.
- **factoryContract**: Instance of the factory contract to interact with.
- **signer**: Wallet instance used to sign transactions.

#### **3. Token Configuration**

```javascript
const USDC = {
  chainId: 11155111,
  address: "0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238",
  decimals: 6,
  symbol: "USDC",
  name: "USD//C",
  isToken: true,
  isNative: true,
  wrapped: false,
};

const LINK = {
  chainId: 11155111,
  address: "0x779877A7B0D9E8603169DdbD7836e478b4624789",
  decimals: 18,
  symbol: "LINK",
  name: "Chainlink",
  isToken: true,
  isNative: true,
  wrapped: false,
};
```

- **USDC** and **LINK**: Objects containing the configuration for the USDC and LINK tokens, including their addresses, decimals, symbols, and names.

#### **4. Approve Token Function**

```javascript
async function approveToken(tokenAddress, tokenABI, amount, wallet) {
  try {
    const tokenContract = new ethers.Contract(tokenAddress, tokenABI, wallet);
    const approveAmount = ethers.parseUnits(amount.toString(), USDC.decimals);
    const approveTransaction = await tokenContract.approve.populateTransaction(
      SWAP_ROUTER_CONTRACT_ADDRESS,
      approveAmount
    );
    const transactionResponse = await wallet.sendTransaction(
      approveTransaction
    );
    console.log(`-------------------------------`);
    console.log(`Sending Approval Transaction...`);
    console.log(`-------------------------------`);
    console.log(`Transaction Sent: ${transactionResponse.hash}`);
    console.log(`-------------------------------`);
    const receipt = await transactionResponse.wait();
    console.log(
      `Approval Transaction Confirmed! https://sepolia.etherscan.io/tx/${receipt.hash}`
    );
  } catch (error) {
    console.error("An error occurred during token approval:", error);
    throw new Error("Token approval failed");
  }
}
```

- **approveToken**: Approves the swap router contract to spend the specified amount of tokens on behalf of the user. This is necessary to perform the swap operation.

#### **5. Get Pool Info Function**

```javascript
async function getPoolInfo(factoryContract, tokenIn, tokenOut) {
  const poolAddress = await factoryContract.getPool(
    tokenIn.address,
    tokenOut.address,
    3000
  );
  if (!poolAddress) {
    throw new Error("Failed to get pool address");
  }
  const poolContract = new ethers.Contract(poolAddress, POOL_ABI, provider);
  const [token0, token1, fee] = await Promise.all([
    poolContract.token0(),
    poolContract.token1(),
    poolContract.fee(),
  ]);
  return { poolContract, token0, token1, fee };
}
```

- **getPoolInfo**: Fetches the pool address for the given tokens from the factory contract, then retrieves the pool details (tokens and fee) from the pool contract.

#### **6. Prepare Swap Params Function**

```javascript
async function prepareSwapParams(poolContract, signer, amountIn) {
  return {
    tokenIn: USDC.address,
    tokenOut: LINK.address,
    fee: await poolContract.fee(),
    recipient: signer.address,
    amountIn: amountIn,
    amountOutMinimum: 0,
    sqrtPriceLimitX96: 0,
  };
}
```

- **prepareSwapParams**: Constructs the parameters needed for the swap transaction, including the addresses of the tokens, fee, recipient address, and amounts.

#### **7. Execute Swap Function**

```javascript
async function executeSwap(swapRouter, params, signer) {
  const transaction = await swapRouter.exactInputSingle.populateTransaction(
    params
  );
  const receipt = await signer.sendTransaction(transaction);
  console.log(`-------------------------------`);
  console.log(`Receipt: https://sepolia.etherscan.io/tx/${receipt.hash}`);
  console.log(`-------------------------------`);
}
```

- **executeSwap**: Executes the token swap using the swap router contract. The function prepares the transaction, sends it, and logs the transaction receipt.

#### **8. Supply to Aave Function**

```javascript
async function supplyToAave(tokenAddress, amount, signer) {
  try {
    // Step 1: Approve Aave to spend your tokens
    const tokenContract = new ethers.Contract(tokenAddress, TOKEN_IN_ABI, signer);
    const approveTransaction = await tokenContract.approve.populateTransaction(
      AAVE_LENDING_POOL_ADDRESS,
      amount
    );
    const approvalResponse = await signer.sendTransaction(approveTransaction);
    await approvalResponse.wait();

    // Step 2: Supply tokens to Aave Lending Pool
    const aaveLendingPool = new ethers.Contract(
      AAVE_LENDING_POOL_ADDRESS,
      AAVE_LENDING_POOL_ABI,
      signer
    );
    const supplyTransaction = await aaveLendingPool.deposit(
      tokenAddress,
      amount,
      signer.address,
      0  // Referral code
    );

    console.log('Supply Transaction:', supplyTransaction);
  
    // Wait for the transaction to be mined
    const supplyResponse = await supplyTransaction.wait();

    console.log('Supply Response:', supplyResponse);

    console.log(`Successfully supplied ${ethers.formatUnits(amount, LINK.decimals)} LINK to Aave.`);
    console.log(`Transaction Hash: https://sepolia.etherscan.io/tx/${supplyTransaction.hash}`);
  } catch (error) {
    console.error("An error occurred while supplying tokens to Aave:", error.message);
    throw new Error("Aave supply failed");
  }
}
```

- **supplyToAave**: This function handles supplying tokens to the Aave Lending Pool:
  - **Approval**: Approves the Aave Lending Pool contract to spend the tokens.
  - **Supply**: Deposits the tokens into the Aave Lending Pool.

#### **9. Main Function**

```javascript
async function main(swapAmount) {
  const inputAmount = swapAmount;
  const amountIn = ethers.parseUnits(inputAmount.toString(), USDC.decimals);

  try {
    await approveToken(USDC.address, TOKEN_IN_ABI, inputAmount, signer);
    const { poolContract } = await getPoolInfo(factoryContract, USDC, LINK);
    const params = await prepareSwapParams(poolContract, signer, amountIn);
    const swapRouter = new ethers.Contract(
      SWAP_ROUTER_CONTRACT_ADDRESS,
      SWAP_ROUTER_ABI,
      signer
    );
    await executeSwap(swapRouter, params, signer);

    // After successful swap, supply the LINK to Aave
    const amountOut = params.amountIn; // Assuming swap returns 1:1 for simplicity
    await supplyToAave(LINK.address, amountOut, signer);
  } catch (error) {
    console.error("An error occurred:", error.message);
  }
}

// Enter Swap Amount
main(1);
```

- **main**: Orchestrates the whole process:
  1. **Approve USDC**: Approves the swap

 router to spend USDC tokens.
  2. **Get Pool Info**: Retrieves the pool information needed for the swap.
  3. **Prepare Swap Params**: Prepares the parameters for the swap transaction.
  4. **Execute Swap**: Executes the token swap.
  5. **Supply to Aave**: After the swap, supplies the resulting LINK tokens to the Aave Lending Pool.

### Summary

This code demonstrates how to automate the process of swapping USDC for LINK and then supplying LINK to the Aave Lending Pool. It covers key operations such as token approval, interaction with Uniswap's swap router, and depositing tokens into a lending protocol. The code ensures transactions are properly handled and provides links to Etherscan for monitoring.