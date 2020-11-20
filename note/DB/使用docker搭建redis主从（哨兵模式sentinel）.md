

# 使用docker搭建redis哨兵模式（sentinel）

以伪部署方式，使用 Docker 在一台机器上部署。

部署使用1个Redis主节点，2个从节点，从主节点复制数据备份，如果不使用 Sentinel 的话，当主节点宕机后，从节点不能自动顶替，只能手动修改应用的配置连接从节点，不能保存高可用，只能保证数据不丢失。

引入 Sentinel 管理 Redis 主从机，当主节点挂掉后，从从节点中选取一个节点来当主节点，保证 Redis 高可用。

## Sentinel介绍

Sentinel使redis高可用，当主节点宕机后，Sentinel会选出一个从节点为主节点，可以继续为业务提供服务。保证了Redis服务的高可用。

使用 1 个主节点， 2 个从节点，3 个Sentinel集群节点。当有两个Sentinel节点投一个从节点时，该从节点为就为主节点。

```
       +----+
       | M1 |
       | S1 |
       +----+
          |
+----+    |    +----+
| R2 |----+----| R3 |
| S2 |         | S3 |
+----+         +----+

Configuration: quorum = 2
```



## 部署规则



### 规则图

![](https://raw.githubusercontent.com/LiQiongchao/image/master/img/blog/20201110111522.png)

### IP 与 端口规则

| IP          | 应用       | 映射端口 |
| ----------- | ---------- | -------- |
| 172.17.1.77 | master     | 17701    |
| 172.17.1.77 | sentinel01 | 17801    |
| 172.17.1.78 | slave01    | 17702    |
| 172.17.1.78 | sentinel02 | 17802    |
| 172.17.1.79 | slave02    | 17703    |
| 172.17.1.79 | sentinel03 | 17803    |

> 注：由于只有一台机器，就全部部署在一台机器上了。



## Docker 部署

Docker 部署 Sentinel 时，如果不配置下网络，获取的是 Docker 内的IP地址与端口。有两种网络配置方式，一种是使用 Docker host 网络模式（与当前宿主机在一个网络中，端口，网络共享），另一种是在redis和sentinel配置中声明自己的IP与端口（详情参考[官网](https://redis.io/topics/sentinel#sentinel-docker-nat-and-possible-issues)）。



### 部署 Redis 1主2从

目录结构

```shell
(py3) [root /data/server-apps/redis-sentinel]#ll
total 24
drwxr-xr-x 3 root root 4096 Nov 10 16:15 master
drwxr-xr-x 4 root root 4096 Nov 11 15:22 sentinel-01
drwxr-xr-x 4 root root 4096 Nov 11 15:23 sentinel-02
drwxr-xr-x 4 root root 4096 Nov 11 15:22 sentinel-03
drwxr-xr-x 3 root root 4096 Nov 10 16:15 slave-01
drwxr-xr-x 3 root root 4096 Nov 10 16:15 slave-02
```



启动Redis master节点

```shell
docker run -d --network host --name redis-master -v /data/server-apps/redis-sentinel/master/data:/data redis:3.2.6-alpine --appendonly yes --port 17701
```



启动Redis slaves节点

```shell
# slave-01
docker run -d --network host --name redis-slave-01 \
-v /data/server-apps/redis-sentinel/slave-01/data:/data \
redis:3.2.6-alpine --appendonly yes \
--port 17702 --slave-read-only yes \
--slaveof 172.17.1.77 17701

# slave-02
docker run -d --network host --name redis-slave-02 \
-v /data/server-apps/redis-sentinel/slave-02/data:/data \
redis:3.2.6-alpine --appendonly yes \
--port 17703 --slave-read-only yes \
--slaveof 172.17.1.77 17701
```



### 部署 Sentinel 集群

#### sentinel 目录结构

```shell
(py3) [root /data/server-apps/redis-sentinel/sentinel-01]#ll
total 12
drwxr-xr-x 2 root root 4096 Nov 11 15:29 conf
drwxr-xr-x 2 root root 4096 Nov 10 16:18 data
-rw-r--r-- 1 root root  253 Nov 11 15:22 docker-sentinel.sh
```

#### Senttinel conf

Sentinel01 的 `conf/sentinel.conf`（要给sentinel.conf文件加**写权限**，因为检测到其它sentinel和redis时会添加到配置文件中去）

```yaml
#端口号
port 17801
# 工作目录
dir "/var/redis/data"
# master001：自定义集群名，所有集群名要一致，2:投票数量必须2个sentinel才能判断主节点是否失败
sentinel monitor master01 172.17.1.77 17701 2
sentinel down-after-milliseconds master01 30000
# 表示在故障转移的时候最多有numslaves在同步更新新的master
sentinel parallel-syncs master01 1
# 故障转移超时时间为180秒
sentinel failover-timeout master01 180000
```

说明：

>  sentinel monitor: 是指当前监控主节点，2: 代表判断主节点失败至少需要2个Sentinel节点节点同意，master003: 是主节点的别名，如果超过30000毫秒且没有回复，则判定不可达

Sentinel02 的  `conf/sentinel.conf`（要给sentinel.conf文件加**写权限**，因为检测到其它sentinel和redis时会添加到配置文件中去）

```yaml
#端口号
port 17802
# 监控日志目录
dir "/var/redis/data"
# master001：自定义集群名，所有集群名要一致，2:投票数量必须2个sentinel才能判断主节点是否失败
sentinel monitor master01 172.17.1.77 17701 2
sentinel down-after-milliseconds master01 30000
# 表示在故障转移的时候最多有numslaves在同步更新新的master
sentinel parallel-syncs master01 1
# 故障转移超时时间为180秒
sentinel failover-timeout master01 180000
```

Sentinel03 的  `conf/sentinel.conf`（要给sentinel.conf文件加**写权限**，因为检测到其它sentinel和redis时会添加到配置文件中去）

```yaml
#端口号
port 17803
# 监控日志目录
dir "/var/redis/data"
# master001：自定义集群名，所有集群名要一致，2:投票数量必须2个sentinel才能判断主节点是否失败
sentinel monitor master01 172.17.1.77 17701 2
sentinel down-after-milliseconds master01 30000
# 表示在故障转移的时候最多有numslaves在同步更新新的master
sentinel parallel-syncs master01 1
# 故障转移超时时间为180秒
sentinel failover-timeout master01 180000
```



#### sentinel 启动脚本

sentinel-01/docker-sentinel.sh

```shell
#!/bin/bash

docker run -d --network host --name redis-sentinel-01 \
-v /data/server-apps/redis-sentinel/sentinel-01/data:/var/redis/data \
-v /data/server-apps/redis-sentinel/sentinel-01/conf:/conf \
redis:3.2.6-alpine \
/conf/sentinel.conf --sentinel
```

sentinel-02/docker-sentinel.sh

```
#!/bin/bash

docker run -d --network host --name redis-sentinel-02 \
-v /data/server-apps/redis-sentinel/sentinel-02/data:/var/redis/data \
-v /data/server-apps/redis-sentinel/sentinel-02/conf:/conf \
redis:3.2.6-alpine \
/conf/sentinel.conf --sentinel
```

sentinel-03/docker-sentinel.sh

```
#!/bin/bash

docker run -d --network host --name redis-sentinel-03 \
-v /data/server-apps/redis-sentinel/sentinel-03/data:/var/redis/data \
-v /data/server-apps/redis-sentinel/sentinel-03/conf:/conf \
redis:3.2.6-alpine \
/conf/sentinel.conf --sentinel
```

## 验证

### 验证master

Sentinel完全启动后的 sentinel.conf文件

```yaml
#端口号
port 17801
# 工作目录
dir "/var/redis/data"
# master001：自定义集群名，2:投票数量必须2个sentinel才能判断主节点是否失败
sentinel myid 49cf13cc7f9b3eb24a0b45806618caba4c8b04ad
sentinel monitor master01 172.17.1.77 17701 2
# 表示在故障转移的时候最多有numslaves在同步更新新的master
sentinel config-epoch master01 0
# 故障转移超时时间为180秒
sentinel leader-epoch master01 0
# Generated by CONFIG REWRITE
sentinel known-slave master01 172.17.1.77 17703
sentinel known-slave master01 172.17.1.77 17702
sentinel known-sentinel master01 172.17.1.77 17803 0ce9b6245ed245839ac99d3eacccdc3c8c2f360e
sentinel known-sentinel master01 172.17.1.77 17802 19a4ec17c47349e35e85f1a6b98186a88e526ac6
sentinel current-epoch 0
```

#### 查看Sentinel上记录的master

登录sentinel

```shell
docker run --rm -it --network container:redis-sentinel-01 redis:3.2.6-alpine redis-cli -p 17801
```

查看master

```shell
127.0.0.1:17801> sentinel masters
1)  1) "name"
    2) "master01"
    3) "ip"
    4) "172.17.1.77"
    5) "port"
    6) "17701"
    7) "runid"
    8) "82e554e8a1a8e85122aa7e9628be4a427ce47f81"
    9) "flags"
   10) "master"
   11) "link-pending-commands"
   12) "0"
   13) "link-refcount"
   14) "1"
   15) "last-ping-sent"
   16) "0"
   17) "last-ok-ping-reply"
   18) "552"
   19) "last-ping-reply"
   20) "552"
   21) "down-after-milliseconds"
   22) "30000"
   23) "info-refresh"
   24) "3846"
   25) "role-reported"
   26) "master"
   27) "role-reported-time"
   28) "1499438"
   29) "config-epoch"
   30) "0"
   31) "num-slaves"
   32) "2"
   33) "num-other-sentinels"
   34) "2"
   35) "quorum"
   36) "2"
   37) "failover-timeout"
   38) "180000"
   39) "parallel-syncs"
   40) "1"
```

sentinel 常用命令

```shell
# 查看某个监控的某个集群的master
sentinel master master01
sentinel masters
# 查看某个监控的某个集群的salves，需要版本大于5.0
sentinel replicas master01
# 查看Setntinel的其它节点
sentinel sentinels master01
```



### 验证高可用

模拟宕机，关闭 redis-master 的容器。

```shell
# 停止容器
docker stop redis-master
# 连接 redis-master
docker run --rm -it --network container:redis-master redis:3.2.6-alpine redis-cli -p 17701
```

过了30s后，会判断 redis-master宕机，再次执行 `sentinel masters` 查看选出redis-slave-02为新的master

```shell
127.0.0.1:17801> sentinel masters
1)  1) "name"
    2) "master01"
    3) "ip"
    4) "172.17.1.77"
    5) "port"
    6) "17702"
    7) "runid"
    8) "3bf90df72f3d8054de29d98a5bd45c297f77bfe8"
    9) "flags"
   10) "master"
   11) "link-pending-commands"
   12) "0"
   13) "link-refcount"
   14) "1"
   15) "last-ping-sent"
   16) "0"
   17) "last-ok-ping-reply"
   18) "661"
   19) "last-ping-reply"
   20) "661"
   21) "down-after-milliseconds"
   22) "30000"
   23) "info-refresh"
   24) "661"
   25) "role-reported"
   26) "master"
   27) "role-reported-time"
   28) "1111"
   29) "config-epoch"
   30) "1"
   31) "num-slaves"
   32) "2"
   33) "num-other-sentinels"
   34) "2"
   35) "quorum"
   36) "2"
   37) "failover-timeout"
   38) "180000"
   39) "parallel-syncs"
   40) "1"
```

宕机之后在redis-slave-01上查看主从状态（只有一个从机）

```shell
127.0.0.1:17702> info Replication
# Replication
role:master
connected_slaves:1
slave0:ip=172.17.1.77,port=17703,state=online,offset=66465,lag=1
master_repl_offset:66617
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:66616
```



在redis-slave-01上redis-master的容器再启动后就成了slave

```shell
127.0.0.1:17702> info Replication
# Replication
role:master
connected_slaves:2
slave0:ip=172.17.1.77,port=17703,state=online,offset=71084,lag=0
slave1:ip=172.17.1.77,port=17701,state=online,offset=71084,lag=1
master_repl_offset:71222
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:71221
```



## 几个问题

1. slave1、slave2是从头开始复制还是从切入点开始复制?

完全复制

2. 从机是否可以写？set可否？

从机不能写，不以使用set指令。

3. 主机shutdown后情况如何？

从机是上位还是原地待命  原地待命。

4. 主机又回来了后，主机新增记录，从机还能否顺利复制？  

能顺利复制。

5. 其中一台从机down后情况如何？依照原有它能跟上大部队吗？

不能，要重新启动，重新slave才可以。

6. 复制的缺点？

由于所有的写操作都是先在Master上操作，然后同步更新到Slave上，所以从Master同步到Slave机器有一定的延迟，当系统很繁忙的时候，延迟问题会更加严重，Slave机器数量的增加也会使这个问题更加严重。



## 参考

> [Docker Redis Sentinel 高可用部署实践](https://www.jianshu.com/p/f3c6139284a6)
>
> [Redis Sentinel Documentation](https://redis.io/topics/sentinel)
>
> [https://docs.docker.com/engine/reference/run/](https://docs.docker.com/engine/reference/run/)



