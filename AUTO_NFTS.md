# AUTOMATED NFT MINTING
This is gen3 script - bulk mints in batches with auto nonce and gas - much more evolved than earlier Spam Haus + Dolphin Pepe examples<br>

## STEPS BEFOREHAND
- manually mint your NFT as per [NFT_MINT](./NFT_MINT.md) so you can get contract number for in the script
- have IPFS image URL for the image to add into script below (free with pinata takes minutes)
- setup automation environment as per [ENV_SETUP](./README.md#automation-environment-setup-steps)

## CREATE MINTING SCRIPT 
- update the following in the script code:
   - change PRIVATE_KEY for your wallet (hide secure details in .env file if it matters for you (not applicable for me))
   - change RECIPIENT_ADDR recipient address
   - change CONTRACT_ADDRESS for your contract
   - Change the BASE_METADATA to match your NFT:
      - nft name
      - nft description
      - image url

```javascript
import { ethers } from "ethers";

// Configuration
const PRIVATE_KEY = "<<< INSERT YOUR PRIVATE KEY HERE >>>>>>";
const RECIPIENT_ADDR = "<<< recipient addresss 0x >>>>>>";
const CONTRACT_ADDRESS = "0xAd8bd71EEc1539508DcA829129D97e0112708e99";
const RPC_URL = "https://rpc-testnet-base.worldmobile.net";
const NUM_NFTS = 10000;
const BATCH_SIZE = 100; // Number of concurrent transactions
const GAS_PRICE_BOOST = 1.5; // Multiplier for gas price

// Basic metadata that will be used for all NFTs - UPDATE YOURS HERE
const BASE_METADATA = {
    name: "Yum",
    description: "Spam Haus",
    image: "https://gateway.pinata.cloud/ipfs/QmRNWGgKesfBsdHkWJvn7FPS2FG9Era6pLAE5GRKW9qXi8",
};

// Setup provider and contract
const provider = new ethers.JsonRpcProvider(RPC_URL);
const wallet = new ethers.Wallet(PRIVATE_KEY, provider);
const ABI = [
    {
        "inputs": [
            {"internalType": "address","name": "recipient","type": "address"},
            {"internalType": "string","name": "tokenURI","type": "string"}
        ],
        "name": "mintNFT",
        "outputs": [{"internalType": "uint256","name": "","type": "uint256"}],
        "stateMutability": "nonpayable","type": "function"
    }
];
const contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, wallet);

// Nonce manager to handle concurrent transactions
async function getNonceManager() {
    const currentNonce = await wallet.getNonce();
    let nextNonce = currentNonce;
    return {
        getNextNonce: () => nextNonce++
    };
}

// Generate metadata URI directly without external storage
function generateMetadataURI(tokenId) {
    const metadata = {
        ...BASE_METADATA,
        name: `${BASE_METADATA.name} #${tokenId}`,
        description: `${BASE_METADATA.description} NFT #${tokenId}`,
        attributes: [{ trait_type: "Token ID", value: tokenId }]
    };
    return `data:application/json;base64,${Buffer.from(JSON.stringify(metadata)).toString("base64")}`;
}

async function speedMint() {
    console.log("Starting high-speed minting process...");
    const nonceManager = await getNonceManager();

    // Get and boost gas price
    const gasPrice = await provider.getFeeData();
    const boostedGasPrice = {
        maxFeePerGas: gasPrice.maxFeePerGas * BigInt(Math.floor(GAS_PRICE_BOOST * 100)) / 100n,
        maxPriorityFeePerGas: gasPrice.maxPriorityFeePerGas * BigInt(Math.floor(GAS_PRICE_BOOST * 100)) / 100n
    };

    // Function to create and send a single mint transaction
    async function sendMintTransaction(tokenId) {
        const nonce = nonceManager.getNextNonce();
        const metadataURI = generateMetadataURI(tokenId);
        
        const tx = await contract.mintNFT.populateTransaction(RECIPIENT_ADDR, metadataURI);
        
        const signedTx = await wallet.sendTransaction({
            ...tx,
            nonce,
            ...boostedGasPrice,
        });

        return {
            tokenId,
            hash: signedTx.hash,
            wait: () => signedTx.wait()
        };
    }

    // Process in batches
    let processedCount = 0;
    while (processedCount < NUM_NFTS) {
        const batchSize = Math.min(BATCH_SIZE, NUM_NFTS - processedCount);
        const batchPromises = [];

        // Create batch of transactions
        for (let i = 0; i < batchSize; i++) {
            const tokenId = processedCount + i + 1;
            batchPromises.push(sendMintTransaction(tokenId));
        }

        // Send batch concurrently
        try {
            const pendingTxs = await Promise.all(batchPromises);
            
            // Log transaction hashes immediately
            pendingTxs.forEach(tx => {
                console.log(`Minting #${tx.tokenId} - Transaction sent: ${tx.hash}`);
            });

            // Wait for confirmations in parallel
            await Promise.all(pendingTxs.map(tx => tx.wait()));
            processedCount += batchSize;
            console.log(`Batch complete! Total minted: ${processedCount}/${NUM_NFTS}`);
        } catch (error) {
            console.error("Error in batch:", error);
            // Continue with next batch even if current batch had errors
        }
    }
}

// Initial RPC connection test
async function testConnection() {
    try {
        const blockNumber = await provider.getBlockNumber();
        console.log(`Connected to blockchain. Current block number: ${blockNumber}`);
        
        const code = await provider.getCode(CONTRACT_ADDRESS);
        if (code === "0x") {
            throw new Error("Contract not deployed");
        }
        console.log("Contract verified. Starting minting process...");
        
        await speedMint();
    } catch (error) {
        console.error("Setup error:", error);
        process.exit(1);
    }
}

testConnection();
```

## SAVE + RUN THE SCRIPT
- save the above file with name like spamnft.js
- run it as follows:
   - ```node spamnft.js```
- Hopefully you now have automated batching of ERC-721 NFT minting!<br>
