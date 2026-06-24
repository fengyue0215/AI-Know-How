# MCP Server 开发入门 —— 让 AI 直接操作你的后端基础设施

> MCP（Model Context Protocol）是 AI 工具与外部系统交互的标准协议。  
> 为你的 Redis / Oracle / ZMQ 写 MCP Server，AI 就能直接查询、调试和操作它们。

---

## 🎯 一、为什么后端团队需要自己写 MCP Server

```
当前状态：AI 只能读代码，需要你手动粘贴日志/数据库查询结果

有了 MCP Server 后：

┌──────────┐     MCP 协议      ┌───────────────────┐
│ AI 工具   │ ←──────────────→ │ 你的 MCP Server    │
│ (Claude)  │   "查询 Redis"    │                   │
│           │ ←──────────────→ │  ├─ Redis 状态查询  │
│           │   "查 Oracle 表"  │  ├─ Oracle 表结构   │
│           │ ←──────────────→ │  ├─ ZMQ 消息检查    │
│           │   "看 ZMQ 积压"   │  └─ 日志分析        │
└──────────┘                   └───────────────────┘
```

**具体场景**：
- `"帮我查一下 Redis 中 key=user:12345 的值，分析为什么缓存没命中"`
- `"查 Oracle 中 orders 表的索引，分析这条 SQL 为什么慢"`
- `"看 ZMQ PUSH socket 的积压情况，判断是不是消费者挂了"`
- `"读最近的 error 日志，帮我定位 crash 的根因"`

---

## 🛠️ 二、10 分钟最小 MCP Server

### 项目结构

```
mcp-servers/
├── redis_mcp.py       # Redis 查询工具
├── oracle_mcp.py      # Oracle 表结构工具
├── zmq_mcp.py         # ZMQ 监控工具
└── log_mcp.py         # 日志分析工具
```

### 最小示例：Redis MCP Server（Python）

```python
#!/usr/bin/env python3
"""
Redis Inspector MCP Server
让 AI 可以查询 Redis 的 key 信息、内存使用、慢查询等
"""
import json
import sys
import redis
from mcp.server import Server, NotificationOptions
from mcp.server.models import InitializationCapabilities
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

# 配置
REDIS_HOST = "localhost"
REDIS_PORT = 6379

server = Server("redis-inspector")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="redis_get_key_info",
            description="Get type, TTL, and memory usage for a Redis key",
            inputSchema={
                "type": "object",
                "properties": {
                    "key": {
                        "type": "string",
                        "description": "The Redis key to inspect"
                    }
                },
                "required": ["key"]
            }
        ),
        Tool(
            name="redis_scan_keys",
            description="Scan Redis keys matching a pattern (e.g., 'user:*')",
            inputSchema={
                "type": "object",
                "properties": {
                    "pattern": {
                        "type": "string",
                        "description": "Key pattern (glob-style)"
                    },
                    "count": {
                        "type": "integer",
                        "description": "Max keys to return (default 50)",
                        "default": 50
                    }
                },
                "required": ["pattern"]
            }
        ),
        Tool(
            name="redis_slowlog",
            description="Get recent slow queries from Redis",
            inputSchema={
                "type": "object",
                "properties": {
                    "count": {
                        "type": "integer",
                        "description": "Number of entries (default 10)",
                        "default": 10
                    }
                }
            }
        ),
        Tool(
            name="redis_memory_stats",
            description="Get Redis memory usage statistics",
            inputSchema={
                "type": "object",
                "properties": {}
            }
        ),
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    try:
        r = redis.Redis(host=REDIS_HOST, port=REDIS_PORT, 
                        socket_connect_timeout=5, decode_responses=True)
        
        if name == "redis_get_key_info":
            key = arguments["key"]
            if not r.exists(key):
                return [TextContent(type="text", text=f"Key '{key}' does not exist")]
            
            key_type = r.type(key)
            ttl = r.ttl(key)
            memory = r.memory_usage(key)
            encoding = r.object("encoding", key)
            
            return [TextContent(type="text", text=json.dumps({
                "key": key,
                "type": key_type,
                "ttl_seconds": ttl,
                "ttl_human": f"{ttl}s" if ttl > 0 else "no expiry",
                "memory_bytes": memory,
                "encoding": encoding
            }, indent=2))]
        
        elif name == "redis_scan_keys":
            pattern = arguments["pattern"]
            count = arguments.get("count", 50)
            keys = []
            cursor = 0
            while True:
                cursor, batch = r.scan(cursor, match=pattern, count=min(count, 100))
                keys.extend(batch)
                if cursor == 0 or len(keys) >= count:
                    break
            return [TextContent(type="text", text=json.dumps({
                "pattern": pattern,
                "matched": len(keys[:count]),
                "keys": keys[:count]
            }, indent=2))]
        
        elif name == "redis_slowlog":
            count = arguments.get("count", 10)
            logs = r.slowlog_get(count)
            result = []
            for entry in logs:
                result.append({
                    "id": entry["id"],
                    "duration_us": entry["duration"],
                    "duration_ms": entry["duration"] / 1000,
                    "timestamp": entry["start_time"],
                    "command": entry["command"].decode() if isinstance(entry["command"], bytes) else entry["command"]
                })
            return [TextContent(type="text", text=json.dumps(result, indent=2))]
        
        elif name == "redis_memory_stats":
            info = r.info("memory")
            return [TextContent(type="text", text=json.dumps({
                "used_memory_human": info.get("used_memory_human"),
                "used_memory_peak_human": info.get("used_memory_peak_human"),
                "mem_fragmentation_ratio": info.get("mem_fragmentation_ratio"),
                "keyspace_hits": info.get("keyspace_hits"),
                "keyspace_misses": info.get("keyspace_misses"),
                "hit_rate": round(
                    info.get("keyspace_hits", 0) / 
                    max(info.get("keyspace_hits", 0) + info.get("keyspace_misses", 0), 1) * 100, 2
                )
            }, indent=2))]
        
    except redis.ConnectionError as e:
        return [TextContent(type="text", text=f"Redis connection failed: {str(e)}")]
    except Exception as e:
        return [TextContent(type="text", text=f"Error: {str(e)}")]

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            InitializationCapabilities(
                sampling=None,
                experimental=None,
                roots=None
            ),
            notification_options=NotificationOptions()
        )

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

### 安装和配置

```bash
# 1. 安装依赖
pip install redis mcp

