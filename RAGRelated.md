Top K 是通用的术语，就是 Top 5、Top 10 那种

可是 top A 和 top B 它们不是那种通用的东西。它们一般只是论文作者用 A、B 作为两个不同的数量参数，比如 A 等于 30、B 等于 50 之类的，并不是一个 universal 的 general item。

在搜索过程中，不只保留当前最好的一个结果，而是同时保存当前最有希望的 B 条路径， 这个是为了不仅知道最后拿到的是什么样的结果，而且还知道这一路上是怎么过来、怎么拿到这个结果的，也就是这个 reasoning/retrieval path。这个是 Beam Search（束搜索）里面的 Top-B paths。这个是知识图谱那边的概念。 -说实话我没看懂。我只直到跟图有关的，但真的好长、好学术啊。等项目复杂度或者要求真需要它的时候再来找它吧，这不是我现在要做的事情。

# RAG 进化了

以前那套是文档切块、embedding、vector database，然后选 top k（k 选 5 还是 10 都行，无所谓），最后送到 language model。

现在进化了, 核心从 vector search 变成了Hybrid Retrieve → Rerank → Context Engineering → Generate → Evaluate。

这边补充一下，RAG 的定义是 retrieval system，它的范围远远不止 vector DB 这么点事情。只要带 retrieval 的，别管你是用 API 还是用 SQL，它都叫 RAG - 哦，我懂了，那我明白为什么面试的时候问 RAG 了。我当时想了半天，我对 RAG 的理解只有 embedding 和 vector DB 那边。我想着这能有多大的资料量，至于搞 RAG 吗？它有多大的资料量，值得在一个打电话的 AI agent system 上搞这个东西？可是如果说它的定义本来就包括了 SQL 和 API 这种数据，是我之前理解错了，那确实得搞 RAG，那我挂得没毛病。

<img width="338" height="332" alt="image" src="https://github.com/user-attachments/assets/06da8b5b-bb15-43ad-a87f-8286f58513de" />

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

### Hybrid = ( dense retrieval / semantic retrieval +  BM25 / Lexical Search ）-> Reciprocal Rank Fusion

就是之前的只用那种 vector search，就是把所有东西 embedding，然后去 vector DB 搜。那个东西叫 dense retrieval / semantic retrieval

但是这样是有问题的。比如说，用户如果只搜索一个代号，而这个代号根本没有语义，这个时候其实应该用传统搜索，也就是 BM25（Lexical /ˈleksɪkl/ Search）。

既然现在两种都用，所以拿到这两种不同的 search 出来的结果，然后去进行融合的时候这个东西就叫 Reciprocal  /rɪˈsɪprəkl/ Rank Fusion。 -这个没怎么变，大概一两年前大家就都在用 hybrid 了。当时 ChatGPT 就直接建议我用 hybrid，所以其实这个理念到现在确实也没有变化。

<img width="316" height="400" alt="image" src="https://github.com/user-attachments/assets/bcfd82e7-b6d0-4b7c-a63d-ca07bac32769" />

### Reranking = 50 candidates -> Reranker -> 5 really useful chunks ; Multi-stage Retrieval / 多阶段检索

这个不太一样，又加了一步。

很久之前比较原始的那种做法，就是直接取 Top 5 或者 Top 10，然后拿着这些 chunk 直接走最后一步，喂给最后的 LLM。

但现在不是了，而是在中间又加了一步：

1. 第一步拿到的东西更多：不是只拿 5 个或 10 个，为了保证不遗漏可能正确的答案，第一步直接拿了 50 个。
   
2. 中间加了一步筛选：问题是这 50 个 chunk 不能全喂给最后的 LLM，所以中间加了一步，拿用户的 query 跟检索到的这些 chunk 做比较，重新筛选出 Top 5，然后再送过去

<img width="826" height="235" alt="image" src="https://github.com/user-attachments/assets/912ac77d-6cb3-4350-9f9e-3a5087e1846e" />


Multi-stage Retrieval / 多阶段检索 = Cheap + Fast retrieval 便宜、快、大范围找 -> Expensive + Accurate reranking 贵一点，但只处理几十个

### Chunking： Layout-aware / Structure-aware Parsing 版面感知 / 文档结构感知解析

其实很久之前这个 chunk 的问题，早没人用固定长度了。用固定长度肯定会出事：句子截一半你怎么搞啊？所以这个其实没有多大可改的。之前都是什么 chunk size 等于多少多少、chunk overlap 等于多少多少，现在早没人这么干了。

现在首先强调的是：Layout-aware / Structure-aware Parsing 版面感知 / 文档结构感知解析

拿最麻烦的 PDF 来说：之前最粗糙的早期版本，就是直接 PDF 转 text；而现在的做法，是把 PDF 里的 title、heading、paragraph、table、list、image、footnote、page number 等，全都做成 structured representation。

Google 明确建议用于 RAG 的 PDF/DOCX/PPTX/XLSX 启用 layout parser/'pɑ:sə/；Microsoft 也把 document structure 和 semantic/structure-aware chunking 放在当前 RAG 设计中。

然后就是，他这么搞下来的话，那一块chunk就会保留 一坨文字 + 它是谁 + 它从哪来 + 它属于谁 + 谁能看它。
<img width="822" height="428" alt="image" src="https://github.com/user-attachments/assets/3cec692f-2c4a-4870-92ca-3371a272759b" />

还有一件事，就是 parent-child 的思想，这也是个比较公认且成熟的方案。

比如说一篇文档它是 parent，里面的小段落就是 child。搜索的时候一般都搜 child，因为小 chunk 更容易形成精准匹配。现在的做法是，把这个小 chunk 匹配出来之后，不只是把它直接喂给大模型，而且它会找到上一级的 parent，拿到附近的上下文，再去送给大模型。

