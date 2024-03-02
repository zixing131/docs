

# 以太坊 P2P 网络深入刨析 - 先知社区

以太坊 P2P 网络深入刨析

- - -

### 文章前言

以太坊作为一种开源的、基于区块链技术的平台，其分布式网络结构是其核心的基石，以太坊 P2P(点对点) 网络的设计和功能对于实现去中心化、安全可靠的交易和智能合约执行至关重要，深入了解以太坊 P2P 网络的工作原理和机制对于理解以太坊的运行方式和其在区块链生态系统中的地位具有重要意义，本文将对以太坊 P2P 网络进行深入的刨析，旨在帮助读者全面了解以太坊网络的架构、通信协议和节点交互，同时将给出公链的 P2P 网络中潜在的安全风险面

### 基本介绍

P2P(点对点) 网络是一种分布式网络模型，其中没有中央服务器，而是由相互连接的节点组成，在 P2P 网络中每个节点既可以是服务提供者，也可以是服务请求者，彼此之间通过直接通信进行数据传输和共享。在 P2P 网络中节点之间共享资源和任务，当一个节点需要某种资源或服务时，它可以通过网络广播请求，其他节点可以响应并提供所需的资源，这种直接的节点之间的通信和资源共享使得 P2P 网络具有高度的可扩展性

### 节点类型

P2P(点对点) 网络中存在多种类型的节点，每种类型具有不同的功能和角色，以下是几种常见的 P2P 节点类型：

-   Full Node(完整节点)：完整节点是 P2P 网络中最基本的节点类型，它们存储和维护完整的区块链数据并能够验证和广播交易和区块，完整节点在网络中扮演关键的角色，它们可以提供节点发现、交易广播、区块同步等功能
-   Lightweight Node(轻节点)：轻节点是相对于完整节点而言的，它们不存储完整的区块链数据，而是通过与其他节点进行交互来获取所需的数据，轻节点通常使用简化的验证方法，依赖于完整节点提供的数据和验证结果
-   Seed Node(种子节点)：种子节点是网络中的起始节点，它们的主要目的是帮助新加入的节点进行初始连接，种子节点的地址通常是硬编码在 P2P 应用程序中，新节点可以使用这些地址来发现其他节点并加入网络
-   Bootstrap Node(引导节点)：引导节点是一组特定的节点，它们的主要功能是帮助新加入的节点发现其他对等节点，引导节点的地址通常是固定的并且由网络的维护者提供
-   Super Node(超级节点)：超级节点是在 P2P 网络中扮演特殊角色的节点，它们通常具有更高的性能和可靠性并提供额外的服务和功能，超级节点通常由网络的管理员或特定的节点组织运营

### 工作原理

P2P 网络的工作原理基于节点之间的直接通信和资源共享，节点可以是计算机、移动设备或其他网络设备，每个节点既可以提供服务，也可以请求服务，它们之间通过建立直接的点对点连接来交换信息

-   节点发现和连接：P2P 网络中的节点需要能够发现其他节点并建立连接，节点发现可以通过多种方式实现，而比较常见的一种方法是使用中央服务器作为引导节点，新加入的节点可以向中央服务器请求已知节点的列表并通过这些节点建立连接，另一种方法是使用已知的种子节点，这些节点在网络中预先设置并充当其他节点的入口点，节点可以连接到种子节点并获取其他节点的信息以建立连接，还有一种方法是使用分布式哈希表 (DHT)。DHT 将节点和资源映射到网络中的唯一标识符并使用一致性哈希算法来确定数据的存储和路由，节点可以查询 DHT 以获取其他节点的信息并建立连接
-   资源共享：P2P 网络的核心是资源共享，节点可以共享各种类型的资源，例如：文件、带宽、计算能力等，文件共享是 P2P 网络中最常见的用例之一，当节点具有某个文件时它可以将文件分成小块并将这些块分发给其他节点，其他节点可以从多个来源下载这些块从而实现更快的下载速度，节点之间的资源共享可以通过协议来实现，例如:BitTorrent 协议用于文件共享，Kademlia 协议用于分布式哈希表
-   路由和数据传输：在 P2P 网络中节点需要能够路由数据并将其传输到目标节点，为了实现这一点，P2P 网络使用路由算法来确定最佳路径并确保数据能够通过网络正确地传递到目标节点，常见的路由算法包括基于邻居关系的洪泛算法和分布式哈希表 (DHT)，在洪泛算法中节点将数据广播到其邻居节点并由邻居节点进一步传播，直到到达目标节点，在 DHT 中节点通过查询 DHT 来找到存储和路由数据的节点，数据被分割成块并根据块的标识符进行路由

