# merklix

Optimized [Merklix Tree][1] for node.js.

## Usage

``` js
const crypto = require('bcrypto');
const {Merklix} = require('merklix');
const {SHA256, randomBytes} = crypto;

const tree = new Merklix(SHA256, 160, '/path/to/my/db');

await tree.open();

let last;

for (let i = 0; i < 500; i++) {
  const key = randomBytes(20);
  const value = randomBytes(300);
  await tree.insert(key, value);
  last = key;
}

await tree.commit();

const root = tree.rootHash();
const proof = await tree.prove(root, last);
const [code, value] = tree.verify(root, last, proof);

if (code === 0 && value)
  console.log('Valid proof for: %s', value.toString('hex'));

await tree.values((key, value) => {
  console.log('Iterated over item:');
  console.log([key.toString('hex'), value.toString('hex')]);
});

await tree.close();
```

## Description

Much like a patricia trie or a sparse merkle tree, the merklix tree follows a
path down each key in order to find the target leaf node.

### Insertion

We start with a simple insertion of leaf `1` (0000).  It becomes the root (`R`)
of the merkle tree.

```
       R
```

Say we insert leaf `2` (1100). The tree grows down and we are now 1 level deep.
Note that we only went right once even though we have 3 extra bits in the key.

```
       R
      / \
     /   \
    1     2
```

This next part is important to how the merklix tree handles key collisions. Say
we insert leaf `3` (1101). You'll notice it has a three bit collision with leaf
`2` (1100). In order to maintain a proper key path within the tree, we grow
down and add "null" nodes (represented by `x`) as children of internal nodes.
This is basically a sign that there's a dead end in one of the internal nodes.
This is the trick to keeping the merklix tree small and ever-growing, unlike a
sparse merkle tree for example.

```
       R
      / \
     /   \
    1    /\
        /  \
       x   /\
          /  \
         /\   x
        /  \
       2    3
```

If we add leaf 4 (1000), it is free to consume one of the "null" nodes.

```
       R
      / \
     /   \
    1    /\
        /  \
       4   /\
          /  \
         /\   x
        /  \
       2    3
```

### Proofs

The proof is a standard merkle proof, with some extra gotchas.  The actual hash
at a leaf is the computed as `HASH(key | value)`. It is important to have the
full key as part of the preimage. If a non-existence proof is necessary, we
need to send the full preimage to prove that we are a leaf, and that we're also
a different key that may have a colliding path with whatever key a peer is
trying to get a proof for. On the other hand, if the key path stops at one of
the "dead-end" nodes, we do not have to send any preimage! Even better, if
there are any "dead-end" nodes up the subtree when creating a proof, we can
compress them since they are redundant zero-hashes.

With 50,000,000 leaves in the tree, the average depth of any given key path
down the tree should be around 26 or 27 (due the inevitable key prefix
collisions). This results in a proof size slightly over 800 bytes, pushing a
1-2ms proof creation time.

### Removal

Removal is tricky when we have "dead-end" nodes in our subtree. We need to
revert all of the subtree growing we just did.

If we were to remove leaf 4 from the above tree, we _must_ replace it with a
"dead-end". The general rule is: if the target node's sibling is an internal
node, replace with a null node. If the sibling is another leaf, attempt to
ungrow the subtree by detecting key collisions.

Removing leaf 4 (we _must_ replace with a dead-end):

```
       R
      / \
     /   \
    1    /\
        /  \
       x   /\
          /  \
         /\   x
        /  \
       2    3
```

Removing leaf 3 (ungrow the subtree):

```
       R
      / \
     /   \
    1     2
```

And we're back to where we started.

### Optimizing a merklix tree on disk

Due to the sheer number of nodes, a flat-file store is necessary. The amount of
database lookups would be overwhelming for something like leveldb. A merklix
tree is much simpler than a patricia trie in that we need only store 2 nodes:
internal nodes and leaves.

Internal nodes are stored as:

``` c
struct {
  uint8_t left_hash[32];
  uint16_t left_file;
  uint32_t left_position;
  uint8_t right_hash[32];
  uint16_t right_file;
  uint32_t right_position;
}
```

Leaf nodes are stored as:

``` c
struct {
  uint8_t leaf_hash[32];
  uint8_t key[20];
  uint16_t value_file;
  uint32_t value_position;
  uint32_t value_size;
}
```

The actual leaf data is stored at `value_position` in `value_file`.

This module will store the tree in a series of append-only files. A
particularly large write buffer is used to batch all insertions. Atomicity with
a parent database can be achieved by fsyncing every write and inserting the
best root hash and file position into something like leveldb (once the fsync
has completed).

### Collision Attacks

It is possible for someone to grind a key to create bit collisions. Currently,
the entire bitcoin network produces 72-80 bit collisions on block hashes. So
worst case, that's 72-80 levels deep, but while storage increases, the rounds
of hashing are still half (or less than half) of that of a sparse merkle tree.

## Contribution and License Agreement

If you contribute code to this project, you are implicitly allowing your code
to be distributed under the MIT license. You are also implicitly verifying that
all code is your original work. `</legalese>`

## License

- Copyright (c) 2018, Christopher Jeffrey (MIT License).

See LICENSE for more info.

[1]: https://www.deadalnix.me/2016/09/24/introducing-merklix-tree-as-an-unordered-merkle-tree-on-steroid/