# 检索器与抓取器深度剖析

> 事实来源：本仓库源码；`路径:行号` 为本检出版本锚点。
>
> 与 [`architecture.md`](architecture.md) 的分工：那篇给出组件图，这篇逐层拆解**工厂解析 → 多检索器协同 → URL 分类 → 后端调度 → HTML 正文清洗 → 质量过滤**的完整数据流，以及每个分支的失败语义。

## 1. 全景数据流

```text
query + query_domains + headers/cfg
  │
  ├─ get_retrievers(headers, cfg)                 actions/retriever.py:151-194
  │    → list[检索器类]（内置 match / 插件 entry point / 默认 Tavily）
  │
  ├─ ResearchConductor.plan_research()
  │    → 用 retrievers[0] 做一次初始搜索                  skills/researcher.py:59-119
  │    → 搜索结果喂给 plan_research_outline() 生成 sub-queries
  │
  └─ 对每个 sub-query（asyncio.gather 并发）              skills/researcher.py:414-428
       └─ _scrape_data_by_urls()                        :1026-1063
            ├─ _search_relevant_source_urls()           :890-1024
            │    ├─ 逐 retriever 实例化 + to_thread(search)
            │    ├─ requires_scraping 判定 → URL 队列 / prefetched_content
            │    └─ dict.fromkeys 去重 + visited_urls + shuffle
            ├─ scraper_manager.browse_urls(urls)        → scrape_urls()  actions/web_scraping.py:12-57
            │    └─ Scraper.run() → extract_data_from_url() × N  scraper/scraper.py:158-194,230-313
            │         ├─ validate_url()（SSRF 防护）   utils/url_security.py:63-119
            │         ├─ get_scraper(link)（路由）      scraper/scraper.py:366-415
            │         ├─ 正文抽取（bs / browser / pdf / ...）
            │         └─ 质量闸门（长度/拦截页/词表/PDF 二进制）
            └─ prefetched_content 合并 → list[{url, raw_content, image_urls, title}]
                 → context_manager.get_similar_content_by_query()（见 context_compression_and_prompts.md）
```

---

## 2. 检索器工厂：名字如何变成类

### 2.1 解析优先级

`get_retrievers(headers, cfg)`（`gpt_researcher/actions/retriever.py:151-194`）的优先级从高到低：

| 优先级 | 来源 | 形态 | 行号 |
|---|---|---|---|
| 1 | `headers["retrievers"]` | 逗号分隔字符串，`.split(",")` | :163-164 |
| 2 | `headers["retriever"]` | 单个名字 | :166-167 |
| 3 | `cfg.retrievers` | list 或逗号分隔字符串 | :169-174 |
| 4 | `cfg.retriever` | 单个名字 | :176-177 |
| 5 | `get_default_retriever()` | `TavilySearch` | :178-180,197-205 |

随后对每个名字做 `strip()`（:182-185，专门修复 `"tavily, exa"` 带空格被静默降级的问题），空列表回退默认，再映射为类（:187-194）。**关键坑**：未知名字不会报错，而是 `get_retriever(r) or get_default_retriever()`（:192）——也就是说拼错 retriever 名字会静默变成 Tavily，而不是失败。配置层的 `Config.parse_retrievers()`（`config/config.py:185-198`）会先用 `get_all_retriever_names()` 做校验并抛出 `Invalid retriever(s) found: ...`，所以走 `Config` 构造的路径会在启动阶段拦住拼错；直接传 headers 的路径不会。

### 2.2 内置名单与插件

`get_retriever()` 的 `match/case`（:50-137）内置 20 个名字：

```
google, searx, searchapi, serpapi, serper, duckduckgo, bing, brave, bocha,
arxiv, tavily, groundroute, exa, crw, semantic_scholar, pubmed_central,
custom, mcp, xquik, openalex, getxapi
```

其余名字走 `_load_plugin_retriever()`（:140-148）：从 `importlib.metadata.entry_points(group="gpt_researcher.retrievers", name=...)` 加载第三方类；加载异常只 `logger.warning` 并返回 `None`，最终由 :192 回退默认。`get_all_retriever_names()`（`retrievers/utils.py:86-107`）还会把插件名追加进列表，`get_plugin_retriever_names()`（:110-120）负责发现。内置名**永远优先**于插件名。

