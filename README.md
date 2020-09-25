---
title: Nsq-SourceCode-Guide-1-bootstrap-of-nsqd
date: '2020-09-24T08:33:39.000Z'
tags: nsq
description: nsq 源码导读（1），nsqd的启动过程
---

# Nsq SourceCode Guide 1 - bootstrap of nsqd

我是一个gopher。

最近在研究NSQ的源码，具体源码阅读逻辑以及原理基本写在了注释上。

本篇先来看下NSQ是怎么启动的。 

## 配置

1. 首先来看下Nsqd的配置项。

   1. 看起来有很多但是可以根据的作用大体上分成3块，我加了一些注释，主要关注前面四部分

   ```go
   type Options struct {
   //Nsqd的基础参数
   ID        int64       flag:"node-id" cfg:"id"//节点ID，主要用于 nsqlookupd
   LogLevel  lg.LogLevel flag:"log-level"//日志级别
   LogPrefix string      flag:"log-prefix"//日志前缀 默认值是[Nsqd ]
   Logger    Logger
   TCPAddress               string        flag:"tcp-address" //暴露的TCP地址 一般都是本地+配置端口
   HTTPAddress              string        flag:"http-address"//暴露的http地址 一般都是本地+配置端口
   HTTPSAddress             string        flag:"https-address"//暴露的https地址 一般都是本地+配置端口
   BroadcastAddress         string        flag:"broadcast-address"//将使用 lookupd 注册的地址（默认为 OS 主机名） 
   NSQLookupdTCPAddresses   []string      flag:"lookupd-tcp-address" //nsqLookUpd 的地址(上报节点MetaData的信息)
   AuthHTTPAddresses        []string      flag:"auth-http-address" //如果指定了鉴权地址，会在连接时用改地址进行鉴权
   HTTPClientConnectTimeout time.Duration flag:"http-client-connect-timeout" 
   HTTPClientRequestTimeout time.Duration flag:"http-client-request-timeout" 
   // 磁盘读写工具「diskqueue」的一些配置，如果读者不清楚的话后续章节会详细说明
   DataPath        string        flag:"data-path"
   MemQueueSize    int64         flag:"mem-queue-size"
   MaxBytesPerFile int64         flag:"max-bytes-per-file"
   SyncEvery       int64         flag:"sync-every"
   SyncTimeout     time.Duration flag:"sync-timeout"
   QueueScanInterval        time.Duration
   QueueScanRefreshInterval time.Duration
   QueueScanSelectionCount  int flag:"queue-scan-selection-count"
   QueueScanWorkerPoolMax   int flag:"queue-scan-worker-pool-max"
   QueueScanDirtyPercent    float64
   // Nsq消息体的一些配置
   MsgTimeout    time.Duration flag:"msg-timeout"
   MaxMsgTimeout time.Duration flag:"max-msg-timeout"
   MaxMsgSize    int64         flag:"max-msg-size"
   MaxBodySize   int64         flag:"max-body-size"
   MaxReqTimeout time.Duration flag:"max-req-timeout"
   ClientTimeout time.Duration
   // 客户端可重写的一些配置
   MaxHeartbeatInterval   time.Duration flag:"max-heartbeat-interval"
   MaxRdyCount            int64         flag:"max-rdy-count"
   MaxOutputBufferSize    int64         flag:"max-output-buffer-size"
   MaxOutputBufferTimeout time.Duration flag:"max-output-buffer-timeout"
   MinOutputBufferTimeout time.Duration flag:"min-output-buffer-timeout"
   OutputBufferTimeout    time.Duration flag:"output-buffer-timeout"
   MaxChannelConsumers    int           flag:"max-channel-consumers"
   // 统计整合
   StatsdAddress       string        flag:"statsd-address"
   StatsdPrefix        string        flag:"statsd-prefix"
   StatsdInterval      time.Duration flag:"statsd-interval"
   StatsdMemStats      bool          flag:"statsd-mem-stats"
   StatsdUDPPacketSize int           flag:"statsd-udp-packet-size"
   // 端对端的消息延迟配置
   E2EProcessingLatencyWindowTime  time.Duration flag:"e2e-processing-latency-window-time"
   E2EProcessingLatencyPercentiles []float64     flag:"e2e-processing-latency-percentile" cfg:"e2e_processing_latency_percentiles"
   // TLS config
   TLSCert             string flag:"tls-cert"
   TLSKey              string flag:"tls-key"
   TLSClientAuthPolicy string flag:"tls-client-auth-policy"
   TLSRootCAFile       string flag:"tls-root-ca-file"
   TLSRequired         int    flag:"tls-required"
   TLSMinVersion       uint16 flag:"tls-min-version"
   // 压缩选项
   DeflateEnabled  bool flag:"deflate"
   MaxDeflateLevel int  flag:"max-deflate-level"
   SnappyEnabled   bool flag:"snappy"
   ```

