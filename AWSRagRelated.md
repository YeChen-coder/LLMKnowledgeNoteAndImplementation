# 知识图谱是一种数据结构，GraphRAG 是一种用到这种结构的 RAG 方法

知识图谱就是把信息存成"实体—关系—实体"这样的三元组，例如：

(耶稣) —遇见→ (撒该)

(撒该) —位于→ (耶利哥)

(撒该) —身份→ (税吏长)

很多个三元组连起来，就是一张图。

2012 年 Google 推出 Knowledge Graph（搜索结果右边那个信息卡片）之后火过一阵。后来慢慢不提了，主要原因是建图太贵：要么靠人工整理，要么靠一套复杂的 NLP 流水线做实体识别、关系抽取，抽得还不准。

GraphRAG： 2024 年微软把它做火的。核心思路是：让大模型从文档里自动抽实体和关系，把图建出来。建图这件以前最贵的事，现在变便宜了，知识图谱也就借着 LLM 又回来了。

它比普通 RAG 多了两样能力：

多跳查询。 比如问"耶稣在耶利哥都遇见了谁？"，可以沿着图走：耶利哥 → 位于这里的人 → 和耶稣的关系。普通的向量检索只能找"文字上相似"的段落，不会顺着关系走。

全局问题。 把图里联系紧密的一群实体归成一个"社区"，提前给每个社区写好摘要。问"福音书里耶稣和法利赛人的冲突有哪几类？"这种需要通读全书的问题时，可以直接用这些摘要来回答。

In short, 知识图谱是材料，GraphRAG 是做法；LLM 让这份材料重新变得做得起。

用 GraphRAG 其实不太适合这件事情。现在本来用的就是 RAGFlow，它本身就自带知识图谱的能力。如果要做的话，在当前已有的这些向量之上再做一些 processing 就行。但是它默认模板不行，是给现在公司用的，基本上全是法规、谓词之类的，这块还得改。-To be specific, 容器里的 knowledge_graph.yaml，默认抽取的实体类型是人物、组织、产品、法规、地点，关系类型是"拥有、属于、导致、监管、位于"，是按企业文档设计的。用在福音书上，"产品"”法规"”监管"基本没用，而真正重要的"事件"”群体（法利赛人、门徒）"”遇见、医治、教导"都没有。

我之前在想愣给它找一个应用场景，能不能放在我的个人网站上，把个人网站的 RAG 部署到 AWS 上面去炫个技，就聊到每个月的开销问题：

1. AWS 那边提供 Neptune Analytics，但是太贵了。它是按图开着的时长收费，和有没有人访问没关系。官方价格 256GB 的图每小时大概 $7.68。但如果放在个人网站上，做个展示个人经历的东西，哪用得着 256GB？能凑够 1GB 都算我疯狂往里灌水了。按小时算真的太贵，这个方案不行。
   
2. 后来顺手再找应用场景，想说能不能在个人网站上搞 Knowledge Base 再做 hybrid search。看了一眼，唯一对钱包友好也比较实际的，就是 Knowledge Base 用 S3 Vectors 做 search。可是用 S3 Vectors 基本上就只是 BM25，根本用不了 hybrid。但凡想用 hybrid search，就得用 OpenSearch，一个月起码 $150 打底。

这边稍微补充一下：在 AWS Knowledge Base 里面处理文档，必须给它配一个向量库，重点其实是在这个向量库的成本上。（- 顺手吐槽一下，AWS 里面那个 Hybrid 检索，它的权重是不公开的，完全是个黑盒。我一直觉得这事非常让人牙疼：这种 weight 怎么能做成黑盒呢？还是说它底层用了特别复杂的算法，导致根本就不能从一个 weight 的角度去理解，反正就是开箱即用、一用就完了那种，主打一个非常简单直接易懂）

能给它配的向量库主要有这些方案：

