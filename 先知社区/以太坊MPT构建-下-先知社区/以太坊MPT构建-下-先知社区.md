

# 以太坊 MPT 构建 (下) - 先知社区

以太坊 MPT 构建 (下)

- - -

PS：本篇文章是以太坊 MPT 构建 (上) 的续篇

## 源码分析

### 树的提交

以太坊树的提交通过 Commit 函数来实现，具体实现代码如下所示：

```plain
// Commit writes all nodes to the trie's memory database, tracking the internal
// and external (for account tries) references.
func (t *Trie) Commit(onleaf LeafCallback) (root common.Hash, err error) {
    if t.db == nil {
        panic("commit called on trie with nil database")
    }
    if t.root == nil {
        return emptyRoot, nil
    }
    // Derive the hash for all dirty nodes first. We hold the assumption
    // in the following procedure that all nodes are hashed.
    rootHash := t.Hash()
    h := newCommitter()
    defer returnCommitterToPool(h)

    // Do a quick check if we really need to commit, before we spin
    // up goroutines. This can happen e.g. if we load a trie for reading storage
    // values, but don't write to it.
    if _, dirty := t.root.cache(); !dirty {
        return rootHash, nil
    }
    var wg sync.WaitGroup
    if onleaf != nil {
        h.onleaf = onleaf
        h.leafCh = make(chan *leaf, leafChanSize)
        wg.Add(1)
        go func() {
            defer wg.Done()
            h.commitLoop(t.db)
        }()
    }
    var newRoot hashNode
    newRoot, err = h.Commit(t.root, t.db)
    if onleaf != nil {
        // The leafch is created in newCommitter if there was an onleaf callback
        // provided. The commitLoop only _reads_ from it, and the commit
        // operation was the sole writer. Therefore, it's safe to close this
        // channel here.
        close(h.leafCh)
        wg.Wait()
    }
    if err != nil {
        return common.Hash{}, err
    }
    t.root = newRoot
    return rootHash, nil
}
```

在上述代码中首先检查了 Trie 的 DB 是否存在，之后检查 Trie 的 Root 根是否存在，之后调用 Trie 的 Hash 方法获取 Trie 的 root 根的值：

```plain
if t.db == nil {
        panic("commit called on trie with nil database")
    }
    if t.root == nil {
        return emptyRoot, nil
    }
    // Derive the hash for all dirty nodes first. We hold the assumption
    // in the following procedure that all nodes are hashed.
    rootHash := t.Hash()
```

Hash 方法如下所示，在这里继续调用了 hashRoot 方法：

```plain
// Hash returns the root hash of the trie. It does not write to the
// database and can be used even if the trie doesn't have one.
func (t *Trie) Hash() common.Hash {
    hash, cached, _ := t.hashRoot()
    t.root = cached
    return common.BytesToHash(hash.(hashNode))
}
```

hashRoot 方法主要用于计算 Root 的 Hash 值，具体实现如下，在这里首先检查 Trie 的 Root 是否为空，如果为空则直接返回空字节 (emptyRoot) 的 hashNode，否则继续检查更改数是否低于 100，如果低于则用一个线程来处理，之后调用 hash.hash 来处理 Trie，之后将 unhashed 更新为 0，之后返回处理后的 Trie Root 的 Hash 值：

```plain
// filedir:go-ethereum-1.10.2\trie\trie.go  L544
// hashRoot calculates the root hash of the given trie
func (t *Trie) hashRoot() (node, node, error) {
    if t.root == nil {
        return hashNode(emptyRoot.Bytes()), nil, nil
    }
    // If the number of changes is below 100, we let one thread handle it
    h := newHasher(t.unhashed >= 100)
    defer returnHasherToPool(h)
    hashed, cached := h.hash(t.root, true)
    t.unhashed = 0
    return hashed, cached, nil
}
// filedir:go-ethereum-1.10.2\trie\trie.go  L31
var (
    // emptyRoot is the known root hash of an empty trie.
    emptyRoot = common.HexToHash("56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421")

    // emptyState is the known hash of an empty state trie entry.
    emptyState = crypto.Keccak256Hash(nil)
)

// filedir:go-ethereum-1.10.2\trie\hasher.go  L56
func newHasher(parallel bool) *hasher {
    h := hasherPool.Get().(*hasher)
    h.parallel = parallel
    return h
}

// filedir:go-ethereum-1.10.2\trie\hasher.go    L62
func returnHasherToPool(h *hasher) {
    hasherPool.Put(h)
}
```

