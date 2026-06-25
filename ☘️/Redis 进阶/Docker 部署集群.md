# Docker 部署集群
> 相关笔记：[[集群|Redis 集群]]

这次我们用 docker 部署 11 个 redis 节点，九个用于部署集群，两个用于扩容练习，集群的结构类似于如图，由于 redis 部署集群时内部的随机性，可能节点的静态 ip 所属的角色都不太一样，无伤大雅～
![[Pasted image 20260601105749.png|533]]
## Step 1：创建目录和配置
创建 redis-cluster 目录，在目录内部创建 `docker-compose.yml` 和 `docker-compose.yml` 文件，对应命令为
```bash
mkdir redis-cluster
cd redis-cluster
vim docker-compose.yml
vim docker-compose.yml
```
文件结构为
```txt
redis-cluster/
├── docker-compose.yml
└── generate.sh
```
## Step 2：编写脚本
脚本 `generate.sh` 里内容为
```shell
for port in $(seq 1 9); \
do \
mkdir -p redis${port}/
touch redis${port}/redis.conf
cat << EOF > redis${port}/redis.conf
port 6379
bind 0.0.0.0
protected-mode no
appendonly yes
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-announce-ip 172.30.0.10${port}
cluster-announce-port 6379
cluster-announce-bus-port 16379
EOF
done

# 注意 cluster-announce-ip 的值有变化.
for port in $(seq 10 11); \
do \
mkdir -p redis${port}/
touch redis${port}/redis.conf
cat << EOF > redis${port}/redis.conf
port 6379
bind 0.0.0.0
protected-mode no
appendonly yes
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-announce-ip 172.30.0.1${port}
cluster-announce-port 6379
cluster-announce-bus-port 16379
EOF
done
```
保存后就能执行脚本啦，这个脚本的意思是在当前目录 `redis-cluster` 中建立从 redis 1～redis 11 的文件夹，每个中都有对应节点的必要配置文件。
**⚠️注意：如果你服务器上有其他的网段占用，则需要修改当前脚本与后续的 `yml` 文件中的网段配置，只要不冲突即可。**
```bash
sh generate.sh
```
执行完后如果没有任何反应那就是执行成功了，我们可以看看
![[Pasted image 20260601110846.png|584]]
## Step 3：配置 yml
ok，我们接着来配置 `docker-compose.yml` 文件，内容有点长
```bash
version: '3.7'

networks:
  mynet:
    ipam:
      config:
        - subnet: 172.30.0.0/24

services:

  redis1:
    image: 'redis:5.0.9'
    container_name: redis1
version: '3.7'

networks:
  mynet:
    ipam:
      config:
        - subnet: 172.30.0.0/24

services:

  redis1:
    image: 'redis:5.0.9'
    container_name: redis1
    restart: always
    volumes:
      - ./redis1/:/etc/redis/
    ports:
      - 6371:6379
      - 16371:16379
    command:
      redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.101

  redis2:
    image: 'redis:5.0.9'
    container_name: redis2
    restart: always
    volumes:
      - ./redis2/:/etc/redis/
    ports:
      - 6372:6379
      - 16372:16379
    command:
      redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.102

  redis3:
    image: 'redis:5.0.9'
    container_name: redis3
    restart: always
    volumes:
      - ./redis3/:/etc/redis/
    ports:
      - 6373:6379
      - 16373:16379
    command:
      redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.103

  redis4:
    image: 'redis:5.0.9'
    container_name: redis4
    restart: always
    volumes:
      - ./redis4/:/etc/redis/
    ports:
      - 6374:6379
      - 16374:16379
    command:
      redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.104

  redis5:
    image: 'redis:5.0.9'
    container_name: redis5
    restart: always
    volumes:
      - ./redis5/:/etc/redis/
    ports:
      - 6375:6379
      - 16375:16379
    command:
      redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.105

  redis6:
    image: 'redis:5.0.9'
    container_name: redis6
    restart: always
    volumes:
      - ./redis6/:/etc/redis/
    ports:
      - 6376:6379
      - 16376:16379
    command:
      redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.106

  redis7:
    image: 'redis:5.0.9'
    container_name: redis7
    restart: always
    volumes:
      - ./redis7/:/etc/redis/
    ports:
      - 6377:6379
      - 16377:16379
    command:
      redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.107

  redis8:
    image: 'redis:5.0.9'
    container_name: redis8
    restart: always
    volumes:
      - ./redis8/:/etc/redis/
    ports:
      - 6378:6379
      - 16378:16379
    command:
      redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.108

  redis9:
    image: 'redis:5.0.9'
    container_name: redis9
    restart: always
    volumes:
      - ./redis9/:/etc/redis/
    ports:
      - 6379:6379
      - 16379:16379
    command:
      redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.109

  redis10:
    image: 'redis:5.0.9'
    container_name: redis10
    restart: always
    volumes:
      - ./redis10/:/etc/redis/
    ports:
      - 6380:6379
      - 16380:16379
    command:
      redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.110

  redis11:
    image: 'redis:5.0.9'
    container_name: redis11
    restart: always
    volumes:
      - ./redis11/:/etc/redis/
    ports:
      - 6381:6379
      - 16381:16379
    command:
      redis-server /etc/redis/redis.conf
    networks:
      mynet:
        ipv4_address: 172.30.0.111
```
如果没问题，就执行 `sudo docker-compose up -d`，启动容器并后台运行。
## Step 4：配置集群
还有一个步骤，就是让各自的节点都能“认识对方”，目前只是配置了各个节点的配置，还没有配置他们之间的联系，需要用到一行命令，手动的将各个主机都连系起来。
这个命令直接在 cluster 路径下直接执行即可，无需进入 redis 服务。
```bash
redis-cli --cluster create 172.30.0.101:6379 172.30.0.102:6379 172.30.0.103:6379 172.30.0.104:6379 172.30.0.105:6379 172.30.0.106:6379 172.30.0.107:6379 172.30.0.108:6379 172.30.0.109:6379 --cluster-replicas 2
```
执行后 redis 会默认帮你弄好集群的分片配置，这里都描述了每个分片的 slot 数量，主从节点的分配等...确认无误的话就能输入 yes 下一步真正配置咯
![[Pasted image 20260601104244.png|847]]
到这里 redis 的分片集群就真正配置好了～
