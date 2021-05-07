---
title: 使用canal连接kafka
date: 2021-04-20 14:06:49
tags: kafka
category: other
---
# 使用canal连接kafka
这篇主要是项目还原，目的是记录构建时遇到的各种奇葩坑，避免下次迷路。废话不多说，直接上手。

> 默认已安装`docker`，`docker-compose`，`nodejs`，`yarn`，`typescript`  

1. 首先根据 [kafka-docker](https://github.com/wurstmeister/kafka-docker/blob/master/docker-compose.yml) 这个官方的仓库下的`docker-compose.yml`复制一份到自己的项目中

```yaml
version: '2'
services:
  zookeeper:
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
  kafka:
    build: .
    ports:
      - "9092"
    environment:
      # 更改为自己的ip地址，最好是ip地址，我先使用localhost
      KAFKA_ADVERTISED_HOST_NAME: localhost
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

将`kafka`下的`build`项，更改为`kafka`镜像，可以从[dockerhub](https://hub.docker.com/r/wurstmeister/kafka)中查找指定版本的`kafka`，这里使用`wurstmeister/kafka:2.13-2.7.0`

在`environment`下添加配置属性

```yaml
...
  kafka:
    image: wurstmeister/kafka:2.13-2.7.0
    ports:
      - "9092:9092" #向外暴露端口
    environment:
      KAFKA_BROKER_ID: 1 #新增一个broker id
      KAFKA_ADVERTISED_HOST_NAME: localhost
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_CREATE_TOPICS: "test:2:1" #新增一个topic或多个 
...
```

然后拉取镜像，并运行起来

```bash
$ docker-compose up
```

2. 编写`Producer`和`Customer` 

   `kafkajs`版

   ```js
   //config.ts //简单的配置
   const Config = {
       brokers: [
           "localhost:9092" //kafka的服务器
       ],
       topic: 'test' //与kafka添加的topcs一样
   }
   
   export default Config;
   
   
   //kafka.ts //实例化一个kafkajs对象
   import { Kafka } from "kafkajs";
   import Config from "./config";
   
   const kafka = new Kafka({
       clientId: 'kafkajs',
       brokers: Config.brokers
   });
   
   export default kafka;
   
   //producer.ts //kafka Producer
   import { Message } from "kafkajs";
   import Config from "./config";
   import kafka from "./kafka";
   
   async function producer(messages: Message[]) {
       const producer = kafka.producer();
       await producer.connect();
       await producer.send({
           topic: Config.topic,
           messages
       });
       await producer.disconnect()
   }
   
   export default producer;
   
   //consumer.ts //kafka Consumer
   import kafka from "./kafka";
   import Config from "./config";
   import { ConsumerConfig, EachMessagePayload } from "kafkajs";
   
   async function consumer(config: ConsumerConfig) {
       const consumer = kafka.consumer(config);
       await consumer.connect()
       await consumer.subscribe({
           topic: Config.topic,
           fromBeginning: true
       });
       await consumer.run({
           eachMessage: async ({topic, partition, message}: EachMessagePayload): Promise<void> => {
               console.log({
                   value: message.value.toString(),
                   topic,
                   partition
               })
           }
       })
   }
   
   export default consumer;
   
   //index.ts //函数入口
   import producer from "./producer";
   import consumer from './consumer';
   
   async function start() {
       await producer([
           {value: 'Hello'},
           {value: ','},
           {value: 'I\'m'},
           {value: 'kafkajs'}
       ])
   
       await consumer({
           groupId: 'consumer-1'
       })
       await consumer({
           groupId: 'consumer-2'
       })
   }
   
   start()
   .catch(console.log)
   ```

   然后编译文件，并运行，可以看到我们消息从`Producer`发送到了`Consumer`

3. 接入`canal`

   修改`docker-compose.yml`，添加`canal`的镜像和相关配置，同时添加一个测试的`mysql`镜像（注，由于项目需求，我还配置了`wordpress`镜像）

   [canal配置说明](https://github.com/alibaba/canal/wiki/AdminGuide) 

   ```yaml
   ...
     canal:
       image: canal/canal-server:v1.1.4
       environment:
         - canal.instance.mysql.slaveId=54321 #slave id 不要与mysql的一样就行
         - canal.instance.master.address=mysql:3306 #mysql地址
         - canal.instance.dbUsername=kafka #mysql 对应的用户名
         - canal.instance.dbPassword=kafka #mysql 对应的密码
         - canal.instance.parser.parallel=false #由于我用的虚拟机，cpu为1G，所以设为false
         - canal.instance.filter.regex=kafka\.user #数据库中要监听的表，详细看官方说明
         - canal.mq.dynamicTopic=.*\..* #动态生成topic
         - canal.zkServers=zookeeper:2181 #链接zookeeper集群的链接信息
         #canal.properties 配置
         - canal.serverMode=kafka #MQ使用的kafka
         - canal.mq.servers=kafka:9092 #kafka地址
       depends_on:
         - zookeeper
         - kafka
   
     mysql:
       image: mysql:5.7
       restart: always
       volumes:
         - ./configuration/conf.d/binlog.cnf:/etc/mysql/conf.d/binlog.cnf #为了让mysql开启bin_log模式的配置
       restart: always
       environment:
         MYSQL_ROOT_PASSWORD: root_password_can_you
         MYSQL_DATABASE: kafkadb
         MYSQL_USER: kafka
         MYSQL_PASSWORD: kafka
       ports:
         - 3306:3306
   
   ```

   `binlog.cnf`配置文件内容，[canal官方说明](https://github.com/alibaba/canal/wiki/QuickStart#%E5%87%86%E5%A4%87) 

   ```mysql
   [mysqld]
   log-bin=mysql-bin # 开启 binlog
   binlog-format=ROW # 选择 ROW 模式
   server_id=1 # 配置 MySQL replaction 需要定义，不要和 canal 的 slaveId 重复
   ```

   拉取镜像，启动项目

   ```bash
   $ docker-compose up
   ```

   更改`mysql`权限 ，使用`root`登录到`mysql`

   ```mysql
   CREATE USER kafka IDENTIFIED BY 'kafka';  # 创建与docker-compose.yml中对应的用户和密码
   GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'kafka'@'%'; #给mysql用户权限
   -- GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ; #也可以给所有权限
   FLUSH PRIVILEGES;
   ```

   创建一个数据库`kafkadb`并添加一个`user`表

   向`user`表插入数据

   ```mysql
   INSERT INTO user ( `id`, `username`) VALUES ( 1, 'yan');
   ```

   好像没有数据过来（至少我的是这样）

4. 排查问题

   首先查看是否镜像运行正常

   ```bash
   $ docker ps 
   ```

   发现没有问题，只有依次排查每个镜像日志，先从`canal`查起

   ```bash
   $ docker exec -it <canal 镜像> bash
   #然后进入canal-server文件夹
   $ cd canal-server
   $ cat logs/example/example.log
   #发现出错了，以下为片段
   # Caused by: java.util.concurrent.ExecutionException: org.apache.kafka.common.errors.TimeoutException: Failed to update metadata after 60000 ms.
   ```

   百度后，发现和[这个问题很像](https://blog.csdn.net/maoyuanming0806/article/details/80553632)，那应该就是我们前面说的`kafka`的`ip`设置成`localhost`导致的，尝试更改一下，问题解决

   再插入数据，可以看到数据被接收到了

## 后记

其实在部署之间，遇到了很多问题，由于这次是问题重现，有些问题并没有再出现

例如有自己写的`Producer`程序推送消息时，报错`There is no leader for this topic-partition as we are in the middle of a leadership election` 这是由于，没有设置`KAFKA_BROKER_ID`导致每次构建项目，都重新生成了`brokder id`，可以在构建项目时在其后添加`--no-recreate` ，[可以再这里找到](https://github.com/wurstmeister/kafka-docker/issues/516) 。记得使用`docker-compose rm -vfs`删除后再构建项目

也有`zookeeper`报错`Zookeeper Report Error:KeeperErrorCode = NoNode`，[可以再这里找到](https://github.com/wurstmeister/kafka-docker/issues/427) 

等等