1. S3 Vector
它是按存储和查询次数计费的。就我那点资料基本可以忽略不计，一个月也就几分钱的事。
2. OpenSearch
• OpenSearch Serverless：经典版一个月 $175 起步
• OpenSearch 托管集群：既然是集群，一个月起码得开着一台机器，而且文档里写了它不支持 Hybrid Search
3. Aurora PostgreSQL (pgvector)
我之前在别的文档里写过它的扫盲。如果用 Serverless v2，没人访问的闲时可以缩容到 0。问题在于冷启动需要等十几秒，聊天的时候会先卡顿一下。不过为了成本和钱包考虑，顶多在文档里说明一下冷启动要十几秒，我觉得其实也行，没准真行。另外 PostgreSQL 是可以做混合检索的。（- 补充一下，我觉得这事可行。它是按照启动之后的时间计费的。反正它冷启动也就十几秒，而且也只是在第一个问题上慢，后续还可以搞一些工程手段对它进行优化。我觉得这事能干，完全可以炫技，给我简历上再加点东西）
4. Neptune Analytics (GraphRAG)
这个非常贵，是按小时计费的。

剩下的几个都是第三方方案，不是 AWS 原生支持的：

• Pinecone：有免费档，但不支持混合检索。
• MongoDB Atlas：虽然有免费集群，但能不能给 Knowledge Base 用，还需要再去核实。
• Redis Enterprise Cloud：闲时也烧钱，而且根本不能做混合检索。

整件事情看下来，这种场景本来就不该用，绝大部分其实是成本问题。而且我觉得我还挺聪明的，我又不是冤大头，为什么要在这种一个月都不见得有人用、网站都不见得被访问几次的状态下去给他付费呢？其实这个就有点像我之前吐槽的那些情感陪伴类项目：一天都不见得有真人用，不见得能产生几条 log，却非要上 Redis 集群。我对这件事情的感觉是一样的，这不完全是拿着一把屠龙宝剑削土豆吗？我脑袋都快想疼了，这种场景给它上这些东西，真是太烦了。

# RAGFlow Vs AWS

RAGFlow 的话，那是好消息：** Elasticsearch 被 RAGFlow 包在里面，你没直接碰到。**

RAGFlow 用 Docker Compose 起的那一堆容器里，有一个就是 Elasticsearch（`es01`），默认的文档引擎就是它。`docker/.env` 里还有个 `DOC_ENGINE` 配置，可以切成 `opensearch` 或者 RAGFlow 自家的 `infinity`。

这个对比要分两层：**RAGFlow 整体**对应 AWS 上的一套托管服务，**RAGFlow 里的 ES** 对应 OpenSearch Service。

---

## 第一层：RAGFlow 的每个容器，搬到 AWS 上换成什么

| RAGFlow 本地容器 | 它干什么 | AWS 上对应的 |
|---|---|---|
| **es01（Elasticsearch）** | 存切好的块：全文索引 + 向量 | **OpenSearch Service** |
| **MinIO** | 存你上传的原始文件 | **S3**（MinIO 本来就是兼容 S3 接口的） |
| **MySQL** | 存知识库、用户、对话这类元数据 | **RDS / Aurora MySQL** |
| **Redis / Valkey** | 解析任务队列、缓存 | **ElastiCache**（或者把队列换成 SQS） |
| **ragflow-server**（API + 解析任务进程） | 业务逻辑、DeepDoc 解析 | **ECS + Fargate** |
| nginx + 前端 | 网页入口 | **ALB** + Fargate，或者 S3 + CloudFront |
| 你配置的 LLM / Embedding 模型 | 生成回答、算向量 | **Bedrock** |
| 容器日志 | 排错 | **CloudWatch** |


## 第二层：RAGFlow 的概念，对应 OpenSearch 和 Bedrock 的哪个功能

AWS 上和 RAGFlow 整体最像的，其实是 **Bedrock Knowledge Bases**：它也是"上传文档 → 解析 → 切块 → 向量化 → 存起来 → 检索 → 生成"这一整套，而**它默认的向量库就是 OpenSearch Serverless**。

| 你在 RAGFlow 里熟的 | 在 OpenSearch 里 | 在 Bedrock Knowledge Bases 里 |
|---|---|---|
| 知识库 | 一个 **index** | 一个 Knowledge Base |
| 上传文件 | 不管这一层 | 数据源指向一个 **S3 桶** |
| 解析方法（General、Q&A、Paper、Laws、Table……）+ DeepDoc | 不管这一层 | 解析选项（默认解析或用基础模型解析）+ 切块策略（固定长度、分层、语义） |
| 嵌入模型 | `knn_vector` 字段（维度要和模型对上） | 选一个 Bedrock 的 embedding 模型 |
| 检索测试里的**"关键词相似度权重"** | **hybrid 查询 + normalization 管道里的 weights** | 检索时选 hybrid 搜索 |
| 相似度阈值、Top N | `min_score`、`k`、`size` | `numberOfResults` 等参数 |
| Rerank 模型 | 搜索管道里的 rerank processor | 检索配置里的 rerank 模型 |
| 聊天助手 | 不管这一层 | `RetrieveAndGenerate` API |
| Agent 画布 | 不管这一层 | **Bedrock Agents / Bedrock Flows** |