这个好做：上一段 chunk 在切的时候，已经保留了 page number、paragraph 这些元数据，往上找往下拿很好摸，就是一个双向链表的事嘛

也是因为这个，它现在的一个思路，里面的一个 principle 就是：小块的 chunk 用于精准匹配，因为好匹配、容易找到，similarity 表现会更好一点。但光找到它还不够，既然能匹配到，说明这个 parent 一定有可取之处，所以还得把 parent 这个稍微大一点的块拿出来，方便让 model 进一步理解。 Azure 目前甚至直接把 chunk ↔ parent 的关系作为主流 RAG indexing pattern。

### Metadata Filter； Filter first / Filter during retrieval

这个主要是因为一些 authentication，它额外加了一道门。

但讲道理，如果前期在 authentication 的时候就已经把 permission 分好，这步根本就不需要。不过他乐意加就加吧。

它的主要用途就是对发起query的用户，不该看到他不该看到 confidential

AWS 当前的企业安全指导明确要求 ingestion、storage、retrieval、inference 多层防护，并在 retrieval 阶段做 metadata filtering / RBAC。Azure 也支持 query-time ACL/RBAC 这类权限检索。-这样就对嘛！直接一开始就把 RBAC 给确定好啊。怎么会有人一开始没有把 RBAC 做好，让 chunk 里面已经包含了用户不该知道的信息，然后再跑去 filter 呀？怎么会有这么反逻辑的事情呢？

### Contextual Embedding； ingestion enrichment

这个跟前面说的切 chunk 时保存元数据作为 chunking 的一部分，其实在很大程度上是一样的，但这样比那样更好一点。

因为之前的那种方式，它能有的信息仍然是有限的。而真正的 contextual chunking，能让模型根据整篇文档来理解这个 chunk，并结合它的上下文信息（比如指代明确 this company 到底意味着什么、这是哪一年的报表等），生成一小段说明，讲清楚这个 chunk 在原文中到底是什么。

虽然计算量肯定会大很多，但整体效果绝对会更好。不过考虑到一般来说，一个文档做 chunk，它一共也就切这么一回，一共也就做那么一回 embedding，所以其实这种一次性的事情也是值得的。

总之，这边的流程就是先切 chunk，然后做 enrichment，给它补一些上下文信息或者**整个**文档的说明，最后把 enrich 过的 chunk 送去 embedding 嘛。

### Query Rewrite / Query Decomposition /ˌdiːˌkɑːmpəˈzɪʃn/ 分解

正如常说的那句老话：用户不知道他们想要什么。

如果直接把用户的 query 拿去 embedding 然后检索，效果确实很费劲。所以现在会先处理一下、改写一下。比如用户问"A 和 B 有什么区别"，直接在文档里找 A 和 B 的对比其实挺难的。按人的思维来说，肯定得先知道 A 是什么、B 是什么，拿到这两部分数据之后再去谈对比。

系统侧的处理逻辑也是这样：

1. 先拆成 subquery，分别去看 A 是怎么回事、B 又是怎么回事
2. 各自去做 retrieval，拿到 evidence A 和 evidence B
3. 最后再去做 compare

Microsoft 当前 RAG Architecture Guide 已经把 query augmentation、rewrite、decomposition、multiple searches、reranking 放进一套完整 retrieval pipeline。

现代 RAG 的逻辑，就是拿到用户的 query（也就是 question）之后，先做一个 research planner，然后再去到 search engine。

而不是直接拿 query 做 embedding，然后拿着 embedding 开始哐哐搜，在那之前加了这么一个思考、分拆问题的过程

### Agentic RAG 用于复杂问题，如multi-step reasoning

Agentic RAG 就是在里面加了一个 loop，引入了 LLM-as-a-Judge，底层还是那套agent orchestration。

解决的问题是，第一遍出的结果其实不对、不够或者有欠缺。所以现在的逻辑依然是第一步先拿前一套流程跑出一个 research A 的结果；看完结果发现不够，就再去查 B；查完 B 觉得还不行，还差了点东西（比如日期什么的），然后再去查 C。-我总觉得这个东西能出来的一个很重要的结果，就是现在 AI 智力水平的大幅上升。不然搁以前，AI 没有这个水平去评估收集到的信息够不够。

它中间其实就是加了这么一个 loop 环。

微软现在把它定义得非常清楚：适用于 multi-step reasoning、dynamic source selection、query decomposition、iterative refinement 和 retrieval + action。反过来，如果一个问题一次搜索就能解决，微软直接建议：用标准 RAG，别为了 Agentic 而 Agentic。

### GraphRAG 适合 relationship-heavy / multi-hop questions

这个主要是用于网状的 network 关系，像一对多、多对多的，反正大家互相都得连着。

比如说公司的供应商怎么样、公司的上游变了会怎么样、下游变了又会怎么样，就是这么一个网状的图结构。

GraphRAG 并没有成为所有企业 RAG 的默认底座。

更有意思的是，Microsoft 自己的 GraphRAG 官方 GitHub 现在已经明确写了：这是一个 research project，目前 largely in maintenance mode，不会继续当成快速扩展的新平台推进。

这个的发展一般，确实挺一般，反正未来状态存疑，未来发展前景存疑。 - 我想了一下，这个是不是因为在具体场景中，大家一般都是给某一个具体公司做这种 RAG 的？上下游的关系真的没有很多，顶多上面两层、下面两层就到头了。这里面还能有什么很复杂的关系吗？一般公司也不会这么错综复杂吧。所以它其实没有那么大的普适性，没有被大家觉得重要性有那么大，也是挺有道理的吧。

### ColBERT / Late Interaction

这个是真的新的东西。就是用在之前的那个 hybrid 加 reranker 出来的质量仍然不够的时候，就加大资源投资用这个。

