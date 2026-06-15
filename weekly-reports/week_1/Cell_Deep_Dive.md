## Cell

### Structure of a Script

How I like to see a ```Script``` in Bitcoin is simply a Vec of Commands. See [Programming Bitcoin](https://github.com/OWK50GA/programming-bitcoin/blob/master/src/ch06_script/script.rs) Now the commands can either be elements (data such as hashes of keys, or signatures, or simple numbers) or operations (opcodes such as OP_DUB, OP_CHECKSIGVERIFY, OP_CHECKSIG, etc). 
How a Script is executed on Bitcoin is that the ScriptSig is placed on top of the ScriptPubkey, they are combined into a single Script, and run.
In CKB, a lock Script is different. It contains three things:
- ```code_hash```: this is the hash of the actual bytecode that will run in CKB-VM. It is like an identifier, since no two different pieces of data have the same hash (with an ideal hashing algorithm). It is like the opcode template. OP_CHECKSIG for instance, refers to a function that will run, where the function is written somewhere entirely different. It is the key to that value, if that makes any sense. You refer to the function by the name OP_CHECKSIG. Here, you refer to the piece of logic that will run by the code_hash. A bit like ClassHash on Starknet, Haha.
- ```args```: This is a byte string passed to the program. For a standard single-signature lock, the args should contain the public key (or its hash), just like with ScriptPubkey where you want an output spendable by whoever can provide a signature that verifies with the pubkey put in the ScriptPubkey. It is like the data pushed after ```OP_DUP OP_HASH160 <pubkeyHash> OP_EQUALVERIFY OP_CHECKSIG```. The ```<pubkeyHash>``` is the ```args``` here.
- ```hash_type```: This tells how to find the code. This tells CKB whether ```code_hash``` refers to a cell's data or a type script's code or something else. So you can lock a cell to logic, not just ownership. 

### Spending a Cell

In order to spend a cell (use it as an input to a transaction), you must provide witness data, such as the signature. 
The VM will:
- Load the program ```code_hash``` points to
- Run the program with your args (from the cell's lock script) and witness data
- If program returns 0, proceed with the transaction i.e. lock is opened.

Ownership, just like with Bitcoin, is by who can provide the witness data that makes the script run successfully.

### Bitcoin analogy Table



### Example: Standard single-signature lock

In a single-signature lock, the:
- code_hash = hash of the secp256k1 verify program
- args = compressed public key
- witness = signature

This is exactly like p2pk, or p2pkh. (I would say p2pk since no material says the hash of the public key is used in the args. The extra p2pkh security is that a potential attacker does not know the public key, so it is more likely that no one other than the owner would be able to provide both a public key that hashes to the pkh, and a signature that verifies with the public key).

### CKB vs Account-based Chains

| Feature | Account Model | Nervos CKB |
|----------|-------------|-------------|
| Data Model | State is stored in a global, mutable, key-value store for each contract account | State is stored across many immutable, self-contained cells, that are created and destroyed |
| Contract Logic | Contract code is stored once in a contract account. Users interact by calling functions on it. | The contract logic is split between the lock script and they type script |
| Execution | A transaction provides input for a function, and the network uses the input to compute its new state: onchain computation | A transaction provides the new state, and the network validates that the state transition happened in tandem with the type script and lock script |