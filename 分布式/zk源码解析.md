

## 一. zk启动宏观流程图

![](..\images\zk\zk-start.png)



## 二. 启动源码

在 zk 中集群的启动类为 `QuorumPeerMain`。

```java
// QuorumPeerMain.java
public static void main(String[] args) {
    QuorumPeerMain main = new QuorumPeerMain();
    try {
        main.initializeAndRun(args);
    } catch (IllegalArgumentException e) {
        // 省略
    }
    LOG.info("Exiting normally");
    ServiceUtils.requestSystemExit(ExitCode.EXECUTION_FINISHED.getValue());
}

protected void initializeAndRun(String[] args) throws ConfigException, IOException, AdminServerException {
    // 构建一个配置文件类，存储配置文件信息
    QuorumPeerConfig config = new QuorumPeerConfig();
    if (args.length == 1) {
        // 解析配置文件
        config.parse(args[0]);
    }

    // 构建清除器管理器，用于清理过期的快照
    DatadirCleanupManager purgeMgr = new DatadirCleanupManager(
        config.getDataDir(),
        config.getDataLogDir(),
        config.getSnapRetainCount(),
        config.getPurgeInterval());
    // 启动清理器
    purgeMgr.start();

    if (args.length == 1 && config.isDistributed()) {
        // 以集群方式启动
        runFromConfig(config);
    } else {
        LOG.warn("Either no config or no quorum defined in config, running in standalone mode");
        // 单机模式启动
        ZooKeeperServerMain.main(args);
    }
}

/**
* 根据配置文件启动zk
*/
public void runFromConfig(QuorumPeerConfig config) throws IOException, AdminServerException {
    try {
        ManagedUtil.registerLog4jMBeans();
    } catch (JMException e) {
        LOG.warn("Unable to register log4j JMX control", e);
    }

    LOG.info("Starting quorum peer, myid=" + config.getServerId());
    final MetricsProvider metricsProvider;
    try {
        metricsProvider = MetricsProviderBootstrap.startMetricsProvider(
            config.getMetricsProviderClassName(),
            config.getMetricsProviderConfiguration());
    } catch (MetricsProviderLifeCycleException error) {
        throw new IOException("Cannot boot MetricsProvider " + config.getMetricsProviderClassName(), error);
    }
    try {
        ServerMetrics.metricsProviderInitialized(metricsProvider);
        ProviderRegistry.initialize();
        ServerCnxnFactory cnxnFactory = null;
        ServerCnxnFactory secureCnxnFactory = null;

        if (config.getClientPortAddress() != null) {
            // 创建服务，默认为NIO，可以通过zookeeper.serverCnxnFactory修改
            cnxnFactory = ServerCnxnFactory.createFactory();
            // 绑定ip、端口等属性
            cnxnFactory.configure(config.getClientPortAddress(), config.getMaxClientCnxns(), config.getClientPortListenBacklog(), false);
        }

        if (config.getSecureClientPortAddress() != null) {
            // 配置安全连接
            secureCnxnFactory = ServerCnxnFactory.createFactory();
            secureCnxnFactory.configure(config.getSecureClientPortAddress(), config.getMaxClientCnxns(), config.getClientPortListenBacklog(), true);
        }
		
        // quorumPeer：可以看作集群中的一个节点
        quorumPeer = getQuorumPeer();
        // FileTxnSnapLog：zk中快照和日志的管理器
        quorumPeer.setTxnFactory(new FileTxnSnapLog(config.getDataLogDir(), config.getDataDir()));
        quorumPeer.enableLocalSessions(config.areLocalSessionsEnabled());
        quorumPeer.enableLocalSessionsUpgrading(config.isLocalSessionsUpgradingEnabled());
        //quorumPeer.setQuorumPeers(config.getAllMembers());
        quorumPeer.setElectionType(config.getElectionAlg());
        quorumPeer.setMyid(config.getServerId());
        quorumPeer.setTickTime(config.getTickTime());
        quorumPeer.setMinSessionTimeout(config.getMinSessionTimeout());
        quorumPeer.setMaxSessionTimeout(config.getMaxSessionTimeout());
        quorumPeer.setInitLimit(config.getInitLimit());
        quorumPeer.setSyncLimit(config.getSyncLimit());
        quorumPeer.setConnectToLearnerMasterLimit(config.getConnectToLearnerMasterLimit());
        quorumPeer.setObserverMasterPort(config.getObserverMasterPort());
        quorumPeer.setConfigFileName(config.getConfigFilename());
        quorumPeer.setClientPortListenBacklog(config.getClientPortListenBacklog());
        // 创建节点存储数据结构
        quorumPeer.setZKDatabase(new ZKDatabase(quorumPeer.getTxnFactory()));
        quorumPeer.setQuorumVerifier(config.getQuorumVerifier(), false);
        if (config.getLastSeenQuorumVerifier() != null) {
            quorumPeer.setLastSeenQuorumVerifier(config.getLastSeenQuorumVerifier(), false);
        }
        quorumPeer.initConfigInZKDatabase();
        quorumPeer.setCnxnFactory(cnxnFactory);
        quorumPeer.setSecureCnxnFactory(secureCnxnFactory);
        quorumPeer.setSslQuorum(config.isSslQuorum());
        quorumPeer.setUsePortUnification(config.shouldUsePortUnification());
        quorumPeer.setLearnerType(config.getPeerType());
        quorumPeer.setSyncEnabled(config.getSyncEnabled());
        quorumPeer.setQuorumListenOnAllIPs(config.getQuorumListenOnAllIPs());
        if (config.sslQuorumReloadCertFiles) {
            quorumPeer.getX509Util().enableCertFileReloading();
        }
        quorumPeer.setMultiAddressEnabled(config.isMultiAddressEnabled());
        quorumPeer.setMultiAddressReachabilityCheckEnabled(config.isMultiAddressReachabilityCheckEnabled());
        quorumPeer.setMultiAddressReachabilityCheckTimeoutMs(config.getMultiAddressReachabilityCheckTimeoutMs());

        // sets quorum sasl authentication configurations
        quorumPeer.setQuorumSaslEnabled(config.quorumEnableSasl);
        if (quorumPeer.isQuorumSaslAuthEnabled()) {
            quorumPeer.setQuorumServerSaslRequired(config.quorumServerRequireSasl);
            quorumPeer.setQuorumLearnerSaslRequired(config.quorumLearnerRequireSasl);
            quorumPeer.setQuorumServicePrincipal(config.quorumServicePrincipal);
            quorumPeer.setQuorumServerLoginContext(config.quorumServerLoginContext);
            quorumPeer.setQuorumLearnerLoginContext(config.quorumLearnerLoginContext);
        }
        quorumPeer.setQuorumCnxnThreadsSize(config.quorumCnxnThreadsSize);
        quorumPeer.initialize();

        if (config.jvmPauseMonitorToRun) {
            quorumPeer.setJvmPauseMonitor(new JvmPauseMonitor(config));
        }
		
        // 启动集群
        quorumPeer.start();
        ZKAuditProvider.addZKStartStopAuditLog();
        // 保证上面主线程执行完
        quorumPeer.join();
    } catch (InterruptedException e) {
        // warn, but generally this is ok
        LOG.warn("Quorum Peer interrupted", e);
    } finally {
        try {
            metricsProvider.stop();
        } catch (Throwable error) {
            LOG.warn("Error while stopping metrics", error);
        }
    }
}
```



