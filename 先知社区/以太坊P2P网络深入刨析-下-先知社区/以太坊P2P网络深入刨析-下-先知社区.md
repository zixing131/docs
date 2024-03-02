

# 以太坊 P2P 网络深入刨析 (下) - 先知社区

以太坊 P2P 网络深入刨析 (下)

- - -

### 文章前言

本篇文章是"以太坊 P2P 网络深入刨析"的续篇

### 源码分析

#### 服务停用

下面的代码用于停止服务器和所有活动的对等连接，它会阻塞直到所有活动连接都关闭

```plain
// Stop terminates the server and all active peer connections.
// It blocks until all active connections have been closed.
func (srv *Server) Stop() {
    srv.lock.Lock()
    if !srv.running {
        srv.lock.Unlock()
        return
    }
    srv.running = false
    if srv.listener != nil {
        // this unblocks listener Accept
        srv.listener.Close()
    }
    close(srv.quit)
    srv.lock.Unlock()
    srv.loopWG.Wait()
}
```

#### 节点启动

下面的 Start 函数用于启动一个 P2P 节点，获取 srv.lock 的互斥锁，以确保线程安全，随后使用 defer 语句释放锁，检查服务器是否已经在运行，如果是则返回一个错误，表示服务器已经在运行，通过将 srv.running 标志设置为 true，表示服务器正在运行，紧接着配置日志记录器 srv.log，如果未设置则使用默认的根日志记录器 log.Root()，随后配置时钟 srv.clock，如果未设置，则使用系统时钟 mclock.System{}，如果设置了 NoDial 标志且未设置 ListenAddr 监听地址，则记录警告信息，表示 P2P 服务器无法进行拨号或监听

```plain
// filedir:go-ethereum-1.10.2\p2p\server.go  L433
// Start starts running the server.
// Servers can not be re-used after stopping.
func (srv *Server) Start() (err error) {
    srv.lock.Lock()
    defer srv.lock.Unlock()
    if srv.running {
        return errors.New("server already running")
    }
    srv.running = true
    srv.log = srv.Config.Logger
    if srv.log == nil {
        srv.log = log.Root()
    }
    if srv.clock == nil {
        srv.clock = mclock.System{}
    }
    if srv.NoDial && srv.ListenAddr == "" {
        srv.log.Warn("P2P server will be useless, neither dialing nor listening")
    }

    // static fields
    if srv.PrivateKey == nil {
        return errors.New("Server.PrivateKey must be set to a non-nil key")
    }
    if srv.newTransport == nil {
        srv.newTransport = newRLPX
    }
    if srv.listenFunc == nil {
        srv.listenFunc = net.Listen
    }
    srv.quit = make(chan struct{})
    srv.delpeer = make(chan peerDrop)
    srv.checkpointPostHandshake = make(chan *conn)
    srv.checkpointAddPeer = make(chan *conn)
    srv.addtrusted = make(chan *enode.Node)
    srv.removetrusted = make(chan *enode.Node)
    srv.peerOp = make(chan peerOpFunc)
    srv.peerOpDone = make(chan struct{})

    if err := srv.setupLocalNode(); err != nil {
        return err
    }
    if srv.ListenAddr != "" {
        if err := srv.setupListening(); err != nil {
            return err
        }
    }
    if err := srv.setupDiscovery(); err != nil {
        return err
    }
    srv.setupDialScheduler()

    srv.loopWG.Add(1)
    go srv.run()
    return nil
}
```

紧接着会调用 setupLocalNode 来启动一个本地节点，包括握手信息、节点数据库、本地节点的属性和 IP 地址等，这在 P2P 服务器启动过程中的一个重要步骤

```plain
if err := srv.setupLocalNode(); err != nil {
        return err
    } 

// fileidir：go-ethereum-1.10.2\p2p\server.go L490
func (srv *Server) setupLocalNode() error {
    // Create the devp2p handshake.
    pubkey := crypto.FromECDSAPub(&srv.PrivateKey.PublicKey)
    srv.ourHandshake = &protoHandshake{Version: baseProtocolVersion, Name: srv.Name, ID: pubkey[1:]}
    for _, p := range srv.Protocols {
        srv.ourHandshake.Caps = append(srv.ourHandshake.Caps, p.cap())
    }
    sort.Sort(capsByNameAndVersion(srv.ourHandshake.Caps))

    // Create the local node.
    db, err := enode.OpenDB(srv.Config.NodeDatabase)
    if err != nil {
        return err
    }
    srv.nodedb = db
    srv.localnode = enode.NewLocalNode(db, srv.PrivateKey)
    srv.localnode.SetFallbackIP(net.IP{127, 0, 0, 1})
    // TODO: check conflicts
    for _, p := range srv.Protocols {
        for _, e := range p.Attributes {
            srv.localnode.Set(e)
        }
    }
    switch srv.NAT.(type) {
    case nil:
        // No NAT interface, do nothing.
    case nat.ExtIP:
        // ExtIP doesn't block, set the IP right away.
        ip, _ := srv.NAT.ExternalIP()
        srv.localnode.SetStaticIP(ip)
    default:
        // Ask the router about the IP. This takes a while and blocks startup,
        // do it in the background.
        srv.loopWG.Add(1)
        go func() {
            defer srv.loopWG.Done()
            if ip, err := srv.NAT.ExternalIP(); err == nil {
                srv.localnode.SetStaticIP(ip)
            }
        }()
    }
    return nil
}
```

之后建立本地监听：

```plain
if srv.ListenAddr != "" {
        if err := srv.setupListening(); err != nil {
            return err
        }
    }

// fileidir: go-ethereum-1.10.2\p2p\server.go L658
func (srv *Server) setupListening() error {
    // Launch the listener.
    listener, err := srv.listenFunc("tcp", srv.ListenAddr)
    if err != nil {
        return err
    }
    srv.listener = listener
    srv.ListenAddr = listener.Addr().String()

    // Update the local node record and map the TCP listening port if NAT is configured.
    if tcp, ok := listener.Addr().(*net.TCPAddr); ok {
        srv.localnode.Set(enr.TCP(tcp.Port))
        if !tcp.IP.IsLoopback() && srv.NAT != nil {
            srv.loopWG.Add(1)
            go func() {
                nat.Map(srv.NAT, srv.quit, "tcp", tcp.Port, tcp.Port, "ethereum p2p")
                srv.loopWG.Done()
            }()
        }
    }

    srv.loopWG.Add(1)
    go srv.listenLoop()
    return nil
}
// fileidir: go-ethereum-1.10.2\p2p\server.go L828
// listenLoop runs in its own goroutine and accepts
// inbound connections.
func (srv *Server) listenLoop() {
    srv.log.Debug("TCP listener up", "addr", srv.listener.Addr())

    // The slots channel limits accepts of new connections.
    tokens := defaultMaxPendingPeers
    if srv.MaxPendingPeers > 0 {
        tokens = srv.MaxPendingPeers
    }
    slots := make(chan struct{}, tokens)
    for i := 0; i < tokens; i++ {
        slots <- struct{}{}
    }

    // Wait for slots to be returned on exit. This ensures all connection goroutines
    // are down before listenLoop returns.
    defer srv.loopWG.Done()
    defer func() {
        for i := 0; i < cap(slots); i++ {
            <-slots
        }
    }()

    for {
        // Wait for a free slot before accepting.
        <-slots

        var (
            fd      net.Conn
            err     error
            lastLog time.Time
        )
        for {
            fd, err = srv.listener.Accept()
            if netutil.IsTemporaryError(err) {
                if time.Since(lastLog) > 1*time.Second {
                    srv.log.Debug("Temporary read error", "err", err)
                    lastLog = time.Now()
                }
                time.Sleep(time.Millisecond * 200)
                continue
            } else if err != nil {
                srv.log.Debug("Read error", "err", err)
                slots <- struct{}{}
                return
            }
            break
        }

        remoteIP := netutil.AddrIP(fd.RemoteAddr())
        if err := srv.checkInboundConn(remoteIP); err != nil {
            srv.log.Debug("Rejected inbound connection", "addr", fd.RemoteAddr(), "err", err)
            fd.Close()
            slots <- struct{}{}
            continue
        }
        if remoteIP != nil {
            var addr *net.TCPAddr
            if tcp, ok := fd.RemoteAddr().(*net.TCPAddr); ok {
                addr = tcp
            }
            fd = newMeteredConn(fd, true, addr)
            srv.log.Trace("Accepted connection", "addr", fd.RemoteAddr())
        }
        go func() {
            srv.SetupConn(fd, inboundConn, nil)
            slots <- struct{}{}
        }()
    }
}
```

