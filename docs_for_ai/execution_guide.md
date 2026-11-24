# 执行指南：API 配置、Docker、多检索器集成与参数调优

适用对象：需要运行、集成或调优 GPT Researcher 的 AI/工程师。所有配置名与默认值以源码为准（`gpt_researcher/config/variables/default.py`, `.env.example`, `docker-compose.yml`）。

## 1. 环境准备

### 1.1 本地安装

```bash
git clone https://github.com/assafelovic/gpt-researcher.git
cd gpt-researcher
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # 填入 API key
```

- Python：`pyproject.toml:22` 要求 `>=3.12`；README 建议 3.12+（`README.md:104`）。
- pip 包用户：`pip install gpt-researcher` 后可直接 `from gpt_researcher import GPTResearcher`（`README.md:143-159`）。
- Claude Skill 用户：`npx skills add assafelovic/gpt-researcher`（`README.md:47-55`）。

### 1.2 最小环境变量

`.env.example` 的前两项：

```bash
OPENAI_API_KEY=...
TAVILY_API_KEY=...
```

其他按需（注意 `.env.example` 的具体行）：

| 变量 | 用途 |
|---|---|
| `OPENAI_BASE_URL` | OpenAI 兼容端点；`create_chat_completion()` 会转成 `openai_api_base`（`utils/llm.py:112-115`） |
| `TYPESAFE_API_KEY` | 有它则 `CONTEXT_FILTER=auto` 选 Jev |
| `BRAVE_API_KEY`, `XQUIK_API_KEY`, `GETXAPI_API_KEY`, `GROUNDROUTE_API_KEY` | 对应检索器 |
| `OPENALEX_EMAIL`, `OPENALEX_API_KEY`, `NCBI_API_KEY`, `PUBMED_DB=pmc` | 学术检索器 |
| `DOC_PATH=./my-docs` | 本地文档目录；`Config` 会 `makedirs`（`config.py:249-251`） |
| `LANGCHAIN_TRACING_V2`, `LANGCHAIN_API_KEY`, `LANGCHAIN_PROJECT` | LangSmith |
| `MONOCLE_TRACING`, `MONOCLE_EXPORTERS`, `OKAHU_API_KEY` | Monocle（可选 extra `monocle`，`multi_agents/main.py:12-35`） |
| `ALLOW_PRIVATE_URLS=true` | **仅当**需要研究内网地址时开启（默认拒绝，见 SSRF 防护） |
| `LLM_TIMEOUT`, `LOG_LEVEL`, `LOG_TO_FILE`, `LOG_DIR`, `HOST`, `PORT` | 运行/日志 |

**配置优先级**：环境变量 > 配置文件 > `DEFAULT_CONFIG`（`config/config.py:54-75,149-174`）。配置文件是 JSON，路径可由 `config_path` 参数或 `CONFIG_PATH` 环境变量指定；未知配置键会被 `setattr` 成属性但不会校验，改配置时留意拼写。

## 2. 三种运行方式

### 2.1 Python API（推荐用于集成）

```python
import asyncio
from gpt_researcher import GPTResearcher

async def main():
    researcher = GPTResearcher(
        query="What are the latest AI developments?",
        report_type="research_report",   # research_report | detailed_report | deep | ...
        report_source="web",             # web | local | hybrid | azure | ...
        verbose=True,
    )
    context = await researcher.conduct_research()
    report = await researcher.write_report()
    print(report)
    print("cost:", researcher.get_costs(), "sources:", researcher.get_source_urls())

asyncio.run(main())
```

`conduct_research()` 与 `write_report()` 分离很重要：先拿 context，才能检查是否为空、再决定是否写报告；`ReportGenerator.write_report()` 对空 context 会弃答（`skills/writer.py:79-88`）。

### 2.2 FastAPI 服务

```bash
# 仓库根目录（main.py 会 load_dotenv 并配置日志）
python -m uvicorn main:app --reload --port 8000
# 或
python main.py     # 读取 HOST/PORT，默认 127.0.0.1:8000
```

访问 `http://localhost:8000`。常见请求：

```bash
curl -X POST http://localhost:8000/report/ \
  -H 'Content-Type: application/json' \
  -d '{"task":"AI regulation in the EU","report_type":"research_report",
       "report_source":"web","tone":"objective","repo_name":"","branch_name":"",
       "generate_in_background":false}'
```

`ResearchRequest` 必填字段：`task`, `report_type`, `report_source`, `tone`, `repo_name`, `branch_name`；`headers` 可传retriever 配置，`generate_in_background` 默认 true（`backend/server/app.py:53-61`）。tone 值用 `Tone` 枚举名（如 `objective`, `formal`, `analytical`），服务内用 `Tone[research_request.tone]` 转换（backend/server/app.py:342）。

