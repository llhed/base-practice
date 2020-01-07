1. 查看集群信息  
```
redis-cli --cluster info 127.0.0.1:7001
```
2. 创建集群
```
./redis-cli --cluster create  127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005  127.0.0.1:7006 --cluster-replicas 1 
```    
3. 检查集群
```
./redis-cli --cluster check 127.0.0.1:7001
```
4. 修复异常
```
./redis-cli --cluster check 
```
5. 迁移slot
```
./redis-cli --cluster reshard 127.0.0.1:7001
``` 
6. 均衡slot分配
```
./redis-cli --cluster reblance 127.0.0.1:7001
```
7. 查看所有节点
```
./redis-cli -c -p 7001 cluster nodes
```
8. 删除节点
```
./redis-cli --cluster  del-node 127.0.0.1:7005 c5ae1fec5aab57337a338701cf857eb09e19b244
``` 
9. 添加节点
```
./redis-cli --cluster add-node 127.0.0.1:7005  127.0.0.1:7001
```
10. 均衡slot分配，包括给新加的节点分配
```
./redis-cli  --cluster rebalance --cluster-use-empty-masters --cluster-threshold 1  127.0.0.1:7001
```

