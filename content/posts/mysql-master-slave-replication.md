---
title: MySQL主从复制集群搭建与读写分离配置教程
date: 2026-04-03
tags:
  - MySQL
  - 主从复制
  - 读写分离
  - 数据库集群
categories:
  - 数据库
  - 运维
---

## 1. 环境准备

### 1.1 服务器规划

| 角色 | IP地址 | 端口 | 服务器配置 |
|------|--------|------|------------|
| 主库 (Master) | 192.168.1.10 | 3306 | 2核4G |
| 从库1 (Slave1) | 192.168.1.11 | 3306 | 2核4G |
| 从库2 (Slave2) | 192.168.1.12 | 3306 | 2核4G |
| 负载均衡 (ProxySQL) | 192.168.1.13 | 6033 | 2核4G |

### 1.2 软件版本

- MySQL: 8.0.36
- ProxySQL: 2.5.5
- CentOS: 7.9

## 2. 主库配置

### 2.1 安装MySQL

```bash
# 安装MySQL 8.0
yum localinstall https://repo.mysql.com/mysql80-community-release-el7-3.noarch.rpm
yum install mysql-community-server -y

# 启动服务
systemctl start mysqld
systemctl enable mysqld

# 获取初始密码
grep 'temporary password' /var/log/mysqld.log

# 登录并修改密码
mysql -u root -p
ALTER USER 'root'@'localhost' IDENTIFIED BY 'YourStrongPassword123!';
```

### 2.2 配置主库参数

编辑 `/etc/my.cnf` 文件：

```ini
[mysqld]
bind-address = 0.0.0.0
server-id = 1
log-bin = mysql-bin
binlog-format = ROW
sync-binlog = 1
binlog-ignore-db = information_schema
binlog-ignore-db = mysql
binlog-ignore-db = performance_schema
binlog-ignore-db = sys

transaction-isolation = READ-COMMITTED
innodb_flush_log_at_trx_commit = 1
innodb_support_xa = 1
innodb_file_per_table = 1
```

重启MySQL服务：

```bash
systemctl restart mysqld
```

### 2.3 创建复制用户

```sql
CREATE USER 'repl'@'%' IDENTIFIED BY 'ReplPassword123!';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
```

### 2.4 查看主库状态

```sql
SHOW MASTER STATUS;
```

记录输出中的 `File` 和 `Position` 值，后续从库配置需要使用。

## 3. 从库配置

### 3.1 安装MySQL

在所有从库服务器上执行相同的MySQL安装步骤。

### 3.2 配置从库参数

编辑 `/etc/my.cnf` 文件（从库1）：

```ini
[mysqld]
bind-address = 0.0.0.0
server-id = 2  # 从库2设置为3
log-bin = mysql-bin
relay-log = relay-bin
read-only = 1
skip-slave-start = 1

# 其他参数同主库
```

重启MySQL服务：

```bash
systemctl restart mysqld
```

### 3.3 配置主从复制

在从库上执行：

```sql
CHANGE MASTER TO
    MASTER_HOST = '192.168.1.10',
    MASTER_USER = 'repl',
    MASTER_PASSWORD = 'ReplPassword123!',
    MASTER_LOG_FILE = 'mysql-bin.000001',  # 替换为实际值
    MASTER_LOG_POS = 156;  # 替换为实际值

START SLAVE;
```

### 3.4 验证复制状态

```sql
SHOW SLAVE STATUS\G;
```

确保 `Slave_IO_Running` 和 `Slave_SQL_Running` 均为 `Yes`。

## 4. 配置ProxySQL实现读写分离

### 4.1 安装ProxySQL

```bash
# 添加ProxySQL仓库
cat > /etc/yum.repos.d/proxysql.repo << EOF
[proxysql_repo]
name=ProxySQL YUM repository
baseurl=https://repo.proxysql.com/ProxySQL/proxysql-2.5.x/centos/7/
gpgcheck=1
gpgkey=https://repo.proxysql.com/ProxySQL/repo_pub_key
EOF

# 安装ProxySQL
yum install proxysql -y

# 启动服务
systemctl start proxysql
systemctl enable proxysql
```

### 4.2 配置ProxySQL

登录ProxySQL管理界面：

```bash
mysql -u admin -padmin -h 127.0.0.1 -P 6032
```

### 4.3 添加MySQL服务器

```sql
-- 添加主库（写节点）
INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight, max_connections) VALUES (10, '192.168.1.10', 3306, 1, 1000);

-- 添加从库（读节点）
INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight, max_connections) VALUES (20, '192.168.1.11', 3306, 1, 1000);
INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight, max_connections) VALUES (20, '192.168.1.12', 3306, 1, 1000);

-- 保存配置
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

### 4.4 配置读写分离规则

```sql
-- 配置查询规则
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply) VALUES (1, 1, '^SELECT.*FOR UPDATE$', 10, 1);
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply) VALUES (2, 1, '^SELECT', 20, 1);

