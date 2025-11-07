# 访问实例

AI Database 提供多种连接方式，以满足不同场景下的访问需求。默认情况下，系统通过 **内网连接** 提供高安全性与低延迟的数据访问。同时，AI Database 也支持 **外网访问配置**，方便开发者、测试人员或第三方应用在外部环境中安全地连接数据库，实现灵活的数据交互与管理。


## 内网连接

AI Database 支持通过多种方式在 **同一 VPC** 内安全访问数据库。以下示例展示了在不同语言和工具环境下的典型连接方式。

### 方式一：通过 psql 客户端访问

在与 AI Database 位于同一 VPC 的云主机上，安装 PostgreSQL 客户端 psql，并使用以下命令进行连接：

```shell
psql 'postgres://{username}:{password}@{db_host}/{database}?sslmode=require&options=project%3D{instance_id}'
```
**说明：**

- `%3D` 为 URL 编码后的 `=` 符号。
- 若 `username` 或 `password` 中包含特殊字符，请进行 URL 编码。

### 方式二：通过 Python Psycopg 访问

安装 psycopg2 依赖 `pip install psycopg2-binary`。

安装依赖：

```python
import psycopg2

conn = psycopg2.connect(
    host="{db_host}",
    port="{port}",
    user="{username}",
    password="{password}",
    dbname="{database}",
    options="project={instance_id}"
)

cur = conn.cursor()
cur.execute("SELECT version();")
result = cur.fetchone()
print("PostgreSQL version:", result[0])

cur.close()
conn.close()
```

### 方式三：通过 Java JDBC 访问

安装 PostgreSQL 驱动包（版本需 ≥ 42.2.6）。

使用 Connection URL 连接：

```java
String url = "jdbc:postgresql://{db地址}/{database}?user={username}&password={password}&options=project%3D{实例_资源id}";
Connection conn = DriverManager.getConnection(url);
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT version();");

// 处理查询结果
...

// 关闭连接
...
```

使用 Properties 参数连接：

```java
String url = "jdbc:postgresql://{db_host}/{database}";
Properties props = new Properties();
props.setProperty("user", "{username}");
props.setProperty("password", "{password}");
props.setProperty("options", "project={instance_id}");

Connection conn = DriverManager.getConnection(url, props);
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT version();");


// 处理查询结果
...

// 关闭连接
...
```

> 提示<br>
> 所有内网访问均需确保云主机与 AI Database 实例处于同一 VPC 网络内，并开放相应端口（默认为 5432）以确保连通性。


## 外网访问

### 方式一：通过 postgres client psql 访问

通过 Nginx 配置反向代理来进行外网访问。

1. 创建一台与 AI Database 实例处于同一子网下的外网云主机。

2. 在云主机上安装 Nginx。

    ```shell
    yum install -y nginx
    ```
3. 配置 nginx 代理。

    3.1 编辑 `/etc/nginx/nginx.conf`，增加 `stream` 配置。

    ```shell
    stream {
        log_format proxy '$remote_addr [$time_local] '
                    '$protocol $status $bytes_sent $bytes_received '
                    '$session_time "$upstream_addr" '
                    '"$upstream_bytes_sent" "$upstream_bytes_received" "$upstream_connect_time"';

        access_log /var/log/nginx/tcp-access.log proxy;
        open_log_file_cache off;

        # 统一放置，方便管理
        include /etc/nginx/tcpConf.d/*.conf;
    }
    ```

    3.2 创建 `/etc/nginx/tcpConf.d/` 目录。

    ```shell
    mkdir -p /etc/nginx/tcpConf.d/
    ```
    3.3 编辑 `/etc/nginx/tcpConf.d/maxir.conf` 配置文件。

    ```shell
    upstream maxir_tcp5432 {
        server ${MAXIR_IP}:5432; // 其中MAXIR_IP为MAXIR的接入地址ip
    }
    server {
        listen 5432;
        proxy_connect_timeout 8s;
        proxy_timeout 24h;
        proxy_pass maxir_tcp5432;
    }
    ```
    3.4 进行应用配置。

    ```shell
    nginx -s reload
    ```

4. 开通云主机外网防火墙端口 `5432`。

5. 通过 pg 客户端进行登录验证。

    ```shell
    psql -h ${IP} -p 5432 -U ${用户名} -d postgres
    ```

    其中，`${IP}` 为 ngnix 服务器的 IP 地址。


### 通过 GUI 客户端 pgAdmin 访问

可使用 **pgAdmin** 等可视化工具进行连接和管理。注册服务器时，请按照以下步骤配置：

1. 基本连接参数

    - 主机地址：填写 Nginx 公网 IP
    - 端口：默认 5432
    - 用户名与密码：与实例一致

2. 连接参数配置

    - SSL 模式（sslmode）：选择 require
    - Options：填写 project={instance_id}

完成配置后，即可通过 pgAdmin 访问外网数据库实例，实现远程数据管理与查询。

![alidocs.png](/images/alidocs.png)