这里我们着重看一下 hash 的处理过程，在这里 hash 将节点向下折叠为 hash node，同时用计算出的散列初始化后的原始节点副本替换原始节点

```plain
// filedir:go-ethereum-1.10.2\trie\hasher.go  L66
// hash collapses a node down into a hash node, also returning a copy of the
// original node initialized with the computed hash to replace the original one.
func (h *hasher) hash(n node, force bool) (hashed node, cached node) {
    // Return the cached hash if it's available
    if hash, _ := n.cache(); hash != nil {
        return hash, n
    }
    // Trie not processed yet, walk the children
    switch n := n.(type) {
    case *shortNode:
        collapsed, cached := h.hashShortNodeChildren(n)
        hashed := h.shortnodeToHash(collapsed, force)
        // We need to retain the possibly _not_ hashed node, in case it was too
        // small to be hashed
        if hn, ok := hashed.(hashNode); ok {
            cached.flags.hash = hn
        } else {
            cached.flags.hash = nil
        }
        return hashed, cached
    case *fullNode:
        collapsed, cached := h.hashFullNodeChildren(n)
        hashed = h.fullnodeToHash(collapsed, force)
        if hn, ok := hashed.(hashNode); ok {
            cached.flags.hash = hn
        } else {
            cached.flags.hash = nil
        }
        return hashed, cached
    default:
        // Value and hash nodes don't have children so they're left as were
        return n, n
    }
}
```

在这里首先检查缓存的 hash 是否可用，如果可用则直接返回该缓存的 hash 值：

```plain
// Return the cached hash if it's available
    if hash, _ := n.cache(); hash != nil {
        return hash, n
    }
```

如果缓存不可用则根据当前的节点类型做相关处理：

```plain
// Trie not processed yet, walk the children
    switch n := n.(type) {
    case *shortNode:
        collapsed, cached := h.hashShortNodeChildren(n)
        hashed := h.shortnodeToHash(collapsed, force)
        // We need to retain the possibly _not_ hashed node, in case it was too
        // small to be hashed
        if hn, ok := hashed.(hashNode); ok {
            cached.flags.hash = hn
        } else {
            cached.flags.hash = nil
        }
        return hashed, cached
    case *fullNode:
        collapsed, cached := h.hashFullNodeChildren(n)
        hashed = h.fullnodeToHash(collapsed, force)
        if hn, ok := hashed.(hashNode); ok {
            cached.flags.hash = hn
        } else {
            cached.flags.hash = nil
        }
        return hashed, cached
    default:
        // Value and hash nodes don't have children so they're left as were
        return n, n
    }
```

如果是一个扩展节点，则首先调用 hashShortNodeChildren 来折叠扩展节点：

```plain
case *shortNode:
        collapsed, cached := h.hashShortNodeChildren(n)
        hashed := h.shortnodeToHash(collapsed, force)
        // We need to retain the possibly _not_ hashed node, in case it was too
        // small to be hashed
        if hn, ok := hashed.(hashNode); ok {
            cached.flags.hash = hn
        } else {
            cached.flags.hash = nil
        }
        return hashed, cached
```

hashShortNodeChildren 函数将所有的子节点替换成他们的 hash 值，在这里 cached 变量接管了原来的 Trie 树的完整结构，collapsed 变量存储子节点的 hash 值，之后调用 hexToCompact 将 Hex Encoding 编码转换为 Compact Encoding，之后检查扩展节点的子节点的类型，并递归调用 hash 方法计算子节点的 hash 值，从而将子节点替换成子节点的 hash 值：

```plain
// hashShortNodeChildren collapses the short node. The returned collapsed node
// holds a live reference to the Key, and must not be modified.
// The cached
func (h *hasher) hashShortNodeChildren(n *shortNode) (collapsed, cached *shortNode) {
    // Hash the short node's child, caching the newly hashed subtree
    collapsed, cached = n.copy(), n.copy()
    // Previously, we did copy this one. We don't seem to need to actually
    // do that, since we don't overwrite/reuse keys
    //cached.Key = common.CopyBytes(n.Key)
    collapsed.Key = hexToCompact(n.Key)
    // Unless the child is a valuenode or hashnode, hash it
    switch n.Val.(type) {
    case *fullNode, *shortNode:
        collapsed.Val, cached.Val = h.hash(n.Val, false)
    }
    return collapsed, cached
}
```

