---
title: mysql8主从分离
date: 2020-01-01
categories:
 - mysql
tags:
 - mysql
---
# MySQL 8.0 主从分离部署全指南：从原理到实战

在高并发业务场景中，单台 MySQL 数据库往往难以承受读写压力 —— 大量查询操作（读）会占用数据库资源，导致写入操作（写）响应变慢。**主从分离**通过将读写操作拆分到不同服务器，能有效提升数据库吞吐量，同时实现数据备份与故障转移。本文将详细讲解 MySQL 8.0 主从分离的部署流程，从原理到实操一步到位。

## 一、主从分离核心原理

MySQL 主从分离基于**二进制日志（binlog）** 实现，核心流程如下：



1.  **主库（Master）**：负责处理所有写操作（INSERT/UPDATE/DELETE），并将操作记录到 binlog 中。

2.  **从库（Slave）**：通过 IO 线程读取主库的 binlog，写入本地中继日志（relay log），再通过 SQL 线程执行中继日志中的操作，实现与主库数据同步。

3.  **应用层**：写操作指向主库，读操作（SELECT）分配到从库，实现读写分离。

**优势**：



*   减轻主库压力：读操作分流到从库，主库专注处理写操作。

*   数据备份：从库实时同步主库数据，可作为热备份。

*   高可用：主库故障时，可手动或自动切换到从库。

## 二、部署前准备

### 1. 环境要求



*   两台服务器（或虚拟机），建议配置：2 核 CPU + 4GB 内存（生产环境可更高）。

*   操作系统：CentOS 7/8 或 Ubuntu 20.04+（本文以 CentOS 7 为例）。

*   MySQL 版本：8.0.x（主从库版本需保持一致，避免兼容性问题）。

*   网络：主从库需互通，关闭防火墙或开放 3306 端口。



| 角色 | IP 地址         | 用途              |
| -- | ------------- | --------------- |
| 主库 | 192.168.1.100 | 处理写操作，生成 binlog |
| 从库 | 192.168.1.101 | 同步主库数据，处理读操作    |

### 2. 安装 MySQL 8.0

主从库均需安装 MySQL 8.0，以 CentOS 7 为例：



```
\# 安装MySQL官方仓库

wget https://dev.mysql.com/get/mysql80-community-release-el7-7.noarch.rpm

rpm -ivh mysql80-community-release-el7-7.noarch.rpm

\# 安装MySQL 8.0

yum install -y mysql-community-server

\# 启动服务并设置开机自启

systemctl start mysqld

systemctl enable mysqld

\# 查看初始密码（首次登录用）

grep 'temporary password' /var/log/mysqld.log
```

登录 MySQL 并修改密码（主从库均需执行）：



```
mysql -u root -p

\# 输入初始密码后执行以下命令（密码需包含大小写、数字和特殊字符）

ALTER USER 'root'@'localhost' IDENTIFIED BY 'YourStrong!Passw0rd';
```

## 三、主库（Master）配置

### 1. 修改 MySQL 配置文件

编辑 `/etc/my.cnf`，添加以下配置：



```
\[mysqld]

\# 启用binlog，指定日志文件名前缀

log\_bin = /var/lib/mysql/mysql-bin

\# 主库唯一ID（1-255，不可与从库重复）

server-id = 100

\# 同步的数据库（如需同步所有库可省略）

binlog\_do\_db = test\_db

\# 忽略同步的数据库（可选）

binlog\_ignore\_db = mysql

\# binlog格式（ROW模式更安全，记录每行数据变化）

binlog\_format = ROW

\# 自动删除7天前的binlog，避免磁盘占满

expire\_logs\_days = 7
```

重启 MySQL 使配置生效：



```
systemctl restart mysqld
```

### 2. 创建用于主从同步的用户

登录主库 MySQL，创建一个仅用于从库同步的用户（如 `repl`）：



```
\-- 允许从库IP（192.168.1.101）使用repl用户连接

CREATE USER 'repl'@'192.168.1.101' IDENTIFIED BY 'ReplPass!2024';

\-- 授予复制权限

GRANT REPLICATION SLAVE ON \*.\* TO 'repl'@'192.168.1.101';

\-- 刷新权限

FLUSH PRIVILEGES;
```

### 3. 查看主库状态

执行以下命令获取 binlog 文件名和位置（从库配置需要）：



```
\-- 锁定数据库，禁止写入（避免配置期间数据变化）

FLUSH TABLES WITH READ LOCK;

\-- 查看主库状态

SHOW MASTER STATUS;
```

输出如下（记录 `File` 和 `Position` 的值）：



```
+------------------+----------+--------------+------------------+-------------------+

\| File             | Position | Binlog\_Do\_DB | Binlog\_Ignore\_DB | Executed\_Gtid\_Set |

+------------------+----------+--------------+------------------+-------------------+

\| mysql-bin.000001 |      156 | test\_db      | mysql            |                   |

+------------------+----------+--------------+------------------+-------------------+
```

