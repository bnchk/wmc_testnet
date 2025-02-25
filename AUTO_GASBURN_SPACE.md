# AUTO GAS BURN - FILL SPACE
To follow this process you will create:
* parametarised loading contract
* javascript to create other wallets/configure load and run
* javascript to fetch back any funds distributed to the other wallets
<br>

## SETUP CONTRACT
As per usual, easy to use remix.
<br>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

contract GasBurnViaSpace {
    // Simple counter to track usage
    uint256 public operationCount;
    
    // Storage mapping - uses blockchain space
    mapping(uint256 => uint256) public dataStorage;
    
    // Event for tracking operations
    event OperationComplete(
        uint256 indexed operationId,
        uint256 storageSize,
        uint256 iterations,
        uint256 timestamp
    );
    
    // Simple space filler with adjustable intensity
    // intensity: 1-5 scale where 5 is highest space usage
    function fillSpace(uint256 intensity) public {
        require(intensity >= 1 && intensity <= 5, "Intensity must be between 1-5");
        
        // Calculate parameters based on intensity
        uint256 storageEntries = intensity * 100;
        uint256 storageStart = operationCount * 1000;
        
        // Simple sequential storage - very reliable
        for (uint256 i = 0; i < storageEntries; i++) {
            dataStorage[storageStart + i] = block.number * 1000 + i;
        }
        
        // Update operation counter
        operationCount++;
        
        // Emit event to use some additional space in logs
        emit OperationComplete(
            operationCount,
            storageEntries,
            intensity,
            block.timestamp
        );
    }
    
    // More detailed control for advanced users
    function fillSpaceCustom(
        uint256 entryCount,
        uint256 startPosition
    ) public {
        require(entryCount <= 1000, "Entry count too high");
        
        uint256 start = startPosition == 0 ? operationCount * 1000 : startPosition;
        
        // Write data to storage
        for (uint256 i = 0; i < entryCount; i++) {
            dataStorage[start + i] = block.number * 1000 + i;
        }
        
        // Update operation counter
        operationCount++;
        
        // Emit event 
        emit OperationComplete(
            operationCount,
            entryCount,
            0,
            block.timestamp
        );
    }
    
    // Optional clear function to free up space
    function clearStorage(uint256 startId, uint256 count) public {
        require(count <= 1000, "Can only clear 1000 entries at once");
        
        for (uint256 i = 0; i < count; i++) {
            delete dataStorage[startId + i];
        }
    }
    
    // Get estimated gas for a given operation
    function estimateSpaceUsage(uint256 intensity) public pure returns (uint256) {
        uint256 storageEntries = intensity * 100;
        // ~21K base + ~5K per 100 storage entries
        return 21000 + (storageEntries * 50);
    }
}
```
<br>

## RUN LOADING JOB
This script will call the contract.  Replace the parameters to suit:
* private wallet key
* contract from the above
* how many other wallets (ACCOUNT_COUNT) to create (has 3 in here), will setup each with 0.05WMTx from calling wallet (0.05 is within code, can adust to suit)
* set stress load - medium seemed best at time I was running it
* run via:  `node scriptname.js`
* will create a json file called `generated-wallets.json` which is used by the next script tp retrieve any leftover WMTX and return to primary wallet

```javascript
// WARNING - THIS WILL CREATE SUBWALLETS AND SEND GAS TO THEM
//         - the private keys will be saved into a json, then 
//           recovery script has to be run to fetch any leftover funds back
// Script for running the SimpleSpaceFiller contract with improved nonce handling
const { ethers } = require("ethers");
const fs = require('fs');

// Contract configuration
const CONTRACT_ADDRESS = "INSERT_CONTRACT_ADDRESS_HERE"; // Your SpaceFiller contract
const ABI = [
    "function fillSpace(uint256 intensity) public",
    "function fillSpaceCustom(uint256 entryCount, uint256 startPosition) public",
    "function clearStorage(uint256 startId, uint256 count) public",
    "function estimateSpaceUsage(uint256 intensity) public pure returns (uint256)",
    "function operationCount() public view returns (uint256)"
];

// Network configuration
const RPC_URL = "https://rpc-testnet-base.worldmobile.net";
const PRIVATE_KEY = "INSERT_PRIVATE_KEY_HERE"; // Replace with your private key

