# Checkpoint 清理策略实战

> **核心问题**: Checkpoint 会无限增长,占用存储空间,需要定期清理旧数据

---

## 一、为什么需要清理 Checkpoint?

### 1.1 数据增长问题

```python
# 假设每个 checkpoint 占用 50 KB
# 一个 thread 执行 100 步,产生 100 个 checkpoint
storage_per_thread = 100 * 50 KB = 5 MB

# 1000 个用户,每人 10 个 thread
total_storage = 1000 * 10 * 5 MB = 50 GB

# ⚠️ 问题:
# 1. 数据库膨胀,查询变慢
# 2. 备份时间增长
# 3. 存储成本增加
# 4. 大部分旧 checkpoint 永远不会被访问
```

---

### 1.2 真实场景分析

```python
# DeepAgents CLI 的实际数据 (来自测试)
# - 平均每个任务: 20 个 checkpoint
# - 平均 checkpoint 大小: 30 KB (含 messages + files)
# - 每天活跃用户: 100 人
# - 每人每天平均任务: 5 个

daily_growth = 100 * 5 * 20 * 30 KB = 300 MB/天
monthly_growth = 300 MB * 30 = 9 GB/月

# ⚠️ 3 个月后:
# - 数据库 27 GB
# - list() 查询变慢 (遍历几十万条记录)
# - 备份时间从 1 分钟变成 10 分钟
```

---

## 二、清理策略

### 2.1 策略对比

| 策略 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **按时间清理** | 所有场景 | 简单,可预测 | 可能删除重要的旧数据 |
| **按数量保留** | 调试/审计 | 保证历史深度 | 不同 thread 增长速度不同 |
| **仅保留最新** | 无需历史 | 空间最小 | 丢失时间旅行能力 |
| **分层保留** | 生产环境 | 平衡空间和历史 | 实现复杂 |
| **手动标记保留** | 关键节点 | 精确控制 | 需要用户干预 |

---

## 三、实战代码

### 3.1 按时间清理 (推荐)

```python
# 源码位置: 自定义实现 (LangGraph 未提供内置 API)

import datetime
from langgraph.checkpoint.sqlite import SqliteSaver

def cleanup_old_checkpoints(
    checkpointer: SqliteSaver,
    retention_days: int = 30
) -> dict[str, int]:
    """删除超过 retention_days 天的 checkpoint
    
    Args:
        checkpointer: SqliteSaver 实例
        retention_days: 保留天数 (默认 30 天)
    
    Returns:
        {"deleted_checkpoints": count, "deleted_writes": count}
    """
    cutoff_time = datetime.datetime.now(datetime.timezone.utc) - datetime.timedelta(days=retention_days)
    cutoff_iso = cutoff_time.isoformat()
    
    # ===== 1. 查询要删除的 checkpoint =====
    with checkpointer.cursor(transaction=False) as cur:
        cur.execute(
            """
            SELECT checkpoint_id, thread_id, checkpoint_ns
            FROM checkpoints
            WHERE json_extract(checkpoint, '$.ts') < ?
            """,
            (cutoff_iso,)
        )
        
        to_delete = cur.fetchall()
    
    if not to_delete:
        print(f"No checkpoints older than {retention_days} days")
        return {"deleted_checkpoints": 0, "deleted_writes": 0}
    
    # ===== 2. 批量删除 =====
    deleted_checkpoints = 0
    deleted_writes = 0
    
    with checkpointer.cursor() as cur:
        for checkpoint_id, thread_id, checkpoint_ns in to_delete:
            # 删除 checkpoint
            cur.execute(
                "DELETE FROM checkpoints WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id = ?",
                (thread_id, checkpoint_ns, checkpoint_id)
            )
            deleted_checkpoints += cur.rowcount
            
            # 删除 pending writes
            cur.execute(
                "DELETE FROM writes WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id = ?",
                (thread_id, checkpoint_ns, checkpoint_id)
            )
            deleted_writes += cur.rowcount
    
    print(f"Deleted {deleted_checkpoints} checkpoints and {deleted_writes} writes")
    
    # ===== 3. VACUUM 回收空间 =====
    checkpointer.conn.execute("VACUUM")
    
    return {
        "deleted_checkpoints": deleted_checkpoints,
        "deleted_writes": deleted_writes
    }

# 使用示例
import sqlite3

conn = sqlite3.connect("checkpoints.db", check_same_thread=False)
checkpointer = SqliteSaver(conn)
checkpointer.setup()

# 清理 30 天前的数据
result = cleanup_old_checkpoints(checkpointer, retention_days=30)
print(result)
# Output: {'deleted_checkpoints': 1234, 'deleted_writes': 56}
```

