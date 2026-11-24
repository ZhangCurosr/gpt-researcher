# 上下文压缩与 Prompt 模板完全参考

> 事实来源：本仓库源码；`路径:行号` 为本检出版本锚点。
>
> 配套：[`retrievers_and_scrapers_deep_dive.md`](retrievers_and_scrapers_deep_dive.md) 负责“文本从哪来”，本文负责“文本如何被切块、打分、截断、拼成 Prompt，以及每个内置 Prompt 长什么样”。

## 1. 从网页文本到 LLM 上下文：管线全景

```text
list[{url, raw_content, title}]                ← scraper / prefetched retriever
  │
  ▼  SearchAPIRetriever._get_relevant_documents   context/retriever.py:20-38
  │    per-page 截断 MAX_CONTENT_CHARS（默认 50000 字符）
  ▼  Document(page_content, metadata{title, source})
  │
  ▼  select_context(query, pages, cfg, ...)      context/select.py:53-101
  │    ├─ mode=none       → pretty_print_docs 全部页面
  │    ├─ 小内容短路      → COMPRESSION_THRESHOLD（默认 8000 字符）且页数 ≤ max_results
  │    ├─ mode=jev        → JevContextCompressor（四档 usefulness 评分）
  │    ├─ mode=embeddings → ContextCompressor（Recursive splitter + EmbeddingsFilter）
  │    └─ 其他/失败降级   → LexicalContextCompressor（BM25）
  ▼  str： "Source: ...\nTitle: ...\nContent: ..."
  │
  ▼  generate_report_prompt / generate_subtopic_report_prompt / ...
  ▼  create_chat_completion(smart_llm, temperature=0.35, max_tokens=SMART_TOKEN_LIMIT)
```

**一个关键认知**：这里的“压缩”是**选择式压缩（selection）**，不是生成式摘要。代码把长文本切成块、按分数挑块，然后把原文块拼进 Prompt；只有 `generate_summary_prompt` / `generate_quick_summary_prompt` 才让 LLM 真正改写内容（`prompts.py:517-552`）。

---

## 2. 切块与相似度过滤的实现细节

### 2.1 统一入口 `select_context()`

`context/select.py:1-13` 的模块注释给出模式语义：

| `CONTEXT_FILTER` | 行为 | 依赖 |
|---|---|---|
| `auto` | 有 `TYPESAFE_API_KEY` → `jev`；否则 `keyword` | 见 :38-46 |
| `jev` | TypeSafe Jev 逐块打 usefulness 分 | 需要 key |
| `keyword` | 本地 BM25，无 API/模型/embedding | 无 |
| `embeddings` | embedding 余弦相似度过滤 | 需要 embedding provider |
| `none` | 全量输出，不过滤 | 无 |

`resolve_context_filter()`（:38-46）把未知值降级为 `auto`；`auto` 依据 `TYPESAFE_API_KEY` 是否存在决定；`DEFAULT_CONFIG["CONTEXT_FILTER"]="auto"`（`config/variables/default.py:7`）。

`select_context()`（:53-101）的执行顺序：

1. `mode = resolve_context_filter(cfg.context_filter)`（:68）；
2. `none` → 直接 `pretty_print_docs(_as_documents(pages))`（:71-72）；
3. **小内容短路**：`total_chars = sum(len(str(p.get("raw_content") or "")))`，若 `< COMPRESSION_THRESHOLD`（默认 8000，:76）且 `len(pages) <= max_results`，直接输出前 `max_results` 个文档（:74-77）。这解释了为什么小报告根本不会调用 Jev/BM25/embedding；
4. `jev` → 捕获 `JevError` 后 warning 并降级到 keyword（:79-84）；
5. `embeddings` → 任何异常都 warning 并降级到 keyword（:86-97），`embeddings()` 是零参数工厂，只有选中该模式才会构造模型（:65-66）；
6. 兜底 `LexicalContextCompressor`，阈值与条数来自环境变量 `KEYWORD_RELATIVE_THRESHOLD=0.5` / `KEYWORD_MAX_RESULTS=25`（:31-35,99-101）。

注意 `keyword` 模式**不使用**调用方传入的 `max_results=10`，而用 `KEYWORD_MAX_RESULTS=25`（:99-101）；这是有意的评测结论（:31-33：相对阈值 + 更多候选击败纯 top-10 BM25 与 embeddings）。

