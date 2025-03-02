# raft

在消息队列的设计中会在Broker节点单独开启一个协程用于处理该节点上 raft 相关的事务。

```go
rpcServer.Start(opts) -> server.make()
```

在初始化 Broker 节点时同时开启该协程

```go
//func (p *parts_raft) StartServer()
	//本地创建parts——raft，为raft同步做准备
	s.parts_rafts = NewParts_Raft()
	go s.parts_rafts.Make(opt.Name, opt_cli, s.aplych, s.me)
	s.parts_rafts.StartServer()

```

其中 GetApplych 协程循环监听 applych 通道中的信息，并同步 leader 节点中的消息。applych 中的信息来自于 Raft 对象中的 Commited 方法。Commited 方法会将raft产生的日志或快照放入 applych 通道中。

```go
//接收applych管道的内容
//写入partition文件中
func (s *Server) GetApplych(applych chan info) {

    // 遍历从 applych 通道中接收到的每个消息
    for msg := range applych {

        // 如果消息的 producer 字段为 "Leader"
        if msg.producer == "Leader" {
            s.BecomeLeader(msg) // 调用函数处理成为 Leader 的逻辑
        } else {
            // 获取读锁，安全访问 topics 映射
            s.mu.RLock()
            topic, ok := s.topics[msg.topic_name] // 查找消息所属的 topic
            s.mu.RUnlock()

            // 记录日志，显示接收到的消息
            logger.DEBUG(logger.DLog, "S%d the message from applych is %v\n", s.me, msg)

            // 如果当前 broker 中不存在该 topic
            if !ok {
                // 记录错误日志，显示该 topic 未找到
                logger.DEBUG(logger.DError, "topic(%v) is not in this broker\n", msg.topic_name)
            } else {
                // 如果找到了 topic，填充消息的相关信息
                msg.me = s.me
                msg.BrokerName = s.Name
                msg.zkclient = &s.zkclient
                msg.file_name = "NowBlock.txt"
                
                // 调用 topic 的 addMessage 方法同步消息
                topic.addMessage(msg)
            }
        }
    }
}
```

StartServer 函数在消息队列集群中的作用是处理和应用 Raft 日志条目和快照，确保系统的高一致性和可靠性。它通过监听和处理 applyCh 和超时事件，来管理 Raft 状态机的应用和维护，从而支持消息队列系统的稳定和一致操作。

