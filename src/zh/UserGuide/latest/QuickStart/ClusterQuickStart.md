<!--

    Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements.  See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership.  The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License.  You may obtain a copy of the License at
    
        http://www.apache.org/licenses/LICENSE-2.0
    
    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied.  See the License for the
    specific language governing permissions and limitations
    under the License.

-->

# 集群版
本文将简单介绍 IoTDB 集群的安装配置、扩容和缩容等常规操作。
遇到问题可以看: 
[FAQ](../FAQ/Frequently-asked-questions.md)

## 安装部署

部署集群时我们推荐优先使用`hostname`来配置集群，这样可以避免一些网络问题。需要在每个节点上分别配置`/etc/hosts` ，windows 上为`C:\Windows\System32\drivers\etc\hosts`。

我们将以最小的改动，启动一个含有3个 ConfigNode 和3个DataNode(3C3D)集群：
- 数据/元数据副本数为1
- 集群名称为defaultCluster
- Confignode JVM 的最大堆内存配置为机器内存的 1/4
- Datanode JVM 的最大堆内存配置为机器内存的 1/4

假设有3台物理机(下面称节点)，操作系统为Linux，并且已经安装配置好了JAVA环境(具体见[单机版对安装环境说明](../QuickStart/QuickStart.md))，安装目录均为`/data/iotdb`。
IP地址和服务角色分配如下：

| 节点IP | 192.168.132.10 | 192.168.132.11 | 192.168.132.12 |
|--------|:---------------|:---------------|:---------------|
| `hostname`   | iotdb-1     | iotdb-2     | iotdb-3     |
| 服务   | ConfigNode     | ConfigNode     | ConfigNode     |
| 服务   | DataNode       | DataNode       | DataNode       |

端口占用：

| 服务 | ConfigNode | DataNode |
|---|---|---|
| 端口 | 10710, 10720 | 6667, 10730, 10740, 10750, 10760 |

**说明：**
- 可以使用`IP地址`或者`hostname(机器名/域名)`来安装配置 IoTDB 集群，本文以`hostname(机器名/域名)`为例。使用`hostname(机器名/域名)`，需要配置`/etc/hosts`。
- 优先推荐使用 `hostname(机器名/域名)` 进行配置，这样可以避免一些网络问题，也更方便迁移集群。
- JVM堆内存配置: `confignode-env.sh` 和 `datanode-env.sh` 内配置`ON_HEAP_MEMORY`, 建议设置值大于等于1G。ConfigNode 1~2G就足够了，DataNode的内存配置则要取决于数据接入的数据量和查询数据量。

