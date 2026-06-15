# PROJECTS

## Summary

Since getting into CKB, I have been more inclined with script writing, and building infrastructure for the Nervos CKB Chain, and the infrastructure around scripts. Right now, my focus is on making scripts machine-verifiable, so that scripts can be verified by other scripts, and in other contexts. That increases the possibility of what can be done with CKB Scripts.
Aso, I am focused on Rust mainly, but in time these projects will touch other languages.

## PROJECT I - First CKB Script
- [Project Link](https://github.com/OWK50GA/first-ckb-script)
- Status: Complete

### Description:

This is an introduction to building scripts on CKB. It is a simple Type Script that makes sure neither the inputs nor the outputs contain the word, "carrot". It was just to learn how script verification works, Source::GroupInput and Source::GroupOutput.
The repository contains the scripts, and it contains tests for the scripts that pass.

## PROJECT II - CKB Chain CLI Explorer
- [Project Link](https://github.com/OWK50GA/ckb-chain-cli-explorer)
- Status: Complete, Improvable

### Description:

This is a project to demonstrate what can be done in CKB with RPC URLs on the Nervos CKB Chain. It is a simple CLI tool that fetches transactions, cells, or any other Nervos CKB units, depending on what you want.
It is a simple project, also written in Rust. 

## PROJECT III - CKB SUDT SCRIPTS -> CKB LOCK SCRIPTS
- [Project Link](https://github.com/OWK50GA/ckb_sudt_script)
- Status: Structured, Regularly Improving

### Description:
This started out as a simple SUDT script, with a rule that verifies that the output cells are never worth more than the input cells, except in owner mode, since for an SUDT, the owner is allowed to mint tokens.
The repository contains the script, and contains tests, and deployment scripts, all wired up and working.

Today, the project has evolved into a center for more Nervos CKB Scripts - I write multiple scripts and push to this repository now, that are used to test my Interface Description Language that is being worked on with CKB.

## PROJECT IV - CKB IDL CLIENT
- [Project Link](https://github.com/OWK50GA/ckb-idl-client)
- Status: Structuted, Pending for another project

### Description:

According to my potential RFC for the Interface Description Language (IDL), the major logic is for IDLs to be derived, and then the hashes to be appened to code cells, so that when they are about to be used, a client fetches the hashes from the code cells, and the hash can be verified by other machines.
More description of the [RFC](../weekly-reports/week_2/ckb-lock-script-idl-rfc.md)
This project is such a client. Right now, it is complete for where it should be.
The next step with it should be making it verify the IDLs, but the design decisions are not finalized.

## PROJECT V - CKB IDL DERIVE
- [Project Link](https://github.com/OWK50GA/ckb-idl-derive)
- Status: Structured, Improving

### Description:

This is a library that can be installed when you write Lock scripts or type scripts, that can be used to derive IDLs (IDLs are not hand-written by the developers). This derives IDL documents, which are pushed to a public registry (from which they can't be deleted, or shouldn't be able to), and the hashes are pushed into code cells.
It is a network of projects being worked on, and the IDL derivation part has a good-enough structure to keep moving.