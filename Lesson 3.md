# Lesson 3

使用上一节可以讲的 sysbench、go-ycsb 或者 go-tpc 对 TiDB 进行压力测试，
然后对 TiDB 或 TiKV 的 CPU 、内存或 IO 进行 profile，寻找潜在可以优化的地
方并提 enhance 类型的 issue 描述。

## 环境配置

| 编号 |  CPU  | 内存 |   储存   |      OS      | 组件 |
| :--: | :--: | :--: | :------: | :----------: | :--: |
|  1   |  2核  |  2G  | 虚拟硬盘nvme | Ubuntu 20.04 | TiDB |
|  2   |  2核  |  2G  | 虚拟硬盘nvme | Ubuntu 20.04 | Tikv |
|  2   |  2核  |  2G  | 虚拟硬盘nvme | Ubuntu 20.04 | Tikv |
|  3   |  2核  |  2G  | 虚拟硬盘nvme | Ubuntu 20.04 | Tikv |
|  3   |  2核  |  2G  | 虚拟硬盘nvme | Ubuntu 20.04 |  PD  |
|  4   |   2核  |  2G  | 虚拟硬盘nvme | Ubuntu 20.04 |  PD  |
|  4   |   2核  |  2G  | 虚拟硬盘nvme | Ubuntu 20.04 |  PD  |

## 测试
运行sysbench测试，导入32个表，数据100000条，运行Point select 测试测试，获取TiDB的火焰图
```
SQL statistics:
    queries performed:
        read:                            5035877
        write:                           0
        other:                           0
        total:                           5035877
    transactions:                        5035877 (8376.15 per sec.)
    queries:                             5035877 (8376.15 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          601.2144s
    total number of events:              5035877

Latency (ms):
         min:                                    0.36
         avg:                                   30.51
         max:                                  809.44
         95th percentile:                       69.29
         sum:                            153633736.59

Threads fairness:
    events (avg/stddev):           19671.3945/50.06
    execution time (avg/stddev):   600.1318/0.18
```
火焰图
<img src="Img/Lesson 3/img1.png" style="zoom:75%;" /><br>
经过具体分析，考虑executor.(*ExecStmt).FinishExecuteStmt 和server.(*clientConn).flush存在瓶颈
<img src="Img/Lesson 3/img4.png" style="zoom:75%;" /><br>