`check_pkg()`（`retrievers/utils.py:44-59`）用 `importlib.util.find_spec` 检测可选依赖，缺失时给出 `pip install -U <pkg>` 的明确提示（DuckDuckGo 用的 `ddgs` 就走这里，`duckduckgo.py:31`）。

### 2.3 最重要的接口：`requires_scraping`

`BaseRetriever`（`retrievers/base.py:35-61`）是一个**可选**基类：文档明确说“subclassing is not required”，下游只看属性 `requires_scraping`（:35-41）。其语义：

| 值 | 结果形态 | 下游行为 |
|---|---|---|
| `True`（默认） | `list[{href/url, body/snippet}]` | 一切随附文本都只是预览，URL 进入抓取队列 |
| `False` | `list[{url, raw_content}]` | 内容已被检索器取回，直接作为证据，不抓取 |
| 未声明 | 任意 | 退化为历史启发式：`raw_content and len(raw_content) > 100` 就当作已抓全文 |

历史启发式的来龙去脉写在 `retrievers/base.py:11-29`：超过 100 字的 snippet 会被误判为全文，导致 URL 从不抓取、报告里没有可验证引用（issue #1846/#1892）。因此**新检索器必须显式声明**；老插件保持旧行为以免破坏兼容。

当前显式声明情况（源码 grep 结果）：

| 检索器 | 声明 | 位置 |
|---|---|---|
| Tavily | `True` | `retrievers/tavily/tavily_search.py:23-25` |
| DuckDuckGo | `True` | `retrievers/duckduckgo/duckduckgo.py:27-29` |
| SearxNG | `True` | `retrievers/searx/searx.py:30-32` |
| PubMed Central | `False` | `retrievers/pubmed_central/pubmed_central.py:51-53` |
| Custom | `False` | `retrievers/custom/custom.py:11-13` |
| Arxiv / Semantic Scholar 等 | **未声明** | 走 100 字符历史启发式 |

未声明类要特别注意：Arxiv 返回摘要（:42-48）、Semantic Scholar 返回 abstract（:67-73），长度经常 >100，因此这些结果可能被当作 prefetched 内容而不走抓取；这不是配置错误，而是当前实现的真实行为（见 `skills/researcher.py:976`）。

---

## 3. 四种代表性检索器的实现细节

### 3.1 Tavily（默认）

`retrievers/tavily/tavily_search.py`：

- API key：先读 headers 的 `tavily_api_key`，没有则读 `TAVILY_API_KEY` 环境变量；缺失时打印提示并返回空串（:47-62）。**注意 __init__ 在设置 key 后又用 `{"Content-Type": "application/json"}` 覆盖了 `self.headers`（:38-44）**，key 作为 JSON body 的 `api_key` 字段发送（:93）。
- `site:` 运算符翻译（:117-139）：LLM 生成的查询常带 Google 风格 `site:domain/path`。正则 `_SITE_OPERATOR_PATTERN`（:15）提取后，从 query 中删除，只保留域名部分（`split("/")[0]`），合并进 `include_domains` 并去重。这是必要的，因为 Tavily 不接受 `site:`。
- 请求体与超时（:64-105）：`search_depth="basic"`、`topic`、`days=2`、`use_cache=True`；`requests.post(..., timeout=100)`；非 200 走 `raise_for_status()`。
- 查询长度：截断到 400 字符（:141-148），因为 Tavily 拒绝更长的 query。
- 结果防御（:149-170）：只接受 dict 响应与 list 型 `results`；逐条要求 dict 且 `url` 非空；`body = content or snippet or ""`；任何异常打印并降级为空列表；最后切片 `[:max_results]`。
- `max_results <= 0` 直接返回 `[]`（:113-114）。

### 3.2 DuckDuckGo（`ddgs`）

`retrievers/duckduckgo/duckduckgo.py`：