可以看到，OpenSearch 只负责"存和查"这一段。RAGFlow 其他那些你熟的东西（解析、切块、对话、Agent），在 AWS 上要么归 Bedrock 管，要么你自己写在 Fargate 或 Lambda 里。

---
# Start from RagFlow

<img width="371" height="347" alt="image" src="https://github.com/user-attachments/assets/4a808a39-e81b-44d2-9b79-352db7e1cee3" />

当前这个图，是我本地项目里，RAGFlow 开的这 6 个容器。然后从这里边，开始正儿八经去对.

首先，它这边默认仍然是 RAGFlow 里面是 self-hosted 的 Infinity。在完全 default、没有换参数的情况下，它没有用 OpenSearch.

先不谈其他这几个东西是干嘛的，它里面真正跟搜索有关的其实只有一个东西叫 Infinity。Infinity 跟 OpenSearch（也就是 ElasticSearch）是同一个生态位的东西； 之前去调它的时候，不是搞了一个 keyword similarity weight = 0.3 吗？意思就是它里边的 similarity 占 30%，而 semantic 占 70%。这些都是 Infinity 里面的东西。对应到 OpenSearch，它也能做混合检索，调的这个比例在 OpenSearch 里面也有对应的设置。 -吐槽一下，那确实得有啊。一旦涉及到 hybrid，没有 weight 这事儿怎么办呢？你硬五五分吗？这是一个很容易想到的逻辑

再说一下处理好的这些数据。

因为这边用的是 Docker，所以数据放在 Docker 自己管理的一个 volume 里。它用的这个 volume 名字叫 jesussayshi-ragflow_infinity_data，大概 200MB。

里面主要是切好的块、算好的向量，还有关键词索引。

既然谈这件事情，就顺手把 RAGFlow 的事先理清吧。RAGFlow 理清了，再去涉及到 AWS 那边也没什么难的。

圣经它本来的结构是先是书，书里有章，章里有节，这个节类似于 John 21:20。

除此之外，因为我用了这个译本，它在节和节之间加了小标题，把讲同一件事的几节归成一组，比如 John 21:20-25。

所以我这边其实是按照小标题区分的，把整个原文档按照它一共有的 487 个小标题给切分开; 这个叫 “structure-aware chunking（按文档自身结构切块）”

反正这个就是 RAGFlow 里面一个文档打开的样子。因为这个项目本来就做了一个 ingestion enrichment，所以它是对每个 chunk 单独先让 LLM 过了一遍，去给它写 5 个关键词和 3 个问题。

<img width="1352" height="785" alt="image" src="https://github.com/user-attachments/assets/9899292d-a93e-462b-b31b-af4941492490" />

<img width="822" height="618" alt="image" src="https://github.com/user-attachments/assets/9392b46a-c20b-45e7-9492-0b2731a6dfe4" />

对不起啊，我之前确实没有仔细看，之前进度实在太迅速了，导致我根本顾不上来。但我自己看了一下，发现那个鬼关键词我不是很理解：为什么有的 chunk 它把这个 verse 的编号也当成了一个关键词？- 因为它就是个小标题，真的就这么一句话，你非让它从中提炼出 5 个关键词来，它也费劲。以下图为例吧

<img width="762" height="592" alt="image" src="https://github.com/user-attachments/assets/283996c9-d5fe-4b9e-85e4-6490bf49ae2d" />

再说，然后再往下就是 ingestion enrichment 的一些配置了

几个参数的说明：

1. Candidate：先粗选出多少条，然后给 Rerank/Relevance 排序用的，我这里的程序里设的是 30
2. Metadata filter：指的是按标签筛选。在程序的 V2 版本中，因为要求必须得是 Japanese，所以当时设的是 yes；但现在 V3 根本不用那个了，所以这个目前也没用
3. Top N：指的是最后显示几条，我这个程序里配置的是 3
4. 网页上没写，但程序里设了 min score = 0.3。意思是如果排第一的结果都不到 0.3，那就当没找到，挑出来的这三个就一个都不要了