之后配置一个 DiscoveryV5 网络协议，生成节点路由表：

```plain
if err := srv.setupDiscovery(); err != nil {
        return err
    }
// filedir: go-ethereum-1.10.2\p2p\server.go  L535
func (srv *Server) setupDiscovery() error {
    srv.discmix = enode.NewFairMix(discmixTimeout)

    // Add protocol-specific discovery sources.
    added := make(map[string]bool)
    for _, proto := range srv.Protocols {
        if proto.DialCandidates != nil && !added[proto.Name] {
            srv.discmix.AddSource(proto.DialCandidates)
            added[proto.Name] = true
        }
    }

    // Don't listen on UDP endpoint if DHT is disabled.
    if srv.NoDiscovery && !srv.DiscoveryV5 {
        return nil
    }

    addr, err := net.ResolveUDPAddr("udp", srv.ListenAddr)
    if err != nil {
        return err
    }
    conn, err := net.ListenUDP("udp", addr)
    if err != nil {
        return err
    }
    realaddr := conn.LocalAddr().(*net.UDPAddr)
    srv.log.Debug("UDP listener up", "addr", realaddr)
    if srv.NAT != nil {
        if !realaddr.IP.IsLoopback() {
            srv.loopWG.Add(1)
            go func() {
                nat.Map(srv.NAT, srv.quit, "udp", realaddr.Port, realaddr.Port, "ethereum discovery")
                srv.loopWG.Done()
            }()
        }
    }
    srv.localnode.SetFallbackUDP(realaddr.Port)

    // Discovery V4
    var unhandled chan discover.ReadPacket
    var sconn *sharedUDPConn
    if !srv.NoDiscovery {
        if srv.DiscoveryV5 {
            unhandled = make(chan discover.ReadPacket, 100)
            sconn = &sharedUDPConn{conn, unhandled}
        }
        cfg := discover.Config{
            PrivateKey:  srv.PrivateKey,
            NetRestrict: srv.NetRestrict,
            Bootnodes:   srv.BootstrapNodes,
            Unhandled:   unhandled,
            Log:         srv.log,
        }
        ntab, err := discover.ListenV4(conn, srv.localnode, cfg)
        if err != nil {
            return err
        }
        srv.ntab = ntab
        srv.discmix.AddSource(ntab.RandomNodes())
    }

    // Discovery V5
    if srv.DiscoveryV5 {
        cfg := discover.Config{
            PrivateKey:  srv.PrivateKey,
            NetRestrict: srv.NetRestrict,
            Bootnodes:   srv.BootstrapNodesV5,
            Log:         srv.log,
        }
        var err error
        if sconn != nil {
            srv.DiscV5, err = discover.ListenV5(sconn, srv.localnode, cfg)
        } else {
            srv.DiscV5, err = discover.ListenV5(conn, srv.localnode, cfg)
        }
        if err != nil {
            return err
        }
    }
    return nil
}
```

随后调用 setupDialScheduler 启动主动拨号连接过程，然后开一个协程，在其中做 peer 的维护：

```plain
srv.setupDialScheduler()

    srv.loopWG.Add(1)
    go srv.run()
    return nil
}
```

setupDialScheduler 代码如下所示，这里通过 newDialScheduler 来建立连接，参数 discmix 确定了进行主动建立连接时的节点集，它是一个迭代器，同时将 setupConn 连接建立函数传入：

```plain
func (srv *Server) setupDialScheduler() {
    config := dialConfig{
        self:           srv.localnode.ID(),
        maxDialPeers:   srv.maxDialedConns(),
        maxActiveDials: srv.MaxPendingPeers,
        log:            srv.Logger,
        netRestrict:    srv.NetRestrict,
        dialer:         srv.Dialer,
        clock:          srv.clock,
    }
    if srv.ntab != nil {
        config.resolver = srv.ntab
    }
    if config.dialer == nil {
        config.dialer = tcpDialer{&net.Dialer{Timeout: defaultDialTimeout}}
    }
    srv.dialsched = newDialScheduler(config, srv.discmix, srv.SetupConn)
    for _, n := range srv.StaticNodes {
        srv.dialsched.addStatic(n)
    }
}
```

newDialScheduler 函数如下所示，在这里通过 d.readNodes(it) 从迭代器中取得节点，之后通过通道传入 d.loop(it) 中进行连接：

```plain
// filedir:go-ethereum-1.10.2\p2p\dial.go   L162
func newDialScheduler(config dialConfig, it enode.Iterator, setupFunc dialSetupFunc) *dialScheduler {
    d := &dialScheduler{
        dialConfig:  config.withDefaults(),
        setupFunc:   setupFunc,
        dialing:     make(map[enode.ID]*dialTask),
        static:      make(map[enode.ID]*dialTask),
        peers:       make(map[enode.ID]connFlag),
        doneCh:      make(chan *dialTask),
        nodesIn:     make(chan *enode.Node),
        addStaticCh: make(chan *enode.Node),
        remStaticCh: make(chan *enode.Node),
        addPeerCh:   make(chan *conn),
        remPeerCh:   make(chan *conn),
    }
    d.lastStatsLog = d.clock.Now()
    d.ctx, d.cancel = context.WithCancel(context.Background())
    d.wg.Add(2)
    go d.readNodes(it)
    go d.loop(it)
    return d
}
```

loop 函数是拨号器的主要循环过程，代码如下所示：

