# 向量检索

## 前提条件

在开始使用 AI Database 之前，请确保已完成以下准备工作：

- 已完成 AI Database 实例创建，且实例处于可用状态。

- 已创建数据库用户。

- 已成功连接到数据库。

> 提示
> 若需了解实例创建、用户管理及连接方法，请参阅[相关操作文档](#链接至对应的文档)。



## 步骤 1：创建表并写入数据

如下示例创建了一个名为 `test_tbl` 的包含向量列的表，并向其中写入随机测试数据。

```sql
-- 创建包含向量列的表
CREATE TABLE test_tbl (
  id serial primary key, 
  embedding vector(3), 
  ts TIMESTAMP
);

-- 插入测试数据
INSERT INTO test_tbl (embedding, ts)
SELECT
  ARRAY[random(), random(), random()]::real[ ]::vector,
  now() - '1 days'::interval * (random() * 10)::int
FROM generate_series(1, 1000);
```

## 步骤 2：创建向量索引

使用 HNSW（Hierarchical Navigable Small World） 索引与 L2 距离（平方欧氏距离），实现高效相似度检索：

```sql
CREATE INDEX test_vector_idx 
ON test_tbl 
USING vectors (embedding vector_l2_ops);
```

> 说明
>  `vector_l2_ops` 为向量运算符类，用于计算平方欧氏距离（L2 距离）。


## 步骤 3：执行向量搜索

### 基础查询：Top-K 最近邻搜索

通过向量距离计算，可以快速检索与目标向量最相似的记录。以下示例展示如何获取与指定向量最接近的前 5 条数据：

```sql
SELECT id, embedding <-> '[3,1,2]' as dist
FROM test_tbl 
ORDER BY dist 
LIMIT 5;
```

> 说明
> `<->` 运算符用于计算向量间的相似度（此处为 L2 距离），结果按距离升序排列，数值越小表示越相似。


## 高级查询：带过滤条件的相似度搜索

在实际应用中，常需在时间、分类或数值条件下进行相似度过滤。以下示例展示了如何结合时间过滤与相似度阈值执行更复杂的查询：

```sql
-- 查询过去 5 天内，相似度超过阈值的 20 条数据
SELECT 
  b.* 
FROM (
  SELECT 
    a.id, a.dist * 10 AS similarity 
  FROM (
    SELECT id, embedding <-> '[3, 1, 2]' AS dist
    FROM test_tbl 
    WHERE ts > now() - '5 days'::interval
    ORDER BY dist ASC 
    LIMIT 100
  ) AS a
) AS b
WHERE b.similarity > 50
OFFSET 0 LIMIT 20;
```