## 启动过程

1. 代码的入口 \[/nsq/apps/nsqd/main.go\].
2. ```go
   func main() {
      prg := &program{}
      //接受一个 services接口(需要实现 Init Start Stop 三个接口)
      //这里的Init方法主要是对Windows的一些初始化过程，我们就略过不看了。
      if err := svc.Run(prg, syscall.SIGINT, syscall.SIGTERM); err != nil {
         logFatal("%s", err)
      }
   }
   ```

   1. 接着我们来看Run方法,我删除了一些不重要的代码段
   2. ```go
      func (p *program) Start() error {
       //NewOptions 初始化一些么默认的配置值包括 TCP、HTTP、HTTPS端口号、HTTP等各种超时时间、内存队列大小、同步时间等等 关键方法之一
       opts := nsqd.NewOptions()
       /*somde code*/
       //配置的有效性检查与赋值
       cfg.Validate()
       options.Resolve(opts, flagSet, cfg)
       //☆关键的方法之二，根据配置项新建Nsqd对象，并且赋值给p作为成员变量
       nsqd, err := nsqd.New(opts)
       if err != nil {
          logFatal("failed to instantiate nsqd - %s", err)
       }
       p.nsqd = nsqd

       //从硬盘中读取一些元数据信息了例如我们有哪些Topic或者Channel
       err = p.nsqd.LoadMetadata()
       if err != nil {
          logFatal("failed to load metadata - %s", err)
       }
       //持久化这些源数据信息
       err = p.nsqd.PersistMetadata()
       if err != nil {
          logFatal("failed to persist metadata - %s", err)
       }
       go func() {
          //NSQD的主逻辑
          err := p.nsqd.Main()
          if err != nil {
             p.Stop()
             os.Exit(1)
          }
       }()

       return nil
      }
      ```
3. 然后我们需要跳进到\[ nsqd, err := nsqd.New\(opts\)\] 看下
   1. ```go
      func New(opts *Options) (*NSQD, error) {
         var err error
         //数据在硬盘上的存储路径,默认是当前的路径(PWD)
         dataPath := opts.DataPath
         if opts.DataPath == "" {
            cwd, _ := os.Getwd()
            dataPath = cwd
         }
         //日志初始化
         if opts.Logger == nil {
            opts.Logger = log.New(os.Stderr, opts.LogPrefix, log.Ldate|log.Ltime|log.Lmicroseconds)
         }
         //主要数据结构的初始化
         n := &NSQD{
            startTime:            time.Now(),
            topicMap:             make(map[string]*Topic),
            clients:              make(map[int64]Client),
            exitChan:             make(chan int),
            notifyChan:           make(chan interface{}),
            optsNotificationChan: make(chan struct{}, 1),
            dl:                   dirlock.New(dataPath),
         }

         //初始化一个HTTPClient
         httpcli := http_api.NewClient(nil, opts.HTTPClientConnectTimeout, opts.HTTPClientRequestTimeout)
         n.ci = clusterinfo.New(n.logf, httpcli)

         n.lookupPeers.Store([]*lookupPeer{})
         //回写配置项
         n.swapOpts(opts)
         n.errValue.Store(errStore{})

         //对数据目录上锁
         err = n.dl.Lock()
         if err != nil {
            return nil, fmt.Errorf("failed to lock data-path: %v", err)
         }
         //确定压缩等级
         if opts.MaxDeflateLevel < 1 || opts.MaxDeflateLevel > 9 {
            return nil, errors.New("--max-deflate-level must be [1,9]")
         }
         //nodeID 检查
         if opts.ID < 0 || opts.ID >= 1024 {
            return nil, errors.New("--node-id must be [0,1024)")
         }

         if opts.StatsdPrefix != "" {
            var port string
            _, port, err = net.SplitHostPort(opts.HTTPAddress)
            if err != nil {
               return nil, fmt.Errorf("failed to parse HTTP address (%s) - %s", opts.HTTPAddress, err)
            }
            statsdHostKey := statsd.HostKey(net.JoinHostPort(opts.BroadcastAddress, port))
            prefixWithHost := strings.Replace(opts.StatsdPrefix, "%s", statsdHostKey, -1)
            if prefixWithHost[len(prefixWithHost)-1] != '.' {
               prefixWithHost += "."
            }
            opts.StatsdPrefix = prefixWithHost
         }
         //https tls
         if opts.TLSClientAuthPolicy != "" && opts.TLSRequired == TLSNotRequired {
            opts.TLSRequired = TLSRequired
         }

         tlsConfig, err := buildTLSConfig(opts)
         if err != nil {
            return nil, fmt.Errorf("failed to build TLS config - %s", err)
         }
         if tlsConfig == nil && opts.TLSRequired != TLSNotRequired {
            return nil, errors.New("cannot require TLS client connections without TLS key and cert")
         }
         n.tlsConfig = tlsConfig

         for _, v := range opts.E2EProcessingLatencyPercentiles {
            if v <= 0 || v > 1 {
               return nil, fmt.Errorf("invalid E2E processing latency percentile: %v", v)
            }
         }

         n.logf(LOG_INFO, version.String("nsqd"))
         n.logf(LOG_INFO, "ID: %d", opts.ID)
         //启动TCP监听端口
         n.tcpServer = &tcpServer{}
         n.tcpListener, err = net.Listen("tcp", opts.TCPAddress)
         if err != nil {
            return nil, fmt.Errorf("listen (%s) failed - %s", opts.TCPAddress, err)
         }
         //启动HTTP监听端口
         n.httpListener, err = net.Listen("tcp", opts.HTTPAddress)
         if err != nil {
            return nil, fmt.Errorf("listen (%s) failed - %s", opts.HTTPAddress, err)
         }
         //启动HTTPS监听端口
         if n.tlsConfig != nil && opts.HTTPSAddress != "" {
            n.httpsListener, err = tls.Listen("tcp", opts.HTTPSAddress, n.tlsConfig)
            if err != nil {
               return nil, fmt.Errorf("listen (%s) failed - %s", opts.HTTPSAddress, err)
            }
         }

         return n, nil
      }
      ```
   2. 这里可以知道NSQ支持长连接的方式\(client的默认方式\)、http/https的方式\(RestfulApi的方式\)
