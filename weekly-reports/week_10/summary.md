## SUMMARY

### Introduction to Fibre Network

This week, instead of continuing the IDL project, I decided to take a step back, and look at the fibre network and what I could do on there, so that I can participate in the upcoming hackathon.

Fibre network is a peer-to-peer payment/swap network, which is basically the Lightning network idea rebuilt for CKB's transaction structure. Apparently, it is the same core insight as lightning: do not put every payment on-chain - lock funds into a channel, transact off-chain by exchanging signed state updates, and only touch the chain to open/close.

Here is where it diverges from Lightning:

1. Multi-party funding is native, not bolted on:
Channel establishment uses a TxUpdate/TxComplete construction protocol, which is an adaptation and simplification of Lightning's Bolt 02 interactive transaction construction, simplified specifically because CKB's transaction structure is more complex/flexible than Bitcoin's. In dual funding, the initiator contributes inputs first via TxUpdate, the responder adds their own inputs, and both sides converge by exchanging TxComplete messages until consecutive TxCompletes confirm agreement. It as interactive PSBT-building over the wire - which is literally my PSCT territory, so file that connection away.

2. Point Time-Locked Contracts instead of Hash Time-Locked Contracts:
Fiber is built on more advanced cryptographic techniques for security and privacy, specifically using PTLC (Point Time-Locked Contracts) rather than HTLC (Hash Time-Locked Contracts). Quick primer since you know HTLC from Lightning: an HTLC reveals the same hash preimage at every hop of a multi-hop payment — an observer correlating hops can link the whole payment path. A PTLC uses adaptor signatures on elliptic curve points instead of hash preimages, so each hop's "proof of payment" is cryptographically unlinkable to the others. Same function (conditional, time-locked payment forwarding), much better privacy — this is the same upgrade Lightning itself has been slow-walking toward

3. Multi-asset by design:
Fiber natively supports stable coins, RGB++ assets issued on the Bitcoin ledger, and UDT assets issued on the CKB ledger — not just the native token. This falls straight out of the Cell model: a Cell can carry a type script (defining what asset it represents) and the channel just moves Cells around. Lightning can't do this natively because Bitcoin UTXOs have no type field — that's the account-model-adjacent workaround (wrapped/pegged assets) Lightning has to lean on.

4. Cross-network interoperability with actual Lightning:
Cross-chain functionality with the Bitcoin Lightning Network has been verified — via RGB++ (Bitcoin-anchored assets settling on CKB). So a payment can hop from a Lightning node onto a Fiber node and back, using RGB++ as the bridge asset layer.

5. Watchtowers: same job, same problem shape:
Exactly like Lightning: since a counterparty could try to broadcast an old, more-favorable channel state after you go offline, Fiber has watchtower support so node operators don't have to stay online 24/7 to police their channels. If you understand Lightning's justice-transaction / penalty model, you already understand this piece — the underlying commitment-transaction-revocation logic transfers directly.

In the coming days, I look forward to properly understanding fiber, looking at and understanding what is already being built on it, and seeing what I could build for the Hackathon, which could scale and turn into a full-on product idea.