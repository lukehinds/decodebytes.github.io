---
author: "Luke Hinds"
date: "2020-08-28T12:29:56+01:00"
linktitle: Creating a Merkle Tree in the shell
title: Creating a Merkle Tree in the shell
weight: 10
tags: [
    "coding",
    "crypto",
]
categories: [
    "Development",
    "crypto",
]
---


I have known about and used Merkle Trees for a while, but never really bothered to construct one manually (no real need as so many good libraries are out there). Well it’s Saturday night, we are in a lock down, so why not?

For anyone new to Merkle tree’s, they are a binary tree of hashed values concatenated together (one-way hash functions) until an eventual root hash is achieved. The root hash can then be used a sort of integrity snapshot of the entire tree. Merkle trees are currently utilised in three well known applications

* Blockchains (verify transactions)
* Bitorrent (verify file parts)
* Git (bind code revision)
* Transpareny logs (sigstore/rekor & certificate transparency)

The tree starts at the base with a set of “Data segments”. In a blockchain this would be the transaction. In bitorrent, it would be a file part.

Lets build the tree using concatenation and hashing (sha256)

```bash
#!/bin/bash

# Hash the data segments into lead nodes
leafnodes=(HA HB HC HD HE HF HG HH)

for i in "${leafnodes[@]}"
do
   hash=$(echo -n $i | openssl sha256 | awk '{print $2}')
   eval $i=$hash
done

# Construct the parent nodes
HAB=$(echo -n ${HA}${HB} | openssl sha256 | awk '{print $2}')
HCD=$(echo -n ${HC}${HD} | openssl sha256 | awk '{print $2}')

HEF=$(echo -n ${HE}${HF} | openssl sha256 | awk '{print $2}')
HGH=$(echo -n ${HG}${HH} | openssl sha256 | awk '{print $2}')

HABCD=$(echo -n ${HAB}${HCD} | openssl sha256 | awk '{print $2}')
HEFGH=$(echo -n ${HEF}${HGH} | openssl sha256 | awk '{print $2}')

# Compute the Merkle Tree Root
HABCDEFGH=$(echo -n ${HABCD}${HEFGH} | openssl sha256 | awk '{print $2}')

# Get the values we need for verification
echo "HC: ${HC}"
echo "HAB: ${HAB}"
echo "HEFGH: ${HEFGH}"

echo -n "HABCDEFGH: ${HABCDEFGH}"
```

The result of this script will be the merkle root `HABCDEFGH` and we also spit out some other hashes will we need for verifying later on.

```yaml
HC: 616e8cc2cc762815bce92ba8e817da87e92c98b3e3e5c42697caaa76e18c6129
HAB: 545f638df3d2ba4d2295cd1cf6506c05b8340afd5a014f704c741245aab86831
HEFGH: 6d109b1eb6c0ae1353ce98977f15091d2e5d28664f24f362e697da8fc13a6617
HABCDEFGH: fe65b452f05c006bee04415be7a53030dbcb16040bfd1eb19b3b02f95b4d44d7
```

## Verify a single Data Segment

Let’s say we now want to verify data segment `HD`. instead of requiring every data segment, we only need the leafs of `HEFGH`, `HAB` and `HC` and the root `HABCDEFGH`.

![merkle](https://raw.githubusercontent.com/lukehinds/lukehinds.github.io/master/img/merkle-tree.jpg)

Lets start by grabbing `HD` the segment we wish to verify

```bash
HD=$(echo -n HD | openssl sha256 | awk '{print $2}') ; echo $HD
323e41792751e840ad9e398d5057120cb1f1b730acbb193ee6498eb3b481059c
```

We can now concat and hash `HC` | `HD` to get `HCD`

```bash
HCD=$echo -n ${HC}${HD} | openssl sha256 | awk '{print $2}')
0f0143cd71707a8db9e6c53b0c7b3fefbf6e16a3a0048d83e81979f753380c42
```

Concat and hash `HAD` to `HAB` to get `HABCD`

```bash
HABCD=$(echo -n ${HAB}${HCD} | openssl sha256 | awk '{print $2}'); echo $HABCD
843518e9d62f8b81c794d0ac47e82e2f3f335892b33014262958c9215ff8703b
```

Concat and hash `HABCD` to `HEFGH`

```bash
HABCDEFGH=$(echo -n ${HABCD}${HEFGH} | openssl sha256 | awk '{print $2}'); echo $HABCDEFGH
fe65b452f05c006bee04415be7a53030dbcb16040bfd1eb19b3b02f95b4d44d7
```

And there we have our root and have verified HD. We were able to verify a data segment from hashing the object we want verified, by having only three hashes from the tree and the root hash `"HEFGH", "HAB", "HC", "HABCDEFGH"`