> #### 启动集群：quorumPeer.start();

```java
/**
* QuorumPeer：继承了 Thread
*/ 
public synchronized void start() {
    if (!getView().containsKey(myid)) {
        throw new RuntimeException("My id " + myid + " not in the peer list");
    }
    // 加载快照、日志文件中的数据
    loadDataBase();
    // 启动netty或nio服务
    startServerCnxnFactory();
    try {
        adminServer.start();
    } catch (AdminServerException e) {
        LOG.warn("Problem starting AdminServer", e);
        System.out.println(e);
    }
    // 启动选举
    startLeaderElection();
    startJvmPauseMonitor();
    // 启动线程
    super.start();
}
```



### 1. 启动时加载快照、日志数据

zk 中的数据存储在内存`ZKDatabase`中，但是对 zk 中节点数据的写操作都记录到日志文件中，当写入一定次数时会生成一次快照，进行数据备份。

快照日志相关配置参数：

| 参数                      | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| dataLogDir                | 事务日志目录                                                 |
| zookeeper.preAllocSize    | 预先申请内存，由于后续事务日志写入，默认64M                  |
| zookeeper.snapCount       | 每进行snapCount次事务日志输出后，触发一次快照生成，默认是100,000 |
| autopurge.snapRetainCount | 自动清除时 保留的快照数                                      |
| autopurge.purgeInterval   | 清除时间间隔，单位为小时， -1 表示不自动清除。               |