// Stress test configuration
const CONCURRENT_TRANSACTIONS = 20;         // Lower concurrent transactions to start
const MAX_NONCE_AHEAD = 50;                 // Reduced max nonce ahead
const ACCOUNTS_COUNT = 3;                   // Number of accounts to use
const GAS_PRICE_BOOST = 1.1;                // Multiplier for gas price
const FIXED_GAS_LIMIT = 5000000;            // Reduced gas limit for the simpler contract

// Space filling configuration
const INTENSITY_LEVELS = {
    low: {
        intensity: 1,
        customEntryCount: 100,
        useCustom: false
    },
    medium: {
        intensity: 2,
        customEntryCount: 200,
        useCustom: false
    },
    high: {
        intensity: 3,
        customEntryCount: 300,
        useCustom: false
    },
    extreme: {
        intensity: 5,
        customEntryCount: 500,
        useCustom: false
    }
};

// Selected intensity for this run
const SELECTED_INTENSITY = "medium"; // Change to low, medium, high, or extreme as needed
const USE_CUSTOM_MODE = false;     // If true, uses custom entry count instead of preset intensity

// Storage management configuration
const STORAGE_CLEAR_INTERVAL = 50; // Clear storage every X transactions
const STORAGE_CLEAR_BATCH = 500;   // Number of storage slots to clear at once

async function createWallets(provider, count) {
    const wallets = [];
    const mainWallet = new ethers.Wallet(PRIVATE_KEY, provider);
    wallets.push(mainWallet);
    
    // Only create additional wallets if needed
    if (count > 1) {
        console.log("Funding additional wallets for parallel transactions...");
        // Generate additional wallets
        for (let i = 1; i < count; i++) {
            const newWallet = ethers.Wallet.createRandom().connect(provider);
            wallets.push(newWallet);
            
            // Fund the new wallet from the main wallet
            const fundingAmount = ethers.parseEther("0.05");
            const tx = await mainWallet.sendTransaction({
                to: newWallet.address,
                value: fundingAmount
            });
            await tx.wait();
            console.log(`Funded wallet ${i}: ${newWallet.address}`);
        }
    }
    
    // Save wallet information for recovery
    const walletInfo = wallets.map(w => ({
        address: w.address,
        privateKey: w.privateKey
    }));
    
    fs.writeFileSync(
        'generated-wallets.json',
        JSON.stringify(walletInfo, null, 2)
    );
    console.log('Saved wallet information to generated-wallets.json');
    
    return wallets;
}

async function estimateContractGasUsage(contract, intensity) {
    try {
        // Use the contract's built-in estimation function
        const params = INTENSITY_LEVELS[intensity];
        const gasEstimate = await contract.estimateSpaceUsage(params.intensity);
        
        return gasEstimate;
    } catch (err) {
        console.error("Failed to estimate gas usage:", err.message);
        return BigInt(1000000); // Lower default fallback estimation for simpler contract
    }
}

