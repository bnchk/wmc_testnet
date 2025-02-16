# AUTOMATED TRANSFERS TO RANDOM ADDRESSES
V2 of script to generates completely random addresses and loop sending ERC20 token to them.<br>

## STEPS BEFOREHAND
- manually mint your ERC20 token as per [TOKEN_MINT](./TOKEN_MINT.md) so you can get contract number for in the script
- setup automation environment as per [ENV_SETUP](./README.md#automation-environment-setup-steps)

## POPULATE VARIABLES IN SCRIPT 
- update the following in the script code:
   - change PRIVATE_KEY for your wallet (hide secure details in .env file if it matters for you (not applicable for me))
   - change CONTRACT_ADDRESS for your contract
   - change BATCH_DELAY_MS to minimise pending

## HOW TO RUN
- tested with node v20
- run by `node scriptname.js`
<br>

```javascript
// OPTIMIZED BULK TOKEN SENDER with delay between batches variable
import { ethers } from "ethers";
import crypto from "crypto";
const RPC_URL = "https://rpc-testnet-base.worldmobile.net";
const TOKEN_CONTRACT_ADDRESS = "INSERT-TOKEN-CONTRACT-HERE";
const TOKEN_ABI = [
    "function transfer(address to, uint256 amount) public returns (bool)",
];
const PRIVATE_KEY = "INSERT-PRIVATE-KEY-HERE";
const AMOUNT = ethers.parseUnits("10", 18);
const BATCH_SIZE = 10; // Reduced batch size for better stability
const TOTAL_TRANSACTIONS = 100000;
const BATCH_DELAY_MS = 2000; // Delay between batches in milliseconds

// Provider
const provider = new ethers.JsonRpcProvider(RPC_URL);
const wallet = new ethers.Wallet(PRIVATE_KEY, provider);
const tokenContract = new ethers.Contract(TOKEN_CONTRACT_ADDRESS, TOKEN_ABI, wallet);
function generateRandomAddress() {
    const randomBytes = crypto.randomBytes(20);
    return ethers.getAddress("0x" + randomBytes.toString("hex"));
}
async function sendBatch(startNonce, count) {
    const txPromises = [];
    
    for (let i = 0; i < count; i++) {
        const randomAddress = generateRandomAddress();
        
        // Create transaction with explicit nonce
        const txPromise = tokenContract.transfer(randomAddress, AMOUNT, {
            gasLimit: 3000000,
            nonce: startNonce + i,
        }).then(tx => {
            console.log(`Transaction sent: ${tx.hash}`);
            return tx;
        }).catch(err => {
            console.error(`Error in transaction ${startNonce + i}:`, err);
            return null;
        });
        
        txPromises.push(txPromise);
    }
    
    return Promise.all(txPromises);
}
async function sendTokens() {
    let currentNonce = await provider.getTransactionCount(wallet.address, "latest");
    console.log(`Starting nonce: ${currentNonce}`);
    
    let processedTx = 0;
    
    while (processedTx < TOTAL_TRANSACTIONS) {
        const remainingTx = TOTAL_TRANSACTIONS - processedTx;
        const batchSize = Math.min(BATCH_SIZE, remainingTx);
        
        console.log(`Processing batch of ${batchSize} transactions starting at nonce ${currentNonce}`);
        
        try {
            const results = await sendBatch(currentNonce, batchSize);
            const successfulTxs = results.filter(tx => tx !== null);
            processedTx += successfulTxs.length;
            currentNonce += batchSize;
            
            console.log(`Progress: ${processedTx}/${TOTAL_TRANSACTIONS} transactions processed`);
            
            // Small delay between batches to allow the node to catch up
            await new Promise(resolve => setTimeout(resolve, BATCH_DELAY_MS));
        } catch (err) {
            console.error("Batch error:", err);
            // If we encounter an error, fetch the current nonce again to resync
            currentNonce = await provider.getTransactionCount(wallet.address, "latest");
            await new Promise(resolve => setTimeout(resolve, 1000)); // Longer delay after error
        }
    }
    
    console.log(`Process completed. ${processedTx} transactions processed.`);
}
// Error handling wrapper
(async () => {
    try {
        await sendTokens();
    } catch (err) {
        console.error("Fatal error during token sending:", err);
    }
})();
```
