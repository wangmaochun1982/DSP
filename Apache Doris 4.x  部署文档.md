服务器节点规划表

服务器 IP	       主机名 (Hostname)	   节点角色 (Roles)	        硬件配置	                                              关键端口规划
192.168.221.62	 bigdata-1	        FE (Master) BE	          CPU: 8 Core RAM: 32 GB Disk: 100G (/data)	           FE: 8030/9030 BE: 8040/9050
192.168.221.63	 bigdata-2	        FE (Follower) BE	        CPU: 8 Core RAM: 32 GB Disk: 100G (/data)	           同上
192.168.221.64	 bigdata-3	        FE (Follower) BE	        CPU: 8 Core RAM: 32 GB Disk: 100G (/data)	           同上


本教程涉及的所有核心组件（Apache Doris 4.0.2 安装包、JDK 17 环境包、MySQL 客户端工具）已打包整理完毕。

已上传到服务器中的/data/目录


核心安装包清单

组件名称	               文件名	                                                     说明
Doris 内核	           apache-doris-4.0.2-bin-x64.tar.gz	                          Doris 4.0.2 核心程序包（AVX2版）
Java 环境	             jdk-17.0.12_linux-x64_bin.tar.gz	                            Doris 运行强依赖的 JDK 环境
客户端工具	              mysql-客户端rpm.zip	                                        用于连接 Doris FE 的 MySQL 命令行工具



# 1. 配置 SSH 免密互信
第一步：配置 hosts 映射

在大数据集群运维中，配置 SSH 免密登录 是基础中的基础。它能让你在一个节点（通常是 Master）上通过脚本批量控制所有节点，后续的分发文件、启动集群都会轻松很多。

```shell

[root@bigdata-1 ~]# hostname -i
192.168.221.62
[root@bigdata-1 ~]# 
[root@bigdata-1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.221.62 bigdata-1
192.168.221.63 bigdata-2
192.168.221.64 bigdata-3
[root@bigdata-1 ~]# 
[root@bigdata-1 ~]#

```

第二步：生成密钥 (三台执行)

```shell
## 生成密钥对（一路回车即可，不要设置密码）：
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
```


分发公钥给所有节点（包括自己）： 这一步会要求你输入一次各台机器的 root 密码，输入完这一次，以后就不用输了。


```shell
## 发给自己 (很重要，很多脚本需要ssh localhost)
ssh-copy-id bigdata-1

## 发给 63
ssh-copy-id bigdata-2

## 发给 64
ssh-copy-id bigdata-3
```


第三步：验证免密

```shell
ssh bigdata-1 date
ssh bigdata-2 date
ssh bigdata-3 date

```

第四步：配置三台机器两两互通

（将 62 的权杖复制给别人）：既然 62 已经能免密访问所有人了，你可以直接把 62 的 .ssh 目录分发给 63 和 64。这样它们就拥有了同样的“钥匙”。在 62 节点上执行：

```shell

scp -r ~/.ssh bigdata-2:~/
scp -r ~/.ssh bigdata-3:~/
```

做完以上步骤就完整配齐了SSH免密登录，完整步骤记录如下：

```shell
[root@bigdata-1 ~]# ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
Generating public/private rsa key pair.
Created directory '/root/.ssh'.
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:TeWP7+J1EsKpwGjIIv08u58pOnq3e8YRvPHvHP7MsWw root@bigdata-1
The key's randomart image is:
+---[RSA 2048]----+
|            .    |
|           o     |
|     .    . .    |
| . . .+o o . +   |
|. o o o=S . = o  |
| . + .o .. . o . |
|    +. . .o . + .|
|  o oo+o o.=E= o |
|.o.+=O+  .++B..  |
+----[SHA256]-----+
[root@bigdata-1 ~]# 
[root@bigdata-1 ~]# ssh-copy-id bigdata-1
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
The authenticity of host 'bigdata-1 (192.168.221.62)' can't be established.
ECDSA key fingerprint is SHA256:0bFBqa1O8fkLAVg9OxN7u2BdI7nfxW6h/B5LksdwtHc.
ECDSA key fingerprint is MD5:7e:17:c5:bd:2b:b5:d3:ad:bc:c4:09:b5:89:72:ec:fa.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@bigdata-1's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'bigdata-1'"
and check to make sure that only the key(s) you wanted were added.

[root@bigdata-1 ~]# ssh-copy-id bigdata-2
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
The authenticity of host 'bigdata-2 (192.168.221.63)' can't be established.
ECDSA key fingerprint is SHA256:0bFBqa1O8fkLAVg9OxN7u2BdI7nfxW6h/B5LksdwtHc.
ECDSA key fingerprint is MD5:7e:17:c5:bd:2b:b5:d3:ad:bc:c4:09:b5:89:72:ec:fa.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@bigdata-2's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'bigdata-2'"
and check to make sure that only the key(s) you wanted were added.

[root@bigdata-1 ~]# 
```