之前普通 embedding 就是把一整个 chunk，你别管里面加了什么、怎么处理，别管是做没做 ingestion enrichment，还是说把 metadata 有没有放进去，都无所谓，反正就是把整个 chunk 做成一个 vector。它就可以理解成在向量空间中，这一整个 chunk 的信息都被压成了一个点。

但是比较新的 ColBERT 这个方法，它是把 chunk 的各个 token 各自映射成独立的 vector。去查询的时候也是，把 query 的每个 token 各自去找它们的 vector，然后去做 late interaction。

不只是比较 query 和那一段 chunk 的整体 similarity，而是让 query 的不同部分（也就是不同的 token），分别去找到 document 中最匹配的部分，那不用想，肯定更精准，绝对能找到非常精准的东西。

可是它的 trade-off 就是 storage 和 compute 都明显更高：

1. Compute 方面：
并不是说只在文档处理阶段耗费一次就好，实际上并不好，每次 query 时的 compute 都是一样的高。因为得拿 query 的每个 token，去跟 chunk 里的每个 token 的 vector 逐一对齐，这不是一次性的开销，计算资源消耗确实很大。

2. Storage 方面：
原来一个 chunk 只对应一个 vector、一串 string；但现在 chunk 的每个 token 都对应自己独立的 vector，那你存储的量确实是有多少 token 就会乘上多少倍，膨胀得非常厉害

Weaviate 的 multi-vector / ColBERT 类检索目前已经进入 production-ready/GA 支持，而不是纯论文玩具。

### SPLADE

这个也是新的东西。之前那个 BM25（BM = Best Match / Best Matching） 是用户搜什么词就找什么词，纯粹是纯搜。但现在这个 SPLADE，它是处于纯关键词的 BM25 和 Dense Embedding 之间的 

Qdrant 等现在已经正式支持 SPLADE 这种 learned sparse retrieval。

但现实里：BM25 + Dense + Reranker 仍然是更保守、更成熟的默认配置。

SPLADE 是你做 retrieval tuning 时值得试的升级项。



<img width="818" height="538" alt="image" src="https://github.com/user-attachments/assets/91f0a1a3-ea78-42d7-9f3c-2f6017212f55" />

稍微补充一下：BM25 的意思就是，当用户搜索（比如搜索 cybersecurity）时，它会看 query 里的这个词在文档中有没有出现、出现过几次、这个词稀不稀有，以及文档是不是特别长，以此来判定当前这个 chunk 和用户 query 的相关程度。 主要考虑：
TF = Term Frequency 这个词在文档出现多少次

IDF = Inverse Document Frequency 这个词有多稀有 （-就是看这个词有多稀有，它是去用于解决类似于query里， the 这种到处都有的问题的； 对于 Query 里只有少数文档有的词，它就是很有辨识度，然后分数很高嘛）； 而且并不是单纯出现 10 次的分数就比出现 1 次高 10 倍，他们是有做处理的。大概是用了一个指数函数吧，反正它存在一个瓶颈，差不多到了某一个点之后，加分就会变得非常非常平滑， 即为 BM25 会做 Term Frequency Saturation / 词频饱和。

Document Length 文档长度

SPLADE 就卡在中间：它用神经网络理解意思，但最终仍然用“词”来表示结果。这句话得解释一下哈：他用了某种办法把具有同种语义的词做了一个语义理解，这就让他拥有了识别能力，知道 car、automobile 和 vehicle 讲的是同一样东西。可是他在进行表示的时候，并不是用 dense 的那种方式（有一长串的 vector），而是仍然跟刚才的 BM25 一样，只有一个标量。所以这句话会说它用神经网络理解意思，但最终是用词的数据结构来表示结果。

SPLADE 的全称本身就是：SParse Lexical AnD Expansion

它利用类似 BERT masked-language-model head 的机制，为 vocabulary 中的词学习权重和扩展，然后通过 sparsity regularization 强迫最终表示保持稀疏。- 这句话我真没看懂，但是他解释的就是上文的那个意思。

我看了一下解释，不确定能不能完整准确地表述它的意思，就单纯说一下我的理解吧：

它首先是用某种方式去处理 query 里的 token。原本一个 token 就像一个小点，但因为它做了 expansion，现在不只是一个小点了，外部还会向外扩一圈模糊的范围，让它有能力跟同类词产生很多 overlap。

比如原本 car 和 automobile 是两个独立的小点，但用了 SPLADE 处理之后，这两个小点各自都长出了一圈虚化的范围。这样它们做对比的时候就能有一块 overlap 到，从而增加了找到它的能力

- 就 SPLADE 这个事，先停在这里吧。反正就是知道 SPLADE 比 BM25 更好，BM25 是原始人用的老古董就行了

### Multimodel RAG

还是那个问题嘛：遇到带图、带表格、带扫描页或者带流程图的文档，原本那些只针对纯文本 TXT 的方案就跑不通了。

所以现在有了 multimodal RAG。最简单的情况是直接把整页 PDF 作为一个 image 做 multimodal embedding，直接 retrieve 到整页 page，然后再给到 visual language model

### Evaluation

就是把 evaluation 评估给标准化了嘛，算 recall、算 precision、算 ranking，算这个那个的。

这种对 evaluation 的重新定义和标准化其实挺常见的，只要是一个 AI 系统，大家大差不差都得看类似的结果。

<img width="830" height="565" alt="image" src="https://github.com/user-attachments/assets/76764e79-a866-4909-86d1-88a813817c12" />

AWS 现在也直接提供 correctness、completeness、faithfulness、citation precision、citation coverage 等 RAG 指标。

