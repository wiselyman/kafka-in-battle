# 基于K8s的Kafka实战

## 1. Operator Framework

- Operator Framework是一个用来管理k8s原生应用（Operator）的开源工具。

- Operator Framework支持的Operator分享地址：https://operatorhub.io/。

- 如安装Kafka使用**Strimzi Apache Kafka Operator**，地址为：https://operatorhub.io/operator/strimzi-kafka-operator 。

- 打开Strimzi Apache Kafka Operator页面，右侧有**install**按钮，按照页面提示进行Operator安装。

## 2. 安装Operator Lifecycle Manager

Operator Lifecycle Manager是Operator Framework的一部分，OLM扩展了k8s提供声明式方法安装、管理、更新Operator以及他们的依赖。

点击页面上的**install**显示如何安装**Strimzi Apache Kafka Operator**，我们首先第一步要安装**Operator Lifecycle Manager**（不要执行下句命令）：

```shell
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/0.12.0/install.sh | bash -s 0.12.0
```

该命令需要使用quay.io的镜像，我们需采取从源码安装，并修改源码中的镜像地址加速。

源码下载地址：https://github.com/operator-framework/operator-lifecycle-manager/releases

当前最新版本为`0.12.0`

下载：https://github.com/operator-framework/operator-lifecycle-manager/releases/download/0.12.0/crds.yaml

下载：https://github.com/operator-framework/operator-lifecycle-manager/releases/download/0.12.0/olm.yaml

将`olm.yml`中：

```yaml
quay.io ->  quay.azk8s.cn
```

执行安装：

```shell
kubectl apply -f crds.yaml
kubectl apply -f olm.yaml
```

## 3. 安装Strimzi Apache Kafka Operator

```shell
kubectl create -f https://operatorhub.io/install/strimzi-kafka-operator.yaml
```

使用下面命令观察Operator启动情况

```shell
kubectl get csv -n operators
```

显示如下则安装成功

```
wangyunfeis-MacBook-Pro:olm wangyunfei$ kubectl get csv -n operators
NAME                               DISPLAY                         VERSION   REPLACES                           PHASE
strimzi-cluster-operator.v0.14.0   Strimzi Apache Kafka Operator   0.14.0    strimzi-cluster-operator.v0.13.0   Succeeded
```

## 4. 安装Kafka

- 下载：https://raw.githubusercontent.com/strimzi/strimzi-kafka-operator/0.14.0/examples/kafka/kafka-persistent.yaml

```shell
kubectl apply -f kafka-persistent.yml -n kafka 
```

- 发送消息测试

```shell
kubectl exec -i -n kafka my-cluster-kafka-0 -- bin/kafka-console-producer.sh --broker-list my-cluster-kafka-bootstrap:9092 --topic strimizi-my-topic
```

- 接受消息测试

```shell
kubectl exec -i -n kafka  my-cluster-kafka-0 -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic strimizi-my-topic --from-beginning
```

- 显示集群Topic

```shell
kubectl exec -n kafka my-cluster-kafka-0   -- bin/kafka-topics.sh --list --zookeeper localhost:2181
```

## 5. Kafka Connect

### 5.1 SQL Server To PosgreSQL

- 本节将外部的SQL Server中的数据通过Kafka Connect同步至K8s集群里的PostgreSQL中。

- 输入（source）：下载SQL Server Connector plugin：http://central.maven.org/maven2/io/debezium/debezium-connector-sqlserver/

- 输出（sink）：下载Kafka Connect JDBC：https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc

- 新建Dockerfile文件。

- 将`debezium-connector-sqlserver-0.10.0.Final-plugin.zip`解压放置到Dockerfile相同目录下的`plugins`目录。

- 在`plugins`目录下新建目录`kafka-connect-jdbc`,解压`confluentinc-kafka-connect-jdbc-5.3.1.zip`，将`lib`下的`kafka-connect-jdbc-5.3.1.jar`和`postgresql-9.4.1212.jar`放置在`kafka-connect-jdbc`

- 编写Dockerfile

  ```dockerfile
  FROM strimzi/kafka:0.14.0-kafka-2.3.0
  USER root:root
  COPY ./plugins/ /opt/kafka/plugins/
  USER 1001
  MAINTAINER 285414629@qq.com
  ```

- 使用阿里云“容器镜像服务”（[https://cr.console.aliyun.com/](https://cr.console.aliyun.com/)）编译镜像，目前我们的源码地址位于:[https://github.com/wiselyman/kafka-in-battle](https://github.com/wiselyman/kafka-in-battle)。

- “镜像仓库”->“创建镜像仓库”:

  ![](images/aliyun1.png)

  ![](images/aliyun2.png)

- 点击镜像仓库列表中的“kafka-connect-mysql-postgres”->“构建”->“添加规则”：

  ![](images/aliyun3.png)

- 确认后，“构建规则设置”->“立即构建”，“构建日志”显示“构建状态”为“成功”即可。

- 编写Kafka Connect集群部署文件`kafka-connect-sql-postgres.yml`：

  ```yaml
  apiVersion: kafka.strimzi.io/v1beta1
  kind: KafkaConnect
  metadata:
    name: my-connect-cluster
  spec:
    version: 2.3.0
    replicas: 1
    bootstrapServers: 'my-cluster-kafka-bootstrap:9093'
    image: registry.cn-hangzhou.aliyuncs.com/wiselyman/kafka-connect-from-sql-to-jdbc:0.1
    tls:
      trustedCertificates:
        - secretName: my-cluster-cluster-ca-cert
          certificate: ca.crt
  ```

- 执行安装

  ```shell
  kubectl apply -f kafka-connect-sql-postgres.yml -n kafka
  ```

- 查询已安装的插件

  ```shell
  kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X GET http://my-connect-cluster-connect-api:8083/connector-plugins
  ```

  结果如：

  ```
  [{
  	"class": "io.confluent.connect.jdbc.JdbcSinkConnector",
  	"type": "sink",
  	"version": "5.3.1"
  }, {
  	"class": "io.confluent.connect.jdbc.JdbcSourceConnector",
  	"type": "source",
  	"version": "5.3.1"
  }, {
  	"class": "io.debezium.connector.sqlserver.SqlServerConnector",
  	"type": "source",
  	"version": "0.10.0.Final"
  }, {
  	"class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
  	"type": "sink",
  	"version": "2.3.0"
  }, {
  	"class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
  	"type": "source",
  	"version": "2.3.0"
  }]
  ```

- 使用helm安装PostgreSQL

  定制的`value.yaml`如下：

  ```yaml
  global.storageClass: standard
  postgresqlUsername: wisely
  postgresqlPassword: zzzzzz
  postgresqlDatabase: center
  service.type: NodePort
  ```

  安装:

  ```shell
  helm install --name my-pg --set global.storageClass=standard,postgresUser=wisely,postgresPassword=zzzzzz,postgresDatabase=center,service.type=NodePort,service.nodePort=5432 stable/postgresql
  ```

  

