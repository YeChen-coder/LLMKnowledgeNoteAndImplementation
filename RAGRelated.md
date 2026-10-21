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