```plain
// loop is the main loop of the dialer.
func (d *dialScheduler) loop(it enode.Iterator) {
    var (
        nodesCh    chan *enode.Node
        historyExp = make(chan struct{}, 1)
    )

loop:
    for {
        // Launch new dials if slots are available.  如果插槽可用则启动拨号连接 (插槽相当于空间，拨号连接的数目限制)
        slots := d.freeDialSlots()
        slots -= d.startStaticDials(slots)
        if slots > 0 {
            nodesCh = d.nodesIn
        } else {
            nodesCh = nil
        }
        d.rearmHistoryTimer(historyExp)
        d.logStats()

        select {
        case node := <-nodesCh:       // 接收到readnode中发来消息，迭代器中的所有节点
            if err := d.checkDial(node); err != nil {
                d.log.Trace("Discarding dial candidate", "id", node.ID(), "ip", node.IP(), "reason", err)
            } else {
                d.startDial(newDialTask(node, dynDialedConn))   //启动拨号
            }

        case task := <-d.doneCh:    // 拨号完成
            id := task.dest.ID()
            delete(d.dialing, id)
            d.updateStaticPool(id)
            d.doneSinceLastLog++

        case c := <-d.addPeerCh:    
            if c.is(dynDialedConn) || c.is(staticDialedConn) {
                d.dialPeers++
            }
            id := c.node.ID()
            d.peers[id] = c.flags
            // Remove from static pool because the node is now connected.
            task := d.static[id]
            if task != nil && task.staticPoolIndex >= 0 {
                d.removeFromStaticPool(task.staticPoolIndex)
            }
            // TODO: cancel dials to connected peers

        case c := <-d.remPeerCh:
            if c.is(dynDialedConn) || c.is(staticDialedConn) {
                d.dialPeers--
            }
            delete(d.peers, c.node.ID())
            d.updateStaticPool(c.node.ID())

        case node := <-d.addStaticCh:
            id := node.ID()
            _, exists := d.static[id]
            d.log.Trace("Adding static node", "id", id, "ip", node.IP(), "added", !exists)
            if exists {
                continue loop
            }
            task := newDialTask(node, staticDialedConn)
            d.static[id] = task
            if d.checkDial(node) == nil {
                d.addToStaticPool(task)
            }

        case node := <-d.remStaticCh:
            id := node.ID()
            task := d.static[id]
            d.log.Trace("Removing static node", "id", id, "ok", task != nil)
            if task != nil {
                delete(d.static, id)
                if task.staticPoolIndex >= 0 {
                    d.removeFromStaticPool(task.staticPoolIndex)
                }
            }

        case <-historyExp:
            d.expireHistory()

        case <-d.ctx.Done():
            it.Close()
            break loop
        }
    }

    d.stopHistoryTimer(historyExp)
    for range d.dialing {
        <-d.doneCh
    }
    d.wg.Done()
}
```

startDial 函数代码如下所示，它在单独的 goroutine 中运行给定的拨号任务：

```plain
// filedir:go-ethereum-1.10.2\p2p\dial.go  L452
// startDial runs the given dial task in a separate goroutine.
func (d *dialScheduler) startDial(task *dialTask) {
    d.log.Trace("Starting p2p dial", "id", task.dest.ID(), "ip", task.dest.IP(), "flag", task.flags)
    hkey := string(task.dest.ID().Bytes())
    d.history.add(hkey, d.clock.Now().Add(dialHistoryExpiration))
    d.dialing[task.dest.ID()] = task
    go func() {
        task.run(d)
        d.doneCh <- task
    }()
}
```

run 函数如下所示，在这里首先会查找 IP，之后调用 dial 进行拨号：

```plain
// filedir：go-ethereum-1.10.2\p2p\dial.go   L482
func (t *dialTask) run(d *dialScheduler) {
    if t.needResolve() && !t.resolve(d) {
        return
    }

    err := t.dial(d, t.dest)
    if err != nil {
        // For static nodes, resolve one more time if dialing fails.
        if _, ok := err.(*dialError); ok && t.flags&staticDialedConn != 0 {
            if t.resolve(d) {
                t.dial(d, t.dest)
            }
        }
    }
}
```

dial 方法如下所示，在这里主动拨号，带有目的地址，在这里通过调用 setupFunc 所代表的 setupConn 函数来建立连接：

```plain
// filedir:go-ethereum-1.10.2\p2p\dial.go  L536
// dial performs the actual connection attempt.
func (t *dialTask) dial(d *dialScheduler, dest *enode.Node) error {
    fd, err := d.dialer.Dial(d.ctx, t.dest)
    if err != nil {
        d.log.Trace("Dial error", "id", t.dest.ID(), "addr", nodeAddr(t.dest), "conn", t.flags, "err", cleanupDialErr(err))
        return &dialError{err}
    }
    mfd := newMeteredConn(fd, false, &net.TCPAddr{IP: dest.IP(), Port: dest.TCP()})
    return d.setupFunc(mfd, t.flags, dest)
}
```

#### 服务监听

在上面的服务启动过程中有一个 setupListening 函数，该函数用于监听事件，具体代码如下所示：

```plain
func (srv *Server) setupListening() error {
    // Launch the listener.
    listener, err := srv.listenFunc("tcp", srv.ListenAddr)
    if err != nil {
        return err
    }
    srv.listener = listener
    srv.ListenAddr = listener.Addr().String()

    // Update the local node record and map the TCP listening port if NAT is configured.
    if tcp, ok := listener.Addr().(*net.TCPAddr); ok {
        srv.localnode.Set(enr.TCP(tcp.Port))
        if !tcp.IP.IsLoopback() && srv.NAT != nil {
            srv.loopWG.Add(1)
            go func() {
                nat.Map(srv.NAT, srv.quit, "tcp", tcp.Port, tcp.Port, "ethereum p2p")
                srv.loopWG.Done()
            }()
        }
    }

    srv.loopWG.Add(1)
    go srv.listenLoop()
    return nil
}
```

在上述代码中又调用了一个 srv.listenLoop()，该函数是一个死循环的 goroutine，它会监听端口并接收外部的请求：

```plain
// listenLoop runs in its own goroutine and accepts
// inbound connections.
func (srv *Server) listenLoop() {
    srv.log.Debug("TCP listener up", "addr", srv.listener.Addr())

    // The slots channel limits accepts of new connections.
    tokens := defaultMaxPendingPeers
    if srv.MaxPendingPeers > 0 {
        tokens = srv.MaxPendingPeers
    }
    slots := make(chan struct{}, tokens)
    for i := 0; i < tokens; i++ {
        slots <- struct{}{}
    }

    // Wait for slots to be returned on exit. This ensures all connection goroutines
    // are down before listenLoop returns.
    defer srv.loopWG.Done()
    defer func() {
        for i := 0; i < cap(slots); i++ {
            <-slots
        }
    }()

    for {
        // Wait for a free slot before accepting.
        <-slots

        var (
            fd      net.Conn
            err     error
            lastLog time.Time
        )
        for {
            fd, err = srv.listener.Accept()
            if netutil.IsTemporaryError(err) {
                if time.Since(lastLog) > 1*time.Second {
                    srv.log.Debug("Temporary read error", "err", err)
                    lastLog = time.Now()
                }
                time.Sleep(time.Millisecond * 200)
                continue
            } else if err != nil {
                srv.log.Debug("Read error", "err", err)
                slots <- struct{}{}
                return
            }
            break
        }

        remoteIP := netutil.AddrIP(fd.RemoteAddr())
        if err := srv.checkInboundConn(remoteIP); err != nil {
            srv.log.Debug("Rejected inbound connection", "addr", fd.RemoteAddr(), "err", err)
            fd.Close()
            slots <- struct{}{}
            continue
        }
        if remoteIP != nil {
            var addr *net.TCPAddr
            if tcp, ok := fd.RemoteAddr().(*net.TCPAddr); ok {
                addr = tcp
            }
            fd = newMeteredConn(fd, true, addr)
            srv.log.Trace("Accepted connection", "addr", fd.RemoteAddr())
        }
        go func() {
            srv.SetupConn(fd, inboundConn, nil)
            slots <- struct{}{}
        }()
    }
}
```

