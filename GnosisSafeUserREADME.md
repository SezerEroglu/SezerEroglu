# Safe Contracts

## 0- Project Dependencies

Bu repoyu klonladıysanız veya package.json dosyasını ayrı indirdiyseniz:

```ps
yarn install

```
Manuel eklemek için tüm dependencyler

```ps
yarn add ethers
yarn add @gnosis.pm/safe-core-sdk

```
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
## 2 – Master Copy ve Proxy Factory Deploy Etmek 

Gnosis Safe-Contracts reposunda Hardhat içinde deploy scriptleri yer almaktadır. Desteklenen provider Infura’dır. Custom networklere deploy için Hardhat Config içinde custom network ve RPC_URL tanımlanabilir.

### Master Copy

GnosisSafe.sol ve GnosisSafeL2.sol kontratlarıdır. Bu kontratlar kodun büyük çoğunluğunu içerir fakat bu kontrata doğrudan Call gönderilmez. Onun yerine her bir kullanıcı grubu, Safe, için bir Proxy Contract aracılığıyla DelegateCall gönderilir.

### Proxy Factory

GnosisSafeProxyFactory.sol. Bu kontrat Proxy kontrat deploy eder.

## 3 – Bir Safe Objesi yaratmak

Safe objesi, ağda bulunan Proxy kontratın interface'idir. Bu interface Safe ile yapılacak imzalama, gönderme, owner ekleme ve Proxy kontratın bulunduğu adresi ve diğer özelliklerini okumak için gereklidir. 

```js
const Safe = require("@gnosis.pm/safe-core-sdk").default;
```

Safe class'ını require aldıktan sonra bir Safe objesi oluşturmak için senaryo vardır.

### 3.1 Ağa bir Proxy Kontrat deploy etmek

Ağa yeni bir Proxy deploy etmek için `SafeFactory` clasını dahil etmelisiniz.

```js
const { SafeFactory } = require("@gnosis.pm/safe-core-sdk");
```

ProxyFactory, MasterCopy ağda deploy edilmiş halde olmalıdır. Bu durumda adresleri aşağıdaki gibi bir objeye tanımlanmalıdır.

```js
const id = await ethAdapter.getChainId();

  const contractNetworks = {
    [id]: {
      safeMasterCopyAddress: "0x9c5ba02c7ccd1f11346e43785202711ce1dcc130",
      safeProxyFactoryAddress: "0x23ccc7463468e3c56a4ce37afab577eb3dd0e3cb",
    },
  };
```

