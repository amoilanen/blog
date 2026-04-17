---
layout: default
title: "Merkle Trees and Blockchain Verification"
date: 2026-04-17
---

# Merkle Trees and Blockchain Verification

Imagine you are running a Bitcoin wallet on your phone. The full Bitcoin blockchain is over 500 GB and growing. You certainly cannot store it all on a mobile device. Yet somehow your wallet can confirm that a payment you received is legitimate, included in a real block, without downloading that block's worth of transactions. How?

The answer lies in a data structure invented by Ralph Merkle in 1979 and described in his 1987 paper *"A Digital Signature Based on a Conventional Encryption Function"*. Merkle trees make it possible to verify membership in a dataset using only a logarithmic number of hash values. This property is so useful that Merkle trees have become one of the foundational primitives in distributed systems, from blockchains to file distribution networks to database replication protocols.

In this post we will look at how Merkle trees work, why they are secure, and how they enable lightweight blockchain clients. Along the way we will examine a Rust implementation and walk through concrete verification scenarios.

## Contents

- [What is a Merkle tree?](#what-is-a-merkle-tree)
- [Domain separation: a subtle but important detail](#domain-separation-a-subtle-but-important-detail)
- [Inclusion proofs](#inclusion-proofs)
- [Blockchain verification with SPV](#blockchain-verification-with-spv)
- [Beyond blockchains](#beyond-blockchains)
- [Implementation highlights](#implementation-highlights)
- [Summary](#summary)

## What is a Merkle tree?

A Merkle tree is a binary tree where every leaf node contains the hash of a data block, and every internal node contains the hash of its two children concatenated together. The single hash at the top, the **root hash**, acts as a fingerprint for the entire dataset. Change a single byte in any leaf, and the root hash changes.

Here is a Merkle tree built from four transactions:

```
                 Root
              H(H01 | H23)
              /           \
           H01             H23
        H(H0 | H1)     H(H2 | H3)
        /       \        /       \
      H0        H1     H2        H3
    H(tx0)   H(tx1)  H(tx2)   H(tx3)
      |         |       |         |
    tx0       tx1     tx2       tx3
```

Each `H(...)` denotes a cryptographic hash (SHA-256 in our case). The leaf hashes `H0` through `H3` are computed from the raw transaction data. The internal nodes `H01` and `H23` are computed by hashing the concatenation of their children. The root is computed from those two intermediate hashes.

This construction has a key property: given the root hash and a small number of intermediate hashes, anyone can verify that a specific leaf belongs to the tree. We will see exactly how shortly.

**Handling odd numbers of leaves.** What happens when the number of leaves is not a power of two? The standard approach, used in Bitcoin, is to duplicate the last leaf so that every level has an even number of nodes. For example, a tree with three transactions `[tx0, tx1, tx2]` is built as if it had four: `[tx0, tx1, tx2, tx2]`. This keeps the construction simple at the cost of a minor space overhead.

## Domain separation: a subtle but important detail

There is a security concern that is easy to overlook. Consider what happens if leaf data and internal node data are hashed the same way. An attacker could potentially craft leaf data that, when hashed, produces the same bytes as a legitimate internal node hash. This is known as a **second-preimage attack** on the tree structure.

The solution, standardized in [RFC 6962](https://www.rfc-editor.org/rfc/rfc6962) (Certificate Transparency), is **domain separation**: prefix leaf hashes with a `0x00` byte and internal node hashes with a `0x01` byte. This ensures that a leaf hash and an internal node hash can never collide, even if the underlying data happens to be identical.

In Rust this looks quite straightforward:

```rust
/// Domain-separated leaf hash: SHA-256(0x00 || data).
fn hash_leaf(data: &[u8]) -> Hash {
    let mut hasher = Sha256::new();
    hasher.update([0x00]);
    hasher.update(data);
    Hash(hasher.finalize().into())
}

/// Domain-separated internal node hash: SHA-256(0x01 || left || right).
fn hash_pair(left: &Hash, right: &Hash) -> Hash {
    let mut hasher = Sha256::new();
    hasher.update([0x01]);
    hasher.update(left.0);
    hasher.update(right.0);
    Hash(hasher.finalize().into())
}
```

A small prefix, but it closes a real attack vector. It is a good reminder that in cryptographic constructions, the details that seem trivial are often the ones that matter most.

## Inclusion proofs

The most powerful feature of a Merkle tree is the **inclusion proof** (also called a Merkle proof or authentication path). Given a leaf, we can prove it belongs to the tree by providing only the sibling hashes along the path from that leaf to the root. The verifier recomputes the root by hashing upward and checks whether it matches the expected root hash.

Let's trace through a concrete example. Suppose we want to prove that `tx2` is included in our four-transaction tree. The proof consists of the sibling hashes marked with `*`:

```
                 Root
              H(H01 | H23)        <-- verifier computes this and compares
              /           \
          *H01             H23
        H(H0 | H1)     H(H2 | H3)  <-- verifier computes H23 from H2 and *H3
        /       \        /       \
      H0        H1     H2       *H3
                      H(tx2)   H(tx3)
                        |
                      tx2  <-- the leaf we are proving
```

The proof for `tx2` contains two hashes: `H3` (the sibling at the leaf level) and `H01` (the sibling at the next level up). The verification process is:

1. Start with the leaf hash: `H2 = hash_leaf(tx2)`
2. Combine with sibling `H3`: `H23 = hash_pair(H2, H3)`
3. Combine with sibling `H01`: `Root' = hash_pair(H01, H23)`
4. Check: does `Root'` equal the expected root? If yes, `tx2` is in the tree.

The proof size is **O(log n)**: for a tree with *n* leaves, you need only *log2(n)* sibling hashes. A tree with a million leaves requires just 20 hashes in the proof, roughly 640 bytes with SHA-256. This is remarkably compact.

## Blockchain verification with SPV

This logarithmic proof size is exactly what makes **Simplified Payment Verification (SPV)** possible. SPV was described in Section 7 of the Bitcoin whitepaper and is the reason lightweight wallets can function without storing the entire blockchain.

The architecture looks like this:

```
  Full Node                          Light Node
  =========                          ==========

  +------------------+               +------------------+
  | Block 1          |               | Block 1 Header   |
  |   Header         |               |   merkle_root    |
  |   tx0, tx1, ...  |               +------------------+
  |   Merkle tree    |               | Block 2 Header   |
  +------------------+               |   merkle_root    |
  | Block 2          |               +------------------+
  |   Header         |               | ...              |
  |   tx0, tx1, ...  |               +------------------+
  |   Merkle tree    |
  +------------------+
  | ...              |
  +------------------+

  Stores everything.                 Stores only headers
  Can generate proofs.               (including Merkle roots).
                                     Verifies proofs.
```

A **full node** stores every block with all its transactions and the corresponding Merkle tree. A **light node** stores only block headers, which include the Merkle root of each block's transaction tree. When the light node wants to verify a transaction, it asks a full node for a Merkle proof, then verifies it locally against the stored root.

The trust model is important here: the light node does not need to trust the full node. If the full node provides a fraudulent proof, it will not match the Merkle root that the light node already has from the block header chain. The proof is either mathematically correct or it is not.

Let's look at how this works in practice using a Rust implementation. First, we define the participants:

```rust
/// A block on the full node: contains the complete transaction list
/// and its Merkle tree.
struct Block {
    id: String,
    transactions: Vec<Transaction>,
    tree: MerkleTree,
}

/// A full node stores complete blocks and can generate inclusion proofs.
struct FullNode {
    blocks: Vec<Block>,
}

/// A light node stores only block headers — no full transaction lists.
struct LightNode {
    headers: Vec<BlockHeader>,
}
```

The full node generates proofs by looking up a transaction in its Merkle tree:

```rust
impl FullNode {
    fn proof_for(&self, block_id: &str, transaction_index: usize) -> Option<Proof> {
        let block = self.blocks.iter().find(|b| b.id == block_id)?;
        let leaf = block.tree.get_leaf_hash(transaction_index)?;
        block.tree.generate_proof(leaf)
    }
}
```

The light node verifies proofs against its stored Merkle roots:

```rust
impl LightNode {
    fn verify(&self, block_id: &str, proof: &Proof) -> bool {
        self.headers
            .iter()
            .find(|h| h.id == block_id)
            .is_some_and(|h| proof.verify(&h.merkle_root))
    }
}
```

Now let's walk through three scenarios that demonstrate the security properties.

### Scenario 1: Successful verification

A client wants to confirm that the third transaction in block-1 was really included. The full node provides a Merkle proof; the light node verifies it:

```rust
let proof = full_node
    .proof_for("block-1", 2)
    .expect("transaction exists in block");

let verified = light_node.verify("block-1", &proof);
// verified == true
```

For a block with 5 transactions, the proof contains just 3 hashes (*log2(5)* rounded up), regardless of how many transactions the block contains. In Bitcoin, which can have several thousand transactions per block, a proof is still only around 10-12 hashes.

### Scenario 2: Tamper detection

An attacker intercepts the proof and modifies one of the sibling hashes, hoping to trick the light node into accepting a fraudulent transaction:

```rust
let mut tampered_proof = proof.clone();
tampered_proof.steps[0].hash = merkle_tree::hash(b"tampered-data");

let tampered_verified = light_node.verify("block-1", &tampered_proof);
// tampered_verified == false
```

The tampered proof computes a completely different root hash. Even a single bit change in any proof step cascades through the hash chain and produces a root that does not match. This is the avalanche property of cryptographic hash functions working in our favour.

### Scenario 3: Cross-block rejection

A proof generated for one block cannot verify against another block's root. Each block has its own Merkle tree, so proofs are scoped to the block they were generated from:

```rust
let block_b_proof = full_node
    .proof_for("block-2", 0)
    .expect("transaction exists in block-2");

let cross_verified = light_node.verify("block-1", &block_b_proof);
// cross_verified == false

let own_block_verified = light_node.verify("block-2", &block_b_proof);
// own_block_verified == true
```

This ensures that transactions cannot be "moved" between blocks by a malicious full node. A proof is bound to the specific Merkle root, and therefore to the specific block, it was generated from.

## Beyond blockchains

While blockchains are perhaps the most well-known application, Merkle trees appear in many other systems where data integrity and efficient verification matter.

**File distribution and software updates.** Systems like IPFS split files into chunks and organize them into a Merkle DAG (a generalization of Merkle trees). The root hash serves as the content address. A client downloading chunks from untrusted peers can verify each chunk against the root without trusting any single peer. The same principle applies to package managers and firmware update systems: publish a signed root hash, and clients can verify individual files independently.

The pattern is the same one we saw with blockchains. A server holds the full file set and its Merkle tree. A client needs only the trusted root hash. For each file it downloads, it receives a compact proof:

```rust
struct Server {
    files: Vec<ReleaseFile>,
    tree: MerkleTree,
}

struct Client {
    trusted_root: merkle_tree::Hash,
}

impl Client {
    fn verify(&self, proof: &merkle_tree::Proof) -> bool {
        proof.verify(&self.trusted_root)
    }
}
```

A constrained device that only needs two out of six files in a release can verify just those two, without downloading the rest. The proof size is the same regardless of how many files are in the release.

**Database anti-entropy.** Distributed databases like Apache Cassandra and Amazon DynamoDB use Merkle trees to detect inconsistencies between replicas. Each node builds a Merkle tree over its local data. To check whether two replicas are in sync, they compare root hashes. If the roots differ, they walk down the tree together, comparing hashes level by level, to identify exactly which data ranges are out of sync. This is far more efficient than comparing every record: most of the tree is identical, and the traversal focuses only on the divergent branches.

**Certificate Transparency.** Google's Certificate Transparency framework uses Merkle trees to create an append-only log of TLS certificates. Monitors can efficiently verify that a certificate was logged, and auditors can verify that the log is consistent (no entries were removed or altered). This is the same RFC 6962 that standardized domain separation.

## Implementation highlights

Let's look at a few key parts of the Rust implementation to see how the concepts translate into code.

**Building the tree.** The tree is constructed bottom-up: hash each leaf with domain separation, then repeatedly pair and hash until one root remains. When a level has an odd number of nodes, the last one is duplicated:

```rust
pub fn build<T: AsRef<[u8]>>(leaves: &[T]) -> MerkleTree {
    if leaves.is_empty() {
        return MerkleTree {
            levels: vec![vec![]],
            leaf_count: 0,
        };
    }

    let leaf_count = leaves.len();
    let mut current_level: Vec<Hash> =
        leaves.iter().map(|l| hash_leaf(l.as_ref())).collect();
    let mut levels: Vec<Vec<Hash>> = Vec::new();

    while current_level.len() > 1 {
        if current_level.len() % 2 != 0 {
            if let Some(&last) = current_level.last() {
                current_level.push(last);
            }
        }
        levels.push(current_level.clone());

        current_level = current_level
            .chunks_exact(2)
            .map(|pair| hash_pair(&pair[0], &pair[1]))
            .collect();
    }

    levels.push(current_level);
    levels.reverse();

    MerkleTree { levels, leaf_count }
}
```

The tree stores all levels from root to leaves. This is a deliberate trade-off: it uses more memory than strictly necessary, but makes proof generation straightforward since we can walk from any leaf upward through the stored levels.

**Generating proofs.** To generate a proof, we find the leaf in the bottom level and walk upward, collecting the sibling hash at each level:

```rust
pub fn generate_proof(&self, leaf_hash: &Hash) -> Option<Proof> {
    let leaf_level = self.levels.last()?;
    let mut position = leaf_level.iter().position(|h| h == leaf_hash)?;

    let mut steps = Vec::with_capacity(self.levels.len().saturating_sub(1));

    for depth in (1..self.levels.len()).rev() {
        let level = &self.levels[depth];
        let sibling_pos = if position % 2 == 0 {
            position + 1
        } else {
            position - 1
        };
        let sibling_hash = level[sibling_pos];
        let direction = if position % 2 == 0 {
            Direction::Right
        } else {
            Direction::Left
        };
        steps.push(ProofStep {
            hash: sibling_hash,
            direction,
        });
        position /= 2;
    }

    Some(Proof {
        leaf: *leaf_hash,
        steps,
    })
}
```

Note that the direction (left or right) is recorded for each sibling. This is essential: `hash_pair(A, B)` produces a different result from `hash_pair(B, A)`, so the verifier must reconstruct the pairs in the correct order.

**Verifying proofs.** Verification is a fold: start with the leaf hash and combine it with each sibling, respecting the recorded direction, until you reach the root:

```rust
pub fn compute_root(&self) -> Hash {
    self.steps.iter().fold(self.leaf, |current, step| {
        match step.direction {
            Direction::Left => hash_pair(&step.hash, &current),
            Direction::Right => hash_pair(&current, &step.hash),
        }
    })
}

pub fn verify(&self, expected_root: &Hash) -> bool {
    self.compute_root() == *expected_root
}
```

This is the entire verification logic. It is elegant in its simplicity: a single pass over the proof steps, a hash comparison at the end. No tree reconstruction, no complex state. The verifier does not even need to know the size of the original tree.

## Summary

Merkle trees solve a fundamental problem in distributed systems: how to verify that a piece of data belongs to a larger dataset without possessing the entire dataset. The key insights are:

- **A single root hash summarizes arbitrarily large data.** Any change to any leaf propagates up and alters the root. This gives us tamper evidence.
- **Inclusion proofs are logarithmic.** Verifying one leaf out of a million requires only 20 hashes. This makes verification practical even for resource-constrained clients.
- **Verification is trustless.** The verifier needs only the root hash from a trusted source. Everything else, the proof data, the full node providing it, can be untrusted. The math either checks out or it does not.
- **Domain separation prevents structural attacks.** The simple technique of prefixing leaf and internal node hashes with different bytes prevents a class of attacks where crafted leaf data could masquerade as internal nodes.

These properties explain why Merkle trees appear everywhere: Bitcoin and Ethereum for transaction verification, IPFS for content addressing, Certificate Transparency for auditing TLS certificates, Cassandra and DynamoDB for replica synchronization, Git for content-addressable storage, and many more.

Ralph Merkle patented the hash tree in 1979. Nearly fifty years later, it remains one of the most practically important ideas in computer science: a simple construction with profound consequences for how we build systems that operate without mutual trust.

The full Rust implementation discussed in this post, including the blockchain verification and file integrity examples, is available at [github.com/amoilanen/merkle-tree](https://github.com/amoilanen/merkle-tree).
