# 基于Docker在AWS构建EMQ集群

## 概览

​	在物联网中，需要将各种形态的智能产品相互连接，从智慧农业、智慧工业、智慧城市、智慧交通等各种形态的垂直行业，大部分应用场景的数据流向是：终端数据上报给手机APP，手机下行反控设备，要实现这些设备之间稳定、可靠、高效、长期的互联，对连接平台的接入能力具有极高要求。*EMQ*(Erlang/Enterprise/Elastic MQTT Broker) 是基于 Erlang/OTP 语言平台开发，支持大规模连接和分布式集群，发布订阅模式的MQTT 消息服务器。

​	EMQ支持完整的物联网MQTT协议，通过发布/订阅的消息传输模式，可以实现终端设备之间的互联，具体实现的时候，订阅者只需要关注特定的主题，当有发送者针对该主题发送消息时，EMQ会转发该消息到对应的订阅者，订阅者就可以收到该消息。

### MQTT简介

​	大部分的物联网行业，MQTT 协议与相关产品负责把数据由传感器有效的传送到服务器，完成在受限、不稳定网络到因特网或企业网络的连接，实现两者互联互通。在此基础上，互通的物品不仅能通过设备采集信息、实现智能的感知，更能对数据进行信息处理、数据挖掘、人工智能等技术手段，与业务应用整合，实现从后台到前端设备的智能监控，完成进一步的信息化工作。归根结底，MQTT 协议是为大量计算能力有限，且工作在低带宽、不可靠的网络的远程传感器和控制设备通讯而设计的协议，它具有以下主要的几项特性：

- 非常小的通信开销（最小的消息大小为 2 字节）；
- 支持各种流行编程语言（包括 C，Java，Ruby，Python 等等）且易于使用的客户端；
- ​ 支持“发布 / 订阅”模型，简化应用程序的开发；
-  提供三种不同消息发布服务质量，让消息能按需到达目的地，适应在不稳定网络情况下的传输需求：

​		“至多一次”，会发生消息丢失或重复。

​		“至少一次”，确保消息到达，但消息重复可能会发生。

​		只有一次”，确保消息到达且只到达一次。


### Docker简介

​	Docker 也是容器技术的一种，它运行于 Linux 宿主机之上，每个运行的容器都是相互隔离的，作为一个管理容器的开源平台，它可以很轻松地创建轻量级，可移植的容器，这种低投入，轻量级的分布式运作平台对环境的构建减少了工作量。当前，越来越多的公司都已将以 Docker 为代表的容器技术用于企业级业务平台，本文将介绍如何使用 Docker 来构建 EMQ分布式集群以及基于Docker集群来测试百万连接测试。

## 测试场景概述

### Topic设计

在本文中，我们模拟100万Device、100万Mobile接入，以及一定量的消息吞吐测试，一台Server订阅topic/#获取所有数据。

上行数据：device->EMQ->mobile ，设备发送数据到移动终端；

下行数据：mobile->EMQ ->device 移动终端将反控数据到设备。

发布/订阅流向图：

![sequence](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/sequence.png)

### 测试用例

连接测试：需要考虑连接方式、MQTT客户端数量、KeepAlive等因素；

吞吐量测试：需要考虑消息大小，消息发布服务质量，消息总数，网络带宽和 MQTT 客户端数量：

- 连接方式

  ​采用SSL的连接方式对资源占用较大，实际应用场景可通过LB进行终结；

- KeepAlive

  ​定义服务器端从客户端接收消息的最大时间间隔，建议设置为300秒

- 消息大小  

  MQTT 固定长度的头部是 2 字节，所以在传输比较小的消息时 MQTT 是非常具有优势性的。消息的大小对性能有很大的影响，默认EMQ配置载荷限制64K。在消息大小很大的情况下，如果网络环境不稳定，经常发生断线重传，则会引起网络拥塞，影响性能。

- 服务质量（QoS）

  有三种消息发布服务质量，即 QoS 为 0、1 、2：

- 消息总数

  短时间内消息总数越大对网络的压力就越大。通过增大消息总数，可以观察在网络条件一定的情况下，消息总数和 MQTT 吞吐量之间的关系。