- `check_pkg('ddgs')` 后延迟导入 `DDGS`（:31-33）；
- `query_domains` 被翻译成 `(site:d1 OR site:d2) query`（:50-59），与 Tavily 的 include_domains 参数化方式不同；
- `ddg.text(search_query, region='wt-wt', max_results=...)`（:62）；异常降级为空（:63-65）；
- **规范化层**（:70-92）：兼容 ddgs 新旧版本的键名——URL 取 `href|link|url`，正文取 `body|snippet|description`，可选 `title`；缺失 URL 的行被丢弃；
- **`_MAX_PREFETCHED_LEN = 100`**（:19）：body 被截断到 100 字符，专门为了让“未声明 requires_scraping 的下游启发式”不把 snippet 误判为全文（:4-18）；截断只影响分类和早期规划，真正抓取后用的是页面全文。
- 最后用 `islice(normalized, max_results)` 保证客户端忽略 `max_results` 时也不超量（:94-95）。

### 3.3 Arxiv 与 Semantic Scholar（学术检索）

| 项 | Arxiv（`retrievers/arxiv/arxiv.py`） | Semantic Scholar（`retrievers/semantic_scholar/semantic_scholar.py`） |
|---|---|---|
| 客户端 | `arxiv.Client().results(arxiv.Search(...))`（:23-29） | `requests.get(BASE_URL, params=..., timeout=10)`（:43-46） |
| 排序 | `sort='Relevance'|'SubmittedDate'` → `arxiv.SortCriterion`（:8-12），构造时 `assert` | `relevance/citationCount/publicationDate`（:12,22）；**保留 camelCase**，因为 API 要求原样（:23-27） |
| 字段 | `title`, `pdf_url or entry_id → href`, `summary → body`（:39-48）；跳过无 href 的残缺结果 | `fields=title,abstract,url,venue,year,authors,isOpenAccess,openAccessPdf`（:36-41） |
| 重要过滤 | 无 | 只保留 `isOpenAccess` **且** `openAccessPdf` 是 dict、有 url 的论文（:61-66）；非 OA 结果被丢弃 |
| 失败语义 | 异常打印并 `[]`（:30-32） | 请求/解析异常打印并 `[]`（:47-55） |

两者都**未声明** `requires_scraping`，因此它们的摘要长度会触发历史启发式（见 §2.3）；设计自定义学术检索器时优先显式声明以避免歧义。

### 3.4 其余检索器速查

| 名字 | 工厂位置 | 备注 |
|---|---|---|
| Google 自定义搜索 | `actions/retriever.py:51-54` / `retrievers/google/google.py` | 依赖 Google CSE 凭据 |
| Serper / SerpAPI / SearchAPI / Bing / Brave / Bocha | :59-86,95-98 | 商业搜索 API；作为 URL 来源，通常走抓取路径 |
| SearxNG | :55-58 | 自托管；与 DDG 一样截断 snippet 到 100 字（`searx.py:7-22`） |
| Exa / CRW / GroundRoute | :99-106 | 内容型/API 型服务 |
| OpenAlex / PubMed Central | :111-114,127-130 | 学术数据源；PMC 声明 prefetched |
| MCP | :119-122 | 两阶段工具发现与调用；在 `ResearchConductor` 中有专门缓存/策略，见 §4.3 |
| Custom | :115-118 | 契约 `{url, raw_content}`；`RETRIEVER_ENDPOINT` + `RETRIEVER_ARG_*` 环境变量（`custom.py:15-31`），GET + `timeout=20`（:51-55），严格返回 list（:63-94） |
| Xquik / GetXAPI | :123-134 | X/Twitter 数据源 |

---

## 4. 多检索器协同机制

### 4.1 初始搜索与规划只使用第一个检索器

`_get_initial_search_results()`（`skills/researcher.py:59-84`）调用 `get_search_results(query, self.researcher.retrievers[0], ...)`，异常时发 `search_failed` 事件（带 `reason_code=异常类名`）再抛出。`plan_research()`（:86-119）把这些结果作为 `context` 传给 `plan_research_outline()`，让 sub-query 生成基于**真实网页信息**而不是模型记忆（`actions/query_processing.py:109-115`、`prompts.py:239-244`）。

`_get_context_by_web_search()`（:329-439）随后调用 `_get_context_from_initial_results()`（:875-888）：把规划用的初始结果也当成证据抓取并压缩，避免“搜过但没进报告”。当首个 retriever 是 MCP 时跳过这一步（:877-878）。

### 4.2 每个 sub-query 如何 fan-out

