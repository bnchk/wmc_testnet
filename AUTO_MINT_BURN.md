# AUTOMATED ERC-20 MINT/BURN

Needs contract already created as per [here](https://github.com/bnchk/wmc_testnet/blob/main/TOKEN_MINT.md) just using varient with burnable capability eg:<br>

Replace the ROASTCHOOK and RCH with whatever you want:<br>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract ROASTCHOOK is ERC20, ERC20Burnable, Ownable {
    constructor(uint256 initialSupply) ERC20("ROASTCHOOK", "RCH") Ownable(msg.sender)  {
        _mint(msg.sender, initialSupply); // Mint the initial supply to the deployer
    }

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount); // Allows the owner to mint new tokens
    }
}
```

then can use a script called say `mintburn.js` called via `node mintburn.js` as follows replacing:<br>
- PRIVATE_KEY - wallet private key
- TOKEN_ADDRESS - token contract address
- BATCH_SIZE - can scale up till??

The script sequentially mints 1000, then burns 100 of the ERC20 token:<br>

```javascript
const { ethers } = require("ethers");

// RPC Provider and wallet setup
const RPC_URL = "https://rpc-testnet-base.worldmobile.net";
const PRIVATE_KEY = "INSERT_WALLET_PRIVATE_KEY";
const provider = new ethers.JsonRpcProvider(RPC_URL);
const wallet = new ethers.Wallet(PRIVATE_KEY, provider);

// Token contract details
const TOKEN_ADDRESS = "0xeE92be1608e65e52FB2E5715542896027DCD9ca9";
const TOKEN_ABI = [
    "function mint(address to, uint256 amount) public",
    "function burn(uint256 amount) public",
];
const tokenContract = new ethers.Contract(TOKEN_ADDRESS, TOKEN_ABI, wallet);

// Configuration
const BATCH_SIZE = 10; // Number of concurrent transactions
const NONCE_BUFFER = 5; // Number of nonces to get ahead
const GAS_PRICE_BOOST = 1.5; // Multiplier for gas price to ensure quick mining

async function getNonceManager() {
    const currentNonce = await wallet.getNonce();
    let nextNonce = currentNonce;
    
    return {
        getNextNonce: () => nextNonce++
    };
}

async function mintAndBurnLoop() {
    const mintAmount = ethers.parseUnits("1000", 18);
    const burnAmount = ethers.parseUnits("100", 18);
    const nonceManager = await getNonceManager();

    // Get current gas price and boost it
    const gasPrice = await provider.getFeeData();
    const boostedGasPrice = {
        maxFeePerGas: gasPrice.maxFeePerGas * BigInt(Math.floor(GAS_PRICE_BOOST * 100)) / 100n,
        maxPriorityFeePerGas: gasPrice.maxPriorityFeePerGas * BigInt(Math.floor(GAS_PRICE_BOOST * 100)) / 100n
    };

    async function sendTransaction(isMintsending = true) {
        const nonce = nonceManager.getNextNonce();
        const tx = isMintsending
            ? await tokenContract.mint.populateTransaction(wallet.address, mintAmount)
            : await tokenContract.burn.populateTransaction(burnAmount);

        const signedTx = await wallet.sendTransaction({
            ...tx,
            nonce,
            ...boostedGasPrice,
        });

        return {
            hash: signedTx.hash,
            type: isMintsending ? 'mint' : 'burn',
            wait: () => signedTx.wait()
        };
    }

    async function processBatch() {
        while (true) {
            const transactions = [];
            
            // Send a batch of mint transactions
            for (let i = 0; i < BATCH_SIZE; i++) {
                transactions.push(sendTransaction(true));
            }
            
            // Send a batch of burn transactions
            for (let i = 0; i < BATCH_SIZE; i++) {
                transactions.push(sendTransaction(false));
            }

            // Send all transactions concurrently
            const pendingTxs = await Promise.all(transactions);
            
            // Log transaction hashes immediately
            pendingTxs.forEach(tx => {
                console.log(`${tx.type} transaction sent: ${tx.hash}`);
            });

            // Wait for confirmations in parallel
            try {
                await Promise.all(pendingTxs.map(tx => tx.wait()));
                console.log(`Batch of ${BATCH_SIZE * 2} transactions completed!`);
            } catch (error) {
                console.error("Error waiting for transactions:", error);
            }
        }
    }

    await processBatch();
}

// Start the loop
mintAndBurnLoop().catch(console.error);
```
