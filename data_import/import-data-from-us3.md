# 从对象存储 US3 导入数据

AI Database 支持使用 `INSERT INTO ... SELECT * FROM read_xxx()` 从 US3 对象存储直接导入数据，可通过以下语法高效加载外部文件至数据库表中：

> 说明
> 关于 US3 访问凭证（`accessid` 和 `secret`）的获取和配置，请参考[访问US3数据](DuckDB_Instruction/access-us3.md)文档。


## 基本语法

```sql
INSERT INTO <target_table>
SELECT 
  r['<conlumn_1>']::<data_type>,
  r['<conlumn_2>']::<data_type>,
  ...
FROM read_xxx('s3://<endpoint>/<bucket>/<path> accessid=XXXX secret=YYYY') r;
```

**说明：** `read_xxx` 为数据读取函数，如 `read_parquet`、`read_csv` 等。

## 数据类型转换

在从 US3 导入数据时，需对字段进行显式类型转换，以确保数据类型与目标表结构一致。可使用 `::` 语法进行类型指定：

```sql
r['column_name']::integer    -- 整数
r['column_name']::varchar    -- 字符串
r['column_name']::float      -- 浮点数
r['column_name']::date       -- 日期
r['column_name']::timestamp  -- 时间戳
```

## 示例

### 示例 1：导入 Parquet 数据

以下示例展示如何从 US3 存储中的 Parquet 文件 读取数据并导入到数据库表中：

```sql
-- 创建目标表
CREATE TABLE NATION (
    N_NATIONKEY  INTEGER NOT NULL,
    N_NAME       VARCHAR NOT NULL,
    N_REGIONKEY  INTEGER NOT NULL,
    N_COMMENT    VARCHAR,
    N_DUMMY      VARCHAR
);

-- 导入数据
INSERT INTO nation 
SELECT 
  r['n_nationkey']::integer,
  r['n_name']::varchar,
  r['n_regionkey']::integer,
  r['n_comment']::varchar,
  r['n_dummy']::varchar
FROM read_parquet('
    s3://s3-cn-gd.ufileos.com/my-bucket/path/nation.parquet
    accessid={MY_ACCESS_KEY}
    secret={MY_SECRET_KEY}') r;

```

### 示例 2：导入无表头 CSV 数据

以下示例展示如何从 US3 存储中的无表头 CSV 文件 导入数据，并指定分隔符为 `|`：

```sql
-- 创建目标表
CREATE TABLE ORDERS (
    O_ORDERKEY      INTEGER NOT NULL,
    O_CUSTKEY       INTEGER NOT NULL,
    O_ORDERSTATUS   VARCHAR(1) NOT NULL,
    O_TOTALPRICE    DECIMAL(15,2) NOT NULL,
    O_ORDERDATE     DATE NOT NULL
);

-- 导入数据（指定分隔符为 '|'）
INSERT INTO ORDERS 
SELECT 
  r['column0']::integer,
  r['column1']::integer,
  r['column2']::varchar,
  r['column3']::decimal(15,2),
  r['column4']::date
FROM read_csv('
    s3://s3-cn-gd.ufileos.com/my-bucket/path/order_data/orders.tbl.*
    accessid={MY_ACCESS_KEY}
    secret={MY_SECRET_KEY}', delim=>'|') r;
```

**说明：**

- `read_parquet()` 与 `read_csv()` 函数会自动解析 US3 文件并流式读取数据。

- 可根据文件格式调整解析函数与参数（如分隔符、是否包含表头等）。

- 为确保安全性，请妥善管理访问凭证 `accessid` 与 `secret`。


## 注意事项

- **列顺序**：确保 `SELECT` 中的列顺序与目标表字段顺序完全一致，以避免数据错位。

- **数据类型**：源数据类型需与目标表字段类型兼容，必要时使用显式类型转换。

- **权限要求**：执行导入操作的用户必须具备目标表的 `INSERT` 权限。

- **性能优化**：针对大批量数据导入，建议采用分批次导入方式，以减少事务压力并提升导入效率。
