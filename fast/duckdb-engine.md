# DuckDB 分析引擎

AI Database 内置 **pg_duckdb** 插件，基于 **DuckDB 列式向量化分析引擎** 提供高性能分析能力，支持复杂聚合、窗口函数和联邦查询，适用于实时分析与数据探索场景。


## 前提条件

在开始使用 AI Database 之前，请确保已完成以下准备工作：

- 已完成 AI Database 实例创建，且实例处于可用状态。

    关于如何创建实例，请参考[实例管理](/aidb/guide/manage-instances.md#创建实例)对应章节。

- 已创建数据库用户。

    关于如何创建数据库用户，请参考[用户管理](/aidb/guide/manage-users.md#创建用户)对应章节。

- 已成功连接到数据库。

    关于如何链接至数据库，请参考[访问实例](/aidb/guide/access-instances.md)。


## 核心优势

- **无缝集成**：完全兼容 PostgreSQL 语法与生态，无需修改现有 SQL 语句即可启用。

- **高性能**：自动采用列式向量化执行引擎，大幅提升分析查询性能。

- **易于使用**：仅需设置参数 `duckdb.force_execution = true` 即可开启 DuckDB 加速模式。


## 步骤 1：创建表并插入数据

如下示例创建了一个名为 `orders` 的订单表并插入了 3 条测试数据。

```sql
-- 创建标准 PostgreSQL 表
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    product_name TEXT,
    amount NUMERIC,
    order_date DATE
);

-- 插入测试数据
INSERT INTO orders (product_name, amount, order_date)
VALUES 
    ('Laptop', 1200.00, '2024-07-01'),
    ('Keyboard', 75.50, '2024-07-01'),
    ('Mouse', 25.00, '2024-07-02');
```

## 步骤 2：启用 DuckDB 并执行查询

启用 DuckDB 执行引擎后，查询会自动切换至列式向量化模式，无需修改 SQL 语法即可获得分析加速：

```sql
-- 启用 DuckDB 执行引擎
SET duckdb.force_execution = true;

-- 示例：按产品统计销售额
SELECT
    product_name,
    SUM(amount) as total_sales,
    COUNT(*) as order_count
FROM
    orders
GROUP BY
    product_name
ORDER BY
    total_sales DESC;

-- 示例：按日期统计销售额
SELECT
    order_date,
    SUM(amount) as daily_sales,
    COUNT(*) as order_count
FROM
    orders
GROUP BY
    order_date
ORDER BY
    order_date;
```

## 使用说明

### 自动 Fallback 机制

当查询中包含 DuckDB 暂不支持的 SQL 语法 时，系统会**自动回退（fallback）至 PostgreSQL 引擎**执行，以确保查询兼容性与稳定性。开发者无需进行额外配置或修改 SQL，即可在两种引擎间无缝切换。

### 查看执行引擎

可通过 `EXPLAIN` 命令查看当前查询的执行计划，判断是否由 DuckDB 引擎执行：

```sql
-- 查看执行计划
EXPLAIN SELECT
    product_name,
    SUM(amount) as total_sales
FROM
    orders
GROUP BY
    product_name;
```

若执行计划中出现与 **DuckDB** 相关的标识信息，即表示该查询已启用 DuckDB 引擎执行。