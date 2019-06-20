# 未决交易中继

### 概览

上一章节中，我们要给一笔交易记账的话，必须自己手动进行一次挖矿，才会把交易记录加到一个区块里面去。 这一章节中，我们将会引入未决交易中继的机制。有了这个机制之后，我们要进行一笔交易的时候，就不需要自己动手挖矿，而是将自己的交易发送到我们的区块链网络中去(即中继传递的概念)，由其他节点在挖矿之后，将我们的交易记录加到他们挖出的新的区块中去。其中这些交易就被称之为「未决交易」。一个典型的例子就是，当一个用户想要发起一笔交易(把一定数量的币发送到指定的地址)，他会把这笔交易广播到整个网络，并希望其他矿工把该笔交易放到区块中去。

对于一个加密货币系统来说，这个功能异常的重要。因为这将意味着我们再也不需要因为要进行一笔交易而自己进行挖矿以对交易记录进行保存。这将大大提高效率。 毕竟，如比特币一样，随时着时间转移，挖一个矿是越来越难。 如果我们这些家里没矿(矿机)的用户想跟别人交易一些比特币，还要自己挖个矿，那这个交易就不知道何年何月才能达成了。

为了达到广播到其他节点并进行同步的目的，和第一章节进行区块广播和同步一样，我们需要对「未决交易」也要进行广播。 也就是说，我们的区块链系统现在会包含以下的广播和同步：

- 区块链 (即区块链及包含在区块中的「已决交易」记录)
- 未决交易 (还没有包含在任何区块中的交易)

