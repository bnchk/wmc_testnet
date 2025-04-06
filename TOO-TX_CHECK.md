# CHECK TX STATUS
Have a tx hash - but is it on-chain?<br>
Usage:  `node checktx.cjs 0xhashityityhash304985034989038690348590348590349058349058034`
<br>
## SAMPLE OUTPUT
```
$ node checktx.cjs 0x9735bffda5e2c06dadce64337f2fa66149e4027deab55686bc4ae031845a5ff2
Checking transaction: 0x9735bffda5e2c06dadce64337f2fa66149e4027deab55686bc4ae031845a5ff2
Transaction found!
Block number: 5156851
Transaction status: Success
Gas used: 21000
$
$ node checktx.cjs 0xb3f369db57e230bb85715b3f4694f0707c9e850ea93def21d3a59362bc0c91e8
Checking transaction: 0xb3f369db57e230bb85715b3f4694f0707c9e850ea93def21d3a59362bc0c91e8
Transaction is pending or not found on chain

```
<br>

# SCRIPT

```javascript
// Check if tx hash is onchain
const { ethers } = require('ethers');

async function checkTransaction(txHash) {
  // Connect to an RPC endpoint for your EVM chain
  const provider = new ethers.JsonRpcProvider('https://rpc-testnet-base.worldmobile.net'); 
  
  try {
    // Get transaction receipt
    const receipt = await provider.getTransactionReceipt(txHash);
    
    if (receipt === null) {
      console.log('Transaction is pending or not found on chain');
      return null;
    }
    
    console.log('Transaction found!');
    console.log('Block number:', receipt.blockNumber);
    console.log('Transaction status:', receipt.status === 1 ? 'Success' : 'Failed');
    console.log('Gas used:', receipt.gasUsed.toString());
    
    return receipt;
  } catch (error) {
    console.error('Error querying transaction:', error);
    return null;
  }
}

// Get the transaction hash from command line arguments
const txHash = process.argv[2];

if (!txHash) {
  console.error('Please provide a transaction hash as an argument');
  console.error('Usage: node checktx.cjs 0xYOUR_TRANSACTION_HASH');
  process.exit(1);
}

console.log(`Checking transaction: ${txHash}`);
checkTransaction(txHash);
```