async function main() {
    const provider = new ethers.JsonRpcProvider(RPC_URL);
    try {
        const network = await provider.getNetwork();
        console.log(`Connected to network: ${network.name} (${network.chainId})`);
    } catch (error) {
        console.error("Failed to connect to network:", error.message);
        process.exit(1);
    }
    
    // Create wallets for distributing load
    const wallets = await createWallets(provider, ACCOUNTS_COUNT);
    console.log(`Using ${wallets.length} accounts for transaction distribution`);
    
    // Create contract instance
    const contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, wallets[0]);
    
    // Get gas estimate for selected intensity
    const estimatedGas = await estimateContractGasUsage(contract, SELECTED_INTENSITY);
    console.log(`Estimated gas usage per transaction: ${estimatedGas.toString()}`);
    
    // Transaction tracking
    let pendingTxCount = 0;
    let totalTxSent = 0;
    let successfulTx = 0;
    let failedTx = 0;
    const nonceTrackers = {};
    
    // Gas consumption tracking
    let totalGasUsed = BigInt(0);
    let txWithGasData = 0;
    
    // Storage clearing tracking
    let txSinceLastClear = 0;
    let lastStorageClearPosition = 0;
    
    // Initialize nonce trackers - use "latest" for stable nonce value
    for (const wallet of wallets) {
        const latestNonce = await provider.getTransactionCount(wallet.address, "latest");
        console.log(`Wallet ${wallet.address}: starting nonce = ${latestNonce}`);
        nonceTrackers[wallet.address] = latestNonce;
    }
    
    // Get current operation count
    let operationCount = await contract.operationCount();
    console.log(`Current operation count: ${operationCount}`);
    
    // Performance monitoring
    const startTime = Date.now();
    setInterval(() => {
        const elapsedSeconds = (Date.now() - startTime) / 1000;
        const txPerSecond = totalTxSent / elapsedSeconds;
        const avgGasPerTx = txWithGasData > 0 ? Number(totalGasUsed / BigInt(txWithGasData)) : 0;
        
        console.log(`\n------ STATS (${new Date().toISOString()}) ------`);
        console.log(`Intensity level: ${SELECTED_INTENSITY} (${USE_CUSTOM_MODE ? 'custom' : 'preset'} mode)`);
        console.log(`Transactions sent: ${totalTxSent} (${txPerSecond.toFixed(2)}/sec)`);
        console.log(`Success: ${successfulTx}, Failed: ${failedTx}, Pending: ${pendingTxCount}`);
        console.log(`Average gas used per tx: ${avgGasPerTx.toLocaleString()}`);
        
        // Estimated gas burned
        if (txWithGasData > 0) {
            const estimatedTotalGas = Number(totalGasUsed) * totalTxSent / txWithGasData;
            console.log(`Estimated total gas burned: ${estimatedTotalGas.toLocaleString()}`);
        }
        console.log(`Storage clearing: ${txSinceLastClear}/${STORAGE_CLEAR_INTERVAL} txs since last clear`);
        console.log(`------------------`);
    }, 10000);
    
    async function sendClearStorageTransaction(wallet) {
        const contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, wallet);
        
        try {
            // Get current gas price and boost it
            const gasPrice = await provider.getFeeData();
            const boostedGasPrice = {
                maxFeePerGas: gasPrice.maxFeePerGas * BigInt(Math.floor(GAS_PRICE_BOOST * 100)) / BigInt(100),
                maxPriorityFeePerGas: gasPrice.maxPriorityFeePerGas * BigInt(Math.floor(GAS_PRICE_BOOST * 100)) / BigInt(100)
            };
            
            // Get and verify nonce
            const latestNonce = await provider.getTransactionCount(wallet.address, "latest");
            const nonce = latestNonce;
            
            // Update nonce tracker for consistency
            nonceTrackers[wallet.address] = Math.max(nonceTrackers[wallet.address], nonce);
            
            // Calculate storage range to clear based on operation count
            const startId = lastStorageClearPosition;
            lastStorageClearPosition += STORAGE_CLEAR_BATCH;
            
            // Clear storage with range parameters
            const tx = await contract.clearStorage(
                startId,
                STORAGE_CLEAR_BATCH,
                {
                    nonce,
                    gasLimit: 3000000, // Lower gas limit for clearing
                    ...boostedGasPrice
                }
            );
            
            console.log(`Storage clearing transaction sent: ${tx.hash} (clearing range ${startId}-${startId + STORAGE_CLEAR_BATCH})`);
            await tx.wait();
            console.log("Storage successfully cleared");
            txSinceLastClear = 0;
            return true;
        } catch (err) {
            console.error("Failed to clear storage:", err.message);
            return false;
        }
    }
    
    async function sendTransaction(wallet, nonce) {
        const contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, wallet);
        
        try {
            // Double-check nonce
            const pendingNonce = await provider.getTransactionCount(wallet.address, "pending");
            const latestNonce = await provider.getTransactionCount(wallet.address, "latest");
            
            if (nonce < latestNonce) {
                console.log(`Nonce ${nonce} is too low for ${wallet.address}, updating to ${latestNonce}`);
                nonce = latestNonce;
                nonceTrackers[wallet.address] = latestNonce;
            }
            
            // Get current gas price and boost it
            const gasPrice = await provider.getFeeData();
            const boostedGasPrice = {
                maxFeePerGas: gasPrice.maxFeePerGas * BigInt(Math.floor(GAS_PRICE_BOOST * 100)) / BigInt(100),
                maxPriorityFeePerGas: gasPrice.maxPriorityFeePerGas * BigInt(Math.floor(GAS_PRICE_BOOST * 100)) / BigInt(100)
            };
            
            let tx;
            const params = INTENSITY_LEVELS[SELECTED_INTENSITY];
            
            if (USE_CUSTOM_MODE) {
                // Use custom entry count and calculate start position
                const startPosition = operationCount.toString() * 1000;
                tx = await contract.fillSpaceCustom.populateTransaction(
                    params.customEntryCount,
                    startPosition
                );
            } else {
                // Use simple intensity parameter
                tx = await contract.fillSpace.populateTransaction(
                    params.intensity
                );
            }
            
            // Skip gas estimation and use fixed gas limit
            const signedTx = await wallet.sendTransaction({
                ...tx,
                nonce,
                gasLimit: FIXED_GAS_LIMIT,
                ...boostedGasPrice
            });
            
            pendingTxCount++;
            totalTxSent++;
            txSinceLastClear++;
            
            // Handle transaction completion asynchronously
            signedTx.wait().then((receipt) => {
                pendingTxCount--;
                successfulTx++;
                
                // Increment operation count on successful transaction
                operationCount = operationCount + BigInt(1);
                
                // Track gas used
                if (receipt && receipt.gasUsed) {
                    totalGasUsed += receipt.gasUsed;
                    txWithGasData++;
                }
            }).catch(err => {
                pendingTxCount--;
                failedTx++;
                console.error(`Transaction failed: ${err.message.substring(0, 100)}...`);
            });
            
            return true;
        } catch (err) {
            console.error(`Error sending transaction: ${err.message.substring(0, 100)}...`);
            return false;
        }
    }
    
    // Main transaction sending loop
    console.log(`Starting space fill test with up to ${CONCURRENT_TRANSACTIONS} concurrent transactions`);
    console.log(`Using SimpleSpaceFiller contract at ${CONTRACT_ADDRESS}`);
    console.log(`Intensity: ${SELECTED_INTENSITY}, Mode: ${USE_CUSTOM_MODE ? 'Custom' : 'Preset'}`);
    
    // Initial storage clearing
    await sendClearStorageTransaction(wallets[0]);
    
    while (true) {
        // Check if we need to clear storage
        if (txSinceLastClear >= STORAGE_CLEAR_INTERVAL) {
            await sendClearStorageTransaction(wallets[0]);
        }
        
        // Only send more transactions if we're below our concurrent limit
        if (pendingTxCount < CONCURRENT_TRANSACTIONS) {
            // Distribute transactions among wallets
            for (const wallet of wallets) {
                let currentNonce = nonceTrackers[wallet.address];
                const actualNonce = await provider.getTransactionCount(wallet.address, "pending");
                
                // Validate nonce state before sending
                if (currentNonce < actualNonce) {
                    console.log(`Correcting nonce for ${wallet.address}: ${currentNonce} → ${actualNonce}`);
                    nonceTrackers[wallet.address] = actualNonce;
                    currentNonce = actualNonce;
                }
                
                // Only send more transactions if we're not too far ahead with nonces
                if (currentNonce < actualNonce + MAX_NONCE_AHEAD) {
                    const sent = await sendTransaction(wallet, currentNonce);
                    if (sent) {
                        nonceTrackers[wallet.address]++;
                        
                        // Longer pause between sends to avoid overwhelming the node
                        await new Promise(resolve => setTimeout(resolve, 200));
                    }
                }
            }
        } else {
            // If we've reached our concurrency limit, wait a bit before trying again
            await new Promise(resolve => setTimeout(resolve, 250));
        }
    }
}