# 2. 测试 MCP Server
python3 redis_mcp.py
# (会通过 stdio 等待 MCP 客户端连接)

# 3. 注册到 Claude Code
claude mcp add redis-inspector -- python3 ~/mcp-servers/redis_mcp.py

# 4. 在 Claude Code 中使用
# /mcp → 确认 redis-inspector 已连接
# 然后直接对 AI 说：
# "查一下 Redis 中 user:* 的 key 有多少个，最近有没有慢查询"
```

---

## 📦 三、Oracle MCP Server 示例

```python
#!/usr/bin/env python3
"""Oracle Schema Inspector MCP Server"""
import json
import oracledb
from mcp.server import Server, NotificationOptions
from mcp.server.models import InitializationCapabilities
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

DB_CONFIG = {
    "user": "scott",
    "password": "tiger",
    "dsn": "localhost:1521/ORCL"
}

server = Server("oracle-inspector")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="oracle_table_schema",
            description="Get table schema: columns, types, constraints, indexes",
            inputSchema={
                "type": "object",
                "properties": {
                    "table_name": {"type": "string", "description": "Table name (case-sensitive)"}
                },
                "required": ["table_name"]
            }
        ),
        Tool(
            name="oracle_explain_plan",
            description="Get execution plan for a SQL query",
            inputSchema={
                "type": "object",
                "properties": {
                    "sql": {"type": "string", "description": "SQL query to explain"}
                },
                "required": ["sql"]
            }
        ),
        Tool(
            name="oracle_table_stats",
            description="Get table statistics: row count, size, last analyzed",
            inputSchema={
                "type": "object",
                "properties": {
                    "table_name": {"type": "string"}
                },
                "required": ["table_name"]
            }
        ),
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    conn = oracledb.connect(**DB_CONFIG)
    cursor = conn.cursor()
    
    try:
        if name == "oracle_table_schema":
            table = arguments["table_name"].upper()
            
            # Columns
            cursor.execute("""
                SELECT column_name, data_type, data_length, nullable
                FROM user_tab_columns 
                WHERE table_name = :t 
                ORDER BY column_id
            """, {"t": table})
            columns = [{"name": r[0], "type": r[1], "length": r[2], "nullable": r[3]} 
                      for r in cursor.fetchall()]
            
            # Indexes
            cursor.execute("""
                SELECT index_name, column_name, descend
                FROM user_ind_columns 
                WHERE table_name = :t 
                ORDER BY index_name, column_position
            """, {"t": table})
            indexes = [{"index": r[0], "column": r[1]} for r in cursor.fetchall()]
            
            return [TextContent(type="text", text=json.dumps({
                "table": table, "columns": columns, "indexes": indexes
            }, indent=2))]
        
        elif name == "oracle_explain_plan":
            sql = arguments["sql"]
            cursor.execute(f"EXPLAIN PLAN FOR {sql}")
            cursor.execute("""
                SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY)
            """)
            plan = "\n".join([r[0] for r in cursor.fetchall()])
            return [TextContent(type="text", text=plan)]
        
        elif name == "oracle_table_stats":
            table = arguments["table_name"].upper()
            cursor.execute("""
                SELECT num_rows, blocks, avg_row_len, last_analyzed
                FROM user_tables WHERE table_name = :t
            """, {"t": table})
            row = cursor.fetchone()
            if row:
                return [TextContent(type="text", text=json.dumps({
                    "table": table, "num_rows": row[0], "blocks": row[1],
                    "avg_row_len": row[2], "last_analyzed": str(row[3])
                }, indent=2))]
            return [TextContent(type="text", text=f"Table {table} not found")]
    
    finally:
        cursor.close()
        conn.close()

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream,
            InitializationCapabilities(sampling=None, experimental=None, roots=None),
            notification_options=NotificationOptions())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