# 总之，流程上 Parse → Chunk → Enrich → Hybrid Retrieve → Rerank → Pack Context → Generate → Cite → Evaluate （解析 → 切块 → 补信息 → 混合召回 → 精排 → 整理证据 → 生成 → 引用 → 评测）

这个里面的 pack_context，是去把 re-rank 出来的 5 个 chunk 再整理一下。

具体需要做以下几点：

1. 去重与版本处理：比如做 deduplicate（去重），如果其中有一个是来自旧版本的，要做好版本处理。
2. 补上下文：也就是做 parent expansion。有的时候标题也需要补上。
3. 调整顺序：如果其中有两个 chunk 来自同一篇文章，就把它们重新排一下放一起；如果是同一章节、内容靠近的，就顺手给排了。
4. 控制 Token 量：控制一下给下面 large language model（大模型）的 token 输入量。
5. 保留它的来源，就是保留这些 evidence 的 metadata，比如 document ID、page、section、URL 等等。

这些都做好了，然后才把这个 context package 送到下面的什么 model，然后生成答案和 translation (顺便注意一下：• Context Packing 是发生在送到最后大语言模型（LLM）生成前，目的是让已经搜到的 chunk 更适合给大模型看
• Contextual Embedding 则是发生在检索侧（做 chunk 的时候），目的是让切出来的这份 chunk 更容易被搜到)


                     企业资料
 Enterprise Data Sources
                         │
                         ▼
             Parse / OCR / Layout
              解析 / OCR / 版面
                         │
                         ▼
             Normalize + Dedup
                 清洗 / 去重
                         │
                         ▼
            Structure-aware Chunk
                  结构切块
                         │
                         ▼
       Metadata + Parent + ACL + Version
       元数据 + 父文档 + 权限 + 版本
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
          BM25 Index            Dense Index
        关键词倒排索引           向量索引
              │                     │
              └──────────┬──────────┘
                         │
──────────────── QUERY ──┼──────────────────
                         │
User
 │
 ▼
Authentication
身份
 │
 ▼
Query Router / Planner
问题路由 / 查询规划
 │
 ├── simple → one query
 │
 └── complex → decomposition
                         │
                         ▼
                  ACL + Filters
                    权限过滤
                         │
                         ▼
              ┌──────────┴──────────┐
              ▼                     ▼
             BM25                 Dense
              │                     │
              └──────────┬──────────┘
                         ▼
                        RRF
                         ▼
                   Top 20~50
                         ▼
                      Rerank
                         ▼
                    Top 3~10
                         ▼
       Dedup + Parent Context + Citations
           去重 + 补上下文 + 来源
                         ▼
                        LLM
                         ▼
            Grounded Answer / Refuse
              有依据回答 / 没依据不答
                         ▼
               Logs + Evaluation
                  日志 + 评测


- 写在最后：真要用 RAG 来干活的话，别折腾了，直接用大公司的吧。人家当前的流程框架既然已经经过验证，成了这么成熟的标准，就不要再去自己拼了，走前人已经铺好的路不好吗？它里面具体怎么编排不是开发者需要纠结的东西，只要知道是什么意思、会用就行，清楚哪种方法比哪种方法好，以及它的 trade-off 是什么就可以了。

---

在做这些设计项目，给他配 RAG 用的是 RAGFlow。

我想明白了为什么之前做 RAG 项目的时候，整个过程如同流水一般在我脑子里滑过去了，过程中顺得完完全全没有被折磨到，导致最后一点印象都没有，我都不知道自己干了点什么。

就是因为现在这个框架搞得太好了，人家都已经给你配全了。你所做的事情就是把 model 的 API 填上，选一下 default model 要哪个就完事了，根本不用自己拼，整个流水线都搞好了。那确实留不下什么印象，哎呦

---

不得不说呀，实在是太容易忘了，就真的雁过无痕、水过无声，在脑子里一点印象都没有。距离我写这个笔记才刚过去两天，我现在又给忘了 Rerank 是干啥的了。我光记得它是做一个精排，比如从 50 个里面挑 5 个，但是它具体在技术上怎么做的，我又给忘干净了。

所以这边就基于那个 Jesus 项目，顺手说一下吧。

以"房租"这道题为例：

用户说："I can’t sleep because I keep thinking about how I’ll pay rent next month."
（我一直在想下个月房租怎么付，睡不着。）

正确答案应该是 Matthew 6:34："不要为明天忧虑……"

### 把原quota找个小LLM模型做改写

先看下改写吧。

改写这边是调了一个小模型（我用的是 GPT 5.4 Mini），给它一段指令让它自由写一段检索词。这个没有固定分组，不是一开始就分什么 friends、family 这种，就是写了一段 prompt。

我这边大概是让它把用户的话改写成一句用来搜文档的查询：

指令原文在 retrieval.py 里：

把用户的话改写成一句用来搜四福音书的查询。说出背后的人的需要，以及耶稣谈到过的主题，用现代英文圣经会用的朴素词语，再加几个具体关键词。不要猜章节号。25 个词以内。

这次模型现场写出来的是：

“worry about money and rent, sleepless fear, trust God’s care, daily needs, anxiety, provision”

主要是因为四福音书跟现在的英文口语实在对不上，所以必须得有改写。后来做了 ablation study，反正不管其他的变量怎么变，每次加了改写的都比不加好太多了

注意它做的事：用户嘴里说的是"房租"”睡不着"，这些词经文里没有；改写把它翻译成了经文会用的说法："worry"（忧虑）、"daily needs"（日用所需）、"provision"（供应）。

为什么不用固定分类？ 因为分类是死的。"我对我妈说了难听话"该归"家庭"还是"愧疚"还是"愤怒"？三个都有。自由改写可以同时写出"guilt, hurting family, anger, reconciliation"，不用硬塞进一个格子。

