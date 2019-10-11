---
title: Devcon 5 - "Chain Relay Under Attack CTF" Writeup
date: 2019-10-11T13:55:33+09:00
draft: false
bigimg: [{src: "/img/relay-attack.png"}]
tags: ["Blockchain", "CTF"]
---

[Devcon 5 // Oct 8 - 11, 2019](https://devcon.org/)

の3日目に開催されたworkshop「[Nuts and Bolts of Cross-Chain Communication](https://devcon.org/agenda?talk=recmTw2dH0oMt9Dzb)」でのCTFが新鮮で面白かったです。(DEFCONではない)

問題はシンプルで、「Ethereumのスマートコントラクトが与えられる。このコントラクトはBitcoinのブロックチェーンを構築するが、不正なブロックが送られても対処できるようになっていない。直してね」というもの。なんでこういう問題かっていうのは、[彼らのクロスチェーンコミュニケーションの論文](https://eprint.iacr.org/2019/1128.pdf)を参照。

Flagを取る形式ではなく、実際に攻撃を捌ければ点数が貰えるフルフィードバック形式。

## 問題

問題リポジトリ: [crossclaim/devcon5-relay-attack: Game repo for DEVCON 5 Cross-Chain Communication Workshop](https://github.com/crossclaim/devcon5-relay-attack)

`attackTestCases.test.min.js`にサンプルケースが転がってます。
testで結果が変わることがあって大変でした。

### TESTCASE 1: Do we allow users to reset the relay with a new genesis block at will??

**サンプルケース**
```js
it("TESTCASE 1: set duplicate initial parent - should fail",
    async () => {
    l(),
        await c.reverts(relay.setInitialParent(genesis.header,
            genesis.height))
    }),
```

Genesis block が改ざんされないようにする問題。
```
require(_headers[blockHeaderHash].merkleRoot == 0, ERR_GENESIS_ALREADY_SET);
```
や
```
require(_heaviestHeight == 0, ERR_GENESIS_ALREADY_SET);
```
を追加すればいい。


### TESTCASE 2: Maybe we should not allow duplicate submissions?

```js
it("TESTCASE 2: duplicate block submission - should fail",
    async () => {
    l(),
        block1 = r[1];
        let e = await relay.submitBlockHeader(block1.header);
        c.eventEmitted(e,
            "StoreHeader",
            e => e.blockHeight == block1.height),
            await c.reverts(relay.submitBlockHeader(block1.header),
                t.ERROR_CODES.ERR_DUPLICATE_BLOCK)
    }),
```

すでに block が store されてきたらダメ。
```
require(_headers[hashCurrentBlock].merkleRoot == 0, ERR_DUPLICATE_BLOCK_SUBMISSION);
```

### TESTCASE 3a, 3b: block header is provided by the user. What could go wrong??

```js
it("TESTCASE 3a: too large block header - should fail",
    async () => {
    l(),
        block1 = r[1],
        await c.reverts(relay.submitBlockHeader(block1.header + "123"),
            t.ERROR_CODES.ERR_INVALID_HEADER)
    }),
it("TESTCASE 3b: too small block header - should fail",
    async () => {
    l(),
        block1 = r[1],
        await c.reverts(relay.submitBlockHeader(block1.header.substring(0,
            28)),
            t.ERROR_CODES.ERR_INVALID_HEADER)
    }),
```

ブロックヘッダーの長さは80byteでなくてはいけない。
```
require(blockHeaderBytes.length == 80, ERR_INVALID_HEADER_FORMAT);
```

### TESTCASE 4: Shall we make sure we are building a chain and not storing random blocks?

```js
it("TESTCASE 4: submit block where prev block is not in main chain - should fail",
    async () => {
    l(),
        block2 = r[2],
        await c.reverts(relay.submitBlockHeader(block2.header),
            t.ERROR_CODES.ERR_PREV_BLOCK)
    }),
```

Previous block が store されてなければ、不正。
```
require(_headers[hashPrevBlock].merkleRoot != 0, ERR_PREV_BLOCK_NOT_FOUND);
```

### TESTCASE 5: Did the miner do the work?

```js
it("TESTCASE 5: weak block submission - should fail",
    async () => {
    fakeGenesis = {
        hash: "0x00000000000000000012af6694accf510ca4a979824f30f362d387821564ca93",
        height: 597613,
        merkleroot: "0x1c7b7ac77c221e1c0410eca20c002fa7b6467ba966d700868928dae4693b3b78",
        header: "0x00000020614db6ddb63ec3a51555336aed1fa4b86e8cc52e01900e000000000000000000783b3b69e4da28898600d766a97b46b6a72f000ca2ec10041c1e227cc77a7b1c6a43955d240f1617cb069aed"
    },
        fakeBlock = {
            hash: "0x000000000000000000050db24a549b7b9dbbc9de1f44cd94e82cc6863b4f4fc0",
            height: 597614,
            merkleroot: "0xc090099a4b0b7245724be6c7d58a64e0bd7718866a5afa81aa3e63ffa8acd69d",
            header: "0x0000002093ca64158287d362f3304f8279a9a40c51cfac9466af120000000000000000009dd6aca8ff633eaa81fa5a6a861877bde0648ad5c7e64b7245720b4b9a0990c07745955d240f16171c168c88"
        },
        await relay.setInitialParent(fakeGenesis.header,
            fakeGenesis.height),
        await c.reverts(relay.submitBlockHeader(fakeBlock.header),
            t.ERROR_CODES.ERR_LOW_DIFF)
    }),
```

```
uint256 target = getTargetFromHeader(blockHeaderBytes)
```

で、difficulty target を取得している。
ブロックのハッシュ値がこれを上回らなければOK。

```
require(hashCurrentBlock <= bytes32(target), ERR_DIFF_TARGET_HEADER);
```



### TESTCASE 6: txid is provided by the user. What could go wrong?

```js
it("TESTCASE 6: empty txid - should fail",
    async () => {
    l(),
        block1 = r[1];
        let e = await relay.submitBlockHeader(block1.header);
        c.eventEmitted(e,
            "StoreHeader",
            e => e.blockHeight == block1.height),
            tx = block1.tx[0],
            await c.reverts(relay.verifyTx("0x0000000000000000000000000000000000000000000000000000000000000000",
                block1.height,
                tx.tx_index,
                tx.merklePath,
                0),
                t.ERR_INVALID_TXID)
    }),
```

トランザクションIDが0だったら不正。

```
require(txid != bytes32(0), ERR_INVALID_TXID);
```

### TESTCASE 8: Are we sure this transaction is "securely" included?

```js
it("TESTCASE 8: missing tx confirmation check - should fail",
    async () => {
    l(),
        block1 = r[1];
        let e = await relay.submitBlockHeader(block1.header);
        c.eventEmitted(e,
            "StoreHeader",
            e => e.blockHeight == block1.height),
            confirmations = 10,
            r.slice(2,
                4).forEach(e => { relay.submitBlockHeader(e.header) }),
            tx = block1.tx[0],
            await c.reverts(relay.verifyTx(tx.tx_id,
                block1.height,
                tx.tx_index,
                tx.merklePath,
                confirmations),
                t.ERR_CONFIRMS)
    })
```

10 confirmations してるか確認すればいい

```
require(_headers[_heaviestBlock].blockHeight - txBlockHeight >= confirmations, ERR_CONFIRMS)
```