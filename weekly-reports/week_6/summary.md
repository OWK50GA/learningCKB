# Weekly Progress Report
## Week 3: Early Feedback on IDL Project

The past week was mostly about reaching out to testers to tell them about the IDL project, and ask them to test the [IDL](https://github.com/OWK50GA/ckb-idl-derive) project on their scripts, and find out whether it is something they would use.

So far, a few testers have been gotten who are willing to write special scripts to test IDLs on.
The feedback so far points to three things:

- Type Ambiguity: there is a type ambiguity with the project, and it is also something I saw coming from the start. I thought it was avoidable, but it turns out its not. There are a few blessed types, driven mainly by Rust generics, but the number of bytes and the byte-size of a variable doesn't give you all the information you might want about a witness argument and can still end in mistakes to an extent, which is a problem. However, since compilers are the target, this might not be too much of a problem
- Extra work involved: currently, the CKB-IDL-DERIVE requires the user to define a separate struct by themselves called the Witness Struct, and this would be used to generate the IDL. This was done just for prototyping. The original plan was to actually analyze their code and see how they go about validating witnesses, and then derive the witness struct from there.
This is still being looked into, and it would eliminate the extra work.
Partners in the project have talked about AI automatically deriving IDLs, but also agreed that it is not right to leave this computation that is needed to be deterministic, to the hands of probabilistic AI.

This week is sure to come with more feedback, and improvements are being worked on for this project.