这里的 SetupConn 主要执行执行握手协议，并尝试把链接创建为一个 peer 对象：

```plain
// SetupConn runs the handshakes and attempts to add the connection
// as a peer. It returns when the connection has been added as a peer
// or the handshakes have failed.
func (srv *Server) SetupConn(fd net.Conn, flags connFlag, dialDest *enode.Node) error {
    c := &conn{fd: fd, flags: flags, cont: make(chan error)}
    if dialDest == nil {
        c.transport = srv.newTransport(fd, nil)
    } else {
        c.transport = srv.newTransport(fd, dialDest.Pubkey())
    }

    err := srv.setupConn(c, flags, dialDest)
    if err != nil {
        c.close(err)
    }
    return err
}
```

在上述代码中又去调用了 srv.setupConn(c, flags, dialDest) 函数，该函数用于执行握手协议

```plain
func (srv *Server) setupConn(c *conn, flags connFlag, dialDest *enode.Node) error {
    // Prevent leftover pending conns from entering the handshake.
    srv.lock.Lock()
    running := srv.running
    srv.lock.Unlock()
    if !running {
        return errServerStopped
    }

    // If dialing, figure out the remote public key.
    var dialPubkey *ecdsa.PublicKey
    if dialDest != nil {    //  dest=nil 被动连接，dest!=nil 主动连接诶
        dialPubkey = new(ecdsa.PublicKey)
        if err := dialDest.Load((*enode.Secp256k1)(dialPubkey)); err != nil {
            err = errors.New("dial destination doesn't have a secp256k1 public key")
            srv.log.Trace("Setting up connection failed", "addr", c.fd.RemoteAddr(), "conn", c.flags, "err", err)
            return err
        }
    }

    // Run the RLPx handshake.
    remotePubkey, err := c.doEncHandshake(srv.PrivateKey)    // 公钥交换，确定共享秘钥RLPx层面的握手一来一去
    if err != nil {
        srv.log.Trace("Failed RLPx handshake", "addr", c.fd.RemoteAddr(), "conn", c.flags, "err", err)
        return err
    }
    if dialDest != nil {
        c.node = dialDest
    } else {
        c.node = nodeFromConn(remotePubkey, c.fd)
    }
    clog := srv.log.New("id", c.node.ID(), "addr", c.fd.RemoteAddr(), "conn", c.flags)
    err = srv.checkpoint(c, srv.checkpointPostHandshake)
    if err != nil {
        clog.Trace("Rejected peer", "err", err)
        return err
    }

    // Run the capability negotiation handshake.
    phs, err := c.doProtoHandshake(srv.ourHandshake)  // 进行协议层面的握手,也即p2p握手，一来一去
    if err != nil {
        clog.Trace("Failed p2p handshake", "err", err)
        return err
    }
    if id := c.node.ID(); !bytes.Equal(crypto.Keccak256(phs.ID), id[:]) {
        clog.Trace("Wrong devp2p handshake identity", "phsid", hex.EncodeToString(phs.ID))
        return DiscUnexpectedIdentity
    }
    c.caps, c.name = phs.Caps, phs.Name
    err = srv.checkpoint(c, srv.checkpointAddPeer)  // 状态校验
    if err != nil {
        clog.Trace("Rejected peer", "err", err)
        return err
    }

    return nil
}
```

秘钥握手通过 deEncHandshake 函数实现，在函数之中调用了 Handshake() 函数：

```plain
// filedir:go-ethereum-1.10.2\p2p\transport.go  L123
func (t *rlpxTransport) doEncHandshake(prv *ecdsa.PrivateKey) (*ecdsa.PublicKey, error) {
    t.conn.SetDeadline(time.Now().Add(handshakeTimeout))
    return t.conn.Handshake(prv)
}
```

Handshake 代码如下所示，在这里会根据是主动握手还是被动握手来进行执行对应的握手逻辑：

```plain
// filedir:go-ethereum-1.10.2\p2p\rlpx\rlpx.go   L253
// Handshake performs the handshake. This must be called before any data is written
// or read from the connection.
func (c *Conn) Handshake(prv *ecdsa.PrivateKey) (*ecdsa.PublicKey, error) {
    var (
        sec Secrets
        err error
    )
    if c.dialDest != nil {   //主动握手
        sec, err = initiatorEncHandshake(c.conn, prv, c.dialDest)  //主动发起秘钥验证握手结束，确定共享秘钥
    } else { // 被动握手
        sec, err = receiverEncHandshake(c.conn, prv)
    }
    if err != nil {
        return nil, err
    }
    c.InitWithSecrets(sec)
    return sec.remote, err
}
```

主动发起握手过程过程如下，在这里会调用 makeAuthMsg 来生成 Auth 身份信息，包含签名，随机 nonce 生成的与签名对应的公钥和版本号，之后调用 sealEIP8 方法进行 rlpx 编码，之后发起加密握手，之后接收返回的 authResp 消息，并验证解密，获取对方公钥，之后生成 AES，MAC：

```plain
// filedir:go-ethereum-1.10.2\p2p\rlpx\rlpx.go  L477
// initiatorEncHandshake negotiates a session token on conn.
// it should be called on the dialing side of the connection.
//
// prv is the local client's private key.
func initiatorEncHandshake(conn io.ReadWriter, prv *ecdsa.PrivateKey, remote *ecdsa.PublicKey) (s Secrets, err error) {
    h := &encHandshake{initiator: true, remote: ecies.ImportECDSAPublic(remote)}
    authMsg, err := h.makeAuthMsg(prv)
    if err != nil {
        return s, err
    }
    authPacket, err := sealEIP8(authMsg, h)
    if err != nil {
        return s, err
    }

    if _, err = conn.Write(authPacket); err != nil {
        return s, err
    }

    authRespMsg := new(authRespV4)
    authRespPacket, err := readHandshakeMsg(authRespMsg, encAuthRespLen, prv, conn)
    if err != nil {
        return s, err
    }
    if err := h.handleAuthResp(authRespMsg); err != nil {
        return s, err
    }
    return h.secrets(authPacket, authRespPacket)
}
```

receiverEncHandshake 如下所示，和 initiatorEncHandshake 相差无几：

```plain
// receiverEncHandshake negotiates a session token on conn.
// it should be called on the listening side of the connection.
//
// prv is the local client's private key.
func receiverEncHandshake(conn io.ReadWriter, prv *ecdsa.PrivateKey) (s Secrets, err error) {
    authMsg := new(authMsgV4)
    authPacket, err := readHandshakeMsg(authMsg, encAuthMsgLen, prv, conn)
    if err != nil {
        return s, err
    }
    h := new(encHandshake)
    if err := h.handleAuthMsg(authMsg, prv); err != nil {
        return s, err
    }

    authRespMsg, err := h.makeAuthResp()
    if err != nil {
        return s, err
    }
    var authRespPacket []byte
    if authMsg.gotPlain {
        authRespPacket, err = authRespMsg.sealPlain(h)
    } else {
        authRespPacket, err = sealEIP8(authRespMsg, h)
    }
    if err != nil {
        return s, err
    }
    if _, err = conn.Write(authRespPacket); err != nil {
        return s, err
    }
    return h.secrets(authPacket, authRespPacket)
}
```

之后通过 doProtoHandshake 来完成协议握手操作，在这里调用 send 发送一次握手操作，之后通过 readProtocolHandshake 来读取返回信息，之后进行检查：