### 2.2 每页字符上限与 Document 化

`context/retriever.py`：

- `_MAX_CONTENT_CHARS = int(os.environ.get("MAX_CONTENT_CHARS", 50000))`（:9-13），默认约 12.5k tokens；注释指出设置原因是大 PDF 可能超过 embedding API 的单请求 token 上限；
- `SearchAPIRetriever._get_relevant_documents()`（:20-38）把 `raw_content` 先 `or ""` 再切片（:26-29，避免 `None` 切片 TypeError），metadata 中 `source = page.get("source") or page.get("url") or ""`（:30-33）——**这就是最终 Prompt 里 `Source:` 的来源**；
- `SectionRetriever`（:40-70）用于已写章节：`written_content → page_content`，`section_title → metadata`，不做截断。

### 2.3 三种切块器对照

| 组件 | splitter 参数 | 每页上限 | 选择依据 | 位置 |
|---|---|---|---|---|
| `ContextCompressor`（embeddings） | `RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=100)` | 每页 50k 字符（`SearchAPIRetriever`） | `EmbeddingsFilter(similarity_threshold=...)` | `context/compression.py:130-149,101-128` |
| `JevContextCompressor` | `chunk_size` 来自 `JEV_CHUNK_SIZE`（默认 1000），overlap = `chunk_size // 10` | 同上 | Jev 四档分数 ≥ `JEV_MIN_SCORE`（默认 1.5） | `context/jev_filter.py:139-161,173-190` |
| `LexicalContextCompressor`（BM25） | `chunk_size=1000`，overlap = `chunk_size // 10` | 同上 | BM25 分数 ≥ `relative_threshold * best` | `context/lexical.py:77-104` |
| `WrittenContentCompressor` | `chunk_size=1000, chunk_overlap=100` | 不截断 | embedding 相似度阈值（构造参数） | `context/compression.py:221-239` |
| `rank_written_sections`（无 embedding 回退） | `chunk_size=1000, chunk_overlap=100` | 无 | BM25 + 0.5 相对阈值 | `context/lexical.py:110-133` |

`RecursiveCharacterTextSplitter` 的具体分隔符序列来自 LangChain，仓库未硬编码；可确认的是上述 size/overlap 与“先切块再过滤”的顺序。所有切块路径都用 `SearchAPIRetriever(pages).invoke("")` 或 `SectionRetriever` 先转成 `Document`（`lexical.py:91`、`jev_filter.py:159`、`compression.py:143-145`）。

### 2.4 embeddings 路径

`ContextCompressor.__get_contextual_retriever()`（`compression.py:130-149`）组装：

```text
SearchAPIRetriever(pages)
  → ContextualCompressionRetriever(
        base_retriever=上面的 retriever,
        base_compressor=DocumentCompressorPipeline(
            transformers=[RecursiveCharacterTextSplitter, EmbeddingsFilter]))
```

- 阈值：构造参数 `similarity_threshold` 为 None 时读 `SIMILARITY_THRESHOLD` 环境变量（默认 `0.35`，:125-127）；但正常应用路径中 `select_context()` 会传 `cfg.similarity_threshold`（`select.py:93`），而 `DEFAULT_CONFIG` 设为 `0.42`（`config/variables/default.py:6`），因此**应用内实际生效值通常是 0.42**；只有直接构造 compressor 且不传阈值时才是 0.35。
- `async_get_context()`（:151-191）先做与 `select_context` 相同的小内容短路（:165-184），大内容才调用 `estimate_embedding_cost(model=OPENAI_EMBEDDING_MODEL, docs=...)`（:188-189，成本通过 `cost_callback` 上报）并在线程池里 `compressed_docs.invoke(query)`（:190）。
- `VectorstoreCompressor`（:36-85）走 `vector_store.asimilarity_search(query, k, filter)`，不切块，默认 `max_results=7`；`ContextManager` 调用时传 8（`skills/context_manager.py:83-87`）。

### 2.5 Jev 路径：按“是否有用”而不是“是否相关”打分

`context/jev_filter.py` 的设计说明（:1-14）：余弦相似度衡量主题接近度，无法区分“同主题废话”与“真正回答问题的段落”，所以直接问模型。