之后调用 shortnodeToHash 方法根据扩展节点来创建一个 HashNode，在这里的 shortNode 子节点被替换成了子节点的 hash 值，之后通过 rlp.Encode 方法对这个节点进行编码，如果编码后的值小于 32，并且这个节点不是根节点，那么就把他们直接存储在他们的父节点里面，否则调用 hashDate 方法通过 h.sha.Write 方法进行 hash 计算，然后把 hash 值和编码后的数据存储到数据库里面，然后返回 hash 值

```plain
// shortnodeToHash creates a hashNode from a shortNode. The supplied shortnode
// should have hex-type Key, which will be converted (without modification)
// into compact form for RLP encoding.
// If the rlp data is smaller than 32 bytes, `nil` is returned.
func (h *hasher) shortnodeToHash(n *shortNode, force bool) node {
    h.tmp.Reset()
    if err := rlp.Encode(&h.tmp, n); err != nil {
        panic("encode error: " + err.Error())
    }

    if len(h.tmp) < 32 && !force {
        return n // Nodes smaller than 32 bytes are stored inside their parent
    }
    return h.hashData(h.tmp)
}
```

hashDate 对 data 进行一次 hash 处理：

```plain
// hashData hashes the provided data
func (h *hasher) hashData(data []byte) hashNode {
    n := make(hashNode, 32)
    h.sha.Reset()
    h.sha.Write(data)
    h.sha.Read(n)
    return n
}
```

如果当前节点是 fullNode, 那么遍历每个子节点，把子节点替换成子节点的 Hash 值：

```plain
case *fullNode:
        collapsed, cached := h.hashFullNodeChildren(n)
        hashed = h.fullnodeToHash(collapsed, force)
        if hn, ok := hashed.(hashNode); ok {
            cached.flags.hash = hn
        } else {
            cached.flags.hash = nil
        }
        return hashed, cached
```

hashFullNodeChildren 函数的具体实现如下，这里的 cache 变量接管了原来的 Trie 树的完整结构，collapsed 变量用于将子节点替换成子节点的 hash 值，在这里首先检查是否使用并发线程，如果采用并发线程则执行 else 部分代码进行并发操作，否则通过 for 循环来递归调用 hash 函数处理分支节点的每一个子节点：

```plain
func (h *hasher) hashFullNodeChildren(n *fullNode) (collapsed *fullNode, cached *fullNode) {
    // Hash the full node's children, caching the newly hashed subtrees
    cached = n.copy()
    collapsed = n.copy()
    if h.parallel {
        var wg sync.WaitGroup
        wg.Add(16)
        for i := 0; i < 16; i++ {
            go func(i int) {
                hasher := newHasher(false)
                if child := n.Children[i]; child != nil {
                    collapsed.Children[i], cached.Children[i] = hasher.hash(child, false)
                } else {
                    collapsed.Children[i] = nilValueNode
                }
                returnHasherToPool(hasher)
                wg.Done()
            }(i)
        }
        wg.Wait()
    } else {
        for i := 0; i < 16; i++ {
            if child := n.Children[i]; child != nil {
                collapsed.Children[i], cached.Children[i] = h.hash(child, false)
            } else {
                collapsed.Children[i] = nilValueNode
            }
        }
    }
    return collapsed, cached
}
```

fullnodeToHash 与 shortnodeToHash 方法类似，在这里的 fullnode 子节点被替换成了子节点的 hash 值，之后通过 rlp.Encode 方法对这个节点进行编码，如果编码后的值小于 32，并且这个节点不是根节点，那么就把他们直接存储在他们的父节点里面，否则调用 hashDate 方法通过 h.sha.Write 方法进行 hash 计算，然后把 hash 值和编码后的数据存储到数据库里面，然后返回 hash 值

```plain
// shortnodeToHash is used to creates a hashNode from a set of hashNodes, (which
// may contain nil values)
func (h *hasher) fullnodeToHash(n *fullNode, force bool) node {
    h.tmp.Reset()
    // Generate the RLP encoding of the node
    if err := n.EncodeRLP(&h.tmp); err != nil {
        panic("encode error: " + err.Error())
    }

    if len(h.tmp) < 32 && !force {
        return n // Nodes smaller than 32 bytes are stored inside their parent
    }
    return h.hashData(h.tmp)
}
```

如果节点没有子节点则直接返回：

```plain
default:
        // Value and hash nodes don't have children so they're left as were
        return n, n
    }
```

