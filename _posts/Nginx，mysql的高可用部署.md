# Nginx，mysql的高可用部署

## NacOS集群部署

Nacos集群部署需要 3个或3个以上Nacos节点，而且要使用 MySQL 作为数据源，数据库也需要支持高可用。主要用于服务配置的数据持久化。

要求最少三个节点才能组成一个有效的集群的原因是Nacos选举算法的特殊性。一般选举算法都建议奇数个节点，2个节点的数据一致性可能无法保障。

其部署的结构如下，在微服务和nacos集群之间可以采用直连的方式，也可以采用nginx集群来进行代理。

![Nacos高可用集群搭建与使用_MySQL_02](../assets/blog_res/Nginx%EF%BC%8Cmysql%E7%9A%84%E9%AB%98%E5%8F%AF%E7%94%A8%E9%83%A8%E7%BD%B2.assets/25091429_646eb675db80016468.png)



这里有以下几部分组成：

- 高可用 Nginx 集群（代理模式需要）

- Nacos 集群(至少三个实例)

- 高可用数据库集群，比如MySQL的高可用方案

配置集群时，nacos的conf目录下有配置文件cluster.conf，每行配置成ip:port即可。

集群下的 Nacos 节点状态分为LEADER 和 FOLLOWER 两种，跟熟悉的主从架构相似。

##### **Nacos 节点间的数据同步过程**

Nacos 节点间的数据同步过程

在 Nacos的选举算法中，只有 Leader 才拥有数据处理与信息分发的权利。因此当微服务启动时，假如注册中心指定为 Follower 节点，则步骤如下：

- Follower 会自动将注册心跳包转给 Leader 节点；

- Leader 节点完成实质的注册登记工作；

- 完成注册后向其他 Follower 节点发起“同步注册日志”的指令；

- 所有可用的 Follower 在收到指令后进行“ack应答”，通知 Leader 消息已收到；

- 当 Leader 接收过半数 Follower 节点的 “ack 应答”后，返回给微服务“注册成功”的响应信息。

对于 Nacos 集群中的N个节点来说，只要 UP 状态节点不少于"1+N/2"，集群就能正常运行。但少于“1+N/2”，集群仍然可以提供基本服务，但已无法保证 Nacos 各节点数据一致性。

### 1. 环境准备

- JDK8+
- Mysql 5.6.5+

### 2. 下载安装文件

https://github.com/alibaba/nacos/releases?spm=a2c6h.12873639.article-detail.17.5b0052fdXdMWcf

为了安全需要，还是创建一个用户**nacos**用于启停nacos服务。

### 3. 初始化MySQL数据库

NacOS自带的derby数据库并不能很好的适应在集群模式下的部署，所以要将数据库改为MySQL

在mysql新建一个数据库nacos_config，**执行初始化脚本mysql-schema.sql**，脚本在naocs安装文件夹**conf**下

![在这里插入图片描述](../assets/blog_res/Nginx%EF%BC%8Cmysql%E7%9A%84%E9%AB%98%E5%8F%AF%E7%94%A8%E9%83%A8%E7%BD%B2.assets/bVcYSy2)

### 4. 修改配置

配置文件也在naocs安装文件夹**conf**下，文件名称**application.properties**，配置文件中提供修改默认端口、访问路径的属性等。

找到*Connect URL of DB*的选项，修改成MySQL数据库的内容，数据库地址，用户名和密码根据实际情况修改

```properties
spring.datasource.platform=mysql

db.num=1
db.url.0=jdbc:mysql://IP_ADDRESS:PORT/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
db.user=root
db.password=123456
```

集群配置文件在安装文件夹**conf**下，按照cluster.conf.example将每个nacos节点按照IP:PORT格式配置即可，按行分割。

```properties
#IP:PORT
192.168.9.121:8848 #节点1
192.168.9.122:8848 #节点2
192.168.9.122:8848 #节点3
```

配置完后将文件名改成cluster.conf

**将Nacos的每个节点按照上述内容操作，安装并修改配置**

### 5 启动nacos 集群

在Linux 系统中，nacos 的默认启动方式是cluster 模式，即集群模式。

```bash
cd YOUR_LOCATION/nacos/nacos/bin/
./startup.sh 
```

### 6. 客户端连接

![img](../assets/blog_res/Nginx%EF%BC%8Cmysql%E7%9A%84%E9%AB%98%E5%8F%AF%E7%94%A8%E9%83%A8%E7%BD%B2.assets/794174-20230512163244890-2002514048.png)

由于该方案中客户端和NacOS之间不使用nginx，所以在spring项目中的nacos采用直连模式，在spring的配置文件中将多个NacOS节点地址以逗号分隔开。

```properties
spring.cloud.nacos.discovery.server-addr=192.168.9.121:8848,192.168.9.122:8848,192.168.9.122:8848
```

## MySQL高可用

MySQL常见的高可用部署方案包括主从复制、主主复制、MySQL Cluster（NDB Cluster）、Galera Cluster、Percona XtraDB Cluster等。下面是这些方案的简要介绍：