## configuration 的一些配置参数

<img width="1271" height="783" alt="image" src="https://github.com/user-attachments/assets/1faf1ad3-5dc2-44fb-a69f-56c3ee2f5592" />

这个窗口管的是**数据进图书馆之前要经过的加工流水线**。前面讲的都是"怎么找书"，这里讲"书是怎么上架的"。

### Parse mode：选一套现成的流水线

**General（Built in）** 意思是"通用型，系统自带的"。RAGFlow 给不同类型的文档预备了不同的流水线，比如 Q&A（问答对）、Paper（论文）、Book（书）、Laws（法律条文）、Table（表格）等。你也可以自己拼一套。

你的经文是纯文本，所以用 General，然后在里面改了几个设置。

### 下面四个标签 = 流水线的四道工序，从左到右依次执行

| 工序 | 干什么 | 打比方 | 你的项目里怎么设的 |
|---|---|---|---|
| **Parser** | 把文件读成纯文字 | 把书拆封、摊开 | 默认。你的文件本来就是 txt，没什么要处理的 |
| **General Chunker** | 把文字切成卡片 | 把书拆成一张张卡片 | **按换行切，一行一张卡片；整个文件是父，每行是子**。这就是之前讲的"一节经文一张卡片" |
| **Extractor** | 让 LLM 给每张卡片补标签 | 给卡片写上关键词和"可能回答的问题" | **每张卡片 5 个关键词、3 个问题**。你在卡片里看到的那些就是这一步生成的 |
| **Indexer** | 建索引 | 把卡片编进两套目录 | **两套都建：向量（按意思找）+ 全文（按关键词找）** |



**Parser 页上那个 Enable vision enhancement**：遇到图片、扫描版 PDF 时，调一个能看图的模型去识别内容。我的文件是纯文字，用不上。

### 和 AWS 的对应（顺便记一下）

Bedrock Knowledge Bases 建知识库时，要选的东西几乎就是这四步：
- **Parsing strategy**：对应 Parser
- **Chunking strategy**：对应 Chunker
- **自定义处理（Lambda）**：对应 Extractor。AWS 不自带"每块生成问题"的功能，要自己写
- **Embedding model + 向量库（比如 OpenSearch）**：对应 Indexer


这边它还有一个 memory，我顺手看了一下是什么意思：它指的是对用户的那种 memory。

我也很想吐槽Jusus项目里的 memory，搞得实在是太有歧义了。在我自己的Jesus项目里，memory 指的是从 Gospel 里边搞出来的耶稣生平的那些 chunk。其实一般来说，大家普遍说的 memory system 都是指对用户的一些记忆。

AWS 上管"记住用户"的是 Bedrock AgentCore Memory，分短期记忆（这次对话）和长期记忆（跨对话的用户信息），和 RAGFlow 的 Memory 是同一类东西。

# 个人网站上的Bedrock chatbot

## hybrid research 
抓大放小吧：

HNSW 指的是向量索引，它和 OpenSearch 的 KNN 都是一种东西。具体算法什么的就别揪了，反正知道它是一个针对语义空间的 Vector 索引就得了。

然后那个 GIN，它是 PostgreSQL 的一个倒排索引，记录每个词出现在哪些块里边。

BM25 是一种打分方式，BM25 本身不是索引。但这东西也是抓大放小，具体算法各家都有各家的独门之道，不重要，知道它是个关键词索引就行了。

我这边是用 Hybrid，两边都用了，完事儿了; weight 参数它是 Bedrock 的黑盒，不知道，没法调，就是它握在 AWS 手里边，没法看.

## Bedrock 概述

AWS 的 Bedrock 是一个大平台，Knowledge Base 只是其中的一个功能。Bedrock 里面除了 Knowledge Base，还包括：

• 各种大语言模型（比如 Claude 等）
• Agents Workflow（托管的 Agent 工作流）
• Guardrails（内容安全过滤）
• Evaluations

这个项目之中，虽然说 AWS 那边 Knowledge Base 直接提供了一个全包的“检索加生成”方案，但我这边没有用这个全包的，仍然是给它拆开处理的。