> 注意：此时不要关闭 MySQL 终端，配置完从库后再解锁（执行 
>
> `UNLOCK TABLES;`
>
> ）。

## 四、从库（Slave）配置

### 1. 修改 MySQL 配置文件

编辑 `/etc/my.cnf`，添加以下配置：



```
\[mysqld]

\# 从库唯一ID（不可与主库重复）

server-id = 101

\# 中继日志文件名前缀

relay\_log = /var/lib/mysql/mysql-relay-bin

\# 从库只读（禁止写操作，可选但推荐）

read\_only = 1

\# 忽略同步的数据库（与主库保持一致）

replicate\_ignore\_db = mysql
```

重启 MySQL：



```
systemctl restart mysqld
```

### 2. 配置从库同步主库

登录从库 MySQL，执行以下命令配置主库信息（替换为实际值）：



```
\-- 停止从库同步进程

STOP SLAVE;

\-- 配置主库连接信息

CHANGE MASTER TO

MASTER\_HOST = '192.168.1.100',  -- 主库IP

MASTER\_USER = 'repl',           -- 同步用户

MASTER\_PASSWORD = 'ReplPass!2024',  -- 同步用户密码

MASTER\_LOG\_FILE = 'mysql-bin.000001',  -- 主库binlog文件名（从SHOW MASTER STATUS获取）

MASTER\_LOG\_POS = 156;           -- 主库binlog位置（从SHOW MASTER STATUS获取）

\-- 启动从库同步进程

START SLAVE;
```

### 3. 查看从库同步状态

执行以下命令验证同步状态：



```
SHOW SLAVE STATUS\G
```

关键参数需满足：



*   `Slave_IO_Running: Yes`（IO 线程正常运行）

*   `Slave_SQL_Running: Yes`（SQL 线程正常运行）

若均为 `Yes`，说明主从同步配置成功！

> 此时回到主库终端，执行 
>
> `UNLOCK TABLES;`
>
>  解锁数据库。

## 五、验证主从同步

### 1. 主库写入数据

在主库创建测试数据库和表，并插入数据：



```
\-- 创建数据库（与binlog\_do\_db配置一致）

CREATE DATABASE test\_db;

USE test\_db;

\-- 创建表

CREATE TABLE user (

&#x20; id INT PRIMARY KEY AUTO\_INCREMENT,

&#x20; name VARCHAR(50) NOT NULL

);

\-- 插入数据

INSERT INTO user (name) VALUES ('Alice'), ('Bob');
```

### 2. 从库验证数据

在从库查询数据，确认已同步：



```
USE test\_db;

SELECT \* FROM user;
```

若输出以下结果，说明同步成功：



```
+----+-------+

\| id | name  |

+----+-------+

\|  1 | Alice |

\|  2 | Bob   |

+----+-------+
```

## 六、常见问题与解决

### 1. 从库 `Slave_IO_Running: Connecting`



*   **原因**：网络不通、主库防火墙未开放 3306 端口、同步用户密码错误。

*   **解决**：


    *   检查主从库网络：`ping ``192.168.1.100`

    *   开放主库端口：`firewall-cmd --add-port=3306/tcp --permanent && firewall-cmd --reload`

    *   验证同步用户权限：在从库执行 `mysql -h ``192.168.1.100`` -u repl -p`

### 2. 从库 `Slave_SQL_Running: No`



*   **原因**：主从库数据不一致（如从库已有同名表）、binlog 格式不兼容。

*   **解决**：


    *   重置从库状态：`STOP SLAVE; RESET SLAVE ALL;`（清空之前的配置）

    *   重新配置主从同步（参考步骤四.2）

    *   确保主从库表结构一致，必要时手动同步初始数据（如用 `mysqldump` 导出主库数据导入从库）。

### 3. 主库 binlog 文件过大



*   **解决**：配置 `expire_logs_days` 自动清理旧日志，或手动删除：



```
\-- 查看binlog文件

SHOW BINARY LOGS;

\-- 删除指定文件之前的日志

PURGE BINARY LOGS TO 'mysql-bin.000003';
```

## 七、总结与扩展

本文详细讲解了 MySQL 8.0 主从分离的部署流程，核心步骤包括：



1.  主库启用 binlog 并创建同步用户。

2.  从库配置中继日志并关联主库。

3.  验证同步状态和数据一致性。

**扩展方向**：



*   **多从库架构**：一台主库可配置多台从库，进一步分流读压力。

*   **半同步复制**：通过 `rpl_semi_sync_master_enabled` 配置，确保主库收到至少一个从库的确认后再返回成功，提升数据安全性。

*   **读写分离中间件**：使用 MyCat、ShardingSphere 等工具自动分配读写请求，无需手动指定主从库地址。

掌握主从分离是 MySQL 高可用架构的基础，合理配置可显著提升系统稳定性与性能。
