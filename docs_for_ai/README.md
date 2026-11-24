# GPT Researcher — AI 维护者入口（docs_for_ai）

> 本文面向零上下文 AI。事实来源为本仓库源码；`路径:行号` 是本检出版本的定位锚点。
>
> **目录边界**：`docs/` 是 Docusaurus 静态网站源码，**不要修改**。本套文档只位于 `docs_for_ai/`。

## 1. 项目定位

GPT Researcher 是一个通用的**自主深度研究引擎**：对任意任务同时做 web 与本地文档研究，产出带引用的长篇报告。README 将它们描述为 planner + execution agents + publisher 的协作，并支持多智能体、递归 deep research、上下文过滤、MCP 数据源与图像生成（`README.md:25-27,57-70`）。

任务描述称其为“3 万星”项目；star 数是动态外部指标，本文不把它当作源码事实，请以 GitHub 仓库页面为准。

可确认的仓库事实：

- 入口包只导出一个类：`gpt_researcher/__init__.py` → `GPTResearcher`。
- 核心研究编排在 `gpt_researcher/agent.py:42` 的 `GPTResearcher`。
- API 服务是 FastAPI：`backend/server/app.py:97` 的 `app`，由 `main.py:52-63` 通过 uvicorn 启动。
- CLI 在 `cli.py`；多智能体在 `multi_agents/`（LangGraph）与 `multi_agents/ag2/`（AG2）；另有基于 `deepagents` 的 `deep_agents/`。
- 许可：Apache 2（README 免责声明 `README.md:357-359`）。

## 2. 五种使用形态

| 形态 | 入口 | 典型命令/调用 |
|---|---|---|
| Python 包 | `GPTResearcher` | `await researcher.conduct_research(); report = await researcher.write_report()`（`README.md:148-159`） |
| FastAPI 服务 + 前端 | `main.py`, `backend/server/app.py` | `python -m uvicorn main:app --reload`，访问 `localhost:8000`（`README.md:132-139`） |
| CLI | `cli.py` | `python cli.py "<query>" --report_type research_report --report_source web` |
| 多智能体（LangGraph / AG2） | `multi_agents/main.py`, `multi_agents/ag2/main.py` | `asyncio.run(main())` 或 `POST /api/multi_agents` |
| MCP 数据源 / 外部助手 | `RETRIEVER=tavily,mcp` + `mcp_configs` | 见 `execution_guide.md` §5 |

README 也提供 Claude Skill 安装：`npx skills add assafelovic/gpt-researcher`（`README.md:47-55`）。仓库内 `.claude/SKILL.md` 是面向“理解/扩展/调试/集成”的开发 skill 描述。

## 3. 一句话架构

```text
query
  → GPTResearcher 选择 agent/role
  → ResearchConductor 规划 sub-queries
  → 多 retriever 并行搜 URL/全文
  → Scraper 并发抓取（去重、质量过滤、SSRF 防护）
  → ContextManager + select_context（Jev / BM25 / embeddings / none）重排长文本
  → ReportGenerator / PromptFamily 写 Markdown（强制引用提示）
  → 可选 add_references + PDF/DOCX 导出
```

详细展开见 [`architecture.md`](architecture.md) 与 [`pipeline_and_dataflow.md`](pipeline_and_dataflow.md)。

## 4. 代码地图（先看这些文件）

| 关注点 | 文件 | 关键符号 |
|---|---|---|
| 主智能体 | `gpt_researcher/agent.py` | `GPTResearcher`, `conduct_research()`, `write_report()`, `quick_search()` |
| 研究编排 | `gpt_researcher/skills/researcher.py` | `ResearchConductor`, `conduct_research()`, `_process_sub_query()` |
| 上下文选择 | `gpt_researcher/context/select.py`, `skills/context_manager.py` | `select_context()`, `resolve_context_filter()`, `ContextManager` |
| 长文本重排 | `gpt_researcher/context/{jev_filter,lexical,compression}.py` | `JevContextCompressor`, `LexicalContextCompressor`, `ContextCompressor` |
| 报告生成 | `gpt_researcher/skills/writer.py`, `actions/report_generation.py` | `ReportGenerator`, `generate_report()` |
| Prompt | `gpt_researcher/prompts.py` | `PromptFamily`, `get_prompt_by_report_type()`, `get_prompt_family()` |
| 配置 | `gpt_researcher/config/config.py`, `config/variables/default.py` | `Config`, `DEFAULT_CONFIG` |
| 检索器 | `gpt_researcher/actions/retriever.py`, `retrievers/` | `get_retriever()`, `get_retrievers()`, `BaseRetriever` |
| 抓取器 | `gpt_researcher/scraper/scraper.py`, `actions/web_scraping.py` | `Scraper`, `scrape_urls()` |
| API | `backend/server/app.py`, `backend/server/websocket_manager.py` | `app`, `ResearchRequest`, `run_agent()` |
| 详细报告 | `backend/report_type/detailed_report/detailed_report.py` | `DetailedReport` |
| 深度研究 | `gpt_researcher/skills/deep_research.py` | `DeepResearchSkill`, `deep_research()`, `run()` |
| 多智能体 | `multi_agents/agents/orchestrator.py` | `ChiefEditorAgent`, LangGraph `StateGraph` |
| LLM 抽象 | `gpt_researcher/llm_provider/generic/base.py`, `utils/llm.py` | `GenericLLMProvider`, `create_chat_completion()` |
| 观测 | `gpt_researcher/observability.py`, `utils/costs.py` | `ResearchObserver`, `emit_research_event()`, `calculate_llm_cost()` |

