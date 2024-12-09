# AUTOMATED NFT MINTING
This is script I have used to mint towards 90k NFTs on testnet at this point - Spam Haus + Dolphin Pepe<br>

It requires node.js, ethers and also nft.storage as per very loose instructions in [AUTO_TRANSACTIONS](./AUTO_TRANSACTIONS.md) + below<br>

Needs account at nft.storage (1G storage for think was $3 - was cheap).  Then get an API key from them for in code below.<br>

## Verify environment
- make sure you can get responses for/have installed the following (recommend VM so can revert if something doesn't work):<br>
```bash
node -v  #and get a version number back
npm -v   #and get a version number back

#then I dropped back a version of node.js
nvm install 20
nvm use 20  #using older version for better compatibility apparenty

npm install ethers       # has the commands we need to use
npm install nft.storage  # has the api calls for online storage of metadata files at this provider (nft.storage)

#then in the folder in installed in edit packages.json and add this line in the top half
"type": "module"
```

## STEPS BEFOREHAND
- manually mint your NFT as per [NFT_MINT](./NFT_MINT.md) so you can get contract number for in the script
- in this example I swapped to NFT.STORAGE for the metadata to be stored - got an API key + insert that in the following script code

## CREATE MINTING SCRIPT 
- have already minted your NFT manually via Remix as per [NFT_MINT](./NFT_MINT.md)
- check/change in the script code:
   - change the private key for your wallet (hide secure details in .env file if it matters for you (not applicable for me))
   - Your contract needs to be changed
   - Change the NFT_NAME and NFT_DESC to match yours
   - Change the IMAGE_URL to be your image
   - Verify your ABI matches code below (should do if following previous step above to mint it)
   - Insert your NFT.STORAGE api key into NFT_STORAGE_API_KEY variable

```javascript
//======================================================================
// MASS MFT MINTER - Spam Haus
//======================================================================

// .env secrecy not used for this as disposable accounts but would be like this:
//import * as dotenv from "dotenv";   
//dotenv.config();
//const RPC_URL = process.env.RPC_URL;
//const PRIVATE_KEY = process.env.PRIVATE_KEY;
//const CONTRACT_ADDRESS = process.env.CONTRACT_ADDRESS;

import { ethers } from "ethers";

// VARIABLES - ACCOUNT
const PRIVATE_KEY = "<<INSERT MINTING WALLET PRIVATE KEY>>";
const RECIPIENT_ADDR = "<<RECIPIENT PUBLIC ADDR - I USED SAME AS MINTING BUT PROBABLY BETTER TO USE ANOTHER>>";


// VARIABLES - NFT+MINT
const NUM_NFTS = 100000;       // How many in total for looping
const NFT_NAME = "Yum";        // Match to your NFT
const NFT_DESC = "Spam Haus";  //Match to your NFT
const CONTRACT_ADDRESS = "0xef2DC2D817a036734fd2973644f1cf3DF32642f6";    //Use your NFT contract address, this is my spam one
const IMAGE_URL = "https://gateway.pinata.cloud/ipfs/QmRNWGgKesfBsdHkWJvn7FPS2FG9Era6pLAE5GRKW9qXi8";   // this is my spam image - replace with yours

const RPC_URL = "https://rpc-testnet-base.worldmobile.net";  // L3 WMC RPC


// BASIC CHECKS
if (!RPC_URL || !PRIVATE_KEY || !CONTRACT_ADDRESS) {
  console.error("Missing environment variables. Check your .env file/config.");
  process.exit(1);
}


// DEFINE RPC + TEST
const provider = new ethers.JsonRpcProvider(RPC_URL);
console.log("Provider connected");

// TEST RPC CALL - can it find the tip
(async () => {
  try {
    const blockNumber = await provider.getBlockNumber();
    console.log(`Connected to blockchain. Current block number: ${blockNumber}`);
  } catch (error) {
    console.error("Failed to connect to the RPC provider:", error.message);
	process.exit(1); 
  }
})();


// TEST NFT CONTRACT EXISTS
async function checkContractAddress(address) {
  try {
    const code = await provider.getCode(address);

    if (code === "0x") {
      console.log(`FAIL: Contract address ${address} is not a deployed contract.`);
	  process.exit(1); 
    } else {
      console.log(`The address ${address} is a valid contract.`);
    }
  } catch (error) {
    console.error(`Error checking the contract address:`, error);
	process.exit(1); 
  }
}

// Call the function with the contract address
checkContractAddress(CONTRACT_ADDRESS);



//======================================================================
// DEFINE THE NFT - ABI should be ok if contract declared in same way as earlier guide
//======================================================================


const ABI = [
	{
		"inputs": [
			{"internalType": "address","name": "recipient","type": "address"},
			{"internalType": "string","name": "tokenURI","type": "string"}
		],
		"name": "mintNFT",
		"outputs": [
			{"internalType": "uint256","name": "","type": "uint256"}
		],
		"stateMutability": "nonpayable","type": "function"
	}
];

const wallet = new ethers.Wallet(PRIVATE_KEY, provider);
const contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, wallet);



// Function - dynamically generate JSON metadata
function generateMetadata(tokenId, nftName, nftDesc) {
  return {
    name: `${nftName} #${tokenId}`,
    description: `${nftDesc} NFT #${tokenId}`,
    image: IMAGE_URL,
    attributes: [
      {
        trait_type: "Toking ID",
        value: tokenId
      }
    ]
  };
}


// Function - upload metadata to NFT.STORAGE (optional)
import { NFTStorage, File } from "nft.storage";
const NFT_STORAGE_API_KEY = "<<INSERT_NFT_STORAGE_API_KEY_HERE>>"


async function uploadMetadataToNFTStorage(metadata) {
  const client = new NFTStorage({ token: NFT_STORAGE_API_KEY });

  // Store the metadata object directly
  const metadataCID = await client.store({
    name: metadata.name,
    description: metadata.description,
    image: IMAGE_URL, // Ensure this is a File or URL object
    attributes: metadata.attributes,
  });

  console.log("Metadata successfully uploaded:", metadataCID.url);
  return metadataCID.url; // Return the URL pointing to IPFS
}


// Function - Mint the NFTs
async function batchMint(recipient) {
  for (let i = 1; i <= NUM_NFTS; i++) {
    const metadata = generateMetadata(i, NFT_NAME, NFT_DESC);
    console.log(`Generated metadata for token #${i}:`, metadata);

    // Optionally, upload metadata to NFT.STORAGE
    const metadataURI = `data:application/json;base64,${Buffer.from(
      JSON.stringify(metadata)
    ).toString("base64")}`;

    console.log(`Minting token #${i} to ${recipient} with URI: ${metadataURI}`);
    const tx = await contract.mintNFT(recipient, metadataURI);
    console.log(`Transaction sent: ${tx.hash}`);
    await tx.wait(); // Wait for confirmation
    console.log(`Token #${i} minted successfully.`);
  }
}

const recipientAddress = RECIPIENT_ADDR; // Replace with recipient address
batchMint(recipientAddress).catch(console.error);
```

## SAVE + RUN THE SCRIPT
- save the above file with name like spamnft.js
- run it as follows:
   - ```node spamnft.js```
- Hopefully you now have automated ERC-721 NFT minting!<br>
- Definitely there are ways to do this in bulk much more neatly, but leaving that for another day - for now this focus is overloading testnet.<br>
- Any corrections/feedback are most welcome, as I am learning as I go!