```plain
func (t *rlpxTransport) doProtoHandshake(our *protoHandshake) (their *protoHandshake, err error) {
    // Writing our handshake happens concurrently, we prefer
    // returning the handshake read error. If the remote side
    // disconnects us early with a valid reason, we should return it
    // as the error so it can be tracked elsewhere.
    werr := make(chan error, 1)
    go func() { werr <- Send(t, handshakeMsg, our) }()   
    if their, err = readProtocolHandshake(t); err != nil {
        <-werr // make sure the write terminates too
        return nil, err
    }
    if err := <-werr; err != nil {
        return nil, fmt.Errorf("write error: %v", err)
    }
    // If the protocol version supports Snappy encoding, upgrade immediately
    t.conn.SetSnappy(their.Version >= snappyProtocolVersion)

    return their, nil
}
```

#### 循环监听

下面的 run() 方法是 P2P 服务器的主要逻辑循环，在此循环中处理各种事件，包括信任节点的添加和移除、对等节点操作、握手后的检查、添加对等节点、对等节点的断开等，该方法还负责在服务器停止时执行清理工作，包括终止发现功能和断开所有对等节点的连接

```plain
// run is the main loop of the server.
func (srv *Server) run() {
    srv.log.Info("Started P2P networking", "self", srv.localnode.Node().URLv4())
    defer srv.loopWG.Done()
    defer srv.nodedb.Close()
    defer srv.discmix.Close()
    defer srv.dialsched.stop()

    var (
        peers        = make(map[enode.ID]*Peer)
        inboundCount = 0
        trusted      = make(map[enode.ID]bool, len(srv.TrustedNodes))
    )
    // Put trusted nodes into a map to speed up checks.
    // Trusted peers are loaded on startup or added via AddTrustedPeer RPC.
    for _, n := range srv.TrustedNodes {
        trusted[n.ID()] = true
    }

running:
    for {
        select {
        case <-srv.quit:
            // The server was stopped. Run the cleanup logic.
            break running

        case n := <-srv.addtrusted:
            // This channel is used by AddTrustedPeer to add a node
            // to the trusted node set.
            srv.log.Trace("Adding trusted node", "node", n)
            trusted[n.ID()] = true
            if p, ok := peers[n.ID()]; ok {
                p.rw.set(trustedConn, true)
            }

        case n := <-srv.removetrusted:
            // This channel is used by RemoveTrustedPeer to remove a node
            // from the trusted node set.
            srv.log.Trace("Removing trusted node", "node", n)
            delete(trusted, n.ID())
            if p, ok := peers[n.ID()]; ok {
                p.rw.set(trustedConn, false)
            }

        case op := <-srv.peerOp:
            // This channel is used by Peers and PeerCount.
            op(peers)
            srv.peerOpDone <- struct{}{}

        case c := <-srv.checkpointPostHandshake:
            // A connection has passed the encryption handshake so
            // the remote identity is known (but hasn't been verified yet).
            if trusted[c.node.ID()] {
                // Ensure that the trusted flag is set before checking against MaxPeers.
                c.flags |= trustedConn
            }
            // TODO: track in-progress inbound node IDs (pre-Peer) to avoid dialing them.
            c.cont <- srv.postHandshakeChecks(peers, inboundCount, c)

        case c := <-srv.checkpointAddPeer:
            // At this point the connection is past the protocol handshake.
            // Its capabilities are known and the remote identity is verified.
            err := srv.addPeerChecks(peers, inboundCount, c)
            if err == nil {
                // The handshakes are done and it passed all checks.
                p := srv.launchPeer(c)
                peers[c.node.ID()] = p
                srv.log.Debug("Adding p2p peer", "peercount", len(peers), "id", p.ID(), "conn", c.flags, "addr", p.RemoteAddr(), "name", p.Name())
                srv.dialsched.peerAdded(c)
                if p.Inbound() {
                    inboundCount++
                }
            }
            c.cont <- err

        case pd := <-srv.delpeer:
            // A peer disconnected.
            d := common.PrettyDuration(mclock.Now() - pd.created)
            delete(peers, pd.ID())
            srv.log.Debug("Removing p2p peer", "peercount", len(peers), "id", pd.ID(), "duration", d, "req", pd.requested, "err", pd.err)
            srv.dialsched.peerRemoved(pd.rw)
            if pd.Inbound() {
                inboundCount--
            }
        }
    }

    srv.log.Trace("P2P networking is spinning down")

    // Terminate discovery. If there is a running lookup it will terminate soon.
    if srv.ntab != nil {
        srv.ntab.Close()
    }
    if srv.DiscV5 != nil {
        srv.DiscV5.Close()
    }
    // Disconnect all peers.
    for _, p := range peers {
        p.Disconnect(DiscQuitting)
    }
    // Wait for peers to shut down. Pending connections and tasks are
    // not handled here and will terminate soon-ish because srv.quit
    // is closed.
    for len(peers) > 0 {
        p := <-srv.delpeer
        p.log.Trace("<-delpeer (spindown)")
        delete(peers, p.ID())
    }
}
```

#### 节点信息

NodeInfo() 方法用于收集关于主机的元数据并返回一个 NodeInfo 对象，首先获取主机的通用节点信息，例如：名称、Enode URL、节点 ID、IP 地址和监听地址，设置发现端口和监听端口，获取主机的 ENR(Ethereum Node Record) 字符串， 
遍历正在运行的协议，收集每个协议的节点信息，并将其存储在 info.Protocols 映射中，返回包含所有信息的 NodeInfo 对象

```plain
// NodeInfo gathers and returns a collection of metadata known about the host.
func (srv *Server) NodeInfo() *NodeInfo {
    // Gather and assemble the generic node infos
    node := srv.Self()
    info := &NodeInfo{
        Name:       srv.Name,
        Enode:      node.URLv4(),
        ID:         node.ID().String(),
        IP:         node.IP().String(),
        ListenAddr: srv.ListenAddr,
        Protocols:  make(map[string]interface{}),
    }
    info.Ports.Discovery = node.UDP()
    info.Ports.Listener = node.TCP()
    info.ENR = node.String()

    // Gather all the running protocol infos (only once per protocol type)
    for _, proto := range srv.Protocols {
        if _, ok := info.Protocols[proto.Name]; !ok {
            nodeInfo := interface{}("unknown")
            if query := proto.NodeInfo; query != nil {
                nodeInfo = proto.NodeInfo()
            }
            info.Protocols[proto.Name] = nodeInfo
        }
    }
    return info
}
```

PeersInfo() 方法用于返回一个描述已连接对等节点的元数据对象数组具体步骤如下：

-   创建一个空的 PeerInfo 对象数组 infos。
-   遍历服务器中的所有对等节点，对于每个对等节点，获取其元数据信息（通过调用 peer.Info() 方法）并将其添加到 infos 数组中。
-   对 infos 数组按照节点标识符进行字母顺序排序。
-   返回排序后的 infos 数组

```plain
// PeersInfo returns an array of metadata objects describing connected peers.
func (srv *Server) PeersInfo() []*PeerInfo {
    // Gather all the generic and sub-protocol specific infos
    infos := make([]*PeerInfo, 0, srv.PeerCount())
    for _, peer := range srv.Peers() {
        if peer != nil {
            infos = append(infos, peer.Info())
        }
    }
    // Sort the result array alphabetically by node identifier
    for i := 0; i < len(infos); i++ {
        for j := i + 1; j < len(infos); j++ {
            if infos[i].ID > infos[j].ID {
                infos[i], infos[j] = infos[j], infos[i]
            }
        }
    }
    return infos
}
```

