# Safe Contracts
## 1 – EthAdapter
Bir Ethers.js objesini wrapleyen Gnosis objesi. Safe üzerinden tx göndermek veya chaindeki Safelerin, Proxylerin özelliklerini okumak için gerekli. Bir signer objesine ihtiyaç vardır, Web3 kütüphanesinde signer objesi var deniyor ama Ethers.js üzerinden bir wallet ta signer görevini görüyor.

```js
const { ethers } = require("ethers");
const provider = new ethers.providers.InfuraProvider(
  "rinkeby",
  "91ce96ee878d4ee6a72592e802293f58"
);


const wallet1 = new ethers.Wallet(process.env.ACCOUNT_1_PK /*Private Key*/, provider);

const EthersAdapter = require("@gnosis.pm/safe-ethers-lib").default;
const ethAdapter = new EthersAdapter({ ethers, signer: wallet1 });
```