### 向量和语义空间：空间不是人划分的，是模型学出来的

向量：embedding 模型（text-embedding-3-small）把任何一段文字变成 1536 个数字，刚才跑出来的长度就是 1536。

语义空间：你可以把它想象成一张 1536 维的地图，每段文字都是地图上的一个点。这张地图没有人去规定"第 3 维代表家庭、第 7 维代表金钱"，每一维也没有人能叫出名字。模型是在海量文本上训练出来的，学到的唯一规则是：意思相近的文字，位置就靠近。

相似度：两个点有多近，用"余弦相似度"量，越接近 1 越像。

---

我这边先补充一下：其实项目里处于从 0 到 1 的阶段出的结果真的很乱。所以我再简单把流程梳理说明一下。

各种东西为了追求效果叠在一起，让整个事情变得非常 chaotic.

先说一下，之前一直说的那个全文检索，其实就是 BM25;

step 1 生成 487 个段落文件（pericopes.py）

每个 BSB 段落存成一个文本文件，第一行是标题，后面每行一节经文。这是一个真实的文件：

Matthew 11:25-30 · Rest for the Weary

Matthew 11:25 At that time Jesus declared, "I praise You, Father, ...

Matthew 11:26 Yes, Father, for this was well-pleasing in Your sight.
...

Matthew 11:28 Come to Me, all you who are weary and burdened, and I will give you rest.

Matthew 11:29 Take My yoke upon you and learn from Me, ...

Matthew 11:30 For My yoke is easy and My burden is light."

统计一下这 487 个段落，一个段落一般就是一个小故事或一段讲话：

| | 最短 | 中位数 | 平均 | 最长 |
|---|---|---|---|---|
| 节数 | 2 节 | **6 节** | 7.7 节 | 37 节 |
| 英文词数 | 26 词 | **135 词** | | 808 词 |

135 个英文词，大概就是**中文一小段话的长度**，读出来不到一分钟。按长度分布：

| 每段节数 | 段落数 |
|---|---|
| 1–3 节 | 86 |
| 4–6 节 | 166 |
| 7–10 节 | 115 |
| 11–20 节 | 108 |
| 21 节以上 | 12 |

超过 20 节的只有 12 段，都是很长的对话或讲论。最长的三段是"生命的粮"（John 6，37 节）、"文士和法利赛人有祸了"（Matthew 23）、"撒玛利亚妇人"（John 4）。

"劳苦的人得安息"这个段落一共 6 节，就是刚才看到的那个文件：

```
Matthew 11:25-30 · Rest for the Weary            ← 标题行（也算一个子块）
Matthew 11:25 At that time Jesus declared, ...   ← 子块
Matthew 11:26 Yes, Father, ...                   ← 子块
Matthew 11:27 All things have been entrusted ... ← 子块
Matthew 11:28 Come to Me, all you who are weary  ← 子块
Matthew 11:29 Take My yoke upon you ...          ← 子块
Matthew 11:30 For My yoke is easy ...            ← 子块
└──────────────── 整个文件 = 一个父块 ────────────────┘
```

- **子块**就是文件里的**一行**：一节经文，或者标题行。检索时用子块来匹配，因为一行意思集中，好对上。
- **父块**就是**整个文件**，也就是整个段落。某一行被命中后，交出去的是整个段落，这样模型能看到上下文：比如 11:28 前面的 11:25 是耶稣在祷告，后面的 11:29 讲"负我的轭"。

项目里，"块"其实很直观：**子块 = 一节经文，父块 = 一个带小标题的段落。**

# Jesus 整体流程

先讲整体的大流程，细节后面再补充。

整个流程从用户输入一句话（原文）开始：

1. 改写输入
用一个小的生成式 model，配合刚才提到的 prompt，对用户的原始输入进行改写。

2. 检索段落（RAG flow）
这一步分为两个小阶段：
(a) 初筛出 30 个 candidate。
(b) 进行 rerank，最终返回 3 个结果及其 rerank score（with their rerank score）。要求 score 必须高于 0.3，不高于 0.3 的段落直接丢弃。

3. 匹配耶稣原话
根据段落 ID 到知识库中查询这 3 个段落对应的耶稣原话。比如找到了 15 句，但因为之前已经做过一轮筛选，需要检查这 15 句是否在预设的 278 句范围内：在的话保留，不在就直接剔除。经过这一步后，假设只剩下 5 句候选。

4. 候选排序并取 Top 3
在本地花几毫秒单纯比对向量相似度，对剩下的 5 句候选排个序，取前 3 句交给模型。

补充说明：为什么第 4 步还要拿改写后的原话去做向量相似度排序？
虽然当前例子里只有 5 句，全塞进一条 memory 给模型也能处理，但常规情况下候选句可能多达 20 几句，直接全塞进 memory 会给模型造成负担。所以加了这一步相似度比对并排序，筛选出 Top 3。反正就是得把它的输出量给标准化嘛。

因为之前去挑、去 match，谁知道它会出来一个什么结果呢？少了无所谓，但万一很多，你让模型怎么办呢？

它那个 real-time 的 context window 也得控制啊。你 context 炸了，时延就长了；时延一长，沉浸感就差很多了.

补充一下，第四步这边的重新排序其实迭代了好几版：

第一版原本是用 cross-encoder 的 model 来调，后来又找了一个小模型来看它们是不是有关系。它本来是一个保全策略，但太慢了，得花一秒，体验很不好。

现在用的是本地的向量排序，就是为了让它能快速开口，目前几毫秒就可以搞定。

如果换成 Voyage 重排就是来算当前筛出来的这几句话单独是不是跟用户谈的这件事情有关系，claude code 估了一下大概要 150 多毫秒。这在效果上确实可以改进，但 trade-off 就是要多花大概 100 多毫秒

