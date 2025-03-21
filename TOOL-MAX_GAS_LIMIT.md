# CHECK MAX GAS LIMIT PER BLOCK

```javascript
import { ethers } from "ethers";

// Configurations
const RPC_URL = "https://rpc-testnet-base.worldmobile.net"; // Replace with your RPC URL

// Set up provider
const provider = new ethers.JsonRpcProvider(RPC_URL);

(async () => {
    try {
        const block = await provider.getBlock("latest");
        console.log(`Block gas limit: ${block.gasLimit.toString()}`);
    } catch (err) {
        console.error("Error fetching block information:", err);
    }
})();
```

Returns as at 20241210:<br>
`Block gas limit: 30000000`