```go
func (p *parts_raft) StartServer() {

    // 记录日志，表明 Raft 服务器启动
    logger.DEBUG_RAFT(logger.DSnap, "S%d parts_raft start\n", p.me)

    // 启动一个 Goroutine 以处理 Raft 日志和应用消息
    go func() {

        for {
            // 检查当前 Raft 实例是否被杀死（停止运行）
            if !p.killed() {
                // 选择器，用于处理从不同通道接收的消息
                select {
                // 接收从 applyCh 通道传递过来的消息
                case m := <-p.applyCh:

                    // 如果当前消息表明此分区成为 Leader
                    if m.BeLeader {
                        str := m.TopicName + m.PartName
                        logger.DEBUG_RAFT(logger.DLog, "S%d Broker tPart(%v) become leader aply from %v to %v\n", p.me, str, p.applyindexs[str], m.CommandIndex)
                        p.applyindexs[str] = m.CommandIndex

                        // 如果此节点是 Leader，则向 appench 通道发送信息
                        if m.Leader == p.me {
                            p.appench <- info{
                                producer:   "Leader",
                                topic_name: m.TopicName,
                                part_name:  m.PartName,
                            }
                        }
                    } else if m.CommandValid && !m.BeLeader {
                        // 处理有效命令，但当前节点不是 Leader
                        start := time.Now()

                        logger.DEBUG_RAFT(logger.DLog, "S%d try lock 847\n", p.me)
                        p.mu.Lock() // 加锁以保护共享状态
                        logger.DEBUG_RAFT(logger.DLog, "S%d success lock 847\n", p.me)

                        ti := time.Since(start).Milliseconds()
                        logger.DEBUG_RAFT(logger.DLog2, "S%d AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA%d\n", p.me, ti)

                        O := m.Command // 获取命令

                        // 检查并初始化 CDM 和 CSM 映射
                        _, ok := p.CDM[O.Tpart]
                        if !ok {
                            logger.DEBUG_RAFT(logger.DLog, "S%d make CDM Tpart(%v)\n", p.me, O.Tpart)
                            p.CDM[O.Tpart] = make(map[string]int64)
                        }
                        _, ok = p.CSM[O.Tpart]
                        if !ok {
                            logger.DEBUG_RAFT(logger.DLog, "S%d make CSM Tpart(%v)\n", p.me, O.Tpart)
                            p.CSM[O.Tpart] = make(map[string]int64)
                        }

                        // 记录当前命令的处理情况
                        logger.DEBUG_RAFT(logger.DLog, "S%d TTT CommandValid(%v) applyindex[%v](%v) CommandIndex(%v) CDM[C%v][%v](%v) O.Cmd_index(%v) from(%v)\n", p.me, m.CommandValid, O.Tpart, p.applyindexs[O.Tpart], m.CommandIndex, O.Tpart, O.Cli_name, p.CDM[O.Tpart][O.Cli_name], O.Cmd_index, O.Ser_index)

                        // 如果命令的索引比当前 applyindex 大 1，则应用此命令
                        if p.applyindexs[O.Tpart]+1 == m.CommandIndex {
                            if O.Cli_name == "TIMEOUT" {
                                logger.DEBUG_RAFT(logger.DLog, "S%d for TIMEOUT update applyindex %v to %v\n", p.me, p.applyindexs[O.Tpart], m.CommandIndex)
                                p.applyindexs[O.Tpart] = m.CommandIndex
                            } else if p.CDM[O.Tpart][O.Cli_name] < O.Cmd_index {
                                // 更新 CDM 和 applyindex，表示命令已处理
                                logger.DEBUG_RAFT(logger.DLeader, "S%d get message update CDM[%v][%v] from %v to %v update applyindex %v to %v\n", p.me, O.Tpart, O.Cli_name, p.CDM[O.Tpart][O.Cli_name], O.Cmd_index, p.applyindexs[O.Tpart], m.CommandIndex)
                                p.applyindexs[O.Tpart] = m.CommandIndex
                                p.CDM[O.Tpart][O.Cli_name] = O.Cmd_index

                                // 如果操作是 Append，则将消息发送到 appench 通道
                                if O.Operate == "Append" {
                                    p.appench <- info{
                                        producer:   O.Cli_name,
                                        message:    O.Msg,
                                        topic_name: O.Topic,
                                        part_name:  O.Part,
                                        size:       O.Size,
                                    }

                                    // 将命令索引写入 Add 通道
                                    select {
                                    case p.Add <- COMD{index: m.CommandIndex}:
                                    default:
                                    }
                                }
                            } else if p.CDM[O.Tpart][O.Cli_name] == O.Cmd_index {
                                logger.DEBUG_RAFT(logger.DLog2, "S%d this cmd had done, the log had two update applyindex %v to %v\n", p.me, p.applyindexs[O.Tpart], m.CommandIndex)
                                p.applyindexs[O.Tpart] = m.CommandIndex
                            } else {
                                logger.DEBUG_RAFT(logger.DLog2, "S%d the topic_partition(%v) producer(%v) OIndex(%v) < CDM(%v)\n", p.me, O.Tpart, O.Cli_name, O.Cmd_index, p.CDM[O.Tpart][O.Cli_name])
                                p.applyindexs[O.Tpart] = m.CommandIndex
                            }

                        } else if p.applyindexs[O.Tpart]+1 < m.CommandIndex {
                            logger.DEBUG_RAFT(logger.DWarn, "S%d the applyindex + 1 (%v) < commandindex(%v)\n", p.me, p.applyindexs[O.Tpart], m.CommandIndex)
                        }

                        // 检查是否需要快照
                        if p.maxraftstate > 0 {
                            p.CheckSnap()
                        }

                        p.mu.Unlock() // 解锁
                        logger.DEBUG_RAFT(logger.DLog, "S%d Unlock 1369\n", p.me)

                    } else { // 处理读取快照
                        r := bytes.NewBuffer(m.Snapshot)
                        d := raft.NewDecoder(r)
                        logger.DEBUG_RAFT(logger.DSnap, "S%d the snapshot applied\n", p.me)
                        var S SnapShot
                        p.mu.Lock() // 加锁以应用快照
                        logger.DEBUG_RAFT(logger.DLog, "S%d lock 1029\n", p.me)
                        if d.Decode(&S) != nil {
                            p.mu.Unlock()
                            logger.DEBUG_RAFT(logger.DLog, "S%d Unlock 1384\n", p.me)
                            logger.DEBUG_RAFT(logger.DSnap, "S%d labgob fail\n", p.me)
                        } else {
                            // 恢复快照中的数据
                            p.CDM[S.Tpart] = S.Cdm
                            p.CSM[S.Tpart] = S.Csm
                            logger.DEBUG_RAFT(logger.DSnap, "S%d recover by SnapShot update applyindex(%v) to %v\n", p.me, p.applyindexs[S.Tpart], S.Apliedindex)
                            p.applyindexs[S.Tpart] = S.Apliedindex
                            p.mu.Unlock()
                            logger.DEBUG_RAFT(logger.DLog, "S%d Unlock 1397\n", p.me)
                        }
                    }

                // 处理超时，向每个分区发出超时操作
                case <-time.After(TIMEOUT * time.Microsecond):
                    O := raft.Op{
                        Ser_index: int64(p.me),
                        Cli_name:  "TIMEOUT",
                        Cmd_index: -1,
                        Operate:   "TIMEOUT",
                    }
                    logger.DEBUG_RAFT(logger.DLog, "S%d have log time applied\n", p.me)
                    p.mu.RLock()
                    for str, raft := range p.Partitions {
                        O.Tpart = str
                        raft.Start(O, false, 0)
                    }
                    p.mu.RUnlock()
                }
            }
        }

    }()
}
```

主要功能

1. 启动 Raft 服务器

   记录日志以表明 Raft 服务器已启动，并启动一个 Goroutine 来处理 Raft 日志和应用消息。

2. 处理来自 applyCh 通道的消息：

   * 成为 Leader：当 ApplyMsg 表示此分区成为 Leader 时，更新相应的 applyindexs。如果当前节点是 Leader，则向 appench 通道发送相关信息。
   * 有效命令处理：处理有效的命令（CommandValid 为 true），检查并更新 CDM（客户端数据映射）和 CSM（客户端状态映射），然后将消息发送到 appench 通道。如果命令的索引比当前 applyindex 大 1，则表示可以处理此命令，并根据命令的操作类型（如 Append）做出相应处理。
   * 快照应用：当接收到快照消息（SnapshotValid 为 true）时，从快照数据中恢复之前的状态，并更新内部状态。

3. 处理超时操作：
   * 定期触发超时操作，向每个分区发出超时日志操作。这个操作可能会触发一些状态更新或其他处理逻辑。
  