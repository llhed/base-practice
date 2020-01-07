Reshard命令迁移槽
```
$ redis-cli --cluster reshard 127.0.0.1:7000
>>> Performing Cluster Check (using node 127.0.0.1:7000)
M: 1ac9fbbfe11362e151204132e3d110b18139a1d9 127.0.0.1:7000
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 09792d31e728ad714a5a90bc7639f277d817fb4e 127.0.0.1:7005
   slots: (0 slots) slave
   replicates a89a427b5fe8b2b0ef07ac8c6252dc3c8efa1f77
S: 2d19dda2a8a790d5636a664fe3ed54aa3dd7677c 127.0.0.1:7003
   slots: (0 slots) slave
   replicates 1ac9fbbfe11362e151204132e3d110b18139a1d9
M: a3c0d3b42da023dc402faf439d4f93a1cb44d402 127.0.0.1:7001
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: a1c1f02eb3e1236f76bb31dbdbd82d73d9fa012c 127.0.0.1:7007
   slots: (0 slots) slave
   replicates 2677387c78c27c55184125161466957b0b52dd61
M: 2677387c78c27c55184125161466957b0b52dd61 127.0.0.1:7006
   slots: (0 slots) master
   1 additional replica(s)
S: 5a4f085dee8400093f45ce2cfa42cbd206167f73 127.0.0.1:7004
   slots: (0 slots) slave
   replicates a3c0d3b42da023dc402faf439d4f93a1cb44d402
M: a89a427b5fe8b2b0ef07ac8c6252dc3c8efa1f77 127.0.0.1:7002
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
#16383 ÷ 3 = 5460   #16383 ÷ 4 = 4096
How many slots do you want to move (from 1 to 16384)? 4096  # 迁移4096个需要自己指定
What is the receiving node ID? 2677387c78c27c55184125161466957b0b52dd61
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: all #选择的是all,即所有有槽的节点都向新节点转移槽

Ready to move 4096 slots.
  Source nodes:
    M: 1ac9fbbfe11362e151204132e3d110b18139a1d9 127.0.0.1:7000
       slots:[0-5460] (5461 slots) master
       1 additional replica(s)
    M: a3c0d3b42da023dc402faf439d4f93a1cb44d402 127.0.0.1:7001
       slots:[5461-10922] (5462 slots) master
       1 additional replica(s)
    M: a89a427b5fe8b2b0ef07ac8c6252dc3c8efa1f77 127.0.0.1:7002
       slots:[10923-16383] (5461 slots) master
       1 additional replica(s)
  Destination node:
    M: 2677387c78c27c55184125161466957b0b52dd61 127.0.0.1:7006
       slots: (0 slots) master
       1 additional replica(s)
  Resharding plan: #下面的区间范围是自己改的，实际上是一行行输出的log，只是计划log,尚未转移
    Moving slot [5461 ~  6826] from a3c0d3b42da023dc402faf439d4f93a1cb44d402
    Moving slot [0 ~ 1364] from 1ac9fbbfe11362e151204132e3d110b18139a1d9
    Moving slot [10923 ~ 12287] from a89a427b5fe8b2b0ef07ac8c6252dc3c8efa1f77
Do you want to proceed with the proposed reshard plan (yes/no)? yes # 给你反悔的机会
    Moving slot [5461 ~  6826] from 127.0.0.1:7001 to 127.0.0.1:7006: 
    Moving slot [0 ~ 1364]  from 127.0.0.1:7000 to 127.0.0.1:7006:
    Moving slot [10923 ~ 12287] from 127.0.0.1:7002 to 127.0.0.1:7006:
#槽里没有数据迁移过程快，有数据迁移过程慢,每个节点都平均迁移点儿。
```

集群缩容
```
$ redis-cli --cluster reshard --cluster-from {node_id_7006} --cluster-to {node_id_7000} --cluster-slots 1365 127.0.0.1:7006
$ redis-cli --cluster reshard --cluster-from {node_id_7006} --cluster-to {node_id_7001} --cluster-yes --cluster-slots 1366 127.0.0.1:7006
$ redis-cli --cluster reshard --cluster-from {node_id_7006} --cluster-to {node_id_7002} --cluster-yes --cluster-slots 1365 127.0.0.1:7006


#1. 先下线从节点，再下线主节点
$ redis-cli --cluster del-node 127.0.0.1:7007 {node_id_7007}
$ redis-cli --cluster del-node 127.0.0.1:7006 {node_id_7006}
$ redis-cli -p 7000 cluster nodes #已经差不到7006，7007节点
```



