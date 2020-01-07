#### JAVA环境
```
[root@localhost app]# tar xzf jdk-8u111-linux-x64.tar.gz
[root@localhost app]# mv jdk1.8.0_111 /mnt/app/java
[root@localhost app]# echo 'JAVA_HOME=/mnt/app/java' | tee /etc/profile.d/java.sh
[root@localhost app]# echo 'JRE_HOME=${JAVA_HOME}/jre' | tee -a /etc/profile.d/java.sh
[root@localhost app]# echo 'CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib' | tee -a /etc/profile.d/java.sh
[root@localhost app]# echo 'export PATH=${JAVA_HOME}/bin:$PATH' | tee -a /etc/profile.d/java.sh
[root@localhost app]# source  /etc/profile
[root@localhost app]# java -version
```

#### Erlang 依赖
```
sudo yum install -y gcc gcc-c++ glibc-devel make ncurses-devel openssl-devel autoconf java-1.8.0-openjdk-devel git 
```

#### Erlang安装
```
[root@localhost app]# tar xzf otp_src_19.2.tar.gz
[root@localhost app]# cd otp_src_19.2
[root@localhost otp_src_19.2]# ./configure --prefix=/mnt/app/erlang
[root@localhost otp_src_19.2]# make
[root@localhost otp_src_19.2]# make install

[root@localhost otp_src_19.2]# echo 'export ERLANG_HOME=/mnt/app/erlang' | tee /etc/profile.d/erlang.sh
[root@localhost otp_src_19.2]# echo 'export ERLANG_BIN=${ERLANG_HOME}/bin' | tee -a /etc/profile.d/erlang.sh
[root@localhost otp_src_19.2]# echo 'export PATH=${ERLANG_BIN}:$PATH' | tee -a /etc/profile.d/erlang.sh  
[root@localhost otp_src_19.2]# source /etc/profile
```

#### RabbitMQ install 
1. 主机名 
```
[root@localhost app]# echo rabbitmq188 | tee /etc/hostname
[root@localhost app]# echo '192.168.13.188 rabbitmq188' |tee -a /etc/hosts
[root@localhost app]# hostname rabbitmq188
[root@localhost app]# $SHELL
```
2. RabbitMQ安装 
```
[root@localhost app]# xz -d rabbitmq-server-generic-unix-3.6.10.tar.xz
[root@localhost app]# tar xf rabbitmq-server-generic-unix-3.6.10.tar
[root@localhost app]# mv rabbitmq_server-3.6.10 /mnt/app/rabbitmq
[root@localhost app]# chown -R wisdom.wisdom /mnt/app/rabbitmq
```
3. RabbitMQ 环境变量
```
[root@localhost app]# echo 'export RABBITMQ_HOME=/mnt/app/rabbitmq' |tee /etc/profile.d/rabbitmq.sh
[root@localhost app]# echo 'export RABBITMQ_BIN=$RABBITMQ_HOME/sbin' |tee -a /etc/profile.d/rabbitmq.sh                     
[root@localhost app]# echo 'export PATH=$RABBITMQ_BIN:$PATH' |tee -a /etc/profile.d/rabbitmq.sh                                
[root@localhost app]# source /etc/profile
```
4. RabbitMQ 配置文件
```
[wisdom@localhost ~]$ cat > /mnt/app/rabbitmq/etc/rabbitmq/rabbitmq-env.conf <<EOF
RABBITMQ_MNESIA_BASE=/mnt/data/rabbitmq/mnesia
RABBITMQ_LOG_BASE=/mnt/log/rabbitmq
EOF

[wisdom@localhost ~]$ cat > /mnt/app/rabbitmq/etc/rabbitmq/rabbitmq.config <<EOF
[
  {rabbit,
    [
      {vm_memory_high_watermark, 0.6},
      {vm_memory_high_watermark_paging_ratio, 0.3},
      {disk_free_limit, "10GB"},
      {hipe_compile, true},
      {queue_index_embed_msgs_below, 4096}
    ]
  }
].
EOF

[root@localhost app]# mkdir -p /mnt/data/rabbitmq/mnesia
[root@localhost app]# mkdir -p /mnt/log/rabbitmq
[root@localhost app]# chown -R wisdom.wisdom /mnt/data/rabbitmq
[root@localhost app]# chown -R wisdom.wisdom /mnt/log/rabbitmq
``` 
5. RabbitMQ 启动
```
[wisdom@localhost ~]$ /mnt/app/rabbitmq/sbin/rabbitmq-server -detached
or:
[wisdom@localhost ~]$ /mnt/app/rabbitmq/sbin/rabbitmq-server &

[wisdom@localhost ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl stop
```
6. RabbitMQ 安装插件rabbitmq_management(web控制台) 
```
[root@localhost app]# su - wisdom
[wisdom@localhost ~]$ /mnt/app/rabbitmq/sbin/rabbitmq-plugins list
[wisdom@localhost ~]$ /mnt/app/rabbitmq/sbin/rabbitmq-plugins enable rabbitmq_management
```
7. RabbitMQ 创建vhost
```
[wisdom@localhost ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl list_vhosts
[wisdom@localhost ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl add_vhost /zabbix

```
8. 高可用RabbitMQ
```
[wisdom@localhost ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl set_policy -p /zabbix ha-all "^" '{"ha-mode":"all"}'
[wisdom@localhost ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl list_policies -p /zabbix
```
9. RabbitMQ用户创建
```
[wisdom@localhost ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl list_users
[wisdom@localhost ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl delete_user guest

[wisdom@localhost ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl add_user zabbix zabbix123

[wisdom@localhost ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl set_user_tags zabbix administrator monitoring

[wisdom@localhost ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl list_permissions
[wisdom@localhost ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl list_user_permissions zabbix
[wisdom@localhost ~]$ /mnt/app/rabbitmq/sbin/rabbitmqctl set_permissions -p /zabbix zabbix '.*' '.*' '.*'

```