比较典型的一个示例是比特币网络，比特币网络是一个基于 P2P 的分布式账本，它使用共识算法 (工作量证明) 确保节点对交易的一致性达成共识，在比特币网络中每个节点都是一个矿工，可以创建新的区块并验证交易，节点通过连接到其他节点并广播新区块和交易来进行通信，其他节点可以验证区块和交易并根据共识算法确定哪个区块将被添加到区块链中，比特币网络中的节点通过点对点通信实现交易的广播和验证，而无需中央服务器或权威机构的干预

### 源码分析

下面我们通过对以太坊的 P2P 网络架构源码分析来刨析公链的网络设计：

#### 数据结构

下面的代码实现了一个基于 Kademlia 协议的节点表和桶数据结构，用于管理和存储 P2P 网络中的节点信息，节点表用于维护已知节点的索引以便进行节点的发现、连接和更新，桶用于按照节点与自身的距离将节点进行分组并根据节点的活动情况进行排序，这种基于 Kademlia 的节点表和桶数据结构是 P2P 网络中节点管理的核心组件，用于支持节点之间的通信和资源共享，在代码的开头首先定义了一些常量，包括 alpha、bucketSize、maxReplacements 等，这些常量主要用于配置 Kademlia 算法的参数，例如：并行度、桶的大小和替换列表的大小，接下来是 Table 结构体的定义，它代表了节点表，Table 结构体包含了互斥锁 (mutex) 用于保护数据访问以及一个 buckets 数组，用于存储根据节点与自身的距离分组的桶，它还包含了一个 nursery 数组用于存储引导节点 (bootstrap nodes) 以及一个 rand 对象用于生成随机数，Table 结构体还有一些其他字段，例如：日志记录器 (log)、已知节点数据库 (db)、网络传输模块 (net) 等，而在 Table 结构体下方，定义了 bucket 结构体，每个 bucket 代表一个存储节点的桶，bucket 结构体包含了两个切片：entries 和 replacements，entries 切片按照节点的最后活动时间进行排序，存储了最近活动的节点，replacements 切片存储了最近看到但尚未验证的节点，可以在验证失败时使用，bucket 结构体还包含了一个 ips 字段，用于存储节点的 IP 地址

```plain
// filedir:go-ethereum-1.10.2\p2p\discover\table.go L40
const (
    alpha           = 3  // Kademlia concurrency factor
    bucketSize      = 16 // Kademlia bucket size
    maxReplacements = 10 // Size of per-bucket replacement list

    // We keep buckets for the upper 1/15 of distances because
    // it's very unlikely we'll ever encounter a node that's closer.
    hashBits          = len(common.Hash{}) * 8
    nBuckets          = hashBits / 15       // Number of buckets
    bucketMinDistance = hashBits - nBuckets // Log distance of closest bucket

    // IP address limits.
    bucketIPLimit, bucketSubnet = 2, 24 // at most 2 addresses from the same /24
    tableIPLimit, tableSubnet   = 10, 24

    refreshInterval    = 30 * time.Minute
    revalidateInterval = 10 * time.Second
    copyNodesInterval  = 30 * time.Second
    seedMinTableTime   = 5 * time.Minute
    seedCount          = 30
    seedMaxAge         = 5 * 24 * time.Hour
)

// Table is the 'node table', a Kademlia-like index of neighbor nodes. The table keeps
// itself up-to-date by verifying the liveness of neighbors and requesting their node
// records when announcements of a new record version are received.
type Table struct {
    mutex   sync.Mutex        // protects buckets, bucket content, nursery, rand
    buckets [nBuckets]*bucket // index of known nodes by distance
    nursery []*node           // bootstrap nodes
    rand    *mrand.Rand       // source of randomness, periodically reseeded
    ips     netutil.DistinctNetSet

    log        log.Logger
    db         *enode.DB // database of known nodes
    net        transport
    refreshReq chan chan struct{}
    initDone   chan struct{}
    closeReq   chan struct{}
    closed     chan struct{}

    nodeAddedHook func(*node) // for testing
}
// filedir:go-ethereum-1.10.2\p2p\discover\table.go L94
// bucket contains nodes, ordered by their last activity. the entry
// that was most recently active is the first element in entries.
type bucket struct {
    entries      []*node // live entries, sorted by time of last contact
    replacements []*node // recently seen nodes to be used if revalidation fails
    ips          netutil.DistinctNetSet
}
```

#### 节点列表

