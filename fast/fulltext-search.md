# 全文检索

AI Database** 默认内置 **pg_search** 插件，为数据库提供高性能的全文检索能力。通过该插件，用户可以在结构化与非结构化文本数据中实现快速、精确的关键词匹配与模糊搜索。

AI Database 的 **pg_search** 插件是在开源项目 [Paradedb](https://github.com/paradedb/paradedb) 的基础上进行增强与优化的版本，具备更高的查询性能、更完善的索引管理机制以及与向量检索的更强融合能力。


## 前提条件

在开始使用 AI Database 之前，请确保已完成以下准备工作：

- 已完成 AI Database 实例创建，且实例处于可用状态。

    关于如何创建实例，请参考[实例管理](guide/manage-instances.md#创建实例)对应章节。

- 已创建数据库用户。

    关于如何创建数据库用户，请参考[实例管理](guide/manage-users.md#创建用户)对应章节。

- 已成功连接到数据库。

    关于如何链接至数据库，请参考[访问实例](guide/access-instances.md)。


## 核心优势

- **实时检索**：支持数据写入后即时可检索，确保查询结果实时更新，无需额外的索引刷新或延迟处理。

- **高性能**：采用并行化的 **bm25** 索引构建与查询机制，大幅提升全文检索与排序效率，适用于高并发、大规模文本场景。

- **ACID 事务保障**：全面支持 **ACID** 特性，确保全文索引操作与数据库事务保持一致性，避免脏读与数据丢失。

- **崩溃恢复（Crash Recovery）**：基于 **WAL（Write-Ahead Logging）** 实现可靠的备份与崩溃恢复机制，保障系统的高可用与数据安全。

- **自定义分词**：支持用户自定义分词器与分词策略，满足多语言、领域专属词汇或特定文本分析场景的需求。


## 步骤 1：创建表并插入数据

AI Database 提供内置函数 `create_bm25_test_table`，可快速创建示例表并自动插入测试数据，方便开发者验证全文检索功能。

```sql
CALL pgsearch.create_bm25_test_table(
      schema_name => 'public',
      table_name => 'mock_items'
    );
```
执行后，系统将在指定模式下创建 `mock_items` 表，并生成包含文本与元数据字段的样例数据。

## 步骤 2：创建 BM25 索引

创建 BM25 索引以实现高效的全文搜索能力。BM25 是一种基于概率统计的文本相关性算法，能够为检索结果提供更准确的排序。

```sql
CREATE INDEX search_idx ON mock_items
USING bm25 (id, description, category, rating, in_stock, created_at, metadata)
WITH (key_field='id');
```

**说明：**

- `USING bm25` 指定索引类型为 BM25。

- `key_field='id'` 用于定义主键字段，以便在查询结果中返回唯一标识。该索引将自动为多个字段（如 `description`、`category`、`metadata` 等）构建倒排索引，支持多字段联合检索与相关性评分。


## 步骤 3：使用 BM25 索引查询

如下示例涵盖了 BM25 索引的主要检索能力，包括基础关键词搜索、条件过滤、模糊匹配、结果高亮、相关度评分与权重控制等。通过这些特性，AI Database 能够支持从简单查询到复杂语义搜索的多层次全文检索需求。

### 单个 Term 查询

查询单个关键词（Term），适用于基础的文本检索场景。

```sql
SELECT description, rating, category
FROM mock_items
WHERE description @@@ 'shoes'
```
    
### 多个 Term 查询

支持逻辑组合（`OR` / `AND`）以实现多字段、多关键词匹配。

```sql
SELECT description, rating, category
FROM mock_items
WHERE description @@@ 'keyboard' OR category @@@ 'toy'
```


### 带过滤条件的查询

结合结构化字段过滤，实现更精准的检索，例如时间、数值或集合范围过滤。
    
```sql
SELECT id, description, rating
FROM mock_items
WHERE description @@@ 'shoes' AND rating @@@ '>3'
ORDER BY id;
    
-- 或者
SELECT id, description, rating
FROM mock_items
WHERE description @@@ 'shoes' AND rating > 3;
    
SELECT description, rating, category
FROM mock_items
WHERE description @@@ 'shoes' AND rating @@@ '4';
    
SELECT description, rating, category
FROM mock_items
WHERE description @@@ 'shoes' AND created_at @@@ '"2023-04-20T16:38:02Z"'
    
SELECT description, rating, category
FROM mock_items
WHERE description @@@ 'shoes' AND in_stock @@@ 'true'
    
SELECT description, rating, category
FROM mock_items
WHERE description @@@ 'shoes' AND rating @@@ '[1 TO 4]'
    
SELECT description, rating, category
FROM mock_items
WHERE description @@@ '[book TO camera]'
    
    
-- 集合过滤
SELECT description, rating, category
FROM mock_items
WHERE description @@@ 'shoes' AND rating @@@ 'IN [2 3 4]'
```
    
### 前缀 / 后缀匹配（波浪线操作符）

`~` 操作符用于执行模糊匹配或短语接近搜索（phrase proximity）。
    
```sql
SELECT description, rating, category
FROM mock_items
WHERE description @@@ '"ergonomic keyboard"~1';
    
    
-- 输出
SELECT description, rating, category
FROM mock_items
WHERE description @@@ '"ergonomic keyboard"~1';
    
       description        | rating |  category   
--------------------------+--------+-------------
 Ergonomic metal keyboard |      4 | Electronics
(1 row)
```
    
### 高亮显示（Highlighting）

`pgsearch.snippet()` 函数可在结果中高亮匹配片段，便于展示搜索上下文。
    
```sql
SELECT id, pgsearch.snippet(description)
FROM mock_items
WHERE description @@@ 'shoes'
LIMIT 5
```

### 相似度得分（Score）

`pgsearch.score()` 返回匹配结果的相关性分数，可按相关度排序输出。

```sql
SELECT description, rating, category, pgsearch.score(id)
FROM mock_items
WHERE description @@@ 'shoes'
ORDER BY score DESC
LIMIT 5
```
    
### 权重提升（Boosting Weight）

通过 `^` 设置字段或关键词权重，提升特定字段的重要性。
    
```sql
SELECT id, pgsearch.score(id)
FROM mock_items
WHERE description @@@ 'shoes^2' OR category @@@ 'footwear'
ORDER BY score DESC
LIMIT 5
```
    
### Match 查询（JSON / 函数式语法）

支持结构化 `match` 查询，可指定字段、匹配值与分词方式：

```sql
SELECT
    description,
    rating,
    category
FROM
    mock_items
WHERE
    id @@@ pgsearch.match('description', 'running shoes');

SELECT
    description,
    rating,
    category
FROM
    mock_items
WHERE
    id @@@ '{
      "match": {
          "field": "description",
          "value": "running shoes"
      }
  }'::jsonb;

SELECT
    description,
    rating,
    category
FROM
    mock_items
WHERE
    id @@@ pgsearch.match(
        'description',
        'running shoes',
        tokenizer => pgsearch.tokenizer('whitespace')
    );
```