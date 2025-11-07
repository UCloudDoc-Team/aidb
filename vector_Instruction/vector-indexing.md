# 向量索引

在处理大型数据集或需要快速访问和检索数据的场景（数据库查询优化、机器学习和数据挖掘、图像和视频检索、空间数据查询等）中，创建向量索引是加速向量检索的有效方式，可以提高查询性能、加速数据分析和优化搜索任务，从而提高系统的效率和响应速度。

## 背景介绍

AI Database 采用了主流的 HNSW（Hierarchical Small World Graph）算法，将 `skip list` 的概念与 NSW（Navigable Small World）图相结合，通过分层结构实现了高效的近似最近邻搜索。在该结构中，上层包含较长的边，用于快速定位目标，而下层则包含较短的边，以提升搜索精度。

在构建图时，可以通过调整参数 `m` 来控制新节点与其最近邻节点的连接数量。较高的 `m` 值会使图结构更加密集，节点之间的连接增多，从而提升搜索性能，但同时会增加内存消耗并延长插入时间。在节点插入时，算法会找到最近的 `m` 个节点，并与它们建立双向连接。

此外，AIDB 还支持量化（Product Quantization，PQ）功能，可对高维向量进行降维处理。通过在索引中存储降维后的向量，可以减少回表操作的次数，从而提高向量插入和查询时的检索性能。

## 语法

```sql
CREATE INDEX <index_name>
ON <schema_name>.<table_name>
USING vectors (<column_name> <distance_measure>) 
WITH (options = $$
  <common_option_key1> = <common_option_value1>
  <common_option_key2> = <common_option_value2>
  ...
  [indexing.hnsw]
  <hnsw_option_key1> = <hnsw_option_value1>
  <hnsw_option_key2> = <hnsw_option_value2>
  ...
$$);
```

## 参数说明

- `<index_name>`：索引名称，为可选参数。
    
- `<schema_name>`：Schema 名称。
    
- `<table_name>`：表名称。
    
- `<column_name>`：向量索引列名称。
    
- `<distance_measure>`：相似度距离度量算法，其表示形式为 `<vector_data_type>`\_`<distance_type>`\_ops。可选值包括：
    
   - `vecf16_l2_ops`: 16 位浮点数类型的平方欧式距离
        
   - `vecf16_dot_ops`: 16 位浮点数类型的负点积距离
        
   - `vecf16_cos_ops`: 16 位浮点数类型的余弦距离
        
   - `vector_l2_ops`: 32 位浮点数类型的平方欧式距离
        
   - `vector_dot_ops`: 32 位浮点数类型的负点积距离
        
   - `vector_cos_ops`: 32 位浮点数类型的余弦距离
        
- 其它可选通用向量索引参数（`<common_option_n>`）说明如下表所示：
    

| 键 | 类型 | 取值范围 | 默认值 | 说明 |
| :- | :- | :- | :- | :- |
| `optimizing.optimizing_threads` | `integer` | [1, 65535] | `1` | 构建索引的最大线程数。 |
| `optimizing.sealing_secs` | `integer` | [1, 60] | `60` | 构建索引合并探测时间。 |
| `segment.max_growing_segment_size` | `integer` | [1, 4_000_000_000] | `20_000` | 未创建索引的向量的最大大小。 |
| `segment.max_sealed_segment_size` | `integer` | [1, 4_000_000_000] | `1_000_000` | 用于索引创建的向量的最大大小。 |

- HNSW 算法的相关参数（`HNSW_OPTION`）说明如下表所示：
    

| 键 | 类型 | 取值范围 | 默认值 | 说明 |
| :- | :- | :- | :- | :- |
| `m` | `integer` | [4, 128] | `12` | 节点的最大度数。 |
| `ef_construction` | `integer` | [10, 2000] | `300` | 构建中的搜索范围。 |
| `quantization` | `table` | `trivial`、`scalar` 或 `product` | N/A | 可选值为：<br>`trivial`：不使用量化<br>`scalar`：使用标量量化<br>`product`：使用乘积量化（推荐）<br>计算距离使用的量化算法，其具体参数见下方表格”`quantization.product` 的选项“。 |

- `quantization.product` 的选项
    

| 键 | 类型 | 取值范围 | 默认值 | 说明 |
| :- | :- | :- | :- | :- |
| `sample` | `integer` | [1, 1_000_000] | `65535` | 用于量化的样本。 |
| `ratio` | `enum` | "x4"、"x8"、"x16"、"x32"、"x64" | `"x4"` | 量化的压缩比。 |

## 示例

假设有一个文本知识库，将文章分割成块（Chunk）后，转换为 512 维的 Embedding 向量，最后存入数据库中，其中切割生成的 `docs` 表包含以下字段：

| 字段 | 类型 | 说明 |
| :- | :- | :- |
| `id` | `serial` | 编号。 |
| `chunk` | `varchar(1024)` | 文章切块后的文本块。 |
| `intime` | `timestamp` | 文章的入库时间。 |
| `url` | `varchar(1024)` | 文本块所属文章的链接。 |
| `embedding` | `vecf16(512)` | 文本块 Embedding 向量。 |

1.  创建存储向量的数据表。
    
    ```sql
    CREATE TABLE docs (
        id SERIAL PRIMARY KEY,
        chunk VARCHAR(1024),
        intime TIMESTAMP,
        url VARCHAR(1024),
        embedding VECF16(512)
    );
    ```
    
2.  将向量列的存储模式设置为 `PLAIN`，以降低数据行扫描成本，获得更好的性能。
    
    ```sql
    ALTER TABLE docs ALTER COLUMN embedding SET STORAGE PLAIN;
    ```
    
3.  对向量列建立向量索引。
    
    ```sql
    -- 创建欧氏距离度量的向量索引。
    CREATE INDEX idx_docs_feature_l2 ON docs
    USING vectors(embedding vecf16_l2_ops)
    WITH (options = $$
      optimizing.optimizing_threads = 3
      segment.max_growing_segment_size = 100000
      segment.max_sealed_segment_size = 8000000
      [indexing.hnsw]
      m=30
      ef_construction=500
    $$);
    ```
    
4.  为常用的结构化列建立索引，提升融合查询速度。
    

    ```sql
    CREATE INDEX ON docs(intime);
    ```

## 相关参考

### 支持的运算符类型

| 名称 | 描述 | 公式定义 |
| --- | --- | --- |
| `<->` | 平方欧氏距离 | ![image](/images/001.png) |
| `<#>` | 负点积 | ![image](/images/002.png) |
| `<=>` | 余弦距离 | ![image](/images/003.png) |

向量运算符的使用示例：

```sql
-- 平方欧氏距离
SELECT '[1, 2, 3]'::vector <-> '[3, 2, 1]'::vector;

-- 负点积
SELECT '[1, 2, 3]'::vector <#> '[3, 2, 1]'::vector;

-- 余弦距离
SELECT '[1, 2, 3]'::vector <=> '[3, 2, 1]'::vector;
```