# Lesson 2

## 课程要求

使用 sysbench、go-ycsb 和 go-tpc 分别对 TiDB 进行测试并且产出测试报告。测试报告需要包括以下内容：

● 部署环境的机器配置（CPU、内存、磁盘规格型号），拓扑结构（TiDB、TiKV各部署于哪些节点）
● 调整过后的 TiDB 和 TiKV 配置
● 测试输出结果
● 关键指标的监控截图
	○ TiDB Query Summary 中的 qps 与 duration
	○ TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标
	○ TiKV Details 面板中 grpc 的 qps 以及 duration
输出：写出你对该配置与拓扑环境和 workload下 TiDB 集群负载的分析，提出你认为的 TiDB的性能的瓶颈所在（能提出大致在哪个模块即可）
截止时间：下周二（8.25） 24:00:00（逾期提交不给分）

## 项目实施

### 集群部署

首先，配置4台虚拟机，每台分配100G硬盘，2G内存，2核CPU，网络连接为桥接方式。完成一台虚拟机的配置和基本工具的安装后，使用VMWare的克隆功能，配置另外三台虚拟机。需要注意的是，TiUP需要每台设备开放root账户的ssh登录，依次执行以下过程即可。

```shell
# 安装openssh-server
blue@ubuntu:~$ sudo apt install openssh-server
# 设置root用户密码：
blue@ubuntu:~$ sudo passwd root
#允许root用户登录；编辑配置文件：
blue@ubuntu:~$ sudo vim /etc/ssh/sshd_config
# 设置其中PermitRootLogin prohibit-password为PermitRootLogin yes
# 重启ssh服务：
blue@ubuntu:~$ sudo systemctl restart sshd

```

而后，在其中一台虚拟机上开始部署，参考TiUP文档，依次执行

```
#TiUp部署文档
https://docs.pingcap.com/zh/tidb/stable/production-deployment-using-tiup
```

```shell
使用普通用户登录中控机，以 tidb 用户为例，后续安装 TiUP 及集群管理操作均通过该用户完成：

# 1.执行如下命令安装 TiUP 工具：
blue@ubuntu:~$ curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
# 2.按如下步骤设置 TiUP 环境变量：
# 重新声明全局环境变量：
blue@ubuntu:~$ source /home/blue/.bashrc
# 确认 TiUP 工具是否安装：
blue@ubuntu:~$ which tiup
# 3.安装 TiUP cluster 组件
blue@ubuntu:~$ tiup cluster
# 4.如果已经安装，则更新 TiUP cluster 组件至最新版本：
blue@ubuntu:~$ tiup update --self && tiup update cluster
# 预期输出 “Update successfully!” 字样。
# 5.验证当前 TiUP cluster 版本信息。执行如下命令查看 TiUP cluster 组件版本：
blue@ubuntu:~$ tiup --binary cluster
```

下一步编辑配置文件

```shell
blue@ubuntu:~$ gedit topology.yaml
```

本人的配置文件如下，具有3个tikv，3个pd，1个tidb，分别部署在四台虚拟机上

```
# # Global variables are applied to all deployments and used as the default value of
# # the deployments if a specific deployment value is missing.
global:
  user: "blue"
  ssh_port: 22
  deploy_dir: "/tidb-deploy"
  data_dir: "/tidb-data"
  
monitored:
  node_exporter_port: 9100
  blackbox_exporter_port: 9115
  
server_configs:
  tidb:
    log.slow-threshold: 300
  tikv:
    # server.grpc-concurrency: 4
    # raftstore.apply-pool-size: 2
    # raftstore.store-pool-size: 2
    # rocksdb.max-sub-compactions: 1
    # storage.block-cache.capacity: "16GB"
    # readpool.unified.max-thread-count: 12
    readpool.storage.use-unified-pool: false
    readpool.coprocessor.use-unified-pool: true
  pd:
    schedule.leader-schedule-limit: 4
    schedule.region-schedule-limit: 2048
    schedule.replica-schedule-limit: 64
    replication.enable-placement-rules: true

pd_servers:
  - host: 192.168.31.181
  - host: 192.168.31.214
  - host: 192.168.31.6

tidb_servers:
  - host: 192.168.31.104

tikv_servers:
  - host: 192.168.31.181
  - host: 192.168.31.214
  - host: 192.168.31.6
    
monitoring_servers:
  - host: 192.168.31.104

grafana_servers:
  - host: 192.168.31.104

alertmanager_servers:
  - host: 192.168.31.104
```

