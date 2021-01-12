# MuSig (*placeholder*) Paxos

MuSig (Multi Signature) paxos is an exponentially scalable decentralized paxos algorithm with byzantine fault tolerance. Scalability of this protocol comes from an underlying aggregatable multi signature scheme called MuSig.

Note: MuSig multi signature scheme is still experimental.

## State machine and events

Like any other paxos algorithm MuSig is a transaction handler. Hence a state machine or an event handler is not directly part of MuSig. A state machine (for example a database) can be integrated with MuSig. Transactions from MuSig can be configured as events to the state machine. There are state machines internal to the protocol.

| Transaction-T1    |     | Transaction-T2    |     | Transaction-T3   |     | Transaction-T4    |
| ----------------- | --- | ----------------- | --- | ---------------- | --- | ----------------- |
|                   | --> |                   | --> |                  | --> |                   |
| Create document 1 |     | Update document 1 |     | Patch document 1 |     | Delete document 1 |

## Protocol

Primarily MuSig is a protocol for collectively deciding the order of incoming transactions. It works in 2 phases and an optional 3rd phase which ensures byzantine fault tolerance. MuSig has 2 kinds of nodes, signers and peers. Signers sign transactions and collectively decide the order of transactions. Signers are like the acceptors and peers are like learners in the basic paxos algorithm. MuSig uses a balanced binary tree network topology of signers where nodes are positioned in the tree pseudo randomly.

## Phase 1a (top-down)

<img align="left" width="250px" src="https://lh5.googleusercontent.com/V3IvAtoqdyJvz80v1Xm6JG1yfKxPCNlkCTKH92XpWvLit9zqgHp2_LbzVV7r9LIH62SZSw2D6IAnkV49ljSv_FQROUzK4pYsrbJHPW39Ng6TdK7bUk6stvAQECG0Eg5jwrZGrKXT"> A client will initiate a transaction by sending it to a signer. Signer then generates a deterministic tree of signers from the transaction id or fingerprint. Then it will do a pre-consensus check and will pass the transaction to its children. It also stores the local arrival order of the transaction.

Every child will run a pre-consensus check upon receiving the transaction from its parent and will pass it to its children until the transaction reaches the leaf signers. No signing is done in this half of the phase.

Even though a client can arbitrarily pick any signer to initiate a transaction, sending all dependent transactions to the same signer could improve performance significantly. More about dependent transactions are explained under “Concurrency via dependency” section.

Every signer uses the same algorithm and seed to generate the pseudo random tree so that everybody gets the same tree for the same transaction. Seed could be the transaction id or a transaction fingerprint. It is also possible to determine the tree based on geographical location of signers for better network performance.

## Phase 1b (bottom-up)

<img align="left" width="250px" src="https://lh5.googleusercontent.com/kXlrPbos0UtlNQ-SOzfr1o6z5K7uQgkUC3BHOIVXOL2EDlqTnIsxmQVce-XB6xEgirt3Rdx6WYWv-YSsT79GeE5cjL2CBbmk4iScmaxnlNM8r-q0RY3OmoZ8jA1vicgS6z-ig3J3">

In this half of the phase, a signer first collects signatures from its children. After a post-consensus check, it co-sign the transaction and its arrival order. It then aggregates its signature with children’s signatures and will send it back to its parent.

A signature is a single collective signature for both the transaction and its order. With signature, a signer also sends a combined order of transaction to its parent. This is a combined set of the orders from its children and its local order.

```
localSignature = sign(transaction, local order)
aggregatedSignature = aggregate(localSignature, signatures from children)
combinedOrder = combine(local order, orders from children)
send(parent, {aggregatedSignature, combinedOrder})
```

## Phase 2 (top-down)

<img align="left" width="250px" src="https://lh4.googleusercontent.com/nPZ0zk_RZllSWNO6QJzXUtGiAjPWJzIF6TWnPX9LXSyRgucrTrZ1GGUeKCow2yKe4jCOUS-xoubgB_LQn-cmFywWrbBiPw1No5pgBEhCV3Yk2C1eBd_y0qe7jCaQg-v5tUDV7p5o">

Root signer collects and aggregates all signatures and arrival orders. It then sends this aggregated signature and order to its children. Once all signers have the signature and order of transaction, each signer can individually derive a global transaction order from it as described in the ‘Ordering transactions’ section.

Signers will also calculate an ordering score. If this score fails to pass a threshold, a byzentine fault is possible and phase 3 is necessary to prevent it.

<p>&nbsp;</p>

## Phase 3a (optional, bottom-up)

