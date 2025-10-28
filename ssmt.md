# Sum-Certifying Sparse Merkle Tree

## General Structure

Sum-certifying Merkle trees in Unicity are implemented as an extension of sparse Merkle trees (SMT) where each node has an associated non-negative value; for a leaf node it's the value associated with the data in the leaf; for a non-leaf node it's the sum of values associated with the leaf nodes in its sub-tree.

The figure below shows a sum-certifying tree with all nodes labelled with their associated values and edges labelled with the descriptions of the paths.

<img src="ssmt-fig-augment.svg" width="200" />

The hash value of a leaf node is `hash(path, data, value)` where `path` is the label on the edge connecting the node to its parent. The hash value of a non-leaf node (also called a branch node) is `hash(path, left-hash, left-value, right-hash, right-value)` where `path` is the label on the edge connecting the node to its parent, `left-hash` and `right-hash` are the hash values of the child nodes, and `left-value` and `right-value` are the values associated with the child nodes. As with regular SMT, the `path` of the root node is taken to be an empty (zero-length) bit-string.

## Inclusion Proofs

The inclusion proof for any leaf can be presented as a sequence of `(path, data, value)` triples. The first triple in the sequence contains the `path`, `data` and `value` arguments from the `hash(path, data, value)` expression of the leaf node's hash value. In each of the subsequent pairs, the `path` element is the `path` argument from the `hash(path, left-hash, left-value, right-hash, right-value)` expression of the next node on the path from the leaf to the root, the `data` element is the "sibling" hash value, and the `value` element is the value associated with the "sibling" node. That is, if the starting leaf is in the left sub-tree of the branch node, then `data` and `value` will contain the `right-hash` and `right-value` arguments from the corresponding `hash(path, left-hash, left-value, right-hash, right-value)` expression, and vice versa.

Suppose the inclusion proof for some leaf node is `(path[1], data[1], value[1]), (path[2], data[2], value[2]), ..., (path[N], data[N], value[N])`. It's easy to verify that the root hash of the tree can then be recomputed as follows:

- `hash[1] = hash(path[1], data[1], value[1])`
- if `value[1]` is negative, fail with error
- `sum = value[1]`
- if the rightmost bit of `path[1]` is 0, then \
   `hash[2] = hash(path[2], hash[1], sum, data[2], value[2])`, else \
   `hash[2] = hash(path[2], data[2], value[2], hash[1], sum)`
- if `value[2]` is negative or `sum + value[2]` would overflow, fail with error
- `sum = sum + value[2]`
- ...
- if the rightmost bit of `path[N-1]` is 0, then \
   `hash[N] = hash(path[N], hash[N-1], sum, data[N], value[N])`, else \
   `hash[N] = hash(path[N], data[N], value[N], hash[N-1], sum)`
- if `value[N]` is negative or `sum + value[N]` would overflow, fail with error
- `sum = sum + value[N]`

If, after these steps, `hash[N]` matches the root hash of the tree, this proves that the leaf with the path to root equal to `path[1] + path[2] + ... + path[N]` indeed contained `data[1]` and had `value[1]`associated to it in the SMT from which the sequence was extracted, and the root node had the final value of `sum` associated to it.

## Sharding

We currently do not define sharding for sum-certifying trees.

## CBOR Serialization

We apply the same serialization rules to sum-certifying trees as to the regular SMTs.

## Examples

TODO
