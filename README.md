# Specifications for the Unicity Project

## Consensus Layer

### PoW Ledger

- [PoW Documentation Hub](https://github.com/unicitynetwork/unicity-pow/blob/master/docs/README.md)

### BFT Core

- [BFT Core](bft-core-spec/unicity-bft-core.pdf) is part of the Consensus Layer. Summary of [data structures in ABNF format](bft-core-spec/unicity-data-structs.abnf).


## Aggregation Layer

- [Sparse Merkle Trees](smt/smt.md) are used in the aggregator to track spent states
- [Sum-Certifying Sparse Merkle Trees](smt/ssmt.md) are used in split transactions to prove that the sum of values of the newly minted tokens is equal to the value of the original split token


## Token Layer

- [Next version](https://github.com/unicitynetwork/execution-model-tex/blob/main/execution-sdk-doc.md) of state transition SDK data structures (initially with reduced feature set)