响应（`backend/server/app.py:360-374`）：`report`, `docx_path`, `pdf_path`, `research_id`，以及 `research_information`（source_urls、research_costs、visited_urls、research_images）。

WebSocket 客户端连接 `ws://localhost:8000/ws`，发送包含 command/data 的 JSON；进度由 `stream_output()` 以 `{type, step, content, data?}` 推送（`retrievers/utils.py:14-42`）。

### 2.3 CLI

```bash
python cli.py "why is Nvidia stock going up?" \
  --report_type research_report --tone objective \
  --query_domains reuters.com,arxiv.org --report_source web
```

`--report_type` 必需；`detailed_report` 会走 `DetailedReport` 多子主题流程；`deep` 走递归深度研究。输出写入 `outputs/`：带 YAML frontmatter 的 `.md`，默认同时生成 PDF/DOCX（`--no-pdf`, `--no-docx` 可关闭）。文件名由 LLM 生成标题后清洗（`cli.py:146-263,270-364`）。

### 2.4 Docker

```bash
cp .env.example .env    # 填 key
docker compose up --build
```

默认启动两个服务（`docker-compose.yml`）：

| 服务 | 端口 | 说明 |
|---|---|---|
| `gpt-researcher` | 8000 | 后端；根 `Dockerfile` 的 CMD 为 `uvicorn main:app --host ${HOST} --port ${PORT} --workers ${WORKERS}` |
| `gptr-nextjs` | 3000 | Next.js 前端，服务端通过 `GPTR_API_URL=http://gpt-researcher:8000` 访问后端 |

其他 profile：`test`（`gpt-researcher-tests`）、`discord`（discord bot）。卷挂载：`my-docs`, `outputs`, `logs`。

注意：`backend/Dockerfile` 中的 `uvicorn app.main:app` 与仓库实际模块布局不一致；compose 使用根 `Dockerfile`，优先以它为准。配置浏览器抓取时，根镜像已安装 Chromium/Chromedriver/Firefox/Geckodriver。

## 3. 多检索器集成

### 3.1 选择内置检索器

设置 `RETRIEVER`（逗号分隔）或通过 headers/API 传 `retriever` / `retrievers`：

```bash
export RETRIEVER=tavily,arxiv,pubmed_central
```

优先级：`headers.retrievers` > `headers.retriever` > `cfg.retrievers` > `cfg.retriever` > 默认 `TavilySearch`（`actions/retriever.py:151-194`）。名称会被 strip；无效名称回退默认。可用内置名称见 `retrievers/utils.py:62-84` 与目录名扫描（:86-107）：tavily, groundroute, custom, duckduckgo, searchapi, serper, serpapi, google, searx, bing, brave, bocha, arxiv, semantic_scholar, pubmed_central, exa, crw, getxapi, mcp, xquik, openalex。

**关键契约**：检索器要么声明 `requires_scraping = True`（返回 URL，后续抓取），要么声明 `False`（返回含 `raw_content` 的全文）。示例：

```python
from gpt_researcher.retrievers.base import BaseRetriever

class MyRetriever(BaseRetriever):
    requires_scraping = False  # 已自带全文

    def __init__(self, query, query_domains=None, **kwargs):
        self.query = query

    def search(self, max_results: int = 7):
        return [{"url": "https://example.org/a", "raw_content": "...", "title": "..."}]
```

类必须可由 `retriever(query, query_domains=...)` 实例化；若要被工厂按名字找到，可在 `get_retriever()` 注册，或打包成插件并用 entry-point 组 `gpt_researcher.retrievers`（`actions/retriever.py:13,140-148`）。

### 3.2 多检索器如何被使用

- 初始搜索（规划用）只用**第一个** retriever（`skills/researcher.py:59-74`）。
- 每个 sub-query 的 `_search_relevant_source_urls()` 会遍历**所有非 MCP** retriever（:900-924）。
- `GPTResearcher.quick_search(..., all_retrievers=True)` 会并发查询全部并按 URL 去重（`gpt_researcher/agent.py:605-639`）。
- 单个 provider 失败不会终止其他 provider（异常事件 `search_failed`）。

### 3.3 MCP 数据源

```bash
export RETRIEVER=tavily,mcp
```

```python
researcher = GPTResearcher(
    query="What are the top open source web research agents?",
    mcp_configs=[{
        "name": "github",
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-github"],
        "env": {"GITHUB_TOKEN": os.getenv("GITHUB_TOKEN")},
    }],
    mcp_strategy="fast",   # fast | deep | disabled
)
```