`_search_relevant_source_urls()`（:890-1024）的循环：

1. 若传了 `initial_results`，只用 `retrievers[:1]`（:900），避免把首个检索器的结果重复归因到其他 provider；
2. 跳过 MCP retriever（:902-904）；
3. `retriever_class(query, query_domains=query_domains)` 实例化后，用 `asyncio.to_thread(retriever.search, max_results=cfg.max_search_results_per_query)` 保持事件循环空闲（:906-916）；
4. 空结果发 `search_empty` 事件并 continue（:918-924）；
5. 逐条结果取 `url = result.get("href") or result.get("url")`（:935），无 URL 跳过；发 `candidate_discovered` 事件（含 rank）（:941-947）；
6. 按 `requires_scraping` 三分支（:949-994）：True → URL 队列；False → 有 raw_content 就进 prefetched、注册来源、加入 `visited_urls`、发 `content_prefetched`；未声明 → 长度 >100 的旧逻辑；
7. `dict.fromkeys` 保序去重 → `_get_new_urls()` 按 `visited_urls` 再过滤（:1004-1006,852-873）；
8. 发 `selected_for_fetch` / `candidate_deduplicated(reason_code=ALREADY_VISITED)`（:1007-1021）；
9. `random.shuffle(new_search_urls)`（:1022）后返回 `(urls, prefetched)`；
10. `_scrape_data_by_urls()`（:1026-1063）把抓取结果与 prefetched 合并（:1055-1058），再灌入 vector store（:1060-1061）。

`stream_output` 会把这些阶段推给前端（如 `added_source_url`、`researching`），而 `ResearchObserver` 事件（通过 `_emit_research_event`，:51-57）给程序化观测使用；两者不是同一条通道。

### 4.3 MCP 的特殊策略

- `_get_mcp_strategy()`（:441-465）解析 `fast/deep/disabled`；`fast` 用 `_mcp_results_cache` 只跑主查询一次（:362-376），`deep` 每条 sub-query 都跑（:377-387），`disabled` 完全跳过（:352-361）。
- 缓存用 `asyncio.Lock` 保护，因为 hybrid 模式下两路研究并发（:348-350）。
- `_tavily_mcp_redundant_with_direct()`（:523-539）在默认 `RETRIEVER=tavily` 且同时配置 tavily-mcp 时跳过 MCP 路径，避免双重付费（调用处 :572-584）。
- `_combine_mcp_and_web_context()`（:778-830）把两路上下文合并成一份字符串。

---

## 5. 抓取器调度：`scrape_urls()` → `Scraper`

### 5.1 入口与并发模型

`actions/web_scraping.py:12-57`：

- 构造 `Scraper(urls, user_agent, cfg.scraper, worker_pool, event_handler)`；`user_agent` 来自 `cfg.user_agent`（:27-31）；
- `await scraper.run()`，并从每条结果的 `image_urls` 汇总图片（:42-48）；
- `finally` 中关闭 `requests.Session`，释放连接池（:51-55）；
- 任何异常都打印并降级为部分结果（:49-50）。

`Scraper.__init__()`（`scraper/scraper.py:121-156`）：

- `dict.fromkeys(urls)` 保序去重（:134-138）；
- `requests.Session()` 挂上 `validate_redirect_hook`（:139-141）：重定向目标也要过 SSRF 校验（`utils/url_security.py:131-138`）；
- 依赖 `tavily_extract` / `firecrawl` 时调用 `_check_pkg()`，必要时 `pip install`（:143-146,196-228）。

`Scraper.run()`（:158-194）用 `asyncio.gather` 并发所有 URL，然后：

1. 对每条结果按 `raw_content is not None` 发 `fetch_succeeded` / `fetch_failed` 事件，字段含 `url/scraper/content_length`，失败附带 `failure_reason`（:166-184）；
2. 只保留 dict 且 `raw_content is not None` 的行（:186-193）——**失败行被静默丢弃**，这是“报告来源数少于预期”的直接原因。

`WorkerPool`（`utils/workers.py:8-50`）：`ThreadPoolExecutor(max_workers)` + `asyncio.Semaphore(max_workers)` 控制本池并发；`throttle()` 还等待**全局** `GlobalRateLimiter`（`utils/rate_limiter.py:13-90`，单例 + 全局 lock，`SCRAPER_RATE_LIMIT_DELAY` 秒数间隔）。`extract_data_from_url()` 在 `async with self.worker_pool.throttle()` 内执行全部工作（:234）。

