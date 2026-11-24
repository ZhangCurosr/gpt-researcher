# GPT Researcher 排错指南：反爬、检索超时、上下文溢出与 Docker

> 事实来源：本仓库源码；`路径:行号` 为本检出版本锚点。
>
> 这是 [`execution_guide.md` §7](execution_guide.md) 的源码级扩展；不修改 Docusaurus 站点文档 `docs/`。
>
> 通用原则：GPT Researcher 的绝大多数失败**不会抛到顶层**——检索器、抓取器、上下文过滤器都倾向于“降级并返回空”。因此排错第一步永远是**看日志/事件**，而不是看报告里缺了什么。

## 0. 先打开可观测性

| 手段 | 做法 | 依据 |
|---|---|---|
| `verbose=True` | Python API 或配置打开；`stream_output` 会把每个阶段推送到 websocket/stdout | `skills/researcher.py:138-150,404-412` |
| 日志级别 | `LOGGING_LEVEL=INFO`（docker-compose 已设置） | `docker-compose.yml:17` |
| 关键日志行 | `Active retrievers: [...]`、`Initial search results obtained: N results`、`Generated sub-queries: [...]`、`Scraped data size: N`、`Combined context size: N chars`、`Research completed. Context size: N` | `skills/researcher.py:129-130,83,398,639,434,273` |
| 结构化事件 | `ResearchObserver` 收到 `search_failed`、`search_empty`、`candidate_discovered`、`content_prefetched`、`selected_for_fetch`、`candidate_deduplicated`、`fetch_succeeded`、`fetch_failed` | `skills/researcher.py:51-57` 及 :918-1021；`scraper/scraper.py:166-184` |
| JSON handler | `get_json_handler()` 写入 query/costs/context 与 `sub_query`/`content_found` 事件 | `skills/researcher.py:43,123-124,553-557,675-681` |
| 独立检索器冒烟测试 | `docs/docs/gpt-researcher/search-engines/test-your-retriever.md`（站点文档）或按 §2.1 直接实例化 `.search()` | — |

**最重要的判定式**：`Scraper.run()` 只保留 `raw_content is not None` 的结果（`scraper/scraper.py:186-193`）。所以“抓到了但报告里没引用”通常意味着该行在质量闸门被 `_reject()` 标记成失败；去看 `fetch_failed` 事件的 `failure_reason`。

---

## 1. 网页反爬拦截应对

### 1.1 拦截页如何被识别

`scraper/scraper.py:41-56` 维护 `_BLOCK_PAGE_MARKERS`（大小写不敏感子串匹配），包括：

- `anubis uses a proof-of-work scheme`（HAL 等 Anubis 前置站点）；
- `making sure you're not a bot`；
- `checking your browser before accessing`；
- `enable javascript and cookies to continue`；
- `researchgate - temporarily unavailable`（特意用站点特定文案而非泛化的 “temporarily unavailable”）；
- `please verify you are a human`；
- `attention required! | cloudflare`；
- `sorry, you have been blocked`；
- `a required part of this site couldn't load` 的两种撇号写法（wikiHow 等 Client Challenge 页）。

`_looks_like_block_page()`（:80-82）**只检查前 5000 字符**（`_BLOCK_PAGE_CHECK_PREFIX_LEN`，:62），因为拦截页短且出现在开头；这样避免对每个多 MB 正文做全量小写化（:58-61）。命中后调用 `_reject(link, title, "Anti-bot/challenge page detected")`（:281-282），最终在 `Scraper.run()` 被丢弃。

### 1.2 HTTP 层的第一道重试

`BeautifulSoupScraper._fetch()`（`scraper/beautiful_soup/beautiful_soup.py:65-127`）已有内建策略：

