# Canal to Redis Example

极客时间《后端存储实战课：17 | 大厂都是怎么做MySQL to Redis同步的？》示例代码。这个示例代码代码演示如何利用Canal从MySQL实时同步数据到Redis中去。

## 环境要求

运行示例之前需要安装：

* JDK 1.8以上版本
* Maven 3.3.9以上版本

```bash
$java -version
java version "1.8.0_202"
Java(TM) SE Runtime Environment (build 1.8.0_202-b08)
Java HotSpot(TM) 64-Bit Server VM (build 25.202-b08, mixed mode)

$mvn -version
Apache Maven 3.3.9 (bb52d8502b132ec0a5a3f4c09453c07478323dc5; 2015-11-11T00:41:47+08:00)
```

## 准备MySQL

本机安装好MySQL，运行在默认端口3306上。创建数据库和表：

```sql
CREATE DATABASE test;
use test;
CREATE TABLE `account_balance` (
  `user_id` int NOT NULL COMMENT '用户ID',
  `balance` int NOT NULL COMMENT '余额',
  `timestamp` datetime NOT NULL COMMENT '时间戳',
  `log_id` int NOT NULL COMMENT '最后一笔交易的流水号',
  PRIMARY KEY (`user_id`)
);
```

然后来配置MySQL，我们需要在MySQL的配置文件中开启Binlog，并设置Binlog的格式为ROW格式。

```properties
[mysqld]
log-bin=mysql-bin # 开启Binlog
binlog-format=ROW # 设置Binlog格式为ROW
server_id=1 # 配置一个ServerID
```

给Canal开一个专门的MySQL用户并授权，确保这个用户有复制Binlog的权限：

```sql
CREATE USER canal IDENTIFIED BY 'canal';  
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
FLUSH PRIVILEGES;
```

重启一下MySQL，确保所有的配置生效。重启后检查一下当前的Binlog文件和位置：

```sql
mysql> show master status;
+---------------+----------+--------------+------------------+-------------------+
| File          | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+---------------+----------+--------------+------------------+-------------------+
| binlog.000009 |      155 |              |                  |                   |
+---------------+----------+--------------+------------------+-------------------+
```

记录下File和Position二列的值。

## 准备Redis

本机安装Redis，运行在默认端口6379上。

## 安装部署Canal

下载并解压Canal 最新的1.1.4版本到本地并解压：

```bash
wget https://github.com/alibaba/canal/releases/download/canal-1.1.4/canal.deployer-1.1.4.tar.gz
tar zvfx canal.deployer-1.1.4.tar.gz
```

编辑Canal的实例配置文件canal/conf/example/instance.properties，以便让Canal连接到我们的MySQL上。

```properties
canal.instance.gtidon=false

# position info
canal.instance.master.address=127.0.0.1:3306
canal.instance.master.journal.name=binlog.000009
canal.instance.master.position=155
canal.instance.master.timestamp=
canal.instance.master.gtid=

# username/password
canal.instance.dbUsername=canal
canal.instance.dbPassword=canal
canal.instance.connectionCharset = UTF-8
canal.instance.defaultDatabaseName=test
# table regex
canal.instance.filter.regex=.*\\..*
```

这个配置文件需要配置MySQL的连接地址、库名、用户名和密码之外，还需要配置canal.instance.master.journal.name和canal.instance.master.position这两个属性，取值就是刚刚记录的File和Position二列。然后就可以启动Canal服务了：

```bash
$canal/bin/startup.sh
```

启动之后看一下日志文件canal/logs/example/example.log，如果里面没有报错，就说明启动成功并连接到我们的MySQL上了。

## 下载编译源代码

```bash
$git clone git@github.com:liyue2008/canal-to-redis-example.git
$cd canal-to-redis-example
$mvn package
```

## 运行示例

```bash
$java -jar target/canal-to-redis-example-1.0-SNAPSHOT.jar
```

## 验证同步效果

在账户余额表插入一条记录：

```sql
mysql> insert into account_balance values (888, 100, NOW(), 999);
```

然后来看一下Redis缓存：

```redis
127.0.0.1:6379> get 888
"{\"log_id\":\"999\",\"balance\":\"100\",\"user_id\":\"888\",\"timestamp\":\"2020-03-08 16:18:10\"}"
```

## 参考

[后端存储实战课](https://time.geekbang.org/column/intro/287)
[Canal@GitHub](https://github.com/alibaba/canal)  
[Canal安装详解](https://juejin.im/post/5d88b109f265da03ef7a51f1)