`mcp_configs` 也支持 `connection_url`, `connection_type`（stdio/websocket/http）, `connection_token`, `tool_name`（`gpt_researcher/agent.py:123-132`）。配置通过 `self.cfg.retrievers` 局部追加 `mcp`，**不修改环境变量**（`_process_mcp_configs()`, gpt_researcher/agent.py:312-338）。

策略：`fast`（默认，原始 query 跑一次并缓存）/`deep`（每个 sub-query 都跑）/`disabled`。兼容旧参数：`mcp_max_iterations=0→disabled`, `1→fast`, `-1→deep`（`gpt_researcher/agent.py:270-283`）。注意前端 Tavily MCP preset 与直接 Tavily retriever 同时启用时会被去重跳过（`skills/researcher.py:523-548`）。

## 4. 上下文过滤与性能调优

### 4.1 过滤器

```bash
export CONTEXT_FILTER=auto    # auto | jev | keyword | embeddings | none
export TYPESAFE_API_KEY=...   # auto 下启用 Jev
```

| 场景 | 建议 |
|---|---|
| 有 `TYPESAFE_API_KEY`，追求句级相关性 | `jev`/`auto`；Jev 按“对问题的有用性”四级打分而非主题相似度 |
| 无外部 API、低成本、低延迟 | `keyword`（BM25，本地）；README 基准为 0.02s/报告（`README.md:237-246`） |
| 已有 embedding 基础设施 | `embeddings`；注意构建 embedding provider 与成本 |
| 开放式问题希望最宽上下文 | `none`；README 指出成本可高约 65% |
| 小结果集 | 自动走 `COMPRESSION_THRESHOLD`（默认 8000 字符）捷径，无需调参 |

Jev 相关旋钮：`JEV_CHUNK_SIZE`（1000）, `JEV_MIN_SCORE`（1.5）, `JEV_CONCURRENCY`（64）, `JEV_MODEL`（jev-latest）, `TYPESAFE_BASE_URL`。keyword：`KEYWORD_RELATIVE_THRESHOLD`（0.5）, `KEYWORD_MAX_RESULTS`（25）。embeddings：`SIMILARITY_THRESHOLD`, `MAX_CONTENT_CHARS`（50000）。

### 4.2 抓取并发与限速

```bash
export MAX_SCRAPER_WORKERS=5
export SCRAPER_RATE_LIMIT_DELAY=6.0   # Firecrawl 免费版 10 req/min → 6 秒/请求
export FIRECRAWL_CONCURRENCY=2
```

`WorkerPool` 的 semaphore 限制本池并发，全局 singleton limiter 限制跨池请求频率（`utils/workers.py:8-50`）；文档示例见 `.env.example` “Scraper Configuration”。提高并发前先确认目标站点/服务条款。

### 4.3 模型与 token 预算

```bash
export SMART_LLM=openai:gpt-5.4-mini
export SMART_TOKEN_LIMIT=16000
export STRATEGIC_LLM=openai:gpt-5.4
export STRATEGIC_TOKEN_LIMIT=16000
export REASONING_EFFORT=medium    # low | medium | high（见 ReasoningEfforts）
export LLM_KWARGS='{"top_p":0.9}'
```

- `FAST_LLM` 用于摘要/快速调用，`SMART_LLM` 写报告，`STRATEGIC_LLM` 规划/深度研究（`config/variables/default.py:8-16`）。
- 推理模型的 `max_tokens` 包含思考 token，需留余量（default.py:11-13 注释；`utils/llm.py:82-88` 模型能力适配）。
- `LLM_KWARGS` 会覆盖计算出的参数（`utils/llm.py:98-110`），用于模型只允许单一 temperature 等场景。
- `create_chat_completion()` 对未知模型有 `supports_temperature()` 探测；不支持 temperature 时传 `None`。

### 4.4 研究规模

```bash
export MAX_SEARCH_RESULTS_PER_QUERY=5
export MAX_ITERATIONS=3
export MAX_SUBTOPICS=3
export TOTAL_WORDS=1200
export DEEP_RESEARCH_BREADTH=3
export DEEP_RESEARCH_DEPTH=2
export DEEP_RESEARCH_CONCURRENCY=4
```

深度研究规模约按 `breadth × depth` 增长，README 提示典型耗时 7–12 分钟、约 $1.30（`README.md:219-229`）；这些数字是仓库说明而非当前账单保证，实际用 `researcher.get_step_costs()` 观察。

### 4.5 本地文档

