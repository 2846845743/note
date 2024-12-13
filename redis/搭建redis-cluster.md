# 使用docker搭建redis-cluster集群

## 什么是redis-cluster？
redis-cluster是redis的分布式集群，它提供了自动分片的功能，可以将数据分片存储在多个节点上，从而提高了redis的性能和可用性。
举个例子：假设有6个redis节点，那么redis-cluster会将数据分片存储在这6个节点上，当有一个节点宕机时，redis-cluster会自动将宕机节点的数据迁移到其他节点上，从而保证数据的可用性。
如果我们链接到redis-cluster集群，那么我们可以通过任意一个节点来存储集群中的数据，redis-cluster会自动将请求转发到正确的节点上。


## 如何搭建redis-cluster？

### 1. 拉取镜像
```shell
docker pull redis
```

### 2. 创建docker网络
```shell
docker network create myredis 
```

### 3. 使用shell脚本编写配置文件
```shell
for port in $(seq 6379 6384); 
do 
mkdir -p /home/redis/node-${port}/conf
touch /home/redis/node-${port}/conf/redis.conf
cat  << EOF > /home/redis/node-${port}/conf/redis.conf
port ${port}
requirepass 1234
bind 0.0.0.0
protected-mode no
daemonize no
appendonly yes
cluster-enabled yes 
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-announce-ip  服务器ip地址 #通常是宿主机的ip地址，也就是外网ip地址
cluster-announce-port ${port}
cluster-announce-bus-port 1${port}
EOF
done
```

tree命令可以查看目录结构
```shell
tree /home/redis
```
![img_1.png](img_1.png)

### 4. 循环创建redis容器
```shell
for port in $(seq 6379 6384); \
do \
   docker run -it -d -p ${port}:${port} -p 1${port}:1${port} \
  --privileged=true -v /home/redis/node-${port}/conf/redis.conf:/usr/local/etc/redis/redis.conf \
  --privileged=true -v /home/redis/node-${port}/data:/data \
  --restart always --name redis-${port} --net myredis \
  --sysctl net.core.somaxconn=1024 redis redis-server /usr/local/etc/redis/redis.conf
done
```
![img.png](img.png)
此时可以看到容器创建成功


### 5. 创建集群
```shell
docker exec -it redis-6379 /bin/bash
redis-cli  -a 1234 --cluster create 117.72.91.159:6379 117.72.91.159:6380 117.72.91.159:6381 117.72.91.159:6382 117.72.91.159:6383 117.72.91.159:6384   --cluster-replicas 1
```

创建成功！
![img_2.png](img_2.png)

### 6. 查看集群信息
```shell
docker exec -it redis-6379 /bin/bash
redis-cli -p 6380 -a 1234 cluster slots
```

![img_3.png](img_3.png)

### 7. 测试集群
可以看到，往其中一个里面添加，会自动路由到正确的节点上
![img_4.png](img_4.png)

**！注意：一定要开放6379-6384，并且16379-16384也要，本次实验的节点是通过外网进行连接的。（后续也可以改成内网的，但是不熟悉docker网络）**

## 总结
关键命令解释：
>redis-cli  -a 1234 --cluster create 117.72.91.159:6379 117.72.91.159:6380 117.72.91.159:6381 117.72.91.159:6382 117.72.91.159:6383 117.72.91.159:6384   --cluster-replicas 1

这个命令代表使用redis客户端，连接自己的redis集群，然后创建集群，集群的节点是6379-6384，--cluster-replicas 1代表每个主节点有一个从节点。前三个代表主节点