| 元素 | 实现 | 行号 |
|---|---|---|
| 四档评分标准 `RELEVANCE_LEVELS` | Unrelated / Same topic but no help / Partially answers / Directly answers | :37-42 |
| 请求体 | `{"state": passage, "model": JEV_MODEL, "questions": {"usefulness": {"type":"score","instructions":"How useful is this passage for answering the question: {query}","criteria": RELEVANCE_LEVELS}}}` | :90-103 |
| endpoint | `TYPESAFE_BASE_URL`（默认 `https://api.typesafe.ai/v1`）+ `/systemone` | :32-35,78-82 |
| 鉴权 | `Authorization: Bearer TYPESAFE_API_KEY`，缺 key 直接 `JevError` | :74-76,103 |
| 并发 | 每个事件循环一个 `Semaphore(JEV_CONCURRENCY=64)`，用 `WeakKeyDictionary` 按 loop 隔离 | :44-54 |
| 重试 | 最多 `max_retries=3` 次；只对 429/529 重试；指数退避 `0.5 * 2 ** attempt` | :72,104-125 |
| 分数解析 | `body["answers"]["usefulness"]["score"]` → float，范围 [0,3]；记录 `usage.input_tokens` | :111-118 |
| 成本 | `input_tokens / 1e6 * 0.042` 美元，输出免费 | :34-35,127-129 |
| 选择 | 所有块按 `(-score, position)` 排序；保留 `score >= JEV_MIN_SCORE` 的前 `max_results` 个 | :163-190 |

`JevError` 会冒泡到 `select_context()` 的 except（:83-84），降级到 BM25——因此 Jev 不可用**不会**让研究失败，但会改变上下文选择结果，排查时务必看 warning 日志。

### 2.6 BM25 路径：零依赖的本地排序

`context/lexical.py`：

- 分词：`_TOKEN = re.compile(r"\w+", re.UNICODE)`（:19），小写化、长度 >1、过滤 `_STOPWORDS`（:20-28）；
- 轻量词干 `_stem()`（:31-41）：`batteries→battery`、`costs→cost`、`ss` 结尾不动、长度 ≤4 不动；
- BM25 公式（:48-66）：
  ```text
  idf(t) = ln((N - df(t) + 0.5) / (df(t) + 0.5) + 1)
  score(d) = Σ_{t∈query} idf(t) * tf(t,d) * (k1+1) / (tf + k1*(1 - b + b*len(d)/avg_len))
  k1 = 1.5, b = 0.75
  ```
  IDF 从**本次 sub-query 抓到的块**自身计算，不使用全局语料；
- 选择（:95-104）：最优分 `best`；若 `best <= 0`（完全无词面重叠）则退回前 `max_results` 个块（:99-101）；否则按分数降序、同分按原顺序，保留 `score >= relative_threshold * best` 的前 N 个（:102-104）；
- `rank_written_sections()`（:110-133）是已写章节的同款回退：把 `section_title + "\n" + chunk` 一起打分，无重叠时返回 `[]`（:127-128）。`ContextManager` 在 embedding 不可用或单条查询失败时用它（`context_manager.py:116-123,158-164`）。

### 2.7 输出格式与模型特化

`PromptFamily.pretty_print_docs()`（`prompts.py:554-561`）：

```python
"\n".join(
    f"Source: {d.metadata.get('source')}\n"
    f"Title: {d.metadata.get('title')}\n"
    f"Content: {d.page_content}\n"
    for i, d in enumerate(docs)
    if top_n is None or i < top_n
)
```

- `join_local_web_documents()`（:563-566）：`Context from local documents: ...\n\nContext from web sources: ...`；hybrid 模式用它合并两路上下文（`skills/researcher.py:216-219`）。
- Granite 3（:776-803）：用 `<|start_of_role|>documents<|end_of_role|>` / `<|end_of_text|>` 包裹，每篇以 `Document <source>\nTitle: ...\n` 开头；合并时剥离重复的角色标记。
- Granite 3.3（:806-834）：每篇用 `<|start_of_role|>document {"document_id": "..."}<|end_of_role|>...<|end_of_text|>`；标题被前置到正文（:813-818）。
- `GranitePromptFamily`（:756-773）根据 `cfg.smart_llm` 字符串里是否含 `3.3` / `3` 动态委托给上述两个类。

