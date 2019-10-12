# 基于K8s、Strimzi的Kafka Connect实战

## 0. 源码地址

[https://github.com/wiselyman/kafka-in-battle](https://github.com/wiselyman/kafka-in-battle)

## 1. Operator Framework

Operator Framework是一个用来管理k8s原生应用（Operator）的开源工具。

Operator Framework支持的Operator分享地址：[https://operatorhub.io](https://operatorhub.io)。

如安装Kafka使用**Strimzi Apache Kafka Operator**，地址为：[https://operatorhub.io/operator/strimzi-kafka-operator](https://operatorhub.io/operator/strimzi-kafka-operator) 。

打开Strimzi Apache Kafka Operator页面，右侧有**install**按钮，按照页面提示进行Operator安装。

## 2. 安装Operator Lifecycle Manager

Operator Lifecycle Manager是Operator Framework的一部分，OLM扩展了k8s提供声明式方法安装、管理、更新Operator以及他们的依赖。

点击页面上的**install**显示如何安装**Strimzi Apache Kafka Operator**，我们首先第一步要安装**Operator Lifecycle Manager**（不要执行下句命令）：

```shell
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/0.12.0/install.sh | bash -s 0.12.0
```

该命令需要使用quay.io的镜像，我们需采取从源码安装，并修改源码中的镜像地址加速。

源码地址：[https://github.com/operator-framework/operator-lifecycle-manager/releases](https://github.com/operator-framework/operator-lifecycle-manager/releases),当前最新版本为`0.12.0`。

- 下载：[https://github.com/operator-framework/operator-lifecycle-manager/releases/download/0.12.0/crds.yaml](https://github.com/operator-framework/operator-lifecycle-manager/releases/download/0.12.0/crds.yaml)

- 下载：[https://github.com/operator-framework/operator-lifecycle-manager/releases/download/0.12.0/olm.yaml](https://github.com/operator-framework/operator-lifecycle-manager/releases/download/0.12.0/olm.yaml)

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

## 4. 安装Kafka集群

下载[https://raw.githubusercontent.com/strimzi/strimzi-kafka-operator/0.14.0/examples/kafka/kafka-persistent.yaml](https://raw.githubusercontent.com/strimzi/strimzi-kafka-operator/0.14.0/examples/kafka/kafka-persistent.yaml),主要修改的是所需存储空间为5Gi作为测试条件，这里的存储需要K8s集群中有默认的`StorageClass`。

```yaml
apiVersion: kafka.strimzi.io/v1beta1
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    version: 2.3.0
    replicas: 3
    listeners:
      plain: {}
      tls: {}
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      delete.topic.enable: "true"
      transaction.state.log.min.isr: 2
      log.message.format.version: "2.3"
    storage:
      type: jbod
      volumes:
      - id: 0
        type: persistent-claim
        size: 5Gi
        deleteClaim: false
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 5Gi
      deleteClaim: false
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

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

本节将外部的SQL Server中的表person（字段只有`id`和`name`）通过Kafka Connect同步至K8s集群里的PostgreSQL中。

### 5.1 开启SQL Server数据库的CDC（Change Data Capture）功能

#### 5.1.1 启用数据库CDC

```sql
USE bs_portal
EXEC sys.sp_cdc_enable_db;
```

`bs_portal`为数据库名，此时会自动给我们创建cdc的schema和相关表：

- `captured_columns`
- `change_tables`
- `dbo_person_CT`
- `ddl_history`
- `index_columns`
- `lsn_time_mapping`

可使用下面sql语句查询已开启CDC的数据库：

```sql
select * from sys.databases where is_cdc_enabled = 1 
```

#### 5.1.2 启用表的CDC

```sql
USE bs_portal 
EXEC sys.sp_cdc_enable_table  
    @source_schema = 'dbo',  
    @source_name = 'person',  
    @role_name = 'cdc_admin',
    @supports_net_changes = 1;
```

`@source_name`为表名，查询表开启CDS的sql语句：

```sql
select name, is_tracked_by_cdc from sys.tables where object_id = OBJECT_ID('dbo.person')  
```

查看新增的job

```sql
SELECT job_id,name,enabled,date_created,date_modified FROM msdb.dbo.sysjobs ORDER BY date_created
```

确定用户有权限访问CDC表

```
EXEC sys.sp_cdc_help_change_data_capture;
```

#### 5.1.3 开启“SQL Server 代理”

检查安装了SQL Server的操作系统中“服务”中是否开启了“SQL Server 代理”。

#### 5.1.4 关闭CDC

关闭数据库的CDC

```sql
USE bs_portal
EXEC sys.sp_cdc_disable_db;
```

关闭表的CDC

```
USE bs_portal
EXEC sys.sp_cdc_disable_table   
    @source_schema = 'dbo',  
    @source_name = 'person',  
    @capture_instance = 'all';
```

### 5.2 SQL Server To PosgreSQL

#### 5.2.1 准备Kafka Connect镜像

输入插件（source）：下载SQL Server Connector plugin：[http://central.maven.org/maven2/io/debezium/debezium-connector-sqlserver/](http://central.maven.org/maven2/io/debezium/debezium-connector-sqlserver/)；输出插件（sink）：下载Kafka Connect JDBC：[https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc](https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc)。

新建Dockerfile文件，将`debezium-connector-sqlserver-0.10.0.Final-plugin.zip`解压放置到Dockerfile相同目录下的`plugins`目录；在`plugins`目录下新建目录`kafka-connect-jdbc`,解压`confluentinc-kafka-connect-jdbc-5.3.1.zip`，将`lib`下的`kafka-connect-jdbc-5.3.1.jar`和`postgresql-9.4.1212.jar`放置在`kafka-connect-jdbc`。

编写Dockerfile

```dockerfile
FROM strimzi/kafka:0.14.0-kafka-2.3.0
USER root:root
COPY ./plugins/ /opt/kafka/plugins/
USER 1001
MAINTAINER 285414629@qq.com
```

使用阿里云“容器镜像服务”（[https://cr.console.aliyun.com/](https://cr.console.aliyun.com/)）编译镜像，目前我们的源码地址位于：[https://github.com/wiselyman/kafka-in-battle](https://github.com/wiselyman/kafka-in-battle)。

- “镜像仓库”->“创建镜像仓库”:

  1. 仓库名称：kafka-connect-form-sql-to-jdbc

  2. 仓库类型：公开
- 下一步后，选择“Github”标签页，使用自己的GitHub库，“构建设置”只勾选“海外机器构建”，然后点击“创建镜像仓库”。
- 点击镜像仓库列表中的“kafka-connect-mysql-postgres”->“构建”->“添加规则”：

  1. 类型：Branch

  2. Branch/Tag：master

  3. Dockerfile目录：/sqlserver-to-jdbc/

  4. Dockfile文件名：Dockerfile

  5. 镜像版本：0.1
- 确认后，“构建规则设置”->“立即构建”，“构建日志”显示“构建状态”为“成功”即可。

#### 5.2.2 安装Kafka Connect

编写Kafka Connect集群部署文件`kafka-connect-sql-postgres.yml`：

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

执行安装

```shell
kubectl apply -f kafka-connect-sql-postgres.yml -n kafka
```

查询已安装的插件

```shell
kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X GET http://my-connect-cluster-connect-api:8083/connector-plugins
```

结果如：

```json
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

#### 5.2.3 使用Helm安装PostgreSQL

使用helm安装PostgreSQL,这里的PostgreSQL库来自于[https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts/](https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts/)，可在Helm中配置。

对PostgreSQL的账号、密码、初始化数据库、服务类型进行定制后安装：

```shell
helm install --name my-pg --set global.storageClass=standard,postgresUser=wisely,postgresPassword=zzzzzz,postgresDatabase=center,service.type=NodePort,service.nodePort=5432 stable/postgresql
```

#### 5.2.4 Kafka Connect Source配置

编写source配置：`sql-server-source.json`

```javascript
{
  "name": "sql-server-connector",
  "config": {
    "connector.class" : "io.debezium.connector.sqlserver.SqlServerConnector",
    "tasks.max" : "1",
    "database.server.name" : "exam",
    "database.hostname" : "172.16.8.221",
    "database.port" : "1433",
    "database.user" : "sa",
    "database.password" : "sa",
    "database.dbname" : "bs_portal",
    "database.history.kafka.bootstrap.servers" : "my-cluster-kafka-bootstrap:9092",
    "database.history.kafka.topic": "schema-changes.person",
    "table.whitelist": "dbo.person"
  }
}
```

编写sink配置：`postgres-sink.json`

```javascript
{
  "name": "postgres-sink",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "tasks.max": "1",
    "topics": "exam.dbo.MH_YCZM",
    "connection.url": "jdbc:postgresql://my-pg-postgresql.default.svc.cluster.local:5432/center?user=wisely&password=zzzzzz",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "auto.create": "true",
    "insert.mode": "upsert",
    "delete.enabled": "true",
    "pk.fields": "IPDZ",
    "pk.mode": "record_key"
  }
}
```

#### 5.2.5 使用

将配置文件提交到Kafka Connect

```shell
cat sql-server-source.json | kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://my-connect-cluster-connect-api:8083/connectors -d @-
```

```shell
cat postgres-sink.json| kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://my-connect-cluster-connect-api:8083/connectors -d @-
```

查看所有的Connector

```shell
kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X GET http://my-connect-cluster-connect-api:8083/connectors
```

删除Connect

```
kubectl exec -i -n kafka my-cluster-kafka-0 -- curl -X DELETE http://my-connect-cluster-connect-api:8083/connectors/postgres-sink
```

查看所有的topic

```
kubectl exec -n kafka my-cluster-kafka-0   -- bin/kafka-topics.sh --list --zookeeper localhost:2181
```

查看SQL Server Connector中的数据

```shell
kubectl exec -i -n kafka my-cluster-kafka-0 -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic exam.dbo.person --from-beginning
```

我们此时查看PostgreSQL数据库已经有了person表和数据，当对SQL Server新增、修改、删除数据时，PostgreSQL中也会同步更新。