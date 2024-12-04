# AUTOMATED TOKEN TRANSFERS
I am using 3 wallets to send in a loop between them forever.  They try to load the chain up by submitting multiple transactions at once (by incrementing the nonce - a counter for transactions).<br>

Possibly you could simply sent tokens to yourself from 1 wallet back to that wallet.  I haven't tested it, but it may work.<br>

## Setup environment
- install node.js and the package ethers - there are guides on the web for this, but would recommend snapshoting your VM before doing this in case you need to revert, I certainly did a few times..
  - it took me a bunch of attempts to get it correct but ultimately you want to be able to run:
```bash
node -v  #and get a version number back
npm -v   #and get a version number back

#then I dropped back a version of node.js
nvm install 20
nvm use 20  #using older version for better compatibility apparenty

npm install ethers  # has the commands we need to use

#then in the folder in installed in edit packages.json and add this line in the top half
"type": "module"
```
I wish I could help more but getting that far took so long and I didn't take notes.

## MINT YOUR TOKEN FOR SENDING (or use one you have)
- mint guide here [token minting](./TOKEN_MINT.md) 

## CREATE SCRIPT FOR SENDING REPEATEDLY
- edit the following and insert your:
   - Token contract code
   - Send Wallet PRIVATE key (not seed key, keeping clicking into wallet details in Metamask and you'll find it)
   - Receive wallet PUBLIC address<br>

```javascript
// Wallet spam transfers
import { ethers } from "ethers";

// Configurations
const RPC_URL = "https://rpc-testnet-base.worldmobile.net"; // Replace with your RPC URL
const TOKEN_CONTRACT_ADDRESS = "<YOUR_TOKEN_CONTRACT>"; // Token contract
const TOKEN_ABI = [
    "function transfer(address to, uint256 amount) public returns (bool)",
];
const PRIVATE_KEY = "<YOUR_PRIVATE_KEY_FOR_SEND_WALLET>";       // Private send key
const NEXT_WALLET_ADDRESS = "<PUBLIC_KEY_FOR_RECEIVE_WALLET>";  // Public receive key
const AMOUNT = ethers.parseUnits("100", 18); // Adjust decimals as needed
const BATCH_SIZE = 10; // Number of transactions per batch - 10 seems ok for WMC testnet

// Set up provider and wallet
const provider = new ethers.JsonRpcProvider(RPC_URL);
const wallet = new ethers.Wallet(PRIVATE_KEY).connect(provider);
const tokenContract = new ethers.Contract(TOKEN_CONTRACT_ADDRESS, TOKEN_ABI, wallet);

// Function to send tokens in batches
async function sendTokens() {
    const startingNonce = await provider.getTransactionCount(wallet.address, "latest");
    console.log(`Starting nonce: ${startingNonce}`);

    let nonce = startingNonce;
    const pendingTxs = [];

    for (let i = 0; i < BATCH_SIZE; i++) {
        try {
            console.log(`Preparing transaction ${i + 1}...`);
            const tx = await tokenContract.transfer(NEXT_WALLET_ADDRESS, AMOUNT, {
                gasLimit: 100000,
                nonce: nonce++, // Use and increment nonce manually
            });
            console.log(`Transaction ${i + 1} sent: ${tx.hash}`);
            pendingTxs.push(tx);
        } catch (err) {
            console.error(`Error preparing transaction ${i + 1}:`, err);
        }
    }

    // Wait for all transactions in the batch to be mined
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

    console.log(`Batch completed. ${receipts.length} transactions confirmed.`);
}

// Run the function
(async () => {
    while (true) {
        try {
            console.log(`Starting new batch...`);
            await sendTokens();
        } catch (err) {
            console.error("Error during batch processing:", err);
        }
        // Optional delay between batches
        await new Promise((resolve) => setTimeout(resolve, 1000));
    }
})();
```
<br>

## SAVE + RUN THE SCRIPT
- save the above file with name like transpam.js
- run it as follows:
   - ```node transpam.js```