-  网络带宽

  网络带宽在 MQ 及 MQTT 进行消息传递时对性能的影响显而易见，在此就不再赘述。本文测试环境是基于AWS内网测试

-  MQTT 客户端数量

  ​在实际应用中，MQTT 客户端的数量通常是巨大的。在横向增加客户端数量的同时，观察 MQTT 服务吞吐量及响应时间的变化是必要的。由于 MQTT 使用的是发布订阅方式，在订阅场景中，增加 MQTT 客户端的数量会使吞吐量相应的线性增长，而响应时间变化不大。在发布场景中，随着发布者的增多，吞吐量会线性增加，而响应时间则被订阅者处理消息的速度所限制。

### 测试场景

考虑以上因素，我们组合了一下的测试场景：

| NO.  | Description                              | QoS                   |
| ---- | ---------------------------------------- | --------------------- |
| 1    | mobile:0.1Mdeveice: 0.1M                 | 100% QoS 0            |
| 2    | mobile:0.3Mdeveice: 0.3M                 | 100% QoS0             |
| 3    | mobile:0.5Mdeveice: 0.5M                 | 100% QoS0             |
| 4    | mobile:0.8Mdeveice: 0.8M                 | 100% QoS0             |
| 5    | mobile:1Mdeveice: 1M                     | 100% QoS0             |
| 6    | mobile:1Mdeveice: 1M                     | 80% QoS0 and 20% QoS1 |
| 7    | 1% of the connection will have til of random selection between 1-5min,mqtt disconnect while bringing up 2M connections | 100% QoS0             |
| 8    | 5% with til of 5-10 mins,mqtt disconnect while bringing up 2M connections | 100% QoS0             |

## 测试环境准备

### 部署架构



![diagram](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/diagram.png)

### 测试框架介绍

XMeter

​	XMeter是企业级的性能测试平台，提供公有云SaaS服务和企业私有环境部署。JMeter是XMeter的测试执行引擎。XMeter对JMeter执行引擎做了扩展和优化，按照用户不同规模的测试需求，规划和调度相应的JMeter集群，共同发起测试，实时处理和展现大规模的测试结果数据，并提供完善的系统监控，辅助分析性能问题等等，通过Xmeter性能测试平台，可生成标准测试报告，其中汇集了各类性能度量数据：测试服务器的资源指标、系统指标。

mqtt-Jmeter插件