5. 组装 Memory 并由模型输出
将排名前 3 的内容作为 memory 输入给模型。Prompt 中要求模型在进入第二阶段回复时，最多挑一句并逐字说出来，不用自己发挥。因为这里要求完全还原原文，所以会通过相似度比对来作为一个 evaluation 指标。

如果跑完这么一套流程，实在是连一句耶稣原话都没有，那也行，就什么都不给。

这个会在最后 real time 那边的 prompt 里写明，直接不引用了.

<img width="945" height="186" alt="image" src="https://github.com/user-attachments/assets/d39402f0-6075-4029-9129-0d8ffab8d951" />

上面这张图是拿那 55 个 ground truths 跑出来的结果分析。

针对那 14 道题，这其实不是检索的问题，RAGFlow 那边找的段落是对的。但因为段落里对应的耶稣原话不在那 278 句里，所以出了问题。

补充说明一下那 278 句的筛选规则：核心是看“能不能原样对一个寻求安慰的人说”。原文里很多耶稣原话确实没法直接说，比如“要爱你们的仇敌”、“要饶恕他 70 个 7 次”，像这类的就被过滤掉了。其实这个也是我的原因。我确实不太乐意把它搞成一个……就是我真的不太乐意让它听起来像是说教的。

我确实对宗教不是很了解，但我对“人机恋”这个群体还是很了解的。反正我在人机恋的圈子里，就看到大家跟不同的模型会有一些情感上的连接。经常会有说换了模型、或者模型降智、又或者被暗改路由到不知道那里去了，反正各种各样的事情，导致用户被模型“创”到。

这样不好，就是不好。所以我这边就尽可能让它走一个更加温和的路线了

以及，再怎么样也得考虑一下 scenario：我们这个 scenario 一开始就是冲着 voice companion 来的，它毕竟不是一个真的 Jesus。

不过这个无所谓，都很好改：把那 278 句扩充一下，再把筛选规则放宽一点就行。这些都能调整，不是什么大问题。

最终模型生成回复并呈现给用户，这就是完整的大致流程

### 0.导入。生成段落文件+四道工序给RAGFlow

先看代码，按实际实现来讲，不凭印象：

再拿一个真实的段落文件，并把一次检索的分数逐项打印出来：

先纠正一个误会：**你其实一直在用 BM25。** 阶段 1 那个我手写的 BM25 已经不用了，但 RAGFlow 的"全文检索"本身就是 BM25 类的关键词打分，由 Infinity 内置。它在混合检索里占 0.3 的权重。

下面分成"导入时做了什么"和"每次查询时做了什么"两部分。

---

## 一、导入时（离线，只做一次）

**1. 生成 487 个段落文件**（`pericopes.py`）

每个 BSB 段落存成一个文本文件，第一行是标题，后面每行一节经文。这是一个真实的文件：

```
Matthew 11:25-30 · Rest for the Weary
Matthew 11:25 At that time Jesus declared, "I praise You, Father, ...
Matthew 11:26 Yes, Father, for this was well-pleasing in Your sight.
...
Matthew 11:28 Come to Me, all you who are weary and burdened, and I will give you rest.
Matthew 11:29 Take My yoke upon you and learn from Me, ...
Matthew 11:30 For My yoke is easy and My burden is light."
```

**2. 上传给 RAGFlow，它按 4 道工序处理**（参数在 `ingest_ragflow.py` 里）

| 工序 | 做什么 | 你的设置 |
|---|---|---|
| Parser | 读文件 | 普通文本 |
| GeneralChunker | 切块 | **整个文件是一个父块**；**按换行切出子块**，每节经文一个子块 |
| Extractor | 调 LLM 给每个块加内容 | 每块生成 **5 个关键词**和 **3 个"可能会被问到的问题"** |
| Tokenizer + Embedding | 建两种索引 | **全文索引**（把词拆开、记下每个词出现在哪里、有多常见）；**向量索引**（`text-embedding-3-small` 生成 1536 维向量） |

以 Matthew 11:28 这个子块为例，它在 Infinity 里大概存着这些：

- 经文原文："Come to Me, all you who are weary and burdened…"
- LLM 生成的关键词，类似：weary, burdened, rest, invitation, comfort
- LLM 生成的问题，类似："Where can I find rest when I'm exhausted?"
- 一个 1536 维的向量
- 它属于哪个父块（"Rest for the Weary"这个文件）

（具体是哪 5 个关键词、哪 3 个问题，是当时 LLM 生成的，上面只是示意。）

**全文检索为什么这次有用？** 阶段 1 的 BM25 只能匹配经文原文的字，而"burned out"这种词经文里根本没有。现在每个块多了 LLM 写的关键词和问题，里面有很多现代说法，关键词匹配就有东西可对了。

---

## 二、每次查询时（以"我累垮了"为例）

```
改写后的查询："Jesus rest for the weary, burden, heavy heart, tired and burned out, ..."
        │
        ▼
① 在 Infinity 里同时做两路检索，对象是所有子块
   ├─ 全文（BM25）：查询里的词和块里的词（原文+关键词+问题）重合了多少，越少见的词越值钱
   └─ 向量：查询向量和块向量在语义空间里有多近
        │
        ▼
② 合并：总分 = 0.3 × 全文分 + 0.7 × 向量分，取前 30 个候选
        │
        ▼
③ Voyage 重排：把查询和这 30 个候选逐个"一起读"，重新打分、重新排序
        │
        ▼
④ 每个段落只留一个（多节经文都命中同一段时，不重复算），取前 3
        │
        ▼
⑤ 拒答判断：第一名的分数 < 0.30 → 一个都不返回
        │
        ▼
⑥ 子块命中 → 返回它所属的整个段落（父块）
```

刚才实际跑出来的前 5 名，对比加不加重排：

