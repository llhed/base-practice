#### 1. RabbitMQ 安装
```
[root@localhost app]# xz -d rabbitmq-server-generic-unix-3.6.6.tar.xz
[root@localhost app]# tar xf rabbitmq-server-generic-unix-3.6.6.tar
[root@localhost app]# mv rabbitmq_server-3.6.6 /mnt/app/rabbitmq
[root@localhost app]# chown -R root.root /mnt/app/rabbitmq
```
#### 2. 安装rabbitmq_management插件
```
[root@localhost app]# /mnt/app/rabbitmq/sbin/rabbitmq-plugins enable rabbitmq_management
```
#### 3. RabbitMQ 环境变量
```
[root@localhost app]# echo 'export RABBITMQ_HOME=/mnt/app/rabbitmq' | tee /etc/profile.d/rabbitmq.sh
[root@localhost app]# echo 'export RABBITMQ_BIN=${RABBITMQ_HOME}/sbin' | tee -a /etc/profile.d/rabbitmq.sh                    
[root@localhost app]# echo 'export PATH=${RABBITMQ_BIN}:$PATH' | tee -a /etc/profile.d/rabbitmq.sh                     
[root@localhost app]# source /etc/profile
```
#### 4. RabbitMQ 创建配置文件和目录(数据+日志)
```
[root@localhost app]# touch /mnt/app/rabbitmq/etc/rabbitmq/rabbitmq-env.conf
[root@localhost app]# touch /mnt/app/rabbitmq/etc/rabbitmq/rabbitmq.config
[root@localhost app]# chown -R wisdom.wisdom /mnt/app/rabbitmq/etc/

[root@localhost app]# mkdir -p /mnt/{data,log}/rabbitmq
[root@localhost app]# mkdir -p /mnt/data/rabbitmq/mnesia
[root@localhost app]# chown -R wisdom.wisdom /mnt/{data,log}/rabbitmq
```
#### 5. 集群主机名设置
```
RabbitMQ cluster-1:
[root@localhost app]# hostname rabbit228
[root@localhost app]# echo 'rabbit228' |tee /etc/hostname
[root@localhost app]# echo '192.168.18.228  rabbit228' |tee -a /etc/hosts
[root@localhost app]# echo '192.168.18.229  rabbit229' |tee -a /etc/hosts  
[root@localhost app]# echo '192.168.18.230  rabbit230' |tee -a /etc/hosts

RabbitMQ cluster-2:
[root@localhost app]# hostname rabbit229
[root@localhost app]# echo 'rabbit229' |tee /etc/hostname
[root@localhost app]# echo '192.168.18.228  rabbit228' |tee -a /etc/hosts
[root@localhost app]# echo '192.168.18.229  rabbit229' |tee -a /etc/hosts  
[root@localhost app]# echo '192.168.18.230  rabbit230' |tee -a /etc/hosts

RabbitMQ cluster-3:
[root@localhost app]# hostname rabbit230
[root@localhost app]# echo 'rabbit230' |tee /etc/hostname
[root@localhost app]# echo '192.168.18.228  rabbit228' |tee -a /etc/hosts
[root@localhost app]# echo '192.168.18.229  rabbit229' |tee -a /etc/hosts  
[root@localhost app]# echo '192.168.18.230  rabbit230' |tee -a /etc/hosts
```
#### 6. 集群配置
```
//cluster-X:(三台机器都要执行)
[root@localhost app]# su - wisdom
[wisdom@localhost ~]$ cat > /mnt/app/rabbitmq/etc/rabbitmq/rabbitmq-env.conf <<EOF
> RABBITMQ_NODE_IP_ADDRESS=
> RABBITMQ_NODE_PORT=5672
> RABBITMQ_DIST_PORT=25672
> RABBITMQ_NODENAME=rabbit@\$HOSTNAME
> RABBITMQ_MNESIA_BASE=/mnt/data/rabbitmq/mnesia
> RABBITMQ_LOG_BASE=/mnt/log/rabbitmq
> EOF

[wisdom@rabbitX ~]$ cat > /mnt/app/rabbitmq/etc/rabbitmq/rabbitmq.config <<EOF
> [
>  {rabbit,
>   [
>   ]},
>  {kernel,
>   [
>   ]},
>  {rabbitmq_management,
>   [
>   ]},
>  {rabbitmq_shovel,
>   [{shovels,
>     [
>     ]}
>   ]},
>  {rabbitmq_stomp,
>   [
>   ]},
>  {rabbitmq_mqtt,
>   [
>   ]},
>  {rabbitmq_amqp1_0,
>   [
>   ]},
>  {rabbitmq_auth_backend_ldap,
>   [
>   ]}
> ].
> EOF
```
#### 7. 启动服务
```
[wisdom@rabbitX ~]$ /mnt/app/rabbitmq/sbin/rabbitmq-server &
[wisdom@rabbitX ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl stop
[wisdom@rabbitX ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl status
```
#### 8. RabbitMQ 集群操作:将要加入集群的机器都使用一个cookie(在三台机器中任选一台)
```
//在228上找到elang.cookie
[wisdom@rabbit228 ~]$ cat ~/.erlang.cookie
HBSJBDVPUCEITCQZQQDB

//将.erlang.cookie内容copy到229和230这两台机器的~/.erlang.cookie文件中
[wisdom@rabbit229 ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl stop
[wisdom@rabbit229 ~]$ chmod 600 ~/.erlang.cookie
[wisdom@rabbit229 ~]$ echo HBSJBDVPUCEITCQZQQDB | tee ~/.erlang.cookie
[wisdom@rabbit229 ~]$ chmod 400 ~/.erlang.cookie  

[wisdom@rabbit230 ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl stop
[wisdom@rabbit230 ~]$ chmod 600 ~/.erlang.cookie
[wisdom@rabbit230 ~]$ echo HBSJBDVPUCEITCQZQQDB | tee ~/.erlang.cookie
[wisdom@rabbit230 ~]$ chmod 400 ~/.erlang.cookie

//修改完~/.erlang.cookie后,启动RabbitMQ服务
[wisdom@rabbit229 ~]$ /mnt/app/rabbitmq/sbin/rabbitmq-server &
[wisdom@rabbit230 ~]$ /mnt/app/rabbitmq/sbin/rabbitmq-server &
```
#### 9. RabbitMQ 集群操作: 将229节点加入到228 RabbitMQ集群中(磁盘存储)
```
//查看228节点的集群状态
[wisdom@rabbit229 ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl -n rabbit@rabbit228 cluster_status
Cluster status of node rabbit@rabbit228 ...
[{nodes,[{disc,[rabbit@rabbit228,rabbit@rabbit229]}]},
 {running_nodes,[rabbit@rabbit228]},
 {cluster_name,<<"rabbit@rabbit228">>},
 {partitions,[]},
 {alarms,[{rabbit@rabbit228,[]}]}]

//停止229节点的Rabbitmq服务
[wisdom@rabbit229 ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl stop_app

//清空229 Rabbit元数据
[wisdom@rabbit229 ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl -n rabbit@rabbit229 reset

//将229 Rabbit加入到228集群中
[wisdom@rabbit229 ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl join_cluster rabbit@rabbit228

//启动229 Rabbit服务
[wisdom@rabbit229 ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl start_app

//查看集群状态
[wisdom@rabbit229 ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl -n rabbit@rabbit228 cluster_status
Cluster status of node rabbit@rabbit228 ...
[{nodes,[{disc,[rabbit@rabbit228,rabbit@rabbit229]}]},
 {running_nodes,[rabbit@rabbit228]},
 {cluster_name,<<"rabbit@rabbit228">>},
 {partitions,[]},
 {alarms,[{rabbit@rabbit228,[]}]}]
```
#### 10. RabbitMQ 集群操作: 将230 RabbitMQ加入到228 RabbitMQ集群中(内存存储)
```
//停止230节点的Rabbitmq服务
[wisdom@rabbit230 ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl stop_app

//清空230 Rabbit元数据
[wisdom@rabbit230 ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl -n rabbit@rabbit230 reset

//将230 Rabbit加入到228集群中,内存存储
[wisdom@rabbit230 ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl join_cluster rabbit@rabbit228 --ram

//启动230 Rabbit服务
[wisdom@rabbit230 ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl start_app

////查看集群状态
[wisdom@rabbit229 ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl -n rabbit@rabbit228 cluster_status
Cluster status of node rabbit@rabbit228 ...
[{nodes,[{disc,[rabbit@rabbit228,rabbit@rabbit229]},{ram,[rabbit@rabbit230]}]},
 {running_nodes,[rabbit@rabbit230,rabbit@rabbit229,rabbit@rabbit228]},
 {cluster_name,<<"rabbit@rabbit228">>},
 {partitions,[]},
 {alarms,[{rabbit@rabbit230,[]},{rabbit@rabbit229,[]},{rabbit@rabbit228,[]}]}]

```
#### 10. 设置集群中所有的队列为镜像队列(任意一台机器上执行)
```
[wisdom@rabbit228 ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}'
```
#### 11. 设置用户权限
```
[wisdom@rabbit228 ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl add_user test test123  
[wisdom@rabbit229 ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl set_user_tags test administrator
[wisdom@rabbit228 ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl set_permissions -p / test ".\*" ".\*" ".\*"
```
#### 12. 安装haproxy,作为反向代理
```
//haproxy 安装
[root@localhost app]# useradd -s /sbin/nologin haproxy

[root@localhost app]# tar xzf haproxy-1.7.2.tar.gz
[root@localhost app]# cd haproxy-1.7.2
[root@localhost haproxy-1.7.2]# make TARGET=generic PREFIX=/mnt/app/haproxy
[root@localhost haproxy-1.7.2]# make install PREFIX=/mnt/app/haproxy

//haproxy 环境变量设置
[root@localhost haproxy-1.7.2]# echo 'export HAPROXY_HOME=/mnt/app/haproxy' | tee /etc/profile.d/haproxy.sh   
[root@localhost haproxy-1.7.2]# echo 'export HAPROXY_BIN=${HAPROXY_HOME}/sbin' | tee -a /etc/profile.d/haproxy.sh
[root@localhost haproxy-1.7.2]# echo 'export PATH=${HAPROXY_BIN}:$PATH' | tee -a /etc/profile.d/haproxy.sh       
[root@localhost haproxy-1.7.2]# source /etc/profile

//haproxy 创建配置文件
[root@localhost haproxy-1.7.2]# mkdir -p /mnt/app/haproxy/conf
[root@localhost haproxy-1.7.2]# touch /mnt/app/haproxy/conf/haproxy.cfg

//haproxy 配置
[root@localhost app]# vim /mnt/app/haproxy/conf/haproxy.cfg
global
    log /mnt/log/haproxy 127.0.0.1 local0
    log /mnt/log/haproxy 127.0.0.1 local1 notice
    stats socket /mnt/log/haproxy/haproxy.socket mode 770 level admin
    pidfile /mnt/log/haproxy/haproxy.pid
    maxconn 5000
    user haproxy
    group haproxy
    deamon

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    retries 3
    option redispatch
    timeout connect 5s
    timeout client  120s
    timeout server  120s

listen haproxy_stats
    bind 0.0.0.0:8080
    stats refresh 30s
    stats uri /haproxy?stats
    stats realm Haproxy Manager
    stats auth admin:admin
    stats hide-version

listen rabbitmq_admin
    bind 0.0.0.0:8090
    server rabbit228 192.168.18.228:15672
    server rabbit229 192.168.18.229:15672
    server rabbit230 192.168.18.230:15672

listen rabbitmq_cluster
    bind 0.0.0.0:5672
    mode tcp
    option tcplog
    option clitcpka
    timeout client  3h
    timeout server  3h
    balance roundrobin
    server rabbit228 192.168.18.228:15672 check inter 5s rise 2 fall 3
    server rabbit229 192.168.18.229:15672 check inter 5s rise 2 fall 3
    server rabbit230 192.168.18.230:15672 check inter 5s rise 2 fall 3

//haproxy 启动
[root@localhost haproxy-1.7.2]# /mnt/app/haproxy/sbin/haproxy -f /mnt/app/haproxy/conf/haproxy.cfg

//通过haproxy查看RabbitMQ状态
在浏览器中打开:
http://192.168.18.223:8090
```