main().catch(console.error);
```
<br>

## FETCH FUNDS BACK
* If you don't do this after each run, the config in `generated-wallets.json` will be lost and your transferred funds as well
* Insert private key into code, and run.

```javascript
// Script to recover funds from temporary wallets back to main wallet
// Run after the v3loadtest created subwallets for parallel running
// Script to recover funds from temporary wallets back to main wallet
const { ethers } = require("ethers");
const fs = require('fs');

// Configuration
const RPC_URL = "https://rpc-testnet-base.worldmobile.net";
const MAIN_WALLET_PRIVATE_KEY = "INSERT_PRIVATE_KET_HERE"; // Your main wallet private key
const GAS_BUFFER_WMTX = "0.00001"; // WMTX to leave for gas (adjust based on network fees)
const WALLET_FILE_PATH = './generated-wallets.json';

async function main() {
  console.log("Starting fund recovery process...");
  
  // Connect to the network
  const provider = new ethers.JsonRpcProvider(RPC_URL);
  const network = await provider.getNetwork();
  console.log(`Connected to network: ${network.name} (${network.chainId})`);
  
  // Set up main wallet
  const mainWallet = new ethers.Wallet(MAIN_WALLET_PRIVATE_KEY, provider);
  console.log(`Main wallet: ${mainWallet.address}`);
  
  // Get main wallet balance
  const mainBalanceBefore = await provider.getBalance(mainWallet.address);
  console.log(`Main wallet starting balance: ${ethers.formatEther(mainBalanceBefore)} WMTX`);
  
  // Load wallet data from file
  let walletData;
  try {
    const fileContent = fs.readFileSync(WALLET_FILE_PATH, 'utf8');
    walletData = JSON.parse(fileContent);
    console.log(`Loaded ${walletData.length} wallets from ${WALLET_FILE_PATH}`);
  } catch (error) {
    console.error(`Error loading wallet file: ${error.message}`);
    console.log('Make sure generated-wallets.json exists in the current directory');
    return;
  }
  
  // Filter out the main wallet
  const generatedWallets = walletData.filter(wallet => 
    wallet.address.toLowerCase() !== mainWallet.address.toLowerCase()
  );
  
  console.log(`Found ${generatedWallets.length} generated wallets to recover funds from`);
  
  // Connect each wallet
  const fundedWallets = generatedWallets.map(wallet => 
    new ethers.Wallet(wallet.privateKey, provider)
  );
  
  // Process each funded wallet
  let totalRecovered = BigInt(0);
  let recoveredWallets = 0;
  const gasBuffer = ethers.parseEther(GAS_BUFFER_WMTX);
  
  for (const wallet of fundedWallets) {
    try {
      // Get balance
      const balance = await provider.getBalance(wallet.address);
      
      if (balance <= gasBuffer) {
        console.log(`Skipping wallet ${wallet.address} - insufficient balance (${ethers.formatEther(balance)} WMTX)`);
        continue;
      }
      
      console.log(`Processing wallet ${wallet.address} with balance: ${ethers.formatEther(balance)} WMTX`);
      
      // Calculate amount to send (leave some for gas)
      const gasPrice = await provider.getFeeData();
      const gasLimit = BigInt(21000); // Standard WMTX transfer
      let gasCost;
      
      if (gasPrice.maxFeePerGas) {
        gasCost = gasLimit * gasPrice.maxFeePerGas;
      } else if (gasPrice.gasPrice) {
        gasCost = gasLimit * gasPrice.gasPrice;
      } else {
        // Fallback to a conservative gas price estimate
        gasCost = gasLimit * BigInt(10000000000); // 10 gwei
      }
      
      // All values must be BigInt for safe calculation
      const amountToSend = balance - gasCost - gasBuffer;
      
      if (amountToSend <= BigInt(0)) {
        console.log(`  Skipping - not enough to cover transfer costs`);
        continue;
      }
      
      console.log(`  Recovering ${ethers.formatEther(amountToSend)} WMTX`);
      
      // Send funds back to main wallet
      const tx = await wallet.sendTransaction({
        to: mainWallet.address,
        value: amountToSend,
        gasLimit: gasLimit
      });
      
      console.log(`  Transaction submitted: ${tx.hash}`);
      await tx.wait();
      console.log(`  Transaction confirmed`);
      
      totalRecovered += amountToSend;
      recoveredWallets++;
      
    } catch (error) {
      console.error(`Error processing wallet ${wallet.address}:`, error.message);
    }
  }
  
  // Show summary
  const mainBalanceAfter = await provider.getBalance(mainWallet.address);
  console.log("\n--- RECOVERY SUMMARY ---");
  console.log(`Wallets processed: ${fundedWallets.length}`);
  console.log(`Wallets with funds recovered: ${recoveredWallets}`);
  console.log(`Total WMTX recovered: ${ethers.formatEther(totalRecovered)} WMTX`);
  console.log(`Main wallet starting balance: ${ethers.formatEther(mainBalanceBefore)} WMTX`);
  console.log(`Main wallet ending balance: ${ethers.formatEther(mainBalanceAfter)} WMTX`);
  console.log(`Net change: ${ethers.formatEther(mainBalanceAfter - mainBalanceBefore)} WMTX`);
}

main().catch(console.error);
```