#### 请求处理

下面的 peer.run 函数用于启动对等节点的运行循环，处理读取和写入操作，处理协议处理程序的启动和错误处理并在适当的时候断开连接

```plain
func (p *Peer) run() (remoteRequested bool, err error) {
    var (
        writeStart = make(chan struct{}, 1)
        writeErr   = make(chan error, 1)
        readErr    = make(chan error, 1)
        reason     DiscReason // sent to the peer
    )
    p.wg.Add(2)
    go p.readLoop(readErr)
    go p.pingLoop()

    // Start all protocol handlers.
    writeStart <- struct{}{}
    p.startProtocols(writeStart, writeErr)

    // Wait for an error or disconnect.
loop:
    for {
        select {
        case err = <-writeErr:
            // A write finished. Allow the next write to start if
            // there was no error.
            if err != nil {
                reason = DiscNetworkError
                break loop
            }
            writeStart <- struct{}{}
        case err = <-readErr:
            if r, ok := err.(DiscReason); ok {
                remoteRequested = true
                reason = r
            } else {
                reason = DiscNetworkError
            }
            break loop
        case err = <-p.protoErr:
            reason = discReasonForError(err)
            break loop
        case err = <-p.disc:
            reason = discReasonForError(err)
            break loop
        }
    }

    close(p.closed)
    p.rw.close(reason)
    p.wg.Wait()
    return remoteRequested, err
}
```

从上述代码中可以看到函数的开头首先定义了一些局部变量，之后启用了两个协程，一个是 readLoop，它通过调用 ReadMsg() 读取 msg，之后又通过调用 peer.handle(msg) 来处理 msg：

```plain
func (p *Peer) readLoop(errc chan<- error) {
    defer p.wg.Done()
    for {
        msg, err := p.rw.ReadMsg()
        if err != nil {
            errc <- err
            return
        }
        msg.ReceivedAt = time.Now()
        if err = p.handle(msg); err != nil {
            errc <- err
            return
        }
    }
}
```

如果 msg 是 pingMsg，则发送一个 pong 回应，如果 msg 与下述特殊情况不相匹配则将 msg 交给 proto.in 通道，等待 protocolManager.handleMsg() 从通道中取出

```plain
func (p *Peer) handle(msg Msg) error {
    switch {
    case msg.Code == pingMsg:
        msg.Discard()
        go SendItems(p.rw, pongMsg)
    case msg.Code == discMsg:
        var reason [1]DiscReason
        // This is the last message. We don't need to discard or
        // check errors because, the connection will be closed after it.
        rlp.Decode(msg.Payload, &reason)
        return reason[0]
    case msg.Code < baseProtocolLength:
        // ignore other base protocol messages
        return msg.Discard()
    default:
        // it's a subprotocol message
        proto, err := p.getProto(msg.Code)
        if err != nil {
            return fmt.Errorf("msg code out of range: %v", msg.Code)
        }
        if metrics.Enabled {
            m := fmt.Sprintf("%s/%s/%d/%#02x", ingressMeterName, proto.Name, proto.Version, msg.Code-proto.offset)
            metrics.GetOrRegisterMeter(m, nil).Mark(int64(msg.meterSize))
            metrics.GetOrRegisterMeter(m+"/packets", nil).Mark(1)
        }
        select {
        case proto.in <- msg:
            return nil
        case <-p.closed:
            return io.EOF
        }
    }
    return nil
}
```

另一个协程是 pingLoop，它主要通过调用 SendItems(p.rw, pingMsg) 来发起 ping 请求：

```plain
func (p *Peer) pingLoop() {
    ping := time.NewTimer(pingInterval)
    defer p.wg.Done()
    defer ping.Stop()
    for {
        select {
        case <-ping.C:
            if err := SendItems(p.rw, pingMsg); err != nil {
                p.protoErr <- err
                return
            }
            ping.Reset(pingInterval)
        case <-p.closed:
            return
        }
    }
}
```

之后调用 starProtocols() 函数让协议运行起来：

```plain
func (p *Peer) startProtocols(writeStart <-chan struct{}, writeErr chan<- error) {
    p.wg.Add(len(p.running))
    for _, proto := range p.running {
        proto := proto
        proto.closed = p.closed
        proto.wstart = writeStart
        proto.werr = writeErr
        var rw MsgReadWriter = proto
        if p.events != nil {
            rw = newMsgEventer(rw, p.events, p.ID(), proto.Name, p.Info().Network.RemoteAddress, p.Info().Network.LocalAddress)
        }
        p.log.Trace(fmt.Sprintf("Starting protocol %s/%d", proto.Name, proto.Version))
        go func() {
            defer p.wg.Done()
            err := proto.Run(p, rw)
            if err == nil {
                p.log.Trace(fmt.Sprintf("Protocol %s/%d returned", proto.Name, proto.Version))
                err = errProtocolReturned
            } else if err != io.EOF {
                p.log.Trace(fmt.Sprintf("Protocol %s/%d failed", proto.Name, proto.Version), "err", err)
            }
            p.protoErr <- err
        }()
    }
}
```

最后通过一个 loop 循环来处理错误或者断开连接等操作：

```plain
// Wait for an error or disconnect.
loop:
    for {
        select {
        case err = <-writeErr:
            // A write finished. Allow the next write to start if
            // there was no error.
            if err != nil {
                reason = DiscNetworkError
                break loop
            }
            writeStart <- struct{}{}
        case err = <-readErr:
            if r, ok := err.(DiscReason); ok {
                remoteRequested = true
                reason = r
            } else {
                reason = DiscNetworkError
            }
            break loop
        case err = <-p.protoErr:
            reason = discReasonForError(err)
            break loop
        case err = <-p.disc:
            reason = discReasonForError(err)
            break loop
        }
    }

    close(p.closed)
    p.rw.close(reason)
    p.wg.Wait()
    return remoteRequested, err
```

#### 节点更新

下面的一系列函数提供了对 Ping 和 Pong 操作的时间跟踪和计数的功能，通过将相关时间戳存储在数据库中，可以在需要时检索和更新这些值

