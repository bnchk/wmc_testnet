# CHECK RPC STATUS
Compare chain to base, plus check current endpoint status<br>
<br>
## SAMPLE OUTPUT
```
Checking L3 RPC status...
Testing RPC endpoint: https://rpc-testnet-base.worldmobile.net
✅ RPC is responding (724ms)
✅ Network ID: 42070
✅ Latest block: 3779911
✅ Node version: Geth/v1.101315.3-stable-8af19cf2/linux-amd64/go1.22.6
✅ Chain ID: 42070
✅ Latest block hash: 0x17ad99db0c27f3fdf0cea20c863aead393488d70cfee459ecf0aa07b111d29b1
✅ Block timestamp: 2025-03-21T02:09:55.000Z
✅ Block age: 330 seconds
⚠️ Warning: Latest block is 330 seconds old

--- Summary ---
RPC Status: ONLINE

Checking L1 connection (Base)...
✅ L1 (Base) is responding - Block height: 23377518
```
<br>

# SCRIPT

```javascript
// L3 WMC RPC Status Checker
// Hows the RPC doing, check base sepolia as well for sequencing

const axios = require('axios');

/**
 * Check the status of an RPC endpoint
 * @param {string} rpcUrl - The URL of the RPC endpoint to check
 * @param {number} timeout - Timeout in milliseconds (default: 5000)
 * @param {boolean} verbose - Whether to print verbose output (default: true)
 * @returns {Promise<object>} - Results of the RPC checks
 */
async function checkRpcStatus(rpcUrl, timeout = 5000, verbose = true) {
  if (!rpcUrl) {
    throw new Error('RPC URL is required');
  }

  const results = {
    url: rpcUrl,
    isResponding: false,
    responseTime: null,
    networkId: null,
    blockHeight: null,
    errors: [],
    timestamp: new Date().toISOString()
  };

  try {
    // Basic connection test with timeout
    const startTime = Date.now();
    
    if (verbose) console.log(`Testing RPC endpoint: ${rpcUrl}`);
    
    // Try a simple JSON-RPC call to get network version
    const networkResponse = await axios.post(
      rpcUrl,
      {
        jsonrpc: '2.0',
        method: 'net_version',
        params: [],
        id: 1
      },
      { timeout: timeout }
    );
    
    const responseTime = Date.now() - startTime;
    results.responseTime = responseTime;
    
    if (networkResponse.data && networkResponse.data.result) {
      results.isResponding = true;
      results.networkId = networkResponse.data.result;
      if (verbose) console.log(`✅ RPC is responding (${responseTime}ms)`);
      if (verbose) console.log(`✅ Network ID: ${results.networkId}`);
    } else {
      results.errors.push('No valid response to net_version call');
      if (verbose) console.log('❌ RPC returned invalid response format');
    }

    // Get latest block information
    try {
      const blockResponse = await axios.post(
        rpcUrl,
        {
          jsonrpc: '2.0',
          method: 'eth_blockNumber',
          params: [],
          id: 2
        },
        { timeout: timeout }
      );
      
      if (blockResponse.data && blockResponse.data.result) {
        const blockHex = blockResponse.data.result;
        results.blockHeight = parseInt(blockHex, 16);
        if (verbose) console.log(`✅ Latest block: ${results.blockHeight}`);
      } else {
        results.errors.push('Failed to get latest block number');
        if (verbose) console.log('❌ Could not retrieve latest block');
      }
    } catch (blockError) {
      results.errors.push(`Block error: ${blockError.message}`);
      if (verbose) console.log(`❌ Block query failed: ${blockError.message}`);
    }

    // Try getting node info (may not be supported by all RPCs)
    try {
      const nodeInfoResponse = await axios.post(
        rpcUrl,
        {
          jsonrpc: '2.0', 
          method: 'web3_clientVersion',
          params: [],
          id: 3
        },
        { timeout: timeout }
      );
      
      if (nodeInfoResponse.data && nodeInfoResponse.data.result) {
        results.nodeVersion = nodeInfoResponse.data.result;
        if (verbose) console.log(`✅ Node version: ${results.nodeVersion}`);
      }
    } catch (nodeError) {
      // This is optional, so not considering it a primary error
      if (verbose) console.log(`ℹ️ Node info not available: ${nodeError.message}`);
    }
    
    // Check chain ID (important for L2/L3 configurations)
    try {
      const chainIdResponse = await axios.post(
        rpcUrl,
        {
          jsonrpc: '2.0',
          method: 'eth_chainId',
          params: [],
          id: 4
        },
        { timeout: timeout }
      );
      
      if (chainIdResponse.data && chainIdResponse.data.result) {
        const chainIdHex = chainIdResponse.data.result;
        results.chainId = parseInt(chainIdHex, 16);
        if (verbose) console.log(`✅ Chain ID: ${results.chainId}`);
      } else {
        results.errors.push('Failed to get chain ID');
        if (verbose) console.log('❌ Could not retrieve chain ID');
      }
    } catch (chainIdError) {
      results.errors.push(`Chain ID error: ${chainIdError.message}`);
      if (verbose) console.log(`❌ Chain ID query failed: ${chainIdError.message}`);
    }
    
    // Check L2/L3 specific endpoints if this is a rollup
    try {
      // Try to get the L1 block info (works on OP Stack and some other L2/L3s)
      const l1BlockResponse = await axios.post(
        rpcUrl,
        {
          jsonrpc: '2.0',
          method: 'eth_getBlockByNumber',
          params: ['latest', false],
          id: 5
        },
        { timeout: timeout }
      );
      
      if (l1BlockResponse.data && l1BlockResponse.data.result) {
        // Check if block has L1 specific fields (like l1BlockNumber for OP Stack)
        const block = l1BlockResponse.data.result;
        
        if (block.l1BlockNumber) {
          results.l1BlockNumber = parseInt(block.l1BlockNumber, 16);
          if (verbose) console.log(`✅ L1 Block Number: ${results.l1BlockNumber}`);
        }
        
        // Store the latest block hash for reference
        results.latestBlockHash = block.hash;
        results.latestBlockTimestamp = parseInt(block.timestamp, 16);
        
        // Calculate approximate time since last block
        const blockTime = new Date(results.latestBlockTimestamp * 1000);
        const now = new Date();
        const blockAgeSeconds = Math.floor((now - blockTime) / 1000);
        results.blockAgeSeconds = blockAgeSeconds;
        
        if (verbose) {
          console.log(`✅ Latest block hash: ${results.latestBlockHash}`);
          console.log(`✅ Block timestamp: ${new Date(results.latestBlockTimestamp * 1000).toISOString()}`);
          console.log(`✅ Block age: ${blockAgeSeconds} seconds`);
          
          // Warn if block is old (could indicate sync issues)
          if (blockAgeSeconds > 60) {
            console.log(`⚠️ Warning: Latest block is ${blockAgeSeconds} seconds old`);
          }
        }
      }
    } catch (l1Error) {
      // Not considering L1 block info a critical error
      if (verbose) console.log(`ℹ️ L1 block info not available: ${l1Error.message}`);
    }
    
  } catch (error) {
    results.isResponding = false;
    results.errors.push(`Connection error: ${error.message}`);
    
    if (error.code === 'ECONNREFUSED') {
      if (verbose) console.log('❌ Connection refused - RPC server may be down');
    } else if (error.code === 'ETIMEDOUT' || error.message.includes('timeout')) {
      if (verbose) console.log(`❌ Connection timed out after ${timeout}ms`);
    } else {
      if (verbose) console.log(`❌ RPC error: ${error.message}`);
    }
  }

  // Summary
  if (verbose) {
    console.log('\n--- Summary ---');
    console.log(`RPC Status: ${results.isResponding ? 'ONLINE' : 'OFFLINE'}`);
    if (results.errors.length > 0) {
      console.log('Errors:');
      results.errors.forEach(err => console.log(`  - ${err}`));
    }
  }

  return results;
}

// Configuration parameters - EDIT THESE VALUES
const CONFIG = {
  // WMC RPC endpoint
  rpcUrl: 'https://rpc-testnet-base.worldmobile.net',
  
  // Base RPC endpoint (used for comparison)
  l1RpcUrl: 'https://sepolia.base.org',
  
  // Request timeout in milliseconds
  timeout: 5000,
  
  // Whether to show detailed logs
  verbose: true
};

// Example usage
async function main() {
  // Use configuration from the top of the file
  const rpcUrl = CONFIG.rpcUrl;
  const l1RpcUrl = CONFIG.l1RpcUrl;
  
  try {
    console.log('Checking L3 RPC status...');
    const results = await checkRpcStatus(rpcUrl, CONFIG.timeout, CONFIG.verbose);
    
    // Try checking sync status with L1
    if (results.isResponding && l1RpcUrl) {
      try {
        console.log('\nChecking L1 connection (Base)...');
        const l1Results = await checkRpcStatus(l1RpcUrl, CONFIG.timeout, false);
        
        if (l1Results.isResponding) {
          console.log(`✅ L1 (Base) is responding - Block height: ${l1Results.blockHeight}`);
          
          // Check if L3 is significantly behind L1
          if (results.l1BlockNumber && l1Results.blockHeight) {
            const blockDiff = l1Results.blockHeight - results.l1BlockNumber;
            console.log(`ℹ️ L3 is referencing L1 block that is ${blockDiff} blocks behind current L1 block`);
            
            if (blockDiff > 100) {
              console.log(`⚠️ Warning: L3 appears to be significantly behind L1 (${blockDiff} blocks)`);
            }
          }
        } else {
          console.log('❌ Could not connect to L1 (Base) for comparison');
        }
      } catch (l1Error) {
        console.log(`❌ Error checking L1: ${l1Error.message}`);
      }
    }
    
    // Check if we should try an alternative URL
    if (!results.isResponding && rpcUrl.includes('localhost')) {
      console.log('\nTrying alternative testnet RPC...');
      const altRpcUrl = rpcUrl.replace('localhost', '127.0.0.1');
      await checkRpcStatus(altRpcUrl);
    }
    
    // If still not responding, suggest more debugging steps
    if (!results.isResponding) {
      console.log('\n--- Troubleshooting suggestions ---');
      console.log('1. Check if the L3 sequencer/node is running');
      console.log('2. Verify network configuration (ports, firewall, etc.)');
      console.log('3. Check if the rollup node is connected to Base');
      console.log('4. Check L3 node logs');
      console.log('5. Turn it off and on again');
    }
  } catch (error) {
    console.error('Error running RPC check:', error.message);
  }
}

// Run if executed directly
if (require.main === module) {
  main();
}

module.exports = { checkRpcStatus };
```
