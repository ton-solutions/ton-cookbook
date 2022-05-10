# 🎟 TON smart contract for NFT raffle

Made by [TON Fruits](https://t.me/tonfruits\_news)

This smart contract allows you to raffle off [NFT](https://github.com/ton-blockchain/TIPs/issues/62) on TON Blockchain. To do that, you need to deploy this contract, set the list of participants and transfer your NFT's to this contract.

To add participants you need to send internal message with opcode 2 and array of participant addresses. Here is an the example how to do it on javascript, it is using `ton` package by [tonwhales](https://tonwhales.com):

```typescript
import { Address, serializeDict, Cell } from "ton";

const messageBody = new Cell();

messageBody.bits.writeUint(2, 32); // op

messageBody.bits.writeBit(true)
const ticketsCell = serializeDict(
    new Map<string, Address>(tickets.map((v, k) => [k.toString(), v])),
    32,
    (value, cell) => cell.bits.writeAddress(value)
)
messageBody.refs.push(ticketsCell)
messageBody.bits.writeUint(tickets.length, 32)
```

You can do this multiple times and all the sent addresses will participate. Also several participants with same address are supported, this will increase probability to win for this address.

To choose winners you need to send internal message with opcode 1 and the array of NFT addresses. :

```typescript
import { Address, serializeDict, Cell } from "ton";

const messageBody = new Cell();

messageBody.bits.writeUint(1, 32); // op

messageBody.bits.writeBit(true)
const nftsCell = serializeDict(
    new Map<string, Address>(nfts.map((v, k) => [k.toString(), v])),
    32,
    (value, cell) => cell.bits.writeAddress(value)
)
messageBody.refs.push(nftsCell)
```

Here is the smart-contract source code:

{% embed url="https://gist.github.com/ton-solutions/a691f27fb39abffaf4b56a1b465eabf4" %}
