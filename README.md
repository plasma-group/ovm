# Optimistic Virtual Machine
This repository contains standards and reference resources for those who wish to build or learn about the Optimistic Virtual Machine (OVM).

## What's the OVM?
The OVM is a virtual machine designed to support all layer 2 (L2) protocols. Its generality comes from a reframing of L2 as an “optimistic” fork choice rule layered on top of Ethereum. The formalization borrows heavily from CBC Casper research, and describes layer 2 as a direct extension to layer 1 consensus. This implies a possible unification of all “layer 2 scalability” constructions (Lightning, Plasma, etc) under a single theory and virtual machine: the OVM.

There are two major components:

1. Universal Adjudication Contracts--Ethereum smart contracts which are capible of adjudicating any multi-party dispute.
2. OVM engine--Local OVM engine which optimistically evaluates these on-chain disputes & locally determines the result, enabling L2 scalability.

## Getting Started
Do you want to...

### Learn more about the OVM?
You've come to the right place... or at least you've come to the right place to navigate to the right place! Here are some resources to wet your OVM appetite:

0. [Introducing the OVM](https://medium.com/plasma-group/introducing-the-ovm-db253287af50)
0. [OVM Layer 2 Constructions](https://github.com/plasma-group/ovm/tree/master/specs#ovm-layer-2-constructions)
0. [Universal Adjudication Contract](https://github.com/plasma-group/ovm/tree/master/contracts)
0. [Reference OVM Implementation](https://github.com/plasma-group/pigi)
0. [Rust OVM Implementation](https://github.com/cryptoeconomicslab/plasma-rust-framework)
0. [Development Direction to the L2 Interoperability](https://medium.com/cryptoeconomics-lab/cel-development-direction-to-the-greater-abstraction-6860f87ce0eb)
0. Plase submit a PR with more resources as you come across them!

### Ask questions?
The place to ask questions about the OVM is:

[**https://plasma.build/**](https://plasma.build/)

You'll find a group of folks that are objectively super duper cool and that, more likely than not, would love to answer your questions :)

### Implement the OVM?
Yes please! First step is see if you'd like to contribute to any one of the major implementations:
- Referene OVM implementation
- Rust OVM implementation

If you're looking for another 

### Build a scalable blockchain application?
Soon! And without so much headache!

Developers should not need to learn what a "markle tree" is in order to write a performant & secure application! Instead we should focus on application logic & make use of standard layer 2 backends.

### Integrate the OVM into your wallet?
The OVM is under active development & at this point should not be integrated into any wallets. However, soon the OVM will provide wallets a common interface for querying & managing layer 2 assets. The OVM acts as a cryptoeconomic light clients for layer 2--enabling layer 2 transaction signing & asset managment.

## Contributing
Please submit PRs wherever possible! Chat on plamsa.build! Build an OVM! Build a suuuper cool scalable blockchain application! Be kind to people you meet! Don't listen to strangers on the internet!