下面的代码定义了一个名为 newTable 的函数，用于创建一个新的 Table 实例，函数接收一些参数，包括网络传输模块 (t)、已知节点数据库 (db)、引导节点列表 (bootnodes) 和日志记录器 (log) 并返回创建的 Table 实例和一个可能的错误，在函数内部首先创建了一个 Table 结构体的实例 tab 并使用传入的参数对其进行初始化，具体的初始化步骤如下：

1.  将传入的网络传输模块（t）和已知节点数据库（db）分别赋值给 tab 的 net 和 db 字段。
2.  创建用于通信和控制的通道：refreshReq、initDone、closeReq 和 closed。
3.  创建随机数生成器 (rand) 并使用固定的种子值 (mrand.NewSource(0)) 进行初始化，这个随机数生成器用于节点选择和其他需要随机性的操作
4.  创建了一个 DistinctNetSet 实例 ips，用于存储节点的 IP 地址，设置了子网掩码 (Subnet) 为 tableSubnet，IP 地址限制 (Limit) 为 tableIPLimit，这个 ips 对象用于限制相同子网下的 IP 地址数量
5.  将传入的日志记录器 (log) 赋值给 tab 的 log 字段

```plain
// filedir:go-ethereum-1.10.2\p2p\discover\table.go L102
func newTable(t transport, db *enode.DB, bootnodes []*enode.Node, log log.Logger) (*Table, error) {
    tab := &Table{
        net:        t,
        db:         db,
        refreshReq: make(chan chan struct{}),
        initDone:   make(chan struct{}),
        closeReq:   make(chan struct{}),
        closed:     make(chan struct{}),
        rand:       mrand.New(mrand.NewSource(0)),
        ips:        netutil.DistinctNetSet{Subnet: tableSubnet, Limit: tableIPLimit},
        log:        log,
    }
    if err := tab.setFallbackNodes(bootnodes); err != nil {
        return nil, err
    }
    for i := range tab.buckets {
        tab.buckets[i] = &bucket{
            ips: netutil.DistinctNetSet{Subnet: bucketSubnet, Limit: bucketIPLimit},
        }
    }
    tab.seedRand()
    tab.loadSeedNodes()

    return tab, nil
}
```

tab.setFallbackNodes(bootnodes) 方法用于设置引导节点，这个方法将传入的引导节点列表 bootnodes 添加到 Table 实例的 nursery 字段中，如果设置引导节点时出现错误，将会返回一个错误

```plain
// setFallbackNodes sets the initial points of contact. These nodes
// are used to connect to the network if the table is empty and there
// are no known nodes in the database.
func (tab *Table) setFallbackNodes(nodes []*enode.Node) error {
    for _, n := range nodes {
        if err := n.ValidateComplete(); err != nil {
            return fmt.Errorf("bad bootstrap node %q: %v", n, err)
        }
    }
    tab.nursery = wrapNodes(nodes)
    return nil
}
```

在这里会使用 ValidateComplete 方法验证一个 Node 实例是否具有完整且有效的 IP 地址和 UDP 端口并验证节点的密钥信息，它可以用于确保节点的网络信息和密钥信息的完整性和正确性

```plain
// filedir:go-ethereum-1.10.2\p2p\enode\node.go  L146
// ValidateComplete checks whether n has a valid IP and UDP port.
// Deprecated: don't use this method.
func (n *Node) ValidateComplete() error {
    if n.Incomplete() {
        return errors.New("missing IP address")
    }
    if n.UDP() == 0 {
        return errors.New("missing UDP port")
    }
    ip := n.IP()
    if ip.IsMulticast() || ip.IsUnspecified() {
        return errors.New("invalid IP (multicast/unspecified)")
    }
    // Validate the node key (on curve, etc.).
    var key Secp256k1
    return n.Load(&key)
}
```

随后初始化 K 桶：

```plain
for i := range tab.buckets {
        tab.buckets[i] = &bucket{
            ips: netutil.DistinctNetSet{Subnet: bucketSubnet, Limit: bucketIPLimit},
        }
    }
```

随后从 table.buckets 中随机取 30 个节点加载种子节点到相应的 bucket:

```plain
tab.seedRand()
    tab.loadSeedNodes()

    return tab, nil
```

loadSeedNodes 函数的具体实现如下所示 (这里的 seedCount 为 table.go 中最上方定义的全局变量，值为 30)：

