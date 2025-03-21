# CHECK GAS INFO RETURNED BY RPC
May not match what is happening, but is something

```javascript
// WMC Testnet Gas Query Script (ethers v6)
// This script queries all available gas-related information from an WMC RPC endpoint
// Install dependencies: npm install ethers@6 axios

const { ethers } = require('ethers');
const axios = require('axios');

// Configuration
const RPC_URL = "https://rpc-testnet-base.worldmobile.net";
const EXPLORER_API_URL = "https://explorer2-base-testnet.worldmobile.net/api";
const SENDER_ADDRESS = "0x0000000000000000000000000000000000000000"; // Public addr with funds (not charged, just to mimic trans to fetch cost)

// Initialize provider - updated for ethers v6
const provider = new ethers.JsonRpcProvider(RPC_URL);

async function queryGasInfo() {
  console.log('==== WMC Testnet Gas Information ====\n');
  
  try {
    // 1. Basic network information
    const network = await provider.getNetwork();
    console.log(`Network: ${network.name} (chainId: ${network.chainId})`);
    
    // 2. Current block information
    const blockNumber = await provider.getBlockNumber();
    console.log(`Current Block: ${blockNumber}`);
    
    const latestBlock = await provider.getBlock(blockNumber);
    console.log(`Block Gas Limit: ${latestBlock.gasLimit.toString()}`);
    console.log(`Block Gas Used: ${latestBlock.gasUsed.toString()}`);
    // Calculate gas utilization percentage - updated for ethers v6 (using regular math ops)
    const gasUtilizationPercentage = (Number(latestBlock.gasUsed) * 100 / Number(latestBlock.gasLimit)).toFixed(2);
    console.log(`Gas Utilization: ${gasUtilizationPercentage}%`);
    
    if (latestBlock.baseFeePerGas) {
      console.log(`Base Fee Per Gas: ${ethers.formatUnits(latestBlock.baseFeePerGas, 'gwei')} gwei`);
    } else {
      console.log('Base Fee Per Gas: Not available (pre-EIP-1559 network)');
    }
    
    // 3. Standard gas price information
    const gasPrice = await provider.getFeeData().then(data => data.gasPrice);
    console.log(`Current Gas Price: ${ethers.formatUnits(gasPrice, 'gwei')} gwei`);
    
    // 4. Fee history (EIP-1559)
    try {
      // Using raw RPC call to get fee history (last 10 blocks)
      const feeHistory = await provider.send('eth_feeHistory', [10, 'latest', [25, 50, 75]]);
      
      console.log('\n==== Fee History (Last 10 blocks) ====');
      
      if (feeHistory.baseFeePerGas) {
        console.log('Base Fee Per Gas (gwei):');
        feeHistory.baseFeePerGas.forEach((fee, index) => {
          console.log(`  Block -${feeHistory.baseFeePerGas.length - 1 - index}: ${ethers.formatUnits(fee, 'gwei')} gwei`);
        });
      }
      
      if (feeHistory.reward) {
        console.log('\nPriority Fee Percentiles (gwei):');
        feeHistory.reward.forEach((rewards, index) => {
          console.log(`  Block -${feeHistory.reward.length - 1 - index}:`);
          rewards.forEach((reward, percentileIndex) => {
            const percentile = [25, 50, 75][percentileIndex];
            console.log(`    ${percentile}th percentile: ${ethers.formatUnits(reward, 'gwei')} gwei`);
          });
        });
      }
    } catch (error) {
      console.log('Fee history not supported on this network or provider');
    }
    
    // 5. EIP-1559 specific fee suggestions
    try {
      // Get fee data (maxFeePerGas and maxPriorityFeePerGas)
      const feeData = await provider.getFeeData();
      
      console.log('\n==== EIP-1559 Fee Data ====');
      
      if (feeData.maxFeePerGas) {
        console.log(`Max Fee Per Gas: ${ethers.formatUnits(feeData.maxFeePerGas, 'gwei')} gwei`);
      } else {
        console.log('Max Fee Per Gas: Not available');
      }
      
      if (feeData.maxPriorityFeePerGas) {
        console.log(`Max Priority Fee Per Gas: ${ethers.formatUnits(feeData.maxPriorityFeePerGas, 'gwei')} gwei`);
      } else {
        console.log('Max Priority Fee Per Gas: Not available');
      }
    } catch (error) {
      console.log('EIP-1559 fee data not supported on this network or provider');
    }
    
    // 6. Manual fee calculations for different speeds (based on percentiles if available)
    console.log('\n==== Gas Price Suggestions ====');
    
    // Standard gas price suggestions - updated for ethers v6 (using regular math ops)
    const gasPriceNumber = Number(gasPrice);
    console.log(`Slow: ${ethers.formatUnits(BigInt(Math.floor(gasPriceNumber * 0.8)), 'gwei')} gwei`);
    console.log(`Average: ${ethers.formatUnits(gasPrice, 'gwei')} gwei`);
    console.log(`Fast: ${ethers.formatUnits(BigInt(Math.floor(gasPriceNumber * 1.2)), 'gwei')} gwei`);
    console.log(`Urgent: ${ethers.formatUnits(BigInt(Math.floor(gasPriceNumber * 1.5)), 'gwei')} gwei`);
    
    // 7. Gas estimations for common transactions
    console.log('\n==== Gas Estimations ====');
    
    // Estimate gas for a simple WMTX transfer
    const gasEstimateForTransfer = await provider.estimateGas({
      from: SENDER_ADDRESS,
      to: ethers.ZeroAddress,
      value: ethers.parseEther('0.001')
    });
    console.log(`Estimated gas for WMTX transfer: ${gasEstimateForTransfer.toString()}`);
    
    // Calculate costs
    const transferCostWei = gasEstimateForTransfer * gasPrice;
    console.log(`Estimated cost for WMTX transfer: ${ethers.formatEther(transferCostWei)} WMTX`);
    
    // 8. Check if there are pending transactions
    try {
      const pendingTransactions = await provider.send('txpool_content', []);
      const pendingCount = Object.keys(pendingTransactions.pending).reduce(
        (count, address) => count + Object.keys(pendingTransactions.pending[address]).length, 
        0
      );
      console.log(`\nPending Transactions: ${pendingCount}`);
    } catch (error) {
      console.log('\nCould not get pending transactions (txpool_content not supported)');
      
      // Alternative method: try to get the pending block
      try {
        const pendingBlock = await provider.getBlock('pending');
        console.log(`Pending Block Transaction Count: ${pendingBlock.transactions.length}`);
      } catch (pendingBlockError) {
        console.log('Could not get pending block information');
      }
    }
    
    // 9. Gas price volatility (comparing with historical data)
    try {
      console.log('\n==== Gas Price Volatility ====');
      
      // Get blocks from 1 hour ago (assuming ~15 sec block time)
      const blocksPerHour = 3600 / 15;
      const oneHourAgoBlock = blockNumber - Math.floor(blocksPerHour);
      
      const historicalBlock = await provider.getBlock(oneHourAgoBlock);
      
      if (historicalBlock.baseFeePerGas && latestBlock.baseFeePerGas) {
        const currentBaseFee = Number(latestBlock.baseFeePerGas);
        const historicalBaseFee = Number(historicalBlock.baseFeePerGas);
        
        // Calculate change percentage
        const changePercentage = ((currentBaseFee - historicalBaseFee) / historicalBaseFee * 100).toFixed(2);
        
        console.log(`Base Fee 1 hour ago: ${ethers.formatUnits(historicalBlock.baseFeePerGas, 'gwei')} gwei`);
        console.log(`Current Base Fee: ${ethers.formatUnits(latestBlock.baseFeePerGas, 'gwei')} gwei`);
        console.log(`Change: ${changePercentage}%`);
      } else {
        // For non-EIP-1559 networks
        const historicalGasPrice = await provider.send('eth_gasPrice', [oneHourAgoBlock]);
        const historicalGasPriceNumber = Number(historicalGasPrice);
        const gasPriceNumber = Number(gasPrice);
        
        const changePercentage = ((gasPriceNumber - historicalGasPriceNumber) / historicalGasPriceNumber * 100).toFixed(2);
        
        console.log(`Gas Price 1 hour ago: ${ethers.formatUnits(historicalGasPrice, 'gwei')} gwei`);
        console.log(`Current Gas Price: ${ethers.formatUnits(gasPrice, 'gwei')} gwei`);
        console.log(`Change: ${changePercentage}%`);
      }
    } catch (error) {
      console.log('Could not calculate gas price volatility:', error.message);
    }
    
  } catch (error) {
    console.error('Error querying gas information:', error);
  }
}

// Helper function to get gas prices from other sources (if available)
async function getExternalGasPrices() {
  if (!EXPLORER_API_URL) return null;
  
  try {
    const response = await axios.get(`${EXPLORER_API_URL}/api?module=gastracker&action=gasoracle`);
    return response.data.result;
  } catch (error) {
    console.error('Could not fetch external gas prices:', error);
    return null;
  }
}

// Run the main function
queryGasInfo()
  .then(() => process.exit(0))
  .catch(error => {
    console.error('Fatal error:', error);
    process.exit(1);
  });
```

