## redis install

##### redis 环境
```
[root@localhost app]# echo "net.core.somaxconn = 65535" | tee -a /etc/sysctl.conf
[root@localhost app]# echo "vm.overcommit_memory = 1" | tee -a /etc/sysctl.conf
[root@localhost app]# sysctl -p

[root@localhost app]# echo never > /sys/kernel/mm/transparent_hugepage/enabled
[root@localhost app]# cat >> /etc/rc.local <<EOF
if test -f /sys/kernel/mm/transparent_hugepage/enabled;then
   echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi
EOF
[root@localhost app]# chmod +x /etc/rc.d/rc.local
```

##### redis cluster install
```
[root@localhost app]# tar xzf redis-3.2.6.tar.gz
[root@localhost app]# cd redis-3.2.6/
[root@localhost redis-3.2.6]# make MALLOC=libc PREFIX=/mnt/app/redis install

[root@localhost redis-3.2.6]# echo 'export REDIS_HOME=/mnt/app/redis'|tee /etc/profile.d/redis.sh
[root@localhost redis-3.2.6]# echo 'export REDIS_BIN=$REDIS_HOME/bin'|tee -a /etc/profile.d/redis.sh
[root@localhost redis-3.2.6]# echo 'export PATH=$REDIS_BIN:$PATH'|tee -a /etc/profile.d/redis.sh
[root@localhost redis-3.2.6]# source /etc/profile

[root@localhost redis-3.2.6]# mkdir -p /mnt/app/redis/conf/{7001,7002,7003,7004,7005,7006}
[root@localhost redis-3.2.6]# mkdir -p /mnt/data/redis/{7001,7002,7003,7004,7005,7006}
[root@localhost redis-3.2.6]# mkdir -p /mnt/log/redis/{7001,7002,7003,7004,7005,7006}
[root@localhost redis-3.2.6]# cp redis.conf /mnt/app/redis/conf/{7001,7002,7003,7004,7005,7006}/redis.conf

[root@localhost redis-3.2.6]# chown -R wisdom.wisdom /mnt/app/redis/conf
[root@localhost redis-3.2.6]# chown -R wisdom.wisdom /mnt/data/redis
[root@localhost redis-3.2.6]# chown -R wisdom.wisdom /mnt/log/redis

[root@localhost redis-3.2.6]# vim /mnt/app/redis/conf/7001/redis.conf
port 7001
cluster-enabled yes
cluster-config-file /mnt/data/redis/7001/nodes.conf
cluster-node-timeout 5000
appendonly yes
dir /mnt/data/redis/7001
unixsocket /mnt/data/redis/7001/redis.sock
pidfile /mnt/data/redis/7001/redis.pid
logfile /mnt/log/redis/7001/redis.log

[root@localhost redis-3.2.6]# vim /mnt/app/redis/conf/7002/redis.conf
port 7002
cluster-enabled yes
cluster-config-file /mnt/data/redis/7002/nodes.conf
cluster-node-timeout 5000
appendonly yes
dir /mnt/data/redis/7002
unixsocket /mnt/data/redis/7002/redis.sock
pidfile /mnt/data/redis/7002/redis.pid
logfile /mnt/log/redis/7002/redis.log

[root@localhost redis-3.2.6]# vim /mnt/app/redis/conf/7003/redis.conf
port 7003
cluster-enabled yes
cluster-config-file /mnt/data/redis/7003/nodes.conf
cluster-node-timeout 5000
appendonly yes
dir /mnt/data/redis/7003
unixsocket /mnt/data/redis/7003/redis.sock
pidfile /mnt/data/redis/7003/redis.pid
logfile /mnt/log/redis/7003/redis.log

[root@localhost redis-3.2.6]# vim /mnt/app/redis/conf/7004/redis.conf
port 7004
cluster-enabled yes
cluster-config-file /mnt/data/redis/7004/nodes.conf
cluster-node-timeout 5000
appendonly yes
dir /mnt/data/redis/7004
unixsocket /mnt/data/redis/7004/redis.sock
pidfile /mnt/data/redis/7004/redis.pid
logfile /mnt/log/redis/7004/redis.log

[root@localhost redis-3.2.6]# vim /mnt/app/redis/conf/7005/redis.conf
port 7005
cluster-enabled yes
cluster-config-file /mnt/data/redis/7005/nodes.conf
cluster-node-timeout 5000
appendonly yes
dir /mnt/data/redis/7005
unixsocket /mnt/data/redis/7005/redis.sock
pidfile /mnt/data/redis/7005/redis.pid
logfile /mnt/log/redis/7005/redis.log

[root@localhost redis-3.2.6]# vim /mnt/app/redis/conf/7006/redis.conf
port 7006
cluster-enabled yes
cluster-config-file /mnt/data/redis/7006/nodes.conf
cluster-node-timeout 5000
appendonly yes
dir /mnt/data/redis/7006
unixsocket /mnt/data/redis/7006/redis.sock
pidfile /mnt/data/redis/7006/redis.pid
logfile /mnt/log/redis/7006/redis.log

[wisdom@localhost ~]$ /mnt/app/redis/bin/redis-server /mnt/app/redis/conf/{7001,7002,7003,7004,7005,7006}/redis.conf &

```

##### ruby 工具
```
[root@localhost app]# yum -y install ruby rubygem-redis
[root@localhost app]# tar xzf redis-3.2.6.tar.gz
[root@localhost app]# cd redis-3.2.6/src
[root@localhost src]# ./redis-trib.rb create --replicas 1  127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 127.0.0.1:7006

[root@localhost src]# ./redis-trib.rb check 127.0.0.1:7001
[root@localhost src]# ./redis-trib.rb info 127.0.0.1:7001
```