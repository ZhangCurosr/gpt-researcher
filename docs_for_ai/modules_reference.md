# 核心类与关键接口清单

本文是源码符号目录，供零上下文 AI 直接定位修改点。`路径:行号` 为本检出版本锚点。

## 1. 顶层 API

| 符号 | 位置 | 签名/职责 |
|---|---|---|
| `GPTResearcher` | `gpt_researcher/agent.py:42` | 主类；包唯一导出（`gpt_researcher/__init__.py`） |
| `GPTResearcher.__init__` | `gpt_researcher/agent.py:58-218` | 配置、retrievers、skills、MCP 策略装配；关键参数：`query`, `report_type`, `report_format`, `report_source`, `tone`, `source_urls`, `document_urls`, `complement_source_urls`, `query_domains`, `documents`, `vector_store`, `vector_store_filter`, `config_path`, `websocket`, `agent`, `role`, `parent_query`, `subtopics`, `visited_urls`, `verbose`, `context`, `headers`, `max_subtopics`, `log_handler`, `prompt_family`, `mcp_configs`, `mcp_max_iterations`, `mcp_strategy`, `research_observer` |
| `conduct_research` | `gpt_researcher/agent.py:364-434` | → context（deep 模式走 `_handle_deep_research`） |
| `write_report` | `gpt_researcher/agent.py:484-525` | → Markdown；委托 `ReportGenerator.write_report` |
| `quick_search` | `gpt_researcher/agent.py:552-603` | → `list[dict] | str`；`all_retrievers=True` 并发合并 |
| `get_research_context/get_source_urls/get_costs/get_step_costs` | `gpt_researcher/agent.py:766-796` | 状态读取 |
| `add_research_sources/add_research_images` | `gpt_researcher/agent.py:697-719` | 累积来源与图片 |
| `add_costs` | `gpt_researcher/agent.py:806-833` | 按 `_current_step` 分桶计费 |

## 2. Skills（`gpt_researcher/skills/`）

| 类 | 文件:行 | 关键方法 | 返回 |
|---|---|---|---|
| `ResearchConductor` | `skills/researcher.py:22` | `conduct_research()` :121；`plan_research()` :86；`_get_context_by_web_search()` :329；`_get_context_by_urls()` :276；`_get_context_by_vectorstore()` :296；`_process_sub_query()` :551；`_process_sub_query_with_vectorstore()` :831；`_search_relevant_source_urls()` :890；`_scrape_data_by_urls()` :1026；`_execute_mcp_research()` :696；`_combine_mcp_and_web_context()` :778 | context 字符串 / URL 列表 |
| `ReportGenerator` | `skills/writer.py:20` | `write_report()` :49；`write_report_conclusion()` :144；`write_introduction()` :183；`get_subtopics()` :214；`get_draft_section_titles()` :243 | Markdown / list |
| `ContextManager` | `skills/context_manager.py:19` | `get_similar_content_by_query()` :38；`get_similar_content_by_query_with_vectorstore()` :66；`get_similar_written_contents_by_draft_section_titles()` :89 | context 字符串 / list[str] |
| `BrowserManager` | `skills/browser.py:14` | `browse_urls()` :37；`select_top_images()` :89 | scraped_data / image URLs |
| `SourceCurator` | `skills/curator.py:20` | `curate_sources()` :38 | `list[dict]`（失败回退原数据） |
| `DeepResearchSkill` | `skills/deep_research.py:257` | `generate_search_queries()` :272；`generate_research_plan()` :306；`process_research_results()` :361；`deep_research()` :396；`run()` :597 | learnings/context dict / 最终 context |
| `ImageGenerator` | `skills/image_generator.py:22` | `is_enabled()` :75；`plan_and_generate_images()` :83；`analyze_report_for_images()` :274；`generate_images_for_report()` :472；`process_image_placeholders()` :648 | 图片元数据列表 |

辅助数据类：`ResearchProgress`（`skills/deep_research.py:246`）；模块常量 `MAX_CONTEXT_WORDS = 25000`, `JSON_BLOCK_PATTERNS`, `QUERY_LINE_PATTERN`, `URL_PATTERN` 等（:18-45）。

## 3. Actions（`gpt_researcher/actions/`）