​	mqtt-Jmeter是基于Jmeter扩展的插件，它能更加方便的添加MQTT连接、发布、订阅取样器。构造符合自己业务的应用场景，例如背景连接、多发少收的发布模式、少发多收的广播模式。详细的关于插件的使用介绍参见[官方网站](https://github.com/emqtt/mqtt-jmeter/releases/tag/v0.93)。

### 部署环境配置

所有的测试用例基于AWS VPC环境：

- EMQ

  ​• EMQ (2 nodes): 2.3.2

  ​• 16cores, 32GB RAM: 0.1M mobiles * 0.1M devices, 0.3M mobiles * 0.3M devices, 0.5M mobiles * 0.5 devices

  ​• 16cores, 64GB RAM: Rests of test scenarios

- Redis (1 node): 4.0.2

  ​• 16cores, 32GB RAM for all test scenarios

  ​• Ubuntu 16.04.3 LTS (GNU/Linux 4.4.0-1043-aws x86_64)

- XMeter

  ​• Master (1 node) & asteroid (1 node)

  ​• 8cores, 15GB RAM

  ​• CentOS Linux release 7.2.1511

  ​• Test agents (maximum 41 nodes)

  ​• 4cores, 30.5GB RAM

  ​• CentOS Linux release 7.2.1511

  ​• One VM simulates 50000 (25000 mobile & 25000 device)

  ​• Network

  ​• Intranet of AWS US western

### 部署测试环境

- Redis 部署

```
wget http://download.redis.io/releases/redis-4.0.2.tar.gz
tar xzf redis-4.0.2.tar.gz
cd redis-4.0.2 && sudo make install
```

-  EMQ部署

1. docker安装

```
sudo apt-get update && sudo apt-get install docker docker-engine docker.io
git clone https://github.com/emqtt/emq-docker
cd emq-docker && sudo docker build -t emq:latest  .
```

2. docker 启动EMQ

节点Node1:

```
sudo docker run --net="host" -tid --name emq -e "EMQ_LISTENER__TCP__EXTERNAL__MAX_CLIENTS=1048576" -e "EMQ_NAME=emq" -e "EMQ_HOST=172.31.6.52" -e "EMQ_NODE__DIST_LISTEN_MAX=6369" -e "EMQ_LOADED_PLUGINS=emq_auth_redis,emq_recon,emq_modules,emq_retainer,emq_dashboard" -e EMQ_AUTH__REDIS__SERVER="172.31.6.26:6379" -e "EMQ_AUTH__REDIS__PASSWORD_HASH=plain" -e "EMQ_MQTT__ALLOW_ANONYMOUS=false"  emq:latest
```

节点Node2:

```
sudo docker run --net="host" -tid --name emq -e "EMQ_LISTENER__TCP__EXTERNAL__MAX_CLIENTS=1048576" -e "EMQ_NAME=emq" -e "EMQ_HOST=172.31.6.127" -e "EMQ_NODE__DIST_LISTEN_MAX=6369" -e "EMQ_LOADED_PLUGINS=emq_auth_redis,emq_recon,emq_modules,emq_retainer,emq_dashboard" -e EMQ_AUTH__REDIS__SERVER="172.31.6.26:6379" -e "EMQ_AUTH__REDIS__PASSWORD_HASH=plain" -e "EMQ_JOIN_CLUSTER=emq@172.31.6.52" -e "EMQ_MQTT__ALLOW_ANONYMOUS=false"  emq:latest
```

3. 测试工具部署

-  [XMeter on-premise version](https://www.xmeter.net/)
-  Apache JMeter 3.2 (with OpenJDK 1.8.0_65)
-  [JMeter MQTT plugin](https://github.com/emqtt/mqtt-jmeter) (developed & maintained by XMeter), based on [fusesource-1.14 MQTT library](https://github.com/fusesource/mqtt-client)
-  System monitor tools: XMeter monitor solution (based on Collectd & Grafana)

由Xmeter提供私有化部署，更多了解，详情咨询[官方网站](https://www.xmeter.net/index.html)。

### 数据准备

1. 认证数据录入

   ```
   #!/bin/sh
   REDIS_BIN=redis-cli
   HOST=127.0.0.1
   PORT=6379
   for userid in `seq 1 2000000`;do
   if [ $(($userid%10000)) = '0' ];then
       echo "userid:::  $userid...."
       ${REDIS_BIN} -h ${HOST} -p ${PORT} hset mqtt_user:user${userid} password "123456"
       sleep 7
       continue
   else
       ${REDIS_BIN} -h ${HOST} -p ${PORT} hset mqtt_user:user${userid} password "123456"
   fi
   done
   ```

2. ACL访问控制数据录入

   ```
   #!/bin/sh
   REDIS_BIN=redis-cli
   HOST=127.0.0.1
   PORT=6379
   for t in `seq 1 20`;do
   for userid in `seq 1 1000000`;do
   if [ $(($userid%15000)) = '0' ];then
       echo "userid:::  $userid...."
       echo "topic:::  $t...."
       ${REDIS_BIN} -h ${HOST} -p ${PORT} hset mqtt_acl:user${userid} topic/device_user$userid/t_$t 3
       sleep 7
       continue
   else
       ${REDIS_BIN} -h ${HOST} -p ${PORT} hset mqtt_acl:user${userid} topic/device_user$userid/t_$t 3
   fi
   done
   done
   ```

## 性能调优

**Linux Kernel Tuning**

The system-wide limit on max opened file handles:

sysctl -w fs.file-max=2097152

sysctl -w fs.nr_open=2097152

echo 2097152 > /proc/sys/fs/nr_open

echo 2097152 > /proc/sys/fs/file-max

**Tuning Linux ulimits**

The limit on opened file handles for current session:

ulimit -n 1048576

**/etc/sysctl.conf**

Add the ‘fs.file-max’ to /etc/sysctl.conf, make the changes permanent:

fs.file-max = 1048576

**/etc/security/limits.conf**

Persist the limits on opened file handles for users in /etc/security/limits.conf:

*      soft   nofile      1048576

*      hard   nofile      1048576

**Network Tuning**

Increase number of incoming connections backlog:

​	sysctl -w net.core.somaxconn=32768

​	sysctl -w net.ipv4.tcp_max_syn_backlog=16384

​	sysctl -w net.core.netdev_max_backlog=16384	

Local Port Range

​	sysctl -w net.ipv4.ip_local_port_range=1000 65535

​	

Read/Write Buffer for TCP connections

​	sysctl -w net.core.rmem_default=262144

​	sysctl -w net.core.wmem_default=262144

​	sysctl -w net.core.rmem_max=16777216

​	sysctl -w net.core.wmem_max=16777216

​	sysctl -w net.core.optmem_max=16777216

​	sysctl -w net.ipv4.tcp_rmem=1024 4096 16777216

​	sysctl -w net.ipv4.tcp_wmem=1024 4096 16777216

Connection Tracking

​	sysctl -w net.nf_conntrack_max=1000000

​	sysctl -w net.netfilter.nf_conntrack_max=1000000

​	sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30

The TIME-WAIT Buckets Pool, Recycling and Reuse

​	net.ipv4.tcp_max_tw_buckets=1048576

Timeout for FIN-WAIT-2 sockets:

​	net.ipv4.tcp_fin_timeout = 15

**Erlang VM Tuning**

Tuning and optimize the Erlang VM in etc/emq.conf file, The following parameters are modified by the environment variable of the EMQ Docker container

node.process_limit = 2097152

*## Sets the maximum number of simultaneously existing ports for this system*

node.max_ports = 1048576

**The EMQ Broker**

Tune the acceptor pool, max_clients limit and sockopts for TCP listener in etc/emq.config:

*## TCP Listener*

listener.tcp.external = 0.0.0.0:1883

listener.tcp.external.acceptors = 64

listener.tcp.external.max_clients = 1048576

## 测试步骤

### Scenario 1

0.1M mobiles * 0.1M devices (QoS 0)

| **Sampler** | **Avg resp time(s)** | **Max resp time(s)** | **Succ rate** |
| ----------- | -------------------- | -------------------- | ------------- |
| Devicepub   | 0                    | 0.4230               | 100%          |
| Devicesub   | 0.0071               | 5.065                | 100%          |
| Mobilepub   | 0                    | 0.5430               | 100%          |
| Mobilesub   | 0.0007               | 0.5640               | 100%          |

emqx1 & emqx2 are balanced, and CPUuser usage is about 5% in average. The CPU usage increases to 12% whendisconnecting the connections. Memory usage is about in 3.3GB when load isstable state. 

Notice: Only one ofemq server’s CPU and memory usage will be attached for rests of scenarios,because the load is well balanced between 2 servers.

![No1_emq1_cpu](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No1_emq1_cpu.png)

![No1_emq1_mem](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No1_emq1_mem.png)

![No1-emq2](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No1-emq2.png)

Below report data is fetched from EMQ API.

1. The peak message dispatchingrate is about 4k/s

2.  Connections are ramp-up for 2stages (1st stage for mobile sub, and 2nd stage fordevice pub.

3.  The maximum connection numberis 200000.

   ![No1-emq_broker](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No1-emq_broker.png)

   ### Scenario 2

   0.3M mobiles * 0.3M devices (QoS 0) Refer to below table for response time and

   successful rate, the test result is good.

   | **Sampler** | **Avg resp time(s)** | **Max resp time(s)** | **Succ rate** |
   | ----------- | -------------------- | -------------------- | ------------- |
   | Devicepub   | 0                    | 0.5860               | 100%          |
   | Devicesub   | 0.0008               | 0.6070               | 100%          |
   | Mobilepub   | 0                    | 0.6060               | 100%          |
   | Mobilesub   | 0.0007               | 0.8500               | 100%          |

emqx1 & emqx2 are balanced, and CPU
user usage is about 10% in average. The CPU usage increases to 30% when
disconnecting the connections. Memory usage is about in 9GB when load is stable state. 

![No2-emq1](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No2-emq1.png)

The CPU usage of Redis server is less than
0.5%, and memory is about 1.35GB.

![No2-redis](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No2-redis.png)

Below report data is fetched from EMQ API.

1)        The peak message dispatchingrate is about 8k/s

2)        Connections are ramp-up for 2stages (1st stage for mobile sub, and 2nd stage fordevice pub.

3)        The maximum connection numberis 600000.

![No2-emq_broker](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No2-emq_broker.png)

 This report does not include all of the metrics
collected during the performance testing. 

### Scenario 3

0.5M mobiles * 0.5M devices (QoS 0)

Refer to below table for response time and
successful rate, the test result is good.

| **Sampler** | **Avg resp time(s)** | **Max resp time(s)** | **Succ rate** |
| ----------- | -------------------- | -------------------- | ------------- |
| Devicepub   | 0                    | 0.6330               | 100%          |
| Devicesub   | 0.0008               | 0.7640               | 100%          |
| Mobilepub   | 0                    | 0.5900               | 100%          |
| Mobilesub   | 0.0007               | 0.6570               | 100%          |

emqx1 & emqx2 are balanced, and CPU
user usage is about 20% in average. The CPU usage increases to 30+% when
disconnecting the connections. Memory usage is about in 14GB when load is
stable state. 

![No3-emq](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No3-emq.png)

The CPU usage of Redis server is less than
0.5%, and memory about 1.35GB.

![No3-redis](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No3-redis.png)

Below report data is fetched from EMQ API.

1)        The peak message dispatchingrate is about 10k/s

2)        Connections are ramp-up for 3stages (1st stage for mobile sub, and 2nd stage for mobilesub + device pub, and last stage is only device pub).

3)        The maximum connection numberis 1000000.