---

### 3.2 按数量保留 (每个 Thread 保留最新 N 个)

```python
def cleanup_keep_latest_n(
    checkpointer: SqliteSaver,
    keep_count: int = 10
) -> dict[str, int]:
    """每个 thread 只保留最新的 N 个 checkpoint
    
    Args:
        checkpointer: SqliteSaver 实例
        keep_count: 每个 thread 保留的 checkpoint 数量
    
    Returns:
        {"deleted_checkpoints": count, "deleted_writes": count}
    """
    deleted_checkpoints = 0
    deleted_writes = 0
    
    with checkpointer.cursor(transaction=False) as cur:
        # ===== 1. 获取所有 thread_id =====
        cur.execute("SELECT DISTINCT thread_id, checkpoint_ns FROM checkpoints")
        threads = cur.fetchall()
    
    with checkpointer.cursor() as cur:
        for thread_id, checkpoint_ns in threads:
            # ===== 2. 查询该 thread 的所有 checkpoint (按时间倒序) =====
            cur.execute(
                """
                SELECT checkpoint_id
                FROM checkpoints
                WHERE thread_id = ? AND checkpoint_ns = ?
                ORDER BY checkpoint_id DESC
                """,
                (thread_id, checkpoint_ns)
            )
            
            all_checkpoints = [row[0] for row in cur.fetchall()]
            
            # ===== 3. 删除超过 keep_count 的旧 checkpoint =====
            if len(all_checkpoints) > keep_count:
                to_delete = all_checkpoints[keep_count:]  # 保留前 keep_count 个
                
                for checkpoint_id in to_delete:
                    cur.execute(
                        "DELETE FROM checkpoints WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id = ?",
                        (thread_id, checkpoint_ns, checkpoint_id)
                    )
                    deleted_checkpoints += cur.rowcount
                    
                    cur.execute(
                        "DELETE FROM writes WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id = ?",
                        (thread_id, checkpoint_ns, checkpoint_id)
                    )
                    deleted_writes += cur.rowcount
    
    # VACUUM 回收空间
    checkpointer.conn.execute("VACUUM")
    
    print(f"Deleted {deleted_checkpoints} checkpoints and {deleted_writes} writes")
    return {
        "deleted_checkpoints": deleted_checkpoints,
        "deleted_writes": deleted_writes
    }

# 使用示例
result = cleanup_keep_latest_n(checkpointer, keep_count=10)
# 每个 thread 只保留最新 10 个 checkpoint
```

---

### 3.3 仅保留最新 (最激进)