紧接着我们回到 commit 函数中，在后续的代码中会首先做一个快速的检查来判断是否需要提交变更，如果无需更新则直接返回原 RootHash，否则将节点向下折叠为哈希节点并将其插入数据库并返回最小的 Trie 的 root 值：

```plain
// Do a quick check if we really need to commit, before we spin
    // up goroutines. This can happen e.g. if we load a trie for reading storage
    // values, but don't write to it.
    if _, dirty := t.root.cache(); !dirty {
        return rootHash, nil
    }
    var wg sync.WaitGroup
    if onleaf != nil {
        h.onleaf = onleaf
        h.leafCh = make(chan *leaf, leafChanSize)
        wg.Add(1)
        go func() {
            defer wg.Done()
            h.commitLoop(t.db)
        }()
    }
    var newRoot hashNode
    newRoot, err = h.Commit(t.root, t.db)
    if onleaf != nil {
        // The leafch is created in newCommitter if there was an onleaf callback
        // provided. The commitLoop only _reads_ from it, and the commit
        // operation was the sole writer. Therefore, it's safe to close this
        // channel here.
        close(h.leafCh)
        wg.Wait()
    }
    if err != nil {
        return common.Hash{}, err
    }
    t.root = newRoot
    return rootHash, nil
```

### 树的证明

Prove 方法获取指定 Key 的 proof 证明，proof 方法从根节点开始把经过的节点的 hash 值一个一个存入到 list 中然后返回：

```plain
// Prove constructs a merkle proof for key. The result contains all encoded nodes
// on the path to the value at key. The value itself is also included in the last
// node and can be retrieved by verifying the proof.
//
// If the trie does not contain a value for key, the returned proof contains all
// nodes of the longest existing prefix of the key (at least the root node), ending
// with the node that proves the absence of the key.
func (t *Trie) Prove(key []byte, fromLevel uint, proofDb ethdb.KeyValueWriter) error {
    // Collect all nodes on the path to key.
    key = keybytesToHex(key)
    var nodes []node
    tn := t.root
    for len(key) > 0 && tn != nil {
        switch n := tn.(type) {
        case *shortNode:
            if len(key) < len(n.Key) || !bytes.Equal(n.Key, key[:len(n.Key)]) {
                // The trie doesn't contain the key.
                tn = nil
            } else {
                tn = n.Val
                key = key[len(n.Key):]
            }
            nodes = append(nodes, n)
        case *fullNode:
            tn = n.Children[key[0]]
            key = key[1:]
            nodes = append(nodes, n)
        case hashNode:
            var err error
            tn, err = t.resolveHash(n, nil)
            if err != nil {
                log.Error(fmt.Sprintf("Unhandled trie error: %v", err))
                return err
            }
        default:
            panic(fmt.Sprintf("%T: invalid node: %v", tn, tn))
        }
    }
    hasher := newHasher(false)
    defer returnHasherToPool(hasher)

    for i, n := range nodes {
        if fromLevel > 0 {
            fromLevel--
            continue
        }
        var hn node
        n, hn = hasher.proofHash(n)
        if hash, ok := hn.(hashNode); ok || i == 0 {
            // If the node's database encoding is a hash (or is the
            // root node), it becomes a proof element.
            enc, _ := rlp.EncodeToBytes(n)
            if !ok {
                hash = hasher.hashData(enc)
            }
            proofDb.Put(hash, enc)
        }
    }
    return nil
}
```

VerifyProof 方法，收一个 rootHash 参数、key 参数、、proof 数组，之后一个一个验证是否可以与数据库里面的数据信息对应上：

```plain
// VerifyProof checks merkle proofs. The given proof must contain the value for
// key in a trie with the given root hash. VerifyProof returns an error if the
// proof contains invalid trie nodes or the wrong value.
func VerifyProof(rootHash common.Hash, key []byte, proofDb ethdb.KeyValueReader) (value []byte, err error) {
    key = keybytesToHex(key)
    wantHash := rootHash
    for i := 0; ; i++ {
        buf, _ := proofDb.Get(wantHash[:])
        if buf == nil {
            return nil, fmt.Errorf("proof node %d (hash %064x) missing", i, wantHash)
        }
        n, err := decodeNode(wantHash[:], buf)
        if err != nil {
            return nil, fmt.Errorf("bad proof node %d: %v", i, err)
        }
        keyrest, cld := get(n, key, true)
        switch cld := cld.(type) {
        case nil:
            // The trie doesn't contain the key.
            return nil, nil
        case hashNode:
            key = keyrest
            copy(wantHash[:], cld)
        case valueNode:
            return cld, nil
        }
    }
}
```