<img align="left" width="250px" src="https://lh5.googleusercontent.com/_1RIj6_tsJwIsV1LIJnKcxzDP9GUo2AUxrO8W-KJ-GreR1I3LTQkzeAsrDg1uhf29xNnCg0gwt01YACV11ZcCbeRJlhME5eDuUn5MZ-GV6Tr-McA1y2Oml74pl-HO239KIT0P5F4">

This phase is not necessary if the ordering score passes the threshold. Otherwise every node signs the derived global order and sends it to its parent. How the ordering score is calculated is explained in the ‘Ordering transactions’ section.

Default tolerance is 50% which means the network can tolerate upto 50% malicious signers. For a 50% tolerance, ordering score has to be more than 75%. Below that a flip attack is possible which is explained in the ‘Byzantine faults’ section.

The transaction order which the majority of signers voted for is the global order. All signers will derive the same global order locally irrespective of which order they voted for. Hence all signers will be able to unanimously vote for the same global order in phase 3.

Signers sign the global order if the ordering score is below threshold and will be sent to their parents.

## Phase 3b (optional, top-down)

<img align="left" width="250px" src="https://lh3.googleusercontent.com/kqbUnK3x5duvufaUCCuCE2CtVi6GEADqrozqdmrjlB_QhDRhYTAqRbCKCcxoKn1T_DOVuXrt8E8eOAzO2TXvjB8kz6Nk6D_gCw0AqS2LKWhwxqv9CjiU73iZDmga22HpkzcW_h0E">

Root signer collects and aggregates all global order signatures and sends it back to its children. Failure to achieve the threshold in this phase indicates a vote flip attack. It is easy to identify the faulty signers from the aggregated signature. Also such a transaction will never get accepted.

Ad-hoc solutions can be implemented to remove malicious signers from the network. Presently no such techniques are directly included in MuSig.

Once the signer receives an accepted global order of transaction, it can update its local state machine. Signers then broadcast this transaction to peers.  

<br/>

## Ordering transactions

<img align="left" width="500px" src="https://lh5.googleusercontent.com/ZZVi840BemgbfGPcTKXOgwUKP7tfrrq06RMXCbN1HOzBOtw7_K6ssqQxt9UmZ99K3e-PIbu4C3KwH2Lw4n5teaBZTa6UNTChcfy-oRNpL4SoWcvUHlR-S6rRoIok0oQ00iSJ121c">

A client can initiate a transaction by sending it to any signers in the network. This means multiple signers will be receiving multiple transactions concurrently. Primary duty of MuSig paxos is to enable signers to individually derive a common global order of transactions.

Above diagram shows the timeline of 5 signers and 3 transactions. Usually a signer votr a transaction based on the arrival order. For example, signer-1 receives transactions in an order T1 > T3 > T4. So it votes in the same order. Similarly other signers too.

By the end of phase 2, all signers will have the voting data from other signers. From this a global transaction order can be derived locally. In short, in the above diagram, if the connected line of a transaction is below with respect to another transaction, the transaction below gets the first order. If two lines cross each other, the line with most parts below wins.

All transactions are sorted based on these comparisons. Before comparing 2 transactions, their votes are sorted. This is to avoid the complexities of parallel lines. The result of sorting will be the global order of transactions.

Ordering score of a transaction is its number of votes below with respect to the immediate next transaction. In other words, it’s the percentage length of the line below the next line.

## Concurrency and dependency

Defining dependency is optional. But it is the key for scalability. In basic paxos all transactions fight for a slot which causes bottlenecks. Infact independent transactions can be processed concurrently.

For example Alice wants to transfer some money to Bob. At the same time Carol also wants to make a transfer to Charlie. Since all of them have different accounts, these two transactions are independent and can be processed concurrently. On the other hand, if Alice wants to make a transfer to Bob and Bob wants to make a transfer to Carol then Bob’s account is a dependency and these transactions cannot be done concurrently.

When a client initializes a transaction, it can also send a list of dependencies to the signer. For example in a payment system like above, the account numbers can be the dependencies. In a database CRUD operation, document ids can be dependencies.

MuSig orders dependent transactions as a group. Independent groups will be processed concurrently.

## MuSig Multi Signature

An experimental aggregatable multi signature is the novelty in MuSig paxos which makes the exponential scaling possible.

> ToDo

## Fault tolerance

We can classify faults as Byzantine and non Byzantine faults.

> ToDo

### Non Byzantine faults

These are the unintentional faults of a node like going offline.

> ToDo

### Byzantine faults

These are the intentional faults by a node like double spending.

> ToDo
