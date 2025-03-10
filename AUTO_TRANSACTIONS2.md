# AUTOMATED TOKEN TRANSFERS v2
Updated version of AUTO_TRANSFERS with dynamic gas calculation.

```javascript
// trying dynamic gas calculation
import { ethers } from "ethers";

// Configurations
const RPC_URL = "https://rpc-testnet-base.worldmobile.net"; // Replace with your RPC URL
const TOKEN_CONTRACT_ADDRESS = "0x8dEE6e58B7278fBAAd457c6D627Ecd0729109D98"; // Token contract
const TOKEN_ABI = [
    "function transfer(address to, uint256 amount) public returns (bool)",
];
const PRIVATE_KEY = "<YOUR_PRIVATE_KEY_FOR_SEND_WALLET>";       // Private send key
const NEXT_WALLET_ADDRESS = "<PUBLIC_KEY_FOR_RECEIVE_WALLET>";  // Public receive key
const AMOUNT = ethers.parseUnits("100", 18); // Adjust decimals as needed
const BATCH_SIZE = 25; // Number of transactions per batch

// Set up provider and wallet
const provider = new ethers.JsonRpcProvider(RPC_URL);
const wallet = new ethers.Wallet(PRIVATE_KEY).connect(provider);
const tokenContract = new ethers.Contract(TOKEN_CONTRACT_ADDRESS, TOKEN_ABI, wallet);

// Function to calculate the optimal gas limit
async function getDynamicGasLimit() {
    const block = await provider.getBlock("latest");
    const blockGasLimit = block.gasLimit; // ethers.BigNumber
    const blockGasLimitBigInt = BigInt(blockGasLimit.toString()); // Convert to BigInt

    const gasEstimate = await tokenContract.estimateGas.transfer(NEXT_WALLET_ADDRESS, AMOUNT, {
        from: wallet.address,
    });

    const gasLimit = BigInt(gasEstimate.toString()) * 15n / 10n;
    return gasLimit < blockGasLimitBigInt ? gasLimit : blockGasLimitBigInt - 10000n; // Leave a margin
}

// Function to send tokens in batches
async function sendTokens() {
    const startingNonce = await provider.getTransactionCount(wallet.address, "latest");
    console.log(`Starting nonce: ${startingNonce}`);

    let nonce = startingNonce;
    const pendingTxs = [];

    for (let i = 0; i < BATCH_SIZE; i++) {
        try {
            console.log(`Preparing transaction ${i + 1}...`);
            const gasLimit = await getDynamicGasLimit();
            const tx = await tokenContract.transfer(NEXT_WALLET_ADDRESS, AMOUNT, {
                gasLimit: gasLimit.toString(), // Convert back to string for ethers.js
                nonce: nonce++, // Use and increment nonce manually
            });
            console.log(`Transaction ${i + 1} sent: ${tx.hash}`);
            pendingTxs.push(tx);
        } catch (err) {
            console.error(`Error preparing transaction ${i + 1}:`, err);
        }
    }

    const receipts = await Promise.all(
        pendingTxs.map((tx) =>
            tx
                .wait()
                .then((receipt) => {
                    console.log(`Transaction mined: ${receipt.transactionHash}`);
                    return receipt;
                })
                .catch((err) => {
                    console.error(`Error waiting for transaction:`, err);
                })
        )
    );

    console.log(`Batch completed. ${receipts.filter(Boolean).length} transactions confirmed.`);
}

(async () => {
    while (true) {
        try {
            console.log(`Starting new batch...`);
            await sendTokens();
        } catch (err) {
            console.error("Error during batch processing:", err);
        }
        await new Promise((resolve) => setTimeout(resolve, 1000));
    }
})();
```
