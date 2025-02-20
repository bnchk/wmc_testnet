# AUTO GAS BURN - MIXED
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

contract GasBurnerMixed {
    uint256 public counter;
    mapping(uint256 => uint256) public dataStore;
    mapping(uint256 => bytes32) public hashStore;
    mapping(bytes32 => uint256[]) public arrayStore;
    
    struct ComplexData {
        uint256 value;
        bytes32 hash;
        uint256[] values;
        mapping(uint256 => uint256) nestedMap;
    }
    
    mapping(uint256 => ComplexData) public complexDataStore;
    
    // Default values that can be changed by the owner
    uint256 public defaultHeavyIterations = 50;
    uint256 public defaultRecursiveDepth = 3;
    uint256 public defaultStringIterations = 20;
    uint256 public defaultArraySize = 10;
    
    address public owner;
    
    event Debug(uint256 indexed iteration, uint256 value, bytes32 hash);
    event GasBurned(string functionName, uint256 gasUsed, uint256 timestamp);
    
    constructor() {
        owner = msg.sender;
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this function");
        _;
    }
    
    // Configuration functions
    function setDefaultParameters(
        uint256 _heavyIterations,
        uint256 _recursiveDepth,
        uint256 _stringIterations,
        uint256 _arraySize
    ) public onlyOwner {
        defaultHeavyIterations = _heavyIterations;
        defaultRecursiveDepth = _recursiveDepth;
        defaultStringIterations = _stringIterations;
        defaultArraySize = _arraySize;
    }
    
    // Burns gas through heavy computation, storage writes, and log generation
    function burnGasHeavy() public {
        burnGasHeavyWithParams(defaultHeavyIterations, defaultArraySize);
    }
    
    function burnGasHeavyWithParams(uint256 iterations, uint256 arraySize) public {
        require(iterations <= 500, "Iterations limited to 500 max");
        require(arraySize <= 50, "Array size limited to 50 max");
        
        uint256 startGas = gasleft();
        bytes32 runningHash = keccak256(abi.encodePacked(block.timestamp, msg.sender));
        
        // Generate a complex, unique storage key
        for (uint256 i = 0; i < iterations; i++) {
            // Expensive hash computation
            runningHash = keccak256(abi.encodePacked(runningHash, i, block.number));
            
            // Create complex data for storage
            uint256[] memory values = new uint256[](arraySize);
            for (uint256 j = 0; j < arraySize; j++) {
                values[j] = uint256(keccak256(abi.encodePacked(j, runningHash))) % 1000000;
            }
            
            // Multiple storage operations (SSTORE is expensive)
            dataStore[i] = uint256(runningHash);
            hashStore[i] = runningHash;
            arrayStore[runningHash] = values;
            
            // Complex data storage operations
            ComplexData storage complexData = complexDataStore[i];
            complexData.value = uint256(runningHash);
            complexData.hash = runningHash;
            
            for (uint256 k = 0; k < arraySize; k++) {
                complexData.nestedMap[k] = uint256(keccak256(abi.encodePacked(k, runningHash)));
            }
            
            // Emit log data (LOG operations consume gas)
            if (i % 10 == 0) {
                emit Debug(i, uint256(runningHash), runningHash);
            }
        }
        
        counter++;
        emit GasBurned("burnGasHeavy", startGas - gasleft(), block.timestamp);
    }
    
    // Burns gas through recursive calls and heavy math
    function burnGasRecursive() public {
        burnGasRecursiveWithDepth(defaultRecursiveDepth);
    }
    
    function burnGasRecursiveWithDepth(uint256 depth) public {
        require(depth <= 8, "Maximum depth exceeded"); // Prevent stack too deep errors
        
        uint256 startGas = gasleft();
        _recursiveFunction(depth);
        emit GasBurned("burnGasRecursive", startGas - gasleft(), block.timestamp);
    }
    
    function _recursiveFunction(uint256 depth) internal {
        if (depth == 0) {
            return;
        }
        
        bytes32 computedHash = keccak256(abi.encodePacked(depth, block.timestamp));
        
        // Heavy math operations
        uint256 result = 1;
        for (uint256 i = 1; i <= 50; i++) {
            result = addmod(
                mulmod(result, i, type(uint256).max),
                uint256(computedHash),
                type(uint256).max
            );
            
            // Storage writes between calculations
            dataStore[depth * 1000 + i] = result;
        }
        
        // Recursive call (uses stack space)
        _recursiveFunction(depth - 1);
    }
    
    // Burns gas through string operations and storage
    function burnGasStrings() public {
        burnGasStringsWithParams(defaultStringIterations);
    }
    
    function burnGasStringsWithParams(uint256 iterations) public {
        require(iterations <= 100, "Iterations limited to 100 max");
        
        uint256 startGas = gasleft();
        string memory base = "This is a test string for gas consumption through string manipulation and storage operations.";
        
        for (uint256 i = 0; i < iterations; i++) {
            // String concatenation is expensive in Solidity
            string memory newString = string(abi.encodePacked(base, " Iteration: ", uint2str(i)));
            
            // Store string hash
            bytes32 stringHash = keccak256(abi.encodePacked(newString));
            hashStore[i + 5000] = stringHash;
            
            // Create and store array based on string hash
            uint256[] memory values = new uint256[](defaultArraySize);
            for (uint256 j = 0; j < defaultArraySize; j++) {
                values[j] = uint256(keccak256(abi.encodePacked(j, stringHash)));
            }
            arrayStore[stringHash] = values;
        }
        
        emit GasBurned("burnGasStrings", startGas - gasleft(), block.timestamp);
    }
    
    // Helper function to convert uint to string
    function uint2str(uint256 _i) internal pure returns (string memory) {
        if (_i == 0) {
            return "0";
        }
        
        uint256 j = _i;
        uint256 length;
        while (j != 0) {
            length++;
            j /= 10;
        }
        
        bytes memory bstr = new bytes(length);
        uint256 k = length;
        j = _i;
        while (j != 0) {
            k = k-1;
            uint8 temp = (48 + uint8(j - j / 10 * 10));
            bytes1 b1 = bytes1(temp);
            bstr[k] = b1;
            j /= 10;
        }
        
        return string(bstr);
    }
    
    // Comprehensive gas burner with customizable parameters
    function burnGasComprehensive() public {
        burnGasComprehensiveWithParams(
            defaultHeavyIterations / 2,  // Use half iterations for comprehensive to avoid exceeding block gas limit
            defaultRecursiveDepth - 1,   // Use reduced depth
            defaultStringIterations / 2  // Use half string iterations
        );
    }
    
    function burnGasComprehensiveWithParams(
        uint256 heavyIterations,
        uint256 recursiveDepth,
        uint256 stringIterations
    ) public {
        uint256 startGas = gasleft();
        
        // First perform heavy computation and storage
        burnGasHeavyWithParams(heavyIterations, defaultArraySize);
        
        // Then do recursive operations with configurable depth
        burnGasRecursiveWithDepth(recursiveDepth);
        
        // Finally do string operations with configurable iterations
        burnGasStringsWithParams(stringIterations);
        
        emit GasBurned("burnGasComprehensive", startGas - gasleft(), block.timestamp);
    }
    
    // Function that allows dynamic control of gas consumption
    // intensity: 1-10, where 1 is minimal gas usage and 10 is maximum
    function burnGasAdjustable(uint256 intensity) public {
        require(intensity >= 1 && intensity <= 10, "Intensity must be between 1 and 10");
        
        // Scale parameters based on intensity
        uint256 heavyIterations = intensity * 5;       // 5-50 iterations
        uint256 recursiveDepth = (intensity + 1) / 3;  // 0-3 depth
        uint256 stringIterations = intensity * 2;      // 2-20 iterations
        uint256 arraySize = intensity;                 // 1-10 array size
        
        uint256 startGas = gasleft();
        
        burnGasHeavyWithParams(heavyIterations, arraySize);
        
        if (intensity > 3) {
            burnGasRecursiveWithDepth(recursiveDepth);
        }
        
        if (intensity > 6) {
            burnGasStringsWithParams(stringIterations);
        }
        
        emit GasBurned("burnGasAdjustable", startGas - gasleft(), block.timestamp);
    }
}
```
<br>

## JAVASCRIPT TO RUN JOB
This script will call the contract.  Replace the parameters to suit:
* private wallet key
* contract from the above
* how many other wallets (ACCOUNT_COUNT) to create (has 3 in here), will setup each with 0.05WMTx from calling wallet (0.05 is within code, can adust to suit)
* set SELECTED_INTENSITY - high failed every now and then, but that is a sign you are reaching the limit
* run via:  `node scriptname.js`
* will create a json file called `generated-wallets.json` which is used by the next script tp retrieve any leftover WMTX and return to primary wallet

```javascript
// WARNING - THIS WILL CREATE SUBWALLETS AND SEND GAS TO THEM
//         - the private keys will be saved into a json, then 
//           recovery script has to be run to fetch any leftover funds back
// Added parameterization with V4 of gas burning contract
const { ethers } = require("ethers");
const fs = require('fs');