工厂：`prompt_family_mapping`（:879-886）把 `default/granite/granite3/granite31/granite32/granite33` 映射到类；未知值 warning 后回退 `PromptFamily`（:889-903）。

### 2.8 大长篇报告的“溢出”发生在哪里

代码里**没有**一个统一的全局 token 预算分配器；溢出由多处局部上限共同约束：

| 位置 | 限制 | 行号/配置 |
|---|---|---|
| 每页内容 | `MAX_CONTENT_CHARS=50000` | `context/retriever.py:9-13` |
| 小内容短路 / 大内容才压缩 | `COMPRESSION_THRESHOLD=8000` | `select.py:76`, `compression.py:167`, `.env.example:54-60` |
| 关键词路径候选数 | `KEYWORD_MAX_RESULTS=25` | `select.py:35` |
| 研究上下文传给写作 | `max_results=10`（ContextManager 固定） | `context_manager.py:56-64` |
| 单次写作输出 | `SMART_TOKEN_LIMIT=12000`（默认）；`.env.example` 建议长输出模型设 16000–32000 | `config/variables/default.py:15`, `.env.example:76-92` |
| 规划输出 | `STRATEGIC_TOKEN_LIMIT=8000`；规划失败先用该上限重试，再回退 smart LLM | `default.py:16`, `actions/query_processing.py:117-154` |
| 快速摘要 | `SUMMARY_TOKEN_LIMIT=700` | `default.py:19` |
| 浏览分块 | `BROWSE_CHUNK_MAX_LENGTH=8192` | `default.py:17` |
| 递归深度研究 | `DEEP_RESEARCH_BREADTH=3 / DEPTH=2 / CONCURRENCY=4` | `default.py:40-42` |
| 已写章节复用 | 每路最多 10 条、去重后截断 | `context_manager.py:110-130` |
| 空上下文弃答 | 无材料时直接返回拒答文本，不调用 LLM | `skills/writer.py:79-88` |

`add_references()`（`actions/markdown_processing.py:105-131`）用 `sorted(visited_urls)` 输出确定性的 `## References` 列表——注意这是**追加抓取过的 URL**，并不验证正文中是否真的引用了它们。

---

## 3. 内置 Prompt 模板：结构、占位符、调用点

### 3.1 总表

| 方法（`gpt_researcher/prompts.py`） | 行号 | 占位符 | 调用点 |
|---|---|---|---|
| `generate_mcp_tool_selection_prompt` | :40-83 | `query`, `tools_info`(JSON), `max_tools` | MCP tool_selector |
| `generate_mcp_research_prompt` | :86-118 | `query`, `tool_names` | MCP research |
| `generate_image_analysis_prompt` | :122-175 | `max_images`, `query`, `sections_text`（每节前 500 字） | 图像生成流程 |
| `generate_image_prompt_enhancement` | :178-210 | `research_topic`, `base_prompt`, `section_content[:800]` | 图像生成流程 |
| `generate_search_queries_prompt` | :213-259 | `task`, `context`, 当前日期, `max_iterations` 动态示例 | `actions/query_processing.py:109-115` |
| `generate_report_prompt` | :262-316 | `context`, `question`, `total_words`, `report_format`, `reference_prompt`, `tone_prompt`, `language`, 日期 | `actions/report_generation.py:250-258` |
| `curate_sources` | :319-350 | `query`, `max_results`, `sources` | `skills/curator.py` |
| `generate_resource_report_prompt` | :353-390 | `context`, `question`, `total_words`, `language`, `reference_prompt` | report type = resource |
| `generate_custom_report_prompt` | :393-396 | `context`, `query_prompt` | report type = custom |
| `generate_outline_report_prompt` | :399-415 | `context`, `question`, `total_words` | report type = outline |
| `generate_deep_research_prompt` | :418-487 | `context`, `question`, `total_words`, `report_format`, `tone_prompt`, `language`, `reference_prompt`, 日期 | report type = deep |
| `auto_agent_instructions` | :490-515 | 无（含 3 个 few-shot 示例） | `actions/agent_creator.py` |
| `generate_summary_prompt` | :518-529 | `data`, `query` | 总结步骤 |
| `generate_quick_summary_prompt` | :532-552 | `query`, `context` | quick search |
| `pretty_print_docs` | :554-561 | docs → `Source/Title/Content` | 所有上下文选择路径 |
| `join_local_web_documents` | :563-566 | `docs_context`, `web_context` | hybrid |
| `generate_subtopics_prompt` | :573-593 | `{task}`, `{data}`, `{subtopics}`, `{max_subtopics}`, `{format_instructions}` | `utils/llm.py:construct_subtopics` |
| `generate_subtopic_report_prompt` | :596-670 | `context`, `current_subtopic`, `main_topic`, `max_subsections`, `report_format`, `existing_headers`, `relevant_written_contents`, `total_words`, `tone`, `language`, 日期 | `actions/report_generation.py:253-254` |
| `generate_draft_titles_prompt` | :673-704 | `context`, `current_subtopic`, `main_topic` | detailed report 分节 |
| `generate_report_introduction` | :707-716 | `research_summary`, `question`, `report_format`, 日期, `language` | `write_report_introduction` |
| `generate_report_conclusion` | :720-753 | `query`, `report_content`, `report_format`, `language` | `write_conclusion` |

