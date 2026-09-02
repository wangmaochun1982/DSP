【安装部署】Doris 集群的手动部署

https://kb.fit2cloud.com/?p=60

Apache Doris 简介：
Apache Doris 是一个现代化的 MPP 分析型数据库产品，仅需要亚秒级响应时间即可获得查询结果，可有效的支持实时数据分析。

# 1 搭建环境所需节点
示例准备 4 台机器配置 CentOS 7.6 4核8G 100G

节点	IP	角色
master	10.1.11.77	fe（Leader）
slave1	10.1.11.81	fe（Follower）+ be
slave2	10.1.11.53	be
slave3	10.1.11.24	be



设置系统最大打开文件句柄数:

```shell
vim /etc/security/limits.conf
* soft nofile 65536
* hard nofile 65536

```

安装包下载地址：

```bash
https://f2c-south.oss-cn-shenzhen.aliyuncs.com/yewenhuan/doris资源/doris-source.tar?OSSAccessKeyId=LTAI4GDCfBgxQQaD6av4zwxP&Expires=11645168844&Signature=eXJGKYgkdrZlPDEPPcMslgVpe0w%3D
```

此示例安装包放 /opt/doris 目录：

```shell
cp -ri output /opt/doris

```


# 2 环境要求

注意： 以下配置每个节点上都要操作

2.1 安装 Java 环境
此示例安装与编译 Doris 对应版本的 java 11 环境

```bash
yum install -y java-11-openjdk-11.0.14.1.1-1.el7_9.x86_64 java-11-openjdk-devel-11.0.14.1.1-1.el7_9.x86_64

```


2.2 配置 JAVA_HOME

```shell

vim /etc/profile
### set java environment
JAVA_HOME=/usr/lib/jvm/java-11-openjdk-11.0.14.1.1-1.el7_9.x86_64
JRE_HOME=$JAVA_HOME/jre
PATH=$JAVA_HOME/bin:$PATH
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/jre/lib/tools.jar:$JRE_HOME/lib:$CLASSPATH
export JAVA_HOME JAR_HOME PATH CLASS_PATH

source /etc/profile

```

2.3 注意事项
需注意各节点之间的网络互通及时间同步，可自行关闭防火墙，通过 NTP 协议校准各节点之间的时间。

# 3 配置 FE 主节点

登录到 master 节点，执行以下操作

3.1 创建元数据

```shell
rm -rf /opt/doris/fe/doris-meta && mkdir /opt/doris/fe/doris-meta

```

3.2 修改配置文件


```shell
vim /opt/doris/fe/conf/fe.conf
### 修改
priority_networks = 10.1.11.77/24
### 示例 IP 地址为本机 IP

```

3.3 启动 FE


```shell
bash start_fe.sh --daemon

```


3.4 检测 Doris 是否正常启动
执行 jps，若看到 PaloFe 表示 FE 已正常启动，否则检查日志文件，排查问题后再次启动。


# 4 配置 FE 从节点

登录到 slave2 节点将安装文件复制到 /opt/doris 目录下

```shell
scp -r master:/root/output /opt/doris

```

4.1 修改配置文件

```shell
vim /opt/doris/fe/conf/fe.conf
### 修改
priority_networks = 10.1.11.53/24
### 示例 IP 地址为本机 IP，修改
edit_log_port=9011

```


4.2 创建元数据目录

```shell
rm -rf /opt/doris/fe/doris-meta && mkdir /opt/doris/fe/doris-meta

```

4.3 启动 FE

```shell

bash start_fe.sh --helper 10.1.11.77:9010 --daemon

```

4.4 检测 Doris 是否正常启动
执行 jps ，若看到 PaloFe 表示 FE 已正常启动，否则检查日志文件，排查问题后再次启动。


## 5 将 slave1 添加到 master 节点

登录到 master 节点通过 mysql 客户端进入到 doris-fe，执行：

```shell
mysql -h master -P 9030 -uroot 

```

查看前端，将 slave1 添加 master 节点


```shell
show proc '/frontends' \G;
alter system add observer "10.1.11.53:9011";

```


# 6 添加 Be 节点
6.1 添加 slave2 节点的 BE
修改配置文件

```shell

vi /opt/doris/be/conf/be.conf
### 修改 slave2节点 IP
priority_networks = 10.1.11.53/24
### 存储目录
storage_root_path = /home/storage,50

```

创建元数据目录

```shell
mkdir -p /home/storage

```

启动 be

```shell
bash start_be.sh --daemon

```


登录到 master 节点添加 be，be 上的 heartbeat_service_port 端口，默认 9050：

```shell
alter system add backend "slave2:9050";

```

6.2 添加 slave3 节点的 BE
修改配置文件

```shell
vi /opt/doris/be/conf/be.conf
### 修改 slave3 节点 IP
priority_networks = 10.1.11.24/24
### 存储目录 storage_root_path = /home/storage,50

```

创建元数据目录

```shell
mkdir -p /home/storage

```

启动 be

```shell

bash start_be.sh --daemon
 
```

登录到 master 节点添加 be

```bash
alter system add backend "slave3:9050";

```

6.3 添加 slave1 节点的 BE
修改配置文件


```bash
vi /opt/doris/be/conf/be.conf
修改 slave1 节点 IP
### priority_networks = 10.1.11.81/24
存储目录
### storage_root_path = /home/storage,50


```

创建元数据目录

```bash

mkdir -p /home/storage

```


启动 be

```bash
bash start_be.sh --daemon


```

登录到 master 节点添加 be

```shell

alter system add backend "slave1:9050";

```


如下图所示，访问 master:8030，账户为root，密码默认为空不填写，检查 be 节点状态，alive 必须为 true

# 7 常见问题

7.1 启动 FE 过程中提示端口被占用
端口被占用，一般可修改 fe.conf 文件，然后再次启动：
先查看 FE 日志：

```shell

tail -n 30 /opt/doris/fe/log/fe.log

```

修改 http_port 参数，示例改为 8035

```shell
vim /opt/doris/fe/conf/fe.conf
### 设置 http_port = 8035 
bash start_fe.sh --helper 10.1.11.77:9010 --daemon

```