// Contract configuration
const CONTRACT_ADDRESS = "INSERT_CONTRACT_ADDRESS_HERE"; // Replace with your deployed contract address
const ABI = [
    "function burnGasHeavy() public",
    "function burnGasHeavyWithParams(uint256 iterations, uint256 arraySize) public",
    "function burnGasRecursive() public",
    "function burnGasRecursiveWithDepth(uint256 depth) public",
    "function burnGasStrings() public",
    "function burnGasStringsWithParams(uint256 iterations) public",
    "function burnGasComprehensive() public",
    "function burnGasComprehensiveWithParams(uint256 heavyIterations, uint256 recursiveDepth, uint256 stringIterations) public",
    "function burnGasAdjustable(uint256 intensity) public",
    "function setDefaultParameters(uint256 _heavyIterations, uint256 _recursiveDepth, uint256 _stringIterations, uint256 _arraySize) public"
];

// Network configuration
const RPC_URL = "https://rpc-testnet-base.worldmobile.net";
const PRIVATE_KEY = "INSERT_PRIVATE_KEY_HERE"; // Replace with your private key

// Stress test configuration
const CONCURRENT_TRANSACTIONS = 20;         // Lower concurrent transactions to start
const MAX_NONCE_AHEAD = 50;                 // Reduced max nonce ahead
const ACCOUNTS_COUNT = 3;                   // Number of accounts to use
const GAS_PRICE_BOOST = 1.1;                // Multiplier for gas price
const FIXED_GAS_LIMIT = 10000000;           // Fixed gas limit to avoid estimation

