```shell

# 安装 yum-utils
dnf install -y yum-utils
# 添加 Docker CE 官方软件源
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce
yum -y install docker-ce
docker ps
mkdir -p /etc/docker
 vi /etc/docker/daemon.json
root@localhost ~]# cat /etc/docker/daemon.json 
{
    "registry-mirrors": [
      "https://hub-mirror.c.163.com",
      "https://mirror.ccs.tencentyun.com",
      "https://mirrors.aliyun.com",
      "https://docker.1ms.run"
    ]
}



systemctl daemon-reload
systemctl enable docker
systemctl restart docker
docker images
mkdir -p /data/doris


[root@localhost doris]# ll
total 3735272
drwxr-xr-x  5 root root         47 Sep  3 14:09 apache-doris-4.0.8-bin-x64
-rw-r--r--  1 root root 3824901105 Sep  3 14:09 apache-doris-4.0.8-bin-x64.tar.gz
drwxr-xr-x 13 root root        187 Sep  3 14:12 be
-rw-r--r--  1 root root       1023 Sep  3 16:17 docker-compose.yml
-rw-r--r--  1 root root       1412 Sep  3 15:43 Dockerfile
-rw-r--r--  1 root root        395 Sep  3 15:56 entrypoint.sh
drwxr-xr-x 14 root root       4096 Sep  3 14:12 fe
drwxr-xr-x  4 root root         39 Sep  3 14:09 storage




```

```shell

# 1. 进入指定目录
cd /data/doris

# 2. 解压下载的 Doris 二进制文件
tar -zxvf apache-doris-4.0.8-bin-x64.tar.gz

# 3. 将解压出的子目录移动到当前目录根下，生成 ./fe 和 ./be
mv apache-doris-4.0.8-bin-x64/fe ./fe
mv apache-doris-4.0.8-bin-x64/be ./be

# 4. 创建运行与持久化所需的相关目录
mkdir -p /data/doris/storage/fe-meta
mkdir -p /data/doris/storage/be-storage
mkdir -p /data/doris/fe/logs
mkdir -p /data/doris/be/logs

echo "JAVA_HOME=/usr/lib/jvm/jdk-17.0.20.1-bellsoft-x86_64" >> /data/doris/fe/conf/fe.conf
echo "JAVA_HOME=/usr/lib/jvm/jdk-17.0.20.1-bellsoft-x86_64" >> /data/doris/be/conf/be.conf

# 创建入口脚本文件并写入脚本内容（参见前文文件内容）
touch entrypoint.sh
chmod +x entrypoint.sh

# 创建 Dockerfile 和 docker-compose.yml 文件
touch Dockerfile
touch docker-compose.yml
```


# [root@localhost doris]# cat Dockerfile 

```dockerfile

# 使用基于 Rocky Linux 的 Liberica JDK 镜像作为基础镜像
#FROM bellsoft/liberica-openjdk-rocky:17
# 使用基于 Rocky Linux 的 Liberica JDK 镜像作为基础镜像
FROM docker.1ms.run/bellsoft/liberica-openjdk-rocky:17

LABEL maintainer="Doris Custom Image" \
      description="Apache Doris 4.0.8 Binary Runtime Image based on Liberica OpenJDK Rocky Linux"

# 环境变量设置
ENV DORIS_HOME=/opt/apache-doris \
    JAVA_HOME=/usr/lib/jvm/liberica-openjdk-17 \
    PATH=$PATH:/opt/apache-doris/fe/bin:/opt/apache-doris/be/bin

# 安装 Doris 运行必需的工具（精简版 Rocky 镜像使用 microdnf）
RUN microdnf install -y --setopt=install_weak_deps=0 \
        net-tools \
        which \
        procps-ng \
        findutils \
        hostname \
    && microdnf clean all \
    && rm -rf /var/cache/yum

# 创建程序及数据存储路径
RUN mkdir -p ${DORIS_HOME}/fe ${DORIS_HOME}/be

# 将本地解压好的 FE 和 BE 复制到镜像中
COPY fe ${DORIS_HOME}/fe
COPY be ${DORIS_HOME}/be

# 复制启动脚本
COPY entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

WORKDIR ${DORIS_HOME}

# FE 默认端口: 8030 (HTTP), 9020 (RPC), 9030 (MySQL), 9010 (Edit Log)
# BE 默认端口: 9050 (Heartbeat), 9060 (Backbone), 8040 (HTTP), 8000 (BRPC)
EXPOSE 8030 9020 9030 9010 9050 9060 8040 8000

ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["fe"]


```


