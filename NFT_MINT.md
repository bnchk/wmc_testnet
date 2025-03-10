# NFT minting (quick+dirty as just testing)<br>

There are 3 file based components to minting an NFT (2 of which have to be on the web):
- the ER721 contract itself (not on the web)
- the image file stored somewhere on the web (used Pinata website mostly anywhere works, even referencing WM logo direct from their website worked!)
- a description json file (which contains the URL to the image file), also has to be on the web somewhere (worked loading onto github, currently I'm using account on nft.storage costing $3, but probably pinata website free+works too)

## Image file
Start with the image file - Pinata was free, I uploaded file there, keeping size small.  But you can use anywhere that is online/URL to reference it<br>
If you use https://pinata.cloud, when you upload your image it will give you a CID code - like QmRNWGgKesfBsdHkWJvn7FPS2FG9Era6pLAE5GRKW9qXi8<br>
This is referenced in a URL as such:  https://gateway.pinata.cloud/ipfs/QmRNWGgKesfBsdHkWJvn7FPS2FG9Era6pLAE5GRKW9qXi8<br>
Construct + keep this URL direct to the image for storing in the description json file<br>

## JSON description file
This is a file that contains the name and description of the NFT, plus the location of the image. <br>
Here are 2 example of my early ones showing you do not have to use Pinata - just as long as image is on web.<br>
```json
{
    "name": "WMCNFT",
    "description": "Unconnected Connected!",
    "image": "https://worldmobiletoken.com/meta/wmt-favicon-270x270.png"
}
```
```json
{
    "name": "WMC NFT",
    "description": "Unconnected Connected!",
    "image": "https://gateway.pinata.cloud/ipfs/Qmb7dZUgqwgMZPnYPiKh33axvmquPBqBMzxRZM2n1aTTJ1"
}
```
IIRC I have even used Pinata to store this json descriptor file - as long as it can be referenced as a URL it is all good.
Pinata actually does IPFS pinning of the files (like real NFTs) but anything works.
Here is one of mine:  https://gateway.pinata.cloud/ipfs/QmXSJVAHjoUpVM33VVhXnDkJgq1av1qDSeNg5zZR9jSzXS<br>

## Contract File
This is loaded into Remix at https://remix.ethereum.org as a solidity file - ends in *.sol<br>
Change the names/Description to suit, I always made them match the ones in the descriptor json file<br>
<br>
The 3 bits to change here are:
- Contract name - NFTCONTRACT - eg SPAMNFT
- DescriptionHere - eg "SPAM Haus"
- NFTCODE - eg SPAMNFT
<br>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract NFTCONTRACT is ERC721URIStorage, Ownable {
    uint256 public tokenCounter;

    constructor() ERC721("DescriptionHere", "NFTCODE")  Ownable(msg.sender)  {
        tokenCounter = 0;
    }

    function mintNFT(address recipient, string memory tokenURI) public onlyOwner returns (uint256) {
        uint256 newTokenId = tokenCounter;
        _mint(recipient, newTokenId);
        _setTokenURI(newTokenId, tokenURI);
        tokenCounter++;
        return newTokenId;
    }
}
```

<br><br>
## COMPILE, DEPLOY AND MINT
After editing the solidity contract file above to suit, there are a few steps to rolling it out manually
<br>
### COMPILE
- press control-s to compile the NFT - should get a green tick on the compiler on the left

### DEPLOY
- next to do "Deploy and Run" on the left under compiler
- Make sure the environment on middle panel as select "Injected Provider - Metamask" or your wallet you are using
- Click on the Orange Deploy button and sign to deploy the NFT contract
- You should see the deployed contract appear down bottom of centre panel

### MINT
- Click on the newly deployed contract down the bottom of the centre panel to expand it
   - There are many fields but most do not matter
   - Down most of the way you'll find the TokenURI field - enter the url to the json descriptor file from above (which in turn contains the url to the image)
   - Go back up to the orange mint button and enter your wallet address in field next to it (or probably any destination address!) - and click mintnft
   - Sign in wallet popup and hopefully that is it!
   - Don't expect it to "appear" in your wallet like CNA - you have to import it and the NFT # and it may not show image
   - The import option in metamask is under tokens (not NFTs!), but it will recognise the NFT contract number and direct you to add the NFT#

Hopefully it worked, if so then repeatedly click mint and sign for each NFT - doesn't take to long to get 10 or 20 done!
Just have to import them all one by one then.  But you are doing ERC721 - whoooo!<br>
This way is really quick and dirty - cutting many corners so all same image..<br>
Have fun and let me know if guide worked, or if changes needed or course!
