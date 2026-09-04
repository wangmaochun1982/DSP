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


```sql

-- ============================================================
-- 1. 创建数仓各层数据库
-- ============================================================
CREATE DATABASE IF NOT EXISTS ods_db;
CREATE DATABASE IF NOT EXISTS dwd_db;
CREATE DATABASE IF NOT EXISTS dws_db;

-- ============================================================
-- 2. DDL 建表阶段
-- ============================================================

-- 【ODS层】原始订单明细表 (明细模型)
CREATE TABLE IF NOT EXISTS ods_db.ods_order_detail (
    `order_id` BIGINT NOT NULL COMMENT "订单ID",
    `order_time` DATETIME NOT NULL COMMENT "下单时间",
    `user_id` BIGINT NOT NULL COMMENT "用户ID",
    `goods_id` INT NOT NULL COMMENT "商品ID",
    `goods_num` INT COMMENT "购买数量",
    `order_amount` DECIMAL(12, 2) COMMENT "订单金额",
    `pay_amount` DECIMAL(12, 2) COMMENT "实付金额",
    `order_status` TINYINT COMMENT "订单状态: 1-待支付, 2-已支付, 3-已取消",
    `create_time` DATETIME COMMENT "数据抽取时间"
)
DUPLICATE KEY(`order_id`, `order_time`)
AUTO PARTITION BY RANGE (date_trunc(`order_time`, 'day')) ()
DISTRIBUTED BY HASH(`order_id`) BUCKETS 8
PROPERTIES (
    "replication_num" = "1"
);

-- 【DWD层】商品维度表 (主键模型)
CREATE TABLE IF NOT EXISTS dwd_db.dim_goods (
    `goods_id` INT NOT NULL COMMENT "商品ID",
    `goods_sn` VARCHAR(64) NOT NULL COMMENT "商品编码",
    `goods_name` VARCHAR(100) NOT NULL COMMENT "商品名称",
    `category_name` VARCHAR(50) COMMENT "品类名称",
    `brand_name` VARCHAR(50) COMMENT "品牌名称",
    `price` DECIMAL(10, 2) COMMENT "当前售价",
    `update_time` DATETIME COMMENT "更新时间"
)
UNIQUE KEY(`goods_id`)
DISTRIBUTED BY HASH(`goods_id`) BUCKETS 4
PROPERTIES (
    "replication_num" = "1"
);

-- 【DWD层】用户维度表 (主键模型)
CREATE TABLE IF NOT EXISTS dwd_db.dim_user (
    `user_id` BIGINT NOT NULL COMMENT "用户ID",
    `user_name` VARCHAR(50) NOT NULL COMMENT "用户名称",
    `gender` TINYINT COMMENT "性别: 1-男, 2-女",
    `city` VARCHAR(50) COMMENT "所在城市",
    `register_time` DATETIME COMMENT "注册时间"
)
UNIQUE KEY(`user_id`)
DISTRIBUTED BY HASH(`user_id`) BUCKETS 4
PROPERTIES (
    "replication_num" = "1"
);

-- 【DWD层】交易订单事实表 (明细模型)
CREATE TABLE IF NOT EXISTS dwd_db.fact_order (
    `order_id` BIGINT NOT NULL COMMENT "订单ID",
    `order_time` DATETIME NOT NULL COMMENT "下单时间",
    `user_id` BIGINT NOT NULL COMMENT "用户ID",
    `goods_id` INT NOT NULL COMMENT "商品ID",
    `goods_num` INT COMMENT "购买数量",
    `order_amount` DECIMAL(12, 2) COMMENT "订单金额",
    `pay_amount` DECIMAL(12, 2) COMMENT "实付金额"
)
DUPLICATE KEY(`order_id`, `order_time`, `user_id`, `goods_id`)
AUTO PARTITION BY RANGE (date_trunc(`order_time`, 'day')) ()
DISTRIBUTED BY HASH(`order_id`) BUCKETS 8
PROPERTIES (
    "replication_num" = "1"
);

-- 【DWS层】用户日度消费汇总表 (聚合模型)
CREATE TABLE IF NOT EXISTS dws_db.dws_user_daily_sales (
    `dt` DATE NOT NULL COMMENT "统计日期",
    `user_id` BIGINT NOT NULL COMMENT "用户ID",
    `total_amount` DECIMAL(12, 2) SUM DEFAULT "0.00" COMMENT "当日总消费金额",
    `order_count` BIGINT SUM DEFAULT "0" COMMENT "当日下单总笔数",
    `last_order_time` DATETIME MAX COMMENT "最后一次下单时间"
)
AGGREGATE KEY(`dt`, `user_id`)
DISTRIBUTED BY HASH(`user_id`) BUCKETS 8
PROPERTIES (
    "replication_num" = "1"
);


-- ============================================================
-- 3. 数据写入与流转阶段
-- ============================================================

-- Step 1: 写入基础维度数据 (DWD)
INSERT INTO dwd_db.dim_goods VALUES
(501, 'G-1001', '无线蓝牙耳机', '3C数码', '品牌A', 199.00, '2026-09-03 16:00:00'),
(502, 'G-1002', '机械键盘', '电脑外设', '品牌B', 299.50, '2026-09-03 16:00:00');

INSERT INTO dwd_db.dim_user VALUES
(10001, '张三', 1, '厦门', '2026-01-01 10:00:00'),
(10002, '李四', 2, '深圳', '2026-02-15 11:30:00');

-- Step 2: 写入 ODS 贴源层测试数据 (包含取消的垃圾数据)
INSERT INTO ods_db.ods_order_detail VALUES
(20260903001, '2026-09-03 10:15:00', 10001, 501, 1, 199.00, 199.00, 2, '2026-09-03 16:00:00'),
(20260903002, '2026-09-03 14:20:00', 10001, 502, 1, 299.50, 299.50, 2, '2026-09-03 16:00:00'),
(20260903003, '2026-09-03 15:00:00', 10002, 501, 2, 398.00, 398.00, 2, '2026-09-03 16:00:00'),
(20260903004, '2026-09-03 16:00:00', 10002, 502, 1, 299.50, 0.00, 3, '2026-09-03 16:00:00');

-- Step 3: ODS ➔ DWD 过滤清洗 (只保留已支付 order_status = 2)
INSERT INTO dwd_db.fact_order
SELECT 
    order_id,
    order_time,
    user_id,
    goods_id,
    goods_num,
    order_amount,
    pay_amount
FROM ods_db.ods_order_detail
WHERE order_status = 2;

-- Step 4: DWD ➔ DWS 轻度汇总
INSERT INTO dws_db.dws_user_daily_sales
SELECT 
    DATE(order_time) AS dt,
    user_id,
    pay_amount AS total_amount,
    1 AS order_count,
    order_time AS last_order_time
FROM dwd_db.fact_order;


-- ============================================================
-- 4. 结果验证与星型模型关联查询
-- ============================================================

-- 校验 ODS 原始层 (4条数据)
SELECT * FROM ods_db.ods_order_detail;

-- 校验 DWD 事实层 (3条有效数据)
SELECT * FROM dwd_db.fact_order;

-- 校验 DWS 汇总层 (2条按用户聚合的数据)
SELECT * FROM dws_db.dws_user_daily_sales;

-- 维度-事实星型模型跨库分析查询
SELECT 
    u.city AS "城市",
    g.category_name AS "商品品类",
    COUNT(DISTINCT f.order_id) AS "订单总量",
    SUM(f.pay_amount) AS "总销售额"
FROM dwd_db.fact_order f
JOIN dwd_db.dim_user u ON f.user_id = u.user_id
JOIN dwd_db.dim_goods g ON f.goods_id = g.goods_id
GROUP BY u.city, g.category_name;
```

