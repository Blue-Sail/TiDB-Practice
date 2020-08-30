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