```plain
// filedir: go-ethereum-1.10.2\p2p\discover\table.go    L302
func (tab *Table) loadSeedNodes() {
    seeds := wrapNodes(tab.db.QuerySeeds(seedCount, seedMaxAge))
    seeds = append(seeds, tab.nursery...)
    for i := range seeds {
        seed := seeds[i]
        age := log.Lazy{Fn: func() interface{} { return time.Since(tab.db.LastPongReceived(seed.ID(), seed.IP())) }}
        tab.log.Trace("Found seed node in database", "id", seed.ID(), "addr", seed.addr(), "age", age)
        tab.addSeenNode(seed)
    }
}
```

这里的 addSeenNode 即用于添加节点到 bucket，在这里会检查要添加的节点是否已经存在以及 bucket 是否已满，如果已满则调用 tab.addReplacement(b, n) 将节点添加到 replacement 列表中去，之后添加 IP，之后更新 bucket：

```plain
// filedir: go-ethereum-1.10.2\p2p\discover\table.go L458
// addSeenNode adds a node which may or may not be live to the end of a bucket. If the
// bucket has space available, adding the node succeeds immediately. Otherwise, the node is
// added to the replacements list.
//
// The caller must not hold tab.mutex.
func (tab *Table) addSeenNode(n *node) {
    if n.ID() == tab.self().ID() {
        return
    }

    tab.mutex.Lock()
    defer tab.mutex.Unlock()
    b := tab.bucket(n.ID())
    if contains(b.entries, n.ID()) {
        // Already in bucket, don't add.
        return
    }
    if len(b.entries) >= bucketSize {
        // Bucket full, maybe add as replacement.
        tab.addReplacement(b, n)
        return
    }
    if !tab.addIP(b, n.IP()) {
        // Can't add: IP limit reached.
        return
    }
    // Add to end of bucket:
    b.entries = append(b.entries, n)
    b.replacements = deleteNode(b.replacements, n)
    n.addedAt = time.Now()
    if tab.nodeAddedHook != nil {
        tab.nodeAddedHook(n)
    }
}
```

#### 事件监听

下面的 loop 函数是一个循环调度器，主要用于定时触发并执行刷新 K 桶、验证 K 桶节点有效性和将节点复制到数据库等操作，它通过定时器和通道来实现操作的调度和协同，首先在代码的开头定义了一些常量：

-   refreshInterval 定义为 30 分钟，表示刷新 K 桶的时间间隔
-   revalidateInterval 定义为 10 秒，表示验证 K 桶节点有效性的时间间隔
-   copyNodesInterval 定义为 30 秒，表示将节点复制到数据库中的时间间隔

接下来在函数内部创建了一些变量和计时器：

-   revalidate 是一个计时器，用于定时触发 doRevalidate 操作
-   refresh 是一个定时器，用于定时触发 doRefresh 操作
-   copyNodes 是一个定时器，用于定时触发 copyLiveNodes 操作
-   refreshDone 是一个通道，用于接收 doRefresh 操作的完成信号
-   revalidateDone 是一个通道，用于接收 doRevalidate 操作的完成信号
-   waiting 是一个切片，用于存储等待中的调用者的通道，初始时将 tab.initDone 通道添加到其中

接下来使用 defer 语句停止计时器的运行，在循环开始之前，启动了一个 go 协程来执行 doRefresh 操作并将 refreshDone 通道作为参数传递给它，进入循环后使用 select 语句监听多个通道的事件，循环结束后，则直接使用条件语句检查 refreshDone、revalidateDone 是否为 nil，如果不为 nil 则等待它们的完成信号，然后遍历 waiting 切片中的所有通道并关闭它们，最后关闭 tab.closed 通道，表示 loop 方法执行结束