| 函数 | 位置 | 关键参数 → 返回 |
|---|---|---|
| `choose_agent` | `agent_creator.py:18` | query/cfg/parent_query → `(server, agent_role_prompt)` |
| `handle_json_error` / `extract_json_with_regex` / `_agent_pair_from_payload` | `agent_creator.py:74,133,122` | 容错解析 |
| `get_search_results` | `query_processing.py:44` | query/retriever/query_domains/researcher/max_results → `list[dict]`；内部 `asyncio.to_thread` |
| `generate_sub_queries` / `plan_research_outline` / `_normalize_sub_queries` | `query_processing.py:83,158,12` | → `list[str]` |
| `get_retriever` / `get_retrievers` / `get_default_retriever` / `_load_plugin_retriever` | `retriever.py:16,151,197,140` | 名称 → 类；默认 `TavilySearch` |
| `scrape_urls` | `web_scraping.py:12` | → `(scraped_data, images)` |
| `filter_urls` / `extract_main_content` / `process_scraped_data` | `web_scraping.py:60,84,99` | 工具函数；`extract_main_content` 目前原样返回 HTML（:84-97） |
| `write_report_introduction` / `write_conclusion` / `summarize_url` / `generate_draft_section_titles` / `generate_report` | `report_generation.py:12,63,115,160,209` | → Markdown / `list[str]` |
| `extract_headers` / `extract_sections` / `table_of_contents` / `add_references` | `markdown_processing.py:6,45,76,105` | Markdown 工具；`add_references` 排序去重 URL |
| `calculate_cost` / `format_token_count` / `create_cost_callback` | `utils.py:62,117,178` | 成本工具 |

`actions/__init__.py` 重新导出上述函数；`from gpt_researcher.actions import generate_report` 等是稳定导入方式。

## 4. 检索器（`gpt_researcher/retrievers/`）

### 4.1 基类与工厂

- `BaseRetriever`（`retrievers/base.py:35`）：`requires_scraping: bool = True`；抽象 `search(max_results: int = 7) -> List[Dict[str, Any]]`。
- 内置类清单（类定义位置可由 `grep '^class ' retrievers/*/*.py` 复核）：

| 名称 | 类 | 说明 |
|---|---|---|
| `tavily` | `TavilySearch` | 默认；`requires_scraping = True` |
| `google`, `bing`, `brave`, `bocha`, `duckduckgo`, `searx`, `searchapi`, `serpapi`, `serper`, `exa`, `crw`, `groundroute`, `xquik`, `getxapi` | 对应 `*Search` / `Duckduckgo` / `CRWRetriever` | web 搜索后端，需各自 API key 或配置 |
| `arxiv`, `semantic_scholar`, `openalex`, `pubmed_central` | `ArxivSearch`, `SemanticScholarSearch`, `OpenAlexSearch`, `PubMedCentralSearch` | 学术检索；`PubMedCentralSearch.requires_scraping = False`（返回全文） |
| `custom` | `CustomRetriever` | 用户自定义；契约 `list[{url, raw_content}]`，`requires_scraping = False` |
| `mcp` | `MCPRetriever` | 两阶段工具选择与执行 |

- 插件机制：`RETRIEVER_ENTRY_POINT_GROUP = "gpt_researcher.retrievers"`（`actions/retriever.py:13`）。第三方包用该 entry-point 注册名称，`get_retriever()` 在内置名之后查找。

### 4.2 MCP 子系统

| 类 | 文件:行 | 作用 |
|---|---|---|
| `MCPRetriever` | `retrievers/mcp/retriever.py:27` | `search_async(max_results=10)` :116；同步 `search()` 由 `ResearchConductor` 通过 `asyncio.to_thread` 调用 |
| `MCPClientManager` | `mcp/client.py:20` | 连接管理 |
| `MCPToolSelector` | `mcp/tool_selector.py:16` | `select_relevant_tools()`，默认选 3 个 |
| `MCPResearchSkill` | `mcp/research.py:13` | 工具执行阶段 |
| `MCPStreamer` | `mcp/streaming.py:13` | 同步/异步日志流 |

## 5. 上下文与向量存储

| 符号 | 位置 | 接口/要点 |
|---|---|---|
| `select_context` | `context/select.py:53` | `(query, pages, cfg, *, max_results=10, prompt_family, cost_callback, embeddings)` → str |
| `resolve_context_filter` | `context/select.py:38` | `auto/jev/keyword/embeddings/none`；常量 `CONTEXT_FILTERS`, `KEYWORD_RELATIVE_THRESHOLD`, `KEYWORD_MAX_RESULTS`（:29-35） |
| `JevContextCompressor` / `JevClient` / `ScoredChunk` / `JevError` | `context/jev_filter.py:132,68,61,57` | `rank()`/`async_get_context()`；`RELEVANCE_LEVELS`, `JEV_INPUT_PRICE_PER_MTOK`, `_concurrency()`（:32-54） |
| `LexicalContextCompressor` / `bm25_scores` / `tokenize` / `rank_written_sections` | `context/lexical.py:69,48,44,110` | 无依赖 BM25 |
| `VectorstoreCompressor` / `ContextCompressor` / `WrittenContentCompressor` | `context/compression.py:36,88,194` | `async_get_context(query, max_results, cost_callback)` |
| `SearchAPIRetriever` / `SectionRetriever` | `context/retriever.py:16,40` | LangChain `BaseRetriever`；`MAX_CONTENT_CHARS` 默认 50000（:13） |
| `VectorStoreWrapper` | `vector_store/vector_store.py:10` | `load()`, `_create_langchain_documents()`, `_split_documents(chunk_size=1000, overlap=200)`, `asimilarity_search()` |

