### RabbitMQ集群

#### RabbitMQ始终会记录以下四种内部元数据
* 队列元数据(队列名称和它们的属性)
* 交换器元数据(交换器名称,类型和属性)
* 绑定元数据(一张简单的表格展示了如何将消息路由到队列)
* vhost元数据(为vhost内部队列,交换器和绑定提供命名空间和安全属性)
* 注意:
    单一节点,RabbitMQ会将所有信息存储在内存,同时将那些标记为可持久化的队列和交换器存储到硬盘上
    引入集群,RabbitMQ需要追踪新的元数据类型:集群节点位置,以及节点与已记录的其他类型元数据的关系;集群也提供了选择,将元数据存储到磁盘或者内存中*

#### 集群中的队列
将两个节点组成集群,如果创建队列,集群只会在单个节点而不是所有节点上创建完整的队列信息(元数据,状态,内容),只有队列的所有者知道有关队列的所有信息
所有其他非所有者节点只知道队列的元数据和指向该队列存在的那个节点的指针.因此,当集群节点崩溃时,该节点的队列和关联绑定都消失了.
*注意: 只有队列在最开始没有被设置可持久化时,集群节点崩溃才可以让消费者重连并重新创建队列;如果队列开始已经设置为可持久化,客户端重新声明会得到404错误,这样确保了当失败节点恢复后加入集群,该节点上的队列不会丢失.想要指定队列重回集群的唯一方法是恢复故障节点.*

为什么默认情况下RabbitMQ不将队列内容和状态复制到所有节点:
* 存储空间  如果每个集群节点都拥有所有队列的完全拷贝,那么新计入了节点不会给你带来更多存储空间
* 性能  消息的发布需要将消息复制到每一个集群节点,对持久化消息来说,每一条消息都会触发磁盘IO.每次新节点添加,网络和磁盘负载都会增加

通过设置集群中唯一节点来负责任何特定队列,只有该负责节点才会因队列消息而消耗磁盘IO.所有其他节点需要将接收到该队列的请求转发给该队列的所有者节点.因此,增加Rabbit集群节点意味着你将拥有更多的节点传播队列,这些新节点为你带来了性能提升

#### 内存节点&磁盘节点
每个RabbitMQ节点,不管是单一节点还是集群中的一部分,要么是内存节点,要么是磁盘节点
* 内存节点将所有的队列,交换器,绑定,用户,权限和vhost的元数据定义都仅存储在内存中
* 磁盘节点则将元数据存储在磁盘
*注意:单节点只允许磁盘类型节点,否则你每次重启RabbitMQ,所有关于系统的配置信息都会丢失*
在集群中,你可以选择配置部分节点为内存节点,因为它使得向队列和交换器声明之类的操作更加快速
*注意: 当在集群中声明队列,交换器或绑定时,这些操作直到所有集群节点都成功提交元数据变更后才返回。对内存节点来说,将变更写入内存;对磁盘节点来说,将变更写入磁盘,直到完成之后*

RabbitMQ只要求在集群中至少有一个磁盘节点,所有其他节点都可以是内存节点.当节点加入和离开集群时,它们必须将该变更通知到至少一个磁盘节点
如果集群只有一个磁盘节点,正好它又崩溃了,那么集群还可以路由消息,但不能操作:创建队列,创建交换器,创建绑定,添加用户,更改权限,添加或删除集群节点
解决方法: 建议在集群中设置两台磁盘节点
*注意: 只有一个需要所有的磁盘节点必须在线的操作是添加或者删除集群节点*

当重启内存节点后,它们会连接到预先配置的磁盘节点,下载当前集群元数据拷贝.如果你只讲两个磁盘节点中的一个高速了内存节点,不凑巧这个磁盘节点崩溃了,内存节点重启后就无法找到集群了.所以添加内存节点时,确保告知其所有的磁盘节点.只要内存节点可以找到至少一个磁盘节点,那么它就能在重启后重新加入集群


#### 镜像队列和保留消息
RabbitMQ集群默认情况下队列只存活于集群中的一个节点上.在RabbitMQ 2.6.0版本之后,Rabbit增加了内建的双活冗余选项: 镜像队列
镜像队列: 像普通队列一样,镜像队列的主拷贝仅存在于一个节点上,但与普通队列不同的是,镜像节点在集群中的其他节点上拥有从队列拷贝.一旦队列的主节点不可用,最老的从队列将被选举为新的主队列

例如：
queue_args = {"x-ha-policy":"all"}
channel.queue_declare(queue="hello-queue",arguments=queue_args)

由于队列的镜像性质是由应用程序在允许时指定的,RabbitMQ团队决定你可以同样的方式在允许时指定队列需要镜像到哪些节点.
指定将镜像放在哪个节点:
queue_args = {"x-ha-policy":"nodes","x-ha-policy-params":["rabbit@localhost"]}
channel.queue_declare(queue="hello-queue",arguments=queue_args)

*注意: 新增的从拷贝只会包含那些其添加进来之后从镜像队列发来的消息.RabbitMQ不会将镜像队列现已经存在的内容与新添加的从拷贝进行同步. 新增的从拷贝最终会和现存的队列拷贝拥有相同的状态.*