调度：`report_type_mapping`（:852-859）把 report type 映射到方法名；`get_prompt_by_report_type()`（:862-876）用 `getattr` 取方法，取不到就 warning 并回退 research report。**注意** `plan_research_outline()` 调 `generate_sub_queries()` 时没有转发 `prompt_family` 参数（`actions/query_processing.py:204-213`），因此查询规划始终使用默认 `PromptFamily`；其他报告路径则使用配置选中的 family（`actions/report_generation.py:224,250`）。

### 3.2 查询规划（关键模板全文）

```python
# prompts.py:213-259（结构保留，f-string 占位符原样）
return f"""Write {max_iterations} search queries to research the following task: "{task}"

Each query must be a plain natural language phrase. Do not use search operator syntax
such as site:, filetype:, inurl:, intitle:, OR, AND, or NOT — these operators are
not universally supported and will return empty results on many search backends.

Assume the current date is {datetime.now(timezone.utc).strftime('%B %d, %Y')} if required.

{context_prompt}
You must respond with a list of strings in the following format: [{dynamic_example}].
The response should contain ONLY the list.
"""
```

要点：

- `task` 在 detailed/subtopic 报告中是 `f"{parent_query} - {question}"`，否则就是 `question`（:231-237）；
- `context_prompt` 只有在初始搜索结果非空时才生成（:239-244），内容为“你是资深研究助理…使用实时网页信息细化查询”；
- `dynamic_example = ", ".join([f'"query {i+1}"' for i in range(max_iterations)])`（:246），即要求模型返回 `["query 1", "query 2", "query 3"]`；
- 模型输出由 `_normalize_sub_queries()`（`actions/query_processing.py:12-40`）用 `json_repair.loads` 解析，兼容 dict/list/裸字符串，最后兜底原查询。

### 3.3 大纲规划（subtopics）

```text
# prompts.py:573-593（占位符由 LangChain 输出解析器注入）
Provided the main topic:

{task}

and research data:

{data}

- Construct a list of subtopics which indicate the headers of a report document to be generated on the task.
- These are a possible list of subtopics : {subtopics}.
- There should NOT be any duplicate subtopics.
- Limit the number of subtopics to a maximum of {max_subtopics}
- Finally order the subtopics by their tasks, in a relevant and meaningful order which is presentable in a detailed report

"IMPORTANT!":
- Every subtopic MUST be relevant to the main topic and provided research data ONLY!

{format_instructions}
```

`{format_instructions}` 由调用方的输出解析器提供（不是本仓库硬编码的字符串）；`max_subtopics` 默认来自 `DEFAULT_CONFIG["MAX_SUBTOPICS"]=3`（`config/variables/default.py:31`）。

### 3.4 段落起草（subtopic report，关键约束段）