```python
def cleanup_keep_only_latest(checkpointer: SqliteSaver) -> dict[str, int]:
    """每个 thread 只保留最新的 1 个 checkpoint (最激进)
    
    ⚠️ 警告: 会丢失所有历史,无法时间旅行
    """
    deleted_checkpoints = 0
    deleted_writes = 0
    
    with checkpointer.cursor(transaction=False) as cur:
        cur.execute("SELECT DISTINCT thread_id, checkpoint_ns FROM checkpoints")
        threads = cur.fetchall()
    
    with checkpointer.cursor() as cur:
        for thread_id, checkpoint_ns in threads:
            # 查询最新 checkpoint
            cur.execute(
                """
                SELECT checkpoint_id
                FROM checkpoints
                WHERE thread_id = ? AND checkpoint_ns = ?
                ORDER BY checkpoint_id DESC
                LIMIT 1
                """,
                (thread_id, checkpoint_ns)
            )
            
            latest = cur.fetchone()
            if not latest:
                continue
            
            latest_checkpoint_id = latest[0]
            
            # 删除所有非最新的 checkpoint
            cur.execute(
                """
                DELETE FROM checkpoints
                WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id != ?
                """,
                (thread_id, checkpoint_ns, latest_checkpoint_id)
            )
            deleted_checkpoints += cur.rowcount
            
            cur.execute(
                """
                DELETE FROM writes
                WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id != ?
                """,
                (thread_id, checkpoint_ns, latest_checkpoint_id)
            )
            deleted_writes += cur.rowcount
    
    checkpointer.conn.execute("VACUUM")
    
    print(f"Deleted {deleted_checkpoints} checkpoints and {deleted_writes} writes")
    return {
        "deleted_checkpoints": deleted_checkpoints,
        "deleted_writes": deleted_writes
    }

# 使用示例
result = cleanup_keep_only_latest(checkpointer)
# ⚠️ 每个 thread 只保留最新 checkpoint,丢失所有历史
```

---

### 3.4 分层保留策略 (生产环境推荐)

```python
def cleanup_tiered_retention(checkpointer: SqliteSaver) -> dict[str, int]:
    """分层保留策略:
    
    - 最近 7 天: 保留所有 checkpoint
    - 8-30 天: 每天保留 1 个 (最新的)
    - 31-90 天: 每周保留 1 个
    - 90 天以上: 删除
    """
    now = datetime.datetime.now(datetime.timezone.utc)
    
    # 定义时间边界
    day_7 = now - datetime.timedelta(days=7)
    day_30 = now - datetime.timedelta(days=30)
    day_90 = now - datetime.timedelta(days=90)
    
    deleted_checkpoints = 0
    deleted_writes = 0
    
    with checkpointer.cursor(transaction=False) as cur:
        cur.execute("SELECT DISTINCT thread_id, checkpoint_ns FROM checkpoints")
        threads = cur.fetchall()
    
    with checkpointer.cursor() as cur:
        for thread_id, checkpoint_ns in threads:
            # ===== 1. 删除 90 天以上的所有 checkpoint =====
            cur.execute(
                """
                SELECT checkpoint_id
                FROM checkpoints
                WHERE thread_id = ? AND checkpoint_ns = ?
                  AND json_extract(checkpoint, '$.ts') < ?
                """,
                (thread_id, checkpoint_ns, day_90.isoformat())
            )
            
            for (checkpoint_id,) in cur.fetchall():
                cur.execute(
                    "DELETE FROM checkpoints WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id = ?",
                    (thread_id, checkpoint_ns, checkpoint_id)
                )
                deleted_checkpoints += cur.rowcount
                
                cur.execute(
                    "DELETE FROM writes WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id = ?",
                    (thread_id, checkpoint_ns, checkpoint_id)
                )
                deleted_writes += cur.rowcount
            
            # ===== 2. 31-90 天: 每周保留 1 个 =====
            cur.execute(
                """
                SELECT checkpoint_id, json_extract(checkpoint, '$.ts') as ts
                FROM checkpoints
                WHERE thread_id = ? AND checkpoint_ns = ?
                  AND ts >= ? AND ts < ?
                ORDER BY ts DESC
                """,
                (thread_id, checkpoint_ns, day_90.isoformat(), day_30.isoformat())
            )
            
            weekly_checkpoints = cur.fetchall()
            # 按周分组
            weeks = {}
            for checkpoint_id, ts in weekly_checkpoints:
                dt = datetime.datetime.fromisoformat(ts)
                week_key = dt.isocalendar()[:2]  # (year, week_number)
                if week_key not in weeks:
                    weeks[week_key] = []
                weeks[week_key].append(checkpoint_id)
            
            # 每周只保留最新的 1 个
            for week_checkpoints in weeks.values():
                if len(week_checkpoints) > 1:
                    to_delete = week_checkpoints[1:]  # 第一个是最新的
                    for checkpoint_id in to_delete:
                        cur.execute(
                            "DELETE FROM checkpoints WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id = ?",
                            (thread_id, checkpoint_ns, checkpoint_id)
                        )
                        deleted_checkpoints += cur.rowcount
                        
                        cur.execute(
                            "DELETE FROM writes WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id = ?",
                            (thread_id, checkpoint_ns, checkpoint_id)
                        )
                        deleted_writes += cur.rowcount
            
            # ===== 3. 8-30 天: 每天保留 1 个 =====
            cur.execute(
                """
                SELECT checkpoint_id, json_extract(checkpoint, '$.ts') as ts
                FROM checkpoints
                WHERE thread_id = ? AND checkpoint_ns = ?
                  AND ts >= ? AND ts < ?
                ORDER BY ts DESC
                """,
                (thread_id, checkpoint_ns, day_30.isoformat(), day_7.isoformat())
            )
            
            daily_checkpoints = cur.fetchall()
            # 按天分组
            days = {}
            for checkpoint_id, ts in daily_checkpoints:
                dt = datetime.datetime.fromisoformat(ts)
                day_key = dt.date()
                if day_key not in days:
                    days[day_key] = []
                days[day_key].append(checkpoint_id)
            
            # 每天只保留最新的 1 个
            for day_checkpoints in days.values():
                if len(day_checkpoints) > 1:
                    to_delete = day_checkpoints[1:]
                    for checkpoint_id in to_delete:
                        cur.execute(
                            "DELETE FROM checkpoints WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id = ?",
                            (thread_id, checkpoint_ns, checkpoint_id)
                        )
                        deleted_checkpoints += cur.rowcount
                        
                        cur.execute(
                            "DELETE FROM writes WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id = ?",
                            (thread_id, checkpoint_ns, checkpoint_id)
                        )
                        deleted_writes += cur.rowcount
    
    # VACUUM 回收空间
    checkpointer.conn.execute("VACUUM")
    
    print(f"Tiered cleanup: Deleted {deleted_checkpoints} checkpoints and {deleted_writes} writes")
    return {
        "deleted_checkpoints": deleted_checkpoints,
        "deleted_writes": deleted_writes
    }

# 使用示例
result = cleanup_tiered_retention(checkpointer)
# 平衡存储空间和历史深度
```

