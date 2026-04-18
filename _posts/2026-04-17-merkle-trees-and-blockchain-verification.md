---
layout: default
title: "Merkle Trees and Blockchain Verification"
date: 2026-04-17
---

# Merkle Trees and Blockchain Verification

Imagine you are running a Bitcoin wallet on your phone. The full Bitcoin blockchain is over 500 GB and growing. You certainly cannot store it all on a mobile device. Yet somehow your wallet can confirm that a payment you received is legitimate, included in a real block, without downloading that block's worth of transactions. How?

The answer lies in a data structure invented by Ralph Merkle in 1979 and described in his 1987 paper *"A Digital Signature Based on a Conventional Encryption Function"*. Merkle trees make it possible to verify membership in a dataset using only a logarithmic number of hash values. This property is so useful that Merkle trees have become one of the foundational primitives in distributed systems, from blockchains to file distribution networks to database replication protocols.

In this post we will look at how Merkle trees work, why they are secure, and how they enable lightweight blockchain clients. Along the way we will examine a [Rust implementation](https://github.com/amoilanen/merkle-tree) in detail: the core data structures, the tree construction algorithm, proof generation, and proof verification. Only after we have a solid understanding of the data structure itself will we dive into the blockchain application.

## Contents

- [What is a Merkle tree?](#what-is-a-merkle-tree)
- [The Hash type and helper functions](#the-hash-type-and-helper-functions)
- [Domain separation: a subtle but important detail](#domain-separation-a-subtle-but-important-detail)
- [Building the tree: bottom-up construction](#building-the-tree-bottom-up-construction)
- [Handling odd numbers of leaves](#handling-odd-numbers-of-leaves)
- [Inclusion proofs](#inclusion-proofs)
- [Generating proofs: walking from leaf to root](#generating-proofs-walking-from-leaf-to-root)
- [Verifying proofs: a fold over the path](#verifying-proofs-a-fold-over-the-path)
- [Blockchain verification with SPV](#blockchain-verification-with-spv)
- [Beyond blockchains](#beyond-blockchains)
- [Summary](#summary)

## What is a Merkle tree?

A Merkle tree is a binary tree where every leaf node contains the hash of a data block, and every internal node contains the hash of its two children concatenated together. The single hash at the top, the **root hash**, acts as a fingerprint for the entire dataset. Change a single byte in any leaf, and the root hash changes.

Here is a Merkle tree built from four transactions:

![Merkle tree with four transactions]({{ site.baseurl }}/assets/images/merkle-trees/merkle-tree-structure.svg)

Each leaf hash (`H0` through `H3`) is computed from the raw transaction data using a cryptographic hash function (SHA-256 in our implementation). The internal nodes `H01` and `H23` are computed by hashing the concatenation of their children. The root is computed from those two intermediate hashes.

This construction has a key property: given the root hash and a small number of intermediate hashes, anyone can verify that a specific leaf belongs to the tree. We will see exactly how that works shortly. But first, let's look at the Rust implementation to understand how these ideas translate into concrete code.

## The Hash type and helper functions

Before we can build a Merkle tree we need a type to represent SHA-256 digests and a function to hash arbitrary data. The implementation starts with a `Hash` wrapper around a 32-byte array:

```rust
/// A SHA-256 digest (32 bytes).
///
/// This is the fundamental unit of data throughout the tree. It wraps a
/// fixed-size byte array, avoiding heap allocations and enabling `Copy`.
#[derive(Clone, Copy, PartialEq, Eq, Hash)]
pub struct Hash([u8; 32]);

impl Hash {
    /// Returns a reference to the underlying 32-byte array.
    pub fn as_bytes(&self) -> &[u8; 32] {
        &self.0
    }
}
```

Why wrap a plain `[u8; 32]` in a newtype? There are a few reasons. First, it makes the API self-documenting: a function that takes a `Hash` clearly expects a SHA-256 digest, not any 32-byte value. Second, we can implement `Display` to get nice hex formatting and `Debug` with a truncated representation for readable debug output:

```rust
impl fmt::Display for Hash {
    /// Full lowercase hex string (64 characters).
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        for byte in &self.0 {
            write!(f, "{byte:02x}")?;
        }
        Ok(())
    }
}

impl fmt::Debug for Hash {
    /// Short hex representation (first 4 bytes / 8 hex chars) for readable debug output.
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Hash({}…)", &format!("{self}")[..8])
    }
}
```

These trait implementations may seem like minor polish, but they make the library much more pleasant to work with. When you print a proof or debug a tree, you see meaningful hex strings instead of arrays of raw bytes.

For plain hashing of arbitrary data (outside the tree) there is a simple `hash` function:

```rust
/// Hash arbitrary data with SHA-256.
pub fn hash(data: &[u8]) -> Hash {
    let mut hasher = Sha256::new();
    hasher.update(data);
    Hash(hasher.finalize().into())
}
```

This is a straightforward wrapper around the `sha2` crate. We will use it in our examples when we need to produce hashes that are *not* tree leaves (for example, when simulating tampered data).

## Domain separation: a subtle but important detail

There is a security concern that is easy to overlook. Consider what happens if leaf data and internal node data are hashed the same way. An attacker could potentially craft leaf data that, when hashed, produces the same bytes as a legitimate internal node hash. This is known as a **second-preimage attack** on the tree structure.

The solution, standardized in [RFC 6962](https://www.rfc-editor.org/rfc/rfc6962) (Certificate Transparency), is **domain separation**: prefix leaf hashes with a `0x00` byte and internal node hashes with a `0x01` byte. This ensures that a leaf hash and an internal node hash can never collide, even if the underlying data happens to be identical.

![Domain separation: leaf vs internal node hashing]({{ site.baseurl }}/assets/images/merkle-trees/domain-separation.svg)

In Rust this looks quite straightforward. Here are the two hashing functions that the tree uses internally:

```rust
/// Domain-separated leaf hash: SHA-256(0x00 || data).
///
/// The 0x00 prefix prevents second-preimage attacks where an attacker
/// crafts leaf data that collides with an internal node hash.
///
/// Reference: Certificate Transparency (RFC 6962, §2.1) uses domain
/// separation for exactly this reason.
pub fn hash_leaf(data: &[u8]) -> Hash {
    let mut hasher = Sha256::new();
    hasher.update([0x00]);
    hasher.update(data);
    Hash(hasher.finalize().into())
}

/// Domain-separated internal node hash: SHA-256(0x01 || left || right).
///
/// The 0x01 prefix ensures internal nodes live in a different hash domain
/// than leaves, preventing second-preimage attacks.
fn hash_pair(left: &Hash, right: &Hash) -> Hash {
    let mut hasher = Sha256::new();
    hasher.update([0x01]);
    hasher.update(left.0);
    hasher.update(right.0);
    Hash(hasher.finalize().into())
}
```

Notice that `hash_leaf` is public — consumers of the library need it to compute leaf hashes for proof lookup (for example, to verify that downloaded content matches a proof's leaf hash). In contrast, `hash_pair` is private: it is only used internally during tree construction and proof verification. There is no reason for external code to pair arbitrary hashes.

A small prefix, but it closes a real attack vector. It is a good reminder that in cryptographic constructions, the details that seem trivial are often the ones that matter most.

We can verify the domain separation works by checking that hashing the same data as a leaf and as a pair of internal nodes produces different results:

```rust
#[test]
fn leaf_and_internal_hashes_differ_due_to_domain_prefix() {
    let data = hash(b"x");
    let as_leaf = hash_leaf(&data.as_bytes().repeat(2));
    let as_pair = hash_pair(&data, &data);
    assert_ne!(as_leaf, as_pair);
}
```

Even though the raw bytes being hashed are the same (two copies of `hash(b"x")`), the different prefix byte means the two functions produce completely different digests.

## Building the tree: bottom-up construction

With the hash functions defined, let's look at how the tree itself is built. The `MerkleTree` struct stores all levels of the tree from root (index 0) down to leaves (last index):

```rust
/// A SHA-256 Merkle tree built from leaf data.
///
/// Internally stores every level of hashes from root (index 0) down to
/// the leaves (last index). This enables proof generation by walking
/// from the leaf level upward.
#[derive(Clone, Debug)]
pub struct MerkleTree {
    /// Tree levels, root-first. `levels[0]` contains the single root hash
    /// (or is empty for an empty tree), and `levels[len-1]` contains the
    /// leaf hashes.
    levels: Vec<Vec<Hash>>,
    /// Number of original leaves (before odd-leaf duplication).
    leaf_count: usize,
}
```

Why store all levels? It is a deliberate trade-off. Storing only the root would be sufficient if we only ever needed to verify proofs, but we also need to *generate* proofs. Proof generation requires walking from a leaf upward through the tree, collecting the sibling hash at each level. Having all levels in memory makes this straightforward. The space overhead is acceptable: for *n* leaves, the total number of hashes across all levels is roughly *2n* (a geometric series that converges to 2n), so we use about twice the space of just the leaves. For a tree with a million SHA-256 leaves, that is about 64 MB — entirely reasonable.

The `build` method constructs the tree bottom-up:

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
        // Duplicate the last node when the level has an odd count, matching
        // Bitcoin's Merkle tree construction which duplicates the trailing
        // hash so every node has a pair.
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

Let's trace through this for four leaves `["tx0", "tx1", "tx2", "tx3"]` to understand exactly what happens at each stage:

![Bottom-up construction of a Merkle tree]({{ site.baseurl }}/assets/images/merkle-trees/build-algorithm.svg)

**Step 1.** Each leaf is hashed with domain separation: `hash_leaf(b"tx0")` produces `H0`, `hash_leaf(b"tx1")` produces `H1`, and so on. This gives us the initial `current_level = [H0, H1, H2, H3]`.

**Step 2.** The level has 4 elements (even), so no duplication is needed. We push this level onto `levels`, then produce the next level by pairing adjacent hashes: `H01 = hash_pair(H0, H1)` and `H23 = hash_pair(H2, H3)`. Now `current_level = [H01, H23]`.

**Step 3.** The level has 2 elements (even again). We push it onto `levels`, then pair: `Root = hash_pair(H01, H23)`. Now `current_level = [Root]`.

**Step 4.** `current_level.len() == 1`, so the `while` loop exits. We push the root onto `levels` and reverse the whole vector so that `levels[0]` contains the root and `levels[last]` contains the leaves.

The resulting `levels` array looks like:

```
levels[0] = [Root]                 // 1 hash
levels[1] = [H01, H23]            // 2 hashes
levels[2] = [H0, H1, H2, H3]     // 4 hashes
```

A few implementation details worth noting. The generic type parameter `T: AsRef<[u8]>` means `build` accepts any type that can be viewed as a byte slice: `&str`, `String`, `Vec<u8>`, `&[u8]`, etc. This makes the API flexible without sacrificing performance. The `chunks_exact(2)` method from the standard library gives us an iterator over pairs, which maps cleanly onto the pairing operation.

The time complexity is **O(n)**: we hash each leaf once, and each subsequent level has half as many nodes, giving us n + n/2 + n/4 + … ≈ 2n hash operations total.

Accessor methods provide read access to the tree's contents:

```rust
/// Returns the root hash, or `None` if the tree was built from empty input.
pub fn get_root_hash(&self) -> Option<&Hash> {
    self.levels.first().and_then(|level| level.first())
}

/// Returns the domain-separated hash of the leaf at the given index,
/// or `None` if the index is out of bounds.
pub fn get_leaf_hash(&self, index: usize) -> Option<&Hash> {
    if index >= self.leaf_count {
        return None;
    }
    self.levels.last().and_then(|level| level.get(index))
}
```

Note the bounds check in `get_leaf_hash`: it uses `self.leaf_count` rather than the leaf level's length. This is important because the leaf level may contain duplicated entries (when the original number of leaves was odd). We do not want to expose those duplicates to callers — they are an internal implementation detail.

## Handling odd numbers of leaves

What happens when the number of leaves is not a power of two? The standard approach, used in Bitcoin, is to duplicate the last leaf so that every level has an even number of nodes. For example, a tree with three transactions `[tx0, tx1, tx2]` is built as if it had four: `[tx0, tx1, tx2, tx2]`.

![Handling odd number of leaves by duplicating the last one]({{ site.baseurl }}/assets/images/merkle-trees/odd-leaves.svg)

This duplication happens at every level, not just the leaf level. Consider a tree with five leaves: `[tx0, tx1, tx2, tx3, tx4]`. At the leaf level, `tx4` is duplicated to make six. Those six hashes produce three pairs at the next level, and then the last one is duplicated again to make four. This continues until we reach the root.

The test suite verifies this behavior:

```rust
#[test]
fn odd_leaf_count_duplicates_last() -> anyhow::Result<()> {
    // 3 leaves → [a, b, c, c_dup] at leaf level
    let tree = MerkleTree::build(&["a", "b", "c"]);
    assert_eq!(tree.levels.len(), 3);
    // Leaf level has 4 entries (3 + 1 duplicate)
    assert_eq!(tree.levels[2].len(), 4);
    assert_eq!(
        tree.levels[2][2], tree.levels[2][3],
        "last leaf must be duplicated"
    );
    Ok(())
}

#[test]
fn five_leaves_duplicate_at_each_odd_level() {
    let tree = MerkleTree::build(&["a", "b", "c", "d", "e"]);
    // 5 leaves → 6 (dup) → 3 pairs → 4 (dup) → 2 → 1
    assert_eq!(tree.levels.len(), 4);
    assert_eq!(tree.levels[0].len(), 1);
    assert_eq!(tree.levels[3].len(), 6);
}
```

The first test confirms that a 3-leaf tree produces a leaf level of length 4, with the last two entries being identical (the duplicated leaf). The second test shows the cascading duplication for 5 leaves: the leaf level has 6 entries (5 + 1 duplicate), which produces 3 pairs, which are padded to 4, producing 2, producing 1 (the root). Four levels total.

Proofs still work correctly for leaves in odd-sized trees, including the duplicated leaf itself:

```rust
#[test]
fn odd_leaf_duplication_does_not_break_proof() -> anyhow::Result<()> {
    // In a 3-leaf tree [a, b, c], "c" is duplicated to make [a, b, c, c].
    // Proof for "c" should still verify correctly.
    let tree = MerkleTree::build(&["a", "b", "c"]);
    let root = tree.get_root_hash().context("missing root")?;
    let leaf_c = hash_leaf(b"c");
    let proof = tree.generate_proof(&leaf_c).context("proof not found")?;
    assert!(proof.verify(root));
    Ok(())
}
```

This keeps the construction simple at the cost of a minor space overhead: at most one extra hash per level, which is negligible compared to the level's size.

## Inclusion proofs

The most powerful feature of a Merkle tree is the **inclusion proof** (also called a Merkle proof or authentication path). Given a leaf, we can prove it belongs to the tree by providing only the sibling hashes along the path from that leaf to the root. The verifier recomputes the root by hashing upward and checks whether it matches the expected root hash.

Before looking at the proof generation algorithm, let's define the data structures. A proof consists of a leaf hash and a sequence of steps, where each step records a sibling hash and its position:

```rust
/// Whether a sibling is to the left or right of the path node in the tree.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Direction {
    Left,
    Right,
}

/// A single step in a Merkle inclusion proof: a sibling hash and its position.
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct ProofStep {
    /// The sibling hash at this tree level.
    pub hash: Hash,
    /// The position of the sibling relative to the path node.
    pub direction: Direction,
}

/// A Merkle inclusion proof: a leaf hash plus the sibling hashes needed to
/// recompute the root.
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Proof {
    /// The leaf hash this proof is for.
    pub leaf: Hash,
    /// Sibling hashes from leaf level up to (but not including) the root.
    pub steps: Vec<ProofStep>,
}
```

Why record the direction? Because `hash_pair(A, B)` produces a different result from `hash_pair(B, A)`. When the verifier recomputes the root, it must place the sibling and the current hash in the correct order. If the sibling is on the left, it goes first; if on the right, it goes second.

Let's trace through a concrete example. Suppose we want to prove that `tx2` is included in our four-transaction tree. The proof consists of the sibling hashes along the path from `tx2` to the root:

![Inclusion proof for tx2 showing the verification path]({{ site.baseurl }}/assets/images/merkle-trees/inclusion-proof.svg)

The proof for `tx2` contains two hashes: `H3` (the sibling at the leaf level, on the right) and `H01` (the sibling at the next level up, on the left). The verification process is:

1. Start with the leaf hash: `H2 = hash_leaf(tx2)`
2. Combine with sibling `H3` (which is on the right): `H23 = hash_pair(H2, H3)`
3. Combine with sibling `H01` (which is on the left): `Root' = hash_pair(H01, H23)`
4. Check: does `Root'` equal the expected root? If yes, `tx2` is in the tree.

The proof size is **O(log n)**: for a tree with *n* leaves, you need only *⌈log₂(n)⌉* sibling hashes. A tree with a million leaves requires just 20 hashes in the proof — roughly 640 bytes with SHA-256. This is remarkably compact.

The test suite verifies this logarithmic relationship:

```rust
#[test]
fn ten_thousand_leaves_proof_depth_is_log2_n() -> anyhow::Result<()> {
    let n = 10000;
    let leaves: Vec<String> = (0..n).map(|i| format!("leaf_{i}")).collect();
    let refs: Vec<&str> = leaves.iter().map(|s| s.as_str()).collect();
    let tree = MerkleTree::build(&refs);
    let root = tree.get_root_hash().context("missing root")?;

    let target = hash_leaf(b"leaf_500");
    let proof = tree.generate_proof(&target).context("proof not found")?;

    // Proof steps should be ⌈log₂(n)⌉ = 14 for n=10000
    let expected_depth = (n as f64).log2().ceil() as usize;
    assert_eq!(proof.steps.len(), expected_depth);
    assert!(proof.verify(root));
    Ok(())
}
```

For 10,000 leaves, the proof contains exactly ⌈log₂(10000)⌉ = 14 hashes. That is 14 × 32 = 448 bytes to prove membership in a dataset that could be megabytes or gigabytes in size.

## Generating proofs: walking from leaf to root

Now let's look at the proof generation algorithm. Given a leaf hash, we find it at the bottom level of the tree and walk upward, collecting the sibling hash at each level:

```rust
pub fn generate_proof(&self, leaf_hash: &Hash) -> Option<Proof> {
    let leaf_level = self.levels.last()?;
    let mut position = leaf_level.iter().position(|h| h == leaf_hash)?;

    let mut steps = Vec::with_capacity(self.levels.len().saturating_sub(1));

    // Walk from leaf level upward toward the root, collecting each sibling
    // hash. `levels[0]` is the root and `levels[last]` is the leaf level.
    // At each level we find the sibling of our current position, record it,
    // then halve the position to move to the parent level.
    for depth in (1..self.levels.len()).rev() {
        let level = &self.levels[depth];
        // Nodes are paired (0-1, 2-3, …): even positions pair with the next
        // node (right sibling), odd positions pair with the previous one
        // (left sibling).
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

This is the core of the proof generation, so let's walk through it carefully for our `tx2` example in a four-leaf tree.

The tree levels are:

```
levels[0] = [Root]              // root level
levels[1] = [H01, H23]         // internal level
levels[2] = [H0, H1, H2, H3]  // leaf level
```

We search for `H2` (the hash of `tx2`) in the leaf level (`levels[2]`) and find it at `position = 2`.

**Iteration 1** (`depth = 2`, the leaf level). Position 2 is even, so the sibling is at position 3 (the next node): that is `H3`. The direction is `Right` (the sibling is to the right of our node). We record `ProofStep { hash: H3, direction: Right }`. Then we halve the position: `position = 2 / 2 = 1`.

**Iteration 2** (`depth = 1`, the internal level). Position 1 is odd, so the sibling is at position 0 (the previous node): that is `H01`. The direction is `Left` (the sibling is to the left of our node). We record `ProofStep { hash: H01, direction: Left }`. Then we halve: `position = 1 / 2 = 0`.

The loop does not enter for `depth = 0` because the range `(1..self.levels.len()).rev()` starts at 2 and ends at 1.

The result is a proof with two steps: `[{H3, Right}, {H01, Left}]`. This matches exactly what we drew in the diagram above.

The position halving (`position /= 2`) is the key insight: it reflects the tree's structure. At the leaf level, nodes 0 and 1 share parent 0, nodes 2 and 3 share parent 1. At the next level, nodes 0 and 1 share parent 0. Dividing the position by 2 maps a node to its parent's position in the level above.

Note that the method returns `None` if the leaf hash is not found in the tree. This is the correct behavior: you cannot generate a proof for data that is not in the tree.

A special case worth mentioning: a single-leaf tree. If there is only one leaf, the tree has a single level `levels[0] = [H0]`, and `generate_proof` produces a proof with zero steps. The root *is* the leaf hash, so no sibling hashes are needed:

```rust
#[test]
fn single_leaf_proof_has_no_steps_and_verifies() -> anyhow::Result<()> {
    let tree = MerkleTree::build(&["only"]);
    let root = tree.get_root_hash().context("missing root")?;
    let leaf = hash_leaf(b"only");
    let proof = tree.generate_proof(&leaf).context("proof not found")?;
    assert!(proof.steps.is_empty(), "single-leaf proof needs no steps");
    assert!(proof.verify(root));
    assert_eq!(proof.compute_root(), *root);
    Ok(())
}
```

## Verifying proofs: a fold over the path

Verification is the elegant counterpart to generation. Starting from the leaf hash, we combine it with each sibling in the proof — respecting the recorded direction — until we reach the root:

```rust
impl Proof {
    /// Recompute the root hash from this proof's leaf and steps.
    ///
    /// Starting from the leaf hash, each step provides a sibling hash and
    /// its direction. The pair is combined with domain-separated hashing
    /// (preserving left/right order) to produce the parent hash, until the
    /// root is reached.
    pub fn compute_root(&self) -> Hash {
        self.steps.iter().fold(self.leaf, |current, step| {
            match step.direction {
                // Sibling is on the left → sibling comes first.
                Direction::Left => hash_pair(&step.hash, &current),
                // Sibling is on the right → current comes first.
                Direction::Right => hash_pair(&current, &step.hash),
            }
        })
    }

    /// Verify that this proof's leaf belongs to a tree with the given root.
    pub fn verify(&self, expected_root: &Hash) -> bool {
        self.compute_root() == *expected_root
    }
}
```

This is the entire verification logic. It is elegant in its simplicity: a single `fold` over the proof steps, a hash comparison at the end. No tree reconstruction, no complex state. The verifier does not even need to know the size of the original tree.

Let's trace through verification of our `tx2` proof:

1. Start: `current = H2` (the leaf hash)
2. Step 1: sibling `H3` is on the Right → `current = hash_pair(H2, H3) = H23`
3. Step 2: sibling `H01` is on the Left → `current = hash_pair(H01, H23) = Root'`
4. Compare: `Root' == Root`? If yes, the proof is valid.

The `fold` operation maps perfectly onto the mathematical definition: we are literally "folding" the path from leaf to root, accumulating hash values as we go. Rust's type system and the `fold` combinator make this both concise and correct.

The test suite exhaustively verifies that every leaf in a tree produces a valid proof:

```rust
#[test]
fn every_leaf_proof_verifies_against_root() -> anyhow::Result<()> {
    let items = &["a", "b", "c", "d"];
    let tree = MerkleTree::build(items);
    let root = tree.get_root_hash().context("missing root")?;

    for item in items {
        let leaf = hash_leaf(item.as_bytes());
        let proof = tree.generate_proof(&leaf).context("proof not found")?;
        assert_eq!(proof.leaf, leaf);
        assert!(proof.verify(root), "proof failed for leaf '{item}'");
    }
    Ok(())
}
```

And it also verifies the tamper-detection properties. Changing *any* part of the proof — a sibling hash, the leaf hash, or even the direction of a sibling — causes verification to fail:

```rust
#[test]
fn tampered_sibling_hash_invalidates_proof() -> anyhow::Result<()> {
    let tree = MerkleTree::build(&["a", "b", "c", "d"]);
    let root = tree.get_root_hash().context("missing root")?;
    let leaf = hash_leaf(b"a");
    let mut proof = tree.generate_proof(&leaf).context("proof not found")?;
    proof.steps[0].hash = hash(b"tampered");
    assert!(!proof.verify(root));
    Ok(())
}

#[test]
fn flipped_sibling_direction_invalidates_proof() -> anyhow::Result<()> {
    let tree = MerkleTree::build(&["a", "b", "c", "d"]);
    let root = tree.get_root_hash().context("missing root")?;
    let leaf = hash_leaf(b"a");
    let mut proof = tree.generate_proof(&leaf).context("proof not found")?;
    proof.steps[0].direction = match proof.steps[0].direction {
        Direction::Left => Direction::Right,
        Direction::Right => Direction::Left,
    };
    assert!(!proof.verify(root));
    Ok(())
}
```

The direction test is particularly interesting. It confirms that `hash_pair` is not commutative: swapping the order of arguments produces a completely different hash. This is by design — it prevents an attacker from rearranging siblings in the proof to forge a valid path.

## Blockchain verification with SPV

Now that we understand how Merkle trees work at a fundamental level — the data structures, the construction, proof generation, and verification — we can see why they are so valuable in blockchain systems. The logarithmic proof size is exactly what makes **Simplified Payment Verification (SPV)** possible. SPV was described in Section 7 of the Bitcoin whitepaper and is the reason lightweight wallets can function without storing the entire blockchain.

![SPV architecture: full node vs light node]({{ site.baseurl }}/assets/images/merkle-trees/spv-architecture.svg)

A **full node** stores every block with all its transactions and the corresponding Merkle tree. A **light node** stores only block headers, which include the Merkle root of each block's transaction tree. When the light node wants to verify a transaction, it asks a full node for a Merkle proof, then verifies it locally against the stored root.

The trust model is important here: the light node does not need to trust the full node. If the full node provides a fraudulent proof, it will not match the Merkle root that the light node already has from the block header chain. The proof is either mathematically correct or it is not.

Let's look at how this works in practice using a Rust implementation. First, we need a transaction type with deterministic serialisation:

```rust
/// A simplified blockchain transaction.
///
/// The `nonce` here is a per-sender sequence number (as in Ethereum's
/// account-based model) that prevents replay attacks and orders transactions
/// from the same sender.
struct Transaction {
    from: String,
    to: String,
    value: u64,
    nonce: u64,
}

impl Transaction {
    /// Deterministic binary serialisation used as Merkle tree leaf data.
    ///
    /// A real implementation would use RLP, SSZ, or another canonical encoding;
    /// here we concatenate fields with a delimiter that cannot appear in the
    /// address/nonce representation to keep things simple.
    fn to_bytes(&self) -> Vec<u8> {
        let mut buf = Vec::new();
        buf.extend_from_slice(self.from.as_bytes());
        buf.push(b'|');
        buf.extend_from_slice(self.to.as_bytes());
        buf.push(b'|');
        buf.extend_from_slice(self.value.to_le_bytes().as_slice());
        buf.push(b'|');
        buf.extend_from_slice(self.nonce.to_le_bytes().as_slice());
        buf
    }
}
```

The serialisation uses a simple delimiter-based format. In a real blockchain implementation you would use a canonical encoding like [RLP](https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/) (Ethereum) or a length-prefixed binary format to avoid ambiguity. The key requirement is determinism: the same transaction must always produce the same bytes, so the same leaf hash.

Now we define the participants: blocks, full nodes, and light nodes:

```rust
/// A block on the full node: contains the complete transaction list
/// and its Merkle tree.
struct Block {
    id: String,
    transactions: Vec<Transaction>,
    tree: MerkleTree,
}

impl Block {
    fn new(id: impl Into<String>, transactions: Vec<Transaction>) -> Self {
        let id = id.into();
        let leaves: Vec<Vec<u8>> = transactions.iter().map(|tx| tx.to_bytes()).collect();
        let tree = MerkleTree::build(&leaves);
        Self { id, transactions, tree }
    }
}

/// A full node stores complete blocks and can generate inclusion proofs.
struct FullNode {
    blocks: Vec<Block>,
}

impl FullNode {
    /// Generate a Merkle proof for `transaction_index` within `block_id`.
    fn proof_for(&self, block_id: &str, transaction_index: usize) -> Option<Proof> {
        let block = self.blocks.iter().find(|b| b.id == block_id)?;
        let leaf = block.tree.get_leaf_hash(transaction_index)?;
        block.tree.generate_proof(leaf)
    }
}
```

The `Block::new` constructor builds the Merkle tree at block creation time. Each transaction is serialised to bytes, and those byte slices become the leaves. The full node can then generate proofs for any transaction by index.

The light node stores only block headers — the minimal metadata needed for verification:

```rust
/// The minimal per-block metadata a light client needs: the block
/// identifier and the Merkle root of its transaction tree.
///
/// In a real blockchain (e.g. Bitcoin, Ethereum) a block header carries
/// additional fields (previous-block hash, timestamp, nonce, …); here we
/// keep only what is required for SPV-style verification.
struct BlockHeader {
    id: String,
    merkle_root: merkle_tree::Hash,
}

/// A light node stores only block headers — no full transaction lists.
struct LightNode {
    headers: Vec<BlockHeader>,
}

impl LightNode {
    fn from_full_node(full: &FullNode) -> Result<Self> {
        let headers = full
            .blocks
            .iter()
            .map(|b| {
                let merkle_root = *b
                    .tree
                    .get_root_hash()
                    .with_context(|| format!("block {} has no transactions", b.id))?;
                Ok(BlockHeader {
                    id: b.id.clone(),
                    merkle_root,
                })
            })
            .collect::<Result<Vec<_>>>()?;
        Ok(Self { headers })
    }

    /// Verify a transaction proof against the stored root for `block_id`.
    fn verify(&self, block_id: &str, proof: &Proof) -> bool {
        self.headers
            .iter()
            .find(|h| h.id == block_id)
            .is_some_and(|h| proof.verify(&h.merkle_root))
    }
}
```

Notice how thin the light node's `verify` method is: look up the block header, call `proof.verify()` against its Merkle root. All the cryptographic heavy lifting happens inside the proof's `compute_root` method, which we already examined in detail.

Now let's set up a scenario with two blocks of transactions and walk through three security scenarios:

```rust
fn main() -> Result<()> {
    let block_a = Block::new(
        "block-1",
        vec![
            Transaction { from: "0xAlice".into(), to: "0xBob".into(),   value: 50, nonce: 1 },
            Transaction { from: "0xBob".into(),   to: "0xCarol".into(), value: 30, nonce: 1 },
            Transaction { from: "0xCarol".into(), to: "0xDave".into(),  value: 10, nonce: 1 },
            Transaction { from: "0xDave".into(),  to: "0xAlice".into(), value:  5, nonce: 1 },
            Transaction { from: "0xAlice".into(), to: "0xDave".into(),  value: 20, nonce: 2 },
        ],
    );
    let block_b = Block::new(
        "block-2",
        vec![
            Transaction { from: "0xEve".into(),   to: "0xAlice".into(), value: 100, nonce: 1 },
            Transaction { from: "0xAlice".into(), to: "0xBob".into(),   value:  75, nonce: 3 },
        ],
    );

    let full_node = FullNode {
        blocks: vec![block_a, block_b],
    };
    let light_node = LightNode::from_full_node(&full_node)?;
    // ...
}
```

### Scenario 1: Successful verification

A client wants to confirm that the third transaction in block-1 (Carol → Dave, 10 units) was really included. The full node provides a Merkle proof; the light node verifies it:

```rust
let proof = full_node
    .proof_for("block-1", 2)
    .expect("transaction exists in block");

let verified = light_node.verify("block-1", &proof);
// verified == true
```

For a block with 5 transactions, the proof contains just 3 hashes (⌈log₂(5)⌉ = 3), regardless of how many transactions the block contains. In Bitcoin, which can have several thousand transactions per block, a proof is still only around 10-12 hashes — about 320-384 bytes.

### Scenario 2: Tamper detection

An attacker intercepts the proof and modifies one of the sibling hashes, hoping to trick the light node into accepting a fraudulent transaction:

```rust
let mut tampered_proof = proof.clone();
tampered_proof.steps[0].hash = merkle_tree::hash(b"tampered-data");

let tampered_verified = light_node.verify("block-1", &tampered_proof);
// tampered_verified == false
```

The tampered proof computes a completely different root hash. Even a single bit change in any proof step cascades through the hash chain and produces a root that does not match. This is the **avalanche property** of cryptographic hash functions working in our favour: a small change to the input produces a completely unpredictable change in the output.

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

The pattern is the same one we saw with blockchains. A server holds the full file set and its Merkle tree. A client needs only the trusted root hash. For each file it downloads, it receives a compact proof. Here is a simplified version from the implementation's `file_integrity` example:

```rust
/// A release: all files and the Merkle tree built from their contents.
struct Release {
    files: Vec<ReleaseFile>,
    tree: MerkleTree,
}

impl Release {
    fn new(files: Vec<ReleaseFile>) -> Self {
        let leaves: Vec<&[u8]> = files.iter().map(|f| f.content.as_slice()).collect();
        let tree = MerkleTree::build(&leaves);
        Self { files, tree }
    }

    fn manifest_root(&self) -> Result<merkle_tree::Hash> {
        self.tree
            .get_root_hash()
            .copied()
            .context("release has no files")
    }
}

/// The client: knows only the trusted manifest root.
struct Client {
    trusted_root: merkle_tree::Hash,
}

impl Client {
    /// Verify that `data` is an authentic file from the release.
    ///
    /// Two checks are needed:
    /// 1. hash_leaf(data) == proof.leaf — the downloaded bytes match the
    ///    hash claimed by the proof.
    /// 2. proof.verify(&trusted_root) — the leaf hash chains up through
    ///    sibling hashes to the trusted root.
    fn verify(&self, data: &[u8], proof: &merkle_tree::Proof) -> bool {
        merkle_tree::hash_leaf(data) == proof.leaf && proof.verify(&self.trusted_root)
    }
}
```

Note the two-step verification in the client. The first check (`hash_leaf(data) == proof.leaf`) ensures the downloaded bytes match the hash claimed by the proof. The second check (`proof.verify(&trusted_root)`) ensures the leaf hash chains up to the trusted root. Both are necessary: without the first check, an attacker who controls the download channel could replace both the file and the proof's leaf hash, and verification would still pass. Without the second check, there would be no chain of trust back to the known-good root.

A constrained device that only needs two out of six files in a release can verify just those two, without downloading the rest. The proof size is the same regardless of how many files are in the release.

**Database anti-entropy.** Distributed databases like Apache Cassandra and Amazon DynamoDB use Merkle trees to detect inconsistencies between replicas. Each node builds a Merkle tree over its local data. To check whether two replicas are in sync, they compare root hashes. If the roots differ, they walk down the tree together, comparing hashes level by level, to identify exactly which data ranges are out of sync. The implementation's `database_sync` example demonstrates this pattern:

```rust
struct Replica {
    name: &'static str,
    records: Vec<Record>,
    tree: MerkleTree,
}

impl Replica {
    /// Quick check: do two replicas have identical data?
    fn is_in_sync_with(&self, other: &Replica) -> Result<bool> {
        Ok(self.root()? == other.root()?)
    }

    /// Walk the leaf level to find which record indices differ.
    fn find_divergent_keys(&self, other: &Replica) -> Result<Vec<usize>> {
        let common = self.records.len().min(other.records.len());
        let mut divergent = Vec::new();
        for i in 0..common {
            let hash_a = self.tree.get_leaf_hash(i).context("invalid leaf index")?;
            let hash_b = other.tree.get_leaf_hash(i).context("invalid leaf index")?;
            if hash_a != hash_b {
                divergent.push(i);
            }
        }
        for i in common..self.records.len().max(other.records.len()) {
            divergent.push(i);
        }
        Ok(divergent)
    }

    /// Repair by copying divergent records, then rebuilding the tree.
    fn repair_from(&mut self, source: &Replica, indices: &[usize]) {
        for &idx in indices {
            self.records[idx] = source.records[idx].clone();
        }
        let leaves: Vec<Vec<u8>> = self.records.iter().map(|r| r.to_bytes()).collect();
        self.tree = MerkleTree::build(&leaves);
    }
}
```

This is far more efficient than comparing every record: if two replicas with 8 records have only 2 divergent records, only those 2 need to be transferred. In a real system with millions of records, the savings are enormous — the traversal focuses only on the divergent branches, skipping entire subtrees whose roots match.

**Certificate Transparency.** Google's Certificate Transparency framework uses Merkle trees to create an append-only log of TLS certificates. Monitors can efficiently verify that a certificate was logged, and auditors can verify that the log is consistent (no entries were removed or altered). This is the same RFC 6962 that standardized domain separation.

## Summary

Merkle trees solve a fundamental problem in distributed systems: how to verify that a piece of data belongs to a larger dataset without possessing the entire dataset. The key insights are:

- **A single root hash summarizes arbitrarily large data.** Any change to any leaf propagates up and alters the root. This gives us tamper evidence.
- **Inclusion proofs are logarithmic.** Verifying one leaf out of a million requires only 20 hashes. This makes verification practical even for resource-constrained clients.
- **Verification is trustless.** The verifier needs only the root hash from a trusted source. Everything else — the proof data, the full node providing it — can be untrusted. The math either checks out or it does not.
- **Domain separation prevents structural attacks.** The simple technique of prefixing leaf and internal node hashes with different bytes prevents a class of attacks where crafted leaf data could masquerade as internal nodes.

These properties explain why Merkle trees appear everywhere: Bitcoin and Ethereum for transaction verification, IPFS for content addressing, Certificate Transparency for auditing TLS certificates, Cassandra and DynamoDB for replica synchronization, Git for content-addressable storage, and many more.

Ralph Merkle patented the hash tree in 1979. Nearly fifty years later, it remains one of the most practically important ideas in computer science: a simple construction with profound consequences for how we build systems that operate without mutual trust.

The full Rust implementation discussed in this post, including the blockchain verification, file integrity, and database sync examples, is available at [github.com/amoilanen/merkle-tree](https://github.com/amoilanen/merkle-tree).