```text
# prompts.py:608-670（摘录核心结构；占位符原样）
Context:
"{context}"

Main Topic and Subtopic:
Using the latest information available, construct a detailed report on the subtopic: {current_subtopic} under the main topic: {main_topic}.
You must limit the number of subsections to a maximum of {max_subsections}.

Content Focus:
- The report should focus on answering the question, be well-structured, informative, in-depth, and include facts and numbers if available.
- Use markdown syntax and follow the {report_format.upper()} format.
...
IMPORTANT:Content and Sections Uniqueness:
- Carefully review the existing headers and existing written contents provided below before writing any new subsections.
- Prevent any content that is already covered in the existing written contents.
- Do not use any of the existing headers as the new subsection headers.
...
"Existing Subtopic Reports":
    {existing_headers}
    {relevant_written_contents}
"Structure and Formatting":
- As this sub-report will be part of a larger report, include only the main body divided into suitable subtopics without any introduction or conclusion section.
- You MUST include markdown hyperlinks to relevant source URLs wherever referenced in the report ...
- Use H2 for the main subtopic header (##) and H3 for subsections (###).
...
- The report should have a minimum length of {total_words} words.
- Use an {tone.value} tone throughout the report.
Do NOT add a conclusion section.
```

去重机制是**双保险**：Prompt 要求模型避开既有标题与内容，代码侧 `ContextManager.get_similar_written_contents_by_draft_section_titles()`（`context_manager.py:89-130`）又用“当前 subtopic + 所有 draft section titles”并发检索已写章节，去重后最多传 10 条给这个占位符。`generate_draft_titles_prompt()`（:673-704）则要求 H3 标题、不含 introduction/conclusion/reference 段。

### 3.5 最终报告（research report）

```text
# prompts.py:293-316（reference_prompt / tone_prompt 为条件片段）
Information: "{context}"
---
Using the above information, answer the following query or task: "{question}" in a detailed report --
The report should focus on the answer to the query, should be well structured, informative,
in-depth, and comprehensive, with facts and numbers if available and at least {total_words} words.
...
- You MUST determine your own concrete and valid opinion based on the given information. Do NOT defer to general and meaningless conclusions.
- You MUST write the report with markdown syntax and {report_format} format.
- Structure your report with clear markdown headers: use # for the main title, ## for major sections, and ### for subsections.
- Use markdown tables when presenting structured data or comparisons to enhance readability.
- You MUST NOT include a table of contents, but DO include proper markdown headers (# ## ###) to structure your report clearly.
- Use in-text citation references in {report_format} format and make it with markdown hyperlink placed at the end of the sentence or paragraph that references them like this: ([in-text citation](url)).
- Every substantive claim, figure or quote MUST carry an in-text citation to the source it came from. Do NOT cite sources that do not appear in the provided information.
- {reference_prompt}
- {tone_prompt}
You MUST write the report in the following language: {language}.
Assume that the current date is {date.today()}.
```

`reference_prompt` 在 web 来源下（:279-285）要求结尾列出去重 URL、每条 `[url website](url)` 并在正文引用处加超链接；本地文档来源下（:287-289）改为列出 source document names。`tone_prompt` 只有 `tone` 非空才出现（:291）。

**防幻觉边界**：这是 Prompt 层的指令，**没有**逐句事实验证器。运行时唯一的硬阻断在 `skills/writer.py:79-88`：上下文为空时直接返回“无法生成可靠报告”的拒答文本，不调用 LLM。

调用参数（`actions/report_generation.py:291-306`）：`model=cfg.smart_llm_model`、`messages=[system=agent_role_prompt, user=content]`、`temperature=0.35`、`stream=True`、`max_tokens=cfg.smart_token_limit`；若带 system 的调用失败，会用单条 user 消息重试（:307-325）。

### 3.6 深度总结（deep research）

```text
# prompts.py:455-487（核心）
Using the following hierarchically researched information and citations:

"{context}"

Write a comprehensive research report answering the query: "{question}"

The report should:
1. Synthesize information from multiple levels of research depth
2. Integrate findings from various research branches
3. Present a coherent narrative that builds from foundational to advanced insights
4. Maintain proper citation of sources throughout
5. Be well-structured with clear sections and subsections
6. Have a minimum length of {total_words} words
7. Follow {report_format} format with markdown syntax
8. Use markdown tables, lists and other formatting features when presenting comparative data, statistics, or structured information

Additional requirements:
- Prioritize insights that emerged from deeper levels of research
- Highlight connections between different research branches
...
- You MUST determine your own concrete and valid opinion based on the given information. Do NOT defer to general and meaningless conclusions.
...
- Write in {language}
{reference_prompt}
...
Assume the current date is {datetime.now(timezone.utc).strftime('%B %d, %Y')}.
```