```plain
// filedir: go-ethereum-1.10.2\p2p\discover\table.go L55
    refreshInterval    = 30 * time.Minute
    revalidateInterval = 10 * time.Second
    copyNodesInterval  = 30 * time.Second

// filedir: go-ethereum-1.10.2\p2p\discover\table.go  L218
// loop schedules runs of doRefresh, doRevalidate and copyLiveNodes.
func (tab *Table) loop() {
    var (
        revalidate     = time.NewTimer(tab.nextRevalidateTime())
        refresh        = time.NewTicker(refreshInterval)
        copyNodes      = time.NewTicker(copyNodesInterval)
        refreshDone    = make(chan struct{})           // where doRefresh reports completion
        revalidateDone chan struct{}                   // where doRevalidate reports completion
        waiting        = []chan struct{}{tab.initDone} // holds waiting callers while doRefresh runs
    )
    defer refresh.Stop()
    defer revalidate.Stop()
    defer copyNodes.Stop()

    // Start initial refresh.
    go tab.doRefresh(refreshDone)

loop:
    for {
        select {
        case <-refresh.C:           //定时刷新 k 桶事件，refreshInterval=30 min
            tab.seedRand()
            if refreshDone == nil {
                refreshDone = make(chan struct{})
                go tab.doRefresh(refreshDone)
            }
        case req := <-tab.refreshReq:   //刷新 k 桶的请求事件
            waiting = append(waiting, req)
            if refreshDone == nil {
                refreshDone = make(chan struct{})
                go tab.doRefresh(refreshDone)
            }
        case <-refreshDone:         // doRefresh 完成
            for _, ch := range waiting {
                close(ch)
            }
            waiting, refreshDone = nil, nil
        case <-revalidate.C:       // 验证 k 桶节点有效性，10 second
            revalidateDone = make(chan struct{})
            go tab.doRevalidate(revalidateDone)
        case <-revalidateDone:    // 验证 K 桶节点有效性完成
            revalidate.Reset(tab.nextRevalidateTime())
            revalidateDone = nil
            case <-copyNodes.C:   // 定时 (30 秒) 将节点存入数据库，如果某个节点在 k 桶中存在超过 5 分钟，则认为它是一个稳定的节点
            go tab.copyLiveNodes()
        case <-tab.closeReq:
            break loop
        }
    }

    if refreshDone != nil {
        <-refreshDone
    }
    for _, ch := range waiting {
        close(ch)
    }
    if revalidateDone != nil {
        <-revalidateDone
    }
    close(tab.closed)
}
```

#### 节点查找

下面的 getNode 方法通过遍历桶中的条目并根据给定的 ID 查找返回相应的节点对象，它是在互斥锁的保护下进行的，以确保在查找期间没有其他并发修改对表的访问

```plain
// getNode returns the node with the given ID or nil if it isn't in the table.
func (tab *Table) getNode(id enode.ID) *enode.Node {
    tab.mutex.Lock()
    defer tab.mutex.Unlock()

    b := tab.bucket(id)
    for _, e := range b.entries {
        if e.ID() == id {
            return unwrapNode(e)
        }
    }
    return nil
}
```

#### 节点发现

以太坊分布式网络采用了结构化网络模型，其实现方案使用 Kademlia 协议，在以太坊中 k 值是 16，也就是说每个 k 桶包含 16 个节点，一共 256 个 k 桶，K 桶中记录节点的 NodeId，Distance，Endpoint，IP 等信息并按照与 Target 节点的距离排序，节点查找由 doRefresh() 实现，它主要执行刷新操作以保持桶中节点数量充足，它加载种子节点并插入到表中，执行自身查找以发现新的邻居节点并使用随机目标进行多次查找以刷新桶中的节点信息

```plain
// filedir:go-ethereum-1.10.2\p2p\discover\table.go L278
// doRefresh performs a lookup for a random target to keep buckets full. seed nodes are
// inserted if the table is empty (initial bootstrap or discarded faulty peers).
func (tab *Table) doRefresh(done chan struct{}) {
    defer close(done)

    // Load nodes from the database and insert
    // them. This should yield a few previously seen nodes that are
    // (hopefully) still alive.
    tab.loadSeedNodes()

    // Run self lookup to discover new neighbor nodes.
    tab.net.lookupSelf()

    // The Kademlia paper specifies that the bucket refresh should
    // perform a lookup in the least recently used bucket. We cannot
    // adhere to this because the findnode target is a 512bit value
    // (not hash-sized) and it is not easily possible to generate a
    // sha3 preimage that falls into a chosen bucket.
    // We perform a few lookups with a random target instead.
    for i := 0; i < 3; i++ {
        tab.net.lookupRandom()
    }
}
```

这里的 loadSeedNodes 方法通过调用 QuerySeeds 方法从数据库中查询种子节点并将它们插入到表中，QuerySeeds 方法使用一个迭代器在数据库中查找随机的节点，满足一定的条件后将节点添加到节点切片中