下一步，开始实际部署

```shell
blue@ubuntu:~$ tiup cluster deploy tidb-benchmark nightly ./topology.yaml --user root -p
```

部署完成后显示，Deployed cluster `tidb-benchmark` successfully，下一步，启动集群

```shell
blue@ubuntu:~$ tiup cluster start tidb-benchmark
```

启动成功后，查看集群状态

```shell
blue@ubuntu:~$ tiup cluster display tidb-benchmark
Starting component `cluster`: /home/blue/.tiup/components/cluster/v1.0.9/tiup-cluster display tidb-benchmark
tidb Cluster: tidb-benchmark
tidb Version: nightly
ID                    Role          Host            Ports        OS/Arch       Status  Data Dir                      Deploy Dir
--                    ----          ----            -----        -------       ------  --------                      ----------
192.168.31.104:9093   alertmanager  192.168.31.104  9093/9094    linux/x86_64  Up      /tidb-data/alertmanager-9093  /tidb-deploy/alertmanager-9093
192.168.31.104:3000   grafana       192.168.31.104  3000         linux/x86_64  Up      -                             /tidb-deploy/grafana-3000
192.168.31.181:2379   pd            192.168.31.181  2379/2380    linux/x86_64  Up|L    /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.31.214:2379   pd            192.168.31.214  2379/2380    linux/x86_64  Up      /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.31.6:2379     pd            192.168.31.6    2379/2380    linux/x86_64  Up|UI   /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.31.104:9090   prometheus    192.168.31.104  9090         linux/x86_64  Up      /tidb-data/prometheus-9090    /tidb-deploy/prometheus-9090
192.168.31.104:4000   tidb          192.168.31.104  4000/10080   linux/x86_64  Up      -                             /tidb-deploy/tidb-4000
192.168.31.181:20160  tikv          192.168.31.181  20160/20180  linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.31.214:20160  tikv          192.168.31.214  20160/20180  linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.31.6:20160    tikv          192.168.31.6    20160/20180  linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160


```

启动成功，配置如下

| 编号 |       IP       | CPU  | 内存 |   储存   |      OS      | 组件 |
| :--: | :------------: | :--: | :--: | :------: | :----------: | :--: |
|  1   | 192.168.31.104 | 2核  |  2G  | 虚拟硬盘 | Ubuntu 20.04 | TiDB |
|  2   | 192.168.31.181 | 2核  |  2G  | 虚拟硬盘 | Ubuntu 20.04 | Tikv |
|  2   | 192.168.31.181 | 2核  |  2G  | 虚拟硬盘 | Ubuntu 20.04 | Tikv |
|  3   | 192.168.31.214 | 2核  |  2G  | 虚拟硬盘 | Ubuntu 20.04 | Tikv |
|  3   | 192.168.31.214 | 2核  |  2G  | 虚拟硬盘 | Ubuntu 20.04 |  PD  |
|  4   |  192.168.31.6  | 2核  |  2G  | 虚拟硬盘 | Ubuntu 20.04 |  PD  |
|  4   |  192.168.31.6  | 2核  |  2G  | 虚拟硬盘 | Ubuntu 20.04 |  PD  |

###  Sysbench 性能测试

安装sysbench

```shell
blue@ubuntu:~$ curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash
blue@ubuntu:~$ sudo apt -y install sysbench
```

编辑配置文件

```shell
# blue@ubuntu:~$ gedit config
mysql-host=192.168.31.104
mysql-port=4000
mysql-user=root
mysql-db=sysbench
time=600
threads=128
report-interval=10
db-driver=mysql
```

创建数据库及导入数据

```shell
# 创建数据库
mysql> create database sbtest;
# 导入数据之前先设置为乐观事务模式，导入结束后再设置回悲观模式
mysql> set global tidb_disable_txn_auto_retry = off;
mysql> set global tidb_txn_mode="optimistic";
# 导入数据
blue@ubuntu:~$ sysbench --config-file=config oltp_point_select --tables=32 --table-size=10000000 prepare
```

下面进入测试环节

