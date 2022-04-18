# debezium_oracle_cdc
## 1. 概述
   > Debezium 是一组分布式服务，用于捕获数据库中的更改，以便您的应用程序可以看到这些更改并做出响应。Debezium 将每个数据库表中的所有行级更改记录在更改事件流中，应用程序只需读取这些流即可按照它们发生的相同顺序查看更改事件。
   >  Debezium 的 Oracle 连接器捕获并记录发生在 Oracle 服务器上的数据库中的行级更改，包括在连接器运行时添加的表。您可以将连接器配置为针对特定的模式和表子集发出更改事件，或者忽略、屏蔽或截断特定列中的值。
 > 本demo基于debezium最新版本1.9.0实现，通过oracle数据库flinkods用户动态捕获zhruiqi用户下表结构及数据变更信息，以json格式实时推送kafka。
## 2. Debezium架构
下图显示了基于 Debezium 的变更数据捕获管道的架构：
![debezium-architecture](vx_images/316242210238885.png)
## 3. 环境部署
### 3.1. Oracle开启归档
Oracle 数据库可以作为独立实例安装，也可以使用 Oracle Real Application Cluster (RAC) 安装。Debezium Oracle 连接器与这两种安装类型兼容。
1. Oracle LogMiner 所需的配置（单节点参考）：
```sql
sqlplus / as sysdba
select log_mode from v$database; --查看归档模式,未开启进入下一步
alter system set db_recovery_file_dest_size = 10G;
alter system set db_recovery_file_dest = '/opt/oracle/oradata/recovery_area' scope=spfile;
shutdown immediate;
startup mount;
alter database archivelog;
alter database open;
```
2. 此外，必须为捕获的表或数据库启用补充日志记录，以便数据更改能够捕获已更改数据库行的之前状态。
```sql
--特定表上进行配置
ALTER TABLE inventory.customers ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
--数据库级别启用最小补充日志记录
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
```
### 3.2. 创建oracle数据库flinkods用户和权限配置
```sql
--创建表空间
sqlplus / as sysdba
  CREATE TABLESPACE logminer_tbs DATAFILE '/opt/oracle/oradata/ORCLCDB/logminer_tbs.dbf'
    SIZE 25M REUSE AUTOEXTEND ON MAXSIZE UNLIMITED;
  exit;

--创建用户
sqlplus / as sysdba

  CREATE USER flinkods IDENTIFIED BY flinkods
    DEFAULT TABLESPACE logminer_tbs
    QUOTA UNLIMITED ON logminer_tbs
    CONTAINER=ALL;
--赋权
  --创建会话 使连接器能够连接到 Oracle
  GRANT CREATE SESSION TO flinkods CONTAINER=ALL; 
  --设置容器 使连接器能够在可插入数据库之间切换。仅当 Oracle 安装启用了容器数据库支持 (CDB) 时才需要这样做
  GRANT SET CONTAINER TO flinkods CONTAINER=ALL; 
  --选择 V_$DATABASE 使连接器能够读取V$DATABASE表
  GRANT SELECT ON V_$DATABASE to flinkods CONTAINER=ALL; 
  --闪回任何表 使连接器能够执行闪回查询，这是连接器执行数据初始快照的方
  GRANT FLASHBACK ANY TABLE TO flinkods CONTAINER=ALL; 
  --选择任何表格 使连接器能够读取任何表
  GRANT SELECT ANY TABLE TO flinkods CONTAINER=ALL; 
  --SELECT_CATALOG_ROLE 使连接器能够读取 Oracle LogMiner 会话所需的数据字典
  GRANT SELECT_CATALOG_ROLE TO flinkods CONTAINER=ALL; 
  --EXECUTE_CATALOG_ROLE 使连接器能够将数据字典写入 Oracle 重做日志，这是跟踪架构更改所必需的
  GRANT EXECUTE_CATALOG_ROLE TO flinkods CONTAINER=ALL; 
  --选择任何交易 使快照进程能够对任何事务执行闪回快照查询。当FLASHBACK ANY TABLE被授予时，这也应该被授予
  GRANT SELECT ANY TRANSACTION TO flinkods CONTAINER=ALL; 
  --日志挖掘 在较新版本的 Oracle 中添加了此角色，作为授予对 Oracle LogMiner 及其包的完全访问权限的一种方式。在没有此角色的旧版 Oracle 上，您可以忽略此授权
  GRANT LOGMINING TO flinkods CONTAINER=ALL; 

  --创建表 使连接器能够在其默认表空间中创建其刷新表。刷新表允许连接器显式控制将 LGWR 内部缓冲区刷新到磁盘
  GRANT CREATE TABLE TO flinkods CONTAINER=ALL; 
  --锁定任何表 使连接器能够在模式快照期间锁定表。如果通过配置显式禁用快照锁，则可以安全地忽略此授权
  GRANT LOCK ANY TABLE TO flinkods CONTAINER=ALL; 
  --创建任何序列 使连接器能够在其默认表空间中创建序列
  GRANT CREATE SEQUENCE TO flinkods CONTAINER=ALL; 

  --在 DBMS_LOGMNR 上执行 使连接器能够运行DBMS_LOGMNR包中的方法。这是与 Oracle LogMiner 交互所必需的。在较新版本的 Oracle 上，这是通过LOGMINING角色授予的，但在旧版本上，必须明确授予
  GRANT EXECUTE ON DBMS_LOGMNR TO flinkods CONTAINER=ALL; 
  --在 DBMS_LOGMNR_D 上执行 使连接器能够运行DBMS_LOGMNR_D包中的方法。这是与 Oracle LogMiner 交互所必需的。在较新版本的 Oracle 上，这是通过LOGMINING角色授予的，但在旧版本上，必须明确授予
  GRANT EXECUTE ON DBMS_LOGMNR_D TO flinkods CONTAINER=ALL; 

--选择 V_$… 使连接器能够读取这些表。连接器必须能够读取有关 Oracle 重做和归档日志以及当前事务状态的信息，以准备 Oracle LogMiner 会话。没有这些授权，连接器将无法运行
  GRANT SELECT ON V_$LOG TO flinkods CONTAINER=ALL; 
  GRANT SELECT ON V_$LOG_HISTORY TO flinkods CONTAINER=ALL; 
  GRANT SELECT ON V_$LOGMNR_LOGS TO flinkods CONTAINER=ALL; 
  GRANT SELECT ON V_$LOGMNR_CONTENTS TO flinkods CONTAINER=ALL; 
  GRANT SELECT ON V_$LOGMNR_PARAMETERS TO flinkods CONTAINER=ALL; 
  GRANT SELECT ON V_$LOGFILE TO flinkods CONTAINER=ALL; 
  GRANT SELECT ON V_$ARCHIVED_LOG TO flinkods CONTAINER=ALL; 
  GRANT SELECT ON V_$ARCHIVE_DEST_STATUS TO flinkods CONTAINER=ALL; 
  GRANT SELECT ON V_$TRANSACTION TO flinkods CONTAINER=ALL; 

  exit;
```
### 3.3. 安装Apache ZooKeeper
启动： zkServer.sh start
### 3.4. 安装Apache Kafka
> 安装kafka
> 下载debezium-connector-oracle-1.9.0.Final-plugin.tar.gz并解压，将debezium-connector-oracle 目录下的jar包都拷贝一份到${KAFKA_HOME}/libs中
> 下载oracle客户端instantclient-basic-linux.x64-21.1.0.0.0，版本和oracle服务对应，把jar包复制到${KAFKA_HOME}/libs
> kafka环境修改：${KAFKA_HOME}/config 目录下：单机修改 connect-standalone.propertie，集群修改 connect-distributed.properties，增加的配置项如下：
```sql
bootstrap.servers=192.168.1.161:9092
key.converter.schemas.enable=false
value.converter.schemas.enable=false
internal.key.converter=org.apache.kafka.connect.json.JsonConverter
internal.value.converter=org.apache.kafka.connect.json.JsonConverter
internal.key.converter.schemas.enable=false
internal.value.converter.schemas.enable=false
cleanup.policy=compact
rest.host.name=192.168.1.161
rest.port=8083
plugin.path=/opt/bigdata/debezium/debezium-connector-oracle
```

