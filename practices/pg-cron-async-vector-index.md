# 最佳实践：使用 pg\_cron 异步创建大表向量索引

## 挑战

在大表（数百万行）上创建向量索引是一个耗时的过程，同步的 `CREATE INDEX` 操作可能需要数小时才能完成。如果创建过程中客户端连接断开（网络问题或超时），整个操作将被回滚，需要重新开始。

## 解决方案

[pg\_cron](https://github.com/citusdata/pg_cron) 是 PostgreSQL 的任务调度扩展，可以在后台运行索引创建，与客户端连接解耦。AIDB 实例已内置安装 pg\_cron，连接至 postgres 数据库即可使用。可通过以下 SQL 检查 pg\_cron 是否可用：

```sql
SELECT * FROM pg_extension WHERE extname = 'pg_cron';
```

## 实施步骤

### 步骤 1：创建索引创建函数

在目标数据库中创建函数，封装索引创建逻辑：

```sql
CREATE OR REPLACE FUNCTION create_my_vector_index()
RETURNS TEXT AS $$
BEGIN
  CREATE INDEX IF NOT EXISTS my_vector_idx
  ON my_vector_tbl
  USING vectors (embedding vecf16_l2_ops)
  WITH (options = $outer$
    [indexing.hnsw]
    m=16
    ef_construction=200
  $outer$);

  RETURN 'Vector index created successfully';
EXCEPTION
  WHEN OTHERS THEN
    RAISE EXCEPTION 'Failed to create vector index: %', SQLERRM;
END;
$$ LANGUAGE plpgsql;
```

**索引参数说明：** 以下参数仅作为参考，用户可根据自己的实际情况调整：

*   `vecf16_l2_ops`：用于 vecf16 类型的 L2 距离操作符类
    
*   `m=16`：HNSW 参数，控制每个节点的双向链接数量
    
*   `ef_construction=200`：HNSW 参数，控制索引质量与构建速度的平衡
    

### 步骤 2：调度索引创建任务

在 postgres 数据库中使用 pg\_cron 调度任务：

```sql
SELECT cron.schedule_in_database(
  'create-vector-index-job',           -- 任务名称
  cron.get_next_minute_cron_expr(),    -- 在下一分钟运行
  'SELECT create_my_vector_index()',   -- 调用函数
  'your_database_name'                 -- 目标数据库
);

```

:::
任务会在下一分钟调度执行。索引创建完成后，请务必执行步骤 4 的清理操作来移除调度任务。
:::

### 步骤 3：监控任务执行

在 postgres 数据库中查看任务执行状态：

```sql
-- 查看所有调度任务
SELECT * FROM cron.job;

-- 查看任务运行历史
SELECT * FROM cron.job_run_details
ORDER BY start_time DESC
LIMIT 10;

-- 检查特定任务状态
SELECT jobid, database, status, return_message
FROM cron.job_run_details
WHERE jobid = (
  SELECT jobid FROM cron.job
  WHERE jobname = 'create-vector-index-job'
)
ORDER BY start_time DESC
LIMIT 5;
```

### 步骤 4：清理调度任务

索引创建成功后，移除调度任务：

```sql
SELECT cron.unschedule('create-vector-index-job');
```

## 总结

使用 pg\_cron 进行异步向量索引创建，为在 AIDB 中管理大规模向量数据提供了简单可靠的解决方案。索引创建在后台运行，独立于客户端连接，确保即使发生网络问题，任务也能成功完成。