## **Introduction to CKB**

CKB means "Common Knowledge Base", but more on that later.
CKB is a new UTXO-based blockchain, that can be seen as a **programmable Bitcoin**. In itself, Bitcoin is made for payments, and payments only. There is no storage of data. There is just money (bitcoin), and everything surrounding it.

### **UTXO**
Bitcoin runs on a UTXO model, not an account model. UTXO means **Unspent Transaction Output**. In account-based blockchains, spending is equivalent to pure arithmetic subtraction of the money to be spent, from the total balance available. Say you have 300 ETH. You can simply spend 10 ETH by subtracting 10 from 300, as long as you have up to the amountin your account.
It is a different game with UTXOs. UTXOs are like the money notes you have in your wallet, like a $100 note, a $50 dollar note, a few cents here and there. When you want to spend, you gather the notes you have in your wallet in a way that they make a sum greater or equal to what you want to spend. If they are greater (and you so desire), you make change for yourself. Spending $120 here would entail the spender putting $100 and $50 together, and getting a change of $30.
However, there is a slight difference between physical notes transactions, and transactions using UTXOs. When you transact using physical notes, the recipeint receives the exact notes you have handed to them. If you gave them a $50 bill, they would have a $50 bill. With UTXOs, it is more like putting these units (notes) in a vending machine that destroys the notes you have and creates new notes. Spending $150 is more like putting a $100 and a $50 in a vending machine, the vending machine destroys both notes, and since you want to spend $120, creates 2 notes: the $120 and the $30. Gives the $120 to the recipient (according to how you write the code), and the $30 to another recipient (usually your own wallet in this case.)
There are a lot of things skipped here, but that is basically how a UTXO works.

### **CKB**
CKB operates on the UTXO-model Bitcoin uses. UTXOs in CKB are called *Cells*. Unlike traditional UTXOs are used in Bitcoin, CKB cells do not store only monetary value. They also store other forms of data. The Cell is a more flexible version of Bitcoin's UTXO. It contains two main parts:
- The Capacity: this is the number of bytes that the cell can contain. 1 CKB gives you 1 byte of storage space onchain. It is like the value of UTXOs, but here the value is not just monetary, but also in terms of space. If you have 100 CKB, you can create a cell that is 100 bytes in capacity. 
- Data:  It is usually an unformatted byte array. It is versatile and can store anything from a simple string or hash to the source code of a decentralized application.
- Lock Script: The lockscript in a cell usually holds and enforces the ownership rules. It is comparable to ScriptPubkey in Bitcoin. It acts like a key to a lock, setting the condition for spending or transferring the cell. For a standard payment, it works like the scriptPubKey in Bitcoin, securing the asset to a user's private key
- Type Script: The typescript in a cell defines the state transition rule, so in a way is the main smart contract logic. It ensures that certain rules are enforced when transforming that particular cell during a transaction.

The lock script is the default lock, while the type script is optional, when you want to enforce certain rules on the transformation.

### **The Bitcoin -> CKB Mental Map**

| **Bitcoin** | **CKB** | Definition |
| -------- | ------- | ------------ |
| Unspent Transaction Output (UTXO) | Cell | Container/Store of value |
| ScriptPubkey | Lock Script | Enforce ownership rules on Container |
| ScriptSig (Witness in SegWit) | Witness args | Key to open container when you need to |
| Spending a UTXO | Consuming a Cell | - |
| No generic data | Cell has arbitrary ``data`` field | - |
| No type rules on UTXOs | Type Script validates transactions | - |