```plain
// LastPingReceived retrieves the time of the last ping packet received from
// a remote node.
func (db *DB) LastPingReceived(id ID, ip net.IP) time.Time {
    if ip = ip.To16(); ip == nil {
        return time.Time{}
    }
    return time.Unix(db.fetchInt64(nodeItemKey(id, ip, dbNodePing)), 0)
}

// UpdateLastPingReceived updates the last time we tried contacting a remote node.
func (db *DB) UpdateLastPingReceived(id ID, ip net.IP, instance time.Time) error {
    if ip = ip.To16(); ip == nil {
        return errInvalidIP
    }
    return db.storeInt64(nodeItemKey(id, ip, dbNodePing), instance.Unix())
}

// LastPongReceived retrieves the time of the last successful pong from remote node.
func (db *DB) LastPongReceived(id ID, ip net.IP) time.Time {
    if ip = ip.To16(); ip == nil {
        return time.Time{}
    }
    // Launch expirer
    db.ensureExpirer()
    return time.Unix(db.fetchInt64(nodeItemKey(id, ip, dbNodePong)), 0)
}

// UpdateLastPongReceived updates the last pong time of a node.
func (db *DB) UpdateLastPongReceived(id ID, ip net.IP, instance time.Time) error {
    if ip = ip.To16(); ip == nil {
        return errInvalidIP
    }
    return db.storeInt64(nodeItemKey(id, ip, dbNodePong), instance.Unix())
}

// FindFails retrieves the number of findnode failures since bonding.
func (db *DB) FindFails(id ID, ip net.IP) int {
    if ip = ip.To16(); ip == nil {
        return 0
    }
    return int(db.fetchInt64(nodeItemKey(id, ip, dbNodeFindFails)))
}

// UpdateFindFails updates the number of findnode failures since bonding.
func (db *DB) UpdateFindFails(id ID, ip net.IP, fails int) error {
    if ip = ip.To16(); ip == nil {
        return errInvalidIP
    }
    return db.storeInt64(nodeItemKey(id, ip, dbNodeFindFails), int64(fails))
}

// FindFailsV5 retrieves the discv5 findnode failure counter.
func (db *DB) FindFailsV5(id ID, ip net.IP) int {
    if ip = ip.To16(); ip == nil {
        return 0
    }
    return int(db.fetchInt64(v5Key(id, ip, dbNodeFindFails)))
}

// UpdateFindFailsV5 stores the discv5 findnode failure counter.
func (db *DB) UpdateFindFailsV5(id ID, ip net.IP, fails int) error {
    if ip = ip.To16(); ip == nil {
        return errInvalidIP
    }
    return db.storeInt64(v5Key(id, ip, dbNodeFindFails), int64(fails))
}
```

#### 节点筛选

通过使用随机起始位置和循环遍历数据库中的节点检索到一些随机的节点作为种子节点，用于引导启动和增加节点的连接和发现机会

```plain
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

// reads the next node record from the iterator, skipping over other
// database entries.
func nextNode(it iterator.Iterator) *Node {
    for end := false; !end; end = !it.Next() {
        id, rest := splitNodeKey(it.Key())
        if string(rest) != dbDiscoverRoot {
            continue
        }
        return mustDecodeNode(id[:], it.Value())
    }
    return nil
}
```

### 安全风险

#### P2P 协议的异形攻击

##### 基本介绍

P2P 协议的异形攻击是指利用协议的设计缺陷或实现漏洞以非常规的方式对 P2P 网络进行攻击，这些攻击利用了 P2P 网络的分散性和去中心化特点以及节点之间的直接通信方式从而破坏网络的正常运行、危害节点的安全或者干扰用户的体验

##### 攻击方式

下面是几种常见的 P2P 协议异形攻击：

-   Sybil 攻击：Sybil 攻击是指攻击者通过创建大量虚假身份 (称为 Sybil 节点) 来欺骗 P2P 网络中的其他节点，这些虚假身份可能会伪装成多个独立节点从而影响网络资源分配、协议决策或数据传输
-   Eclipse 攻击：Eclipse 攻击旨在将特定节点完全隔离在网络外部，使其无法与其他节点进行通信，攻击者通过控制大量的恶意节点使目标节点的入站和出站连接只与攻击者的节点建立连接从而阻断目标节点与其他正常节点的通信
-   混淆攻击：混淆攻击旨在通过发送大量无效或混淆的数据包来消耗网络带宽、降低网络性能或迷惑节点，攻击者可能发送大量伪造的数据包、具有错误或无效信息的数据包或者发送大量重复的数据包以使网络拥塞或对节点进行拒绝服务攻击
-   数据污染攻击：数据污染攻击是指攻击者在 P2P 网络中故意篡改或修改数据，以影响网络的完整性和可靠性，攻击者可以在传输过程中修改数据包、篡改共享文件或操纵数据路由，导致节点接收到被篡改的数据，从而影响用户的文件完整性和可信度
-   自私挖矿攻击：自私挖矿攻击是通过自私地保留挖矿奖励而不与其他节点共享的方式来破坏 P2P 网络中的共识机制，攻击者可以伪装成多个节点并选择性地将他们的挖矿结果广播给其他节点，以便获得更多的挖矿奖励，从而破坏公平性和安全性

#### P2P 女巫攻击

##### 基本介绍