# 2. 系统文件句柄数优化
通过修改文件句柄数来控制系统打开文件的数量。通常情况下默认为1024个文件， 通过 ulimit -n 查看目前最大文件句柄数;

Doris 处理海量连接和文件，默认的 1024 远远不够，必须调大。

如果是临时修改可执行 ulimit -n 65535 进行临时生效（重启后失效）。

执行以下命令进行永久配置。三台节点都需执行

第一步：修改 limits.conf

```shell
## 先验证 这个时候输出的应该是1024
ulimit -n

## 备份
cp /etc/security/limits.conf /etc/security/limits.conf.bak

## 追加配置
## 使用 cat >> EOF 的方式比 printf 更清晰，不容易出错
cat >> /etc/security/limits.conf <<EOF
* soft nofile 65536
* hard nofile 65536
* soft nproc 65536
* hard nproc 65536
EOF

## 验证
tail -n 4 /etc/security/limits.conf

```


第二步：修改 sysctl.conf

内核级限制

```shell
## 备份
cp /etc/sysctl.conf /etc/sysctl.conf.bak

## 追加配置
cat >> /etc/sysctl.conf <<EOF
fs.file-max = 2000000
vm.max_map_count = 2000000
EOF

## 让配置立即生效
sysctl -p
```

之后直接使用ulimit -n验证可能并不会生效，而是需要另起一个窗口使用该命令，输出应该是 65536；再使用sysctl vm.max_map_count 输出应该是 vm.max_map_count = 2000000

命令执行记录

```shell
[root@bigdata-1 ~]# cp /etc/security/limits.conf /etc/security/limits.conf.bak
[root@bigdata-1 ~]# 
[root@bigdata-1 ~]# cat >> /etc/security/limits.conf <<EOF
> * soft nofile 65536
> * hard nofile 65536
> * soft nproc 65536
> * hard nproc 65536
> EOF
[root@bigdata-1 ~]# 
[root@bigdata-1 ~]# tail -n 4 /etc/security/limits.conf
* soft nofile 65536
* hard nofile 65536
* soft nproc 65536
* hard nproc 65536
[root@bigdata-1 ~]# 
[root@bigdata-1 ~]# cp /etc/sysctl.conf /etc/sysctl.conf.bak
[root@bigdata-1 ~]# 
[root@bigdata-1 ~]# cat >> /etc/sysctl.conf <<EOF
> fs.file-max = 2000000
> vm.max_map_count = 2000000
> EOF
[root@bigdata-1 ~]# 
[root@bigdata-1 ~]# sysctl -p
fs.file-max = 2000000
vm.max_map_count = 2000000
[root@bigdata-1 ~]# 
[root@bigdata-1 ~]# ulimit -n
1024
[root@bigdata-1 ~]# 
[root@bigdata-1 ~]# sysctl vm.max_map_count
vm.max_map_count = 2000000
[root@bigdata-1 ~]#
[root@bigdata-1 ~]#

```


# 3. 关闭防火墙服务
为确保 Doris 集群节点之间的顺畅通信，需要进行关闭防火墙服务或清空规则。

本文以清空防火墙规则为例，仅关闭 Doris 所需端口的相关规则，其他端口规则保持不变。

请注意⚠️，此操作存在安全风险，仅建议在测试环境或变更窗口期间执行，部署完成后应恢复防火墙并启用相关规则。

三台节点都需执行

```shell

## 查看防火墙命令
systemctl status firewalld

## 开启防火墙命令
systemctl start firewalld

## 关闭防火墙命令
systemctl stop firewalld

## 禁用防火墙命令
systemctl disable firewalld

## 查看系统防火墙规则命令
iptables -L

## 清空系统防火墙规则命令
iptables -F
```

# 4. 禁用 SELinux 安全模块
SeLinux有三种状模式，一个是宽容模式Enforcing，一种是Permissive，还有一种是关闭模式Disabled，其中前两种虽然一个会强制执行一个不会强制执行，但是都会限制部分系统资源访问权限。

所以SeLinux最好设置为关闭状态 --Disabled。

三台节点都需执行

