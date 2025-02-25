# TOKEN MINTING - Lets get your Meme on!<br>

To proceed here, you need a wallet containing WMTx on the L3 WMC Base chain (our chain, the first L3 rollup above base!)<br>
<br>
This means you have already done the faucet/bridging/etc and are now on "W3 WMC base sepolia" with WMTx in your wallet<br>
<br>
If not first check out the guide from Nico [MNTX on W3-WMC](https://github.com/nodebasewm/nodebasewm.github.io/blob/main/wmc/testnet-guide.md)<br>
<br>
If OK - Now minting - There are many ways to mint tokens - and this is just one:<br>
- Start with https://remix.ethereum.org<br>
  - On the top left side click on file explorer - then find and click on create new file<br>
  - It will open a blank file and put cursor down on left panel to name the blank file - call it YOURNAME.sol<br>
     - YOURNAME = anything you want
     - the .sol bit = solidity (not solana)<br>
  - hit enter to accept file name <br>
  - Now you have to paste in your contract code to the blank pane on the right<br>
  - This does not "mint anything" but will be the contract for it<br>
- paste in this code as a template to start with (yes will warn):<br>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract TOKENCONTRACT is ERC20, Ownable {
    constructor(uint256 initialSupply) ERC20("TokenDescription", "TOKENCODE") Ownable(msg.sender)  {
        _mint(msg.sender, initialSupply); // Mint initial supply to the owner
    }
    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount); // Only the owner can mint new tokens
    }
}
```
- EDIT the following **3 keywords** in above script to customise it
   - TOKENCONTRACT => whatever you want but think it is unimportant/doesnt matter eg: MYTOKEN probably OK
   - TokenDescription = "Happy Pandas" (or whatever you want)
   - TOKENCODE = PANDA (this is your ticker)
<br><br>
## COMPILE, DEPLOY AND MINT
After editing the solidity contract file above to suit, and changing the 3 keywords there are 3 steps:
<br>
### COMPILE
- press control-s to compile the contract - should get a green tick on the compiler on the left

### DEPLOY + MINT
- next click "Deploy and Run" on the left under compiler
- Go to your Metamask wallet and make sure you have selected the W3 WMC chain
- Go to the environment selection on middle panel + select "Injected Provider - Metamask" or your wallet
- In the Account box, if correct Metamask account not selected choose it<br>
- Find the Orange Deploy button and the empty field to it's right
- In this field to the right you enter your total number of tokens to mint followed by 18 (yes!) zeros
  - so for 100k tokens enter: 100000000000000000000000
  - that is for decimal places or to drive peopl crazy anyrate, lets get past that
- Click deploy - your wallet should pop up for you to sign transaction
- Be patient, after a few seconds you should see the transactions confirmed flyover message go past
- Now right down the middle panel - there should be a "deployed contract"
   - click on the copy icon next to this deployed contract - that is your token identifier
   - you need this tokeon contract code - you have to import the token into your wallet
   - click on metamask, go to tokens, click import and paste this contract into the window - and WAIT
   - it takes a few seconds, but will load from chain the details for your token - then click ok
   - they are now visible in your wallet 

Have fun and let me know if guide worked, or any issues!

### REGISTER CONTRACT ON CHAIN (optional)
- rclick on the Remix File explorer/file name and select "flatten"
  - this will import all contracts referenced
  - make bigger file with _flattened.sol name
- Copy the first line of the original contract file (the licence line)
- replace that licence line over first (blank) line of flattened file + cntl-s
- Select all the code of flattened file - cntl-a/c
- find your deployed ERC20 contract under blockscout explorer 
- click on "contract", then on "verify and publish" to open verify page
  - select the licence you used (MIT for me), compiler type solidity single, compiler version used (probably the most recent in list - what you compliled using)
  - EVM version for me was cancun
  - Uncheck Optimisation
  - Paste in the flattened code from clipboard
  - Click Verify and Publish
