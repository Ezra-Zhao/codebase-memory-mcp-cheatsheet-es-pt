# 查询示例 | Query Examples

个人学习整理，配合主 README 使用。语法以 openCypher 只读子集为准。

## CLI 一次性查询

```bash
# 先列出项目，拿到 project 的 name
codebase-memory-mcp cli list_projects

# 找所有名字带 Handler 的函数
codebase-memory-mcp cli search_graph --project my-project --name-pattern '.*Handler.*' --label Function

# 追一个函数的调用方与被调用方
codebase-memory-mcp cli trace_path --project my-project --function-name Search --direction both
```

English: Every MCP tool doubles as a one-shot CLI command; `--format json` gives machine-readable output.

## Cypher 常用模式

```cypher
-- 死代码候选：没有任何调用者的函数
MATCH (f:Function) WHERE NOT EXISTS { (f)<-[:CALLS]-() } RETURN f.name LIMIT 20

-- 某条路由由哪个函数处理
MATCH (r:Route)-[:HANDLES]->(f:Function) RETURN r, f LIMIT 10

-- 跨服务：谁通过 HTTP 调用了别人
MATCH (a)-[:HTTP_CALLS]->(b) RETURN a, b LIMIT 10
```

English: `query_graph` is read-only; write clauses and unsupported functions fail with an explicit `unsupported` error instead of silently returning nothing.

## 改动前的影响评估

1. 在分支上改完代码，先跑 `detect_changes`，拿到受影响符号和风险分级。
2. 对高风险符号跑 `trace_path`（`calls` 模式，方向 both），确认调用方都在预期内。
3. 把影响半径写进 PR 描述再提交。

English: Run `detect_changes` after an edit, then `trace_path` on the risky symbols before opening the PR.