### 下载安装包
在每个节点，将安装包[下载](https://iotdb.apache.org/Download/)后，解压到安装目录，这里为`/data/iotdb`。
目录结构:
```shell
/data/iotdb/
├── conf    # 配置文件
├── lib     # jar library
├── sbin    # 启动/停止等脚本
└── tools   # 其他工具
```

### 修改节点配置文件

在每个节点均配置 hosts
```shell
echo "192.168.132.10 iotdb-1" >> /etc/hosts 
echo "192.168.132.11 iotdb-2" >> /etc/hosts 
echo "192.168.132.12 iotdb-3" >> /etc/hosts 
```

配置文件在 `/data/iotdb/conf`目录下。
按照下表修改相应的配置文件：

|  配置|      配置项      |IP:192.168.132.10       | IP:192.168.132.11       | IP:192.168.132.12       |
|------------|:-------------------------------|----------------------|----------------------|:---------------------|
| iotdb-system.properties | cn_internal_address          | iotdb-1       | iotdb-2       | iotdb-3       |
| iotdb-system.properties           | cn_seed_config_node | iotdb-1:10710 | iotdb-1:10710 | iotdb-1:10710 |
| iotdb-system.properties   | dn_rpc_address               | iotdb-1       | iotdb-2       | iotdb-3       |
| iotdb-system.properties           | dn_internal_address          | iotdb-1       | iotdb-2       | iotdb-3       |
| iotdb-system.properties           | dn_seed_config_node | iotdb-1:10710 | iotdb-1:10710 | iotdb-1:10710 |       

**注意：**
我们推荐所有节点的 iotdb-system.properties 和 JVM 的内存配置是一致的。

### 启动集群
启动集群前，需保证配置正确，保证 IoTDB 安装目录下没有数据(`data`目录)。
#### 启动第一个节点
即上面表格中`cn_seed_config_node`配置的节点。
登录该节点 `iotdb-1(192.168.132.10)`，执行下面命令：
```shell
cd /data/iotdb
# 启动 ConfigNode 和 DataNode 服务
sbin/start-standalone.sh
    
# 查看 DataNode 日志以确定启动成功
tail -f logs/log_datanode_all.log
# 期望看见类似下方的日志
# 2023-07-21 20:26:01,881 [main] INFO  o.a.i.db.service.DataNode:192 - Congratulation, IoTDB DataNode is set up successfully. Now, enjoy yourself!
```

如果没有看到上面所说的日志或者看到了 Exception，那么代表启动失败了。请查看 `/data/iotdb/logs` 目录内的`log_confignode_all.log` 和 `log_datanode_all.log` 日志文件。

**注意**：
- 要保证第一个节点启动成功后，再启动其他节点。确切的说，要先保证第一个 ConfigNode 服务启动成功，即`cn_seed_config_node`配置的节点。
- 如果启动失败，需要[清理环境](#【附录】清理环境)后，再次启动。
- ConfigNode 和 DataNode 服务都可以单独启动: 
```shell
# 单独启动 ConfigNode， 后台启动
sbin/start-confignode.sh -d
# 单独启动 DataNode，后台启动
sbin/start-datanode.sh -d
```

#### 启动其他两个节点的 ConfigNode 和 DataNode
在节点 `iotdb-2(192.168.132.11)` 和 `iotdb-3(192.168.132.12)` 两个节点上分别执行：
```shell
cd /data/iotdb
# 启动 ConfigNode 和 DataNode 服务
sbin/start-standalone.sh
```
如果启动失败，需要在所有节点执行[清理环境](#【附录】清理环境)后，然后从启动第一个节点开始，再重新执行一次。

#### 检验集群状态
在任意节点上，在 Cli 执行 `show cluster`:
```shell
/data/iotdb/sbin/start-cli.sh -h iotdb-1
IoTDB>show cluster;
# 示例结果如下：
+------+----------+-------+---------------+------------+-------+---------+
|NodeID|  NodeType| Status|InternalAddress|InternalPort|Version|BuildInfo|
+------+----------+-------+---------------+------------+-------+---------+
|     0|ConfigNode|Running|    iotdb-1    |       10710|1.x.x  |  xxxxxxx|
|     1|  DataNode|Running|    iotdb-1    |       10730|1.x.x  |  xxxxxxx|
|     2|ConfigNode|Running|    iotdb-2    |       10710|1.x.x  |  xxxxxxx|
|     3|  DataNode|Running|    iotdb-2    |       10730|1.x.x  |  xxxxxxx|
|     4|ConfigNode|Running|    iotdb-3    |       10710|1.x.x  |  xxxxxxx|
|     5|  DataNode|Running|    iotdb-3    |       10730|1.x.x  |  xxxxxxx|
+------+----------+-------+---------------+------------+-------+---------+
``` 
**说明：**
`start-cli.sh -h` 后指定的IP地址，可以是任意一个 DataNode 的IP地址。


### 【附录】清理环境
在所有节点执行：
1. 结束 ConfigNode 和 DataNode 进程。
```shell
# 1. 停止 ConfigNode 和 DataNode 服务
sbin/stop-standalone.sh

# 2. 检查是否还有进程残留
jps
# 或者
ps -ef|grep iotdb

# 3. 如果有进程残留，则手动kill
kill -9 <pid>
# 如果确定机器上仅有1个iotdb，可以使用下面命令清理残留进程
ps -ef|grep iotdb|grep -v grep|tr -s '  ' ' ' |cut -d ' ' -f2|xargs kill -9
```

2. 删除 data 和 logs 目录。
```shell
cd /data/iotdb
rm -rf data logs
``` 

说明：删除 data 目录是必要的，删除 logs 目录是为了纯净日志，非必需。


## 集群扩容
扩容方式与上方启动其他节点相同。也就是，在要添加的节点上，下载IoTDB的安装包，解压，修改配置，然后启动。这里要添加节点的IP为 `192.168.132.13`
**注意：**
- 扩容的节点必须是干净的节点，不能有数据(也就是`data`目录)
- iotdb-system.properties中的`cluster_name`的配置必须和已有集群一致。
- `cn_seed_config_node` 和 `dn_seed_config_node`的配置必须和已有集群一致。
- 原有数据不会移动到新节点，新创建的元数据分区和数据分区很可能在新的节点。

### 修改配置
在原节点上新增一行 hosts
```shell
echo "192.168.132.13 iotdb-4" >> /etc/hosts 
```

在节点设置 hosts
```shell
echo "192.168.132.10 iotdb-1" >> /etc/hosts 
echo "192.168.132.11 iotdb-2" >> /etc/hosts 
echo "192.168.132.12 iotdb-3" >> /etc/hosts 
echo "192.168.132.13 iotdb-4" >> /etc/hosts 
```
按照下表修改相应的配置文件：

|  配置 |      配置项      | IP:192.168.132.13  | 
|------------|:-------------------------------|:---------------------|
| iotdb-system.properties | cn_internal_address          | iotdb-4       | 
| iotdb-system.properties           | cn_seed_config_node |  iotdb-1:10710 | 
| iotdb-system.properties   | dn_rpc_address               | iotdb-4       | 
| iotdb-system.properties           | dn_internal_address          | iotdb-4       | 
| iotdb-system.properties           | dn_seed_config_node | iotdb-1:10710 | 

### 扩容
在新增节点`iotdb-4(192.168.132.13)`上，执行：
```shell
cd /data/iotdb
# 启动 ConfigNode 和 DataNode 服务
sbin/start-standalone.sh
```

### 验证扩容结果
在 Cli 执行 `show cluster`，结果如下：
```shell
/data/iotdb/sbin/start-cli.sh -h iotdb-1
IoTDB>show cluster;
# 示例结果如下：
+------+----------+-------+---------------+------------+-------+---------+
|NodeID|  NodeType| Status|InternalAddress|InternalPort|Version|BuildInfo|
+------+----------+-------+---------------+------------+-------+---------+
|     0|ConfigNode|Running|    iotdb-1    |       10710|1.x.x  |  xxxxxxx|
|     1|  DataNode|Running|    iotdb-1    |       10730|1.x.x  |  xxxxxxx|
|     2|ConfigNode|Running|    iotdb-2    |       10710|1.x.x  |  xxxxxxx|
|     3|  DataNode|Running|    iotdb-2    |       10730|1.x.x  |  xxxxxxx|
|     4|ConfigNode|Running|    iotdb-3    |       10710|1.x.x  |  xxxxxxx|
|     5|  DataNode|Running|    iotdb-3    |       10730|1.x.x  |  xxxxxxx|
|     6|ConfigNode|Running|    iotdb-4    |       10710|1.x.x  |  xxxxxxx|
|     7|  DataNode|Running|    iotdb-4    |       10730|1.x.x  |  xxxxxxx|
+------+----------+-------+---------------+------------+-------+---------+
``` 

## 集群缩容
**注意:**
- 可以在任何一个集群内的节点上，执行缩容操作。
- 集群内的任意节点都可以被缩容。但是存留的 DataNode 服务不能小于副本数设置。
- 请耐心等待缩容脚本执行结束，并仔细阅读日志说明，尤其是结束前的指南说明。

### 缩容一个 ConfigNode
```shell
cd /data/iotdb
# 方式一：使用 ip:port 移除
sbin/remove-confignode.sh iotdb-4:10710

# 方式二：使用节点编号移除, `show cluster`中的 NodeID 
sbin/remove-confignode.sh 6
```

### 缩容一个 DataNode
```shell
cd /data/iotdb
# 方式一：使用 ip:port 移除
sbin/remove-datanode.sh iotdb-4:6667

# 方式二：使用节点编号移除, `show cluster`中的 NodeID
sbin/remove-datanode.sh 7
```

### 验证缩容结果

在 Cli 执行 `show cluster`，结果如下：
```shell
+------+----------+-------+---------------+------------+-------+---------+
|NodeID|  NodeType| Status|InternalAddress|InternalPort|Version|BuildInfo|
+------+----------+-------+---------------+------------+-------+---------+
|     0|ConfigNode|Running|    iotdb-1    |       10710|1.x.x  |  xxxxxxx|
|     1|  DataNode|Running|    iotdb-1    |       10730|1.x.x  |  xxxxxxx|
|     2|ConfigNode|Running|    iotdb-2    |       10710|1.x.x  |  xxxxxxx|
|     3|  DataNode|Running|    iotdb-2    |       10730|1.x.x  |  xxxxxxx|
|     4|ConfigNode|Running|    iotdb-3    |       10710|1.x.x  |  xxxxxxx|
|     5|  DataNode|Running|    iotdb-3    |       10730|1.x.x  |  xxxxxxx|
+------+----------+-------+---------------+------------+-------+---------+
```