| 情况 | 行为 | 行号 |
|---|---|---|
| 网络异常 / 429 / 500 / 502 / 503 / 504 | sleep 1s，重试一次 | :11,77-97 |
| **403** | 改用透明身份 `GPT-Researcher/1.0 (+https://github.com/assafelovic/gpt-researcher; source retrieval)` 重试一次 | :13-16,99-105 |
| 其他 ≥400 | 不解析错误页，记 `failure_reason=HTTP_<code>` | :107-111 |
| `Content-Length > 10MB` | 跳过 | :12,113-123 |

因此看到 `fetch_failed` 且 reason 是 `HTTP_403`，说明**两次尝试都被挡**；接下来要换策略而不是继续重试同一后端。

### 1.3 升级策略（按顺序尝试）

1. **降速**：`MAX_SCRAPER_WORKERS`（默认 15，`config/variables/default.py:29`）调小；`SCRAPER_RATE_LIMIT_DELAY` 设为 `60 / 每分钟允许请求数`（`.env.example:42-52`）。该限速是**全局单例**（`utils/rate_limiter.py:13-49`），对所有 WorkerPool 生效；`WorkerPool.throttle()`（`utils/workers.py:35-49`）先拿本池 semaphore，再等全局 limiter。Firecrawl 免费档另有模块级 `FIRECRAWL_CONCURRENCY=2`（`scraper/firecrawl/firecrawl.py:8-19`）。
2. **换 UA**：`USER_AGENT` 配置项（`default.py:21`）；403 时 BS 后端已自动换透明身份，若仍失败说明站点要求 JS/cookie。
3. **换后端**：`SCRAPER=browser` 用 Selenium（`scraper/browser/browser.py:37-74`），会先访问 Google 存 cookie、等待 body 出现 20s、滚动到底；`SCRAPER=nodriver` 用 zendriver 浏览器池（每域一个信号量，随机滚动，`scraper/browser/nodriver_scraper.py:16-21,59-128,190-260`）。**注意 Selenium 默认 `headless=False`（:42）**，容器里没有显示服务时需要自行改代码或用 nodriver/BS。
4. **换数据源**： tavily 作为检索器 + `SCRAPER=tavily_extract` 让 Tavily 侧抽取正文（`scraper/tavily_extract/tavily_extract.py:26-72`），或 FireCrawl（`scraper/firecrawl/firecrawl.py`）。
5. **确认不是别的失败被误认为反爬**：见下表。

### 1.4 相似症状对照

| 现象 / `failure_reason` | 真正原因 | 处理 |
|---|---|---|
| `Anti-bot/challenge page detected` | 命中拦截页 marker | 按 §1.3 升级；marker 不覆盖的新站点可在 `scraper.py:41-56` 增加**足够具体**的文案（注释 :36-40 强调避免误伤正文中普通短语） |
| `Content too short or empty` | 正文 <100 字符（`scraper.py:268-272`） | JS 渲染页、付费墙、纯图片页；换浏览器后端 |
| `Word-list-like content detected` | ≥200k 字符且句末标点密度 <1/5000（`scraper.py:75-89`） | 词表/词汇 dumps 被有意过滤；换更精确的 URL |
| `Skipping unsafe URL ...` | SSRF 防护拒绝（`scraper.py:238-247`） | 公网地址不应触发；内网/本地目标需显式 `ALLOW_PRIVATE_URLS=true`（`utils/url_security.py:44-45,95-117`），仅用于可信环境 |
| `looks like unextracted PDF binary ... retrying with PDFScraper` | URL 无 `.pdf` 后缀但内容是 PDF（`scraper.py:287-296,333-364`） | 正常自动恢复；若仍失败会作为 fetch_failed 丢弃 |
| 报告里 URL 数少于预期但没有错误日志 | `Scraper.run()` 过滤了 None 行；或 `visited_urls` 去重（`skills/researcher.py:852-873`） | 看 `fetch_failed` / `candidate_deduplicated` 事件 |
| 抓到的是 cookie 同意页/导航页 | marker 未覆盖 | 换浏览器渲染；不要把这类内容当证据 |

---

## 2. 检索器超时与重试

### 2.1 先明确事实：检索器本身大多没有重试循环

