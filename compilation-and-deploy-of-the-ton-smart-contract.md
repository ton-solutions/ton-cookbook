---
description: How to compile and deploy smart contract on TON Blockchain using javascript?
---

# 🔨 Compilation and deploy of the TON smart contract

After reading this article, you will know how to compile and deploy TON smart contract with the [TON Fruits](./) example

✅ You can find working example [here](https://gist.github.com/ton-solutions/6552866b5b56b6e76fa32a3811cc4843)

So, first of all let's install dependencies [ton](https://github.com/tonwhales/ton), [ton-compiler](https://github.com/tonwhales/ton-contracts) и [tweetnacl](https://tweetnacl.js.org/#/):

```bash
npm i -D ton ton-compiler tweetnacl
```

Add required imports:

```javascript
import { promises } from 'fs'
import { Address, Cell, CellMessage, contractAddress, serializeDict, TonClient } from 'ton'
import { compileFunc } from 'ton-compiler'
import nacl from 'tweetnacl'
```

Lets create a key pair for the smart contract management, and save it into the file:

```javascript
const { publicKey, secretKey } = nacl.sign.keyPair()
await promises.writeFile('key.pk', Buffer.from(secretKey))
```

Then we need to compile the source code. We use [TON Fruits](./) source code as an example:

```javascript
// You may find slot.fc content here https://gist.github.com/ton-solutions/c300d0ebb0a3ee920c8e8b310a451e29
const funcSource = (await promises.readFile('slot.fc')).toString('utf-8')
const compilationResult = await compileFunc(funcSource)
const initialCode = Cell.fromBoc(compilationResult.cell)[0]
```

Prepare initial contract state:

```javascript
const reelsCount = 5
const symbolsCount = 8
const payTable = new Map([
    ['2111', 50],
    ['221', 100],
    ['311', 200],
    ['32', 250],
    ['41', 2000],
    ['5', 5000]
])

const slotParams = new Cell()
slotParams.bits.writeUint(symbolsCount, 8)
slotParams.bits.writeUint(reelsCount, 8)
slotParams.bits.writeBit(true)
const payTableCell = serializeDict(payTable, 32, (value, cell) => {
    cell.bits.writeUint(value, 32)
})
slotParams.refs.push(payTableCell)

let initialData = new Cell()
 // seq_no
initialData.bits.writeUint(0, 32)
// Owner address
initialData.bits.writeAddress(Address.parse('<YOUR_ADDRESS>'))
// Deployer public key
initialData.bits.writeBuffer(Buffer.from(publicKey))
initialData = initialData.withReference(slotParams)
```

Then we need to get smart contract address. So we call `contractAddress` from the `ton` package, passing `initialCode` and `initialData`, that we received earlier:

```javascript
const contractSource = {
    initialCode,
    initialData,
    workchain: 0,
}

const address = await contractAddress(contractSource)
```

After you need to top up balance of this address (around 0.03 TON). You may just log it into console, and then top up it through [wallet](https://tonkeeper.com) or [@CryptoBot](https://t.me/CryptoBot):

```javascript
console.log(`Please topup ${address.toFriendly()} balance and press enter:`)
```

After topping balance up we can start deploy our contract:

```javascript
const payload = new Cell()
payload.bits.writeUint(0, 32) // seq_no
const hash = await payload.hash()
const signature = nacl.sign.detached(hash, secretKey)
const body = new Cell()
body.bits.writeBuffer(Buffer.from(signature.buffer))
body.writeCell(payload)

const client = new TonClient({
    // you can choose desired network here (mainnet/testnet)
    endpoint: 'https://testnet.toncenter.com/api/v2/jsonRPC'
})
await client.sendExternalMessage(
    {
        address,
        source: contractSource
    },
    body
)
```

Well done, contract is deployed. Happy coding! 💎
