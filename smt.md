# Sparse Merkle Tree

## Contents

- [Path Compression](#path-compr)
- [Inclusion Proofs](#incl-proof)
- [Sharding](#sharding)
- [CBOR Serialization](#cbor)
- [Examples](#examples)
    - [Singleton](#root)
    - [Left Child Only](#left)
    - [Right Child Only](#right)
    - [Two Leaves](#two)
    - [Two Leaves, Sharded](#two-sharded)
    - [Four Leaves](#four)
    - [Four Leaves, Sharded](#four-sharded)

## Path Compression <a name="path-compr">

The aggregation tree in Unicity is a sparse Merkle tree (SMT) maintained in the "compressed paths" form. This means that generally only non-empty leaves and only parent nodes with two child nodes are kept explicitly. (The root node is an exception in that it may have only one or even zero child nodes.)

The figure below shows a tree with all non-empty nodes retained (on the left) and the same tree with path compression applied (on the right). In both cases, each edge is labelled with the description of the path from the child node to the parent node.

<img src="smt-fig-path-compr.svg" width="400" />

The hash value of a leaf node is `hash(path, data)` where `path` is the label on the edge connecting the node to its parent. The hash value of a non-leaf node (also called a branch node) is `hash(path, left, right)` where `path` is the label on the edge connecting the node to its parent and `left` and `right` are the hash values of the child nodes. For hashing the root node, the `path` is taken to be an empty (zero-length) bit-string.

## Inclusion Proofs <a name="incl-proof">

The inclusion proof for any leaf can be presented as a sequence of `(path, data)` pairs. The first pair in the sequence contains the `path` and `data` arguments from the `hash(path, data)` expression of the leaf node's hash value. In each of the subsequent pairs, the `path` element is the `path` argument from the `hash(path, left, right)` expression of the next node on the path from the leaf to the root, and the `data` element is the "sibling" hash value. That is, if the starting leaf is in the left sub-tree of the branch node, then `data` contains the `right` argument from the corresponding `hash(path, left, right)` expression, and vice versa.

Suppose the inclusion proof for some leaf node is `(path[1], data[1]), (path[2], data[2]), ..., (path[N], data[N])`. It's easy to verify that the root hash of the tree can then be recomputed as follows:

- `hash[1] = hash(path[1], data[1])`
- if the rightmost bit of `path[1]` is 0, then \
   `hash[2] = hash(path[2], hash[1], data[2])`, else \
   `hash[2] = hash(path[2], data[2], hash[1])` \
- ...
- if the rightmost bit of `path[N-1]` is 0, then \
   `hash[N] = hash(path[N], hash[N-1], data[N])`, else \
   `hash[N] = hash(path[N], data[N], hash[N-1])`

If, after these steps, `hash[N]` matches the root hash of the tree, this proves that the leaf with the path to root equal to `path[1] + path[2] + ... + path[N]` indeed contained `data[1]` in the SMT from which the sequence was extracted.

## Sharding <a name="sharding">

In sharded setting, the key space is partitioned according to a fixed number of bits of the key. This way, each shard aggregator manages a sub-tree (the two lower dashed boxes in each of the figures below) and the parent aggregator joins the sub-trees into a single tree (the higher dashed boxes).

<img src="smt-fig-sharding.svg" width="520" />

The root nodes of the shard trees are the leaf nodes of the parent tree. To present a stable interface to the shard aggregators, the parent aggregator's tree is not really a sparse one. Instead, it has all the leaves present, and as a corollary also all the internal nodes (as shown on the right in the figure above), even if some leaves are null because there are no entries in the corresponding shard tree. The latter will not happen in practice, because there is no need to shard a tree that has so few entries, but is defined here for completeness of specification.

To facilitate generation of the inclusion proofs as described in the previous section, each shard aggregator computes its root hash as `hash(p, left, right)`, where `p` is the 1-bit "path" consisting of the leftmost bit of its shard identifier. Because the leftmost bit of the shard identifier determines whether the shard's root hash is the left or the right child of its parent node in the parent aggregator's tree, this yields the same value as if the parent aggregator had computed the whole tree by itself.

Note that the inclusion proof extracted from the parent aggregator's tree can't be verified following the steps shown in the previous section. Instead, a slightly modified version would have to be used, because the values in the leaf nodes of the parent tree (shown in red in the figure above) already contain the hash values to be used as the `left` and `right` arguments in the `hash(path, left, right)` expressions of their parent nodes (shown in blue in the figure above). Therefore, the correct first step of the verification algorithm in this case is just `hash[1] = data[1]`.

However, clients of the Unicity service need not concern themselves with this distinction, because the shard aggregator combines the inclusion proof received from the parent aggregator with its own and the resulting complete inclusion proof delivered to clients can always be verified with the regular rule defined in the previous section.

## CBOR Serialization <a name="cbor">

For hashing and signing, data structures are serialized in Concise Binary Object Representation (CBOR, IETF RFC 8949) using the deterministic encoding rules (Sec. 4.2 of the RFC).

Tuples are encoded as CBOR arrays in the natural way: a tuple consisting of *N* elements is encoded as an *N*-element CBOR array. A missing value is encoded as the CBOR simple value `null`.

As there's no native bit-string type in CBOR, bit-strings are represented as follows:

- one 1-bit is prepended to the original bit-string;
- zero to seven 0-bits are prepended to the result of the previous step so that the total number of bits is a multiple of 8;
- the result of the previous step is encoded as a CBOR byte-string in the left-to-right, highest-to-lowest order.

For example, the 12-bit string `0101'1010'1111` is padded to the 16-bit string `0001'0101'1010'1111` and then encoded as the 3-byte sequence `0x42` (byte-string, length 2), `0x15` (the padding `0001` and the bits `0101`), `0xaf` (the bits `1010'1111`).

Hash values, although frequently defined as general bit-strings in cryptographic theory, in practice always have bit-lengths divisible by 8 and are encoded as simple CBOR byte-strings with no padding.

## Examples <a name="examples">

### Singleton <a name="root">

A "singleton" tree consisting of a root node with no child nodes:

<img src="smt-fig-root-only.svg" width="200" />

CBOR diagnostic notation: `[h'01', null, null]`

CBOR encoding, annotated:
```
83         # array(3)
   41      # bytes(1)
      01   # 0000'0001
   F6      # null
   F6      # null
```

```
sha256(83'4101'F6'F6) = 1e54402898172f2948615fb17627733abbd120a85381c624ad060d28321be672
```

#### Inclusion Proofs

No inclusion proof can be extracted from such a tree.

### Left Child Only <a name="left">

A tree consisting of a root node with a left child:

<img src="smt-fig-left-only.svg" width="200" />

The only child has 2-bit key `00` and 1-byte value "a".

**Leaf Node**

CBOR diagnostic notation: `[h'04', h'61']`

CBOR encoding, annotated:
```
82         # array(2)
   41      # bytes(1)
      04   # 0000'0100
   41      # bytes(1)
      61   # "a"
```

```
sha256(82'4104'4161) = 973634e81de87e025343da667dc296872682b66b51432879999238aee6d0373c
```

**Root Node**

CBOR diagnostic notation: `[h'01', h'973634e81de87e025343da667dc296872682b66b51432879999238aee6d0373c', null]`

CBOR encoding, annotated:
```
83         # array(3)
   41      # bytes(1)
      01   # 0000'0001
   58 20   # bytes(32)
      973634E81DE87E025343DA667DC296872682B66B51432879999238AEE6D0373C
   F6      # null
```

```
sha256(83'4101'5820973634E81DE87E025343DA667DC296872682B66B51432879999238AEE6D0373C'F6) = ccd73506d27518c983860a47a6a323d41038a74f9339f5302798563cb168f12f
```

#### Inclusion Proof

The inclusion proof for the only leaf is \
`[h'04', h'61']` \
`[h'01', null]`

### Right Child Only <a name="right">

A tree consisting of a root node with a right child:

<img src="smt-fig-right-only.svg" width="200" />

The only child has 2-bit key `11` and 1-byte value "b".

**Leaf Node**

CBOR diagnostic notation: `[h'07', h'62']`

CBOR encoding, annotated:
```
82         # array(2)
   41      # bytes(1)
      07   # 0000'0111
   41      # bytes(1)
      62   # "b"
```

```
sha256(82'4107'4162) = ea0c1acccbc165a448c4d60d05c0ee3184cb463e6212d5c8c7b5fabe1d70eba1
```

**Root Node**

CBOR diagnostic notation: `[h'01', null, h'ea0c1acccbc165a448c4d60d05c0ee3184cb463e6212d5c8c7b5fabe1d70eba1']`

CBOR encoding, annotated:
```
83         # array(3)
   41      # bytes(1)
      01   # 0000'0001
   F6      # null
   58 20   # bytes(32)
      EA0C1ACCCBC165A448C4D60D05C0EE3184CB463E6212D5C8C7B5FABE1D70EBA1
```

```
sha256(83'4101'F6'5820EA0C1ACCCBC165A448C4D60D05C0EE3184CB463E6212D5C8C7B5FABE1D70EBA1) = 5219d2dac90ad497a82a5231f10cffaf5a12dc65b762be39a6d739b4159136a3
```

#### Inclusion Proof

The inclusion proof for the only leaf is \
`[h'07', h'62']` \
`[h'01', null]`

### Two Leaves <a name="two">

A tree consisting of a root node with a left and a right child:

<img src="smt-fig-two-leaves.svg" width="200" />

The left child has 2-bit key `00` and 1-byte value "a".
The right child has 2-bit key `11` and 1-byte value "b".

**Left Leaf**

CBOR diagnostic notation: `[h'04', h'61']`

CBOR encoding, annotated:
```
82         # array(2)
   41      # bytes(1)
      04   # 0000'0100
   41      # bytes(1)
      61   # "a"
```

```
sha256(82'4104'4161) = 973634e81de87e025343da667dc296872682b66b51432879999238aee6d0373c
```

**Right Leaf**

CBOR diagnostic notation: `[h'07', h'62']`

CBOR encoding, annotated:
```
82         # array(2)
   41      # bytes(1)
      07   # 0000'0111
   41      # bytes(1)
      62   # "b"
```

```
sha256(82'4107'4162) = ea0c1acccbc165a448c4d60d05c0ee3184cb463e6212d5c8c7b5fabe1d70eba1
```

**Root Node**

CBOR diagnostic notation: `[h'01', h'973634e81de87e025343da667dc296872682b66b51432879999238aee6d0373c', h'ea0c1acccbc165a448c4d60d05c0ee3184cb463e6212d5c8c7b5fabe1d70eba1']`

CBOR encoding, annotated:
```
83         # array(3)
   41      # bytes(1)
      01   # 0000'0001
   58 20   # bytes(32)
      973634E81DE87E025343DA667DC296872682B66B51432879999238AEE6D0373C
   58 20   # bytes(32)
      EA0C1ACCCBC165A448C4D60D05C0EE3184CB463E6212D5C8C7B5FABE1D70EBA1
```

```
sha256(83'4101'5820973634E81DE87E025343DA667DC296872682B66B51432879999238AEE6D0373C'5820EA0C1ACCCBC165A448C4D60D05C0EE3184CB463E6212D5C8C7B5FABE1D70EBA1) = 7d527038c3b55ec2e83ad309f4f3b464d3eb337932d150ca4a17d55a245cdf77
```

#### Inclusion Proofs

The inclusion proof for the left leaf: \
`[h'04', h'61']` \
`[h'01', h'ea0c1acccbc165a448c4d60d05c0ee3184cb463e6212d5c8c7b5fabe1d70eba1']`

The inclusion proof for the right leaf: \
`[h'07', h'62']` \
`[h'01', h'973634e81de87e025343da667dc296872682b66b51432879999238aee6d0373c']`

### Two Leaves, Sharded <a name="two-sharded">

A tree consisting of a root node with two leaves, managed as two shards with 1-bit identifiers `0` and `1`, respectively:

<img src="smt-fig-two-leaves-sharded.svg" width="200" />

The left leaf has 2-bit key `00` and 1-byte value "a".
The right leaf has 2-bit key `11` and 1-byte value "b".

#### Left Shard

**Left Leaf**

CBOR diagnostic notation: `[h'02', h'61']`

CBOR encoding, annotated:
```
82         # array(2)
   41      # bytes(1)
      02   # 0000'0010
   41      # bytes(1)
      61   # "a"
```

```
sha256(82'4102'4161) = 2222ead87965dbd1046ff0f4d09f9901222b0426681d222aff2954d7f4dcc1d3
```

**Root Node**

CBOR diagnostic notation: `[h'02', h'2222ead87965dbd1046ff0f4d09f9901222b0426681d222aff2954d7f4dcc1d3', null]`

CBOR encoding, annotated:
```
83         # array(3)
   41      # bytes(1)
      02   # 0000'0010
   58 20   # bytes(32)
      2222EAD87965DBD1046FF0F4D09F9901222B0426681D222AFF2954D7F4DCC1D3
   F6      # null
```

```
sha256(83'4102'58202222EAD87965DBD1046FF0F4D09F9901222B0426681D222AFF2954D7F4DCC1D3'F6) = 256aedd9f31e69a4b0803616beab77234bae5dff519a10e519a0753be49f0534
```

#### Right Shard

**Right Leaf**

CBOR diagnostic notation: `[h'03', h'62']`

CBOR encoding, annotated:
```
82         # array(2)
   41      # bytes(1)
      03   # 0000'0011
   41      # bytes(1)
      62   # "b"
```

```
sha256(82'4103'4162) = 50e3c959cf3fc159f5138e4e2638003a5051ce62ab59dc4605ac8d7a069b35eb
```

**Root Node**

CBOR diagnostic notation: `[h'03', null, h'50e3c959cf3fc159f5138e4e2638003a5051ce62ab59dc4605ac8d7a069b35eb']`

CBOR encoding, annotated:
```
83         # array(3)
   41      # bytes(1)
      03   # 0000'0011
   F6      # null
   58 20   # bytes(32)
      50E3C959CF3FC159F5138E4E2638003A5051CE62AB59DC4605AC8D7A069B35EB
```

```
sha256(83'4103'F6'582050E3C959CF3FC159F5138E4E2638003A5051CE62AB59DC4605AC8D7A069B35EB) = e777763b4ce391c2f8acdf480dd64758bc8063a3aa5f62670a499a61d3bc7b9a
```

#### Parent

Root of left child: `256aedd9f31e69a4b0803616beab77234bae5dff519a10e519a0753be49f0534`

Root of right child: `e777763b4ce391c2f8acdf480dd64758bc8063a3aa5f62670a499a61d3bc7b9a`

**Root Node**

CBOR diagnostic notation: `[h'01', h'256aedd9f31e69a4b0803616beab77234bae5dff519a10e519a0753be49f0534', h'e777763b4ce391c2f8acdf480dd64758bc8063a3aa5f62670a499a61d3bc7b9a']`

CBOR encoding, annotated:
```
83         # array(3)
   41      # bytes(1)
      01   # 0000'0001
   58 20   # bytes(32)
      256AEDD9F31E69A4B0803616BEAB77234BAE5DFF519A10E519A0753BE49F0534
   58 20   # bytes(32)
      E777763B4CE391C2F8ACDF480DD64758BC8063A3AA5F62670A499A61D3BC7B9A
```

```
sha256(83'4101'5820256AEDD9F31E69A4B0803616BEAB77234BAE5DFF519A10E519A0753BE49F0534'5820E777763B4CE391C2F8ACDF480DD64758BC8063A3AA5F62670A499A61D3BC7B9A) = 413b961d0069adfea0b4e122cf6dbf98e0a01ef7fd573d68c084ddfa03e4f9d6
```

#### Inclusion Proofs

**Left Leaf**

The inclusion proof for the left shard's root from the parent aggregator: \
`[h'02', h'256aedd9f31e69a4b0803616beab77234bae5dff519a10e519a0753be49f0534']` \
`[h'01', h'e777763b4ce391c2f8acdf480dd64758bc8063a3aa5f62670a499a61d3bc7b9a']`

The inclusion proof segment for the left leaf from the shard aggregator: \
`[h'02', h'61']` \
`[h'02', null]`

The complete inclusion proof for the left leaf, obtained by concatenating the shard aggregator's proof and the parent aggregator's proof, except for the first step of the latter: \
`[h'02', h'61']` \
`[h'02', null]` \
`[h'01', h'e777763b4ce391c2f8acdf480dd64758bc8063a3aa5f62670a499a61d3bc7b9a']`

**Right Leaf**

The inclusion proof for the right shard's root from the parent aggregator: \
`[h'03', h'e777763b4ce391c2f8acdf480dd64758bc8063a3aa5f62670a499a61d3bc7b9a']` \
`[h'01', h'256aedd9f31e69a4b0803616beab77234bae5dff519a10e519a0753be49f0534']`

The inclusion proof segment for the left leaf from the shard aggregator: \
`[h'03', h'62']` \
`[h'03', null]`

The complete inclusion proof for the right leaf, obtained by concatenating the shard aggregator's proof and the parent aggregator's proof, except for the first step of the latter: \
`[h'03', h'62']` \
`[h'03', null]` \
`[h'01', h'256aedd9f31e69a4b0803616beab77234bae5dff519a10e519a0753be49f0534']`

### Four Leaves <a name="four">

A tree containing four leaves:

<img src="smt-fig-four-leaves.svg" width="200" />

- A leaf with 3-bit key `000` and 1-byte value "a".
- A leaf with 3-bit key `100` and 1-byte value "b".
- A leaf with 3-bit key `011` and 1-byte value "c".
- A leaf with 3-bit key `111` and 1-byte value "d".

**Leaf "a"**

CBOR diagnostic notation: `[h'02', h'61']`

CBOR encoding, annotated:
```
82         # array(2)
   41      # bytes(1)
      02   # 0000'0010
   41      # bytes(1)
      61   # "a"
```

```
sha256(82'4102'4161) = 2222ead87965dbd1046ff0f4d09f9901222b0426681d222aff2954d7f4dcc1d3
```

**Leaf "b"**

CBOR diagnostic notation: `[h'03', h'62']`

CBOR encoding, annotated:
```
82         # array(2)
   41      # bytes(1)
      03   # 0000'0011
   41      # bytes(1)
      62   # "b"
```

```
sha256(82'4103'4162) = 50e3c959cf3fc159f5138e4e2638003a5051ce62ab59dc4605ac8d7a069b35eb
```

**Parent of "a" and "b"**

CBOR diagnostic notation: `[h'04', h'2222ead87965dbd1046ff0f4d09f9901222b0426681d222aff2954d7f4dcc1d3', h'50e3c959cf3fc159f5138e4e2638003a5051ce62ab59dc4605ac8d7a069b35eb']`

CBOR encoding, annotated:
```
83         # array(3)
   41      # bytes(1)
      04   # 0000'0100
   58 20   # bytes(32)
      2222EAD87965DBD1046FF0F4D09F9901222B0426681D222AFF2954D7F4DCC1D3
   58 20   # bytes(32)
      50E3C959CF3FC159F5138E4E2638003A5051CE62AB59DC4605AC8D7A069B35EB
```

```
sha256(83'4104'58202222EAD87965DBD1046FF0F4D09F9901222B0426681D222AFF2954D7F4DCC1D3'582050E3C959CF3FC159F5138E4E2638003A5051CE62AB59DC4605AC8D7A069B35EB) = 571b7ef9469e4516ecc628ac0e7bbfb9032d739bcd44613b3594f03c0b208a67
```

**Leaf "c"**

CBOR diagnostic notation: `[h'02', h'63']`

CBOR encoding, annotated:
```
82         # array(2)
   41      # bytes(1)
      02   # 0000'0010
   41      # bytes(1)
      63   # "c"
```

```
sha256(82'4102'4163) = 6338c7ad0dc943f4e31052cdf2e9751fcaee9ff50a3e1bda97c51e05e7e7c79f
```

**Leaf "d"**

CBOR diagnostic notation: `[h'03', h'64']`

CBOR encoding, annotated:
```
82         # array(2)
   41      # bytes(1)
      03   # 0000'0011
   41      # bytes(1)
      64   # "d"
```

```
sha256(82'4103'4164) = 3fb43b8e381a3d05470aa184c5695c938c7d7a5d43bd595a936b4dbc2539a669
```

**Parent of "c" and "d"**

CBOR diagnostic notation: `[h'07', h'6338c7ad0dc943f4e31052cdf2e9751fcaee9ff50a3e1bda97c51e05e7e7c79f', h'3fb43b8e381a3d05470aa184c5695c938c7d7a5d43bd595a936b4dbc2539a669']`

CBOR encoding, annotated:
```
83         # array(3)
   41      # bytes(1)
      07   # 0000'0111
   58 20   # bytes(32)
      6338C7AD0DC943F4E31052CDF2E9751FCAEE9FF50A3E1BDA97C51E05E7E7C79F
   58 20   # bytes(32)
      3FB43B8E381A3D05470AA184C5695C938C7D7A5D43BD595A936B4DBC2539A669
```

```
sha256(83'4107'58206338C7AD0DC943F4E31052CDF2E9751FCAEE9FF50A3E1BDA97C51E05E7E7C79F'58203FB43B8E381A3D05470AA184C5695C938C7D7A5D43BD595A936B4DBC2539A669) = b77a56cc8a7f0db572a2c95092b722dce4a9e3366d0832ebb0f4668bc942cf88
```

**Root Node**

CBOR diagnostic notation: `[h'01', h'571b7ef9469e4516ecc628ac0e7bbfb9032d739bcd44613b3594f03c0b208a67', h'b77a56cc8a7f0db572a2c95092b722dce4a9e3366d0832ebb0f4668bc942cf88']`

CBOR encoding, annotated:
```
83         # array(3)
   41      # bytes(1)
      01   # 0000'0001
   58 20   # bytes(32)
      571B7EF9469E4516ECC628AC0E7BBFB9032D739BCD44613B3594F03C0B208A67
   58 20   # bytes(32)
      B77A56CC8A7F0DB572A2C95092B722DCE4A9E3366D0832EBB0F4668BC942CF88
```

```
sha256(83'4101'5820571B7EF9469E4516ECC628AC0E7BBFB9032D739BCD44613B3594F03C0B208A67'5820B77A56CC8A7F0DB572A2C95092B722DCE4A9E3366D0832EBB0F4668BC942CF88) = 95005e568fdac5cc01a3a091c70ce89ab2da98c36b254dd2ddf29bd568c377ab
```

#### Inclusion Proofs

The inclusion proof for leaf "a" is \
`[h'02', h'61']` \
`[h'04', h'50e3c959cf3fc159f5138e4e2638003a5051ce62ab59dc4605ac8d7a069b35eb']` \
`[h'01', h'b77a56cc8a7f0db572a2c95092b722dce4a9e3366d0832ebb0f4668bc942cf88']`

The inclusion proof for leaf "b" is \
`[h'03', h'62']` \
`[h'04', h'2222ead87965dbd1046ff0f4d09f9901222b0426681d222aff2954d7f4dcc1d3']` \
`[h'01', h'b77a56cc8a7f0db572a2c95092b722dce4a9e3366d0832ebb0f4668bc942cf88']`

The inclusion proof for leaf "c" is \
`[h'02', h'63']` \
`[h'07', h'3fb43b8e381a3d05470aa184c5695c938c7d7a5d43bd595a936b4dbc2539a669']` \
`[h'01', h'571b7ef9469e4516ecc628ac0e7bbfb9032d739bcd44613b3594f03c0b208a67']`

The inclusion proof for leaf "d" is \
`[h'03', h'64']` \
`[h'07', h'6338c7ad0dc943f4e31052cdf2e9751fcaee9ff50a3e1bda97c51e05e7e7c79f']` \
`[h'01', h'571b7ef9469e4516ecc628ac0e7bbfb9032d739bcd44613b3594f03c0b208a67']`

### Four Leaves, Sharded <a name="four-sharded">

A tree containing four leaves, managed as four shards with 2-bit identifiers `00`, `10`, `01` and `11`, respectively (where the shards `00` and `11` are empty):

<img src="smt-fig-four-leaves-sharded.svg" width="240" />

- A leaf with 4-bit key `0010` and 1-byte value "a".
- A leaf with 4-bit key `1010` and 1-byte value "b".
- A leaf with 4-bit key `0101` and 1-byte value "c".
- A leaf with 4-bit key `1101` and 1-byte value "d".

#### Middle-Left Shard

**Leaf "a"**

CBOR diagnostic notation: `[h'02', h'61']`

CBOR encoding, annotated:
```
82         # array(2)
   41      # bytes(1)
      02   # 0000'0010
   41      # bytes(1)
      61   # "a"
```

```
sha256(82'4102'4161) = 2222ead87965dbd1046ff0f4d09f9901222b0426681d222aff2954d7f4dcc1d3
```

**Leaf "b"**

CBOR diagnostic notation: `[h'03', h'62']`

CBOR encoding, annotated:
```
82         # array(2)
   41      # bytes(1)
      03   # 0000'0011
   41      # bytes(1)
      62   # "b"
```

```
sha256(82'4103'4162) = 50e3c959cf3fc159f5138e4e2638003a5051ce62ab59dc4605ac8d7a069b35eb
```

**Parent of "a" and "b"**

CBOR diagnostic notation: `[h'02', h'2222ead87965dbd1046ff0f4d09f9901222b0426681d222aff2954d7f4dcc1d3', h'50e3c959cf3fc159f5138e4e2638003a5051ce62ab59dc4605ac8d7a069b35eb']`

CBOR encoding, annotated:
```
83         # array(3)
   41      # bytes(1)
      02   # 0000'0010
   58 20   # bytes(32)
      2222EAD87965DBD1046FF0F4D09F9901222B0426681D222AFF2954D7F4DCC1D3
   58 20   # bytes(32)
      50E3C959CF3FC159F5138E4E2638003A5051CE62AB59DC4605AC8D7A069B35EB
```

```
sha256(83'4102'58202222EAD87965DBD1046FF0F4D09F9901222B0426681D222AFF2954D7F4DCC1D3'582050E3C959CF3FC159F5138E4E2638003A5051CE62AB59DC4605AC8D7A069B35EB) = afc631129cf648ef8b4ce8e5027ae587f70191fe04e571d48bd89da7c2dc6a81
```

**Root Node**

CBOR diagnostic notation: `[h'03', h'afc631129cf648ef8b4ce8e5027ae587f70191fe04e571d48bd89da7c2dc6a81', null]`

CBOR encoding, annotated:
```
83         # array(3)
   41      # bytes(1)
      03   # 0000'0011
   58 20   # bytes(32)
      AFC631129CF648EF8B4CE8E5027AE587F70191FE04E571D48BD89DA7C2DC6A81
   F6      # null
```

```
sha256(83'4103'5820AFC631129CF648EF8B4CE8E5027AE587F70191FE04E571D48BD89DA7C2DC6A81'F6) = 10c1dc89e30d51613f2c1a182d16f87fe6709b9735db612adaadaa91955bdaf0
```

#### Middle-Right Shard

**Leaf "c"**

CBOR diagnostic notation: `[h'02', h'63']`

CBOR encoding, annotated:
```
82         # array(2)
   41      # bytes(1)
      02   # 0000'0010
   41      # bytes(1)
      63   # "c"
```

```
sha256(82'4102'4163) = 6338c7ad0dc943f4e31052cdf2e9751fcaee9ff50a3e1bda97c51e05e7e7c79f
```

**Leaf "d"**

CBOR diagnostic notation: `[h'03', h'64']`

CBOR encoding, annotated:
```
82         # array(2)
   41      # bytes(1)
      03   # 0000'0011
   41      # bytes(1)
      64   # "d"
```

```
sha256(82'4103'4164) = 3fb43b8e381a3d05470aa184c5695c938c7d7a5d43bd595a936b4dbc2539a669
```

**Parent of "c" and "d"**

CBOR diagnostic notation: `[h'03', h'6338c7ad0dc943f4e31052cdf2e9751fcaee9ff50a3e1bda97c51e05e7e7c79f', h'3fb43b8e381a3d05470aa184c5695c938c7d7a5d43bd595a936b4dbc2539a669']`

CBOR encoding, annotated:
```
83         # array(3)
   41      # bytes(1)
      03   # 0000'0011
   58 20   # bytes(32)
      6338C7AD0DC943F4E31052CDF2E9751FCAEE9FF50A3E1BDA97C51E05E7E7C79F
   58 20   # bytes(32)
      3FB43B8E381A3D05470AA184C5695C938C7D7A5D43BD595A936B4DBC2539A669
```

```
sha256(83'4103'58206338C7AD0DC943F4E31052CDF2E9751FCAEE9FF50A3E1BDA97C51E05E7E7C79F'58203FB43B8E381A3D05470AA184C5695C938C7D7A5D43BD595A936B4DBC2539A669) = 957b8aaa765b5065e616316573c4d03d153269f8adba2fe2b9ceb6eab0d5f21a
```

**Root Node**

CBOR diagnostic notation: `[h'02', null, h'957b8aaa765b5065e616316573c4d03d153269f8adba2fe2b9ceb6eab0d5f21a']`

CBOR encoding, annotated:
```
83         # array(3)
   41      # bytes(1)
      02   # 0000'0010
   F6      # null
   58 20   # bytes(32)
      957B8AAA765B5065E616316573C4D03D153269F8ADBA2FE2B9CEB6EAB0D5F21A
```

```
sha256(83'4102'F6'5820957B8AAA765B5065E616316573C4D03D153269F8ADBA2FE2B9CEB6EAB0D5F21A) = 981d2f4e01189506c5a36430e7774e3f9498c1c4cc27801d8e6400d4965a8860
```

#### Parent

Root of middle-left shard: `10c1dc89e30d51613f2c1a182d16f87fe6709b9735db612adaadaa91955bdaf0`

Root of middle-right shard: `981d2f4e01189506c5a36430e7774e3f9498c1c4cc27801d8e6400d4965a8860`

**Parent of left and middle-left shards**

CBOR diagnostic notation: `[h'02', null, h'10c1dc89e30d51613f2c1a182d16f87fe6709b9735db612adaadaa91955bdaf0']`

CBOR encoding, annotated:
```
83         # array(3)
   41      # bytes(1)
      02   # 0000'0010
   F6      # null
   58 20   # bytes(32)
      10C1DC89E30D51613F2C1A182D16F87FE6709B9735DB612ADAADAA91955BDAF0
```

```
sha256(83'4102'F6'582010C1DC89E30D51613F2C1A182D16F87FE6709B9735DB612ADAADAA91955BDAF0) = 98a5b16cac1bb4684db0f39dec36baa2c4edb1e3c2d2dc01d097ab73a015b5a8
```

**Parent of middle-right and right shards**

CBOR diagnostic notation: `[h'03', h'981d2f4e01189506c5a36430e7774e3f9498c1c4cc27801d8e6400d4965a8860', null]`

CBOR encoding, annotated:
```
83         # array(3)
   41      # bytes(1)
      03   # 0000'0011
   58 20   # bytes(32)
      981D2F4E01189506C5A36430E7774E3F9498C1C4CC27801D8E6400D4965A8860
   F6      # null
```

```
sha256(83'4103'5820981D2F4E01189506C5A36430E7774E3F9498C1C4CC27801D8E6400D4965A8860'F6) = a6cf558550ceb2e350fb85bbc9dc3c31266ef89317184007ce8b50c611886ce7
```

**Root Node**

CBOR diagnostic notation: `[h'01', h'98a5b16cac1bb4684db0f39dec36baa2c4edb1e3c2d2dc01d097ab73a015b5a8', h'a6cf558550ceb2e350fb85bbc9dc3c31266ef89317184007ce8b50c611886ce7']`

CBOR encoding, annotated:
```
83         # array(3)
   41      # bytes(1)
      01   # 0000'0001
   58 20   # bytes(32)
      98A5B16CAC1BB4684DB0F39DEC36BAA2C4EDB1E3C2D2DC01D097AB73A015B5A8
   58 20   # bytes(32)
      A6CF558550CEB2E350FB85BBC9DC3C31266EF89317184007CE8B50C611886CE7
```

```
sha256(83'4101'582098A5B16CAC1BB4684DB0F39DEC36BAA2C4EDB1E3C2D2DC01D097AB73A015B5A8'5820A6CF558550CEB2E350FB85BBC9DC3C31266EF89317184007CE8B50C611886CE7) = eb1a95574056c988f441a50bd18d0555f038276aecf3d155eb9e008a72afcb45
```

#### Inclusion Proofs

**Leaf "a"**

The inclusion proof from the parent aggregator: \
`[h'03', h'10c1dc89e30d51613f2c1a182d16f87fe6709b9735db612adaadaa91955bdaf0']` \
`[h'02', null]` \
`[h'01', h'a6cf558550ceb2e350fb85bbc9dc3c31266ef89317184007ce8b50c611886ce7']`

The inclusion proof from the shard aggregator: \
`[h'02', h'61']` \
`[h'02', h'50e3c959cf3fc159f5138e4e2638003a5051ce62ab59dc4605ac8d7a069b35eb']` \
`[h'03', null]`

The complete inclusion proof: \
`[h'02', h'61']` \
`[h'02', h'50e3c959cf3fc159f5138e4e2638003a5051ce62ab59dc4605ac8d7a069b35eb']` \
`[h'03', null]` \
`[h'02', null]` \
`[h'01', h'a6cf558550ceb2e350fb85bbc9dc3c31266ef89317184007ce8b50c611886ce7']`

**Leaf "b"**

The inclusion proof from the parent aggregator: \
`[h'03', h'10c1dc89e30d51613f2c1a182d16f87fe6709b9735db612adaadaa91955bdaf0']` \
`[h'02', null]` \
`[h'01', h'a6cf558550ceb2e350fb85bbc9dc3c31266ef89317184007ce8b50c611886ce7']`

The inclusion proof from the shard aggregator: \
`[h'03', h'62']` \
`[h'02', h'2222ead87965dbd1046ff0f4d09f9901222b0426681d222aff2954d7f4dcc1d3']` \
`[h'03', null]`

The complete inclusion proof: \
`[h'03', h'62']` \
`[h'02', h'2222ead87965dbd1046ff0f4d09f9901222b0426681d222aff2954d7f4dcc1d3']` \
`[h'03', null]` \
`[h'02', null]` \
`[h'01', h'a6cf558550ceb2e350fb85bbc9dc3c31266ef89317184007ce8b50c611886ce7']`

**Leaf "c"**

The inclusion proof from the parent aggregator: \
`[h'02', h'981d2f4e01189506c5a36430e7774e3f9498c1c4cc27801d8e6400d4965a8860']` \
`[h'03', null]` \
`[h'01', h'98a5b16cac1bb4684db0f39dec36baa2c4edb1e3c2d2dc01d097ab73a015b5a8']`

The inclusion proof from the shard aggregator: \
`[h'02', h'63']` \
`[h'03', h'3fb43b8e381a3d05470aa184c5695c938c7d7a5d43bd595a936b4dbc2539a669']` \
`[h'02', null]`

The complete inclusion proof: \
`[h'02', h'63']` \
`[h'03', h'3fb43b8e381a3d05470aa184c5695c938c7d7a5d43bd595a936b4dbc2539a669']` \
`[h'02', null]` \
`[h'03', null]` \
`[h'01', h'98a5b16cac1bb4684db0f39dec36baa2c4edb1e3c2d2dc01d097ab73a015b5a8']`

**Leaf "d"**

The inclusion proof from the parent aggregator: \
`[h'02', h'981d2f4e01189506c5a36430e7774e3f9498c1c4cc27801d8e6400d4965a8860']` \
`[h'03', null]` \
`[h'01', h'98a5b16cac1bb4684db0f39dec36baa2c4edb1e3c2d2dc01d097ab73a015b5a8']`

The inclusion proof from the shard aggregator: \
`[h'03', h'64']` \
`[h'03', h'6338c7ad0dc943f4e31052cdf2e9751fcaee9ff50a3e1bda97c51e05e7e7c79f']` \
`[h'02', null]`

The complete inclusion proof: \
`[h'03', h'64']` \
`[h'03', h'6338c7ad0dc943f4e31052cdf2e9751fcaee9ff50a3e1bda97c51e05e7e7c79f']` \
`[h'02', null]` \
`[h'03', null]` \
`[h'01', h'98a5b16cac1bb4684db0f39dec36baa2c4edb1e3c2d2dc01d097ab73a015b5a8']`