**加载快照、日志源码：**

```java
// QuorumPeer.java
private void loadDataBase() {
    try {
        // 从快照/日志中加载数据
        zkDb.loadDataBase();

        // 获取最大zxid
        long lastProcessedZxid = zkDb.getDataTree().lastProcessedZxid;
        long epochOfZxid = ZxidUtils.getEpochFromZxid(lastProcessedZxid);
        try {
            currentEpoch = readLongFromFile(CURRENT_EPOCH_FILENAME);
        } catch (FileNotFoundException e) {

            currentEpoch = epochOfZxid;
            /**
            * 省略日志 
            */
            writeLongToFile(CURRENT_EPOCH_FILENAME, currentEpoch);
        }
        if (epochOfZxid > currentEpoch) {
            // 省略抛异常
        }
        try {
            acceptedEpoch = readLongFromFile(ACCEPTED_EPOCH_FILENAME);
        } catch (FileNotFoundException e) {

            acceptedEpoch = epochOfZxid;
            // 省略日志            
            writeLongToFile(ACCEPTED_EPOCH_FILENAME, acceptedEpoch);
        }
        if (acceptedEpoch < currentEpoch) {
            // 省略异常
        }
    } catch (IOException ie) {
        // 省略异常
    }
}

// ZKDatabase.java
public long loadDataBase() throws IOException {
    long startTime = Time.currentElapsedTime();
    // 从快照、日志中读取数据，返回最大zxid
    long zxid = snapLog.restore(dataTree, sessionsWithTimeouts, commitProposalPlaybackListener);
    initialized = true;
    long loadTime = Time.currentElapsedTime() - startTime;
    ServerMetrics.getMetrics().DB_INIT_TIME.add(loadTime);
    LOG.info("Snapshot loaded in {} ms, highest zxid is 0x{}, digest is {}",
             loadTime, Long.toHexString(zxid), dataTree.getTreeDigest());
    return zxid;
}
```



### 2. 初始化对外服务

```java
private void startServerCnxnFactory() {
    if (cnxnFactory != null) {
        // 启动服务
        cnxnFactory.start();
    }
    if (secureCnxnFactory != null) {
        secureCnxnFactory.start();
    }
}

/**
* cnxnFactory.start();
*/
public void start() {
    if (listenBacklog != -1) {
        bootstrap.option(ChannelOption.SO_BACKLOG, listenBacklog);
    }
    LOG.info("binding to port {}", localAddress);
    parentChannel = bootstrap.bind(localAddress).syncUninterruptibly().channel();
    // Port changes after bind() if the original port was 0, update
    // localAddress to get the real port.
    localAddress = (InetSocketAddress) parentChannel.localAddress();
    LOG.info("bound to port {}", getLocalPort());
}
```



`ServerCnxnFactory`用于构建网络I/O，通过 NIO 或者 Netty 的建立 socket 通道，其 UML 图如下。



![](..\images\zk\servercnxnfactory.png)





> #### 初始化 netty 工厂

