#### 关键词
```
Kafka部分名词解释如下:
1.Broker
  消息中间件处理结点,一个Kafka节点就是一个broker,多个broker可以组成一个Kafka集群
2.Topic
  一类消息,例如page view日志,click日志等都可以以topic的形式存在,Kafka集群能够同时负责多个topic的分发
3.Partition
  topic物理上的分组,一个topic可以分为多个partition,每个partition是一个有序的队列
4.Segment
  partition物理上由多个segment组成
5.offset
  每个partition都由一系列有序的,不可变的消息组成,这些消息被连续的追加到partition中.partition中的每个消息都有一个连续的序列号叫做offset,用于partition唯一标识一条消息.

分析过程分为以下4个步骤:
1.topic中partition存储分布
2.partiton中文件存储方式
3.partiton中segment文件存储结构
4.在partition中如何通过offset查找message
```

#### kafka Partition Recovery机制
```
每个Partition会在磁盘记录一个RecoveryPoint, 记录已经flush到磁盘的最大offset。当broker fail 重启时,会进行loadLogs。 首先会读取该Partition的RecoveryPoint,找到包含RecoveryPoint的segment及以后的segment, 这些segment就是可能没有 完全flush到磁盘segments。然后调用segment的recover,重新读取各个segment的msg,并重建索引

优点
1.以segment为单位管理Partition数据,方便数据生命周期的管理,删除过期数据简单
2.在程序崩溃重启时,加快recovery速度,只需恢复未完全flush到磁盘的segment
3.通过index中offset与物理偏移映射,用二分查找能快速定位msg,并且通过分多个Segment,每个index文件很小,查找速度更快。
```

#### kafka存储过程
```
Kafka运行时很少有大量读磁盘的操作,主要是定期批量写磁盘操作,因此操作磁盘很高效.这跟Kafka文件存储中读写message的设计是息息相关的.Kafka中读写message有如下特点:
写message:
1.消息从java堆转入page cache(即物理内存)
2.由异步线程刷盘,消息从page cache刷入磁盘

读message:
1.消息直接从page cache转入socket发送出去
2.当从page cache没有找到相应数据时,此时会产生磁盘IO,从磁盘Load消息到page cache,然后直接从socket发出去
```

#### kafka Partition Recovery机制
```
每个Partition会在磁盘记录一个RecoveryPoint, 记录已经flush到磁盘的最大offset。当broker fail 重启时,会进行loadLogs。 首先会读取该Partition的RecoveryPoint,找到包含RecoveryPoint的segment及以后的segment, 这些segment就是可能没有 完全flush到磁盘segments。然后调用segment的recover,重新读取各个segment的msg,并重建索引

优点
1.以segment为单位管理Partition数据,方便数据生命周期的管理,删除过期数据简单
2.在程序崩溃重启时,加快recovery速度,只需恢复未完全flush到磁盘的segment
3.通过index中offset与物理偏移映射,用二分查找能快速定位msg,并且通过分多个Segment,每个index文件很小,查找速度更快。
```

#### kafka Partition Replica同步机制
```
1.Partition的多个replica中一个为Leader,其余为follower
2.Producer只与Leader交互,把数据写入到Leader中
3.Followers从Leader中拉取数据进行数据同步
4.Consumer只从Leader拉取数据

ISR:所有不落后的replica集合,不落后有两层含义:距离上次FetchRequest的时间不大于某一个值或落后的消息数不大于某一个值, Leader失败后会从ISR中选取一个Follower做Leader
```

#### 数据一致性保证
```
一致性定义:若某条消息对Consumer可见,那么即使Leader宕机了,在新Leader上数据依然可以被读到
1.HighWaterMark简称HW:
  Partition的高水位,取一个partition对应的ISR中最小的LEO作为HW,消费者最多只能消费到HW所在的位置,另外每个replica都有highWatermark,leader和follower各自负责更新自己的highWatermark状态,highWatermark <= leader.LogEndOffset
2.对于Leader新写入的msg,Consumer不能立刻消费,Leader会等待该消息被所有ISR中的replica同步后,更新HW,此时该消息才能被Consumer消费,即Consumer最多只能消费到HW位置

这样就保证了如果Leader Broker失效,该消息仍然可以从新选举的Leader中获取.对于来自内部Broker的读取请求,没有HW的限制.同时,Follower也会维护一份自己的HW,Folloer.HW = min(Leader.HW,Follower.offset)
```

#### 数据可靠性
```
当Producer向Leader发送数据时,可以通过acks参数设置数据可靠性的级别
1.0: 不论写入是否成功,server不需要给Producer发送Response,如果发生异常,server会终止连接,触发Producer更新meta数据;
2.1: Leader写入成功后即发送Response,此种情况如果Leader fail,会丢失数据
3.-1: 等待所有ISR接收到消息后再给Producer发送Response,这是最强保证
      仅设置acks=-1也不能保证数据不丢失,当Isr列表中只有Leader时,同样有可能造成数据丢失.要保证数据不丢除了设置acks=-1, 还要保证ISR的大小大于等于2,具体参数设置:
      1.request.required.acks:设置为-1等待所有ISR列表中的Replica接收到消息后采算写成功;
      2.min.insync.replicas: 设置为大于等于2,保证ISR中至少有两个Replica
      Producer要在吞吐率和数据可靠性之间做一个权衡 
```