```shell
# Point select 测试命令
blue@ubuntu:~$ sysbench --config-file=config oltp_point_select --threads=128 --tables=32 --table-size=5000000 run
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 128
Report intermediate results every 10 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 10s ] thds: 128 tps: 11847.41 qps: 11847.41 (r/w/o: 11847.41/0.00/0.00) lat (ms,95%): 23.10 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 128 tps: 9176.61 qps: 9176.61 (r/w/o: 9176.61/0.00/0.00) lat (ms,95%): 30.26 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 128 tps: 11315.86 qps: 11315.86 (r/w/o: 11315.86/0.00/0.00) lat (ms,95%): 23.95 err/s: 0.00 reconn/s: 0.00
...
[ 150s ] thds: 128 tps: 14336.84 qps: 14336.84 (r/w/o: 14336.84/0.00/0.00) lat (ms,95%): 17.95 err/s: 0.00 reconn/s: 0.00
[ 160s ] thds: 128 tps: 13764.01 qps: 13764.01 (r/w/o: 13764.01/0.00/0.00) lat (ms,95%): 18.95 err/s: 0.00 reconn/s: 0.00
...
[ 590s ] thds: 128 tps: 13554.64 qps: 13554.64 (r/w/o: 13554.64/0.00/0.00) lat (ms,95%): 19.29 err/s: 0.00 reconn/s: 0.00
[ 600s ] thds: 128 tps: 14112.32 qps: 14112.32 (r/w/o: 14112.32/0.00/0.00) lat (ms,95%): 18.28 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            8050009
        write:                           0
        other:                           0
        total:                           8050009
    transactions:                        8050009 (13415.68 per sec.)
    queries:                             8050009 (13415.68 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          600.0432s
    total number of events:              8050009

Latency (ms):
         min:                                    0.33
         avg:                                    9.54
         max:                                  139.05
         95th percentile:                       19.65
         sum:                             76796482.32

Threads fairness:
    events (avg/stddev):           62890.6953/876.89
    execution time (avg/stddev):   599.9725/0.01

```

```shell
# Update index 测试命令
blue@ubuntu:~$ sysbench --config-file=config oltp_update_index --threads=128 --tables=32 --table-size=5000000 run
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 128
Report intermediate results every 10 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 10s ] thds: 128 tps: 8529.17 qps: 8529.17 (r/w/o: 0.00/0.00/8529.17) lat (ms,95%): 28.16 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 128 tps: 7235.73 qps: 7235.73 (r/w/o: 0.00/0.00/7235.73) lat (ms,95%): 32.53 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 128 tps: 8819.03 qps: 8819.03 (r/w/o: 0.00/0.00/8819.03) lat (ms,95%): 27.17 err/s: 0.00 reconn/s: 0.00
...
[ 210s ] thds: 128 tps: 10070.45 qps: 10070.45 (r/w/o: 0.00/0.00/10070.45) lat (ms,95%): 23.52 err/s: 0.00 reconn/s: 0.00
[ 220s ] thds: 128 tps: 10232.50 qps: 10232.50 (r/w/o: 0.00/0.00/10232.50) lat (ms,95%): 23.52 err/s: 0.00 reconn/s: 0.00
...
[ 580s ] thds: 128 tps: 10184.28 qps: 10184.28 (r/w/o: 0.00/0.00/10184.28) lat (ms,95%): 23.52 err/s: 0.00 reconn/s: 0.00
[ 590s ] thds: 128 tps: 7917.15 qps: 7917.15 (r/w/o: 0.00/0.00/7917.15) lat (ms,95%): 31.37 err/s: 0.00 reconn/s: 0.00
[ 600s ] thds: 128 tps: 8447.55 qps: 8447.55 (r/w/o: 0.00/0.00/8447.55) lat (ms,95%): 29.19 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            0
        write:                           0
        other:                           5717468
        total:                           5717468
    transactions:                        5717468 (9528.49 per sec.)
    queries:                             5717468 (9528.49 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          600.0380s
    total number of events:              5717468

Latency (ms):
         min:                                    1.49
         avg:                                   13.43
         max:                                  164.62
         95th percentile:                       25.74
         sum:                             76797298.10

Threads fairness:
    events (avg/stddev):           44667.7188/126.90
    execution time (avg/stddev):   599.9789/0.01


```

```
# Read-only 测试命令
sysbench --config-file=config oltp_read_only --threads=128 --tables=32 --table-size=5000000 run
```