---

### 3.5 手动标记保留 (关键节点)

```python
def mark_checkpoint_as_important(
    checkpointer: SqliteSaver,
    thread_id: str,
    checkpoint_id: str
) -> None:
    """标记 checkpoint 为重要,清理时跳过
    
    实现方式: 在 metadata 中添加 "important": true
    """
    with checkpointer.cursor(transaction=False) as cur:
        cur.execute(
            "SELECT metadata FROM checkpoints WHERE thread_id = ? AND checkpoint_id = ?",
            (thread_id, checkpoint_id)
        )
        
        row = cur.fetchone()
        if not row:
            raise ValueError(f"Checkpoint {checkpoint_id} not found")
        
        metadata = json.loads(row[0]) if row[0] else {}
        metadata["important"] = True
        
    with checkpointer.cursor() as cur:
        cur.execute(
            "UPDATE checkpoints SET metadata = ? WHERE thread_id = ? AND checkpoint_id = ?",
            (json.dumps(metadata), thread_id, checkpoint_id)
        )

def cleanup_skip_important(
    checkpointer: SqliteSaver,
    retention_days: int = 30
) -> dict[str, int]:
    """清理旧 checkpoint,但跳过标记为重要的"""
    cutoff_time = datetime.datetime.now(datetime.timezone.utc) - datetime.timedelta(days=retention_days)
    cutoff_iso = cutoff_time.isoformat()
    
    deleted_checkpoints = 0
    deleted_writes = 0
    
    with checkpointer.cursor() as cur:
        cur.execute(
            """
            SELECT checkpoint_id, thread_id, checkpoint_ns, metadata
            FROM checkpoints
            WHERE json_extract(checkpoint, '$.ts') < ?
            """,
            (cutoff_iso,)
        )
        
        for checkpoint_id, thread_id, checkpoint_ns, metadata_json in cur.fetchall():
            metadata = json.loads(metadata_json) if metadata_json else {}
            
            # 跳过重要的 checkpoint
            if metadata.get("important"):
                continue
            
            cur.execute(
                "DELETE FROM checkpoints WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id = ?",
                (thread_id, checkpoint_ns, checkpoint_id)
            )
            deleted_checkpoints += cur.rowcount
            
            cur.execute(
                "DELETE FROM writes WHERE thread_id = ? AND checkpoint_ns = ? AND checkpoint_id = ?",
                (thread_id, checkpoint_ns, checkpoint_id)
            )
            deleted_writes += cur.rowcount
    
    checkpointer.conn.execute("VACUUM")
    
    return {
        "deleted_checkpoints": deleted_checkpoints,
        "deleted_writes": deleted_writes
    }

# 使用示例
# 标记关键 checkpoint
mark_checkpoint_as_important(checkpointer, "session-123", "1ed2a3b4-c5d5-6789...")

# 清理时跳过重要的
result = cleanup_skip_important(checkpointer, retention_days=30)
```