![No3-emq_broker](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No3-emq_broker.png)

This report does not include all of the metrics
collected during the performance testing.

### Scenario 4

0.8M mobiles * 0.8M devices (QoS 0)

The scenario adds a subscriber that sub all
of topics (from t1 – t20), which simulating the backend server side
application. Refer to below table for response time and successful rate, the
test result is good. NOTICE:
The test result for ListenToAllTopics is not very accurate – because it
receives message from all topics, but the response time is calculated by:
timestamp in received machine – timestamp in sent machine. It probably has
minor difference between the machines, so the result is not very accurate.

| **Sampler**       | **Avg resp time(s)** | **Max resp time(s)** | **Succ rate** |
| ----------------- | -------------------- | -------------------- | ------------- |
| Devicepub         | 0                    | 0.6240               | 100%          |
| Devicesub         | 0.0011               | 0.6490               | 100%          |
| Mobilepub         | 0                    | 0.6400               | 100%          |
| Mobilesub         | 0.0011               | 0.6290               | 100%          |
| ListenToAllTopics | 0.0014               | 0.2610               | 100%          |

emqx1 & emqx2 are balanced, and CPU
user usage is about 40% in average. The CPU usage increases to 60+% when
disconnecting the connections. Memory usage is about in 23GB when load is in stable state. 

