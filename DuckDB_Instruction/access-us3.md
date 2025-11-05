# 访问 US3 数据

AI Database 基于 DuckDB 引擎，支持直接从 US3 对象存储 读取多种数据格式（包括 CSV、Parquet、JSON 等）。系统内置智能缓存机制，可自动识别重复访问数据并进行本地加速，大幅提升 US3 数据读取性能。

## 基本用法

### 方式一：在路径中直接指定访问密钥

通过在路径中附加访问凭证参数，可直接从 US3 读取数据文件：

```sql
SELECT * FROM read_csv('s3://<endpoint>/<bucket>/<prefix>/file.csv accessid=xxx secret=yyy');

SELECT * FROM read_parquet('s3://<endpoint>/<bucket>/<prefix>/file.parquet accessid=xxx secret=yyy');

SELECT * FROM read_json('s3://<endpoint>/<bucket>/<prefix>/file.json accessid=xxx secret=yyy');
```

### 方式二：预配置密钥后直接访问

[预访问密钥](/DuckDB 手册/manage-keys.md)后，即可直接通过路径访问，无需在查询中显式填写凭证：

```sql
SELECT * FROM read_csv('s3://<bucket>/<prefix>/file.csv');

SELECT * FROM read_parquet('s3://<bucket>/<prefix>/file.parquet');

SELECT * FROM read_json('s3://<bucket>/<prefix>/file.json');
```

### 读取目录（需使用通配符 `*`）

可通过通配符 `*` 读取指定目录下的多个文件，实现批量数据加载：

```sql
SELECT * FROM read_parquet('s3://<endpoint>/<bucket>/<prefix>/* accessid=xxx secret=yyy');
```

## UCloud Endpoint 配置

| 类型 | Endpoint |
| :- | :- |
| 内网 | `internal.s3-cn-gd.ufileos.com` |
| 外网 | `s3-cn-sh2.ufileos.com` |

> 说明
> 更多 Endpoint 信息参考[地域和域名](https://docs.ucloud.cn/ufile/introduction/region)。

## 高级选项

### CSV 格式控制

AI Database 支持自动检测分隔符，也可手动指定：

```sql
SELECT * FROM read_csv('s3://...', 
  delim=>'|', 
  escape=>'"',
  quote=>'"',
  header=>true
);

```

### JSON 错误处理

在解析 JSON 文件时，可通过 `ignore_errors` 参数忽略格式异常的记录，避免导入中断：

```sql
SELECT * FROM read_json('s3://...', ignore_errors=>true);
```

### Parquet 扩展功能

Parquet 文件支持多项高级选项，用于增强可读性与灵活性：

```sql
SELECT * FROM read_parquet('s3://...', 
  filename=>true,              -- 返回文件名
  file_row_number=>true,       -- 返回行号
  hive_partitioning=>true,     -- 解析 Hive 分区路径
  union_by_name=>true          -- 按列名合并不同 Schema
);
```

### Delta / Iceberg 表

AI Database 还支持直接读取 Delta Lake 与 Apache Iceberg 表，实现与主流数据湖格式的无缝集成：

```sql
SELECT * FROM delta_scan('s3://<table_base_path>');
SELECT * FROM iceberg_scan('s3://<table_base_path>');
```