---

## 四、定时任务集成

### 4.1 使用 APScheduler (Python)

```python
from apscheduler.schedulers.background import BackgroundScheduler
import atexit

def setup_checkpoint_cleanup(checkpointer: SqliteSaver):
    """设置定时清理任务"""
    
    scheduler = BackgroundScheduler()
    
    # 每天凌晨 2 点执行清理
    scheduler.add_job(
        func=cleanup_old_checkpoints,
        trigger="cron",
        hour=2,
        minute=0,
        args=[checkpointer, 30]  # 保留 30 天
    )
    
    scheduler.start()
    
    # 程序退出时停止调度器
    atexit.register(lambda: scheduler.shutdown())
    
    print("Checkpoint cleanup scheduler started (daily at 2:00 AM)")

# 使用示例
setup_checkpoint_cleanup(checkpointer)
```

---

### 4.2 使用 Cron (Linux)

```bash
# 添加到 crontab
# 每天凌晨 2 点执行清理脚本

# 编辑 crontab
$ crontab -e

# 添加以下行:
0 2 * * * /usr/bin/python3 /path/to/cleanup_script.py

# cleanup_script.py:
import sqlite3
from langgraph.checkpoint.sqlite import SqliteSaver

conn = sqlite3.connect("/path/to/checkpoints.db", check_same_thread=False)
checkpointer = SqliteSaver(conn)
checkpointer.setup()

result = cleanup_old_checkpoints(checkpointer, retention_days=30)
print(f"Cleanup completed: {result}")

conn.close()
```

---

### 4.3 使用 Systemd Timer (现代 Linux)

```ini
# /etc/systemd/system/checkpoint-cleanup.service
[Unit]
Description=Cleanup old LangGraph checkpoints
After=network.target

[Service]
Type=oneshot
User=your-user
ExecStart=/usr/bin/python3 /path/to/cleanup_script.py
StandardOutput=journal
StandardError=journal

# /etc/systemd/system/checkpoint-cleanup.timer
[Unit]
Description=Run checkpoint cleanup daily at 2 AM

[Timer]
OnCalendar=daily
Persistent=true
OnCalendar=*-*-* 02:00:00

[Install]
WantedBy=timers.target

# 启用 timer
$ sudo systemctl enable checkpoint-cleanup.timer
$ sudo systemctl start checkpoint-cleanup.timer

# 查看状态
$ sudo systemctl status checkpoint-cleanup.timer
$ sudo systemctl list-timers
```

---

## 五、PostgreSQL 的优势

### 5.1 原生 TTL 支持 (需要扩展)

```sql
-- PostgreSQL 可以用 pg_cron 扩展实现定时清理
CREATE EXTENSION pg_cron;

-- 每天凌晨 2 点执行清理
SELECT cron.schedule(
    'cleanup-old-checkpoints',
    '0 2 * * *',
    $$
    DELETE FROM checkpoints
    WHERE (checkpoint->>'ts')::timestamp < NOW() - INTERVAL '30 days';
    $$
);

-- 或者用 Partition + 自动删除旧分区
CREATE TABLE checkpoints_2024_01 PARTITION OF checkpoints
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- 删除旧分区 (瞬间完成,无需扫描)
DROP TABLE checkpoints_2023_12;
```