### 5.2 后端路由规则

`get_scraper(link)`（`scraper/scraper.py:366-415`）的映射表（:384-393）：

| key | 类 | 选择条件 |
|---|---|---|
| `pdf` | `PDFScraper` | `urlparse(link).path.lower().endswith(".pdf")`（只看 path，忽略 query/fragment；:397-405） |
| `arxiv` | `ArxivScraper` | hostname 为 `arxiv.org` 或以 `.arxiv.org` 结尾（:403-407） |
| 其他 | 由 `cfg.scraper` 决定 | `bs`（默认）/ `web_base_loader` / `browser` / `nodriver` / `tavily_extract` / `firecrawl` |

未知 key 抛 `Exception("Scraper not found.")`（:411-413）。若配置的 scraper 不存在映射，整批任务都会失败——这是配置检查的重点。

各后端执行方式（`extract_data_from_url()` :257-266）：有 `scrape_async()` 的（NoDriver、FireCrawl）直接 await；其余同步 `scrape()` 放到 `worker_pool.executor` 里跑，避免阻塞事件循环。

### 5.3 BeautifulSoup 后端与重试策略

`scraper/beautiful_soup/beautiful_soup.py`：

- `RETRYABLE_STATUS_CODES = {429,500,502,503,504}`（:11）；`MAX_CONTENT_BYTES = 10MB`（:12）；
- `_fetch()`（:65-127）最多两次尝试（:77）：
  - 网络异常 → sleep 1s 后重试（:84-90）；
  - 429/5xx → sleep 1s 后重试（:92-97）；
  - **403 → 换透明身份重试**：`GPT-Researcher/1.0 (+https://github.com/assafelovic/gpt-researcher; source retrieval)`（:13-16,99-105）；
  - 其他 ≥400 → `failure_reason = HTTP_<code>`，不把错误页当正文（:107-111）；
  - `Content-Length > 10MB` 直接跳过（:113-123）；
- `scrape()`（:28-63）：只有 Content-Type 声明 charset 时才信任 `response.encoding`，否则让 BeautifulSoup 从文档自行检测（:40-48）；依次 `clean_soup → get_text_from_soup → get_relevant_images → extract_title`；解析异常返回 `("", [], "")`。

### 5.4 浏览器后端（Selenium / NoDriver）

`scraper/browser/browser.py`（Selenium）：

- 构造时即导入 Selenium（:76-96），默认 Chrome、**`headless=False`**（:41-45）；
- `scrape()` 顺序：`setup_driver → _visit_google_and_save_cookies → _load_saved_cookies → _add_header → scrape_text_with_selenium`，finally 中 `driver.quit()` 并删除 cookie 文件（:51-74）；
- cookie 文件名是 8 位随机串（:49,178-180），Google 访问后 pickle 保存（:190-204）；`browser_cookie3` 可选（:146-165）；
- Linux 下加 `--disable-dev-shm-usage`、`--remote-debugging-port=9222`、`--no-sandbox`（:119-124）；
- `WebDriverWait(..., 20).until(presence_of_element_located(body))`（:210-212），超时返回 20 字符的 `"Page load timed out"`（:216）——**该字符串短于 100 字符，会被上层长度检查丢弃**，不会污染正文；
- `_scroll_to_bottom()`（:241-250）每 2 秒滚动到页底，直到 scrollHeight 不再变化；
- PDF/arxiv 分支分别调用 `scrape_pdf` / `scrape_pdf_with_arxiv`（:220-226）；普通页面取 `document.documentElement.outerHTML` 后走与 BeautifulSoup 相同的清洗函数（:228-239）。

`scraper/browser/nodriver_scraper.py`（zendriver，异步）：

