---
title: TiDB-Binlog cluster 版本用户文档
category: advanced
---

> cluster 版 TiDB-Binlog 尚未发布正式版，本文档用于测试环境部署。

## TiDB-Binlog 简介

TiDB-Binlog 是一个用于收集 TiDB 的 Binlog，并提供实时备份和同步功能的商业工具。

TiDB-Binlog 支持以下功能场景:

* **数据同步**：同步 TiDB 集群数据到其他数据库
* **实时备份和恢复**：备份 TiDB 集群数据，同时可以用于 TiDB 集群故障时恢复

### TiDB-Binlog 架构

首先介绍 TiDB-Binlog 的整体架构。

![TiDB-Binlog 架构](../media/tidb_binlog_cluster_architecture.png)

TiDB-Binlog 集群主要分为 Pump 和 Drainer 两个组件：

#### Pump

Pump 用于实时记录 TiDB 产生的 Binlog，并将 Binlog 按照事务的提交时间进行排序，再提供给 Drainer 进行消费。

#### Drainer

Drainer 从各个 Pump 中收集 Binlog 进行归并，再将 Binlog 转化成 SQL 或者指定格式的数据，最终同步到下游。

### 主要特性

* 多个 Pump 形成一个集群，可以水平扩容；
* TiDB 通过内置的 Pump Client 将 Binlog 分发到各个 Pump；
* Pump 负责存储 Binlog，并将 Binlog 按顺序提供给 Drainer；
* Drainer 负责读取各个 Pump 的 Binlog，归并排序后发送到下游。


## TiDB-Binlog 部署

#### 注意