```bash
# 注册
claude mcp add oracle-inspector -- python3 ~/mcp-servers/oracle_mcp.py
```

---

## ⚡ 四、ZMQ 监控 MCP Server

```python
#!/usr/bin/env python3
"""ZMQ Monitor MCP Server - Check message queue status"""
import json
import zmq
from mcp.server import Server, NotificationOptions
from mcp.server.models import InitializationCapabilities
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

server = Server("zmq-monitor")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="zmq_check_endpoint",
            description="Check if a ZMQ endpoint is reachable (PUB/SUB health check)",
            inputSchema={
                "type": "object",
                "properties": {
                    "endpoint": {
                        "type": "string",
                        "description": "ZMQ endpoint, e.g. tcp://localhost:5555"
                    },
                    "socket_type": {
                        "type": "string",
                        "description": "ZMQ socket type: PUB, SUB, PUSH, PULL, ROUTER, DEALER",
                        "enum": ["PUB", "SUB", "PUSH", "PULL", "ROUTER", "DEALER"]
                    }
                },
                "required": ["endpoint", "socket_type"]
            }
        ),
        Tool(
            name="zmq_send_test_message",
            description="Send a test message to a ZMQ endpoint and wait for response (REQ-REP check)",
            inputSchema={
                "type": "object",
                "properties": {
                    "endpoint": {"type": "string"},
                    "message": {"type": "string", "description": "Test message payload"}
                },
                "required": ["endpoint", "message"]
            }
        ),
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    context = zmq.Context()
    
    try:
        if name == "zmq_check_endpoint":
            endpoint = arguments["endpoint"]
            socket_type = getattr(zmq, arguments["socket_type"])
            socket = context.socket(socket_type)
            socket.setsockopt(zmq.RCVTIMEO, 3000)
            socket.setsockopt(zmq.LINGER, 0)
            
            try:
                if socket_type == zmq.SUB:
                    socket.connect(endpoint)
                    socket.setsockopt(zmq.SUBSCRIBE, b"")
                    msg = socket.recv(flags=zmq.NOBLOCK)
                    status = "receiving messages"
                elif socket_type == zmq.PUSH:
                    socket.connect(endpoint)
                    status = "connected (PUSH - cannot verify consumer)"
                else:
                    socket.connect(endpoint)
                    status = "connected"
            except zmq.Again:
                status = "connected but no messages (normal for SUB with no traffic)"
            except Exception as e:
                status = f"error: {str(e)}"
            finally:
                socket.close()
            
            return [TextContent(type="text", text=json.dumps({
                "endpoint": endpoint, "type": arguments["socket_type"], "status": status
            }, indent=2))]
        
        elif name == "zmq_send_test_message":
            socket = context.socket(zmq.REQ)
            socket.setsockopt(zmq.RCVTIMEO, 5000)
            socket.setsockopt(zmq.LINGER, 0)
            socket.connect(arguments["endpoint"])
            socket.send_string(arguments["message"])
            try:
                reply = socket.recv_string()
                return [TextContent(type="text", text=json.dumps({
                    "sent": arguments["message"], "reply": reply, "status": "ok"
                }, indent=2))]
            except zmq.Again:
                return [TextContent(type="text", text="No reply (timeout 5s)")]
            finally:
                socket.close()
    
    finally:
        context.term()

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream,
            InitializationCapabilities(sampling=None, experimental=None, roots=None),
            notification_options=NotificationOptions())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

---

## 🔒 五、安全设计原则

| 原则 | 说明 |
|------|------|
| **只读优先** | 默认只提供查询工具，写入操作需要额外审批 |
| **连接超时** | 所有外部连接必须设置超时（5 秒） |
| **权限最小化** | 使用只读数据库账户，切勿用生产写账户 |
| **敏感信息脱敏** | 密码、token、PII 数据在返回前脱敏 |
| **速率限制** | MCP Server 中实现调用频率限制 |
| **审计日志** | 记录所有 AI 通过 MCP 执行的操作 |

---

## 🚀 六、团队 MCP Server 目录建议

```
~/mcp-servers/
├── redis_mcp.py          # Redis 查询
├── oracle_mcp.py         # Oracle 表结构
├── zmq_mcp.py            # ZMQ 健康检查
├── log_mcp.py            # 日志分析（grep + jq）
├── cmake_mcp.py          # 构建系统查询
└── README.md             # 使用说明
```

### 注册到 Claude Code

```bash
# 一次性注册所有 MCP Server
claude mcp add redis    -- python3 ~/mcp-servers/redis_mcp.py
claude mcp add oracle   -- python3 ~/mcp-servers/oracle_mcp.py
claude mcp add zmq      -- python3 ~/mcp-servers/zmq_mcp.py

# 在 Claude Code 会话中管理
/mcp    # 查看所有已注册的 MCP Server
```

---

*文档生成日期：2026-06-24 | 数据来源：MCP 官方协议文档、Anthropic MCP SDK、社区实践*