---

### 5.2 更高效的 VACUUM

```sql
-- SQLite VACUUM: 需要重写整个数据库文件,阻塞所有操作
VACUUM;  -- ⚠️ 可能需要几分钟到几小时

-- PostgreSQL VACUUM: 增量回收,不阻塞读写
VACUUM checkpoints;  -- ✅ 秒级完成

-- 自动 VACUUM
ALTER TABLE checkpoints SET (autovacuum_enabled = true);
```

---

## 六、监控和告警

### 6.1 数据库大小监控

```python
def get_database_stats(checkpointer: SqliteSaver) -> dict:
    """获取数据库统计信息"""
    with checkpointer.cursor(transaction=False) as cur:
        # 总 checkpoint 数量
        cur.execute("SELECT COUNT(*) FROM checkpoints")
        total_checkpoints = cur.fetchone()[0]
        
        # 总 writes 数量
        cur.execute("SELECT COUNT(*) FROM writes")
        total_writes = cur.fetchone()[0]
        
        # 每个 thread 的 checkpoint 数量
        cur.execute(
            """
            SELECT thread_id, COUNT(*) as count
            FROM checkpoints
            GROUP BY thread_id
            ORDER BY count DESC
            LIMIT 10
            """
        )
        top_threads = cur.fetchall()
        
        # 数据库文件大小 (SQLite)
        import os
        db_size = os.path.getsize(checkpointer.conn.execute("PRAGMA database_list").fetchone()[2])
    
    return {
        "total_checkpoints": total_checkpoints,
        "total_writes": total_writes,
        "database_size_mb": db_size / (1024 * 1024),
        "top_threads": [
            {"thread_id": tid, "checkpoint_count": count}
            for tid, count in top_threads
        ]
    }

# 使用示例
stats = get_database_stats(checkpointer)
print(stats)
# Output:
# {
#     "total_checkpoints": 12345,
#     "total_writes": 678,
#     "database_size_mb": 456.78,
#     "top_threads": [
#         {"thread_id": "session-123", "checkpoint_count": 234},
#         ...
#     ]
# }

# 设置告警阈值
if stats["database_size_mb"] > 1024:  # 超过 1 GB
    send_alert("Checkpoint database exceeds 1 GB, cleanup recommended")
```

---

### 6.2 清理效果监控

```python
def monitor_cleanup(checkpointer: SqliteSaver):
    """监控清理效果"""
    
    # 清理前统计
    before_stats = get_database_stats(checkpointer)
    print(f"Before cleanup: {before_stats['total_checkpoints']} checkpoints, {before_stats['database_size_mb']:.2f} MB")
    
    # 执行清理
    import time
    start_time = time.time()
    result = cleanup_old_checkpoints(checkpointer, retention_days=30)
    elapsed_time = time.time() - start_time
    
    # 清理后统计
    after_stats = get_database_stats(checkpointer)
    print(f"After cleanup: {after_stats['total_checkpoints']} checkpoints, {after_stats['database_size_mb']:.2f} MB")
    
    # 计算节省的空间
    saved_mb = before_stats["database_size_mb"] - after_stats["database_size_mb"]
    
    return {
        "deleted_checkpoints": result["deleted_checkpoints"],
        "deleted_writes": result["deleted_writes"],
        "saved_mb": saved_mb,
        "elapsed_seconds": elapsed_time
    }

# 使用示例
result = monitor_cleanup(checkpointer)
print(result)
# Output:
# Before cleanup: 12345 checkpoints, 456.78 MB
# Deleted 8901 checkpoints and 234 writes
# After cleanup: 3444 checkpoints, 123.45 MB
# {
#     "deleted_checkpoints": 8901,
#     "deleted_writes": 234,
#     "saved_mb": 333.33,
#     "elapsed_seconds": 2.34
# }
```

---

## 七、常见问题