![No4-emq](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No4-emq.png)

The CPU usage of Redis server is less than
1%, and memory about1.4GB. Please refer to below.

![No4-redis](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No4-redis.png)

Below report data is fetched from EMQ API.

1)        The peak message dispatchingrate is about 30k/s

2)        Connections are ramp-up for 3stages (1st stage for mobile sub, and 2nd stage for mobilesub + device pub, and last stage is only device pub).

3)        The maximum connection numberis 1600000.

![No4-emq_broker](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No4-emq_broker.png)

This report does not include all of the metrics
collected during the performance testing

### Scenario 5

1M mobiles * 1M devices (QoS 0)

The scenario adds a subscriber that sub all
of topics (from t1 – t20), which simulating the backend server side
application. Refer to below table for response time and successful rate. NOTICE: The test result for
ListenToAllTopics is not very accurate – because it receives message from all
topics, but the response time is calculated by: timestamp in received machine –
timestamp in sent machine. It probably has minor difference between the
machines, so the result is not very accurate.

| **Sampler**       | **Avg resp time(s)** | **Max resp time(s)** | **Succ rate** |
| ----------------- | -------------------- | -------------------- | ------------- |
| Devicepub         | 0                    | 0.6650               | 100%          |
| Devicesub         | 0.0017               | 0.6910               | 100%          |
| Mobilepub         | 0                    | 0.6900               | 100%          |
| Mobilesub         | 0.0017               | 0.6870               | 100%          |
| ListenToAllTopics | 0.1528               | 12.1050              | 100%          |