```plain
// filedir: go-ethereum-1.10.2\p2p\discover\table.go L302
func (tab *Table) loadSeedNodes() {
    seeds := wrapNodes(tab.db.QuerySeeds(seedCount, seedMaxAge))
    seeds = append(seeds, tab.nursery...)
    for i := range seeds {
        seed := seeds[i]
        age := log.Lazy{Fn: func() interface{} { return time.Since(tab.db.LastPongReceived(seed.ID(), seed.IP())) }}
        tab.log.Trace("Found seed node in database", "id", seed.ID(), "addr", seed.addr(), "age", age)
        tab.addSeenNode(seed)
    }
}

// filedir:go-ethereum-1.10.2\p2p\enode\nodedb.go L440
// QuerySeeds retrieves random nodes to be used as potential seed nodes
// for bootstrapping.
func (db *DB) QuerySeeds(n int, maxAge time.Duration) []*Node {
    var (
        now   = time.Now()
        nodes = make([]*Node, 0, n)
        it    = db.lvl.NewIterator(nil, nil)
        id    ID
    )
    defer it.Release()

seek:
    for seeks := 0; len(nodes) < n && seeks < n*5; seeks++ {
        // Seek to a random entry. The first byte is incremented by a
        // random amount each time in order to increase the likelihood
        // of hitting all existing nodes in very small databases.
        ctr := id[0]
        rand.Read(id[:])
        id[0] = ctr + id[0]%16
        it.Seek(nodeKey(id))

        n := nextNode(it)
        if n == nil {
            id[0] = 0
            continue seek // iterator exhausted
        }
        if now.Sub(db.LastPongReceived(n.ID(), n.IP())) > maxAge {
            continue seek
        }
        for i := range nodes {
            if nodes[i].ID() == n.ID() {
                continue seek // duplicate
            }
        }
        nodes = append(nodes, n)
    }
    return nodes
}
```

随后通过 lookupSelf 来发现新的节点，这里会优先使用当前节点的 ID 来运行 newLookup 发现邻居节点：

```plain
// filedir: go-ethereum-1.10.2\p2p\discover\v5_udp.go L281
// lookupSelf looks up our own node ID.
// This is needed to satisfy the transport interface.
func (t *UDPv5) lookupSelf() []*enode.Node {
    return t.newLookup(t.closeCtx, t.Self().ID()).run()
}

// filedir: go-ethereum-1.10.2\p2p\discover\v5_udp.go L292
func (t *UDPv5) newLookup(ctx context.Context, target enode.ID) *lookup {
    return newLookup(ctx, t.tab, target, func(n *node) ([]*node, error) {
        return t.lookupWorker(n, target)
    })
}

func newLookup(ctx context.Context, tab *Table, target enode.ID, q queryFunc) *lookup {
    it := &lookup{
        tab:       tab,
        queryfunc: q,
        asked:     make(map[enode.ID]bool),
        seen:      make(map[enode.ID]bool),
        result:    nodesByDistance{target: target},
        replyCh:   make(chan []*node, alpha),
        cancelCh:  ctx.Done(),
        queries:   -1,
    }
    // Don't query further if we hit ourself.
    // Unlikely to happen often in practice.
    it.asked[tab.self().ID()] = true
    return it
}
```

最后随机一个 target，进行 lookup:

```plain
// filedir: go-ethereum-1.10.2\p2p\discover\v5_udp.go L275
// lookupRandom looks up a random target.
// This is needed to satisfy the transport interface.
func (t *UDPv5) lookupRandom() []*enode.Node {
    return t.newRandomLookup(t.closeCtx).run()
}
// filedir: go-ethereum-1.10.2\p2p\discover\v5_udp.go L287
func (t *UDPv5) newRandomLookup(ctx context.Context) *lookup {
    var target enode.ID
    crand.Read(target[:])
    return t.newLookup(ctx, target)
}

func (t *UDPv5) newLookup(ctx context.Context, target enode.ID) *lookup {
    return newLookup(ctx, t.tab, target, func(n *node) ([]*node, error) {
        return t.lookupWorker(n, target)
    })
}

// lookupWorker performs FINDNODE calls against a single node during lookup.
func (t *UDPv5) lookupWorker(destNode *node, target enode.ID) ([]*node, error) {
    var (
        dists = lookupDistances(target, destNode.ID())
        nodes = nodesByDistance{target: target}
        err   error
    )
    var r []*enode.Node
    r, err = t.findnode(unwrapNode(destNode), dists)
    if err == errClosed {
        return nil, err
    }
    for _, n := range r {
        if n.ID() != t.Self().ID() {
            nodes.push(wrapNode(n), findnodeResultLimit)
        }
    }
    return nodes.entries, err
}

// filedir:go-ethereum-1.10.2\p2p\discover\v5_udp.go  L362
// findnode calls FINDNODE on a node and waits for responses.
func (t *UDPv5) findnode(n *enode.Node, distances []uint) ([]*enode.Node, error) {
    resp := t.call(n, v5wire.NodesMsg, &v5wire.Findnode{Distances: distances})
    return t.waitForNodes(resp, distances)
}

// call sends the given call and sets up a handler for response packets (of message type
// responseType). Responses are dispatched to the call's response channel.
func (t *UDPv5) call(node *enode.Node, responseType byte, packet v5wire.Packet) *callV5 {
    c := &callV5{
        node:         node,
        packet:       packet,
        responseType: responseType,
        reqid:        make([]byte, 8),
        ch:           make(chan v5wire.Packet, 1),
        err:          make(chan error, 1),
    }
    // Assign request ID.
    crand.Read(c.reqid)
    packet.SetRequestID(c.reqid)
    // Send call to dispatch.
    select {
    case t.callCh <- c:
    case <-t.closeCtx.Done():
        c.err <- errClosed
    }
    return c
}
```