```bash
export DOC_PATH=./my-docs
# 然后 report_source="local" / "hybrid"，或通过前端 "My Documents" 选项
```

支持 PDF、txt、CSV、Excel、Markdown、PPT、Word（`README.md:279-291`）。`hybrid` 会并发跑本地与 web 两路，并用 `join_local_web_documents()` 合并。Granite prompt family 对两路上下文的类型处理有专门覆写（`prompts.py:756-836`）。

## 5. 多智能体运行

```bash
cd multi_agents
python main.py               # 读取 multi_agents/task.json
```

`task.json` 关键字段（`multi_agents/task.json`）：`query`, `max_sections`, `max_plan_revisions`, `max_fact_check_revisions`（可选）, `publish_formats.{markdown,pdf,docx}`, `include_human_feedback`, `follow_guidelines`, `guidelines`, `model`, `verbose`。`STRATEGIC_LLM` 环境变量会覆盖 task 的 model（`multi_agents/main.py:58-65`）。

AG2 版本：`python multi_agents/ag2/main.py`（单独 requirements，`multi_agents/ag2/requirements.txt`）。`deep_agents` 版本见 `deep_agents/main.py` 与其 README/BENCHMARK。

## 6. 观测、成本与调试

- 结构化事件：给 `GPTResearcher(research_observer=obj)` 传带 `on_event(event, **fields)` 的对象；observer 异常被吞掉，不影响研究。事件清单见 `docs_for_ai/architecture.md` §9。
- LangSmith：`LANGCHAIN_TRACING_V2=true`, `LANGCHAIN_API_KEY`, `LANGCHAIN_PROJECT`；`multi_agents/main.py:42-44` 在检测到 key 时自动打开 tracing。
- 成本：`GET /report/...` 响应中的 `research_costs`；Python 侧 `get_costs()` / `get_step_costs()`；Jev 的打分成本单独按 input token 计算（`jev_filter.py:127-129`）。
- 日志：`main.py:13-43` 配置 LOG_LEVEL/文件轮转（10MB × 5）；容器内可设 `LOG_TO_FILE=false` 只输出到控制台。

## 7. 故障排查

| 症状 | 排查点 |
|---|---|
| `Unsupported <provider>` | `FAST_LLM/SMART_LLM/STRATEGIC_LLM` 的 provider 必须在 `_SUPPORTED_PROVIDERS`（`llm_provider/generic/base.py:19-48`）；格式 `provider:model` |
| `Invalid retriever(s) found` | `RETRIEVER` 名称需在 `get_all_retriever_names()` 返回集合中；插件需正确注册 entry point |
| 报告没有引用/来源 | 检查 `requires_scraping`、抓取 `fetch_failed` 事件、`visited_urls` 去重；如果 context 为空，报告会是弃答文本 |
| Jev 未生效 | 确认 `TYPESAFE_API_KEY`, `CONTEXT_FILTER`；`JevError` 会静默退化 keyword，查 warning 日志 |
| 抓取太慢/被封 | 调低 `MAX_SCRAPER_WORKERS`、增大 `SCRAPER_RATE_LIMIT_DELAY`，或换 SCRAPER（bs/browser/firecrawl/nodriver...） |
| 输出被截断 | 增大 `SMART_TOKEN_LIMIT`；推理模型还需考虑 reasoning token 占用 |
| `No MCP server configurations found` | 传 `mcp_configs` 或配置 `MCP_SERVERS`；MCP 检索器需要 researcher 实例 |
| 多智能体卡在修订循环 | 检查 `max_plan_revisions` / `max_fact_check_revisions`；超限会强制 accept（`multi_agents/agents/orchestrator.py:107-135`） |
| Docker 起不来 | 确认 `.env` 存在、`8000/3000` 端口空闲；compose 使用根 Dockerfile，`backend/Dockerfile` 不作为该流程入口 |

## 8. 修改代码后的回归建议

仓库测试集中在 `tests/`（大量针对 retriever、scraper、配置隔离、成本、报告组装的单元测试）。从最小相关集合开始：

```bash
pytest -q tests/test_config_default_isolation.py \
        tests/test_custom_retriever_guard.py \
        tests/test_context_manager_error_isolation.py \
        tests/test_arxiv_scraper.py \
        tests/test_cli_detailed_report.py
```

改动 MCP / 多智能体 / 深度研究时，再跑对应的 `tests/test_mcp*`, `tests/test_deep_research*`, `tests/test_multi_agents*` 文件（文件名以仓库实际存在者为准）。需要真实 API 的测试（如 `tests/report-types.py`）先确认 key 与预算。