### Q1: 删除 Checkpoint 会影响正在运行的 Agent 吗?

**A:** 取决于 Agent 是否持有 checkpoint_id:

```python
# 场景1: Agent 未指定 checkpoint_id (总是使用最新)
config = {"configurable": {"thread_id": "session-1"}}
agent.invoke(input, config)  # ✅ 不受影响,始终使用最新 checkpoint

# 场景2: Agent 指定了 checkpoint_id (时间旅行)
config = {"configurable": {"thread_id": "session-1", "checkpoint_id": "old-id"}}
agent.invoke(input, config)  # ❌ 如果 old-id 被删除,会报错
```

**建议:** 清理前检查是否有 pending 任务:

```python
def safe_cleanup(checkpointer, retention_days):
    # 1. 标记要删除的 checkpoint
    to_delete = find_old_checkpoints(checkpointer, retention_days)
    
    # 2. 检查是否有 pending writes (HITL 中断)
    with checkpointer.cursor(transaction=False) as cur:
        for cp_id in to_delete:
            cur.execute(
                "SELECT COUNT(*) FROM writes WHERE checkpoint_id = ?",
                (cp_id,)
            )
            if cur.fetchone()[0] > 0:
                print(f"Skipping {cp_id}: has pending writes")
                to_delete.remove(cp_id)
    
    # 3. 执行删除
    delete_checkpoints(checkpointer, to_delete)
```

---

### Q2: VACUUM 会锁表吗?

**A:** 是的:

```python
# SQLite VACUUM: 完全锁表,阻塞所有读写
checkpointer.conn.execute("VACUUM")  # ⚠️ 可能需要几分钟

# 解决方案1: 在低峰期执行
import time
if time.localtime().tm_hour in [2, 3, 4]:  # 凌晨 2-4 点
    checkpointer.conn.execute("VACUUM")

# 解决方案2: 用 SQLite 的增量 VACUUM (需要启用 auto_vacuum)
checkpointer.conn.execute("PRAGMA auto_vacuum = INCREMENTAL")
checkpointer.conn.execute("PRAGMA incremental_vacuum(100)")  # 回收 100 页

# PostgreSQL 不需要担心: VACUUM 不阻塞读写
```

---

### Q3: 如何恢复误删的 Checkpoint?

**A:** 需要提前备份:

```bash
# 定期备份 SQLite 数据库
$ sqlite3 checkpoints.db ".backup checkpoints_backup_$(date +%Y%m%d).db"

# 恢复
$ cp checkpoints_backup_20240115.db checkpoints.db

# 或者用 WAL 模式的增量备份
$ sqlite3 checkpoints.db "PRAGMA wal_checkpoint(TRUNCATE)"
$ cp checkpoints.db checkpoints-wal checkpoints-shm /backup/
```

---

## 八、最佳实践总结

1. **生产环境推荐配置**:
   - 使用分层保留策略 (7 天全保留 + 30 天每天 1 个 + 90 天每周 1 个)
   - 每天凌晨 2-4 点执行清理
   - 清理后执行 VACUUM
   - 监控数据库大小,超过阈值告警

2. **避免的陷阱**:
   - ❌ 不要在高峰期执行 VACUUM
   - ❌ 不要删除有 pending writes 的 checkpoint
   - ❌ 不要删除最新的 checkpoint (即使超过保留期)

3. **PostgreSQL 优势**:
   - ✅ 用 Partition 实现自动清理
   - ✅ VACUUM 不阻塞读写
   - ✅ pg_cron 原生定时任务

---

**Linus 式评价:**

> "Checkpoint 清理是个真问题,但 LangGraph 没提供内置 API 是个笑话。你得自己写 SQL 删除数据,这不是用户应该操心的事。"
>
> "SQLite 的 VACUUM 是个灾难 - 锁表几分钟,生产环境直接崩溃。PostgreSQL 的增量 VACUUM 才是正确实现。"
>
> "如果你要生产用,别浪费时间优化 SQLite 的清理策略。直接上 PostgreSQL + Partition,一个 DROP TABLE 搞定,1 秒钟清理 1 亿条记录。"