1. **主从复制**：主服务器处理写操作和部分读操作，从服务器复制主服务器的数据，主要用于读操作。在主从复制中，如果主服务器发生故障，可以手动或自动切换到从服务器上。
2. **主主复制**：两台或多台服务器互为主从服务器，可以处理读写操作，提高了写操作的并发性。在主主复制中，如果一台主服务器发生故障，可以将另一台主服务器切换为唯一的主服务器。
3. **MySQL Cluster**：基于MySQL的集群解决方案，提供高可用性和可伸缩性。MySQL Cluster将数据分片存储在多台服务器上，实现数据的分布式存储和高可用性。
4. **Galera Cluster**：基于Percona XtraDB Cluster和MariaDB Cluster的多主集群解决方案，提供同步复制和自动故障切换功能，确保数据一致性和高可用性。
5. **Percona XtraDB Cluster**：Percona提供的基于Galera Cluster的高可用性解决方案，使用InnoDB存储引擎和Galera Replication实现数据同步和故障切换。
6. **MySQL Group Replication**：MySQL官方提供的基于组复制的高可用性解决方案，支持多主复制和自动故障转移。
7. **MHA（MySQL Master High Availability）**：是一个开源的MySQL高可用性解决方案，可以实现自动故障转移和主从切换。
8. **InnoDB Cluster**：是官方提供的高可用方案,是 MySQL 的一种高可用性(HA)解决方案，它通过使用 MySQL Group Replication 来实现数据的自动复制和高可用性。

**这里采用主从复制的方法部署MySQL集群，其具体原理如下**

#### MySQL主从复制

MySQL主从复制可以分为一主一丛或一主多从。主服务器只负责写，而从服务器只负责读，从而提高了效率减轻压力。

主从复制可以分为：

- 主从同步：当用户写数据主服务器必须和从服务器同步了才告诉用户写入成功，等待时间比较长。
- 主从异步：只要用户访问写数据主服务器，立即返回给用户。
- 主从半同步：当用户访问写数据主服务器写入并同步其中一个从服务器就返回给用户成功。

MySQL 主从复制是基于主服务器在binlog跟踪所有对数据库的更改。因此，要进行复制，必须在主服务器上启用binlog。

每个从服务器从主服务器接收已经记录到日志的数据。当一个从服务器连接到主服务器时，它通知主服务器从服务器日志中读取最后一个更新成功的位置。

从服务器接收从那时发生起的任何更新，并在主机上执行相同的更新。然后封锁等待主服务器通知的更新。

从服务器执行备份不会干扰主服务器，在备份过程中主服务器可以继续处理更新。

**MySQL读写分离基本原理是让主数据库处理写操作，从数据库处理读操作。主数据库将写操作的变更同步到各个从节点。**

### 1. 环境准备和安装

两台CentOS服务器，确保他们之间可以相互ping通

安装MySQL，版本在5.6.5以上，可以使用yum安装。

要注意以下两点：

- 主从服务器操作系统版本和位数一致。

- Mysql版本一致。

### 2. 配置主服务器

编辑/ect/my.cnf配置文件

```properties
[mysqld]
datadir=/data/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql

#主从复制配置
innodb_flush_log_at_trx_commit=1
sync_binlog=1
#需要备份的数据库
binlog-do-db=YOUR_DATABASE_1
binlog-do-db=YOUR_DATABASE_2
#不需要备份的数据库
binlog-ignore-db=mysql
binlog-ignore-db=information_schema

#启动二进制文件
log-bin=mysql-bin

#服务器ID
server-id=1

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```

保存并关闭文件，重启MySQL

然后导出主服务器的备份，用来配置从服务器

### 3. 配置从服务器

添加或修改/ect/my.cnf配置文件

```properties
[mysqld]
#服务器ID，每个MySQL服务器都需要一个唯一的标识符，用于区分主从服务器
server-id=1

#需要备份的数据库
binlog-do-db=YOUR_DATABASE_1
binlog-do-db=YOUR_DATABASE_2
#不需要备份的数据库
binlog-ignore-db=mysql
binlog-ignore-db=information_schema

# 确保服务器能及时获取主服务器的修改，以保持数据一致性
relay-log=mysql-relay
```

保存并关闭文件，重启MySQL

然后导入主服务器的备份



## Redis集群

Redis集群的实现方式主要是主从复制和哨兵模式。主从复制负责数据的同步，而哨兵模式负责监控节点的状态。

#### 主从复制

主从复制是 Redis 高可用服务的最基础的保证，实现方案就是将从前的一台 Redis 服务器，同步数据到多台从 Redis 服务器上，即一主多从的模式，且主从服务器之间采用的是「读写分离」的方式。

![主从复制](../assets/blog_res/Nginx%EF%BC%8Cmysql%E7%9A%84%E9%AB%98%E5%8F%AF%E7%94%A8%E9%83%A8%E7%BD%B2.assets/%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6.png)

#### 哨兵模式

当 Redis 的主从服务器出现故障宕机时，需要手动进行恢复。哨兵模式做到了可以监控主从服务器，并且提供**主从节点故障转移的功能。**

[![哨兵模式](../assets/blog_res/Nginx%EF%BC%8Cmysql%E7%9A%84%E9%AB%98%E5%8F%AF%E7%94%A8%E9%83%A8%E7%BD%B2.assets/%E5%93%A8%E5%85%B5%E6%A8%A1%E5%BC%8F.png)](https://eraseryao.github.io/assets/blog_res/2023-03-16-数据库.assets/哨兵模式.png)

哨兵其实是一个运行在特殊模式下的Redis进程，相当于一个观察节点。**哨兵一般是以集群的方式部署的，至少需要三个结点**，其主要负责三件事情：**监控，选主，通知**。哨兵结点之间是通过Redis的**Pub-Sub**机制来相互发现的。