P2P 女巫攻击 (P2P Sybil Attack) 是一种针对 P2P 网络的攻击方式，其中攻击者通过创建大量虚假身份 (称为女巫节点）来欺骗其他节点，影响网络的性能、可靠性和安全性。在 P2P 网络中节点之间的通信和协作是基于彼此之间的信任建立的，每个节点都有一个身份标识，用于区分和验证其与其他节点的通信，然而 P2P 女巫攻击者利用网络中的匿名性和去中心化特点，通过创建大量的虚假身份来干扰网络的正常运行

#### 具体流程

-   创建虚假身份：攻击者开始创建大量的虚假身份，也就是女巫节点，这些身份通常包括虚假的 IP 地址、端口号和唯一标识符以使它们看起来像是独立的节点
-   加入 P2P 网络：攻击者将这些女巫节点逐步引入目标 P2P 网络中，它们可能通过伪造的身份信息或者控制多个节点来实现加入并与其他正常节点建立连接
-   虚假身份扩散：一旦女巫节点成功加入 P2P 网络，它们开始扩散和传播自己的虚假身份，这可能包括向邻居节点广播自己的存在，与其他节点建立连接并将自己伪装成多个独立的节点
-   影响网络性能：女巫节点开始占用网络资源，例如：带宽、存储和处理能力，它们可能发送大量无效的请求、广播虚假的信息或者参与网络协议的决策过程，从而影响网络的性能和可靠性
-   干扰数据传输：女巫节点可以选择性地干扰数据传输过程，它们可能修改或篡改传输的数据，向其他节点发送错误的数据包或者通过发送大量的重复数据包来产生网络拥塞
-   破坏共识机制：如果 P2P 网络使用共识算法进行决策，女巫节点可以参与其中并产生不公正的结果，它们可能投票或表达偏向以影响协议的决策过程，从而破坏网络的公正性和安全性

#### 拒绝服务攻击

##### 基本介绍

P2P 网络中的拒绝服务攻击 (Denial-of-Service Attack，DoS 攻击) 是一种旨在使网络资源无法提供正常服务的攻击手段。在 P2P 网络中攻击者通过向目标节点发送大量的请求或者利用网络协议的弱点，耗尽目标节点的资源，导致其无法正常响应合法用户的请求或者直接使得节点崩溃停止服务

##### 简易示例

IOST 节点之间 P2P 通信使用的是 libp2p，汇集了各种传输和点对点协议，使开发人员可以轻松构建大型，强大的 p2p 网络，IOST 节点的 P2P service 启动流程首先创建一个 NetService，代码如下：

```plain
// NewNetService returns a NetService instance with the config argument.
func NewNetService(config *common.P2PConfig) (*NetService, error) {
    ns := &NetService{
        config: config,
    }

    if err := os.MkdirAll(config.DataPath, 0755); config.DataPath != "" && err != nil {
        ilog.Errorf("failed to create p2p datapath, err=%v, path=%v", err, config.DataPath)
        return nil, err
    }

    privKey, err := getOrCreateKey(filepath.Join(config.DataPath, privKeyFile))
    if err != nil {
        ilog.Errorf("failed to get private key. err=%v, path=%v", err, config.DataPath)
        return nil, err
    }

    host, err := ns.startHost(privKey, config.ListenAddr)
    if err != nil {
        ilog.Errorf("failed to start a host. err=%v, listenAddr=%v", err, config.ListenAddr)
        return nil, err
    }
    ns.host = host

    ns.PeerManager = NewPeerManager(host, config)

    ns.adminServer = newAdminServer(config.AdminPort, ns.PeerManager)

    return ns, nil
}
```

host 的流处理逻辑在 ns.streamHandler 中

```plain
func (ns *NetService) streamHandler(s libnet.Stream) {
    ns.PeerManager.HandleStream(s, inbound)
}
```

steamHandler 又调用 PeerManager 的 HandleStream 函数

```plain
// HandleStream handles the incoming stream.
//
// It checks whether the remote peer already exists.
// If the peer is new and the neighbor count doesn't reach the threshold, it adds the peer into the neighbor list.
// If peer already exits, just add the stream to the peer.
// In other cases, reset the stream.
func (pm *PeerManager) HandleStream(s libnet.Stream, direction connDirection) {
    remotePID := s.Conn().RemotePeer()
    pm.freshPeer(remotePID)

    if pm.isStreamBlack(s) {
        ilog.Infof("remote peer is in black list. pid=%v, addr=%v", remotePID.Pretty(), s.Conn().RemoteMultiaddr())
        s.Conn().Close()
        return
    }
    ilog.Debugf("handle new stream. pid=%s, addr=%v, direction=%v", remotePID.Pretty(), s.Conn().RemoteMultiaddr(), direction)

    peer := pm.GetNeighbor(remotePID)
    if peer != nil {
        s.Reset()
        return
    }

    if pm.NeighborCount(direction) >= pm.neighborCap[direction] {
        if !pm.isBP(remotePID) {
            ilog.Infof("neighbor count exceeds, close connection. remoteID=%v, addr=%v", remotePID.Pretty(), s.Conn().RemoteMultiaddr())
            if direction == inbound {
                bytes, _ := pm.getRoutingResponse([]string{remotePID.Pretty()})
                if len(bytes) > 0 {
                    msg := newP2PMessage(pm.config.ChainID, RoutingTableResponse, pm.config.Version, defaultReservedFlag, bytes)
                    s.Write(msg.content())
                }
                time.AfterFunc(time.Second, func() { s.Conn().Close() })
            } else {
                s.Conn().Close()
            }
            return
        }
        pm.kickNormalNeighbors(direction)
    }
    pm.AddNeighbor(NewPeer(s, pm, direction))
    return
}
```

对于新建立连接的 peer，IOST 会启动该 peer 并添加到 neighbor list 中

```plain
// AddNeighbor starts a peer and adds it to the neighbor list.
func (pm *PeerManager) AddNeighbor(p *Peer) {

    pm.neighborMutex.Lock()
    defer pm.neighborMutex.Unlock()

    if pm.neighbors[p.id] == nil {
        p.Start()
        pm.storePeerInfo(p.id, []multiaddr.Multiaddr{p.addr})
        pm.neighbors[p.id] = p
        pm.neighborCount[p.direction]++
    }
}
```

peer 启动之后，IOST 会调用 peer 的 readLoop 和 writeLoop 函数对该 peer 进行读写

```plain
// Start starts peer's loop.
func (p *Peer) Start() {
    ilog.Infof("peer is started. id=%s", p.ID())

    go p.readLoop()
    go p.writeLoop()
}
```

IOST 对我们发送的数据处理流程大致主要是在 P2P 网络中的一个节点 (Peer) 上创建一个循环，不断从 p.stream 中读取数据并进行处理，首先它读取一个固定长度的头部数据，其中包含了一些用于解析后续数据的必要信息，然后它根据读取到的头部数据中的链 ID(chainID) 和数据长度 (length) 进行验证，确保收到的数据是有效的。接下来它从 p.stream 中读取完整的数据并将头部数据与消息内容合并为一个完整的数据包，最后它将解析得到的消息进行处理并记录收到的字节数和消息数量，循环继续直到发生错误或者被显式中断，在循环结束后当前节点将从邻居列表中移除

```plain
func (p *Peer) readLoop() {
    header := make([]byte, dataBegin)
    for {
        _, err := io.ReadFull(p.stream, header)
        if err != nil {
            ilog.Warnf("read header failed. err=%v", err)
            break
        }
        chainID := binary.BigEndian.Uint32(header[chainIDBegin:chainIDEnd])
        if chainID != p.peerManager.config.ChainID {
            ilog.Warnf("mismatched chainID. chainID=%d", chainID)
            break
        }
        length := binary.BigEndian.Uint32(header[dataLengthBegin:dataLengthEnd])
        if length > maxDataLength {
            ilog.Warnf("data length too large: %d", length)
            break
        }
        data := make([]byte, dataBegin+length)
        _, err = io.ReadFull(p.stream, data[dataBegin:])
        if err != nil {
            ilog.Warnf("read message failed. err=%v", err)
            break
        }
        copy(data[0:dataBegin], header)
        msg, err := parseP2PMessage(data)
        if err != nil {
            ilog.Errorf("parse p2pmessage failed. err=%v", err)
            break
        }
        tagkv := map[string]string{"mtype": msg.messageType().String()}
        byteInCounter.Add(float64(len(msg.content())), tagkv)
        packetInCounter.Add(1, tagkv)
        p.handleMessage(msg)
    }

    p.peerManager.RemoveNeighbor(p.id)
}
```

在查看源代码时发现 messageLoop 中调用了 handlerHashQuery

```plain
func (sy *SyncImpl) messageLoop() {
    defer sy.wg.Done()
    for {
        select {
        case req := <-sy.messageChan:
            switch req.Type() {
            case p2p.SyncBlockHashRequest:
                var rh msgpb.BlockHashQuery
                err := proto.Unmarshal(req.Data(), &rh)
                if err != nil {
                    ilog.Errorf("Unmarshal BlockHashQuery failed:%v", err)
                    break
                }
                go sy.handleHashQuery(&rh, req.From())
                ...
```

当 messageType 为 p2p.SyncBlockHashRequest，Data 为 BlockHashQuery 时，handlerHashQuery 函数会被调用，BlockHashQuery 的结构如下，End 和 Start 的值可控，此时我们可以构造一个 Message 将 Start 的值设为 0，End 的值设为 math.MaxInt64，将该 Message 发送给节点就可以触发 make 函数的 cap out of range，导致拒绝服务

```plain
type BlockHashQuery struct {
    ReqType              RequireType `protobuf:"varint,1,opt,name=reqType,proto3,enum=msgpb.RequireType" json:"reqType,omitempty"`
    Start                int64       `protobuf:"varint,2,opt,name=start,proto3" json:"start,omitempty"`
    End                  int64       `protobuf:"varint,3,opt,name=end,proto3" json:"end,omitempty"`
    Nums                 []int64     `protobuf:"varint,4,rep,packed,name=nums,proto3" json:"nums,omitempty"`
    XXX_NoUnkeyedLiteral struct{}    `json:"-"`
    XXX_unrecognized     []byte      `json:"-"`
    XXX_sizecache        int32       `json:"-"`
}
```

### 文末小结

本篇文章我们深入探讨了以太坊的 P2P 网络，我们首先介绍了 P2P 网络的基本概念、节点分类、工作原理，随后借助以太坊源代码对以太坊 P2P 网络的各个组成部分，包括节点发现、握手协议和消息传递等进行了详细的介绍，最后我们对节点网络中潜在的安全风险给出了简易介绍和实际的审计示例说明，当然也有更多潜在的安全问题有待大家一起去发现
