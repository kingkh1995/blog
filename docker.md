## [首页](https://kingkh1995.github.io/blog/)

***

### 部分命令

```
创建网络 docker network create mynet
查看容器信息 docker inspect [id]
清空无用卷 docker volume prune
```

***

### 应用部署
- 使用Dockerfile构建应用
```
docker build -t op-user-web:Ace.RC .
```

```
docker run -it --name app1 --network mynet --network-alias app1 -p  8081:8080 -d op-user-web:Ace.RC
docker run -it --name app2 --network mynet --network-alias app2 -p  8082:8080 -d op-user-web:Ace.RC
docker run -it --name app3 --network mynet --network-alias app3 -p  8083:8080 -d op-user-web:Ace.RC
```

***

### mysql

```
docker run -it --name mysql --network mynet --network-alias mysql -p 3306:3306 --restart always  -e TZ=Asia/Shanghai -e MYSQL_ROOT_PASSWORD=123456 -d mysql
```

***

### canal
```
docker run -it --name canal --network mynet --network-alias canal -p 11111:11111 -d canal/canal-server
```

- mysql添加canal用户
``` sql
CREATE USER canal IDENTIFIED WITH mysql_native_password BY 'canal';  
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
-- GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ;
FLUSH PRIVILEGES;
```

- 修改配置
```
vi canal-server/conf/example/instance.properties 
canal.master.position=
```

***

### redis

- [AnotherRedisDesktopManager](https://github.com/qishibo/AnotherRedisDesktopManager/releases)

- 单机部署
```
docker run -it --name redis --network mynet --network-alias redis -p 6379:6379 --restart always  -d redis redis-server --appendfsync everysec
```

- 集群部署
```
IP为宿主机物理机地址
docker run -it -p 6371:6379 -p 16371:16379 --name redis1 --network mynet --network-alias redis1 -d redis redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-announce-ip x.x.x.x --cluster-announce-port 6371 --cluster-announce-bus-port 16371 --appendfsync everysec
docker run -it -p 6372:6379 -p 16372:16379 --name redis2 --network mynet --network-alias redis2 -d redis redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-announce-ip x.x.x.x --cluster-announce-port 6372 --cluster-announce-bus-port 16372 --appendfsync everysec
docker run -it -p 6373:6379 -p 16373:16379 --name redis3 --network mynet --network-alias redis3 -d redis redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-announce-ip x.x.x.x --cluster-announce-port 6373 --cluster-announce-bus-port 16373 --appendfsync everysec
docker run -it -p 6374:6379 -p 16374:16379 --name redis4 --network mynet --network-alias redis4 -d redis redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-announce-ip x.x.x.x --cluster-announce-port 6374 --cluster-announce-bus-port 16374 --appendfsync everysec
docker run -it -p 6375:6379 -p 16375:16379 --name redis5 --network mynet --network-alias redis5 -d redis redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-announce-ip x.x.x.x --cluster-announce-port 6375 --cluster-announce-bus-port 16375 --appendfsync everysec
docker run -it -p 6376:6379 -p 16376:16379 --name redis6 --network mynet --network-alias redis6 -d redis redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-announce-ip x.x.x.x --cluster-announce-port 6376 --cluster-announce-bus-port 16376 --appendfsync everysec
```

```
redis-cli --cluster create x.x.x.x:6371 x.x.x.x:6372 x.x.x.x:6373 x.x.x.x:6374 x.x.x.x:6375 x.x.x.x:6376 --cluster-replicas 1
```

***

### zookeeper

- 单机部署
```
docker run -it --name zookeeper  --network mynet --network-alias zookeeper -p 2188:2181 --restart always  -d zookeeper
```

- 集群部署
```
docker-compose -f zk-cluster-compose.yml up -d
```
```yml
version: '3.8'
networks:
  mynet:
    external: true
services:
  zoo1:
    image: zookeeper
    restart: always
    hostname: zoo1
    container_name: zoo1
    ports:
      - "2181:2181"
    networks:
      - "mynet"
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181 server.4=zoo4:2888:3888;2181 server.5=zoo5:2888:3888;2181
  zoo2:
    image: zookeeper
    restart: always
    hostname: zoo2
    container_name: zoo2
    ports:
      - "2182:2181"
    networks:
      - "mynet"
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181 server.4=zoo4:2888:3888;2181 server.5=zoo5:2888:3888;2181
  zoo3:
    image: zookeeper
    restart: always
    hostname: zoo3
    container_name: zoo3
    ports:
      - "2183:2181"
    networks:
      - "mynet"
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181 server.4=zoo4:2888:3888;2181 server.5=zoo5:2888:3888;2181
  zoo4:
    image: zookeeper
    restart: always
    hostname: zoo4
    container_name: zoo4
    ports:
      - "2184:2181"
    networks:
      - "mynet"
    environment:
      ZOO_MY_ID: 4
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181 server.4=zoo4:2888:3888;2181 server.5=zoo5:2888:3888;2181
  zoo5:
    image: zookeeper
    restart: always
    hostname: zoo5
    container_name: zoo5
    ports:
      - "2185:2181"
    networks:
      - "mynet"
    environment:
      ZOO_MY_ID: 5
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181 server.4=zoo4:2888:3888;2181 server.5=zoo5:2888:3888;2181
```

***

### elasticsearch

```
docker run -it --name elasticsearch --network mynet --network-alias elasticsearch -p 9200:9200 --restart always  -d elasticsearch:7.14.2
```

- 安装中文分词器
```
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.14.2/elasticsearch-analysis-ik-7.14.2.zip
```

***

### kafka

```
docker run -d --name kafka --network mynet --network-alias kafka -p 9092:9092 -e ALLOW_PLAINTEXT_LISTENER=yes -e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181 -e KAFKA_BROKER_ID=0 -e KAFKA_CFG_LISTENERS=PLAINTEXT://:9092 -e  KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://127.0.0.1:9092 bitnami/kafka
```

***