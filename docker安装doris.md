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



# [root@localhost doris]# cat Dockerfile 

```plaintext

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
