# Nsq SourceCode Guide 2 - Nsqd Handle Msg

这篇主要来看下Nsqd关键的 Pub、deferPub、createTopic、CreateChannel是怎么实现的

## Pub、deferPub

1. 消息的发送
2. ```go
   func (p *protocolV2) PUB(client *clientV2, params [][]byte) ([]byte, error) {
      var err error
   //参数检查
      if len(params) < 2 {
         return nil, protocol.NewFatalClientErr(nil, "E_INVALID", "PUB insufficient number of parameters")
      }
   //topic合法性检查
      topicName := string(params[1])
      if !protocol.IsValidTopicName(topicName) {
         return nil, protocol.NewFatalClientErr(nil, "E_BAD_TOPIC",
            fmt.Sprintf("PUB topic name %q is not valid", topicName))
      }
   //body长度是否超过限制或者为0
      bodyLen, err := readLen(client.Reader, client.lenSlice)
      if err != nil {
         return nil, protocol.NewFatalClientErr(err, "E_BAD_MESSAGE", "PUB failed to read message body size")
      }
      if bodyLen <= 0 {
         return nil, protocol.NewFatalClientErr(nil, "E_BAD_MESSAGE",
            fmt.Sprintf("PUB invalid message body size %d", bodyLen))
      }

      if int64(bodyLen) > p.ctx.nsqd.getOpts().MaxMsgSize {
         return nil, protocol.NewFatalClientErr(nil, "E_BAD_MESSAGE",
            fmt.Sprintf("PUB message too big %d > %d", bodyLen, p.ctx.nsqd.getOpts().MaxMsgSize))
      }

      messageBody := make([]byte, bodyLen)
      _, err = io.ReadFull(client.Reader, messageBody)
      if err != nil {
         return nil, protocol.NewFatalClientErr(err, "E_BAD_MESSAGE", "PUB failed to read message body")
      }

      if err := p.CheckAuth(client, "PUB", topicName, ""); err != nil {
         return nil, err
      }
   //获取topic信息然后将message信息写入topic，这里的PutMessage是最关键的地方
      topic := p.ctx.nsqd.GetTopic(topicName)
      msg := NewMessage(topic.GenerateID(), messageBody)
      err = topic.PutMessage(msg)
      if err != nil {
         return nil, protocol.NewFatalClientErr(err, "E_PUB_FAILED", "PUB failed "+err.Error())
      }

      client.PublishedMessage(topicName, 1)

      return okBytes, nil
   }
   ```
3. \[topic.PutMessage\(msg\)\]
4. ```go
   // PutMessage writes a Message to the queue
   func (t *Topic) PutMessage(m *Message) error {
     //对topic加一层读锁判断当前是否已经是退出状态了
      t.RLock()
      defer t.RUnlock()
      if atomic.LoadInt32(&t.exitFlag) == 1 {
         return errors.New("exiting")
      }
     //关键方法Put
      err := t.put(m)
      if err != nil {
         return err
      }
      atomic.AddUint64(&t.messageCount, 1)
      atomic.AddUint64(&t.messageBytes, uint64(len(m.Body)))
      return nil
   }
   ```
5. \[Topic.put\]
6. ```go
   func (t *Topic) put(m *Message) error {
      select {
        //优先会放到内存的Channel里面去，这个memoryMsgChan的大小是由配置来决定的
        //配置了内存的channel的话，如果这个时候NSQd意外宕机，则会丢失数据
        //如果没有配置内存channel那么就不会丢失数据，但是效率会降低不少
      case t.memoryMsgChan <- m:
      default:
        //如果内存满了，会用diskQueue写入到磁盘
        //这里需要注意的是这里的buffer对象是在sync.Pool里面的一个可以重复利用的对象，
        //防止内存的重复分配
         b := bufferPoolGet()
         err := writeMessageToBackend(b, m, t.backend)
         bufferPoolPut(b)
         t.ctx.nsqd.SetHealth(err)
         if err != nil {
            t.ctx.nsqd.logf(LOG_ERROR,
               "TOPIC(%s) ERROR: failed to write message to backend - %s",
               t.name, err)
            return err
         }
      }
      return nil
   }
   ```
7. nsqd维护了另外的一个gorutine
8. ```go
   // messagePump 选择在内存memoryQueue或者diskQueue里面的数据来复制到
   //Topic的每一个channel里面去
   func (t *Topic) messagePump() {
     /*因为篇幅限制我忽略了一些代码*/
      // main message loop
      for {
         select {
         case msg = <-memoryMsgChan:
         case buf = <-backendChan:
            msg, err = decodeMessage(buf)
            if err != nil {
               t.ctx.nsqd.logf(LOG_ERROR, "failed to decode message - %s", err)
               continue
            }
         case <-t.channelUpdateChan:
            //更新channel的信息或者是 重新启动被pause的Topic
            continue
         case <-t.pauseChan:
           //通过置空memoryMsgChan、backendChan的方式暂停消费
            if len(chans) == 0 || t.IsPaused() {
               memoryMsgChan = nil
               backendChan = nil
            } else {
               memoryMsgChan = t.memoryMsgChan
               backendChan = t.backend.ReadChan()
            }
            continue
         case <-t.exitChan:
            goto exit
         }

         for i, channel := range chans {
            chanMsg := msg
            // 为每一个channel复制当前的message消息
            if i > 0 {
               chanMsg = NewMessage(msg.ID, msg.Body)
               chanMsg.Timestamp = msg.Timestamp
               chanMsg.deferred = msg.deferred
            }
            if chanMsg.deferred != 0 {
               channel.PutMessageDeferred(chanMsg, chanMsg.deferred)
               continue
            }
            err := channel.PutMessage(chanMsg)
            if err != nil {
               t.ctx.nsqd.logf(LOG_ERROR,
                  "TOPIC(%s) ERROR: failed to put msg(%s) to channel(%s) - %s",
                  t.name, msg.ID, channel.name, err)
            }
         }
      }

   exit:
      t.ctx.nsqd.logf(LOG_INFO, "TOPIC(%s): closing ... messagePump", t.name)
   }
   ```