| 排名 | 只用混合检索 | 分数 | 加上重排 | 分数 |
|---|---|---|---|---|
| 1 | 劳苦的人得安息 Matthew 11:25 | 0.767 | 劳苦的人得安息 Matthew 11:25 | 0.519 |
| 2 | 差遣七十二人 Luke 10:1 | 0.756 | 我留下平安给你们 John 14:27 | 0.484 |
| 3 | 我留下平安给你们 John 14:27 | 0.755 | 喂饱五千人 Mark 6:30（"来歇一歇"） | 0.445 |
| 4 | 客西马尼园祷告 Matthew 26:36 | 0.751 | 奉我的名求 John 16:23 | 0.413 |
| 5 | 差遣十二门徒 Matthew 10:5 | 0.749 | 撒玛利亚妇人 John 4:1 | 0.402 |

从这张表能直接看出两件事：

1. **重排改变了排名**：只用混合检索时，第 2 名是"差遣七十二人"，和"累垮了"没什么关系，大概是因为里面有"平安"这类词。重排把它挤了出去，换上了"来歇一歇"这种真正贴题的段落。
2. **重排把分数拉开了**：混合检索的 5 个分数挤在 0.749–0.767 之间，几乎分不出好坏；重排后从 0.519 一路排到 0.402，差距明显。这就是为什么"低于 0.30 就拒答"的门槛只能设在重排分数上。

另外说明一下：RAGFlow 1.0 返回的结果里虽然有"全文分""向量分"两个字段，但实际值和总分一模一样，**拿不到两路各自的分数**。所以上面没法列出 0.3 和 0.7 各贡献了多少，这是这个版本 API 的限制。

### rerank 用的是个transformer 模型

"凭什么"打得准？因为它就是专门为这件事训练出来的。 训练数据是海量的"问题 + 段落 + 是否相关"三元组，来源包括搜索引擎里人们点了哪个结果、人工标注、更大的模型给出的判断等等。训练目标很简单：相关的配对打高分，不相关的打低分。看了几百万个例子之后，它就成了一个专门判断"相关不相关"的阅卷老师。

（Voyage 是闭源模型，具体用了哪些训练数据没有公开；上面说的是这类重排模型的通用原理，业内叫 cross-encoder，向量检索那种叫 bi-encoder。）

那为什么不全部用重排？ 因为太慢。它必须在提问的那一刻把问题和每个候选逐对读一遍，没法提前算好。四福音书有几千节，全读一遍来不及。

注意一下，这个项目在第二步 RAG flow 找段落那个部分，是先用混合检索粗筛，然后再重排（Rerank）的。

在第四步挑耶稣的话的时候，他只是用了一个本地的向量相似度,因为这样快，而且它真的不需要那么大的算力，只需要几毫秒。

如果用 Voyage 重排的话，不仅计算量更大，还要再走一次网络，耗时仍然会比这个多一点，其实完全没必要

---
这个注意一下：那 30 个 candidate 是“块”，不是 30 个不同的段落。

其实确实得用“块”，不能说“节”，因为里面的 subtitle 也作为一个独立的东西算作子块了，所以用“节”不好描述，统一都用“块”吧。我们就说拿到这 30 个子块。

对，是 30 个，但要说清楚两个细节。代码里相关的参数是（`ragflow_retrieval.py`）：

```python
"keywords_similarity_weight": 0.3,   # 全文（BM25）占 0.3，向量占 0.7
"rerank_candidates_count": 30,       # 混合检索的前 30 名交给重排
"page_size": max(k * 3, 10),         # k=3，所以重排后返回前 10 个
```

完整的数量变化是这样的：

```
所有块
  │ 混合检索：0.3 × BM25 + 0.7 × 向量
  ▼
前 30 个候选子块 ──→ Voyage 逐个重排
  │
  ▼
重排后的前 10 个子块（RAGFlow 返回给你的程序）
  │ 你的代码：同一个段落只留一个(这边代码设定是它 rerank 之后再次算关键词分+Voyage分权重总得分，要前 10 个。因为如果设定少了，对应段落去重回来的全都是同一个段落，这样就不太好了)
  ▼
从合并到的所有段落里，按平均分排序，只要前 3 个段落（再判断第一名是否 ≥ 0.30， 如果第一个都低于0.3，就全部不要； 当时的思路是：0.30 是用来判断"这个问题本身该不该回答"的，不是用来筛选单个段落的。

它的标定方法就能说明这一点：拿 55 道真实问题和 10 道离题问题（比如"芝加哥天气怎么样"），只比较每个问题的第一名分数：

真实问题的第一名，最低是 0.314；
离题问题的第一名，最高是 0.287。
0.30 刚好卡在两者中间。所以它回答的是"这个问题和四福音书有没有关系"：第一名都不到 0.30，说明整本福音书里都找不到相关的内容，干脆不回忆。第一名过了线，说明问题是相关的，就把前 3 个都交给下一步。）
```

上面那个平均分在源码有写：合并完之后，RAGFlow 会按段落的平均分从高到低重新排序（源码注释就写着 Sort by similarity descending）。

平均分是这么来的：

前 10 个子块，每个都有一个最终分（0.3 × 关键词分 + 0.7 × Voyage 分）。属于同一个段落的那几个子块，把它们的最终分加起来再除以个数，就是这个段落的分数。

拿到前面的前三个段落之后，再去找它们对应的 278 个耶稣亲口说的话。接着把他说过的这些话，用本地几毫秒的向量相似度重新排序，再取前三个送给模型。

---
权重其实用了两次，两次都是针对子块：

第一轮，选 30 个候选时：每个子块的分数 = 0.3 × 关键词分（BM25） + 0.7 × 向量分，按这个分数取前 30 个子块。

