---
description: Как скомпилировать и задеплоить смарт-контракт в TON Blockchain?
---

# 🔨 Компиляция и деплой смарт-контракта

В этой статье мы расскажем, как скомпилировать и задеплоить смарт-контракт на примере контракта [TON Fruits](./). Файл с работающим примером также находится [здесь](https://gist.github.com/ton-solutions/6552866b5b56b6e76fa32a3811cc4843), если после прочтения у вас остались вопросы, вы можете задать их в [чате](https://t.me/tonfruits\_chat)

И так, в первую очередь нам понадобится установить библиотеки [ton](https://github.com/tonwhales/ton), [ton-compiler](https://github.com/tonwhales/ton-contracts) и [tweetnacl](https://tweetnacl.js.org/#/):

```bash
npm i -D ton ton-compiler tweetnacl
```

Добавим необходимые импорты:

```javascript
import { promises } from 'fs'
import { Address, Cell, CellMessage, contractAddress, serializeDict, TonClient } from 'ton'
import { compileFunc } from 'ton-compiler'
import nacl from 'tweetnacl'
import readline from 'readline'
```

Далее нужно создать пару ключей для управления смарт-контрактом:

```javascript
const { publicKey, secretKey } = nacl.sign.keyPair()
```

Далее нужно скомпилировать смарт-контракт. В качестве примера мы будем использовать исходный код смарт контракта [slot.fc](./):

```javascript
// You may find slot.fc content here https://gist.gitjahub.com/ton-solutions/c300d0ebb0a3ee920c8e8b310a451e29
const funcSource = (await promises.readFile('slot.fc')).toString('utf-8')
const compilationResult = await compileFunc(funcSource)
const initialCode = Cell.fromBoc(compilationResult.cell)[0]
```

Подготовим изначальное состояние:

```javascript
const workchain = 0
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

Далее нам нужно получить адрес смарт-контракта. Для этого нужно создать объект `сontractSource` из ранее полученных `codeBoc` и `initialData`:

```javascript
const contractSource = {
    initialCode: codeBoc,
    initialData: initialData,
    workchain,
}
const address = await contractAddress(contractSource)
```

Сохраним приватный ключ в файл, используя адрес контракта в качестве имени файла. Это позволит нам иметь разные ключи в mainnet и testnet:

```javascript
await promises.writeFile(`${address.toFriendly()}.pk`, Buffer.from(secretKey))
```

Далее необходимо пополнить баланс адреса контракта (приблизительно на 0.03 TON). Для этого можно вывести его в консоли и пополнить через [wallet](https://tonkeeper.com) или [@CryptoBot](https://t.me/CryptoBot):

```javascript
console.log(`Please topup ${address.toFriendly()} balance and press enter:`)
```

После пополнения баланса адреса можно приступить к деплою:

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

Готово, ваш смарт контракт задеплоен
