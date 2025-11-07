# 向量表

向量表即存储向量数据的表。本节将通过具体示例演示如何创建向量表。所有示例中使用的语法均遵循 SQL:1999 标准，并兼容部分 SQL:2003 特性。

## 语法

```sql
CREATE TABLE <table_name>(
  C1 <data_type>,
  C2 <data_type>,
  ......,
  CN <vector_type>(128),
  PRIMARY KEY (<col_name1>[, <col_name2>, ... , <col_nameN>])
);
```

表中的向量列（`_<vector_type>_`）括号中的数字表示向量的维度，即 `<vector_type>(128)` 表示向量维度为 128。下表描述了 AI Database 支持的向量列数据类型：

| 类型名称 | 描述 |
| --- | --- |
| `vecf16` | 数据类型为 16 位浮点数组，`float16[]`。 |
| `vector` | 数据类型为 32 位浮点数组，`float32[]`。 |

> 说明<br>
> 与 `vector` 相比，`vecf16` 不仅存储成本小一倍，同时计算性能快一倍，且准度相同，因此更为推荐使用。


## 示例

执行如下命令，创建一个名为 `FACE_TABLE` 的堆表，其中 `id` 为主键，`embedding` 为向量列。

```sql
CREATE TABLE FACE_TABLE (
  id INT,
  create_at TIMESTAMP NOT NULL,
  content VARCHAR(20) NOT NULL,
  embedding VECF16(512) NOT NULL
  PRIMARY KEY (C1)
);
```

## 使用分区表管理增量数据

对于具备时间属性的数据（例如数据按时间维度导入，保留 `N` 天，超出时间过期），可以使用 `PARTITION BY` 语句将表划分成多个子表，可以对子表单独进行导入和 `TRUNCATE`，从而支持增量的表数据管理。

如下为示例语句：

```sql
CREATE TABLE test_tbl (
  c1 text,
  c2 vecf16(512), -- or vector(512)
  post_publish_time timestamp NOT NULL
) PARTITION BY RANGE (post_publish_time);

-- Daily partitions for August 2024
CREATE TABLE test_tbl_20240801 PARTITION OF test_tbl
  FOR VALUES FROM ('2024-08-01') TO ('2024-08-02');

CREATE TABLE test_tbl_20240802 PARTITION OF test_tbl
  FOR VALUES FROM ('2024-08-02') TO ('2024-08-03');

CREATE TABLE test_tbl_extra PARTITION OF test_tbl DEFAULT;
```