```shell
## 查看SeLinux状态
getenforce

## 临时关闭SeLinux
setenforce 0

## 永久关闭SeLinux
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config

## 重启生效

```


# 5. 禁用 Swap 交换分区
swap指的是一个交换分区或文件，主要是在内存使用存在压力时，触发内存回收，这时可能会将部分内存的数据交换到swap空间。

Linux交换分区会给Doris带来很严重的性能问题，需要在安装之前禁用交换分区，不关闭则会导致内存使用率过高，从而引发FE Memory limit exceeded 后挂掉。 

三台节点都需执行

```shell

## 查看Swappiness版本号
swapon --version

## 【永久关闭】修改 fstab 防止重启复活
sed -i '/swap/s/^/#/' /etc/fstab

## 【内核策略】告诉内核尽量别用
echo "vm.swappiness = 0" >> /etc/sysctl.conf

##【立即关闭】先在当前系统里把它关了，如果不重启的情况下
swapoff -a

## 立即生效
sysctl -p
```

# 6. 禁用透明大页 (Transparent Huge Pages)
为避免 THP 引发内存碎片与不可预期的延迟抖动，需要在运行期禁用并设置开机持久化 三台节点都需执行

```shell
## 立即禁用（当前运行期临时有效）
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

## 永久有效(写入开机持久化) 这里和图中不太一样,以这里为准;
cat >> /etc/rc.d/rc.local <<'EOF'
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
EOF

## 赋权
chmod +x /etc/rc.d/rc.local
```


7. 部署官方标准 Java 运行环境(JDK)
官方 4.x quick start 明确建议 Java 17；

前提条件：所有节点已完成 SSH 免密互通，且 JDK 安装包已上传至主节点 (bigdata-1) 的 /data 目录。

目标版本：Oracle JDK 17.0.12 安装路径：/usr/java/jdk-17.0.12

安装包在本文开头提供 三台节点都需执行

```shell
## 检查当前 Java 版本
java -version
rpm -qa | grep jdk

## 卸载所有旧版 OpenJDK
yum -y remove java*openjdk*

## 再次验证（应提示“未找到命令”）
java -version



cd /data

## 分发至 bigdata-2
scp /data/jdk-17.0.12_linux-x64_bin.tar.gz bigdata-2:/data/

## 分发至 bigdata-3
scp /data/jdk-17.0.12_linux-x64_bin.tar.gz bigdata-3:/data/




```

第四步：解压与安装 (所有节点)

统一安装目录规范为 /usr/java。

```shell
## 创建规范标准安装目录
mkdir -p /usr/java

## 解压安装包到目标目录
tar -zxf /data/jdk-17.0.12_linux-x64_bin.tar.gz -C /usr/java/

## 修正文件权限（确保 root 用户拥有权限）
chown -R root:root /usr/java/jdk-17.0.12/

## 验证目录结构
ll /usr/java/
# 预期输出应包含 jdk-17.0.12 目录


将 JDK 路径写入系统级配置文件 /etc/profile，使其永久生效。

## 写入环境变量
cat >> /etc/profile <<EOF

## jdk
export JAVA_HOME=/usr/java/jdk-17.0.12
export PATH=\$JAVA_HOME/bin:\$PATH
EOF

## 加载配置使其立即生效
source /etc/profile


## 执行
java -version
## 预计输出
java version "17.0.12" 2024-07-16 LTS
Java(TM) SE Runtime Environment (build 17.0.12+8-LTS-286)
Java HotSpot(TM) 64-Bit Server VM (build 17.0.12+8-LTS-286, mixed mode, sharing)



```

# 8. 配置 NTP 时钟同步
Doris 的元数据要求时间精度要小于5000ms，所以所有集群所有机器要进行时钟同步，避免因为时钟问题引发的元数据不一致导致服务出现异常。

比如报错：Clock delta: 86508 ms. between Feeder: 10.20.144.14... and this Replica exceeds max permissible delta: 5000 ms.


<h2>二、Doris 4.x HA 安装部署</h2>

下表详细解析了我们将安装包解压至 /data/doris402 后的标准目录结构。

需要特别注意⚠️的是，虽然安装包内预置了默认的数据存放目录（如 FE 的 doris-meta 和 BE 的 storage），

但在本次生产级部署实践中，我们采用了“程序与数据物理分离”的最佳实践架构：

1.程序侧：/data/doris402 仅作为软件运行目录，存放二进制文件、启动脚本及依赖库。

2.数据侧：通过修改 fe.conf 和 be.conf 配置文件，我们将所有的元数据、业务数据及运行日志强制重定向到了 /data/doris/ 目录下。