9. 再来看下「channel.PutMessageDeferred」 和「channel.PutMessage」
10. ```go
    func (c *Channel) PutMessageDeferred(msg *Message, timeout time.Duration) {
       atomic.AddUint64(&c.messageCount, 1)
       c.StartDeferredTimeout(msg, timeout)
    }
    //处理延迟消息的方式
    func (c *Channel) StartDeferredTimeout(msg *Message, timeout time.Duration) error {
        absTs := time.Now().Add(timeout).UnixNano()
      //一个最小堆实现的优先级队列,会把 需要发送的时间作为score放入最小堆
        item := &pqueue.Item{Value: msg, Priority: absTs}
        err := c.pushDeferredMessage(item)
        if err != nil {
            return err
        }
        c.addToDeferredPQ(item)
        return nil
    }
    ```
11. ```go
    // PutMessage writes a Message to the queue
    func (c *Channel) PutMessage(m *Message) error {
        c.RLock()
        defer c.RUnlock()
        if c.Exiting() {
            return errors.New("exiting")
        }
        err := c.put(m)
        if err != nil {
            return err
        }
        atomic.AddUint64(&c.messageCount, 1)
        return nil
    }
    ```

12.

```go
//看到这个方法应该很熟悉,channel topic Put操作如出一辙
func (c *Channel) put(m *Message) error {
   select {
     //首先会选择放入到内存的channel
   case c.memoryMsgChan <- m:
   default:
     //其次会通过diskQueue放到磁盘
     //依然使用了sync.Pool 来做对象的复用，减少GC 
      b := bufferPoolGet()
      err := writeMessageToBackend(b, m, c.backend)
      bufferPoolPut(b)
      c.ctx.nsqd.SetHealth(err)
      if err != nil {
         c.ctx.nsqd.logf(LOG_ERROR, "CHANNEL(%s): failed to write message to backend - %s",
            c.name, err)
         return err
      }
   }
   return nil
}
```

## Topic、channel的创建于销毁

`Topic` 和 `Channel` 都没有预先配置。`Topic` 由第一次发布消息到命名的 `Topic` 或第一次通过订阅一个命名 `Topic` 来创建。`Channel` 被第一次订阅到指定的 `Channel` 创建。`Topic` 和 `Channel` 的所有缓冲的数据相互独立，防止缓慢消费者造成对其他 `Channel` 的积压（同样适用于 `Topic` 级别）。

```go
// GetTopic 是线程安全的 
func (n *NSQD) GetTopic(topicName string) *Topic {
   // 通过topic_namn来获取对应的topic对象，如果没有的话就会新建
   // 应为绝大部分情况下是可以获取到的 所以优先使用读锁
   n.RLock()
   t, ok := n.topicMap[topicName]
   n.RUnlock()
   if ok {
      return t
   }
   n.Lock()
   t, ok = n.topicMap[topicName]
   if ok {
      n.Unlock()
      return t
   }
   deleteCallback := func(t *Topic) {
      n.DeleteExistingTopic(t.name)
   }
  //新建Topic的
   t = NewTopic(topicName, &context{n}, deleteCallback)
   n.topicMap[topicName] = t

   n.Unlock()

   n.logf(LOG_INFO, "TOPIC(%s): created", t.name)
    //    此时Topic已经创建了 但是messagePump线程还没有启动
   if atomic.LoadInt32(&n.isLoading) == 1 {
      return t
   }
   // 如果使用 lookupd，则进行阻塞调用以获取Topic，并立即创建它们。
   // 这样可以确保将收到的任何消息缓冲到正确的channel
   // 因为各个节点之间的需要新建Topic的情况是很少的，
   // 所以在新建Topic的时候悲观的去同步其他节点的channel是很可取的
   lookupdHTTPAddrs := n.lookupdHTTPAddrs()
   if len(lookupdHTTPAddrs) > 0 {
      channelNames, err := n.ci.GetLookupdTopicChannels(t.name, lookupdHTTPAddrs)
      if err != nil {
         n.logf(LOG_WARN, "failed to query nsqlookupd for channels to pre-create for topic %s - %s", t.name, err)
      }
      for _, channelName := range channelNames {
     /*somecode*/
         t.GetChannel(channelName)
      }
   } else if len(n.getOpts().NSQLookupdTCPAddresses) > 0 {
      n.logf(LOG_ERROR, "no available nsqlookupd to query for channels to pre-create for topic %s", t.name)
   }
   // 到这里位置所有的channel都完成加载了
   t.Start()
   return t
}
```

**多路分发** - `producer` 会同时连上 `nsq` 集群中所有 `nsqd` 节点，当然这些节点的地址是在初始化时，通过外界传递进去；当发布消息时，`producer` 会随机选择一个 `nsqd` 节点发布某个 `Topic` 的消息；`consumer` 在订阅 `subscribe` 某个`Topic/Channel`时，会首先连上 `nsqlookupd` 获取最新可用的 `nsqd` 节点，然后通过 `TCP` 长连接方式连上所有发布了指定 `Topic` 的 `producer` 节点，并在本地用 `tornado` 轮询每个连接，当某个连接有可读事件时，即有消息达到，处理即可。

## 结束语

至此 Nsqd的 Pub、deferPub Topic、channel的创建与销毁我们都跟完了

溜了