### 镜像模式
```
镜像模式:

镜像模式和普通模式的区别:
队列的数据都镜像了一份到所有的节点上.这样任何一个节点失效,不会影响整个集群的使用

在实现上,镜像队列内部有一套选举算法,会选出一个master和若干个slaver.master和slaver通过相互间不断发送心跳来检查是否连接断开.可以通过指定net_ticktime来控制心跳检查频率.注意一个单位时间net_ticktime实际上做了4次交互,故当超过net_ticktime(± 25%)秒没有响应的话则认为节点挂掉.另外注意修改net_ticktime时需要所有节点都一致:
配置举例:
[
  {rabbit, [{tcp_listeners, [5672]}]},
  {kernel, [{net_ticktime,  120}]}
].

consumer,任意连接一个节点,若连上的不是master,请求会转发给master,为了保证消息的可靠性,consumer回复ack给master后,master删除消息并广播所有的slaver去删除.

publisher,任意连接一个节点,若连上的不是master,则转发给master,由master存储并转发给其他的slaver存储.如果master挂掉,则从slaver中选择消息队列最长的为master,在这种情况下可以存在消息未同步,给ack消息未同步的情况,会造成消息重发(默认是异步同步的).
总共有以下几件事情发生：
1.1个最老的(队列最长的)的slaver提升为master,如果没有一个slaver是和master同步的则会造成消息丢失
2.要提升为master的slaver会认为以前所有连接挂掉的master的消费者都断开了连接.那么存在clinet发送了ack的消息但还在路上是master挂掉的情况,或者master收到了ack但是在广播给slaver的时候master挂掉的情况,所以新的master别无选择,只能认为消息没有被确认.他会requeue他认为没有ack的消息.那么client可能就收到了重复的消息,并要再次发送ack
3.从镜像队列中消费的client支持了consumer Cancellation通知的,将收到通知并订阅的mirrored-queue被取消了,这是因为该mirrored-queue升级成了master,这是client需要重现去找mirrored-queue上消费,这样就避免了client继续发送ack到老的挂掉的master上.避免收到新的master发送的相同的消息.
4.如果noAck=true,且在mirrored-queue上消费,那么在切换时由于服务器是先ack然后发送到noAck=true的消费者,这时连接断开可能导致该数据丢失

如果slaver挂掉,则集群的节点状态没有任何变化.只要client没有连到这个节点上,也不会给client发送失败的通知.在检测到slaver挂掉的期间publish消息会有延迟.如果配置了高可用策略是自动同步,当slaver起来后,队列中有大量的消息需要同步,将会整个集群阻塞长时间的不能读写直到同步结束.

这两个挂掉的情况都需要客户端镜像容错,比如在连接断开的时候进行重连(官方的Java和.net 客户端提供了callback方法在监听到链接失败的时候调用.Java在Connection和channel类中提供了ShutdownListener 的callback方法,.net client在IConnecton中提供了ConnectionShuedown在Imodel中提供了ImodelShutdown事件供调用).也可以在client和server之间加入LoadBalancer.比如haproxy做负载均衡

指定mirror策略:
有三种策略:
* all
  队列将mirrored到所有集群中的节点中,当新节点添加进来时也会mirrored到新的节点
* exactly(需指定count)
  如果节点数小于count数,则队列将mirrored到所有的节点.如果节点数大于count,新的节点将不再创建队列的mirror(即使原来已创建mirror的节点挂掉也不会创建)
* nodes
  对指定的节点进行mirror.如果没有一个指定的节点在运行中,那么只有client连接的那个节点才会声明queue(这里有个迁移策略:假如queue是在[A,B]上且A为master,若给定的新的策略为nodes[C,D],那么为了防止数据丢失,在迁移中会同时存在[A,C,D]直到C,D已经同步好以后,A才会关闭)

配置举例:
设置queue的名称为ha.的为高可用:
linux:rabbitmqctl set_policy ha-all "^ha\." '{"ha-mode":"all"}'
win:rabbitmqctl set_policy ha-all "^ha\." "{""ha-mode"":""all""}"
http api:PUT /api/policies/%2f/ha-all {"pattern":"^ha\.", "definition":{"ha-mode":"all"}}
web ui:
1:Navigate to Admin > Policies > Add / update a policy.
2:Enter "ha-all" next to Name, "^ha\." next to Pattern, and "ha-mode" = "all" in the first line next to Policy.
3:Click Add policy.

举例2：
rabbitmqctl set_policy ha-two "^two\." \
   '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'

自动或手动同步：
你可以查看哪些slave已经同步好了：
rabbitmqctl list_queues name slave_pids synchronised_slave_pids

你可以手动同步(默认手动同步)：
rabbitmqctl sync_queue name

你可以取消自动同步：
rabbitmqctl cancel_sync_queue name
一个没有同步的mirror，它仍然会同步后续插入队列的数据，但是队列前面的数据却没有。但是随着队列的不断消费，导致空缺的部分的消息被消费掉了，此时mirror也可以是同步了的。
```