4. 接下来看看\[err := p.nsqd.Main\(\)\] 的逻辑
   1. ```go
      func (n *NSQD) Main() error {
         ctx := &context{n}

         exitCh := make(chan error)
         var once sync.Once
         exitFunc := func(err error) {
            once.Do(func() {
               if err != nil {
                  n.logf(LOG_FATAL, "%s", err)
               }
               exitCh <- err
            })
         }
          //并行启动三个端口监听(TCP、HTTP、HTTPs)
         n.tcpServer.ctx = ctx
         n.waitGroup.Wrap(func() {
            exitFunc(protocol.TCPServer(n.tcpListener, n.tcpServer, n.logf))
         })

         httpServer := newHTTPServer(ctx, false, n.getOpts().TLSRequired == TLSRequired)
         n.waitGroup.Wrap(func() {
            exitFunc(http_api.Serve(n.httpListener, httpServer, "HTTP", n.logf))
         })

         if n.tlsConfig != nil && n.getOpts().HTTPSAddress != "" {
            httpsServer := newHTTPServer(ctx, true, true)
            n.waitGroup.Wrap(func() {
               exitFunc(http_api.Serve(n.httpsListener, httpsServer, "HTTPS", n.logf))
            })
         }

         n.waitGroup.Wrap(n.queueScanLoop)
         n.waitGroup.Wrap(n.lookupLoop)
         if n.getOpts().StatsdAddress != "" {
            n.waitGroup.Wrap(n.statsdLoop)
         }

         err := <-exitCh
         return err
      }
      ```