### 3.5.启动kafka

```
/opt/bigdata/kafka/bin/kafka-server-start.sh  /opt/bigdata/kafka/config/server.properties &
```
### 3.6.以环境配置方式启动connect-distributed
```
export KAFKA_LOG4J_OPTS=-Dlog4j.configuration=file:/opt/bigdata/kafka/config/connect-log4j.properties
/opt/bigdata/kafka/bin/connect-distributed.sh /opt/bigdata/kafka/config/connect-distributed.properties

```
### 3.7.提交Oracle-connector，监视Oracle数据库
```json
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://192.168.1.161:8083/connectors/ -d '
{
"name": "debezium2oracle",
"config": {
"connector.class" : "io.debezium.connector.oracle.OracleConnector",
"tasks.max" : "1",
"database.server.name" : "ORCL",
"database.hostname" : "192.168.1.127",
"database.port" : "1521",
"database.user" : "flinkods",
"database.password" : "flinkods",
"database.dbname" : "ORCL",
"database.schema" : "ZHRUIQI",
"database.connection.adapter": "logminer", 
"database.tablename.case.insensitive": "true",
"table.include.list" : "ZHRUIQI.*", 
"snapshot.mode" : "initial",
"schema.include.list" : "ZHRUIQI",
"database.history.kafka.bootstrap.servers" : "192.168.1.161:9092,192.168.1.162:9092,192.168.1.163:9092",
"database.history.kafka.topic": "debezium2oracle"
}
}'

```
### 3.8.消费端查看数据
```shell
/opt/bigdata/kafka/bin/kafka-console-consumer.sh --bootstrap-server 192.168.1.161:9092,192.168.1.162:9092, 192.168.1.163:9092  --topic debezium2oracle  --from-beginning
/opt/bigdata/kafka/bin/kafka-console-producer.sh --broker-list 192.168.1.161:9092 --topic flinktopic

```
### 3.9.查看创建的kafka connector列表
```shell
http://192.168.1.161:8083/connectors
http://192.168.1.161:8083/connectors/debezium2oracle/status
http://192.168.1.161:8083/connectors/debezium2oracle/config
curl -X DELETE http://192.168.1.161:8083/connectors/debezium2oracle
```
### 3.10.启动flink服务和客户端
```sql
./bin/sql-client.sh embedded
CREATE TABLE ZHANGSAN (
ID  STRING,
NAME STRING
) WITH (
'connector' = 'kafka',
'topic' = 'debezium2oracle',
'properties.bootstrap.servers' = '192.168.1.161:9092',
'properties.group.id' = 'zhangsan',
'scan.startup.mode' = 'earliest-offset',
'value.format' = 'debezium-json'
);
Select  *  from ZHANGSAN;
```
### 3.11.安装debezium-ui
```
curl -L "https://github.com/docker/compose/releases/download/2.4.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$ docker run -it --rm --name debezium-ui -p 8780:8080 -e KAFKA_CONNECT_URIS=http://192.168.1.161:8083 debezium/debezium-ui:{debezium-version}
```
![debezium-ui](vx_images/478652123220459.jpg)


