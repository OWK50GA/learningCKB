**Weekly Progress Report**

This week's work was entirely research and protocol design, focused on a gap I identified in CKB's ecosystem.

There are some protocols and standards that could happen between scripts, machine-to-machine, 
just the way smart contracts can talk to each other on EVMs, and some other chains. 

With arrangement like this, there is a lot of things that become possible. CKB is not designed exactly like those chains however.

I studied Bitcoin's BIP-174 (Partially Signed Bitcoin Transactions), and I had thoughts to implement a standard for Partially Signed Cell Transactions. 
This could lead to many applications, in areas such as:
- Standardize hardware wallet signing, universally
- air gapped cold storage of transactions before signing
- improve multisign infrastructures that exist today
- type script govered multi-user cell operations
- time-locked pre-signed transactions

The opportunity is limitless, if we could have partially signed cell transactions on Nervos CKB

I looked at PSBTs at a low level, and why they are possible - the key-value map structure, the six roles (Creator, Updater, Signer, Combiner, Finalizer, Extractor), and how each field in the global, per-input, and per-output maps serves a specific purpose in multi-party and offline signing workflows.

From there I analyzed why a direct adaptation of PSCTs is non-trivial, specifically because CKB's programmable lock scripts cannot declare their witness requirements in any standard machine-readable way, unlike Bitcoin's finite set of known script templates: P2TR, P2SH, P2PKH, etc
THis makes it difficult to create coordination like this between signers. Due to script programmability in CKB, a machine, or anyone constructing something like a PSCT cannot know exactly what witness to expect, so the machine cannot verify that the transaction will most likely succeed. It needs the witness structure to analyze for this to be possible.

This led me to design a proposal for a **Lock Script Interface Description Language (LS-IDL)** — a JSON standard that describes what witness data a lock script requires, anchored on-chain by storing its Blake2b hash in the script's code cell. The design draws on the EVM ABI pattern, but adapted for CKB's cell model and trustless verification requirements.

I intend extending the analysis to type scripts, establishing that the IDL format needs to cover three dimensions — `witness`, `cell_data`, and `args` — to be complete for both lock and type scripts. This way, when we partially sign transactions, the compiler, or in general the machine can verify that this transaction **will most likely succeed**, based on the requirements by the type scripts and lock scripts guarding the cells involved in the transactions.

From the IDL foundation, I drafted a proposal for a **Partially Signed Cell Transaction (PSCT)** format, which depends on the IDL to tell each signer what witness to produce for each input, regardless of what lock script type is involved.

The proposal is still a draft, and will have massive improvements and redesigns as I move forward with the implementation. A week ago, some of the members of the community were talking of ideas on CKB, and an onchain script package manager was mentioned. It wasn't my intention, but I also found out this would cover for that. No, not cover for it, but complement it. 

It is sort of a two-way relationship. The IDL idea is machine facing, while the script registry is developer-facing: pick and plug usable script logic, and I will add that to my implementation as well.

I have begun the implementation locally. Here are the things I am working on:

- IDL Generator in Rust: this is going to be a procedural Macro crate used to generate IDLs for scripts as they are being built on the go. Implementations are going to come in other languages, or TypeScript bindings later, but this is me seeing whether it is possible. It is going to work something like:
```
#[derive(CkbWitness)]
pub struct WitnessArgs {
    pub sig: [u8; 32]
}

pub fn analyze_witness(witness: WitnessArgs) {

}
```
There will be some rules added to this, but the IDL generator will generate the correct interface to be used.
This is currently being engineered locally.

- Script Registry and Package manager: When a script is written, the script has to be deployed in its code cell, where the bytecode for the script is stored in the cell data for the script's code cell. The code cell is then referenced by its code hash. With this new infra, as the code cell is deployed, the IDL is also generated and uploaded to a script registry (just like the one that was talked about). 
The difference is that at deployment, the code cell data is appended with a 32-byte hash of the IDL. This ensures that when any IDL is fetched (IDLs will also be referenced or keyed by code hash, just like the script code cell itself), the fetcher can easily verify that the IDL indeed belongs to the script they are using. This eliminates the need to trust the registry, since that hash is stored onchain.
The IDL solves the issue of machine-interfacing for scripts, and the package manager solves the issue of distribution, since users will now have access to scripts they could easily use in their code.
These are complementary ideas.

- A client library: I worked on this before I got the idea of extending to a script registry, so I do not think it is necessary anymore. It is a client library that can be used to fetch IDLs from an IDL indexer (the package manager by default is now the indexer that makes sense), and verify them against the cell that the IDL is correct. I already worked on this [here](https://github.com/OWK50GA/ckb-idl-client).
However, I think its relevance right now is only for the sake of examples when implementing the script registry.

This IDL idea can give rise to a lot of other innovations, script-to-script interactions, automated testing frameworks, improve dApp frontend integration, script archeology, block explorer script witness decoding, universal wallet integration for all scripts, hardware signing for any lock script, cross-script composition, security auditing tooling, etc. The first of them it will give rise to is the script registry. 
Once developers in the community accept this, the innovations that will follow are limitless.


My work this week on CKB has majorly been research-focused, on scripts.