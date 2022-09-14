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
const provider = new ethers.providers(//PROVIDER INFURA RPC etc.);


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

Safe class'ını require aldıktan sonra bir Safe objesi oluşturmak için iki senaryo vardır.

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
id: Network Chain ID. (Örneğin Rinkeby için 4)

contractNetworks: Her bir Chain ID için o network üzerinde bulunan kontratların adresleri. 

```js
const safeFactory = await SafeFactory.create({
    ethAdapter: ethAdapter,
    contractNetworks: contractNetworks,
  });
```

SafeFactory class'ına sahip bir obje yukarıdaki gibi yaratılır. 

```js
const owners = [address1, address2, address3];
const threshold = 3;
const safeAccountConfig = { owners: owners, threshold: threshold };

const safe = await safeFactory.deploySafe({ safeAccountConfig });
```

Yukarıdaki parametreler ile safeFactory objesine `deploySafe` metodu çağırılır. Bu metod chain üzerinde çalıştığı için await kullanılmalıdır.
owners: Safe yetklili hesapların adresleri.
threshold: Bir MultiSig transcation gönderilebilmesi için gerekli olan minimun onay sayısı.

Bu iki parametre `safeAccountConfig` objesinde görüldüğü gibi tanımlanır ve `deploySafe` metodu için input verilir. Metod Retrun promise olarak bir `Safe` object verir.

### 3.2 Ağda bulunan Proxy Kontratı kullanmak

```js
const safe = await Safe.create({
    ethAdapter: ethAdapter,
    safeAddress: safeAddress,
    contractNetworks: contractNetworks,
  });
```
Bu durumda `Safe`class'ında `create` metodu kullanılır ve parametre olarak fazladan input objesine `safeAddress` eklenir. Bu ağdaki Proxy kontratın adresidir.

## 4 - Transaction Objesi oluşturmak

Onaylanmak ve gönderilmek üzere bir transaction objesi şu şekilde oluşturulur:

```js
const transaction = {
    to: "0x43A8f63c4616376d7f8775E603e8a17CE4D19957",
    value: 10000000000000,
    data: "0x",
    operation: 0,
    safeTxGas: 0,
    baseGas: 0,
    gasPrice: 0,
    gasToken: "0x0000000000000000000000000000000000000000",
    refundReceiver: "0x0000000000000000000000000000000000000000",
    nonce: currentNonce,
  };
```

Nonce: Nonce değeri her Proxy kontrat için ayrıdır ve her başarılı transaction sonrası artar. Bu nonce:

```js
const currentNonce = await safe.getNonce();
```

metoduyla elde edilebilir.

Sonraki adım bu objeyi kullanarak bir safeTransaction objesi oluşturmaktır. Bunun için

```js
const safeTransaction = await safe.createTransaction(transaction);
```

safe objesinde `createTransaction` metodu çağırılır. Bu Promise olarak `EthSafeTransaction` clasında bir obje döndürür.

`EthSafeTransaction` objesi içinde hem transaction bilgilerini hem de bu transaction'ı imzalamış kişilerin imzaları ve adreslerini içerir.

## 5 – Transaction imzalamak (Signing)

Gnosis ile bir transcation onaylamanın birkaç yolu vardır. Bu yöntemlerden ikisi:

	- safeTransaction objesine bir owner’ın kendi ethAdapter’ı ile oluşturduğu safe instance objesi ile hem imzalaması hem de bu imzaları safeTransaction objesine eklemesi.
  
	- safeTransaction objesini imzalayıp signature’ı ayrı olarak elde etmek. Bu yöntem Safe Service kullanmak için şimdilik gereklidir.
  
### 5.1 – SafeTransaction objesini imzalamak

```js
await safe.signTransaction(safeTransaction, "eth_signTypedData");
```

Bu metod ile bir safeTransaction objesi safe instance ile imzalanır ve objeye eklenir. Bu işlem birden fazla owner safe instance ile yapılır ise her bir owner signature bu objeye eklenir.

### 5.2 – SignTypedData ile signature objesini ayrı elde etmek

```js
const senderSignature = await safe.signTypedData(safeTransaction);
```

Bu metod ile senderSignature ayrı olarak elde edilebilir. Daha sonrasında bu signature:

```js
safeTransaction.addSignature(
        new EthSignSignature(
          senderAddress,
          senderSignature
        )
      );
```

şeklinde safeTransaction’a eklenebilir fakat bu yöntem için EthSignSignature class’ının projeye eklenmesi gerekmektedir.

```js
const { EthSignSignature } = require("@gnosis.pm/safe-core-sdk");
```

5.2 yöntemi 