# [root@localhost doris]# cat docker-compose.yml 

```yaml
services:
  doris-fe:
    image: apache-doris:4.0.8
    container_name: doris-fe
    restart: always
    network_mode: "host"
    environment:
      - TZ=Asia/Shanghai
    command: ["fe"]
    volumes:
      - /data/doris/storage/fe-meta:/opt/apache-doris/fe/doris-meta
      - /data/doris/fe/logs:/opt/apache-doris/fe/log
      - /data/doris/fe/conf:/opt/apache-doris/fe/conf # 加上这行挂载 FE 配置
    ulimits:
      nofile:
        soft: 65536
        hard: 65536

  doris-be:
    image: apache-doris:4.0.8
    container_name: doris-be
    restart: always
    network_mode: "host"
    environment:
      - TZ=Asia/Shanghai
    command: ["be"]
    depends_on:
      - doris-fe
    volumes:
      - /data/doris/storage/be-storage:/opt/apache-doris/be/storage
      - /data/doris/be/logs:/opt/apache-doris/be/log
      - /data/doris/be/conf:/opt/apache-doris/be/conf # 加上这行挂载 BE 配置
    ulimits:
      nofile:
        soft: 655360
        hard: 655360
      memlock:
        soft: -1
        hard: -1
```

# [root@localhost doris]# cat entrypoint.sh 

```shell
#!/bin/bash
set -e

ROLE=$1

if [ "$ROLE" = "fe" ]; then
    echo "Starting Doris Frontend (FE)..."
    exec /opt/apache-doris/fe/bin/start_fe.sh
elif [ "$ROLE" = "be" ]; then
    echo "Starting Doris Backend (BE)..."
    exec /opt/apache-doris/be/bin/start_be.sh 
else
    echo "Usage: docker run <image> [fe|be]"
    echo "Defaulting to FE..."
    exec /opt/apache-doris/fe/bin/start_fe.sh
fi
```


```shell

docker build -t apache-doris:4.0.8 .

sysctl -w vm.max_map_count=2000000
echo "vm.max_map_count=2000000" >> /etc/sysctl.conf
sysctl -p


docker run --rm docker.1ms.run/bellsoft/liberica-openjdk-rocky:17 /bin/bash -c "echo \$JAVA_HOME; type -p java"


# 写入 FE 配置文件
echo "JAVA_HOME=/usr/lib/jvm/jdk-17.0.10-bellsoft-x86_64" >> /data/doris/fe/conf/fe.conf

# 写入 BE 配置文件（部分版本的 BE 开启 Java UDF 也需要绑定 JAVA_HOME）
echo "JAVA_HOME=/usr/lib/jvm/jdk-17.0.10-bellsoft-x86_64" >> /data/doris/be/conf/be.conf

 swapoff -a
sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab

docker compose up -d
```


```shell

dnf install -y https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm

dnf install -y mysql-community-client --nogpgcheck

mysql -h 127.0.0.1 -P 9030 -u root

# 1. 连接 Doris FE（默认无密码）
mysql -h 127.0.0.1 -P 9030 -u root

# 2. 注册 BE 节点（在 mysql> 提示符中执行，请替换为你的真实宿主机 IP）
ALTER SYSTEM ADD BACKEND "<宿主机IP>:9050";

# 3. 验证连接状态（Alive 应显示为 true）
SHOW BACKENDS\G

SHOW FRONTENDS\G

-- 修改当前 root 用户的密码（请将 YourStrongPassword123! 替换为实际密码）
SET PASSWORD FOR 'root'@'%' = PASSWORD('YourStrongPassword123!');


```

```shell

mysql -uroot -P9030 -h127.0.0.1 -e 'SELECT `host`, `join`, `alive` FROM frontends()'
mysql -uroot -P9030 -h127.0.0.1 -e 'SELECT `host`, `alive` FROM backends()'


[root@localhost doris]# mysql -uroot -P9030 -h127.0.0.1 -e 'SELECT `host`, `join`, `alive` FROM frontends()'
+------------+------+-------+
| host       | join | alive |
+------------+------+-------+
| 172.17.0.1 | true | true  |
+------------+------+-------+
[root@localhost doris]# mysql -uroot -P9030 -h127.0.0.1 -e 'SELECT `host`, `alive` FROM backends()'
\
+------------+-------+
| host       | alive |
+------------+-------+
| 10.2.1.244 |     1 |
+------------+-------+



```