// Gas burning configuration
const INTENSITY_LEVELS = {
    low: {
        heavyIterations: 10,
        recursiveDepth: 2,
        stringIterations: 5,
        arraySize: 5
    },
    medium: {
        heavyIterations: 20,
        recursiveDepth: 3,
        stringIterations: 10,
        arraySize: 8  
    },
    high: {
        heavyIterations: 40,
        recursiveDepth: 4,
        stringIterations: 15,
        arraySize: 10
    }
};

// Selected intensity for this run
const SELECTED_INTENSITY = "high"; // Change to medium or high as needed

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

async function configureContract(contract, wallet) {
    // Set default parameters based on selected intensity
    const intensity = INTENSITY_LEVELS[SELECTED_INTENSITY];
    console.log(`Configuring contract with ${SELECTED_INTENSITY} intensity:`, intensity);
    
    try {
        // Get gas price
        const feeData = await wallet.provider.getFeeData();
        const boostedGasPrice = {
            maxFeePerGas: feeData.maxFeePerGas * BigInt(Math.floor(GAS_PRICE_BOOST * 100)) / BigInt(100),
            maxPriorityFeePerGas: feeData.maxPriorityFeePerGas * BigInt(Math.floor(GAS_PRICE_BOOST * 100)) / BigInt(100)
        };
        
        const tx = await contract.setDefaultParameters(
            intensity.heavyIterations,
            intensity.recursiveDepth, 
            intensity.stringIterations,
            intensity.arraySize,
            {
                gasLimit: 500000, // Lower gas limit for configuration
                ...boostedGasPrice
            }
        );
        
        console.log(`Configuration transaction sent: ${tx.hash}`);
        await tx.wait();
        console.log("Contract successfully configured with new parameters");
        return true;
    } catch (err) {
        console.error("Failed to configure contract:", err.message);
        return false;
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
    
    // Configure contract with selected intensity parameters
    await configureContract(contract, wallets[0]);
    
    // Transaction tracking
    let pendingTxCount = 0;
    let totalTxSent = 0;
    let successfulTx = 0;
    let failedTx = 0;
    const nonceTrackers = {};
    
    // Gas consumption tracking
    let totalGasUsed = BigInt(0);
    let txWithGasData = 0;
    
    // Initialize nonce trackers
    for (const wallet of wallets) {
        nonceTrackers[wallet.address] = await provider.getTransactionCount(wallet.address);
    }
    
    // Available gas burning functions to use
    // Using parameterized versions to have more control
    const gasBurningFunctions = [
        // Simpler functions for better reliability
        "burnGasHeavy", 
        "burnGasRecursive",
        "burnGasAdjustable" // Using the adjustable function with intensity parameter
    ];
    
    // Performance monitoring
    const startTime = Date.now();
    setInterval(() => {
        const elapsedSeconds = (Date.now() - startTime) / 1000;
        const txPerSecond = totalTxSent / elapsedSeconds;
        const avgGasPerTx = txWithGasData > 0 ? Number(totalGasUsed / BigInt(txWithGasData)) : 0;
        
        console.log(`\n------ STATS (${new Date().toISOString()}) ------`);
        console.log(`Transactions sent: ${totalTxSent} (${txPerSecond.toFixed(2)}/sec)`);
        console.log(`Success: ${successfulTx}, Failed: ${failedTx}, Pending: ${pendingTxCount}`);
        console.log(`Average gas used per tx: ${avgGasPerTx.toLocaleString()}`);
        
        // Estimated gas burned
        if (txWithGasData > 0) {
            const estimatedTotalGas = Number(totalGasUsed) * totalTxSent / txWithGasData;
            console.log(`Estimated total gas burned: ${estimatedTotalGas.toLocaleString()}`);
        }
        console.log(`------------------`);
    }, 10000);
    
    async function sendTransaction(wallet, nonce, functionIndex) {
        const contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, wallet);
        
        try {
            // Get current gas price and boost it
            const gasPrice = await provider.getFeeData();
            const boostedGasPrice = {
                maxFeePerGas: gasPrice.maxFeePerGas * BigInt(Math.floor(GAS_PRICE_BOOST * 100)) / BigInt(100),
                maxPriorityFeePerGas: gasPrice.maxPriorityFeePerGas * BigInt(Math.floor(GAS_PRICE_BOOST * 100)) / BigInt(100)
            };
            
            // Select which function to call
            const functionName = gasBurningFunctions[functionIndex % gasBurningFunctions.length];
            
            let tx;
            // Special handling for different functions
            if (functionName === "burnGasAdjustable") {
                // Use intensity between 1-5 for safer operation
                const intensity = (nonce % 5) + 1;
                tx = await contract.burnGasAdjustable.populateTransaction(intensity);
            } else if (functionName === "burnGasRecursive") {
                // Use safer depth of 2-3 instead of 5
                const depth = Math.min(3, INTENSITY_LEVELS[SELECTED_INTENSITY].recursiveDepth);
                tx = await contract.burnGasRecursiveWithDepth.populateTransaction(depth);
            } else if (functionName === "burnGasHeavy") {
                // Use the regular function which uses default parameters
                tx = await contract.burnGasHeavy.populateTransaction();
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
            
            // Handle transaction completion asynchronously
            signedTx.wait().then((receipt) => {
                pendingTxCount--;
                successfulTx++;
                
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
    console.log(`Starting stress test with up to ${CONCURRENT_TRANSACTIONS} concurrent transactions`);
    console.log(`Using contract at ${CONTRACT_ADDRESS}`);
    let functionRotationCounter = 0;
    
    while (true) {
        // Only send more transactions if we're below our concurrent limit
        if (pendingTxCount < CONCURRENT_TRANSACTIONS) {
            // Distribute transactions among wallets
            for (const wallet of wallets) {
                const currentNonce = nonceTrackers[wallet.address];
                const actualNonce = await provider.getTransactionCount(wallet.address, "pending");
                
                // Only send more transactions if we're not too far ahead with nonces
                if (currentNonce < actualNonce + MAX_NONCE_AHEAD) {
                    const sent = await sendTransaction(wallet, currentNonce, functionRotationCounter);
                    if (sent) {
                        nonceTrackers[wallet.address]++;
                        functionRotationCounter++;
                        
                        // Small pause between sends to avoid overwhelming the node
                        await new Promise(resolve => setTimeout(resolve, 50));
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

## JAVASCRIPT TO FETCH FUNDS BACK
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
const MAIN_WALLET_PRIVATE_KEY = "INSERT_PRIVATE_KEY_HERE"; // Your main wallet private key
const GAS_BUFFER_ETH = "0.00001"; // WMTX to leave for gas (adjust based on network fees)
const WALLET_FILE_PATH = './generated-wallets.json';

async function main() {
  console.log("Starting funds recovery process...");
  
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
  const gasBuffer = ethers.parseEther(GAS_BUFFER_ETH);
  
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
  console.log(`Total ETH recovered: ${ethers.formatEther(totalRecovered)} WMTX`);
  console.log(`Main wallet starting balance: ${ethers.formatEther(mainBalanceBefore)} WMTX`);
  console.log(`Main wallet ending balance: ${ethers.formatEther(mainBalanceAfter)} WMTX`);
  console.log(`Net change: ${ethers.formatEther(mainBalanceAfter - mainBalanceBefore)} WMTX`);
}

main().catch(console.error);
```