-- 保存配置
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```

### 4.5 创建应用用户

```sql
-- 创建监控用户
INSERT INTO mysql_users (username, password, active, default_hostgroup, max_connections) VALUES ('monitor', 'monitor', 1, 10, 100);

-- 创建应用用户
INSERT INTO mysql_users (username, password, active, default_hostgroup, max_connections) VALUES ('app_user', 'AppPassword123!', 1, 10, 1000);

-- 保存配置
LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;
```

### 4.6 配置监控

```sql
-- 配置监控
SET mysql-monitor_username='monitor';
SET mysql-monitor_password='monitor';

-- 保存配置
LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;
```

## 5. 测试验证

### 5.1 连接ProxySQL

```bash
mysql -u app_user -pAppPassword123! -h 192.168.1.13 -P 6033
```

### 5.2 测试写操作

```sql
CREATE DATABASE test_db;
USE test_db;
CREATE TABLE users (id INT PRIMARY KEY AUTO_INCREMENT, name VARCHAR(50), email VARCHAR(100));
INSERT INTO users (name, email) VALUES ('张三', 'zhangsan@example.com');
```

### 5.3 测试读操作

```sql
SELECT * FROM users;
```

### 5.4 验证数据同步

在从库上执行：

```sql
USE test_db;
SELECT * FROM users;
```

确保数据已从主库同步到从库。

### 5.5 验证读写分离

在ProxySQL管理界面查看查询分发情况：

```sql
SELECT * FROM stats_mysql_query_digest;
SELECT * FROM stats_mysql_connection_pool;
```

## 6. 高可用配置

### 6.1 配置Keepalived实现ProxySQL高可用

安装Keepalived：

```bash
yum install keepalived -y
```

配置Keepalived（主节点）：

```ini
# /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.100/24
    }
}
```

配置Keepalived（从节点）：

```ini
# /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 99
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.100/24
    }
}
```

启动Keepalived：

```bash
systemctl start keepalived
systemctl enable keepalived
```

## 7. 监控与维护

### 7.1 监控主从复制状态

创建监控脚本：

```bash
#!/bin/bash

SLAVE_STATUS=$(mysql -u root -pYourStrongPassword123! -e "SHOW SLAVE STATUS\G" | grep -E "Slave_IO_Running|Slave_SQL_Running")

if echo "$SLAVE_STATUS" | grep -q "Yes\|Yes"; then
    echo "主从复制正常"
else
    echo "主从复制异常"
    # 发送告警
fi
```

### 7.2 定期备份

配置定时备份：

```bash
# 每天凌晨2点备份
0 2 * * * mysqldump -u root -pYourStrongPassword123! --all-databases --single-transaction | gzip > /backup/mysql_$(date +%Y%m%d).sql.gz
```

### 7.3 常见问题排查

1. **复制延迟**：检查网络带宽、主库写入量、从库性能
2. **复制错误**：查看 `SHOW SLAVE STATUS\G` 中的错误信息
3. **ProxySQL连接失败**：检查网络、用户权限、配置

## 8. 性能优化

### 8.1 MySQL参数优化

```ini
# 主库优化
innodb_buffer_pool_size = 2G
innodb_log_file_size = 512M
innodb_io_capacity = 2000
innodb_flush_method = O_DIRECT

# 从库优化
innodb_buffer_pool_size = 2G
read_buffer_size = 1M
read_rnd_buffer_size = 4M
```

### 8.2 ProxySQL优化

```sql
-- 调整连接池大小
SET mysql-max_connections = 2000;
SET mysql-default_charset = 'utf8mb4';

-- 保存配置
LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;
```

## 9. 总结

通过本教程，您已经成功搭建了MySQL主从复制集群并配置了读写分离。主要实现了：

1. **数据冗余**：主从复制确保数据安全
2. **读写分离**：提高系统并发处理能力
3. **高可用**：ProxySQL和Keepalived保障服务可靠性
4. **性能优化**：合理配置参数提升系统性能

这种架构适用于中小规模应用，能够满足大部分业务场景的需求。对于大规模应用，可以考虑使用MySQL Cluster或其他分布式数据库解决方案。

## 10. 参考资料

- [MySQL官方文档](https://dev.mysql.com/doc/refman/8.0/en/replication.html)
- [ProxySQL官方文档](https://proxysql.com/documentation/)
- [Keepalived官方文档](https://keepalived.org/documentation.html)