| 检索器 | 超时 | 失败行为 |
|---|---|---|
| Tavily | `requests.post(..., timeout=100)`（`retrievers/tavily/tavily_search.py:97-99`） | `except Exception` → 打印并 `[]`（:167-170） |
| Semantic Scholar | `timeout=10`（`retrievers/semantic_scholar/semantic_scholar.py:43-49`） | 请求/JSON 异常 → `[]` |
| Custom | `timeout=20`（`retrievers/custom/custom.py:51-61`） | 请求/JSON 异常 → `[]` |
| DuckDuckGo | 未显式设置 | `ddg.text()` 异常 → `[]`（`duckduckgo.py:61-65`） |
| Arxiv | 客户端默认 | 异常 → `[]`（`retrievers/arxiv/arxiv.py:22-32`） |

因此“检索器超时重试”要区分三层：

1. **HTTP 客户端层**：上述超时值；要改只能改对应 retriever 源码或通过 CDN/代理层处理。
2. **LLM 规划层**：`generate_sub_queries()` 在 strategic LLM 失败时先用 `max_tokens=cfg.strategic_token_limit` 重试，再回退 smart LLM（`actions/query_processing.py:117-154`）；这是**唯一明确的两级重试**。
3. **抓取层**：BS 的一次重试与 403 换身份（§1.2）、Jev 的 429/529 指数退避（`context/jev_filter.py:104-125`）、`record`-类外部工具无重试。

### 2.2 配置与工厂层面的“假失败”

| 症状 | 根因 | 修复 |
|---|---|---|
| 写错 retriever 名字但照常跑出 Tavily 结果 | `get_retrievers()` 用 `get_retriever(r) or get_default_retriever()`（`actions/retriever.py:192`），未知名静默回退；headers 路径绕过 `Config.parse_retrievers()` 校验 | 看日志 `Active retrievers: [...]`（`skills/researcher.py:129-130`）；用 `cfg.retrievers` 构造以触发 `config/config.py:185-198` 的白名单校验 |
| `"tavily, exa"` 带空格导致只用一个 | 旧版本未 strip；现已在 :182-185 修复 | 仍建议显式写无空格列表 |
| 插件 retriever 不生效 | `_load_plugin_retriever()` 只 warning，失败返回 None（`actions/retriever.py:140-148`） | 查 entry point 名是否在 `gpt_researcher.retrievers` 组；查 warning 日志 |
| `Tavily API key not found, set to blank...` | headers 无 `tavily_api_key` 且环境无 `TAVILY_API_KEY`（`tavily_search.py:47-62`） | 设置环境变量或 per-request header |
| Tavily 零结果 | `site:` 运算符未翻译（:117-139，代码已处理）、query >400 字符（:141-148）、返回体是 list/scalar（:149-155） | 看 `search_empty` 事件；简化 query |
| DDG/Searx 结果被当作已抓全文 | snippet >100 字符触发旧启发式；两者已在源头截断（`duckduckgo.py:19,88`、`searx.py:22`） | 自定义 retriever 请显式声明 `requires_scraping`（见 [`retrievers_and_scrapers_deep_dive.md` §2.3](retrievers_and_scrapers_deep_dive.md)） |
| Semantic Scholar 结果很少 | 只保留 `isOpenAccess` 且有 `openAccessPdf.url` 的结果（`semantic_scholar.py:61-66`） | 换其他学术检索器；这是设计过滤而非 bug |
| 403/429 来自 provider | 无全局重试；rate-limit 状态映射在 `verify_citations` 之外不适用 | 降频、换 key、稍后重试 |

### 2.3 事件驱动的定位方法

`_search_relevant_source_urls()` 在关键节点发事件（`skills/researcher.py`）：

- `search_empty`（:918-924）：provider 返回空；
- `candidate_discovered`（:941-947）：某条结果进入候选，带 `rank`；
- `content_prefetched`（:966-973,985-992）：结果被当作已抓全文；
- `selected_for_fetch` / `candidate_deduplicated(reason_code=ALREADY_VISITED)`（:1007-1021）；
- `search_failed`（:997-1002，以及初始搜索 :75-82）带 `reason_code=异常类名`。