## 6. 抓取器与文档加载

| 类 | 位置 | 接口 |
|---|---|---|
| `Scraper` | `scraper/scraper.py:116` | `run()`, `extract_data_from_url()`, `_reject()`, `_retry_as_pdf()`, `get_scraper()` |
| `PDFScraper`, `ArxivScraper`, `BeautifulSoupScraper`, `WebBaseLoaderScraper`, `BrowserScraper`, `NoDriverScraper`, `TavilyExtract`, `FireCrawl` | `scraper/*/*.py` | 实现 `scrape()` 或 `scrape_async()`，返回 `(content, image_urls, title)` |
| `WorkerPool` | `utils/workers.py:8` | `throttle()` 上下文管理器；semaphore + 全局 rate limiter |
| `validate_url`, `UnsafeURLError` | `utils/url_security.py` | SSRF 防护 |
| `DocumentLoader`, `LangChainDocumentLoader`, `OnlineDocumentLoader`, `AzureDocumentLoader`, `PDFLoader` | `document/*.py` | `load()` → 文档 dict 列表 |
| `Memory` | `memory/embeddings.py:59` | `get_embeddings()`；`_SUPPORTED_PROVIDERS`（:34-56） |

抓取辅助函数见 `scraper/utils.py`：`get_relevant_images`, `parse_dimension`, `extract_title`, `get_image_hash`, `clean_soup`, `get_text_from_soup`。

## 7. 配置、枚举、LLM

### 7.1 `Config`（`config/config.py:21`）

| 方法/属性 | 位置 | 说明 |
|---|---|---|
| `__init__` | :36-52 | 加载配置 + 解析 embedding/LLM/弃用变量；非 web `REPORT_SOURCE` 时校验 doc_path |
| `_set_attributes` | :54-75 | **环境变量覆盖配置文件**；所有键转小写属性；`self.retrievers` 为类可用名称列表 |
| `load_config` | :149-174 | 无路径 → `deepcopy(DEFAULT_CONFIG)`；有路径 → 与默认值合并；`CONFIG_PATH` 环境变量生效 |
| `parse_retrievers` / `parse_llm` / `parse_embedding` / `parse_reasoning_effort` | :185,200,229,221 | `provider:model` 解析；`RETRIEVER` 逗号分隔 |
| `convert_env_value` | :253-302 | 按 `BaseConfig` 类型提示转换；Optional 的 None 哨兵优先；list/dict 用 `json_repair` |
| `get_mcp_server_config` | :309-326 | 按 name 查找 `MCP_SERVERS` 条目 |

`DEFAULT_CONFIG`（`config/variables/default.py:3-59`）关键默认值（**会随版本变动，以源码为准**）：

```text
RETRIEVER=tavily; EMBEDDING=openai:text-embedding-3-small
CONTEXT_FILTER=auto; SIMILARITY_THRESHOLD=0.42
FAST_LLM=openai:gpt-5.4-mini; SMART_LLM=openai:gpt-6-sol; STRATEGIC_LLM=openai:gpt-5.4
FAST/SMART/STRATEGIC_TOKEN_LIMIT=6000/12000/8000
MAX_SEARCH_RESULTS_PER_QUERY=5; MAX_ITERATIONS=3; TOTAL_WORDS=1200; REPORT_FORMAT=APA
SCRAPER=bs; MAX_SCRAPER_WORKERS=15; SCRAPER_RATE_LIMIT_DELAY=0.0
MAX_SUBTOPICS=3; REPORT_SOURCE=web; DOC_PATH=./my-docs; PROMPT_FAMILY=default
CURATE_SOURCES=False; DEEP_RESEARCH_BREADTH/DEPTH/CONCURRENCY=3/2/4
MCP_STRATEGY=fast; MCP_AUTO_TOOL_SELECTION=True; MCP_SERVERS=[]
REASONING_EFFORT=medium
IMAGE_GENERATION_ENABLED=False; IMAGE_GENERATION_MODEL=models/gemini-2.5-flash-image
IMAGE_GENERATION_MAX_IMAGES=3; IMAGE_GENERATION_STYLE=dark; IMAGE_GENERATION_PROVIDER=google
```

