对于集群中的每一个broker都保存着相同的完整的整个集群的metadata信息;

1.metadata信息里包括了每个topic的所有partition的信息: leader, leader_epoch, controller_epoch, isr, replicas等;
+ ISR(in-Sync replicas ）里面维护的是所有与leader数据差异在阈值范围内的副本所在的broker id列表;
+ "controller_epoch": 表示kafka集群中的中央控制器选举次数,
+ "leader": 表示该partition选举leader的brokerId,
+ "version": 版本编号默认为1,
+ "leader_epoch": 该partition leader选举次数,
+ "isr"(in-Sync replicas):里面维护的是所有与leader数据差异在阈值范围内的副本所在的broker id列表;

Kafka客户端从任一broker都可以获取到需要的metadata信息;

#### Metadata的存储在哪里 --- MetadataCache组件
+ 在每个Broker的KafkaServer对象中都会创建MetadataCache组件, 负责缓存所有的metadata信息;
```scale
    val metadataCache: MetadataCache = new MetadataCache(config.brokerId)
```
+ 所在文件: core/src/main/scala/kafka/server/MetadataCache.scala
+ 所有的metadata信息存储在map里, key是topic, value又是一个map, 其中key是parition id, value是PartitionStateInfo
```scale
    private val cache = mutable.Map[String, mutable.Map[Int, PartitionStateInfo]]()
```
+ PartitionStateInfo: 包括LeaderIsrAndControllerEpoch和Replica数组;
2.MetadataCache还保存着推送过来的有效的broker信息;
```scale
    private val aliveBrokers = mutable.Map[Int, Broker]()
```
#### MetadataCache如何获取和更新metadata信息
1.KafkaApis对象处理UpdateMetadataRequest
```scale
    case ApiKeys.UPDATE_METADATA_KEY => handleUpdateMetadataRequest(request)
```
2.handleUpdateMetadataRequest
```scale
    def handleUpdateMetadataRequest(request: RequestChannel.Request) {
        val correlationId = request.header.correlationId
        val updateMetadataRequest = request.body.asInstanceOf[UpdateMetadataRequest]
    
        val updateMetadataResponse =
          if (authorize(request.session, ClusterAction, Resource.ClusterResource)) {
            replicaManager.maybeUpdateMetadataCache(correlationId, updateMetadataRequest, metadataCache)
            new UpdateMetadataResponse(Errors.NONE.code)
          } else {
            new UpdateMetadataResponse(Errors.CLUSTER_AUTHORIZATION_FAILED.code)
          }
    
        val responseHeader = new ResponseHeader(correlationId)
        requestChannel.sendResponse(new Response(request, new ResponseSend(request.connectionId, responseHeader, updateMetadataResponse)))
      }
```
可以看到是调用了ReplicaManager.maybeUpdateMetadataCache方法, 里面又会调用到MetadataCache.updateCache方法
3.MetadataCache.updateCache
```scale
    def updateCache(correlationId: Int, updateMetadataRequest: UpdateMetadataRequest) {
        inWriteLock(partitionMetadataLock) {
          controllerId = updateMetadataRequest.controllerId match {
              case id if id < 0 => None
              case id => Some(id)
            }
          aliveNodes.clear()
          aliveBrokers.clear()
          updateMetadataRequest.liveBrokers.asScala.foreach { broker =>
            val nodes = new EnumMap[SecurityProtocol, Node](classOf[SecurityProtocol])
            val endPoints = new EnumMap[SecurityProtocol, EndPoint](classOf[SecurityProtocol])
            broker.endPoints.asScala.foreach { case (protocol, ep) =>
              endPoints.put(protocol, EndPoint(ep.host, ep.port, protocol))
              nodes.put(protocol, new Node(broker.id, ep.host, ep.port))
            }
            aliveBrokers(broker.id) = Broker(broker.id, endPoints.asScala, Option(broker.rack))
            aliveNodes(broker.id) = nodes.asScala
          }
    
          updateMetadataRequest.partitionStates.asScala.foreach { case (tp, info) =>
            val controllerId = updateMetadataRequest.controllerId
            val controllerEpoch = updateMetadataRequest.controllerEpoch
            if (info.leader == LeaderAndIsr.LeaderDuringDelete) {
              removePartitionInfo(tp.topic, tp.partition)
              stateChangeLogger.trace(s"Broker $brokerId deleted partition $tp from metadata cache in response to UpdateMetadata " +
                s"request sent by controller $controllerId epoch $controllerEpoch with correlation id $correlationId")
            } else {
              val partitionInfo = partitionStateToPartitionStateInfo(info)
              addOrUpdatePartitionInfo(tp.topic, tp.partition, partitionInfo)
              stateChangeLogger.trace(s"Broker $brokerId cached leader info $partitionInfo for partition $tp in response to " +
                s"UpdateMetadata request sent by controller $controllerId epoch $controllerEpoch with correlation id $correlationId")
            }
          }
        }
      }
```
+ 更新aliveBrokers;
+ 如果某个topic的的parition的leader是无效的, 则removePartitionInfo(tp.topic, tp.partition);
+ 新增或更新某个topic的某个parition的信息, addOrUpdatePartitionInfo(tp.topic, tp.partition, info):将信息meta信息保存到MetadataCache的cache对象中;

#### Metadata信息从哪里来
+ 这个问题实际上就是在问UpdateMetaRequest是谁在什么时候发送的;
+ 来源肯定是KafkaController发送的;
+ broker变动, topic创建, partition增加等等时机都需要更新metadata;

#### 谁使用metadata信息
+ 主要是客户端, 客户端从metadata中获取topic的partition信息, 知道leader是谁, 才可以发送和消费msg;
+ KafkaApis对象处理MetadataRequest
```scale
    case ApiKeys.METADATA => handleTopicMetadataRequest(request)
```
+ KafkaApis.handleTopicMetadataRequest
```scale
    def handleTopicMetadataRequest(request: RequestChannel.Request) {
        val metadataRequest = request.body.asInstanceOf[MetadataRequest]
        val requestVersion = request.header.apiVersion()
    
        val topics =
          // Handle old metadata request logic. Version 0 has no way to specify "no topics".
          if (requestVersion == 0) {
            if (metadataRequest.topics() == null || metadataRequest.topics().isEmpty)
              metadataCache.getAllTopics()
            else
              metadataRequest.topics.asScala.toSet
          } else {
            if (metadataRequest.isAllTopics)
              metadataCache.getAllTopics()
            else
              metadataRequest.topics.asScala.toSet
          }
    
        var (authorizedTopics, unauthorizedTopics) =
          topics.partition(topic => authorize(request.session, Describe, new Resource(Topic, topic)))
    
        if (authorizedTopics.nonEmpty) {
          val nonExistingTopics = metadataCache.getNonExistingTopics(authorizedTopics)
          if (config.autoCreateTopicsEnable && nonExistingTopics.nonEmpty) {
            authorizer.foreach { az =>
              if (!az.authorize(request.session, Create, Resource.ClusterResource)) {
                authorizedTopics --= nonExistingTopics
                unauthorizedTopics ++= nonExistingTopics
              }
            }
          }
        }
    
        val unauthorizedTopicMetadata = unauthorizedTopics.map(topic =>
          new MetadataResponse.TopicMetadata(Errors.TOPIC_AUTHORIZATION_FAILED, topic, common.Topic.isInternal(topic),
            java.util.Collections.emptyList()))
    
        // In version 0, we returned an error when brokers with replicas were unavailable,
        // while in higher versions we simply don't include the broker in the returned broker list
        val errorUnavailableEndpoints = requestVersion == 0
        val topicMetadata =
          if (authorizedTopics.isEmpty)
            Seq.empty[MetadataResponse.TopicMetadata]
          else
            getTopicMetadata(authorizedTopics, request.securityProtocol, errorUnavailableEndpoints)
    
        val completeTopicMetadata = topicMetadata ++ unauthorizedTopicMetadata
    
        val brokers = metadataCache.getAliveBrokers
    
        trace("Sending topic metadata %s and brokers %s for correlation id %d to client %s".format(completeTopicMetadata.mkString(","),
          brokers.mkString(","), request.header.correlationId, request.header.clientId))
    
        val responseHeader = new ResponseHeader(request.header.correlationId)
    
        val responseBody = new MetadataResponse(
          brokers.map(_.getNode(request.securityProtocol)).asJava,
          metadataCache.getControllerId.getOrElse(MetadataResponse.NO_CONTROLLER_ID),
          completeTopicMetadata.asJava,
          requestVersion
        )
        requestChannel.sendResponse(new RequestChannel.Response(request, new ResponseSend(request.connectionId, responseHeader, responseBody)))
      }
```
1.先确定需要获取哪些topic的metadata信息,  如果request里未指定topic, 则获取当前所有的topic的metadata信息;

2.有效性验证,将topic分为authorizedTopics和unauthorizedTopics;

3.获取authorizedTopics的metadata, 注意getTopicMetadata方法是关键所在, 它会先筛选出当前不存在的topic, 如果auto.create.topics.enable=true, 则调用AdminUtils.createTopic先创建此topic, 但此时其PartitionStateInfo为空, 不过也会作为Metadata Response的一部分返回给客户端.