初始搜索失败会**直接抛出**并中断该轮研究（:75-82）；sub-query 级失败被 `try/except` 吞掉并返回空字符串（:685-694），所以报告“少了一章”往往要在 sub-query 日志里找 `Error processing sub-query ...`。

---

## 3. 大长篇报告：上下文溢出与截断

### 3.1 判断是“输入上下文溢出”还是“输出被截断”

| 症状 | 更可能是 | 证据 |
|---|---|---|
| 报告在半句/固定字数处停止 | 输出 token 上限 | `SMART_TOKEN_LIMIT`（默认 12000，`config/variables/default.py:15`）；日志中 LLM 返回的 `finish_reason` |
| 报告后期章节明显“忘了”前面内容 | 输入上下文被选择/截断 | `MAX_CONTENT_CHARS`（`context/retriever.py:13`）、`COMPRESSION_THRESHOLD`（`select.py:76`）、`KEYWORD_MAX_RESULTS=25`（:35） |
| 某些来源完全没进 Prompt | 小内容短路或 BM25 低分 | `select.py:74-77`、`lexical.py:99-104` |
| 报告只有“无法生成可靠报告” | 空上下文弃答 | `skills/writer.py:79-88` |
| 引用列表很长但正文没引用 | `add_references()` 追加全部 visited_urls | `actions/markdown_processing.py:121-127` |

### 3.2 逐级检查上下文大小

按管线顺序打印/观察：

```text
1) 抓取行数：日志 "Scraped data size: N"        researcher.py:639
2) 单 sub-query 上下文："Web content found for sub-query: N chars"  researcher.py:644
3) 合并上下文："Combined context for '<sub>': N chars"        researcher.py:650-652
4) 全部 sub-query 合并："Combined context size: N" / "Gathered context from N sub-queries"  researcher.py:429,434
5) 研究结束："Research completed. Context size: N"        researcher.py:273
```

若第 2 步就很小，问题在抓取/过滤（§1–§2）；若第 2 步正常而第 5 步异常大，问题在多 sub-query 合并（每个 sub-query 的上下文用 `" ".join(context)` 直接拼接，`skills/researcher.py:431-435`）。

### 3.3 可用旋钮与代价

| 旋钮 | 位置 | 调大/调小的影响 |
|---|---|---|
| `MAX_CONTENT_CHARS`（默认 50000） | `context/retriever.py:13` | 调大保留更多单页全文，但可能触碰 embedding API 单请求 token 上限（注释 :9-12） |
| `COMPRESSION_THRESHOLD`（默认 8000） | `select.py:76`, `compression.py:167`, `.env.example:54-60` | 调大 = 更多页面走“直接输出”快路，可能把低质全文塞进 Prompt；调小 = 更多切块过滤成本 |
| `KEYWORD_RELATIVE_THRESHOLD` / `KEYWORD_MAX_RESULTS` | `select.py:34-35` | 阈值越低保留越多中等相关块；条数越大上下文越长 |
| `SIMILARITY_THRESHOLD`（配置默认 0.42） | `default.py:6`, `select.py:93`, `compression.py:125-127` | 调高更严格、上下文更短；注意应用路径传 cfg 值会覆盖代码里的 0.35 回退 |
| `JEV_CHUNK_SIZE` / `JEV_MIN_SCORE` / `JEV_CONCURRENCY` | `jev_filter.py:150,153,53` | 更小的 chunk 更精细但更贵；min_score 越高上下文越短 |
| `SMART_TOKEN_LIMIT` / `STRATEGIC_TOKEN_LIMIT` / `FAST_TOKEN_LIMIT` | `default.py:14-16`, `.env.example:76-92` | 输出与规划上限；reasoning 模型的思考 token 也算在 `max_completion_tokens` 里（default.py:11-13） |
| `MAX_SEARCH_RESULTS_PER_QUERY`（默认 5） | `default.py:22` | 减少 URL 数量是最直接的降上下文手段 |
| `DEEP_RESEARCH_BREADTH/DEPTH/CONCURRENCY` | `default.py:40-42` | 递归报告规模；depth 每加一层，分支数近似指数增长 |
| `CONTEXT_FILTER=none` | `select.py:71-72` | 关闭过滤——**最容易溢出**，仅用于小数据集调试 |