- 浏览器池：`max_browsers=5`、`browser_load_threshold=8`，按事件循环共享（:16-21,136-175）；
- 每个域一个 `asyncio.Semaphore(1)`，若已上锁再随机 sleep 0.6–1.2s（:107-128）；
- 滚动策略：随机 46–97% 步幅、等待 idle、随机 0.23–0.56s 间隔，直到累计 ≥500% 或到达页底（:59-80）；
- `scrape_async()`（:190-260）：`get → wait complete/idle → sleep → wait idle → scroll → get_content → BeautifulSoup 清洗`；`len(text)<200` 时仅告警（:228-243）；`debug=True` 可存截图；finally 释放 tab 与 browser，避免槽位泄漏（:253-260）。

其他后端：

| 后端 | 文件 | 要点 |
|---|---|---|
| `tavily_extract` | `scraper/tavily_extract/tavily_extract.py:5-76` | 用 `TavilyClient.extract` 取 `raw_content`（:38-57）；`failed_results` 非空即失败（:43-46）；仅当 session 存在时才走 HTML 侧路取图片/标题（:59-70） |
| `firecrawl` | `scraper/firecrawl/firecrawl.py:8-19,56-121` | 模块级 semaphore，`FIRECRAWL_CONCURRENCY` 默认 2（免费档）；`FIRECRAWL_SERVER_URL` 可覆盖官方地址（:43-54）；有同步与异步两条路径 |
| `web_base_loader` | `scraper/web_base_loader/web_base_loader.py:6-52` | LangChain WebBaseLoader 路径，返回三元组 |
| `pdf` | `scraper/pdf/pdf.py:49-104` | 流式下载到临时文件（8192 字节块），SSL 失败时 `verify=False` 重试（:61-69），`PDFLoader` 解析后**始终删除临时文件**（:71-85）；读取所有页而非只第一页（:92-94）；`_fallback_title()` 从 URL 派生标题（:10-20） |
| `arxiv` | `scraper/arxiv/arxiv.py:13-70` | 用正则从 URL/裸 id 提取 arXiv id（:13-29），`arxiv.Client` 取摘要并拼成 `Published: ...; Author: ...; Content: ...`（:61-68）；无结果时降级为空（:55-59） |

---

## 6. 网页正文清洗与过滤算法

### 6.1 HTML → 文本：`scraper/utils.py`

```text
clean_soup(soup)                         :128-158
  ├─ decompose 标签：script/style/footer/header/nav/menu/sidebar/svg  :130-142
  └─ decompose class ∈ {nav,menu,sidebar,footer} 的元素             :144-156

get_text_from_soup(soup)                 :224-244
  └─ _block_lines(soup)                  :177-221
       ├─ 显式栈遍历（避免深层嵌套触发递归上限）              :192-215
       ├─ _BLOCK_TAGS 边界换行（div/p/li/table/...）         :163-168,209-211
       ├─ <br> 换行；_BOX_TAGS(button/select) 前后补空格      :201-214
       ├─ _PREFORMATTED_TAGS(pre/textarea/...) 保留原始空白    :170,203-207
       └─ 只接受 NavigableString/CData，注释与脚本字符串不入正文 :216-219
  普通块：\n 连接后用 re.sub(r"\s{2,}", " ", ...) 归一空白    :237,243
  预格式块：绕过归一化（空白是内容）                       :235-239
```

设计要点：`soup.get_text(separator="\n")` 会把被 `<a>`/`<em>` 拆开的词也拆到不同行；`_block_lines()` 只在**块级元素**边界换行，因此句子完整性更好。HTML 空白集合只含 `[ \t\n\r\f]`（:174），`&nbsp;` 被视为内容而非空白。

### 6.2 图片打分

`get_relevant_images()`（`scraper/utils.py:16-63`）：

- `urljoin(url, img['src'])` 只保留 http(s)（:24-28）；
- 打分：class 命中 `header/featured/hero/thumbnail/main/content` → 4 分（:31-37）；否则按 width×height：≥2000×1000→3、≥1600 或 ≥800→2、≥800 或 ≥500→1、≥500 或 ≥300→0，更小的跳过（:38-52）；
- 按分数降序取前 10（:56-59）；
- `parse_dimension()`（:65-86）处理 `px` 后缀、浮点字符串、非数字值（debug 级日志，不刷屏）。

### 6.3 `Scraper.extract_data_from_url()` 的质量闸门

按顺序（:230-313）：

