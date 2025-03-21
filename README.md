# WMC - Testnet
<br>
Following are some sample processes/scripts to get started on WMC, from minting NFTs through to basic automation.<br>
<br>

* [MNTX on W3-WMC](https://github.com/nodebasewm/nodebasewm.github.io/blob/main/wmc/testnet-guide.md) - Nicos guide - get WMTx onto WMC (L3 above base)
* [TOKEN MINT](./TOKEN_MINT.md) - Become meme 👑  ==>> Also video guide used by Dodo [here](https://discord.com/channels/739450842108919828/1072502970027081749/1313845801235124266)
* [NFT MINT](./NFT_MINT.md) - Mint your own NFT series
* [AUTO_TRANSACTIONS](./AUTO_TRANSACTIONS2.md) - V2 including auto gas calculations
* [AUTO_NFTS](./AUTO_NFTS.md) - V3 autogas, autononce, batches of 100 at a time
* [AUTO_MINT_BURN](./AUTO_MINT_BURN.md) - sequentially mint and burn tokens 
* [AUTO_ERC20_TO_RANDOM_ADDR](./AUTO_ERC20_TO_ALL_ADDR.md) - spam ERC20 to fabricated addresses
* [AUTO_ERC20_TO_ALL_ADDR](./AUTO_ERC20_TO_ALL_ADDR.md) - spam ERC20 to addresses with gas as at ~8th Feb 2025 
* [TOOL-GAS_CHECK](./TOOL-GAS_CHECK.md) - what stats can be found re gas rates?
* [TOOL-RPC_CHECK](./TOOL-RPC_CHECK.md) - check status of RPC vs Base
<br>

## Automation Environment Setup steps - quick setup
Quick environment setup command in one step.<br>
Intended for updated Ubuntu 22.04 desktop VM, with snapshot take in case of issues.<br>
To be run from home folder of standard user (not superuser):<br>

```bash
cd $HOME && \
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash && \
source ~/.bashrc && \
nvm install 20 && \
node -v && \
npm -v && \
mkdir $HOME/wmctest && cd $HOME/wmctest && \
npm init -y && \
npm pkg set type="module" && \
npm install ethers
```
<br>
