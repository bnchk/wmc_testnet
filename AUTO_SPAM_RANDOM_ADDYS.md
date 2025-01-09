# AUTOMATED TRANSFERS TO RANDOM ADDRESSES
This script generates random addresses and loops sending ERC20 token to them.<br>

Environment build is same as any of the other AUTO* jobs.<br>
<br>

```javascript
// OPTIMIZED BULK TOKEN SENDER
import { ethers } from "ethers";
import crypto from "crypto";

const RPC_URL = "https://rpc-testnet-base.worldmobile.net";
const TOKEN_CONTRACT_ADDRESS = "0x8dEE6e58B7278fBAAd457c6D627Ecd0729109D98";
const TOKEN_ABI = [
    "function transfer(address to, uint256 amount) public returns (bool)",
];
const PRIVATE_KEY = "6e111111111111aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa";
const AMOUNT = ethers.parseUnits("10", 18);
const BATCH_SIZE = 10; // Reduced batch size for better stability
const TOTAL_TRANSACTIONS = 10000000;

// Provider with more conservative settings
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
            await new Promise(resolve => setTimeout(resolve, 200));
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