第二轮，重排之后：每个子块的分数 = 0.3 × RAGFlow 自己写的关键词重合度（TokenSimilarity） + 0.7 × Voyage 分，按这个分数取前 10 个子块。

<img width="847" height="212" alt="image" src="https://github.com/user-attachments/assets/4d238a6a-f01d-4091-a2e3-7989f62976d8" />

BM25 是两部分相乘，"这个词在这一块里出现得多不多" × “这个词在整个库里有多稀有”。第二部分就是你说的"比例"，叫 IDF。

但要纠正一个细节：这个比例不是按 400 多个段落算的，是按大约 3,800 个子块（一节一节的经文）算的。 因为 Infinity 里真正被索引、被搜索的单位是子块，段落（父块）只是最后用来拼回去的。

举个例子：

“weary”（劳苦）只在少数几节经文里出现，非常稀有。一旦匹配上，就是很强的信号，分数高。
“the”几乎每节都有，太常见了，匹配上也几乎不加分。

（实际公式还多做了两件事：一个词重复很多次，分数不会无限涨，会封顶；一块越长，单个词的分量越轻。这样长经文不会因为字多而占便宜。）

BM25（第一轮，选 30 个候选时用）它问的是："这一块里有你的词吗？有的话，这些词有多珍贵？"

对查询里的每个词，算一个分，再全部加起来：这个词的分 = 稀有程度 × 在这一块里出现的次数

稀有程度：在全部约 3,800 节经文里，有多少节含有这个词。越少越珍贵。

“weary”只有几节有 → 很珍贵，分高

“heart”有很多节都有 → 普通，分低

出现次数：在这一节里出现了几次。但有封顶，出现 10 次不会是 1 次的 10 倍。而且一节越长，每个词的分量越轻。哦，它这个是按“节”算的，因为这个“节”在这里面就是子块的意思。这是因为在第一步初筛、到 rerank 之前，它筛出来的这些东西，最终要拿出来的结果是子块，而不是一个一个段落。所以它这里面才会说是按照那 3000 多个子块来算的

这一节里含有 weary、burden、rest 三个词，都比较稀有，所以总分高。

特点：看全局的稀有程度，分数没有上限，是 Infinity 数据库自带的功能。

---

第二个：关键词重合度（第二轮，重排之后用）

它问的是："你的问题里，有百分之多少的词，在这一块里出现了？"

做法：

给查询里每个词一个重要性权重（RAGFlow 自己算的，大致也是越特别的词越重要）；
看这一块（经文原文 + 标题 + LLM 生成的关键词和问题）里有没有这个词，只看有没有，不数次数；
出现了的词的权重之和 ÷ 所有查询词的权重之和。

假设权重是 weary 3、burden 3、rest 2、heart 2、peace 1，总共 11：

这一节出现了 weary、burden、rest → (3 + 3 + 2) ÷ 11 = 0.73

（还有一个小细节：两个相邻的词组成的词组，比如"heavy heart"，如果整个词组都匹配上，会额外加分。所以短语完整匹配比只零散匹配上几个字更值钱。）

特点：结果在 0 到 1 之间，表示"问题被覆盖了多少"；不数次数，只在那 30 个候选里算。


为什么第二轮要换成重合度？因为它要和 Voyage 的分数相加，而 Voyage 的分数在 0 到 1 之间。BM25 没有上限，直接加会乱套；重合度天然就在 0 到 1 之间，可以直接按比例混合。

<img width="845" height="305" alt="image" src="https://github.com/user-attachments/assets/ba02eeb4-8272-4e35-a660-df6fddaee6e6" />


---

**细节 1：这 30 个是"块"，不是 30 个不同的段落。**

同一个段落里可能有好几节经文同时被命中。比如 Matthew 11:28、11:29、11:30 都和"累垮了"很像，它们会各占一个名额。所以 30 个候选块可能只来自十几个段落。这就是为什么 RAGFlow 返回 10 个之后，代码还要按段落去一次重，才剩下 3 个。

**细节 2：重排读的是是单节经文，即为子块。**

另外，这三个数字（0.3、30、10）都是可以调的参数。0.3 是在消融实验里比较 0.3 和 0.7 之后定下来的；30 和 10 当时没有专门做实验，用的是经验值。

---

等一下，谈到这儿事情还没完，我好生气啊！

我之前光看他跟我谈话输出的效果，觉得还挺可以的，挺符合那种 Jesus 该有的形象。对不起啊，我确实不是一个非常懂这边文化的人，我是按照类似于天父那样仁慈、慈悲、非常亲和的印象来判断的。

可是今天把上面这么一大串逻辑走了一遍，看他怎么挑段落、怎么送给模型，我才发现事情非常不对。我第二版的 prompt 搞错了，完全走歪了。当时我光看结果，那天实在太晚，我又急着弄别的事就没好好搞，对不起。

现在得写第三版 prompt 了。第二版纯粹是从 278 个句子里挑，那这整件事有什么意义呢？我直接把那 278 句话送给 real-time model 不好吗？哪怕是拿那两千多句耶稣亲口说的话，两千多句也不是很大的 token 啊。直接送给模型让它去判断，不比现在这么绕一大圈脱裤子放屁更好吗？哎呦，我就是气得没办法了，真是气得脑袋疼

而且这也违背了我的初衷。我之所以坚持要从四福音书里挑子块，是因为我希望这个人物说话时是带有背景的，而不是单独抛出一句话。单单一句话放在不同的情境、不同的身份下，表达的意思完全不一样。我之所以要从段落里取，就是希望他带着上下文背景，去贴合用户真正在聊的事情。

所以现在得重新改了。唉，这就是技术债吧。要不是今天从头到尾跑了一遍，我还真没发现它跟我的本意差了这么多
