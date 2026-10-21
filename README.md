# codebase-memory-mcp 速查表 | Cheatsheet | Guía rápida

> 一页看懂 codebase-memory-mcp：一个本地 MCP 服务器，把整个代码库预先索引成一张持久化知识图谱（函数、类、调用关系、路由、基础设施都成节点和边），编程智能体查图而不是逐个文件翻找。官方报告的五问对比里，查图约用 3,400 tokens，逐文件搜索约用 412,000 tokens，省约 120 倍。单个静态二进制、零依赖、不要 Docker、不要运行时、不要 API key。

- 上游项目：[DeusData/codebase-memory-mcp](https://github.com/DeusData/codebase-memory-mcp)（2026-10-18 实测 46,842 ★）
- 关键词：MCP server、code knowledge graph、Cypher、162 种语言、tree-sitter
- 本页结构：中文主文 → English → Español → Português

---


Apoyo y más: [GitHub Sponsors](https://github.com/sponsors/Ezra-Zhao) · [Gumroad — plantillas y cuadernos que uso](https://ezraflow28.gumroad.com) · [Ko-fi](https://ko-fi.com/ezra777zhao)

## 核心概念

| 概念 | 一句话说明 |
|---|---|
| 知识图谱 | 索引后代码变成图：节点有 Project、File、Class、Function、Method、Route、Table、Resource（K8s）等 30 类；边有 CALLS、IMPORTS、INHERITS、HTTP_CALLS、DATA_FLOWS 等 40 多类 |
| 限定名（qualified name） | 取代码片段时用 `<项目>.<路径部分>.<名字>` 定位函数，先用 `search_graph` 查出来再取 |
| 索引管线 | 内存优先：LZ4 压缩 + 内存 SQLite + Aho-Corasick 模式匹配；Linux 内核 2800 万行代码约 3 分钟索引完，索引后释放内存 |
| 增量与自动同步 | 项目在 MCP 会话中打开时图自动保鲜；用 CLI 或别的会话索引的仓库需要重跑 `index_repository` |
| 基础设施即代码索引 | Dockerfile、docker compose、Kubernetes 清单、Kustomize 都会进图，Service 能一路追到它容器里跑的程序 |
| CBM Lens（Beta） | 浏览器里的图可视化：读源码、看架构与调用图、查覆盖缺口、编辑架构决策记录（ADR） |
| 团队共享图工件 | 索引图可以打包成工件给团队共享，不必每人重索引一遍 |

## 安装

```bash
# macOS / Linux：一键脚本（装二进制 + 自动给检测到的编程智能体写好 MCP 配置）
curl -fsSL https://raw.githubusercontent.com/DeusData/codebase-memory-mcp/main/install.sh | bash

# 可选参数：--skip-config 只装二进制不改智能体配置；--dir=<path> 自定义安装位置
```

Windows（PowerShell）：从上游仓库下载 `install.ps1`，先 `Unblock-File .\install.ps1` 去掉浏览器下载标记，再运行；若报执行策略错误，先 `Set-ExecutionPolicy -Scope Process Bypass`。

其他渠道：GitHub Releases 的预编译包（macOS/Linux/Windows，各 arm64 与 amd64）、Arch 的 AUR、Nix flake、源码构建。装完**重启你的编程智能体**，对它说「Index this project」即可开始索引。

## 17 个 MCP 工具速查

### 索引类（5 个）

| 工具 | 用途 |
|---|---|
| `index_repository` | 把仓库索引进图；大仓建议 `async: true` 后台跑，再用 `status: true` 轮询 |
| `list_projects` | 列出所有已索引项目及节点/边数量 |
| `index_status` | 查某项目的索引状态与新鲜度（显示 `watch` 表示需要重跑） |
| `check_index_coverage` | 查指定路径或范围是否已索引且保鲜；干净只代表没记录到缺口，不等于证明完整 |
| `delete_project` | 删除一个项目及其全部图数据 |

### 查询类（12 个）

| 工具 | 用途 |
|---|---|
| `get_graph_schema` | 先跑这个：看节点/边计数、关系模式、各标签的属性定义 |
| `search_graph` | 结构搜索 + BM25 + 语义搜索，找函数、类、符号 |
| `search_code` | 类 grep 的文本搜索（在已索引文件内） |
| `trace_path` | 追调用链：`calls`（默认）、`data_flow`、`cross_service`、`topology`、`deploy`、`run` 等模式，深度 1–15 |
| `detect_changes` | 把 git diff 映射到受影响符号，给出影响半径和风险分级 |
| `query_graph` | 只读 Cypher 查询（openCypher 子集），见下方示例 |
| `get_code_snippet` | 按限定名读某个函数的源码 |
| `get_file_outline` | 一个文件的声明大纲（按源码顺序） |
| `get_architecture` | 代码库总览：语言、包、路由、热点、聚类、ADR |
| `compare_graphs` | 对比两个索引快照的节点/边增减 |
| `manage_adr` | 架构决策记录的增删改查，可按小节改写而不动其余字节 |
| `ingest_traces` | 把实际观测到的调用路径存进图（两端必须是已索引符号） |

## CLI 模式速查

每个 MCP 工具都能当本地一次性命令用（走共享协调守护进程）：

```bash
codebase-memory-mcp cli index_repository --repo-path /path/to/repo
codebase-memory-mcp cli list_projects
# project 值用 list_projects 返回的 name
codebase-memory-mcp cli search_graph --project my-project --name-pattern '.*Handler.*' --label Function
codebase-memory-mcp cli trace_path --project my-project --function-name Search --direction both
codebase-memory-mcp cli query_graph --project my-project --query 'MATCH (f:Function) RETURN f.name LIMIT 5'
codebase-memory-mcp cli --quiet list_projects --format json
```

输出默认是紧凑树形；加 `--format json` 取机器可读 JSON。`cli <工具> --help` 可看该工具的完整参数。

## Cypher 示例（只读子集）

```cypher
-- 找没人调用的函数（死代码候选）
MATCH (f:Function) WHERE NOT EXISTS { (f)<-[:CALLS]-() } RETURN f.name

-- 看某模块的函数数量
MATCH (f:Function) RETURN count(f)
```

支持 MATCH / WHERE / WITH / RETURN / ORDER BY / LIMIT / UNWIND / UNION 等子句；写操作、MERGE、CALL 等不在子集内，会直接报 `unsupported` 错误而不是返回空结果。

## 典型工作流

1. **新接手一个大仓库**：装好后先 `index_repository` 建图，再让智能体用 `get_architecture` 给总览，用 `trace_path` 追核心链路，比逐文件读快得多。
2. **改代码前评估影响**：先 `detect_changes` 或对目标函数 `trace_path` 看调用方，把影响半径写进改动计划再动手。
3. **查死代码**：用上面的 Cypher 找零调用函数，结合 `TESTS` 边排除被测试引用的，列候选清单人工确认。
4. **跨服务追踪**：HTTP 调用会跨仓库连边（CROSS_HTTP_CALLS 等），`trace_path` 的 `cross_service` 模式能一路追到下游服务。

## 常见坑

- **大仓同步索引会超时**：某些 IDE 客户端单次工具调用有固定时限，同步索引被取消后重试也完不成；改用 `index_repository(async: true)` + `status: true` 轮询。
- **Windows Defender 误报**：官方称发布二进制可能被误报为 `Trojan:Script/Wacatac.B!ml`，是已知误报（同类误报也命中 gh、llama.cpp 等），介意就按上游 SECURITY.md 的方法自验工件。
- **自动同步有条件**：图只在项目处于打开的 MCP 会话时自动保鲜；CLI 索引的仓库改了代码要手动重跑索引。
- **限定名别猜**：`get_code_snippet` 要用 `search_graph` 查到的限定名，手写路径容易对不上。
- **Cypher 是子集**：复杂查询先小步试，报 unsupported 就拆成多步，不要以为是图里没数据。

---

## English (summary)

codebase-memory-mcp (by DeusData, ~46.8k stars as of 2026-10-18) is a local MCP server that indexes a codebase into a persistent knowledge graph, so coding agents query the graph instead of grepping file by file — about 120x fewer tokens in the project's reported five-query comparison (~3,400 vs ~412,000 tokens). It ships as a single static binary with zero dependencies: no Docker, no language runtime, no API keys. 162 languages are supported via vendored tree-sitter grammars compiled into the binary, and infrastructure-as-code (Dockerfiles, compose files, Kubernetes manifests, Kustomize overlays) is indexed as graph nodes too.

Install on macOS/Linux with the one-line `install.sh` from the upstream repo (options `--skip-config`, `--dir=<path>`), on Windows with `install.ps1` (run `Unblock-File` first). Restart your agent and say "Index this project".

The 17 tools split into indexing (`index_repository`, `list_projects`, `index_status`, `check_index_coverage`, `delete_project`) and querying (`get_graph_schema` — run it first, `search_graph`, `search_code`, `trace_path` with calls/data_flow/cross_service/topology/deploy/run modes, `detect_changes` for diff impact and blast radius, `query_graph` for a read-only openCypher subset, `get_code_snippet` by qualified name, `get_file_outline`, `get_architecture`, `compare_graphs`, `manage_adr`, `ingest_traces`). Every tool also works as a one-shot CLI command: `codebase-memory-mcp cli <tool> --flags`.

Watch out: synchronous indexing of a huge repo can exceed MCP client call deadlines — use `async: true` and poll with `status: true`; Microsoft Defender may false-positive the binary (Wacatac) per the project's SECURITY notes; auto-sync only stays fresh while the project is open in an MCP session; qualified names must come from `search_graph`, not guesswork; unsupported Cypher fails loudly with an `unsupported` error instead of returning empty results.

---

## Español (guía rápida — traducción de estudio, pendiente de revisión por un hablante nativo)

codebase-memory-mcp (de DeusData, ~46,8 mil estrellas en octubre de 2026) es un servidor MCP local que indexa todo tu código en un grafo de conocimiento persistente. En lugar de buscar archivo por archivo, el agente de programación consulta el grafo: en la comparación de cinco consultas publicada por el proyecto, consultar el grafo usó unas 3.400 tokens frente a unas 412.000 de la búsqueda por archivos (unas 120 veces menos). Es un único binario estático, sin dependencias: sin Docker, sin runtimes, sin claves API. Soporta 162 lenguajes con gramáticas tree-sitter integradas, y también indexa infraestructura como código (Dockerfile, docker compose, manifiestos de Kubernetes, Kustomize).

Instalación en macOS/Linux: ejecuta el `install.sh` del repositorio oficial (opciones `--skip-config` y `--dir=<ruta>`). En Windows usa `install.ps1` tras `Unblock-File`. Reinicia tu agente y dile «Index this project».

Las 17 herramientas se dividen en indexación (`index_repository`, `list_projects`, `index_status`, `check_index_coverage`, `delete_project`) y consulta (`get_graph_schema` — ejecútala primero —, `search_graph`, `search_code`, `trace_path` con modos calls/data_flow/cross_service/topology/deploy/run, `detect_changes` para impacto de un diff, `query_graph` con un subconjunto de solo lectura de openCypher, `get_code_snippet` por nombre cualificado, `get_file_outline`, `get_architecture`, `compare_graphs`, `manage_adr`, `ingest_traces`). Todas funcionan también como comandos CLI: `codebase-memory-mcp cli <herramienta> --opciones`.

Ejemplo Cypher para detectar código muerto: `MATCH (f:Function) WHERE NOT EXISTS { (f)<-[:CALLS]-() } RETURN f.name`.

Errores frecuentes: indexar un repositorio enorme en modo síncrono puede superar el límite de tiempo del cliente MCP — usa `async: true` y consulta con `status: true`; Microsoft Defender puede marcar el binario como falso positivo (Wacatac), verifícalo con el método del propio proyecto si te preocupa; la sincronización automática solo mantiene el grafo fresco mientras el proyecto está abierto en una sesión MCP; los nombres cualificados deben salir de `search_graph`, no de la intuición; el Cypher no soportado devuelve un error explícito, no resultados vacíos.

---

## Português (guia rápido — tradução de estudo, aguardando revisão de um falante nativo)

O codebase-memory-mcp (da DeusData, ~46,8 mil estrelas em outubro de 2026) é um servidor MCP local que indexa todo o seu código num grafo de conhecimento persistente. Em vez de varrer arquivo por arquivo, o agente de programação consulta o grafo: na comparação de cinco consultas publicada pelo projeto, a consulta ao grafo usou cerca de 3.400 tokens contra cerca de 412.000 da busca por arquivos (cerca de 120 vezes menos). É um único binário estático, sem dependências: sem Docker, sem runtime, sem chaves de API. Suporta 162 linguagens com gramáticas tree-sitter embutidas e também indexa infraestrutura como código (Dockerfile, docker compose, manifestos do Kubernetes, Kustomize).

Instalação no macOS/Linux: execute o `install.sh` do repositório oficial (opções `--skip-config` e `--dir=<caminho>`). No Windows, use o `install.ps1` após `Unblock-File`. Reinicie o seu agente e diga «Index this project».

As 17 ferramentas se dividem em indexação (`index_repository`, `list_projects`, `index_status`, `check_index_coverage`, `delete_project`) e consulta (`get_graph_schema` — rode primeiro —, `search_graph`, `search_code`, `trace_path` com os modos calls/data_flow/cross_service/topology/deploy/run, `detect_changes` para o impacto de um diff, `query_graph` com um subconjunto somente leitura do openCypher, `get_code_snippet` por nome qualificado, `get_file_outline`, `get_architecture`, `compare_graphs`, `manage_adr`, `ingest_traces`). Todas também funcionam como comandos CLI: `codebase-memory-mcp cli <ferramenta> --opções`.

Exemplo de Cypher para achar código morto: `MATCH (f:Function) WHERE NOT EXISTS { (f)<-[:CALLS]-() } RETURN f.name`.

Armadilhas comuns: indexar um repositório enorme em modo síncrono pode estourar o prazo da chamada do cliente MCP — use `async: true` e acompanhe com `status: true`; o Microsoft Defender pode marcar o binário como falso positivo (Wacatac), verifique pelo método do próprio projeto se isso te preocupa; a sincronização automática só mantém o grafo atualizado enquanto o projeto está aberto numa sessão MCP; nomes qualificados devem vir do `search_graph`, não de adivinhação; Cypher não suportado retorna um erro explícito, não resultados vazios.

---

## 来源与声明

- 灵感来源 / Source / Fuente / Fonte: [DeusData/codebase-memory-mcp](https://github.com/DeusData/codebase-memory-mcp)
- 本仓是个人学习整理：内容用我自己的话总结自上游公开文档，不搬运原文；工具清单与命令以 2026-10-18 的上游 README 为准，性能数字为上游自报口径，请以官方 Performance 章节的范围说明为准。
- 西语、葡语部分为学习译稿，待母语校对。Las secciones en español y portugués son traducciones de estudio pendientes de revisión nativa.
- License: MIT © 2026 Guangyi Zhao
