# docker-compose部署Redis-Cluster集群

## 创建目录和文件

创建目录和文件、结构如下：

```text
├── docker-compose.yml
├── redis-6371
│   ├── conf
│   │   └── redis.conf
│   └── data
├── redis-6372
│   ├── conf
│   │   └── redis.conf
│   └── data
├── redis-6373
│   ├── conf
│   │   └── redis.conf
│   └── data
├── redis-6374
│   ├── conf
│   │   └── redis.conf
│   └── data
├── redis-6375
│   ├── conf
│   │   └── redis.conf
│   └── data
└── redis-6376
    ├── conf
    │   └── redis.conf
    └── data
```

## redis.conf 配置文件

```text
port 6371
cluster-enabled yes
cluster-config-file nodes-6371.conf
cluster-node-timeout 5000
appendonly yes
protected-mode no
requirepass 1234
masterauth 1234
cluster-announce-ip 10.35.30.39 # 这里是宿主机IP
cluster-announce-port 6371
cluster-announce-bus-port 16371
```

每个节点的配置只需改变端口。

## docker-compose 配置文件

docker-compose.yml 文件内容如下：

```yaml
version: "3"

# 定义服务，可以多个
services:
  redis-6371: # 服务名称
    image: redis # 创建容器时所需的镜像
    container_name: redis-6371 # 容器名称
    restart: always # 容器总是重新启动
    volumes: # 数据卷，目录挂载
      - ./redis-6371/conf/redis.conf:/usr/local/etc/redis/redis.conf
      - ./redis-6371/data:/data
    ports:
      - 6371:6371
      - 16371:16371
    command:
      redis-server /usr/local/etc/redis/redis.conf

  redis-6372:
    image: redis
    container_name: redis-6372
    volumes:
      - ./redis-6372/conf/redis.conf:/usr/local/etc/redis/redis.conf
      - ./redis-6372/data:/data
    ports:
      - 6372:6372
      - 16372:16372
    command:
      redis-server /usr/local/etc/redis/redis.conf

  redis-6373:
    image: redis
    container_name: redis-6373
    volumes:
      - ./redis-6373/conf/redis.conf:/usr/local/etc/redis/redis.conf
      - ./redis-6373/data:/data
    ports:
      - 6373:6373
      - 16373:16373
    command:
      redis-server /usr/local/etc/redis/redis.conf
      
  redis-6374:
    image: redis
    container_name: redis-6374
    restart: always
    volumes:
      - ./redis-6374/conf/redis.conf:/usr/local/etc/redis/redis.conf
      - ./redis-6374/data:/data
    ports:
      - 6374:6374
      - 16374:16374
    command:
      redis-server /usr/local/etc/redis/redis.conf

  redis-6375:
    image: redis
    container_name: redis-6375
    volumes:
      - ./redis-6375/conf/redis.conf:/usr/local/etc/redis/redis.conf
      - ./redis-6375/data:/data
    ports:
      - 6375:6375
      - 16375:16375
    command:
      redis-server /usr/local/etc/redis/redis.conf

  redis-6376:
    image: redis
    container_name: redis-6376
    volumes:
      - ./redis-6376/conf/redis.conf:/usr/local/etc/redis/redis.conf
      - ./redis-6376/data:/data
    ports:
      - 6376:6376
      - 16376:16376
    command:
      redis-server /usr/local/etc/redis/redis.conf
```

编写完成后使用`docker-compose up -d`启动容器。

> 注：这里没有使用主机模式（host），而是使用 NAT 模式，因为主机模式可能导致外部客户端无法连接。

## 进入容器，创建集群

上面只是启动了 6 个 Redis 实例，并没有构建成 Cluster 集群。

执行`docker exec -it redis-6371 bash`进入一个 Redis 节点容器，随便哪个都行。

继续执行以下命令创建集群：

```text
# 集群创建命令
redis-cli -a 1234 --cluster create 10.35.30.39:6371 10.35.30.39:6372 10.35.30.39:6373 10.35.30.39:6374 10.35.30.39:6375 10.35.30.39:6376 --cluster-replicas 1
# 执行过后会有以下输出
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 10.35.30.39:6375 to 10.35.30.39:6371
Adding replica 10.35.30.39:6376 to 10.35.30.39:6372
Adding replica 10.35.30.39:6374 to 10.35.30.39:6373
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: e9a35d6a9d203830556de89f06a3be2e2ab4eee1 10.35.30.39:6371
   slots:[0-5460] (5461 slots) master
M: 0c8755144fe6a200a46716371495b04f8ab9d4c8 10.35.30.39:6372
   slots:[5461-10922] (5462 slots) master
M: fcb83b0097d2a0a87a76c0d782de12147bc86291 10.35.30.39:6373
   slots:[10923-16383] (5461 slots) master
S: b9819797e98fcd49f263cec1f77563537709bcb8 10.35.30.39:6374
   replicates fcb83b0097d2a0a87a76c0d782de12147bc86291
S: f4660f264f12786d81bcf0b18bc7287947ec8a1b 10.35.30.39:6375
   replicates e9a35d6a9d203830556de89f06a3be2e2ab4eee1
S: d2b9f265ef7dbb4a612275def57a9cc24eb2fd5d 10.35.30.39:6376
   replicates 0c8755144fe6a200a46716371495b04f8ab9d4c8
Can I set the above configuration? (type 'yes' to accept): yes # 这里输入 yes 并回车 确认节点 主从身份 以及 哈希槽的分配
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
.
>>> Performing Cluster Check (using node 10.35.30.39:6371)
M: e9a35d6a9d203830556de89f06a3be2e2ab4eee1 10.35.30.39:6371
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 0c8755144fe6a200a46716371495b04f8ab9d4c8 10.35.30.39:6372
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: b9819797e98fcd49f263cec1f77563537709bcb8 10.35.30.39:6374
   slots: (0 slots) slave
   replicates fcb83b0097d2a0a87a76c0d782de12147bc86291
M: fcb83b0097d2a0a87a76c0d782de12147bc86291 10.35.30.39:6373
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: f4660f264f12786d81bcf0b18bc7287947ec8a1b 10.35.30.39:6375
   slots: (0 slots) slave
   replicates e9a35d6a9d203830556de89f06a3be2e2ab4eee1
S: d2b9f265ef7dbb4a612275def57a9cc24eb2fd5d 10.35.30.39:6376
   slots: (0 slots) slave
   replicates 0c8755144fe6a200a46716371495b04f8ab9d4c8
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

看到上面的输出即为 Cluster 集群配置完成。且为 3 主 3 从。

参考：

https://zysite.top/archives/docker-compose-redis-cluster

https://www.cnblogs.com/coderaniu/p/15352323.html#1694350158
