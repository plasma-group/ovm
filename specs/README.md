# Specifications
The OVM is an open source project to unify layer 2 constructions. This repository serves as a collaborative living document which specifies the different components of the OVM. This includes specifications for:

- The Universal Adjudication contract
- Client OVM engine
- Different layer 2 constructions on the OVM
  - Optimistic Rollup
  - Plasma
  - State Channels

## Contributing
Contributions to specs & informational material here are greatly appreciated. This is intended to be a community resource so please join in if it means fixing typos or even proposing alternatives to different components. It's all welcome!

## The Universal Adjudication contract
Layer 2 constructions rely on an interactive cryptoeconomic game which is played out between multiple agents in a single system. We can often model these games as interactive logical proof evaluation. The Universal Adjudication Contract is a smart contract which faciliates the evaluation of these logical claims & contradictions.

#### [Universal Adjudication contract specs](https://github.com/plasma-group/ovm/tree/master/specs/universal-adjudicator)

## Client OVM engine
The OVM engine is what powers OVM-enabled wallets and allows them to scale well beyond layer 1 blockchains. The engine upon a more sophisticated interpretation of blockchain state than traditional blockchain clients/wallet software. These constructions rely on not only "layer 1" state, but also off-chain messages which are sent & evaluated against the Universal Adjudication contract. The engine's job is to interpret all of the layer 1 state & off-chain messages to return information about your blockchain assets like your balance or other crypto assets.

#### [OVM Engine specs](https://github.com/plasma-group/ovm/tree/master/specs/ovm-engine)

## OVM Layer 2 Constructions
The OVM is a universal client for L2. However, there are only a few L2 constructions that we need to cover most of what is required for blockchain development. The following are the big 3:

### Optimistic Rollup
Optimistic Rollup can be thought of as the layer 1 of layer 2. Even though it relies layer 2 dispute games, it maintains properties which closely resemble layer 1 which relax user-side liveness requirements & **enable smart contracts on L2**.

#### [Optimistic Rollup specs](https://github.com/plasma-group/ovm/tree/master/specs/optimistic-rollup)

### Plasma
Plasma is a massively scalable & relatively general purpose L2 construction. It trades off fully autonomous smart contract execution for enormous scale which is near equlivant to centralized solutions.

#### [Plasma specs](https://github.com/plasma-group/ovm/tree/master/specs/plasma)

### State Channels
State channels are an essential L2 technology which enables arbitrary state machines to be constructed with fully concenting participants. This enables things like micropayments, or competitive video games, with instant counterfactual finality.

#### [State Channel specs](https://github.com/plasma-group/ovm/tree/master/specs/state-channels)