* 在运行 TiDB 时，需要保证至少一个 Pump 正常运行。
* 通过给 TiDB 增加启动参数 enable-binlog 来开启 Binlog。
* Drainer 不支持对 ignore schemas（在过滤列表中的 schemas）的 table 进行 rename DDL 操作。
* 在已有的 TiDB 集群中启动 Drainer，一般需要全量备份并且获取 savepoint，然后导入全量备份，最后启动 Drainer 从 savepoint 开始同步增量数据。
* Drainer 支持将 Binlog 同步到 MySQL/TiDB/Kafka/ 文件。如果需要将 Binlog 同步到其他类型的目的地中，可以设置 Drainer 将 Binlog 同步到 Kafka，再读取 Kafka 中的数据进行自定义处理，参考 [tidb-binlog driver](https://github.com/pingcap/tidb-tools/tree/master/tidb_binlog/driver)。
* Pump/Drainer 的状态需要区分已暂停（paused）和下线（offline），Ctrl+C 或者 kill 进程，Pump 和 Drainer 的状态都将变为 paused。暂停状态的 Pump 不需要将已保存的 Binlog 数据全部发送到 Drainer；如果需要较长时间退出 Pump（或不再使用该 Pump），需要使用 binlogctl 工具来下线 Pump。Drainer 同理。

#### 服务器要求

Pump && Drainer 支持部署和运行在 Intel x86-64 架构的 64 位通用硬件服务器平台上。对于开发，测试以及生产环境的服务器硬件配置有以下要求和建议：

| 服务     | 部署数量       | CPU   | 磁盘          | 内存   |
| -------- | -------- | --------| --------------- | ------ |
| Pump | 3 | 8核+   | SSD, 200 GB+ | 16G |
| Drainer | 1 | 8核+ | SAS, 100 GB+（如果输出为本地文件，则使用 SSD，并增加磁盘大小） | 16G |

### 使用 tidb-ansible 部署 TiDB-Binlog

#### 下载 tidb-ansible

以 tidb 用户登录中控机并进入 `/home/tidb` 目录，使用以下命令下载 tidb-ansible `new-tidb-binlog` 分支，默认的文件夹名称为 tidb-ansible。

```
$ git clone -b new-tidb-binlog https://github.com/pingcap/tidb-ansible.git
```

#### 部署 Pump

A. 修改 tidb-ansible/inventory.ini 文件

1. 设置 `enable_binlog = True`，表示 TiDB 集群开启 binlog。

```
## binlog trigger
enable_binlog = True
```

2. 为 `pump_servers` 主机组添加部署机器 IP。

```
## Binlog Part
[pump_servers]
172.16.10.72
172.16.10.73
172.16.10.74
```

默认 Pump 保留 5 天数据，如需修改可修改 `tidb-ansible/conf/pump.yml` 文件中 `gc` 变量值，并取消注释，如修改为 7。

```
global:
  # a integer value to control expiry date of the binlog data, indicates for how long (in days) the binlog data would be stored. 
  # must bigger than 0
  gc: 7
```

请确保部署目录有足够空间存储 binlog，详见：[部署目录调整](../op-guide/ansible-deployment.md#部署目录调整)，也可为 Pump 设置单独的部署目录。

```
## Binlog Part
[pump_servers]
pump1 ansible_host=172.16.10.72 deploy_dir=/data1/pump
pump2 ansible_host=172.16.10.73 deploy_dir=/data1/pump
pump3 ansible_host=172.16.10.74 deploy_dir=/data1/pump
```

B. 部署并启动 TiDB 集群

使用 Ansible 部署 TiDB 集群的具体方法参考 [TiDB Ansible 部署方案](../op-guide/ansible-deployment.md)，开启 binlog 后默认会部署和启动 Pump 服务。

C. 查看 Pump 服务状态

使用 binlogctl 查看 Pump 服务状态，pd-urls 参数请替换为集群 PD 地址，结果 State 为 online 表示 Pump 启动成功。

```
$ cd /home/tidb/tidb-ansible
$ resources/bin/binlogctl -pd-urls=http://172.16.10.72:2379 -cmd pumps
2018/09/21 16:45:54 nodes.go:46: [info] pump: &{NodeID:ip-172-16-10-72:8250 Addr:172.16.10.72:8250 State:online IsAlive:false Score:0 Label:<nil> MaxCommitTS:0 UpdateTS:403051525690884099}
2018/09/21 16:45:54 nodes.go:46: [info] pump: &{NodeID:ip-172-16-10-73:8250 Addr:172.16.10.73:8250 State:online IsAlive:false Score:0 Label:<nil> MaxCommitTS:0 UpdateTS:403051525703991299}
2018/09/21 16:45:54 nodes.go:46: [info] pump: &{NodeID:ip-172-16-10-74:8250 Addr:172.16.10.74:8250 State:online IsAlive:false Score:0 Label:<nil> MaxCommitTS:0 UpdateTS:403051525717360643}
```

#### 部署 Drainer

A. 获取 initial_commit_ts 

使用 binlogctl 工具生成 Drainer 初次启动所需的 tso 信息，命令：

```
$ cd /home/tidb/tidb-ansible
$ resources/bin/binlogctl -pd-urls=http://127.0.0.1:2379 -cmd generate_meta
INFO[0000] [pd] create pd client with endpoints [http://192.168.199.118:32379]
INFO[0000] [pd] leader switches to: http://192.168.199.118:32379, previous:
INFO[0000] [pd] init cluster id 6569368151110378289
2018/06/21 11:24:47 meta.go:117: [info] meta: &{CommitTS:400962745252184065}
```

该命令会生成一个文件 `{data-dir}/savepoint`，该文件中包含 tso，用该 tso 作为 Drainer 初次启动使用的 initial-commit-ts 参数的值.

B. 全量数据的备份与恢复

如果下游为 MySQL/TiDB， 需要保证数据的完整性，在 Drainer 启动前（Pump 运行后十分钟左右）进行数据的全量备份和恢复。

推荐使用 mydumper 备份 TiDB 的全量数据，再使用 loader 将备份数据导入到下游。具体使用方法参考：[备份与恢复](https://github.com/pingcap/docs-cn/blob/master/op-guide/backup-restore.md)。

C. 修改 `tidb-ansible/inventory.ini` 文件

为 `drainer_servers` 主机组添加部署机器 IP，initial_commit_ts 请设置为获取的 initial_commit_ts，仅用于 Drainer 第一次启动。

1. 以下游为 MySQL 为例，别名为 `drainer_mysql`。

```
[drainer_servers]
drainer_mysql ansible_host=172.16.10.71 initial_commit_ts="402899541671542785"
```

2. 以下游为 pb 为例，别名为 `drainer_pb`。

```
[drainer_servers]
drainer_pb ansible_host=172.16.10.71 initial_commit_ts="402899541671542785"
```

D. 修改配置文件

1. 以下游为 MySQL 为例

```
$ cd /home/tidb/tidb-ansible/conf
$ cp drainer.toml drainer_mysql_drainer.toml
$ vi drainer_mysql_drainer.toml
```

db-type 设置为 "mysql"， 配置下游 MySQL 信息。

```
# downstream storage, equal to --dest-db-type
# valid values are "mysql", "pb", "tidb", "flash", "kafka"
db-type = "mysql"

# the downstream MySQL protocol database
[syncer.to]
host = "172.16.10.72"
user = "root"
password = "123456"
port = 3306
# Time and size limits for flash batch write
# time-limit = "30s"
# size-limit = "100000"
```

2. 以下游为 pb 为例

```
$ cd /home/tidb/tidb-ansible/conf
$ cp drainer.toml drainer_pb_drainer.toml
$ vi drainer_pb_drainer.toml
```

db-type 设置为 "pb"。

```
# downstream storage, equal to --dest-db-type
# valid values are "mysql", "pb", "tidb", "flash", "kafka"
db-type = "pb"

# Uncomment this if you want to use pb or sql as db-type.
# Compress compresses output file, like pb and sql file. Now it supports "gzip" algorithm only. 
# Values can be "gzip". Leave it empty to disable compression. 
[syncer.to]
compression = ""
# default data directory: "{{ deploy_dir }}/data.drainer"
# dir = "data.drainer"
```

E. 部署 Drainer

```
$ ansible-playbook deploy_drainer.yml
```

F. 启动 Drainer

```
$ ansible-playbook start_drainer.yml
```

### 使用 Binary 部署 TiDB-Binlog

#### 下载官方 Binary

```bash
TiDB（Pump Client）
wget https://download.pingcap.org/tidb-v2.0.7-binlog-dev-linux-amd64.tar.gz
wget https://download.pingcap.org/tidb-v2.0.7-binlog-dev-linux-amd64.sha256

# 检查文件完整性，返回 ok 则正确
sha256sum -c tidb-v2.0.7-binlog-dev-linux-amd64.sha256

Pump && Drainer
wget https://download.pingcap.org/tidb-binlog-new-linux-amd64.tar.gz
wget https://download.pingcap.org/tidb-binlog-new-linux-amd64.sha256

# 检查文件完整性，返回 ok 则正确
sha256sum -c tidb-binlog-new-linux-amd64.sha256
```

#### 使用样例

假设有三个 PD，一个 TiDB，另外有两台机器用于部署 Pump，一台机器用于部署 Drainer。各个节点信息如下：

```
TiDB="192.168.0.10"
PD1="192.168.0.16"
PD2="192.168.0.15"
PD3="192.168.0.14"
Pump="192.168.0.11"
Pump="192.168.0.12"
Drainer="192.168.0.13"
```

下面以此为例，说明 Pump/Drainer 的使用。

A. 使用 binary 部署 Pump

1. Pump 命令行参数说明（以在 “192.168.0.11” 上部署为例）

    ```
    Usage of Pump:
    -L string
        日志输出信息等级设置：debug，info，warn，error，fatal (默认 "info")
    -V
        打印版本信息
    -addr string
        Pump 提供服务的 RPC 地址(-addr="192.168.0.11:8250")
    -advertise-addr string
        Pump 对外提供服务的 RPC 地址(-advertise-addr="192.168.0.11:8250")
    -config string
        配置文件路径，如果你指定了配置文件，Pump 会首先读取配置文件的配置；
        如果对应的配置在命令行参数里面也存在，Pump 就会使用命令行参数的配置来覆盖配置文件里面的。
    -data-dir string
        Pump 数据存储位置路径
    -enable-tolerant
        开启 tolerant 后，如果 binlog 写入失败，Pump 不会报错（默认开启）
    -gc int
        Pump 只保留多少天以内的数据 (默认 7)
    -heartbeat-interval int
        Pump 向 PD 发送心跳间隔 (单位 秒)
    -log-file string
        log 文件路径
    -log-rotate string
        log 文件切换频率，hour/day
    -metrics-addr string
        Prometheus pushgateway 地址，不设置则禁止上报监控信息
    -metrics-interval int
        监控信息上报频率 (默认 15，单位 秒)
    -pd-urls string
        PD 集群节点的地址 (-pd-urls="http://192.168.0.16:2379,http://192.168.0.15:2379,http://192.168.0.14:2379")
    ```

2. Pump 配置文件（以在 “192.168.0.11” 上部署为例）

    ```
    # Pump Configuration.

    # Pump 绑定的地址
    addr = "192.168.0.11:8250"

    # Pump 对外提供服务的地址
    advertise-addr = “192.168.0.11:8250"

    # Pump 只保留多少天以内的数据 (默认 7)
    gc = 7

    # Pump 数据存储位置路径
    data-dir = "data.pump"

    # Pump 向 PD 发送心跳的间隔 (单位 秒)
    heartbeat-interval = 2
    
    # PD 集群节点的地址
    pd-urls = "http://192.168.0.16:2379,http://192.168.0.15:2379,http://192.168.0.14:2379"
    ```

3. 启动示例

    ```
    ./bin/pump -config pump.toml
    ```
  
    如果命令行参数与配置文件中的参数重合，则使用命令行设置的参数的值。

B. 使用 binary 部署 Drainer

1. Drainer 命令行参数说明（以在 “192.168.0.13” 上部署为例）

    ```
    Usage of Drainer:
    -L string
        日志输出信息等级设置：debug，info，warn，error，fatal (默认 "info")
    -V
        打印版本信息
    -addr string
        Drainer 提供服务的地址(-addr="192.168.0.13:8249")
    -c int
        同步下游的并发数，该值设置越高同步的吞吐性能越好 (default 1)
    -config string
        配置文件路径，Drainer 会首先读取配置文件的配置；
        如果对应的配置在命令行参数里面也存在，Drainer 就会使用命令行参数的配置来覆盖配置文件里面的
    -data-dir string
        Drainer 数据存储位置路径 (默认 "data.drainer")
    -dest-db-type string
        Drainer 下游服务类型 (默认为 mysql，支持 tidb、kafka、pb)
    -detect-interval int
        向 PD 查询在线 Pump 的时间间隔 (默认 10，单位 秒)
    -disable-detect
       是否禁用冲突监测
    -disable-dispatch
        是否禁用拆分单个 binlog 的 sqls 的功能，如果设置为 true，则按照每个 binlog
        顺序依次还原成单个事务进行同步（下游服务类型为 mysql，该项设置为 False）
    -ignore-schemas string
        db 过滤列表 (默认 "INFORMATION_SCHEMA,PERFORMANCE_SCHEMA,mysql,test")，
        不支持对 ignore schemas 的 table 进行 rename DDL 操作
    -initial-commit-ts (默认为 0)
        如果 Drainer 没有相关的断点信息，可以通过该项来设置相关的断点信息
    -log-file string
        log 文件路径
    -log-rotate string
        log 文件切换频率，hour/day
    -metrics-addr string
        Prometheus pushgateway 地址，不设置则禁止上报监控信息
    -metrics-interval int
        监控信息上报频率（默认 15，单位 秒）
    -pd-urls string
        PD 集群节点的地址 (-pd-urls="http://192.168.0.16:2379,http://192.168.0.15:2379,http://192.168.0.14:2379")
    -safe-mode
        是否开启安全模式（将 update 语句拆分为 delete + replace 语句）
    -txn-batch int
        输出到下游数据库一个事务的 SQL 数量（默认 1）
    ```

2. Drainer 配置文件（以在 “192.168.0.13” 上部署为例）

    ```
    # Drainer Configuration.
 
    # Drainer 提供服务的地址("192.168.0.13:8249")
    addr = "192.168.0.13:8249"
 
    # 向 PD 查询在线 Pump 的时间间隔 (默认 10，单位 秒)
    detect-interval = 10
 
    # Drainer 数据存储位置路径 (默认 "data.drainer")
    data-dir = "data.drainer"
 
    # PD 集群节点的地址
    pd-urls = "http://192.168.0.16:2379,http://192.168.0.15:2379,http://192.168.0.14:2379"
 
    # log 文件路径
    log-file = "drainer.log"
 
    # Syncer Configuration.
    [syncer]
 
    # db 过滤列表 (默认 "INFORMATION_SCHEMA,PERFORMANCE_SCHEMA,mysql,test"),
    # 不支持对 ignore schemas 的 table 进行 rename DDL 操作
    ignore-schemas = "INFORMATION_SCHEMA,PERFORMANCE_SCHEMA,mysql"
 
    # 输出到下游数据库一个事务的 SQL 数量 (default 1)
    txn-batch = 1

    # 同步下游的并发数，该值设置越高同步的吞吐性能越好 (default 1)
    worker-count = 1
 
    # 是否禁用拆分单个 binlog 的 sqls 的功能，如果设置为 true，则按照每个 binlog
    # 顺序依次还原成单个事务进行同步（下游服务类型为 mysql, 该项设置为 False）
    disable-dispatch = false
 
    # Drainer 下游服务类型（默认为 mysql）
    # 参数有效值为 "mysql"，"pb"
    db-type = "mysql"
 
    # replicate-do-db prioritizes over replicate-do-table when they have the same db name
    # and we support regex expressions,
    # 以 '~' 开始声明使用正则表达式
 
    #replicate-do-db = ["~^b.*","s1"]
 
    #[[syncer.replicate-do-table]]
    #db-name ="test"
    #tbl-name = "log"
 
    #[[syncer.replicate-do-table]]
    #db-name ="test"
    #tbl-name = "~^a.*"
 
    # db-type 设置为 mysql 时，下游数据库服务器参数
    [syncer.to]
    host = "192.168.0.13"
    user = "root"
    password = ""
    port = 3306
 
    # db-type 设置为 pb 时，存放 binlog 文件的目录
    # [syncer.to]
    # dir = "data.drainer"
 
    # db-type 设置为 kafka 时，kafka 相关配置
    #[syncer.to]
    # zookeeper-addrs = "127.0.0.1:2181"
    # kafka-addrs = "127.0.0.1:9092"
    # kafka-version = "0.8.2.0"
    ```

3. 启动示例  
    注意：如果下游为 MySQL/TiDB，为了保证数据的完整性，在 Drainer 初次启动前需要获取 initial-commit-ts 的值，并进行全量数据的备份与恢复。该部分在 [部署 Drainer](./tidb-binlog-cluster.md#部署-drainer) 一节中已经介绍了，就不再赘述。
    
    初次启动时使用参数 `initial-commit-ts`， 命令如下：

    ```bash
    ./bin/drainer -config drainer.toml -initial-commit-ts {initial-commit-ts}
    ```

    如果命令行参数与配置文件中的参数重合，则使用命令行设置的参数的值。

## TiDB-Binlog 运维

### Pump/Drainer 状态

首先介绍一下 Pump/Drainer 中状态的定义：

* online：正常运行中；
* pausing：暂停中，当使用 kill 或者 Ctrl+C 退出进程时，都将处于该状态；
* paused：已暂停，处于该状态时 Pump 不接受写 binlog 的请求，也不继续为 Drainer 提供 binlog，Drainer 不再往下游同步数据。当 Pump/Drainer 安全退出了所有的线程后，将自己的状态切换为 paused，再退出进程；
* closing：下线中，使用 binlogctl 控制 Pump/Drainer 下线，在进程退出前都处于该状态。下线时 Pump 不再接受写 binlog 的请求，等待所有的 binlog 数据被 Drainer 消费完；
* offline：已下线，当 Pump 已经将已保存的所有 binlog 数据全部发送给 Drainer 后，该 Pump 将状态切换为 offline；Drainer 只需要等待各个线程退出后即可切换状态为 offline。

注意：

* 当暂停 Pump/Drainer 时，数据同步会中断；
* Pump 在下线时需要确认自己的数据被所有的非 offline 状态的 Drainer 消费了，所以在下线 Pump 时需要确保所有的 Drainer 都是处于 online 状态，否则 Pump 无法正常下线；
* Pump 保存的 binlog 数据只有在被所有非 offline 状态的 Drainer 消费的情况下才会 gc；
* 不要轻易下线 Drainer，只有在永久不需要使用该 Drainer 的情况下才需要下线 Drainer。

关于 Pump/Drainer 暂停、下线、状态查询、状态修改等具体的操作方法，参考如下 binlogctl 工具的使用方法介绍。

### binlogctl 工具

binlogctl 是一个 TiDB-Binlog 配套的运维工具，具有如下功能：

* 获取当前 ts
* 查看 Pump/Drainer 状态
* 修改 Pump/Drainer 状态
* 暂停/下线 Pump/Drainer

使用 binlogctl 的场景：

* 第一次运行 Drainer，需要获取当前的 ts
* Pump/Drainer 异常退出，状态没有更新，对业务造成影响，可以直接使用该工具修改状态
* 同步出现故障/检查运行情况，需要查看 Pump/Drainer 的状态
* 维护集群，需要暂停/下线 Pump/Drainer

binlogctl 下载链接：

```bash
wget https://download.pingcap.org/binlogctl-new-linux-amd64.tar.gz
wget https://download.pingcap.org/binlogctl-new-linux-amd64.sha256

# 检查文件完整性，返回 ok 则正确
sha256sum -c tidb-binlog-new-linux-amd64.sha256
```

binlogctl 使用说明：

命令行参数：

```
Usage of binlogctl:
-V	
输出 binlogctl 的版本信息
-cmd string
    命令模式，包括 "generate_meta", "pumps", "drainers", "update-pump" ,"update-drainer ", "pause-pump", "pause-drainer", "offline-pump", "offline-drainer"
-data-dir string
    保存 Drainer 的 checkpoint 的文件的路径 (默认 "binlog_position")
-node-id string
    Pump/Drainer 的 id
-pd-urls string
    pd 的地址，如果有多个用"," 连接 (默认 "http://127.0.0.1:2379")
-ssl-ca string
    SSL CAs 文件的路径
-ssl-cert string
        PEM 格式的 X509 认证文件的路径
-ssl-key string
        PEM 格式的 X509 key 文件的路径
-time-zone string
    如果设置时区，在 "generate_meta" 模式下会打印出获取到的 tso 对应的时间。例如"Asia/Shanghai" 为 CST 时区，"Local" 为本地时区
```

命令示例：

1. 查询所有的 Pump/Drainer 的状态：

    ```
    bin/binlogctl -pd-urls=http://127.0.0.1:2379 -cmd pumps/drainers

    2018/06/21 11:24:10 nodes.go:53: [info] pump: &{NodeID:ip-192-168-199-118:8250 Host:127.0.0.1:8250 IsAlive:true IsOffline:false LatestFilePos:{Suffix:0 Offset:15320} LatestKafkaPos:{Suffix:0 Offset:382} OfflineTS:0}
    ```
 
2. 修改 Pump/Drainer 的状态

    ```
    Pump/Drainer 的状态可以为：online, pausing, paused, closing and offline. 

    bin/binlogctl -pd-urls=http://127.0.0.1:2379 -cmd update-pump/update-drainer -node-id ip-127-0-0-1:8250/{nodeID} -state {state}
    ```

    这条命令会修改 Pump/Drainer 保存在 pd 中的状态.
 
3. 暂停/下线 Pump/Drainer

    ```
    bin/binlogctl -pd-urls=http://127.0.0.1:2379 -cmd pause-pump/pause-drainer/offline-pump/offline-drainer -node-id ip-127-0-0-1:8250/{nodeID}
    ```

    binlogctl 会发送 http 请求给 Pump/Drainer，Pump/Drainer 收到命令后会退出进程，并且将自己的状态设置为 paused/offline。

4. 生成 Drainer 启动需要的 meta 文件

    ```
    bin/binlogctl -pd-urls=http://127.0.0.1:2379 -cmd generate_meta

    INFO[0000] [pd] create pd client with endpoints [http://192.168.199.118:32379]
    INFO[0000] [pd] leader switches to: http://192.168.199.118:32379, previous:
    INFO[0000] [pd] init cluster id 6569368151110378289
    2018/06/21 11:24:47 meta.go:117: [info] meta: &{CommitTS:400962745252184065}
    ```

    该命令会生成一个文件 `{data-dir}/savepoint`， 该文件中保存了 Drainer 初次启动需要的 tso 信息。

注：binlogctl 的 github 链接：
[binlogctl](https://github.com/pingcap/tidb-tools/tree/develop/tidb-binlog/binlogctl)

### TiDB-Binlog 监控

使用 Ansible 部署成功后，可以进入 Grafana Web 界面（默认地址: <http://grafana_ip:3000>，默认账号：admin，密码：admin）查看 Pump 和 Drainer 的运行状态。