#### 数据结构

下面的代码是用于管理所有的对等连接的 Server 结构体，它包含了各种字段，用于配置服务器、存储状态和通道以及处理事件和连接，Server 结构体主要包含以下关键字段：

-   Config：配置字段，表示服务器的配置。在服务器运行时不可修改
-   newTransport：用于测试的钩子函数，用于创建传输层对象
-   newPeerHook：用于测试的钩子函数，用于处理新对等点
-   listenFunc：用于测试的钩子函数，用于创建网络监听器

```plain
// filedir:go-ethereum-1.10.2\p2p\server.go  L160
// Server manages all peer connections.
type Server struct {
    // Config fields may not be modified while the server is running.
    Config

    // Hooks for testing. These are useful because we can inhibit
    // the whole protocol stack.
    newTransport func(net.Conn, *ecdsa.PublicKey) transport
    newPeerHook  func(*Peer)
    listenFunc   func(network, addr string) (net.Listener, error)

    lock    sync.Mutex // protects running
    running bool

    listener     net.Listener
    ourHandshake *protoHandshake
    loopWG       sync.WaitGroup // loop, listenLoop
    peerFeed     event.Feed
    log          log.Logger

    nodedb    *enode.DB
    localnode *enode.LocalNode
    ntab      *discover.UDPv4
    DiscV5    *discover.UDPv5
    discmix   *enode.FairMix
    dialsched *dialScheduler

    // Channels into the run loop.
    quit                    chan struct{}
    addtrusted              chan *enode.Node
    removetrusted           chan *enode.Node
    peerOp                  chan peerOpFunc
    peerOpDone              chan struct{}
    delpeer                 chan peerDrop
    checkpointPostHandshake chan *conn
    checkpointAddPeer       chan *conn

    // State of run loop and listenLoop.
    inboundHistory expHeap
}
```

Server 配置 (本地节点秘钥、拨号比率、节点最大链接数、拨号比率、事件记录等)：

```plain
// Config holds Server options.
type Config struct {
    // This field must be set to a valid secp256k1 private key.
    PrivateKey *ecdsa.PrivateKey `toml:"-"`

    // MaxPeers is the maximum number of peers that can be
    // connected. It must be greater than zero.
    MaxPeers int

    // MaxPendingPeers is the maximum number of peers that can be pending in the
    // handshake phase, counted separately for inbound and outbound connections.
    // Zero defaults to preset values.
    MaxPendingPeers int `toml:",omitempty"`

    // DialRatio controls the ratio of inbound to dialed connections.
    // Example: a DialRatio of 2 allows 1/2 of connections to be dialed.
    // Setting DialRatio to zero defaults it to 3.
    DialRatio int `toml:",omitempty"`

    // NoDiscovery can be used to disable the peer discovery mechanism.
    // Disabling is useful for protocol debugging (manual topology).
    NoDiscovery bool

    // DiscoveryV5 specifies whether the new topic-discovery based V5 discovery
    // protocol should be started or not.
    DiscoveryV5 bool `toml:",omitempty"`

    // Name sets the node name of this server.
    // Use common.MakeName to create a name that follows existing conventions.
    Name string `toml:"-"`

    // BootstrapNodes are used to establish connectivity
    // with the rest of the network.
    BootstrapNodes []*enode.Node

    // BootstrapNodesV5 are used to establish connectivity
    // with the rest of the network using the V5 discovery
    // protocol.
    BootstrapNodesV5 []*enode.Node `toml:",omitempty"`

    // Static nodes are used as pre-configured connections which are always
    // maintained and re-connected on disconnects.
    StaticNodes []*enode.Node

    // Trusted nodes are used as pre-configured connections which are always
    // allowed to connect, even above the peer limit.
    TrustedNodes []*enode.Node

    // Connectivity can be restricted to certain IP networks.
    // If this option is set to a non-nil value, only hosts which match one of the
    // IP networks contained in the list are considered.
    NetRestrict *netutil.Netlist `toml:",omitempty"`

    // NodeDatabase is the path to the database containing the previously seen
    // live nodes in the network.
    NodeDatabase string `toml:",omitempty"`

    // Protocols should contain the protocols supported
    // by the server. Matching protocols are launched for
    // each peer.
    Protocols []Protocol `toml:"-"`

    // If ListenAddr is set to a non-nil address, the server
    // will listen for incoming connections.
    //
    // If the port is zero, the operating system will pick a port. The
    // ListenAddr field will be updated with the actual address when
    // the server is started.
    ListenAddr string

    // If set to a non-nil value, the given NAT port mapper
    // is used to make the listening port available to the
    // Internet.
    NAT nat.Interface `toml:",omitempty"`

    // If Dialer is set to a non-nil value, the given Dialer
    // is used to dial outbound peer connections.
    Dialer NodeDialer `toml:"-"`

    // If NoDial is true, the server will not dial any peers.
    NoDial bool `toml:",omitempty"`

    // If EnableMsgEvents is set then the server will emit PeerEvents
    // whenever a message is sent to or received from a peer
    EnableMsgEvents bool

    // Logger is a custom logger to use with the p2p.Server.
    Logger log.Logger `toml:",omitempty"`

    clock mclock.Clock
}
```