VerifyRangeProof 方法用于检查给定的叶子节点和证明是否可以证明 Trie 的叶子范围可以与特殊的根匹配，并且范围是连续的 (内部没有间隙) 单调递增：

```plain
// VerifyRangeProof checks whether the given leaf nodes and edge proof
// can prove the given trie leaves range is matched with the specific root.
// Besides, the range should be consecutive (no gap inside) and monotonic
// increasing.
//
// Note the given proof actually contains two edge proofs. Both of them can
// be non-existent proofs. For example the first proof is for a non-existent
// key 0x03, the last proof is for a non-existent key 0x10. The given batch
// leaves are [0x04, 0x05, .. 0x09]. It's still feasible to prove the given
// batch is valid.
//
// The firstKey is paired with firstProof, not necessarily the same as keys[0]
// (unless firstProof is an existent proof). Similarly, lastKey and lastProof
// are paired.
//
// Expect the normal case, this function can also be used to verify the following
// range proofs:
//
// - All elements proof. In this case the proof can be nil, but the range should
//   be all the leaves in the trie.
//
// - One element proof. In this case no matter the edge proof is a non-existent
//   proof or not, we can always verify the correctness of the proof.
//
// - Zero element proof. In this case a single non-existent proof is enough to prove.
//   Besides, if there are still some other leaves available on the right side, then
//   an error will be returned.
//
// Except returning the error to indicate the proof is valid or not, the function will
// also return a flag to indicate whether there exists more accounts/slots in the trie.
func VerifyRangeProof(rootHash common.Hash, firstKey []byte, lastKey []byte, keys [][]byte, values [][]byte, proof ethdb.KeyValueReader) (ethdb.KeyValueStore, *Trie, *KeyValueNotary, bool, error) {
    if len(keys) != len(values) {
        return nil, nil, nil, false, fmt.Errorf("inconsistent proof data, keys: %d, values: %d", len(keys), len(values))
    }
    // Ensure the received batch is monotonic increasing.
    for i := 0; i < len(keys)-1; i++ {
        if bytes.Compare(keys[i], keys[i+1]) >= 0 {
            return nil, nil, nil, false, errors.New("range is not monotonically increasing")
        }
    }
    // Create a key-value notary to track which items from the given proof the
    // range prover actually needed to verify the data
    notary := NewKeyValueNotary(proof)

    // Special case, there is no edge proof at all. The given range is expected
    // to be the whole leaf-set in the trie.
    if proof == nil {
        var (
            diskdb = memorydb.New()
            triedb = NewDatabase(diskdb)
        )
        tr, err := New(common.Hash{}, triedb)
        if err != nil {
            return nil, nil, nil, false, err
        }
        for index, key := range keys {
            tr.TryUpdate(key, values[index])
        }
        if tr.Hash() != rootHash {
            return nil, nil, nil, false, fmt.Errorf("invalid proof, want hash %x, got %x", rootHash, tr.Hash())
        }
        // Proof seems valid, serialize all the nodes into the database
        if _, err := tr.Commit(nil); err != nil {
            return nil, nil, nil, false, err
        }
        if err := triedb.Commit(rootHash, false, nil); err != nil {
            return nil, nil, nil, false, err
        }
        return diskdb, tr, notary, false, nil // No more elements
    }
    // Special case, there is a provided edge proof but zero key/value
    // pairs, ensure there are no more accounts / slots in the trie.
    if len(keys) == 0 {
        root, val, err := proofToPath(rootHash, nil, firstKey, notary, true)
        if err != nil {
            return nil, nil, nil, false, err
        }
        if val != nil || hasRightElement(root, firstKey) {
            return nil, nil, nil, false, errors.New("more entries available")
        }
        // Since the entire proof is a single path, we can construct a trie and a
        // node database directly out of the inputs, no need to generate them
        diskdb := notary.Accessed()
        tr := &Trie{
            db:   NewDatabase(diskdb),
            root: root,
        }
        return diskdb, tr, notary, hasRightElement(root, firstKey), nil
    }
    // Special case, there is only one element and two edge keys are same.
    // In this case, we can't construct two edge paths. So handle it here.
    if len(keys) == 1 && bytes.Equal(firstKey, lastKey) {
        root, val, err := proofToPath(rootHash, nil, firstKey, notary, false)
        if err != nil {
            return nil, nil, nil, false, err
        }
        if !bytes.Equal(firstKey, keys[0]) {
            return nil, nil, nil, false, errors.New("correct proof but invalid key")
        }
        if !bytes.Equal(val, values[0]) {
            return nil, nil, nil, false, errors.New("correct proof but invalid data")
        }
        // Since the entire proof is a single path, we can construct a trie and a
        // node database directly out of the inputs, no need to generate them
        diskdb := notary.Accessed()
        tr := &Trie{
            db:   NewDatabase(diskdb),
            root: root,
        }
        return diskdb, tr, notary, hasRightElement(root, firstKey), nil
    }
    // Ok, in all other cases, we require two edge paths available.
    // First check the validity of edge keys.
    if bytes.Compare(firstKey, lastKey) >= 0 {
        return nil, nil, nil, false, errors.New("invalid edge keys")
    }
    // todo(rjl493456442) different length edge keys should be supported
    if len(firstKey) != len(lastKey) {
        return nil, nil, nil, false, errors.New("inconsistent edge keys")
    }
    // Convert the edge proofs to edge trie paths. Then we can
    // have the same tree architecture with the original one.
    // For the first edge proof, non-existent proof is allowed.
    root, _, err := proofToPath(rootHash, nil, firstKey, notary, true)
    if err != nil {
        return nil, nil, nil, false, err
    }
    // Pass the root node here, the second path will be merged
    // with the first one. For the last edge proof, non-existent
    // proof is also allowed.
    root, _, err = proofToPath(rootHash, root, lastKey, notary, true)
    if err != nil {
        return nil, nil, nil, false, err
    }
    // Remove all internal references. All the removed parts should
    // be re-filled(or re-constructed) by the given leaves range.
    empty, err := unsetInternal(root, firstKey, lastKey)
    if err != nil {
        return nil, nil, nil, false, err
    }
    // Rebuild the trie with the leaf stream, the shape of trie
    // should be same with the original one.
    var (
        diskdb = memorydb.New()
        triedb = NewDatabase(diskdb)
    )
    tr := &Trie{root: root, db: triedb}
    if empty {
        tr.root = nil
    }
    for index, key := range keys {
        tr.TryUpdate(key, values[index])
    }
    if tr.Hash() != rootHash {
        return nil, nil, nil, false, fmt.Errorf("invalid proof, want hash %x, got %x", rootHash, tr.Hash())
    }
    // Proof seems valid, serialize all the nodes into the database
    if _, err := tr.Commit(nil); err != nil {
        return nil, nil, nil, false, err
    }
    if err := triedb.Commit(rootHash, false, nil); err != nil {
        return nil, nil, nil, false, err
    }
    return diskdb, tr, notary, hasRightElement(root, keys[len(keys)-1]), nil
}
```