1. **SSRF 检查**：`validate_url(link)`，不安全则 warn 并返回 `raw_content: None`（:238-247）。`utils/url_security.py:35-60,79-119` 只允许 http/https、必须有 host、DNS 解析后的 IP 不得是 private/loopback/link-local/reserved/multicast/unspecified/非 global；`ALLOW_PRIVATE_URLS=true` 可显式放行（:44-45,95-98）。注释明确说明 DNS 重绑定 TOCTOU 窗口仍存在（:22-25）。
2. **路由 + 执行**（:249-266）。
3. **最小长度**：`content` 为空或 `<100` 字符 → `_reject(..., reason or "Content too short or empty")`（:268-272）。
4. **反爬/挑战页**：`_looks_like_block_page()`（:80-82）只检查前 5000 字符（:62），匹配 `_BLOCK_PAGE_MARKERS`（:41-56：Anubis、Cloudflare "Checking your browser"、"enable javascript and cookies"、ResearchGate 特定文案、"Client Challenge" 两种撇号形式等）；命中即 `_reject(..., "Anti-bot/challenge page detected")`（:281-282）。前缀限制的原因写在 :58-61：拦截页短且出现在开头，避免对多 MB 正文反复小写化扫描。
5. **词表噪声**：`_looks_like_word_list()`（:85-89）只对 ≥200,000 字符的文本生效，且句末标点密度 `< 1/5000` 才命中；`_SENTENCE_ENDING_CHARS` 包含 `.!?。！？`（:71-77），避免误伤中文长文。
6. **未解析 PDF 二进制**：`_looks_like_unextracted_pdf()`（:109-113）检查 `%PDF-` 开头或 `endobj/endstream//FlateDecode/xref/trailer` 中至少 2 个标记（:105-106）；若当前后端不是 PDFScraper，则 `_retry_as_pdf()`（:333-364）用 PDFScraper 重抓，并对重抓结果重复全部质量检查；仍失败才丢弃。
7. **成功返回** `{url, raw_content, image_urls, title}`（:304-309）；任何异常都记 error 并返回 `raw_content: None`（:311-313）。

`_reject()`（:315-330）刻意保持与异常失败相同的形状（`raw_content: None`），只多一个 `failure_reason` 字段供诊断；`Scraper.run()` 的过滤逻辑因此不需要分叉。

### 6.4 数据形状速查

| 阶段 | 形状 | 关键键 |
|---|---|---|
| retriever.search() | `list[dict]` | `href/url`, `body/snippet` 或 `raw_content`, 可选 `title` |
| `_search_relevant_source_urls()` | `(list[str], list[dict])` | URL 队列 + prefetched `{url, raw_content}` |
| scraper backend | `(str, list, str)` | `(content, image_urls, title)` |
| `extract_data_from_url()` | `dict` | `url`, `raw_content`(可为 None), `image_urls`, `title`, 可选 `failure_reason` |
| `Scraper.run()` | `list[dict]` | 仅 `raw_content is not None` 的行 |
| `_scrape_data_by_urls()` | `list[dict]` | 抓取行 + prefetched 行 |
| `select_context()` | `str` | `Source:/Title:/Content:` 格式（见 [`context_compression_and_prompts.md`](context_compression_and_prompts.md)） |

---

## 7. 扩展与维护不变量

1. 新检索器：**显式声明 `requires_scraping`**，返回 `{href 或 url, ...}`；不要把 snippet 塞进 `raw_content`。
2. 新抓取器：在 `Scraper.get_scraper()` 的 `SCRAPER_CLASSES`（:384-393）登记；实现 `scrape() -> (content, image_urls, title)`，或 `scrape_async()`；失败返回空三元组而非异常。
3. 修改 `clean_soup()` 的标签白名单时，注意 `get_text_from_soup()` 的块级换行依赖这些标签是否还存在。
4. 修改质量阈值（100 字符 / 5000 字符 / 200k 字符 / 2 个 PDF 标记）时，同步更新故障排查文档与回归测试；这些阈值直接决定“来源是否消失”。
5. SSRF 防护是纵深防御而非完整方案（DNS 重绑定窗口，`url_security.py:22-25`）；不要把它当作安全边界的唯一一层。
6. `BaseRetriever` 的 100 字符启发式只为兼容旧插件而存在；新代码不要依赖它。