也就是 Knowledge Base 这边只负责检索，检索出来的这些 chunk，我再单独让这里去Lambda调用 Claude它走 Haiku。

这样做的好处是：

1. 对 prompt 能有更多的把控
2. log 会更全一些，方便排查问题
3. 也更方便后续做调整和更改


### Evaluations

针对这种事情，它有一个 Bedrock Evaluation，而且能专门评测 Knowledge Base。它有内置参数，也能自定义。

评测方式是拉上一个 model 做裁判：准备一份问题集（问题加答案放在 S3 里），它会跑一遍我的 RAG，然后再交由大模型去做逐条打分

内置指标分两类任务：

只评检索（Retrieve only）- 上下文相关性：查到的块和问题相不相关。上下文覆盖率：标准答案需要的信息查到了多少（要你提供标准答案）

检索 + 生成	- 正确性、完整性、忠实度（有没有超出资料编造，也就是幻觉）、引用准确度、有用性、逻辑连贯性，还有有害内容、拒答这类安全指标

Bedrock 的内置指标全靠 LLM 当裁判打分。好处是不用逐条标注"正确的块是哪个"，坏处是裁判自己也可能打错，每跑一次还要付裁判模型的钱。

自定义： 你可以写自己的评分提示词，定义新指标。比如给你的场景定义一个"有没有编造联系方式"。


做 evaluation 都得有一套标准答案，其实它本身就是一个 well-defined problem。对于纯文字的内容，我描述可能不是很精准，但意思就是这种有法可依的、有固定文字能确定对错的场景，其实已经有一套非常成熟的技术了。

这种地方没有什么好发挥聪明才智的，而且越是发挥聪明才智，出来的结果八成不会比标准的 best practice 好到哪去，大概率只会更差。所以这边就是他们说什么就干什么。毕竟已经发展到这个程度了，这甚至都不是重新造轮子的问题，自己弄八成根本造不出轮子，只会造出一个方的多边形; 这个又不是写硕士论文，又不是要创新点，你就听听人家的话得了.

## 我的资料进库

相当粗糙的，不过这事儿也不怪 AI。因为我用 Claude Code 帮我干的，我还挺理解它。

<img width="677" height="220" alt="image" src="https://github.com/user-attachments/assets/bfd7f96a-be6a-4de8-abfd-0f724161ff0b" />


它现在的切块（chunking）处于一种非常原始、远古的状态：
• 每一块最多 400 token
• 相邻两块的 overlap 是 15%
• 每块交给 Amazon Titan Text Embeddings V2，得到 1024 维向量
• 再写入 Aurora 的 bedrock_integration 表里（一行是一块）

我想吐槽的就是这个切块的程度。它拿了我 GitHub 上 public 仓库的 README，还有原本网站的文案，一共生成了 42 个 .md 文件。

这事儿真不怪它，主要是我那些 README 写得非常跳脱：分段没好好写，里边写的大多不是技术架构，更多是开发日志和各种吐槽；加上本人表达欲极其旺盛，就堆得非常长，也没好好分段，确实是 garbage in, garbage out 了。

没关系，我之后单独再去调一下，跟它说别看 README，去看一些专门的技术架构文档。算了，我现在就让它干吧 - doing

## 访客提问后

<img width="675" height="405" alt="image" src="https://github.com/user-attachments/assets/fd0f11d7-655b-488c-b2b3-83c40dfb7df3" />

Lambda function, 老朋友了，之前就干过，这个就不在这儿说了。

不过他这边我有一个疑问：为什么这边还跟 DynamoDB 扯上关系了？- 这个因为设定了每天最多是 80 次嘛，所以就为了防止钱包爆炸这事。也是为了防止 DDoS，防止 Cloudflare 他那边：就是号称他老板“人美心善”搞的那个 DDoS 防护墙给失效了，让我这边爆炸了，所以这边就愣给他加上了这么一层。

而且便宜，这就是以各种方式去实现这个需求。因为他这个技术是得在唤醒 PostgreSQL 之前去做的。

<img width="741" height="330" alt="image" src="https://github.com/user-attachments/assets/8596dc52-bbcc-4cf4-9a03-0d4781e12cef" />

<img width="927" height="252" alt="image" src="https://github.com/user-attachments/assets/f8555137-8a93-44a2-a26e-dff9f2dd1870" />