Problems

l   Devicepub: 41 for failed to create connection

l   Mobilepub: 2 pub failed

l   Mobilesub: 1 for failed to create connection

emqx1 & emqx2 are balanced, and CPU
user usage is about 45% in average. The CPU usage increases to 67% when
disconnecting the connections. Memory max usage is about in 30GB. 

![No5-emq](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No5-emq.png)

The CPU usage of Redis server is about 1.5%, and
memory about1.4GB. Please refer to below

![No5-redis](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No5-redis.png)

Below report data is fetched from EMQ API.

1)        The peak message dispatchingrate is about 30k/s

2)        Connections are ramp-up for 3stages (1st stage for mobile sub, and 2nd stage for mobilesub + device pub, and last stage is only device pub).

3)        The maximum connection numberis 2000000.![No5-emq_broker](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No5-emq_broker.png)This report does not include all of the metrics
collected during the performance testing.

### Scenario 6

1M mobiles * 1M devices (80% QoS 0, 20% QoS1)

l   The scenario has 20% of QoS1 pub/subs, rests of are QoS0. 

l   The scenario adds a subscriber that sub all of topics (from t1 –t20), which simulating the backend server side application. Refer to belowtable for response time and successful rate. NOTICE: The test result for ListenToAllTopics is notvery accurate – because it receives message from all topics, but the responsetime is calculated by: timestamp in received machine – timestamp in sentmachine. It probably has minor difference between the machines, so the resultis not very accurate.

| **Sampler**       | **Avg resp time(s)** | **Max resp time(s)** | **Succ rate** |
| ----------------- | -------------------- | -------------------- | ------------- |
| Devicepub (QoS0)  | 0                    | 0.6330               | 100%          |
| Devicesub (QoS0)  | 0.0015               | 0.6770               | 100%          |
| Mobilepub (QoS0)  | 0                    | 0.6760               | 100%          |
| Mobilesub (QoS0)  | 0.0016               | 0.7510               | 100%          |
| Devicepub (QoS1)  | 0.0013               | 0.5530               | 99.99%        |
| Devicesub (QoS1)  | 0.0016               | 0.6830               | 100%          |
| Mobilepub (QoS1)  | 0.0013               | 0.7970               | 100%          |
| Mobilesub (QoS1)  | 0.0016               | 0.8670               | 100%          |
| ListenToAllTopics | 0.0010               | 0.0330               | 100%          |

Problems:

l   Devicepub (QoS1): 49 for failed to create connection

l   Devicesub(QoS0): 1 for failed to subscribe topic

l   Devicepub(Qos0): 1 for failed to pub message

l   The subscriber sometimes cannot receive message after 14 minutes

Problems:

l   Devicepub (QoS1): 49 for failed to create connection

l   Devicesub(QoS0): 1 for failed to subscribe topic

l   Devicepub(Qos0): 1 for failed to pub message

l   The subscriber sometimes cannot receive message after 14 minutes

emqx1 & emqx2 are balanced, and CPU
user usage is about 45% in average. The CPU usage increases to 69% when
disconnecting the connections. Memory max usage is about in 30.3GB. 

![No6-emq](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No6-emq.png)The CPU usage of Redis server is about 1.5%, and
memory about1.4GB. Please refer to below.

![No6-redis](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No6-redis.png)

Below report data is fetched from EMQ API.

1)        The peak QoS0 messagedispatching rate is about 26/s; The peak QoS1 message dispatching rate is about4k/s.

2)        Connections are ramp-up for 3stages (1st stage for mobile sub, and 2nd stage for mobilesub + device pub, and last stage is only device pub).

3)        The maximum connection numberis 2000000.

![No6-emq_broker](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No6-emq_broker.png)

![No6-tps](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No6-tps.png)

This report does not include all of the metrics
collected during the performance testing. 

### Scenario 7

 Radom TTL disconnect (1 – 5 min, QoS0)