类型定义在 `config/variables/base.py:5-52:BaseConfig(TypedDict)`。弃用变量 `EMBEDDING_PROVIDER`, `LLM_PROVIDER`, `FAST_LLM_MODEL`, `SMART_LLM_MODEL` 仍兼容但会 `FutureWarning`（config.py:90-137）。

### 7.2 枚举（`utils/enum.py`）

- `ReportType`（:6-27）：`research_report`, `resource_report`, `outline_report`, `custom_report`, `detailed_report`, `subtopic_report`, `deep`。
- `ReportSource`（:30-51）：`web`, `local`, `azure`, `langchain_documents`, `langchain_vectorstore`, `static`, `hybrid`。
- `Tone`（:54-91）：Objective（默认）/Formal/Analytical 等 17 个值。
- `PromptFamily` 枚举（:94-101）：default / granite / granite3 / granite3.1 / granite3.2 / granite3.3。

### 7.3 LLM 调用

| 符号 | 位置 | 说明 |
|---|---|---|
| `GenericLLMProvider` | `llm_provider/generic/base.py:165` | `from_provider()` :209；`get_chat_response()` :456；`stream_response()` :473；`last_response_metadata`, `last_usage_metadata` |
| `_SUPPORTED_PROVIDERS` | `gpt_researcher/llm_provider/generic/base.py:19-48` | openai/anthropic/azure_openai/cohere/.../nebius 等 |
| `supports_temperature`, `NO_SUPPORT_TEMPERATURE_MODELS`, `SUPPORT_REASONING_EFFORT_MODELS`, `ReasoningEfforts` | `gpt_researcher/llm_provider/generic/base.py:111,50,118,141` | 模型能力适配 |
| `get_llm` | `utils/llm.py:27` | provider 名 → `GenericLLMProvider.from_provider` |
| `create_chat_completion` | `utils/llm.py:41-164` | 统一调用、重试、成本回调；`LLM_KWARGS` 可覆盖；max_tokens 上限 200000 |
| `construct_subtopics` | `utils/llm.py:167-233` | PromptTemplate + PydanticOutputParser → `Subtopics` |
| `calculate_llm_cost`, `estimate_embedding_cost` | `utils/costs.py` | 成本估算 |

## 8. PromptFamily（`gpt_researcher/prompts.py`）

| 方法 | 行 | 用途 |
|---|---|---|
| `generate_mcp_tool_selection_prompt` / `generate_mcp_research_prompt` | 40 / 86 | MCP 两阶段 |
| `generate_image_analysis_prompt` / `generate_image_prompt_enhancement` | 122 / 178 | 图片生成 |
| `generate_search_queries_prompt` | 213 | 子查询规划 |
| `generate_report_prompt` | 262 | 标准报告（逐句引用约束） |
| `curate_sources` | 319 | 来源策展 |
| `generate_resource_report_prompt` / `generate_custom_report_prompt` / `generate_outline_report_prompt` | 353 / 393 / 399 | 其他报告类型 |
| `generate_deep_research_prompt` | 418 | 深度报告 |
| `auto_agent_instructions` | 490 | agent 角色选择 |
| `generate_summary_prompt` / `generate_quick_summary_prompt` | 518 / 532 | 摘要 |
| `pretty_print_docs` / `join_local_web_documents` | 555 / 564 | 上下文格式化（Granite 家族覆写） |
| `generate_subtopics_prompt` / `generate_subtopic_report_prompt` / `generate_draft_titles_prompt` | 573 / 596 / 673 | 详细报告 |
| `generate_report_introduction` / `generate_report_conclusion` | 707 / 720 | 首尾章节 |
| `get_prompt_by_report_type` / `get_prompt_family` | 862 / 889 | 路由函数 |

## 9. API 与入口