## 安全漏洞

在区块链中 MPT(Merkle Patricia 树) 是一种常用的数据结构，用于存储和验证区块链中的账户和交易信息，尽管 MPT 本身并不具备安全漏洞，但在使用 MPT 的过程中可能会存在一些与区块链安全相关的问题，例如：Hash Collision  
Hash Collision 是指攻击者找到两个不同的数据集，但它们的哈希值相同，如果攻击者能够生成具有相同哈希值的两个不同数据集，他们可以将一个有效的数据集替换为恶意数据集，而 Merkle Tree Root 的哈希值将保持不变，那么将导致验证通过，但实际上数据已被篡改，下面是一个漏洞示例的伪代码：

```plain
......
 while len(hashList) > 1:
        newHashList = []

        for i in range(0, len(hashList), 2):
            if i+1 < len(hashList):
                if hashList[i] > hashList[i+1]:
                    combined = hashList[i] + hashList[i+1]
                else:
                    combined = hashList[i+1] + hashList[i]
            else:
                combined = hashList[i] + hashList[i]

            newHash = hashlib.sha3_256(combined.encode()).hexdigest()
            newHashList.append(newHash)

        hashList = newHashList

    return hashList[0]
......
```

这里可能大家会有疑问就是为啥能够生成同样的默克尔树根，这是因为我们再计算默克尔树根时是对交易进行两两取 Hash 计算的，那么这里可以考虑的一个点就是如果交互一下次序之后，是否会变化呢？答案是不会，我们可以通过下面的图来进行一个很好的理解，攻击者可以交换 MT 任意非叶子节点下以两个孩子为根的两颗树整体的位置，即直接交换两组交易的顺序，此时是不会改变 MTR 的值的，于是这样理论上每个区块就能构造出许多