### 3.4 长报告的建议流程

1. 用 `report_type=detailed_report` 而不是把所有材料塞进一次 research report：详细报告把任务拆成 subtopics，每个 subtopic 单独抓取/压缩/写作（`skills/writer.py:122-128`，`generate_subtopic_report_prompt` 的去重约束见 [`context_compression_and_prompts.md` §3.4](context_compression_and_prompts.md)）。
2. 让 `get_similar_written_contents_by_draft_section_titles()` 提供已写章节上下文（最多 10 条，`context_manager.py:89-130`），避免重复内容同时占用输出预算。
3. 若使用 deep research，从 `breadth=2, depth=1` 起步验证，再逐步加大（`default.py:40-42`）。
4. 输出被截断时先提高 `SMART_TOKEN_LIMIT`，而不是删引用；`.env.example:83-92` 给出按模型族的推荐值。
5. 每次改配置后跑一个小样本查询，核对 §3.2 的五级日志数字，避免全量任务代价过高。

---

## 4. Docker 部署踩坑

### 4.1 仓库里有两份 Dockerfile

| 文件 | 基础镜像 | 入口 | 用途判断 |
|---|---|---|---|
| `Dockerfile:1-64` | `python:3.14-slim-bookworm`（注释 :2 说明 Python 3.12+ 是 LangChain v1 要求） | `uvicorn main:app --host ${HOST} --port ${PORT} --workers ${WORKERS}`（:64） | 根目录主镜像，安装 Chromium/Chromedriver/Firefox/Geckodriver（:5-21）；`docker-compose.yml:5` 的 `build: ./` 用的就是它 |
| `backend/Dockerfile:1-17` | `python:3.12-slim` | `uvicorn app.main:app ...`（:17） | 仓库中的另一份镜像定义，入口路径与根目录不同；`docker-compose.yml` 并未使用它 |

若按 README/compose 流程部署，以根目录 `Dockerfile` 为准；不要混用两份配置。

### 4.2 非 root 用户与动态安装冲突

根目录镜像第 3 阶段创建 `gpt-researcher` 用户并 `USER gpt-researcher`（`Dockerfile:51-59`），但注释明确写着：**如果依赖 `Scraper._check_pkg()` 在运行时 `pip install`，不要启用这个非 root 用户**（:52）。`_check_pkg()` 确实会在缺包时执行 `pip install`（`scraper/scraper.py:196-228`），所以用 `SCRAPER=tavily_extract/firecrawl` 且未预装 SDK 时，非 root 容器会安装失败。解决：把依赖写进 `requirements.txt` 预装，或用自己的镜像/distroless 策略。

`docker-compose.yml:27` 又设置了 `user: root`，实际上覆盖了镜像的 `USER`，使得动态安装在 compose 路径下可用；直接 `docker run` 镜像时则不会。

### 4.3 环境变量与卷