### 3.7 其他关键模板全文/要点

**`curate_sources`（`prompts.py:319-350`）**：要求“只返回与原始 sources 相同形状的 JSON 列表，禁止 markdown 代码块或额外文本”（:348-349）；评估维度为 Relevance / Credibility / Currency / Objectivity / Quantitative Value（:328-334）；明确“不要重写、摘要或压缩任何来源内容”（:340-343）。`CURATE_SOURCES` 默认关闭（`config/variables/default.py:18`）。

**`generate_summary_prompt`（:525-529）**：

```text
{data}
 Using the above text, summarize it based on the following task or query: "{query}".
 If the query cannot be answered using the text, YOU MUST summarize the text in short.
 Include all factual information such as numbers, stats, quotes, etc if available.
```

**`generate_quick_summary_prompt`（:540-552）**：只允许基于提供的搜索结果回答，要求单段连续叙述、用 `[1] [2]` 编号引用、信息不足时明确说明。

**`generate_mcp_tool_selection_prompt`（:54-83）**：输出严格 JSON `{"selected_tools":[{"index","name","relevance_score","reason"}],"selection_reasoning":...}`，要求**恰好** `{max_tools}` 个工具；`max_tools` 默认 3。

**`generate_image_analysis_prompt`（:142-175）**：每节正文只取前 500 字（:137-140），输出 JSON suggestions（section_number/section_header/image_prompt/image_type/reason），明确排除纯文字分析、引言与结论（:157）。

**`generate_report_introduction`（:708-716）**：要求 H1 起始、只写引言不写其他章节、句末 `([in-text citation](url))` 格式。

**`generate_report_conclusion`（:732-751）**：2–3 段、recap/重要发现/影响或下一步；若报告末尾没有 `## Conclusion` 标题则自行补上（:745）。

**`generate_custom_report_prompt`（:396）**：最简模板 `f'"{context}"\n\n{query_prompt}'`——用户自定义 prompt 完全接管结构，内置引用规则不再注入。

### 3.8 图像与上下文渲染的 Prompt 特化

Granite 模板不是“另一套报告提示词”，而是**同一上下文的不同序列化方式**（`prompts.py:776-834`）。因此切换 `PROMPT_FAMILY` 会改变 `pretty_print_docs()` 输出结构，但 `generate_report_prompt()` 的正文指令保持不变（除覆盖方法的类外）。扩展新模型家族时，实现契约是“保持方法名不变，可覆盖单个方法”（:14-29）。

---

## 4. 维护与排错要点

| 现象 | 代码根因 | 处理 |
|---|---|---|
| 报告上下文里出现 `Source: None` | `d.metadata.get('source')` 为空字符串或 None；上游 page 既无 source 也无 url（`context/retriever.py:32`） | 检查 scraper 结果是否保留 `url` |
| 明明配置了 embeddings 却没调用 | 小内容短路（`select.py:74-77`）或 Jev 不可用前先走 Jev | 看 `COMPRESSION_THRESHOLD` 与 `CONTEXT_FILTER` |
| 上下文里只有开头段落 | BM25 无词面重叠时退回前 N 块（`lexical.py:99-101`） | 改写 sub-query 或改用 Jev/embeddings |
| 报告截断在半句 | `SMART_TOKEN_LIMIT` 太低 | 按 `.env.example:76-92` 提高；注意 reasoning token 也占额度 |
| 引用 URL 未被正文使用 | `add_references()` 直接追加全部 visited_urls（`markdown_processing.py:121-127`） | 这是设计行为；需要精确引用请在写作 Prompt 约束 |
| 换模型后上下文格式变了 | Granite family 覆盖 `pretty_print_docs` | 检查 `prompt_family_mapping`（`prompts.py:879-886`） |
| Jev 失败但研究继续 | `select.py:83-84` 降级 BM25 | 看 warning；成本与选择质量会变化 |

维护不变量：新增过滤模式必须同时改 `CONTEXT_FILTERS`（`select.py:29`）、`resolve_context_filter()`（:38-46）、`.env.example` 与本文档；新增 Prompt 方法必须保留方法名并在 `report_type_mapping`（`prompts.py:852-859`）登记，否则会触发 warning 并回退默认报告 Prompt。
