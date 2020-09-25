---
description: 来看看如何投递消息给消费者
---

# Nsq SourceCode Guide 3 - Consumer

这一篇 来看下消费数据的细节

## consumer消费数据

1. ```go
   func (p *protocolV2) messagePump(client *clientV2, startedChan chan bool) {
             /*shadow some unreachable code~~*/
      for {
         /*shadow some unreachable code~~*/
         select {
         case <-flusherChan:
          /*shadow some unreachable code~~*/
         case <-client.ReadyStateChan:
         case subChannel = <-subEventChan:
           /*shadow some unreachable code~~*/
         case identifyData := <-identifyEventChan:
           /*shadow some unreachable code~~*/
         case <-heartbeatChan:
          /*shadow some unreachable code~~*/
         case b := <-backendMsgChan:
           //来自diskQueue的消息win的话那么就推送这条消息
            if sampleRate > 0 && rand.Int31n(100) > sampleRate {
               continue
            }
                   // 消息decode
            msg, err := decodeMessage(b)
            if err != nil {
               p.ctx.nsqd.logf(LOG_ERROR, "failed to decode message - %s", err)
               continue
            }
           //投递次数++，主要用于消息的requeue的过程统计已经重复投递过多少次了
            msg.Attempts++

            subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
            client.SendingMessage()
            err = p.SendMessage(client, msg)
            if err != nil {
               goto exit
            }
            flushed = false
         case msg := <-memoryMsgChan:
           // 与backendMsgChan的消息是一样的逻辑
            if sampleRate > 0 && rand.Int31n(100) > sampleRate {
               continue
            }
            msg.Attempts++

            subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
            client.SendingMessage()
            err = p.SendMessage(client, msg)
            if err != nil {
               goto exit
            }
            flushed = false
         case <-client.ExitChan:
            goto exit
         }
      }

   exit:
      p.ctx.nsqd.logf(LOG_INFO, "PROTOCOL(V2): [%s] exiting messagePump", client)
      heartbeatTicker.Stop()
      outputBufferTicker.Stop()
      if err != nil {
         p.ctx.nsqd.logf(LOG_ERROR, "PROTOCOL(V2): [%s] messagePump error - %s", client, err)
      }
   }
   ```