#### 增加节点

AddPeer 函数用于将给定的节点添加到静态节点集合中，当对等节点集合中有空余位置时，服务器将尝试连接该节点，如果连接失败则服务器将尝试重新连接该对等节点，addStatic 函数用于将静态拨号候选节点添加到拨号调度器中：

```plain
// filedir:go-ethereum-1.10.2\p2p\server.go  L318
// AddPeer adds the given node to the static node set. When there is room in the peer set,
// the server will connect to the node. If the connection fails for any reason, the server
// will attempt to reconnect the peer.
func (srv *Server) AddPeer(node *enode.Node) {
    srv.dialsched.addStatic(node)
}

// filedir:go-ethereum-1.10.2\p2p\dial.go  L190
// addStatic adds a static dial candidate.
func (d *dialScheduler) addStatic(n *enode.Node) {
    select {
    case d.addStaticCh <- n:
    case <-d.ctx.Done():
    }
}
```

AddTrustedPeer 函数用于新增一个可信任节点：

```plain
// AddTrustedPeer adds the given node to a reserved whitelist which allows the
// node to always connect, even if the slot are full.
func (srv *Server) AddTrustedPeer(node *enode.Node) {
    select {
    case srv.addtrusted <- node:
    case <-srv.quit:
    }
}
```

#### 删除节点

RemovePeer 函数用于从静态节点集合中移除节点，如果该节点当前作为对等节点连接到服务器上，还会将其断开连接，而 removeStatic 函数用于从拨号调度器中移除静态拨号候选节点：

```plain
// filedir:go-ethereum-1.10.2\p2p\server.go  L325
// RemovePeer removes a node from the static node set. It also disconnects from the given
// node if it is currently connected as a peer.
//
// This method blocks until all protocols have exited and the peer is removed. Do not use
// RemovePeer in protocol implementations, call Disconnect on the Peer instead.
func (srv *Server) RemovePeer(node *enode.Node) {
    var (
        ch  chan *PeerEvent
        sub event.Subscription
    )
    // Disconnect the peer on the main loop.
    srv.doPeerOp(func(peers map[enode.ID]*Peer) {
        srv.dialsched.removeStatic(node)
        if peer := peers[node.ID()]; peer != nil {
            ch = make(chan *PeerEvent, 1)
            sub = srv.peerFeed.Subscribe(ch)
            peer.Disconnect(DiscRequested)
        }
    })
    // Wait for the peer connection to end.
    if ch != nil {
        defer sub.Unsubscribe()
        for ev := range ch {
            if ev.Peer == node.ID() && ev.Type == PeerEventTypeDrop {
                return
            }
        }
    }
}

// filedir:go-ethereum-1.10.2\p2p\dial.go  L198
// removeStatic removes a static dial candidate.
func (d *dialScheduler) removeStatic(n *enode.Node) {
    select {
    case d.remStaticCh <- n:
    case <-d.ctx.Done():
    }
}
```

RemoveTrustedPeer 函数用于移除一个可信任节点：

```plain
// RemoveTrustedPeer removes the given node from the trusted peer set.
func (srv *Server) RemoveTrustedPeer(node *enode.Node) {
    select {
    case srv.removetrusted <- node:
    case <-srv.quit:
    }
}
```

由于篇幅原因，后续部分下篇文章继续介绍~