<img width="667" height="428" alt="image" src="https://github.com/user-attachments/assets/b0084cca-5066-44a3-803f-5583d161a581" />


# 1. 硬件兼容性预检 (AVX2)

Apache Doris 4.0 的核心——向量化执行引擎，本文使用的是 x64(avx2) 二进制包,该包底层强依赖 CPU 的 AVX2 指令集以实现极致的并行计算性能。

如果 CPU 不支持 AVX2，后端节点 (BE) 将因无法识别指令而直接崩溃，导致服务不可用。

因此，在部署前必须执行以下命令进行硬件兼容性自检：

cat /proc/cpuinfo | grep flags | grep avx2 | head -n 1；

若不支持请改用 x64(no avx2) 包。

可以通过自动判断的命令来一件检测：

```shell

if grep -q avx2 /proc/cpuinfo; then echo "✅ CPU 支持 AVX2"; else echo "❌ 不支持 AVX2"; fi

```

# 2. 软件解压与目录初始化

为了规范安装路径并避免多层目录嵌套，我们使用该命令创建目标文件夹 doris402，并在解压时通过 --strip-components=1 参数自动剥离压缩包自带的顶层版本目录，直接将核心程序文件释放到指定位置。


```shell
mkdir -p doris402 && tar -zxf apache-doris-4.0.2-bin-x64.tar.gz -C doris402 --strip-components=1


```

#  3. 批量创建数据存储与日志目录

该命令利用 Shell 循环批量远程连接所有集群节点，一次性建立统一规划的元数据、日志及存储等标准目录结构，并递归赋予最高读写权限（777），以防止服务启动时因权限不足而报错。

```shell
for host in bigdata-1 bigdata-2 bigdata-3; do
   echo ">>> 正在处理 $host ..."
   ssh $host "mkdir -p /data/doris/{doris_meta,fe_log,storage,be_log,jdbc_drivers} && chmod -R 755 /data/doris"
done
```


# 4. FE (控制节点) 配置文件fe.conf调优

```plaintext

4.1.配置优化清单：

类别	              配置项 (Property)	                    修改值 (Value)	                                         修改原因与说明 (Rationale)
1. 路径配置	        LOG_DIR                               /data/doris/fe_log	                                    日志目录分离。 将日志移出安装目录，保持整洁并防止升级误删。
                    meta_dir	                           /data/doris/doris_meta	                                元数据目录分离。 FE 最核心的数据存储位置，必须独立存放以确保数据安全。
                    audit_log_dir	                       /data/doris/fe_log	                                    审计日志目录。 统一存放 SQL 审计日志，便于管理。
2. 网络配置	        priority_networks	                   192.168.221.0/24	                                      强制绑定内网网段。 防止绑定到 Docker (172.x) 等虚拟网卡，确保多节点通信正常。
3. Java 环境	      JAVA_HOME                     	     /usr/java/jdk-17.0.12	                                  指定 JDK 路径。 防止因环境变量缺失导致服务启动失败。
4. 资源与内存 (JVM)	JAVA_OPTS_FOR_JDK_17	              -Xmx8192m -Xms8192m (需确保整行配置无换行)	                **JVM 堆内存优化 (32GB 机器标准)**。 FE 和 BE 混部时，固定 FE 内存为 8GB，防止 FE 占用过多内存导致 BE (数据节点) 内存不足而崩溃 (OOM)。
5. 导入与并发	       max_routine_load_task_num_per_be	    10	                                                  提升 Kafka 导入并发。 默认为 5。增加该值可允许单个 BE 节点同时处理更多的 Kafka 分区任务，提升实时数据写入吞吐量。
6. 业务兼容性	       lower_case_table_names	               1	                                                  表名大小写不敏感。 兼容 MySQL 习惯。注意⚠️：仅初始化前可改，初始化后不可逆。
                     enable_sql_audit	                     true	                                                开启 SQL 审计。 记录查询历史，用于排查慢查询和安全审计。
                     qe_max_connection	                   1600	                                                 提升最大连接数。 默认 1024。调大以支持 Flink、Web 后端、BI 等更多客户端并发连接。
7. 性能与超时	        bdbje_heartbeat_timeout_second      	60	                                                 放宽心跳超时。 防止网络/磁盘波动导致 FE 误判死亡发生主从切换。
                       remote_fragment_exec_timeout_ms	   15000	                                                远程片段超时。 防止僵死查询长时间占用 BE 资源。        
```



4.2.priority_networks 配置项说明

