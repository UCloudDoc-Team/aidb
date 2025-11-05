# 密钥管理

AI Database 支持通过工具函数或外部数据包装器（Foreign Data Wrapper）来配置 DuckDB 密钥，以满足不同场景的需求。

## 方式一：工具函数配置密钥

工具函数是最简单快捷的凭证配置方式，适合一般的使用场景。

### 基础配置示例

以下是配置 US3（UCloud 对象存储）访问密钥的基础示例：

```sql
-- 基础 S3 密钥配置（最常用）
SELECT duckdb.create_simple_secret(
    type := 'S3',
    key_id := 'your_access_key_id',
    secret := 'your_secret_access_key',
    region := 'cn-bj',
    endpoint := 'internal.s3-cn-bj.ufileos.com',
    use_ssl := 'false'
);
```

### 完整参数说明

`create_simple_secret` 函数支持以下参数：

```postgresql
SELECT duckdb.create_simple_secret(
    type          := 'S3',          -- 必填：存储类型，可选值为 S3、GCS、R2
    key_id        := 'access_key_id',  -- 必填：访问密钥 ID
    secret        := 'your_secret',    -- 必填：密钥
    session_token := 'token',          -- 可选：临时会话令牌
    region        := 'cn-bj',          -- 可选：存储区域
    url_style     := 'path',           -- 可选：URL 样式（path/vhost）
    endpoint      := 'xxx.com',        -- 可选：自定义端点地址
    scope         := 's3://bucket/prefix/',  -- 可选：密钥作用域，限制密钥生效路径
    validation    := 'true',           -- 可选：是否验证密钥
    use_ssl       := 'true'            -- 可选：是否使用 SSL（默认 true）
);
```

### 使用密钥访问数据

创建成功后，系统会自动使用配置的密钥访问对应的 S3 路径：

```sql
-- 读取 Parquet 文件
SELECT * FROM read_parquet('s3://my-bucket/data/file.parquet');
```
> **注意**：使用工具函数创建的凭证仅对**当前用户**有效，其他用户无法使用该凭证访问 S3。

## 方式二：Foreign Data Wrapper 配置（多用户场景）

当需要为其他用户配置凭证时，使用 Foreign Data Wrapper 方式可以实现更灵活的权限管理。

### 创建 Server

首先创建一个 Server 对象来定义 S3 连接配置：

```sql
CREATE SERVER my_s3_secret 
TYPE 's3' 
FOREIGN DATA WRAPPER duckdb 
OPTIONS (
    region 'cn-bj',                          -- 存储区域
    endpoint 'internal.s3-cn-bj.ufileos.com', -- 端点地址
    url_style 'path',                        -- URL 样式
    use_ssl 'false',                         -- 是否使用 SSL
    scope 's3://my-bucket/'                  -- 密钥作用域（可选）
);

```

### 为用户创建 User Mapping

然后为特定用户创建凭证映射：

```sql
-- 为单个用户创建凭证
CREATE USER MAPPING FOR alice 
SERVER my_s3_secret
OPTIONS (
    KEY_ID 'alice_access_key_id', 
    SECRET 'alice_secret_access_key'
);

-- 为另一个用户创建凭证
CREATE USER MAPPING FOR bob 
SERVER my_s3_secret
OPTIONS (
    KEY_ID 'bob_access_key_id', 
    SECRET 'bob_secret_access_key'
);

```

### 查看和管理凭证

```sql
-- 查看已创建的 Server
SELECT * FROM pg_foreign_server;

-- 删除 User Mapping
DROP USER MAPPING FOR alice SERVER my_s3_secret;

-- 删除 Server
DROP SERVER my_s3_secret;
```

## 最佳实践

### 使用 scope 限制访问范围

通过 `scope` 参数可以限制密钥的作用范围，提高安全性：

```sql
SELECT duckdb.create_simple_secret(
    type := 'S3',
    key_id := 'your_key_id',
    secret := 'your_secret',
    endpoint := 'internal.s3-cn-bj.ufileos.com',
    scope := 's3://my-bucket/data/public/'  -- 仅限访问此路径
);

```

### 内网访问

在 UCloud 环境内访问 US3 时，建议使用内网域名以获得更好的性能和更低的成本

```sql
-- 内网域名（推荐）
endpoint := 'internal.s3-cn-bj.ufileos.com'

-- 外网端点
endpoint := 's3-cn-bj.ufileos.com'
```

US3 访问域名文档：[https://docs.ucloud.cn/ufile/introduction/region](https://docs.ucloud.cn/ufile/introduction/region)