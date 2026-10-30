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

用 GraphRAG 其实不太适合这件事情。现在本来用的就是 RAGFlow，它本身就自带知识图谱的能力。如果要做的话，在当前已有的这些向量之上再做一些 processing 就行。但是它默认模板不行，是给现在公司用的，基本上全是法规、谓词之类的，这块还得改。

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
我之前在别的文档里写过它的扫盲。如果用 Serverless v2，没人访问的闲时可以缩容到 0。问题在于冷启动需要等十几秒，聊天的时候会先卡顿一下。不过为了成本和钱包考虑，顶多在文档里说明一下冷启动要十几秒，我觉得其实也行，没准真行。另外 PostgreSQL 是可以做混合检索的。
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

<img width="371" height="347" alt="image" src="https://github.com/user-attachments/assets/4a808a39-e81b-44d2-9b79-352db7e1cee3" />

当前这个图，是我本地项目里，RAGFlow 开的这 6 个容器。然后从这里边，开始正儿八经去对.

首先，它这边默认仍然是 RAGFlow 里面是 self-hosted 的 Infinity。在完全 default、没有换参数的情况下，它没有用 OpenSearch.

先不谈其他这几个东西是干嘛的，它里面真正跟搜索有关的其实只有一个东西叫 Infinity。Infinity 跟 OpenSearch（也就是 ElasticSearch）是同一个生态位的东西； 之前去调它的时候，不是搞了一个 keyword similarity weight = 0.3 吗？意思就是它里边的 similarity 占 30%，而 semantic 占 70%。这些都是 Infinity 里面的东西。对应到 OpenSearch，它也能做混合检索，调的这个比例在 OpenSearch 里面也有对应的设置。 -吐槽一下，那确实得有啊。一旦涉及到 hybrid，没有 weight 这事儿怎么办呢？你硬五五分吗？这是一个很容易想到的逻辑

再说一下处理好的这些数据。

因为这边用的是 Docker，所以数据放在 Docker 自己管理的一个 volume 里。它用的这个 volume 名字叫 jesussayshi-ragflow_infinity_data，大概 200MB。

里面主要是切好的块、算好的向量，还有关键词索引。




