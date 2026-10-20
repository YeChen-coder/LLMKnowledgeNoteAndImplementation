以前那套是文档切块、embedding、vector database，然后选 top k（k 选 5 还是 10 都行，无所谓），最后送到 language model。

现在呢，进化了。

那就是核心从 vector search 变成了Hybrid Retrieve → Rerank → Context Engineering → Generate → Evaluate。


现在微软、AWS、Google、Databricks 的生产建议已经高度趋同：Hybrid Search、metadata filtering、reranking、结构化 chunk、evaluation 是主干；复杂问题才升级到 agentic retrieval。- 对不起啊，吐槽一下。它既然都已经是这么一个 super well-defined problem 了，而且云服务商、头部 tech 公司都已经出了这种 tutorial，那 GitHub 上好用、成熟的开源框架都不知道有多少了，直接用人家的呗，折腾什么呢？这个东西根本就已经发展到标准化程度了，不是什么需要从头写的东西。用人家的框架，理解一下人家是干嘛的，再去改改不就行了吗？用到什么学什么嘛，真的没必要造轮子，所有事情找最简单的办法，grow with SOTA 就可以了。

# https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-information-retrieval?utm_source=chatgpt.com

                2026 Production RAG / 企业级 RAG

──────────── OFFLINE / INGESTION 离线知识处理 ────────────

PDF / Word / HTML / DB / Wiki / Tickets
                │
                ▼
      Layout-aware Parsing
      版面/结构感知解析
      标题、段落、表格、图片、页码
                │
                ▼
       Clean + Deduplicate
         清洗 + 去重
                │
                ▼
   Structure-aware Chunking
        结构化切块
                │
                ▼
     Chunk Enrichment
 Chunk 补充上下文 / Metadata
 title / section / date / ACL / version
                │
        ┌───────┴────────┐
        ▼                ▼
 Dense Embedding       Lexical Index
   向量索引           BM25 / 倒排索引
        │                │
        └───────┬────────┘
                ▼
          Search Index
      企业知识检索层


──────────── ONLINE / QUERY 在线问答 ───────────────────

User Question
用户问题
     │
     ▼
Auth / ACL / Tenant
身份、权限、租户
     │
     ▼
Query Understanding
问题理解
     │
     ├── Rewrite / 改写
     ├── Filter / 提取条件
     ├── Decompose / 拆问题
     └── Route / 选知识源
     │
     ▼
┌──────────── Hybrid Retrieval ──────────────┐
│                                             │
│   Dense Vector Search     BM25 Search       │
│     语义搜索                关键词搜索       │
│          │                    │             │
└──────────┴─────────┬──────────┘
                     ▼
              RRF / Fusion
                结果融合
                     │
                     ▼
             Top 20~50 chunks
                大范围召回
                     │
                     ▼
                RERANKER
                 精排
                     │
                     ▼
              Top 3~10 chunks
                高质量证据
                     │
                     ▼
     Dedup / Parent Expansion / Context Pack
          去重 / 补上下文 / 整理证据
                     │
                     ▼
                    LLM
                     │
                     ▼
       Answer + Citations + Abstention
         答案 + 引用 + 不确定时拒答
                     │
                     ▼
       Logging / Eval / Observability
            评测 + 监控 + 反馈


## Changes 

就是之前的那种 vector search，就是把所有东西 embedding，然后去 vector DB 搜。那个东西叫 dense retrieval / semantic retrieval

但是这样是有问题的。比如说，用户如果只搜索一个代号，而这个代号根本没有语义，这个时候其实应该用传统搜索，也就是 BM25（Lexical /ˈleksɪkl/ Search）。

既然现在两种都用，所以拿到这两种不同的 search 出来的结果，然后去进行融合的时候这个东西就叫 Reciprocal  /rɪˈsɪprəkl/ Rank Fusion，

             Query
               │
       ┌───────┴───────┐
       ▼               ▼
Vector Search        BM25
语义搜索             精确词搜索
       │               │
       └───────┬───────┘
               ▼
              RRF
       Reciprocal Rank Fusion
             排名融合
               │
               ▼
          Candidates