1% of the connections will have ttl ofrandom selection between 1-5min, mqtt disconnect while bringing up 2Mconnections. Add following 1 test script (Aeris_1M_10k_TTL.jmx) for simulate10k mobiles (id from 99001 – 100000) & 10k (id from 99001 – 100000) devicesthat connect to EMQ, and with 1-5 minutes, then dis-connect from EMQ. 

 

The scenario adds a subscriber that sub allof topics (from t1 – t20), which simulating the backend server sideapplication. Refer to below table for response time and successful rate. NOTICE: The test result forListenToAllTopics is not very accurate – because it receives message from alltopics, but the response time is calculated by: timestamp in received machine –timestamp in sent machine. It probably has minor difference between themachines, so the result is not very accurate.

| **Sampler**       | **Avg resp time(s)** | **Max resp time(s)** | **Succ rate** |
| ----------------- | -------------------- | -------------------- | ------------- |
| Devicepub         | 0                    | 0.6650               | 100%          |
| Devicesub         | 0.0016               | 0.6910               | 100%          |
| Mobilepub         | 0                    | 0.6900               | 100%          |
| Mobilesub         | 0.0016               | 0.6870               | 100%          |
| ListenToAllTopics | 0.0008               | 0.0730               | 100%          |

Problems

l   The subscriber sometimes cannot receive message after 22 minutes

 

emqx1 & emqx2 are balanced, and CPUuser usage is about 45% in average. The CPU usage increases to 69% whendisconnecting the connections. Memory max usage is about in 30.7GB. 

![No7-emq](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No7-emq.png)

The CPU usage of Redis server is about
1.5%, and memory about1.4GB. Please refer to below.![No7-redis](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No7-redis.png)

Below report data is fetched from EMQ API.

1)        The peak message dispatchingrate is about 30k/s

2)        Connections are ramp-up for 3stages (1st stage for mobile sub, and 2nd stage for mobilesub + device pub, and last stage is only device pub).

3)        The maximum connection numberis 1980000.

![No7-emq_broker](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No7-emq_broker.png)

This report does not include all of the metrics
collected during the performance testing. 

### Scenario 8

Radom TTL disconnect (5 – 10 min, QoS0)

5% with ttl of 5-10 mins, mqtt disconnectwhile bringing up 2M connections, mqtt disconnect while bringing up 2Mconnections. Add following 1 test script (Aeris_1M_500k_TTL) for simulate 50kmobiles (id from 95001 – 100000) & 50k (id from 95001 – 100000) devicesthat connect to EMQ, and with 5-10 minutes, then dis-connect from EMQ. 

 

The scenario adds a subscriber that sub allof topics (from t1 – t20), which simulating the backend server sideapplication. Refer to below table for response time and successful rate. NOTICE: The test result forListenToAllTopics is not very accurate – because it receives message from alltopics, but the response time is calculated by: timestamp in received machine –timestamp in sent machine. It probably has minor difference between themachines, so the result is not very accurate.

| **Sampler**       | **Avg resp time(s)** | **Max resp time(s)** | **Succ rate** |
| ----------------- | -------------------- | -------------------- | ------------- |
| Devicepub         | 0                    | 0.6650               | 100%          |
| Devicesub         | 0.0015               | 0.6910               | 100%          |
| Mobilepub         | 0                    | 0.6900               | 100%          |
| Mobilesub         | 0.0016               | 0.6870               | 100%          |
| ListenToAllTopics | 0.0437               | 4.8690               | 100%          |

emqx1 & emqx2 are balanced, and CPU
user usage is about 45% in average. The CPU usage increases to 66% when
disconnecting the connections. Memory max usage is about in 29GB. 

![No8-emq](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No8-emq.png)

The CPU usage of Redis server is about
1.5%, and memory about1.4GB. Please refer to below.

![No8-redis](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No8-redis.png)

Below report data is fetched from EMQ API.

1)        The peak message dispatchingrate is about 30k/s

2)        Connections are ramp-up for 3stages (1st stage for mobile sub, and 2nd stage for mobilesub + device pub, and last stage is only device pub).

3)        The maximum connection numberis 1900000.

![No8-emq_broker](/Users/huangdan/emq/emq-tutorial-cn/aws_emqDocker_performance/_image/No8-emq_broker.png)

This report does not include all of the metrics
collected during the performance testing. 

## 总结