```java
NettyServerCnxnFactory() {
    x509Util = new ClientX509Util();

    boolean usePortUnification = Boolean.getBoolean(PORT_UNIFICATION_KEY);
    LOG.info("{}={}", PORT_UNIFICATION_KEY, usePortUnification);
    if (usePortUnification) {
        try {
            QuorumPeerConfig.configureSSLAuth();
        } catch (QuorumPeerConfig.ConfigException e) {
            LOG.error("unable to set up SslAuthProvider, turning off client port unification", e);
            usePortUnification = false;
        }
    }
    this.shouldUsePortUnification = usePortUnification;

    this.advancedFlowControlEnabled = Boolean.getBoolean(NETTY_ADVANCED_FLOW_CONTROL);
    LOG.info("{} = {}", NETTY_ADVANCED_FLOW_CONTROL, this.advancedFlowControlEnabled);

    setOutstandingHandshakeLimit(Integer.getInteger(OUTSTANDING_HANDSHAKE_LIMIT, -1));

    // 创建线程组，用于处理与客户端的I/O
    EventLoopGroup bossGroup = NettyUtils.newNioOrEpollEventLoopGroup(NettyUtils.getClientReachableLocalInetAddressCount());
    // 创建工作线程组 
    EventLoopGroup workerGroup = NettyUtils.newNioOrEpollEventLoopGroup();
    ServerBootstrap bootstrap = new ServerBootstrap().group(bossGroup, workerGroup)
        .channel(NettyUtils.nioOrEpollServerSocketChannel())
        // parent channel options
        .option(ChannelOption.SO_REUSEADDR, true)
        // child channels options
        .childOption(ChannelOption.TCP_NODELAY, true)
        .childOption(ChannelOption.SO_LINGER, -1)
        // 添加管道流
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel ch) throws Exception {
                ChannelPipeline pipeline = ch.pipeline();
                if (advancedFlowControlEnabled) {
                    pipeline.addLast(readIssuedTrackingHandler);
                }
                if (secure) {
                    initSSL(pipeline, false);
                } else if (shouldUsePortUnification) {
                    initSSL(pipeline, true);
                }
                pipeline.addLast("servercnxnfactory", channelHandler);
            }
        });
    this.bootstrap = configureBootstrapAllocator(bootstrap);
    this.bootstrap.validate();
}
```

`EventLoopGroup`的 UML 图如下，实现了`ExecutorSerivice`，

![image-20210105112358148](..\images\zk\eventloopgroup.png)



### 3. 选举

```java
public synchronized void startLeaderElection() {
    try {
        // 判断是否时LOOKING状态
        if (getPeerState() == ServerState.LOOKING) {
            // getLastLoggedZxid()：获取最大zxid
            // getCurrentEpoch()：获取选取轮数
            currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
        }
    } catch (IOException e) {
        RuntimeException re = new RuntimeException(e.getMessage());
        re.setStackTrace(e.getStackTrace());
        throw re;
    }

    this.electionAlg = createElectionAlgorithm(electionType);
}

/**
* 创建选举算法，默认是3，即FastLeaderElection
*/
protected Election createElectionAlgorithm(int electionAlgorithm) {
    Election le = null;

    //TODO: use a factory rather than a switch
    switch (electionAlgorithm) {
        case 1:
            throw new UnsupportedOperationException("Election Algorithm 1 is not supported.");
        case 2:
            throw new UnsupportedOperationException("Election Algorithm 2 is not supported.");
        case 3:
            QuorumCnxManager qcm = createCnxnManager();
            QuorumCnxManager oldQcm = qcmRef.getAndSet(qcm);
            if (oldQcm != null) {
                LOG.warn("Clobbering already-set QuorumCnxManager (restarting leader election?)");
                oldQcm.halt();
            }
            QuorumCnxManager.Listener listener = qcm.listener;
            if (listener != null) {
                listener.start();
                FastLeaderElection fle = new FastLeaderElection(this, qcm);
                fle.start();
                le = fle;
            } else {
                LOG.error("Null listener when initializing cnx manager");
            }
            break;
        default:
            assert false;
    }
    return le;
}
```



https://ld246.com/article/1553725955154#2-3--%E5%88%9D%E5%A7%8B%E5%8C%96