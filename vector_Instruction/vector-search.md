# 向量检索

对于向量列的搜索会使用到 HNSW 索引，此方式搜索速度较快，但得到的结果是一个近似的结果，一般召回率都可以达到 95% 以上。

## 语法

如下以欧氏距离为例：

```sql
SELECT
  ID,
  `vector_column_name` <-> '[0.1,0.2,0.3...]' as distance
FROM
  `table_name`
ORDER BY
  `order_column_name` <-> '[0.1,0.2,0.3...]'
LIMIT
  `top_k`;
```

## 使用说明

- 如果没有建立索引，上述近似的索引检索将退化为精确检索，该检索方式速度较慢，但准确率为 100%。
    
- 在使用向量索引时，`ORDER BY` 语句必须包含 `<->`、`<#>` 或 `<=>` 等操作符，否则不能有效使用向量索引的加速能力。同时， `<->`、`<#>` 或 `<=>` 等操作符必须有对应距离度量的向量索引存在，否则也不能使用向量索引的加速能力。支持的操作符及其用法，请参见[向量索引](/aidb/vector_Instruction/vector-indexing.md)。
    

## 示例

以[向量索引](/aidb/vector_Instruction/vector-indexing.md)中的知识库为例，如果我们要通过文本搜索它的来源文章，那么我们就可以直接通过向量索引检索进行查找。


具体 SQL 如下：

```sql
SELECT
  id,
  chunk,
  intime,
  url,
  embedding < -> '[10,2.0,..., 1536.0]' as distance
FROM
  docs
ORDER BY
  embedding < -> '[10,2.0,..., 1536.0]'
LIMIT
  100;
```

