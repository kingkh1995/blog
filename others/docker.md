# [首页](/blog/)

***

## 命令

- docker ps: 查看运行中的docker容器
  - -a: 历史运行过的容器也输出
- docker exec -it (id) /bin/bash: 开启新终端进入容器，退出不影响运行中的容器
- docker attach (id): 进入正在运行的终端，退出时原容器关闭。
- docker network: 网络相关操作
  - create (name) (driver): 创建网络，默认使用Bridge模式
  - inspect (driver): 查看指定模式的所有网络及网络内的容器
- docker inspect (id): 查看容器信息
- docker volume prune: 清除所有无用卷
- docker top (id) (ps options): 查看容器中运行的进程信息，支持ps命令参数
- docker logs (id): 查看容器日志
  - -f：跟踪日志输出
  - --tail (n): 只输出最新几条
- docker stats: 显示容器资源的使用情况

***

## 应用部署

- 使用Dockerfile构建应用
```
docker build -t ddd-user-web:2022.0.0 .
```
```
docker run -it --name app1 --network mynet --network-alias app1 -p  8081:8080 -d ddd-user-web:2022.0.0
docker run -it --name app2 --network mynet --network-alias app2 -p  8082:8080 -d ddd-user-web:2022.0.0
docker run -it --name app3 --network mynet --network-alias app3 -p  8083:8080 -d ddd-user-web:2022.0.0
```

***

## mysql

```
docker run -it --name mysql --network mynet --network-alias mysql -p 3306:3306 --restart always  -e TZ=Asia/Shanghai -e MYSQL_ROOT_PASSWORD=123456 -d mysql
```

***

## canal

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

## redis

- 单机部署
```
docker run -it --name redis --network mynet --network-alias redis -p 6379:6379 --restart always  -d redis redis-server --appendfsync everysec
```

- 集群部署（IP为宿主机物理机地址）
```
docker run -it -p 6371:6379 -p 16371:16379 --name redis1 --network mynet --network-alias redis1 -d redis redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-announce-ip x.x.x.x --cluster-announce-port 6371 --cluster-announce-bus-port 16371 --appendfsync everysec
docker run -it -p 6372:6379 -p 16372:16379 --name redis2 --network mynet --network-alias redis2 -d redis redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-announce-ip x.x.x.x --cluster-announce-port 6372 --cluster-announce-bus-port 16372 --appendfsync everysec
docker run -it -p 6373:6379 -p 16373:16379 --name redis3 --network mynet --network-alias redis3 -d redis redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-announce-ip x.x.x.x
...
```
```
redis-cli --cluster create x.x.x.x:6371 x.x.x.x:6372 x.x.x.x:6373 x.x.x.x:6374 x.x.x.x:6375 x.x.x.x:6376 --cluster-replicas 1
```

***

## zookeeper

- 单机部署
```
docker run -it --name zookeeper  --network mynet --network-alias zookeeper -p 2188:2181 --restart always  -d zookeeper
```

- 集群部署
```
docker-compose -f zk-cluster-compose.yml up -d
```
```yaml
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
  ...
```

***

## elasticsearch

```
docker run -it --name elasticsearch --network mynet --network-alias elasticsearch -p 9200:9200 -e discovery.type=single-node -e "ES_JAVA_OPTS=-Xms2g -Xmx2g" --restart always  -d elasticsearch:7.16.2
```

- 安装中文分词器
```
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.16.2/elasticsearch-analysis-ik-7.16.2.zip
```

***

## kafka

```
docker run -d --name kafka --network mynet --network-alias kafka -p 9092:9092 -e ALLOW_PLAINTEXT_LISTENER=yes -e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181 -e KAFKA_BROKER_ID=0 -e KAFKA_CFG_LISTENERS=PLAINTEXT://:9092 -e  KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://127.0.0.1:9092 bitnami/kafka
```

***

## skywalking

- docker-compose部署（h2数据库）（**links不推荐使用**）
```yaml
version: '3.8'
services:
  skywalking-oap:
    image: apache/skywalking-oap-server
    container_name: skywalking-oap
    restart: always
    ports:
      - 11800:11800
      - 12800:12800
    environment:
      TZ: Asia/Shanghai
  skywalking-ui:
    image: apache/skywalking-ui
    container_name: skywalking-ui
    depends_on:
      - skywalking-oap
    links:
      - skywalking-oap
    restart: always
    ports:
      - 8888:8080
    environment:
      SW_OAP_ADDRESS: skywalking-oap:12800
      TZ: Asia/Shanghai
```
```
docker-compose -f skywalking-compose.yml up -d
```

- 应用接入

  - 下载[Java Agent](https://skywalking.apache.org/downloads/#SkyWalkingJavaAgent)

  - 配置启动项
  ```
  -javaagent:${path}/skywalking-agent.jar -Dskywalking.agent.service_name=${app-name} -Dskywalking.collector.backend_service=localhost:11800
  ```

***

## Nacos Docker

- [docker-compose部署](https://github.com/nacos-group/nacos-docker/blob/master/example)
    
  ```
  docker-compose -f standalone-derby.yaml up
  ```
  
- 控制台地址：http://127.0.0.1:8848/nacos/

***