| 符号 | 位置 | 说明 |
|---|---|---|
| `app` | `backend/server/app.py:97` | FastAPI 应用；lifespan 挂载 outputs/frontend（:71-95） |
| `ResearchRequest` | `backend/server/app.py:53-61` | `task, report_type, report_source, tone, headers?, repo_name, branch_name, generate_in_background=True` |
| `ChatRequest` | `backend/server/app.py:64-68` | `report, messages` |
| `POST /report/` | `app.py:378-388` | 生成报告（可后台） |
| `GET /report/{research_id}`, `/api/reports*` | `app.py:174-255` | 报告存取 |
| `POST /api/reports/{id}/chat`, `/api/chat` | `app.py:257-330,428-...` | 基于报告的聊天 |
| `POST /api/multi_agents` | `app.py:400-402` | 多智能体 |
| `GET /files/`, `POST /upload/`, `DELETE /files/{filename}` | `app.py:391-412` | 本地文档 |
| `WS /ws` | `app.py:415-426` | 进度流 |
| `run_agent` | `backend/server/websocket_manager.py:116-183` | report_type 分支：multi_agents / DetailedReport / BasicReport |
| `BasicReport` | `backend/report_type/basic_report/basic_report.py:9` | `run()` :72 = conduct + write |
| `DetailedReport` | `backend/report_type/detailed_report/detailed_report.py:10` | `run()` :83 等 |
| `write_md_to_pdf` / `write_md_to_word` | `backend/utils.py:67,112` | 导出 |
| `main.py` | :10-63 | load_dotenv → 日志配置 → `uvicorn.run(app, host=HOST, port=PORT)` |

CLI（`cli.py`）：`cli` ArgumentParser（:31-135），`main(args)`（:270-364）；参数 `query`, `--report_type`（必需）, `--tone`, `--encoding`, `--query_domains`, `--report_source`, `--no-pdf`, `--no-docx`。辅助：`_sanitize_filename`, `_build_frontmatter`, `_resolve_unique_path`（:146,210,249）。

## 10. 多智能体符号

| 类/函数 | 位置 | 说明 |
|---|---|---|
| `ChiefEditorAgent`（LangGraph） | `multi_agents/agents/orchestrator.py:30` | `_initialize_agents` :54, `_create_workflow` :65, `_add_workflow_edges` :83, `run_research_task` :149 |
| `ResearchState` | `multi_agents/memory/research.py:5` | 图状态 TypedDict |
| `ResearchAgent`, `WriterAgent`, `EditorAgent`, `PublisherAgent`, `HumanAgent`, `FactCheckerAgent`, `VisualizerAgent`, `ReviserAgent`, `ReviewerAgent` | `multi_agents/agents/*.py` | 由 `multi_agents/agents/__init__.py` 导出 |
| `route_human_feedback`, `MaxPlanRevisionsExceededError` | `multi_agents/agents/plan_review.py:8,4` | 计划修订有界循环 |
| `route_fact_check`, `MaxFactCheckRevisionsExceededError` | `multi_agents/agents/fact_review.py:8,4` | 事实核查有界循环 |
| `open_task`, `run_research_task`, `main` | `multi_agents/main.py:46,69,81` | 读 `multi_agents/task.json` 并运行 |
| `ChiefEditorAgent`（AG2） | `multi_agents/ag2/agents/orchestrator.py:20` | 独立实现 |
| `build_agent`, `CHIEF_EDITOR_PROMPT`, `RESEARCHER_PROMPT` | `deep_agents/agent.py:63,11,47` | deepagents 版本 |
| `build_research_tools` | `deep_agents/tools.py:9` | 返回 `quick_search`, `deep_research` 工具 |

## 11. 观测与测试

- `ResearchObserver`（Protocol）和 `emit_research_event()`：`gpt_researcher/observability.py:16,22`。observer 异常不得中断研究。
- `JSONResearchHandler`：`utils/logging_config.py:7`；`CustomLogsHandler`：`backend/server/server_utils.py:41`。
- 测试位于 `tests/`（如 `test_context_manager_error_isolation.py`, `test_arxiv_scraper.py`, `test_config_default_isolation.py`, `test_custom_retriever_guard.py`, `test_detailed_report_*` 等），修改核心契约时应同步补充/运行相应测试。

## 12. 维护不变量速查

1. 检索器返回契约（URL vs 全文）必须显式声明 `requires_scraping`。
2. scraper 返回三元组 `(content, image_urls, title)`；失败用 `raw_content=None`。
3. `ContextManager.get_similar_content_by_query()` 的返回必须是可拼接的字符串；调用方常做 `" ".join(context)`。
4. `ReportGenerator.write_report()` 的空 context 弃答行为不能删除，它是唯一的“无来源不编造”硬闸。
5. MCP 配置只走 per-session 参数（`mcp_configs`/cfg），不要写 `os.environ`。
6. 新增 report type 要同时更新 `ReportType`, `report_type_mapping`, PromptFamily 与所有入口分支（`cli.py`, `run_agent()`）。
7. 修改 `DEFAULT_CONFIG` 时检查环境变量类型转换（`BaseConfig` 注解）与配置隔离测试。
