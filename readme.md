### 配置描述

```shell
[mysqld]
server_id = 1   # 服务id，不能相同
log-bin= mysql-bin
read-only=0     # 为1时，为只读模式，只限制普通用户
replicate-ignore-db=mysql   # 不需要同步的库，不会同步用户
replicate-ignore-db=sys
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/
```

### 启动服务

```shell
docker run --name mysql-master -d -p 3307:3306 -e MYSQL_ROOT_PASSWORD=123456 -v /tmp/mysql/master/data:/var/lib/mysql -v /tmp/mysql/master/conf/my.cnf:/etc/mysql/my.cnf  mysql:5.7 
docker run --name mysql-slave -d -p 3308:3306 -e MYSQL_ROOT_PASSWORD=123456 -v /tmp/mysql/slave/data:/var/lib/mysql -v /tmp/mysql/slave/conf/my.cnf:/etc/mysql/my.cnf  mysql:5.7
```

#### 迁移数据

- 没有数据，则跳过

```shell
# 导出数据
mysqldump -u root -p --all-databases --lock-all-tables > ~/master_db.sql
# 拷贝
docker cp mysql-master:/root/master_db.sql /Users/myard/Desktop/master_db.sql
docker cp /Users/myard/Desktop/master_db.sql mysql-slave:/root/master_db.sql
# 导入数据
mysql -u root -p -h 127.0.0.1 --port=3306 < ~/master_db.sql
```

### 主从配置

master配置

```mysql
# 创建用于从服务器同步数据使用的帐号
grant replication slave on *.* to 'slave'@'%' identified by 'slave';
# 刷新权限
flush privileges;
# 查看主机状态，并记录File和Position的值
show master status;
"""
File	Position	Binlog_Do_DB	Binlog_Ignore_DB	Executed_Gtid_Set
mysql-bin.000003	520			
"""
```

 <img src="https://gitee.com/zhouyuanmin/images/raw/master/imgs/20210720141844.png" alt="image-20210720141844193" style="zoom:50%;" />

slave配置

```mysql
# 从机连接主机
CHANGE MASTER TO master_host = '127.0.0.1',
master_user = 'slave',
master_password = 'slave',
master_log_file = 'mysql-bin.000003',
master_log_pos = 520,
master_port = 3307;
"""
master_host：主服务器Ubuntu的ip地址
master_log_file: 前面查询到的主服务器日志文件名
master_log_pos: 前面查询到的主服务器日志文件位置
"""
# 启动slave服务器，并查看同步状态
start slave;  -- stop slave;
show slave status;
"""
检查
Slave_IO_Running项是Yes
Slave_SQL_Running项是Yes
则没有问题
"""
```

 