本章节的完整代码请查看[这里](https://github.com/zhubaitian/naivecoin/tree/chapter5)

### 交易池

既然我们的交易现在不是在创建后立即由自己进行挖矿记录的，那么我们就需要将这些交易记录下来，才能广播给其他矿工。我们将保存未决交易的地方叫做「交易池」，在比特币系统中也叫做mempool。 

交易池其实就是我们节点中的一个存储了所有「未决交易」的数据结构，为了实现简单，我们这里将其设计成一个由「交易」数据组成的一个列表(如果不记得交易的结构长什么样的，请翻看上一章节记录):

``` typescript
let transactionPool: Transaction[] = [];
```

当然，为了和此前交易必须自己动手挖矿的那个/mindTransaction入口区分开，我们提供了另外一个接口仅作交易用 **POST /sendTransaction** 。 调用这个方法之后，我们的节点将基于此前的钱包机制在我们的本地交易池中创建一笔新的交易。此后我们都会将这个接口作为进行一笔交易优先使用的交易接口。

``` typescript
app.post('/sendTransaction', (req, res) => {
        ...
    })
```

创建一笔交易的流程和[第四章](https://github.com/zhubaitian/naivecoin/tree/chapter4)描述的流程类似，稍微有点区别的是，当交易创建后，我们会将该交易纳入到我们的交易池，而不再是立刻进行挖矿记录。

### 广播

整个「未决交易】的想法就是将这些交易传播到整个区块链网上，让一些节点最终将其“挖”到区块链中去。为了达成这个目标，我们首先要为这些未决交易的传播建立一些简单的网络规则：

- 当一个节点接收到别人广播出来的交易池时，发现里面有自己此前没有见过的「未决交易】记录时，该节点会将这些没见过的记录加入到自己的交易池中，然后将自己的交易池给广播出去
- 当一个节点和其他节点建立点对点连接时，会向目标节点请求对方保存的完整交易池记录

类似区块链的广播和同步，我们需要建立两个新的消息类型来为我们未决交易池的广播同步进行服务：**QUERY_TRANSACTION_POOL** 和 **RESPONSE_TRANSACTION_POOL** 。最终我们的消息类型数据结构将如下所示:

``` typescript
enum MessageType {
    QUERY_LATEST = 0,
    QUERY_ALL = 1,
    RESPONSE_BLOCKCHAIN = 2,
    QUERY_TRANSACTION_POOL = 3,
    RESPONSE_TRANSACTION_POOL = 4
}
```

交易池广播请求数据最终会通过以下方式进行创建:

``` typescript
const responseTransactionPoolMsg = (): Message => ({
    'type': MessageType.RESPONSE_TRANSACTION_POOL,
    'data': JSON.stringify(getTransactionPool())
}); 

const queryTransactionPoolMsg = (): Message => ({
    'type': MessageType.QUERY_TRANSACTION_POOL,
    'data': null
});
```

当节点收到**RESPONSE_TRANSACTION_POOL**这个广播后，我们需要有相应的代码逻辑来因应请求。无论何时我们收到广播过来的未决交易池，我们都会尝试将其加入到我们的本地交易池中去。当然，我们需要对里面的交易做相应的校验。 只有那些我们此前没有见过的交易(不在我们本地保存的那份交易池清单中)，以及有效的交易，我们才会将其纳入到我们的交易池中。紧跟着，如上所述，我们就会将我们的交易池给广播出去，以达到全网同步的效果:

``` typescript
case MessageType.RESPONSE_TRANSACTION_POOL:
    const receivedTransactions: Transaction[] = JSONToObject<Transaction[]>(message.data);
    receivedTransactions.forEach((transaction: Transaction) => {
        try {
            handleReceivedTransaction(transaction);
            //if no error is thrown, transaction was indeed added to the pool
            //let's broadcast transaction pool
            broadCastTransactionPool();
        } catch (e) {
            //unconfirmed transaction not valid (we probably already have it in our pool)
        }
    });
```

### 未决交易有效性验证

因为广播过来的未决交易数据是不可预知的，我们必须对这些号称是未决交易的数据进行有效性验证。我们此前实现的对交易的有效性检查依然会用上， 比如检查数据格式必须正确，交易的inputs，outputs和签名必须一致(即此前章节描述的validateTxIn函数，简单来说就是对每个交易中的每个input，用交易指向的来源output中的地址作为公钥，对input的签名进行解密，并检查和交易id是否一样，以验证input中引用的交易来源确实是属于用户所有。因为只有该公钥是该用户的才能正确解密本次交易的签名， 这就证明了来源交易的拥有者确实就是本次交易的发起者了)。

``` typescript
const validateTxIn = (txIn: TxIn, transaction: Transaction, aUnspentTxOuts: UnspentTxOut[]): boolean => {
    const referencedUTxOut: UnspentTxOut =
        aUnspentTxOuts.find((uTxO) => uTxO.txOutId === txIn.txOutId && uTxO.txOutIndex === txIn.txOutIndex);
    if (referencedUTxOut == null) {
        console.log('referenced txOut not found: ' + JSON.stringify(txIn));
        return false;
    }
    const address = referencedUTxOut.address;

    const key = ec.keyFromPublic(address, 'hex');
    const validSignature: boolean = key.verify(transaction.id, txIn.signature);
    if (!validSignature) {
        console.log('invalid txIn signature: %s txId: %s address: %s', txIn.signature, transaction.id, referencedUTxOut.address);
        return false;
    }
    return true;
};

```

除了以上已有规则外，我们还需要增加一条新规：如果一笔交易中的任何一个用来引用交易货币来源的input在当前的未决交易池中已经存在了，那么该交易将会被视为无效交易，不会纳入到交易池中去。

``` typescript
const isValidTxForPool = (tx: Transaction, aTtransactionPool: Transaction[]): boolean => {
    const txPoolIns: TxIn[] = getTxPoolIns(aTtransactionPool);

    const containsTxIn = (txIns: TxIn[], txIn: TxIn) => {
        return _.find(txPoolIns, (txPoolIn => {
            return txIn.txOutIndex === txPoolIn.txOutIndex && txIn.txOutId === txPoolIn.txOutId;
        }))
    };

    for (const txIn of tx.txIns) {
        if (containsTxIn(txPoolIns, txIn)) {
            console.log('txIn already found in the txPool');
            return false;
        }
    }
    return true;
};
```
这里没有显式定义将未决交易从交易池中移除的操作，当前做法是，每次网络中产生一个新区块时，将会同时去更新交易池。

### 将交易池挖到区块链中去













