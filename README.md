# WMC - Testnets
If we can break it together, now is the time!<br>
<br>
Any load on the chain helps - lets go team :muscle:

This is how I went about it - and there are likely simpler ways.<br>
But heading down this path is a way to get code which can be used to automate stuff<br>
20241214 - added updated links to better versions of code for automation below<br>

* [MNTX on W3-WMC](https://github.com/nodebasewm/nodebasewm.github.io/blob/main/wmc/testnet-guide.md) - Nicos guide - get WMTx onto WMC (L3 above base)
* [TOKEN MINT](./TOKEN_MINT.md) - Become meme 👑  ==>> Also another way from Dodo [here](https://discord.com/channels/739450842108919828/1072502970027081749/1313845801235124266)
* [NFT MINT](./NFT_MINT.md) - Mint your own NFT series
* [GAS_LIMIT](./MAX_GAS_LIMIT.md) - what is max gas limit for a block?
* [AUTO_TRANSACTIONS](./AUTO_TRANSACTIONS2.md) - V2 including auto gas calculations
* [AUTO_NFTS](./AUTO_NFTS.md) - V3 autogas, autononce, batches of 100 at a time
* [AUTO_MINT_BURN](./AUTO_MINT_BURN.md) - sequentially mint and burn tokens 
<br>

## Automation Environment Setup steps
This is tested on Ubuntu 22.04 desktop, but should work on server.<br>
Would recommend running in VM and taking snapshots just in case you wanted to revert/found issues:<br>
* From home directory of user who will run scripts, install node version manager (recommended standard access user not superuser):
  ```bash
  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
  ```
* Reload environment variables - either close/open terminal or:
  * ```bash
    source ~/.bashrc
    ```
* Install the node version required - in this case v20 sounded good:
  * ```bash
    nvm install 20
    ```
* Should return:  20.18.3
  * ```bash
    node -v
    ```
* Should return:   10.8.2
  * ```bash
    npm -v
    ```
* Setup Project folder
  * Create folder you are storing scripts in and change there eg:
    * ```bash
      mkdir $HOME/wmctest && cd $HOME/wmctest
      ```
  * Initialise the package manager and set to module type:
    * ```bash
      npm init -y
      ```
  * Set module type:
    * ```bash
      npm pkg set type="module"
      ```
  * Install ethers:
    * ```bash
      npm install ethers
      ```
<br>

