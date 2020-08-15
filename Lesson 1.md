# Lesson 1

## 课程要求

本地下载 TiDB，TiKV，PD 源代码，改写源码并编译部署以下环境：<br>● 1 TiDB<br>● 1 PD<br>● 3 TiKV
	

改写后<br>● 使得 TiDB 启动事务时，会打一个 “hello transaction” 的日志



## 项目实施

### 下载源码

```
git clone https://github.com/pingcap/tidb.git
git clone https://github.com/tikv/tikv.git
git clone https://github.com/pingcap/pd.git
```

### 配置环境

调试首先创建了Ubuntu 20 虚拟机，安装Golang、Rust、Cmake、build-essential，mysql-server等组件

需要注意Rust需要切换至nightly版本，并且注意source ~/.cargo/env

### 编译

使用make命令编译tidb和pd，使用cargo build命令编译tikv

### 运行

创建一个脚本以简化启动过程

```shell
#!/bin/env bash
./pd/bin/pd-server --name=pd1 \
                --data-dir=data/pd1 \
                --client-urls="http://127.0.0.1:2379" \
                --peer-urls="http://127.0.0.1:2380" \
                --initial-cluster="pd1=http://127.0.0.1:2380" \
                --log-file=data/pd1.log &

./tikv/target/release/tikv-server --pd-endpoints="127.0.0.1:2379" \
                --addr="127.0.0.1:20160" \
                --data-dir=data/tikv1 \
                --log-file=data/tikv1.log &
./tikv/target/release/tikv-server --pd-endpoints="127.0.0.1:2379" \
                --addr="127.0.0.1:20161" \
                --data-dir=data/tikv2 \
                --log-file=data/tikv2.log &
./tikv/target/release/tikv-server --pd-endpoints="127.0.0.1:2379" \
                --addr="127.0.0.1:20162" \
                --data-dir=data/tikv3 \
                --log-file=data/tikv3.log &

./tidb/bin/tidb-server --store tikv --path 127.0.0.1:2379 \
		--log-file=data/tidb.log &
```

启动完成后访问Sql数据库

```
mysql -h 127.0.0.1 -u root -P 4000
```

显示

```shell
blue@ubuntu:~$ mysql -h 127.0.0.1 -u root -P 4000
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.25-TiDB-v4.0.0-beta.2-953-gbaedc336a-dirty TiDB Server (Apache License 2.0) Community Edition, MySQL 5.7 compatible

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```

成功连接至TiDB

本次的作业任务需要查看TiDB日志。在TiDB 4.0 版本之后引入了Dashboard，能够方便查看、检索日志

使用浏览器登录，root账户默认无密码

```
http://127.0.0.1:2379/dashboard/
```
![](https://github.com/Blue-Sail/TiDB-Practice/tree/master/Img/lesson 1-1.jpg)
<img src="Img/lesson 1-1.jpg" style="zoom:75%;" />

在日志查询页面可以设置查询范围与日志类型等参数，在右侧可以看到，启动节点状态，包含1个PD，1个TiDB，3个TiKV节点

## 修改源码

经过查找定位，在session/session.go 找到了logStmt函数，可以看到注释中显示，该函数能产生与SQL语句相关的日志

```go
// logStmt logs some crucial SQL including: CREATE USER/GRANT PRIVILEGE/CHANGE PASSWORD/DDL etc and normal SQL
// if variable.ProcessGeneralLog is set.
func logStmt(execStmt *executor.ExecStmt, vars *variable.SessionVars)
```

所以在函数内部switch-case中添加

```
	case *ast.BeginStmt:
			logutil.BgLogger().Info("hello transaction")
```

保存并重新make

## 修改结果

使用与之前相同步骤启动TiDB并使用Mysql连接，连接后使用 BEGIN; 启动事务

```sql
mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)

```

打开DashBoard，查看是否产生了新的日志，可以找到

```
2020-08-14 17:51:39 INFO TiDB 0.0.0.0 [session.go:2265] ["hello transaction"]
```

<img src="Img/lesson 1-2.png" style="zoom:75%;" />



至此完成要求
