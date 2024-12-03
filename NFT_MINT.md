# NFT minting (quick+dirty as just testing)<br>

There are 3 file based components to minting an NFT:
- the ER721 contract itself
- the image file stored somewhere on the web (used Pinata maginly now but anywhere works, referencing WM logo direct from their website even worked!)
- a description json file, also has to be on the web somewhere (worked loading github, currently I'm using account on nft.storage currently min storage cost $3, but probably pinata free+ok too)

## Image file
Start with the image file - Pinata was free, I uploaded file there, keeping size small.  But you can use anywhere<br>
If you use https://pinata.cloud, when you upload your image it will give you a CID code - like QmRNWGgKesfBsdHkWJvn7FPS2FG9Era6pLAE5GRKW9qXi8<br>
This is referenced in a URL as such:  https://gateway.pinata.cloud/ipfs/QmRNWGgKesfBsdHkWJvn7FPS2FG9Era6pLAE5GRKW9qXi8<br>
Keep this URL direct to the image for storing in the description json file<br>

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
IIRC I have even used Piata to store this json descriptor file - as long as it can be referenced as a URL it is all good.
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
- next to do "Depoly and Run" on the left under compiler
- Make sure the environment on middle panel as select "Injected Provider - Metamask" or your wallet you are using
- Click on the Orange Depoly button and sign to deploy the NFT contract
- You should see the deployed contract appear down bottom of centre panel

### MINT
- Click on the deploye contract to expand it
   - Down a way you'll find the TokenURI field - enter the url to the json descriptor file above (which in turn contains the url to the image)
   - Go back up to the mint orange button and eEnter your wallet address (or probably any destination address) in the field next to it and click mintnft
   - Sign in wallet popup and hopefully that is it!

Hopefully it worked, then repeatedly click mint and sign for each NFT - doesn't take to long to get 10 or 20 done!
This way is really quick and dirty - cutting many corners so all same image..  Have fun and let me know if guide needs changes
