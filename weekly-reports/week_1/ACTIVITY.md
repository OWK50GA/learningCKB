## Activity for the Week

In Week 1, I took a dive into learning CKB, and reused some of my knowledge on Bitcoin. The chain is really interesting, and I was able to toy with a few concepts.

Here is my activity summary:
- Theory: added learning and understanding the cell and script model of CKB to the current repo, documented in the rest of week 1 folder. I also learned the functional differences betwen a lock script and a type script, and how they are structurally the same, and it all depends on which one is executed where.
- Practical: I toyed with CKB concepts a bit, coming up with the following:
- - First Script: Used the tutorial [Rust example minimal script](https://docs.nervos.org/docs/script/rust/rust-example-minimal-script) to build a simple type script that ensures the inputs and outputs did not contain the word, "carrot" in bytes. I altered it a bit, and wrote more tests (negative and postive cases) to ensure functionality. The repo is [here](https://github.com/OWK50GA/first-ckb-script).
- - CKB Chain CLI Explorer: In order to understand some types and get used to CKB, I built a minimal chain explorer for CKB using Rust. Find it [here](https://github.com/OWK50GA/ckb-chain-cli-explorer). It uses a CKB Client, and can find the latest transactions created, and can locate and print cells by their outpoints, as long as the cells exist.
- - CKB SUDT Script: Wrote a type script to further understand scripting flow in CKB. The Type Script is a simple implementation of the user-defined token, with tests added, both negative and positive test cases. Find it [here](https://github.com/OWK50GA/ckb_sudt_script).