# Lesson 2

[课程要求](#课程要求)<br>
[项目实施](#项目实施)<br>
- [集群部署](#集群部署)
- [go-ycsb性能测试](#go-ycsb性能测试)
- [go-tpc性能测试](#go-tpc性能测试)
- [关键指标监控截图](#关键指标监控截图)
[结论](#结论)

## 课程要求

使用 sysbench、go-ycsb 和 go-tpc 分别对 TiDB 进行测试并且产出测试报告。测试报告需要包括以下内容：

- 部署环境的机器配置（CPU、内存、磁盘规格型号），拓扑结构（TiDB、TiKV各部署于哪些节点）<br>
- 调整过后的 TiDB 和 TiKV 配置<br>
- 测试输出结果<br>
- 关键指标的监控截图<br>
-- TiDB Query Summary 中的 qps 与 duration
-- TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标
-- TiKV Details 面板中 grpc 的 qps 以及 duration
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

###  sysbench性能测试

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

```shell
# Read-only 测试命令
blue@ubuntu:~$ sysbench --config-file=config oltp_read_only --threads=128 --tables=32 --table-size=5000000 run
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 128
Report intermediate results every 10 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 10s ] thds: 128 tps: 410.60 qps: 6659.12 (r/w/o: 5825.83/0.00/833.29) lat (ms,95%): 419.45 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 128 tps: 423.96 qps: 6795.83 (r/w/o: 5947.62/0.00/848.22) lat (ms,95%): 411.96 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 128 tps: 449.79 qps: 7170.86 (r/w/o: 6271.68/0.00/899.18) lat (ms,95%): 376.49 err/s: 0.00 reconn/s: 0.00
...
[ 210s ] thds: 128 tps: 433.71 qps: 6938.96 (r/w/o: 6072.04/0.00/866.92) lat (ms,95%): 411.96 err/s: 0.00 reconn/s: 0.00
[ 220s ] thds: 128 tps: 448.53 qps: 7173.46 (r/w/o: 6276.40/0.00/897.06) lat (ms,95%): 397.39 err/s: 0.00 reconn/s: 0.00
...
[ 580s ] thds: 128 tps: 455.80 qps: 7277.22 (r/w/o: 6366.71/0.00/910.50) lat (ms,95%): 397.39 err/s: 0.00 reconn/s: 0.00
[ 590s ] thds: 128 tps: 458.78 qps: 7351.43 (r/w/o: 6432.87/0.00/918.57) lat (ms,95%): 390.30 err/s: 0.00 reconn/s: 0.00
[ 600s ] thds: 128 tps: 473.71 qps: 7577.26 (r/w/o: 6629.94/0.00/947.32) lat (ms,95%): 363.18 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            3672900
        write:                           0
        other:                           524700
        total:                           4197600
    transactions:                        262350 (437.02 per sec.)
    queries:                             4197600 (6992.32 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          600.3150s
    total number of events:              262350

Latency (ms):
         min:                                   83.40
         avg:                                  292.83
         max:                                 1130.58
         95th percentile:                      411.96
         sum:                             76822947.66

Threads fairness:
    events (avg/stddev):           2049.6094/30.48
    execution time (avg/stddev):   600.1793/0.09

```

### go-ycsb性能测试

安装go-ycsb

```shell
git clone https://github.com/pingcap/go-ycsb.git
cd go-ycsb
make
```

数据导入

```shell
blue@ubuntu:~/go-ycsb$ ./bin/go-ycsb load mysql -P workloads/workloada -p recordcount=100000 -p mysql.host=192.168.31.104 -p mysql.port=4000 --threads 256
***************** properties *****************
"workload"="core"
"readallfields"="true"
"mysql.host"="192.168.31.104"
"dotransactions"="false"
"recordcount"="100000"
"updateproportion"="0.5"
"mysql.port"="4000"
"requestdistribution"="uniform"
"insertproportion"="0"
"readproportion"="0.5"
"operationcount"="1000"
"scanproportion"="0"
"threadcount"="256"
**********************************************
INSERT - Takes(s): 9.8, Count: 30758, OPS: 3134.5, Avg(us): 82056, Min(us): 29420, Max(us): 261829, 99th(us): 164000, 99.9th(us): 239000, 99.99th(us): 256000
INSERT - Takes(s): 19.8, Count: 56227, OPS: 2837.9, Avg(us): 90370, Min(us): 28231, Max(us): 261829, 99th(us): 182000, 99.9th(us): 231000, 99.99th(us): 252000
INSERT - Takes(s): 29.8, Count: 83659, OPS: 2806.1, Avg(us): 91313, Min(us): 28231, Max(us): 279250, 99th(us): 185000, 99.9th(us): 235000, 99.99th(us): 265000
INSERT - Takes(s): 39.8, Count: 111526, OPS: 2801.2, Avg(us): 91278, Min(us): 28231, Max(us): 314861, 99th(us): 194000, 99.9th(us): 245000, 99.99th(us): 286000
INSERT - Takes(s): 49.8, Count: 135383, OPS: 2717.8, Avg(us): 94253, Min(us): 28231, Max(us): 434617, 99th(us): 207000, 99.9th(us): 331000, 99.99th(us): 402000

```

workloada测试

```shell
blue@ubuntu:~/go-ycsb$ ./bin/go-ycsb run mysql -P workloads/workloada -p operationcount=100000 -p mysql.host=192.168.31.104 -p mysql.port=4000 --threads 256
***************** properties *****************
"operationcount"="100000"
"readallfields"="true"
"mysql.host"="192.168.31.104"
"requestdistribution"="uniform"
"recordcount"="1000"
"scanproportion"="0"
"mysql.port"="4000"
"updateproportion"="0.5"
"dotransactions"="true"
"threadcount"="256"
"readproportion"="0.5"
"insertproportion"="0"
"workload"="core"
**********************************************
READ   - Takes(s): 10.0, Count: 13873, OPS: 1392.6, Avg(us): 48634, Min(us): 1767, Max(us): 284044, 99th(us): 151000, 99.9th(us): 238000, 99.99th(us): 277000
UPDATE - Takes(s): 9.8, Count: 13812, OPS: 1407.1, Avg(us): 134199, Min(us): 20088, Max(us): 730422, 99th(us): 438000, 99.9th(us): 591000, 99.99th(us): 677000
READ   - Takes(s): 20.0, Count: 26997, OPS: 1352.4, Avg(us): 48566, Min(us): 1466, Max(us): 284044, 99th(us): 148000, 99.9th(us): 206000, 99.99th(us): 277000
UPDATE - Takes(s): 19.8, Count: 26927, OPS: 1358.9, Avg(us): 140038, Min(us): 20088, Max(us): 842453, 99th(us): 463000, 99.9th(us): 632000, 99.99th(us): 779000
READ   - Takes(s): 30.0, Count: 39830, OPS: 1329.3, Avg(us): 49118, Min(us): 1466, Max(us): 303513, 99th(us): 150000, 99.9th(us): 208000, 99.99th(us): 285000
UPDATE - Takes(s): 29.8, Count: 39884, OPS: 1337.7, Avg(us): 142670, Min(us): 20086, Max(us): 842453, 99th(us): 468000, 99.9th(us): 632000, 99.99th(us): 772000
Run finished, takes 37.942835203s
READ   - Takes(s): 37.9, Count: 49794, OPS: 1313.8, Avg(us): 48248, Min(us): 1466, Max(us): 303513, 99th(us): 147000, 99.9th(us): 205000, 99.99th(us): 285000
UPDATE - Takes(s): 37.8, Count: 50046, OPS: 1325.5, Avg(us): 140240, Min(us): 9116, Max(us): 842453, 99th(us): 466000, 99.9th(us): 619000, 99.99th(us): 755000

```

workloadb测试

```sh
blue@ubuntu:~/go-ycsb$ ./bin/go-ycsb run mysql -P workloads/workloadb -p operationcount=100000 -p mysql.host=192.168.31.104 -p mysql.port=4000 --threads 256
***************** properties *****************
"readallfields"="true"
"requestdistribution"="uniform"
"scanproportion"="0"
"readproportion"="0.95"
"operationcount"="100000"
"updateproportion"="0.05"
"mysql.port"="4000"
"mysql.host"="192.168.31.104"
"insertproportion"="0"
"dotransactions"="true"
"recordcount"="1000"
"workload"="core"
"threadcount"="256"
**********************************************
READ   - Takes(s): 10.0, Count: 52816, OPS: 5289.5, Avg(us): 42899, Min(us): 2049, Max(us): 213392, 99th(us): 102000, 99.9th(us): 151000, 99.99th(us): 178000
UPDATE - Takes(s): 9.8, Count: 2773, OPS: 282.2, Avg(us): 96737, Min(us): 10614, Max(us): 447270, 99th(us): 235000, 99.9th(us): 373000, 99.99th(us): 448000
Run finished, takes 18.280854903s
READ   - Takes(s): 18.3, Count: 94855, OPS: 5193.5, Avg(us): 43198, Min(us): 1727, Max(us): 213392, 99th(us): 102000, 99.9th(us): 149000, 99.99th(us): 181000
UPDATE - Takes(s): 18.1, Count: 4985, OPS: 275.3, Avg(us): 95489, Min(us): 9911, Max(us): 447270, 99th(us): 229000, 99.9th(us): 373000, 99.99th(us): 448000
```

workloadc测试

```shell
blue@ubuntu:~/go-ycsb$ ./bin/go-ycsb run mysql -P workloads/workloadc -p operationcount=100000 -p mysql.host=192.168.31.104 -p mysql.port=4000 --threads 256
***************** properties *****************
"scanproportion"="0"
"dotransactions"="true"
"readallfields"="true"
"recordcount"="1000"
"readproportion"="1"
"threadcount"="256"
"updateproportion"="0"
"operationcount"="100000"
"requestdistribution"="uniform"
"insertproportion"="0"
"mysql.host"="192.168.31.104"
"mysql.port"="4000"
"workload"="core"
**********************************************
READ   - Takes(s): 9.9, Count: 76426, OPS: 7749.9, Avg(us): 33248, Min(us): 7092, Max(us): 155238, 99th(us): 75000, 99.9th(us): 117000, 99.99th(us): 135000
Run finished, takes 13.25299501s
READ   - Takes(s): 13.1, Count: 99840, OPS: 7613.1, Avg(us): 33515, Min(us): 2077, Max(us): 155238, 99th(us): 75000, 99.9th(us): 114000, 99.99th(us): 134000

```

workloadd测试

```shell
blue@ubuntu:~/go-ycsb$ ./bin/go-ycsb run mysql -P workloads/workloadd -p operationcount=100000 -p mysql.host=192.168.31.104 -p mysql.port=4000 --threads 256
***************** properties *****************
"mysql.host"="192.168.31.104"
"mysql.port"="4000"
"dotransactions"="true"
"workload"="core"
"requestdistribution"="latest"
"insertproportion"="0.05"
"recordcount"="1000"
"operationcount"="100000"
"readproportion"="0.95"
"updateproportion"="0"
"scanproportion"="0"
"readallfields"="true"
"threadcount"="256"
**********************************************
INSERT - Takes(s): 9.9, Count: 3710, OPS: 375.5, Avg(us): 24447, Min(us): 938, Max(us): 207589, 99th(us): 71000, 99.9th(us): 170000, 99.99th(us): 208000
READ   - Takes(s): 10.0, Count: 71722, OPS: 7193.5, Avg(us): 34192, Min(us): 3543, Max(us): 277008, 99th(us): 94000, 99.9th(us): 200000, 99.99th(us): 266000
Run finished, takes 13.068844361s
INSERT - Takes(s): 12.9, Count: 4988, OPS: 385.2, Avg(us): 23910, Min(us): 938, Max(us): 207589, 99th(us): 67000, 99.9th(us): 170000, 99.99th(us): 208000
READ   - Takes(s): 13.0, Count: 94852, OPS: 7275.7, Avg(us): 33504, Min(us): 1979, Max(us): 277008, 99th(us): 87000, 99.9th(us): 195000, 99.99th(us): 250000
```

workloade测试

```shell
blue@ubuntu:~/go-ycsb$ ./bin/go-ycsb run mysql -P workloads/workloade -p operationcount=100000 -p mysql.host=192.168.31.104 -p mysql.port=4000 --threads 256
***************** properties *****************
"operationcount"="100000"
"scanproportion"="0.95"
"updateproportion"="0"
"readallfields"="true"
"threadcount"="256"
"insertproportion"="0.05"
"dotransactions"="true"
"mysql.host"="192.168.31.104"
"scanlengthdistribution"="uniform"
"recordcount"="1000"
"readproportion"="0"
"workload"="core"
"maxscanlength"="1"
"mysql.port"="4000"
"requestdistribution"="uniform"
**********************************************
INSERT - Takes(s): 9.8, Count: 1058, OPS: 107.5, Avg(us): 89405, Min(us): 28548, Max(us): 174098, 99th(us): 154000, 99.9th(us): 174000, 99.99th(us): 175000
SCAN   - Takes(s): 9.9, Count: 19211, OPS: 1946.5, Avg(us): 126729, Min(us): 43904, Max(us): 290824, 99th(us): 209000, 99.9th(us): 253000, 99.99th(us): 285000
INSERT - Takes(s): 19.8, Count: 2135, OPS: 107.6, Avg(us): 87678, Min(us): 28244, Max(us): 177860, 99th(us): 155000, 99.9th(us): 177000, 99.99th(us): 178000
SCAN   - Takes(s): 19.9, Count: 39076, OPS: 1966.6, Avg(us): 125418, Min(us): 43810, Max(us): 319773, 99th(us): 207000, 99.9th(us): 247000, 99.99th(us): 285000
INSERT - Takes(s): 29.8, Count: 3180, OPS: 106.5, Avg(us): 86781, Min(us): 28244, Max(us): 177860, 99th(us): 155000, 99.9th(us): 177000, 99.99th(us): 178000
SCAN   - Takes(s): 29.9, Count: 59160, OPS: 1980.6, Avg(us): 124591, Min(us): 43810, Max(us): 319773, 99th(us): 203000, 99.9th(us): 240000, 99.99th(us): 284000
INSERT - Takes(s): 39.8, Count: 4189, OPS: 105.1, Avg(us): 86937, Min(us): 28244, Max(us): 229208, 99th(us): 158000, 99.9th(us): 212000, 99.99th(us): 230000
SCAN   - Takes(s): 39.9, Count: 78753, OPS: 1975.3, Avg(us): 125021, Min(us): 43810, Max(us): 319773, 99th(us): 211000, 99.9th(us): 267000, 99.99th(us): 298000
Run finished, takes 48.223547492s
INSERT - Takes(s): 48.1, Count: 5059, OPS: 105.2, Avg(us): 86975, Min(us): 5588, Max(us): 229208, 99th(us): 158000, 99.9th(us): 205000, 99.99th(us): 230000
SCAN   - Takes(s): 48.1, Count: 94781, OPS: 1970.8, Avg(us): 124605, Min(us): 3378, Max(us): 319773, 99th(us): 207000, 99.9th(us): 264000, 99.99th(us): 296000

```

workloadf测试

```shell
blue@ubuntu:~/go-ycsb$ ./bin/go-ycsb run mysql -P workloads/workloadf -p operationcount=100000 -p mysql.host=192.168.31.104 -p mysql.port=4000 --threads 256
***************** properties *****************
"readallfields"="true"
"mysql.port"="4000"
"readproportion"="0.5"
"dotransactions"="true"
"workload"="core"
"operationcount"="100000"
"recordcount"="1000"
"mysql.host"="192.168.31.104"
"readmodifywriteproportion"="0.5"
"requestdistribution"="uniform"
"insertproportion"="0"
"updateproportion"="0"
"scanproportion"="0"
"threadcount"="256"
**********************************************
READ   - Takes(s): 9.9, Count: 16924, OPS: 1710.5, Avg(us): 65280, Min(us): 4324, Max(us): 306826, 99th(us): 172000, 99.9th(us): 231000, 99.99th(us): 275000
READ_MODIFY_WRITE - Takes(s): 9.8, Count: 8346, OPS: 849.7, Avg(us): 235957, Min(us): 58817, Max(us): 937680, 99th(us): 570000, 99.9th(us): 743000, 99.99th(us): 938000
UPDATE - Takes(s): 9.8, Count: 8346, OPS: 849.7, Avg(us): 170105, Min(us): 44414, Max(us): 841132, 99th(us): 497000, 99.9th(us): 652000, 99.99th(us): 842000
READ   - Takes(s): 19.9, Count: 35481, OPS: 1783.5, Avg(us): 61725, Min(us): 4324, Max(us): 306826, 99th(us): 163000, 99.9th(us): 219000, 99.99th(us): 272000
READ_MODIFY_WRITE - Takes(s): 19.8, Count: 17625, OPS: 889.1, Avg(us): 225755, Min(us): 58817, Max(us): 965203, 99th(us): 562000, 99.9th(us): 726000, 99.99th(us): 938000
UPDATE - Takes(s): 19.8, Count: 17625, OPS: 889.1, Avg(us): 164007, Min(us): 44414, Max(us): 894363, 99th(us): 490000, 99.9th(us): 641000, 99.99th(us): 815000
READ   - Takes(s): 29.9, Count: 55262, OPS: 1848.6, Avg(us): 59603, Min(us): 3755, Max(us): 377808, 99th(us): 157000, 99.9th(us): 217000, 99.99th(us): 275000
READ_MODIFY_WRITE - Takes(s): 29.8, Count: 27460, OPS: 920.8, Avg(us): 217875, Min(us): 29591, Max(us): 965203, 99th(us): 553000, 99.9th(us): 715000, 99.99th(us): 869000
UPDATE - Takes(s): 29.8, Count: 27460, OPS: 920.8, Avg(us): 158386, Min(us): 18400, Max(us): 894363, 99th(us): 483000, 99.9th(us): 637000, 99.99th(us): 815000
READ   - Takes(s): 39.9, Count: 75904, OPS: 1902.6, Avg(us): 57923, Min(us): 3755, Max(us): 377808, 99th(us): 151000, 99.9th(us): 211000, 99.99th(us): 272000
READ_MODIFY_WRITE - Takes(s): 39.8, Count: 37711, OPS: 947.0, Avg(us): 211970, Min(us): 29591, Max(us): 965203, 99th(us): 543000, 99.9th(us): 699000, 99.99th(us): 869000
UPDATE - Takes(s): 39.8, Count: 37711, OPS: 947.0, Avg(us): 154040, Min(us): 18400, Max(us): 894363, 99th(us): 475000, 99.9th(us): 630000, 99.99th(us): 772000
READ   - Takes(s): 49.9, Count: 95552, OPS: 1915.1, Avg(us): 57274, Min(us): 3755, Max(us): 377808, 99th(us): 149000, 99.9th(us): 209000, 99.99th(us): 275000
READ_MODIFY_WRITE - Takes(s): 49.8, Count: 47619, OPS: 955.7, Avg(us): 210180, Min(us): 29591, Max(us): 965203, 99th(us): 539000, 99.9th(us): 705000, 99.99th(us): 886000
UPDATE - Takes(s): 49.8, Count: 47619, OPS: 955.7, Avg(us): 152808, Min(us): 18400, Max(us): 894363, 99th(us): 472000, 99.9th(us): 630000, 99.99th(us): 815000
Run finished, takes 52.90155114s
READ   - Takes(s): 52.8, Count: 99840, OPS: 1891.5, Avg(us): 56401, Min(us): 1522, Max(us): 377808, 99th(us): 148000, 99.9th(us): 209000, 99.99th(us): 275000
READ_MODIFY_WRITE - Takes(s): 52.7, Count: 49883, OPS: 946.3, Avg(us): 207153, Min(us): 8590, Max(us): 965203, 99th(us): 537000, 99.9th(us): 700000, 99.99th(us): 886000
UPDATE - Takes(s): 52.7, Count: 49883, OPS: 946.3, Avg(us): 150688, Min(us): 6083, Max(us): 894363, 99th(us): 470000, 99.9th(us): 628000, 99.99th(us): 815000

```



### go-tpc性能测试

安装

```shell
git clone https://github.com/pingcap/go-tpc.git
make build
```

go-tpc测试，因为硬盘容量问题，只导入100仓库

```shell
./bin/go-tpc tpcc -H 192.168.31.104 -P 4000 -D tpcc --warehouses 100 prepare
```

运行测试

```shell
./bin/go-tpc tpcc -H 192.168.31.104 -P 4000 -D tpcc --warehouses 100 run
```

TPC-H测试，导入

```
./bin/go-tpc tpch prepare -H 192.168.31.104 -P 4000 -D tpch --sf 50 --analyze
```

测试

```
./bin/go-tpc tpch run -H 192.168.31.104 -P 4000 -D tpch --sf 50
```

### 关键指标监控截图
TiDB QPS<br>
<img src="Img/Lesson 2/tidb qps.png" style="zoom:75%;" /><br>
TiDB Duration<br>
<img src="Img/Lesson 2/tidb duration.png" style="zoom:75%;" /><br>
TiKV Cluster CPU<br>
<img src="Img/Lesson 2/tikv cluster cpu.png" style="zoom:75%;" /><br>
TiKV Cluster QPS<br>
<img src="Img/Lesson 2/tikv cluster qps.png" style="zoom:75%;" /><br>
grpc QPS<br>
<img src="Img/Lesson 2/grpc qps.png" style="zoom:75%;" /><br>
grpc Duration<br>
<img src="Img/Lesson 2/grpc duration.png" style="zoom:75%;" /><br>

## 结论