5. 导致里基本Nsqd就完成启动了
   1. http负责restful的接口调用
      1. ```go
         func newHTTPServer(ctx *context, tlsEnabled bool, tlsRequired bool) *httpServer {
            log := http_api.Log(ctx.nsqd.logf)

            router := httprouter.New()
            router.HandleMethodNotAllowed = true
            router.PanicHandler = http_api.LogPanicHandler(ctx.nsqd.logf)
            router.NotFound = http_api.LogNotFoundHandler(ctx.nsqd.logf)
            router.MethodNotAllowed = http_api.LogMethodNotAllowedHandler(ctx.nsqd.logf)
            s := &httpServer{
               ctx:         ctx,
               tlsEnabled:  tlsEnabled,
               tlsRequired: tlsRequired,
               router:      router,
            }

            router.Handle("GET", "/ping", http_api.Decorate(s.pingHandler, log, http_api.PlainText))
            router.Handle("GET", "/info", http_api.Decorate(s.doInfo, log, http_api.V1))

            // v1 negotiate
            router.Handle("POST", "/pub", http_api.Decorate(s.doPUB, http_api.V1))
            router.Handle("POST", "/mpub", http_api.Decorate(s.doMPUB, http_api.V1))
            router.Handle("GET", "/stats", http_api.Decorate(s.doStats, log, http_api.V1))

            // only v1
            router.Handle("POST", "/topic/create", http_api.Decorate(s.doCreateTopic, log, http_api.V1))
            router.Handle("POST", "/topic/delete", http_api.Decorate(s.doDeleteTopic, log, http_api.V1))
            router.Handle("POST", "/topic/empty", http_api.Decorate(s.doEmptyTopic, log, http_api.V1))
            router.Handle("POST", "/topic/pause", http_api.Decorate(s.doPauseTopic, log, http_api.V1))
            router.Handle("POST", "/topic/unpause", http_api.Decorate(s.doPauseTopic, log, http_api.V1))
            router.Handle("POST", "/channel/create", http_api.Decorate(s.doCreateChannel, log, http_api.V1))
            router.Handle("POST", "/channel/delete", http_api.Decorate(s.doDeleteChannel, log, http_api.V1))
            router.Handle("POST", "/channel/empty", http_api.Decorate(s.doEmptyChannel, log, http_api.V1))
            router.Handle("POST", "/channel/pause", http_api.Decorate(s.doPauseChannel, log, http_api.V1))
            router.Handle("POST", "/channel/unpause", http_api.Decorate(s.doPauseChannel, log, http_api.V1))
            router.Handle("GET", "/config/:opt", http_api.Decorate(s.doConfig, log, http_api.V1))
            router.Handle("PUT", "/config/:opt", http_api.Decorate(s.doConfig, log, http_api.V1))

            // debug
            router.HandlerFunc("GET", "/debug/pprof/", pprof.Index)
            router.HandlerFunc("GET", "/debug/pprof/cmdline", pprof.Cmdline)
            router.HandlerFunc("GET", "/debug/pprof/symbol", pprof.Symbol)
            router.HandlerFunc("POST", "/debug/pprof/symbol", pprof.Symbol)
            router.HandlerFunc("GET", "/debug/pprof/profile", pprof.Profile)
            router.Handler("GET", "/debug/pprof/heap", pprof.Handler("heap"))
            router.Handler("GET", "/debug/pprof/goroutine", pprof.Handler("goroutine"))
            router.Handler("GET", "/debug/pprof/block", pprof.Handler("block"))
            router.Handle("PUT", "/debug/setblockrate", http_api.Decorate(setBlockRateHandler, log, http_api.PlainText))
            router.Handler("GET", "/debug/pprof/threadcreate", pprof.Handler("threadcreate"))

            return s
         }
         ```
   2. tcp负责client端的信息处理,是由实现了IOLoop的\[ProtocolV2\]完成的
      1. ```go
         func (p *protocolV2) Exec(client *clientV2, params [][]byte) ([]byte, error) {
            if bytes.Equal(params[0], []byte("IDENTIFY")) {
               return p.IDENTIFY(client, params)
            }
            err := enforceTLSPolicy(client, p, params[0])
            if err != nil {
               return nil, err
            }
            switch {
            case bytes.Equal(params[0], []byte("FIN")):
               return p.FIN(client, params)
            case bytes.Equal(params[0], []byte("RDY")):
               return p.RDY(client, params)
            case bytes.Equal(params[0], []byte("REQ")):
               return p.REQ(client, params)
            case bytes.Equal(params[0], []byte("PUB")):
               return p.PUB(client, params)
            case bytes.Equal(params[0], []byte("MPUB")):
               return p.MPUB(client, params)
            case bytes.Equal(params[0], []byte("DPUB")):
               return p.DPUB(client, params)
            case bytes.Equal(params[0], []byte("NOP")):
               return p.NOP(client, params)
            case bytes.Equal(params[0], []byte("TOUCH")):
               return p.TOUCH(client, params)
            case bytes.Equal(params[0], []byte("SUB")):
               return p.SUB(client, params)
            case bytes.Equal(params[0], []byte("CLS")):
               return p.CLS(client, params)
            case bytes.Equal(params[0], []byte("AUTH")):
               return p.AUTH(client, params)
            }
            return nil, protocol.NewFatalClientErr(nil, "E_INVALID", fmt.Sprintf("invalid command %s", params[0]))
         }
         ```
   3. 下一篇我们来看下Nsqd的具体处理逻辑包括:topic的创建、channel的创建、Pub、deferPub这些逻辑在Nsqd上是怎么处理的。
   4. 溜了