- `docker-compose.yml:9-22` 显式透传 `OPENAI_API_KEY`, `OPENAI_BASE_URL`, `TAVILY_API_KEY`, `LANGCHAIN_API_KEY`, `TYPESAFE_API_KEY`, `TYPESAFE_BASE_URL`, `LOGGING_LEVEL`, `GOOGLE_API_KEY`, `IMAGE_GENERATION_*`；同时 `env_file: .env`（可选，:6-8）会把 `.env` 中的其他变量带入容器。若你用 env_file 之外的方式注入 `CONTEXT_FILTER`、`SCRAPER_RATE_LIMIT_DELAY`、`ALLOW_PRIVATE_URLS`、`FIRECRAWL_CONCURRENCY`，请确认它们确实出现在容器环境里。
- 卷挂载：`my-docs`、`outputs`、`logs` 三个目录（:23-26）；镜像内 `outputs` 被 `chmod 777`（`Dockerfile:55-58`），但宿主机目录权限不匹配仍可能导致写报告失败。
- 前端：`gptr-nextjs` 服务端用 `GPTR_API_URL`（默认 `http://gpt-researcher:8000`，:42），浏览器侧用 `NEXT_PUBLIC_GPTR_API_URL`（:39）；本机直连用 `localhost:8000`，容器内**不能**用 `localhost` 指向后端容器（`.env.example:27-28`）。
- 端口：后端 8000、前端 3000（:29-30,52-53）。
- `TYPESAFE_BASE_URL` 传到 `/v1` 即可，代码自动追加 `/systemone`（`.env.example:12-14`、`context/jev_filter.py:78-82`）。
- 测试服务在 `profiles: ["test"]`（:55-73），默认不会启动。

### 4.4 浏览器相关

根目录镜像已安装浏览器与 driver（`Dockerfile:5-21`），但 `BrowserScraper` 默认 `headless=False`（`scraper/browser/browser.py:42`）；容器无 X server 时优先 `SCRAPER=bs` 或 `SCRAPER=nodriver`，或自行在 `setup_driver()`（:98-134）中打开 headless。Linux 下代码已加 `--no-sandbox` 等参数（:119-124）。

### 4.5 快速检查清单

```bash
docker compose config                 # 检查 env/volume/profile 是否被正确解析
docker compose up -d --build
docker compose logs -f gpt-researcher # 观察 Active retrievers / Scraped data size / Context size
docker compose exec gpt-researcher env | grep -E 'RETRIEVER|SCRAPER|TOKEN_LIMIT|ALLOW_PRIVATE'
```

---

## 5. 一页对照表

| 问题 | 首选检查点 | 首选修复 |
|---|---|---|
| 报告没有来源 | `fetch_failed` 事件 / `failure_reason` | §1 降速换后端 |
| 只有 Tavily 生效 | 日志 `Active retrievers` | `cfg.retrievers` 校验优先于 headers |
| 检索器零结果 | `search_empty` 事件、API key、query 长度 | §2.2 |
| 报告半句截断 | `SMART_TOKEN_LIMIT` | `.env.example:83-92` |
| 报告“忘记”前文 | 五级上下文日志、切块阈值 | §3.3 |
| 报告只有拒答文本 | `skills/writer.py:79-88` | 先修检索/抓取，再写报告 |
| 容器内安装 SDK 失败 | 非 root + `_check_pkg()` | 预装依赖或用 compose 的 root 覆盖 |
| 前端连不上后端 | `GPTR_API_URL` / `NEXT_PUBLIC_GPTR_API_URL` | 容器内用服务名，不用 localhost |

## 6. 锚点速查

| 主题 | 文件:行号 |
|---|---|
| 拦截页与质量过滤 | `gpt_researcher/scraper/scraper.py:32-113,230-364` |
| BS 重试与 403 | `gpt_researcher/scraper/beautiful_soup/beautiful_soup.py:10-16,65-127` |
| 浏览器后端 | `gpt_researcher/scraper/browser/browser.py:37-74,98-134,206-250` |
| 全局限速 | `gpt_researcher/utils/workers.py:8-50`, `utils/rate_limiter.py:13-90` |
| 检索器工厂 | `gpt_researcher/actions/retriever.py:16-205` |
| 检索分支与事件 | `gpt_researcher/skills/researcher.py:890-1024` |
| 上下文选择/降级 | `gpt_researcher/context/select.py:29-101` |
| 空上下文弃答 | `gpt_researcher/skills/writer.py:79-88` |
| 默认配置 | `gpt_researcher/config/variables/default.py:3-59` |
| Docker | `Dockerfile:1-64`, `docker-compose.yml:1-88`, `.env.example:1-106` |