完整接口清单见 [`modules_reference.md`](modules_reference.md)。

## 5. 设计要点（避免常见误解）

1. **不是纯 RAG**：规划器先生成 sub-queries，执行器逐条搜索、抓取、过滤，再由 publisher 汇总（`README.md:27,57-70`）。
2. **引用不是脚本保证的事实校验**：`PromptFamily.generate_report_prompt()` 要求每个实质 claim 带 URL 引用（`prompts.py:301-315`），`ReportGenerator.write_report()` 在无上下文时直接弃答（`skills/writer.py:79-88`）；但没有逐句事实验证器，最终准确性仍需人工/下游审查。
3. **上下文过滤默认自适应**：`CONTEXT_FILTER=auto` 时，有 `TYPESAFE_API_KEY` 用 Jev，否则用本地 BM25；非 embeddings 路径不会构建 embedding 模型（`context/select.py:38-46,53-101`）。
4. **retriever 有两种结果契约**：`BaseRetriever.requires_scraping` 声明结果是“待抓 URL”还是“已抓全文”（`retrievers/base.py:1-29,35-61`）。扩展检索器时这是最重要的接口。
5. **MCP 配置是 per-session 的**：`GPTResearcher._process_mcp_configs()` 修改 `self.cfg.retrievers`，刻意不动 `os.environ`，避免并发请求互相污染（`gpt_researcher/agent.py:312-338`）。
6. **多入口的行为不完全一致**：CLI 的 `detailed_report` 走 `DetailedReport`；API `run_agent()` 用 `BasicReport`/`DetailedReport`/multi_agents 三种分支（`backend/server/websocket_manager.py:133-181`）。改报告流程时要同时检查这些入口。

## 6. 配套文档

- [`docs_for_ai/architecture.md`](architecture.md)：抓取器、检索器、主智能体与多智能体架构。
- [`docs_for_ai/pipeline_and_dataflow.md`](pipeline_and_dataflow.md)：从查询规划到报告生成的阶段与数据形状。
- [`docs_for_ai/modules_reference.md`](modules_reference.md)：核心类与关键接口清单。
- [`docs_for_ai/execution_guide.md`](execution_guide.md)：API 配置、Docker、多检索器集成与参数调优。

### 6.1 源码级深度参考（新增）

| 文档 | 回答什么问题 | 何时加载 |
|---|---|---|
| [`docs_for_ai/retrievers_and_scrapers_deep_dive.md`](retrievers_and_scrapers_deep_dive.md) | 检索工厂与 `requires_scraping` 契约、Tavily/DDG/Arxiv/S2 的实现差异、多检索器 fan-out、Scraper 调度与正文清洗/质量过滤算法 | 扩展检索器或抓取器、排查来源缺失时 |
| [`docs_for_ai/context_compression_and_prompts.md`](context_compression_and_prompts.md) | 切块、BM25/Jev/embedding 过滤、上下文大小旋钮；`prompts.py` 全部模板占位符与关键模板全文 | 调上下文质量、改 Prompt、排查截断时 |
| [`docs_for_ai/troubleshooting.md`](troubleshooting.md) | 反爬拦截、检索超时/重试、长报告上下文溢出、Docker 部署的报错→根因→修复 | 任一阶段结果异常时 |

三篇是**对既有四篇的补充**：先读 §6 建立全貌，`architecture.md` 讲组件关系，深度文档讲逐行实现；所有 `路径:行号` 锚点指向本检出版本。

## 7. 深度参考的维护约定

- 修改 `retrievers/`、`scraper/`、`context/`、`prompts.py` 后，先用 `grep -n` 核对新行号，再同步更新三篇深度文档的锚点。
- 新增检索器必须显式声明 `requires_scraping`；新增抓取器必须登记 `Scraper.get_scraper()` 的映射并遵守 `(content, image_urls, title)` 返回契约（见深度文档 §7）。
- 新增 `CONTEXT_FILTER` 模式或 Prompt 方法时，同步更新 `select.py` 常量、`.env.example`、`docs_for_ai/context_compression_and_prompts.md` 与对应测试。
