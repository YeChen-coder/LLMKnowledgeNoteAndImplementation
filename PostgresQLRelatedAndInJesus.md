# PostgresQL itslef

就是现在先总结一下吧，先谈一下 Postgre 是个什么东西。就之前我不是本来就会 MySQL 吗？所以说，Postgre 它是，就是它在 SQL 这个部分，包括那个 schema， 然后一些什么就是什么 create table 什么这个 SQL 这个部分都是一样的，这个无所谓的。然后就是，但是它跟 SQLite 是不一样的，就是 Postgre 它不是一个文件，而是一个一直运行的服务进程，然后默认监听的是5432端口，这个跟 MySQL 是一样的，就是但是它跟那个 SQLite 是不一样的嘛。以及连接信息的时候，一般都写成一个连接串 connecting string。
postgresql://用户名:密码@主机:端口/数据库名
postgresql://postgres:mypassword@localhost:5432/mydb

## 必须知道的几个概念

**1. 参数化查询（最重要）**

```python
# ❌ 有 SQL 注入风险
cur.execute(f"SELECT * FROM users WHERE name = '{name}'")
# ✅ 让驱动帮你转义
cur.execute("SELECT * FROM users WHERE name = %s", (name,))
```

占位符写法因驱动而异：psycopg 用 `%s`，Node `pg` 和 Go 用 `$1, $2`。

**2. 事务 / commit**

Postgres 的驱动默认会开启事务。**你写入的数据要 commit 之后才真正保存**。用 `with` 块会自动处理；手动管理的话：

```python
conn = psycopg.connect(DSN)
cur = conn.cursor()
cur.execute("UPDATE ...")
conn.commit()      # 忘了这句，改动就丢了
conn.close()
```

"程序跑完了数据却没进去"，十有八九是因为没 commit。

**3. 连接池**

建立连接的成本比较高。Web 服务不要每个请求都 `connect()` 一次，而是开一个**连接池**，所有请求共享。Node 的 `pg.Pool` 就是连接池；Python 可以用 `psycopg_pool.ConnectionPool`。

这边其实可以补充一下，因为我之前对这个也有过疑惑。

我们假设这么一种情况：现在 Postgres 一共连了 3 个连接池，每个连接池里有 10 个连接，所以在同一时刻最多可以有 30 个连接一起访问 Postgres。它并不是 FIFO（先进先出），也就是每一时刻只能让 3 个池子里排在第一位的去连；而是只要池子能容纳，这 30 个连接是可以同时开始的。在这种情况下，它省去了连接池跟数据库之间进行 SSL 握手之类的开销，也省了一些进程相关的资源。

（这边着重补充一下：其实最大的开销在于 Postgres 会为每个连接 fork 一个新进程。

如果有连接池的话，其实还是做了一些 trade-off：
• 没有连接池：一开始很纯净，有东西跟它 connect，它就 fork 一个进程；connect 完了，这个进程就关掉
• 有连接池：对应一个连接的进程其实得一直开着）

再谈一下为什么一开始要设计成一个连接一个进程：

那是因为 PostgreSQL 诞生在 1986 年前后，当时多线程本来就不成熟，所以那时候完全没有考虑过这种性能问题。而且当时觉得进程隔离更安全稳定，比如一个连接的后端进程崩了，不会把别的连接一起拖崩。这么搞在当时代码也确实比较简单。

到了现在如果再去改屎山，PostgreSQL 代码里面已经有非常多的全局变量了，而且它们的前提都是该进程只服务一个连接。现在去搞重构难度真的很大，所以最好不要动屎山，就先用连接池把这个问题解决掉，而不是从程序最原初的地方去改成适应现代的样子，先用着吧。 而且哪怕把 Postgres 改成多线程了，连接池本来也解决了其他问题，比如建连接前的各种开销：TCP 握手、TLS 握手、密码认证、初始化会话这些。哪怕真改成了多线程，这些开销本来也还是在的。

而且每个连接在数据库那边本来就要维持一套状态，比如内存缓存、事务信息和锁信息等。不管是进程还是线程，1000 个连接就是 1000 个状态。

另外并发数还会受 CPU 核数限制。要是并发数太多而 CPU 核数不够，就会开始抢占 CPU 时间，反而限制了 I/O 效率。

所以现在这样挺好的，不用改

另外还有一个好处：本来 Postgres 内部有个 limitation 是 100 个，如果当前有 150 个请求，在不使用连接池的情况下，没连上的那 50 个就会直接报错。但如果用连接池在中间多加了一层，那 50 个请求就会在连接池外排队等待进入，不会直接报错，体验会好很多。 再补充一下：这边的请求里，我们现在说的这个“用户”并不是指真人，在这种场景下它一般指的都是后端程序。

排队顺序： psycopg_pool、Node 的 pg.Pool 这类常见实现基本都是先来先得（FIFO）。但实际中你很少需要关心顺序，因为：

一条简单查询通常只占用连接几毫秒，所以 10 条连接每秒能处理上千个请求
如果经常出现排队，说明池子太小、某些查询太慢，或者有代码借了连接却没还

等太久会怎样？ 池子一般有超时设置，比如 psycopg_pool 的 timeout 默认是 30 秒。超时后这个请求直接报错，不会无限等下去。

### connection pool 几个容易踩的坑

池子只创建一次：放在全局或在应用启动时创建，不要在每个请求里 new Pool()，那样就等于没用池子。

借了一定要还：用 with / try...finally。连接泄漏的典型症状是程序跑一段时间后所有请求都卡住。

事务必须在同一条连接上执行：BEGIN 和 COMMIT 如果分别借了不同的连接，事务就不成立。

池子不是越大越好：Postgres 每个连接都是一个进程，连接太多反而更慢。一般 10～20 条就够用。注意：如果你部署了 5 个实例、每个实例池子大小为 20，总共就是 100 条，正好碰到默认上限。

什么时候不需要池子：一次性脚本、命令行小工具，直接 connect() 用完关掉就行。



**4. ORM（可选）**

如果不想手写 SQL，可以用 ORM，把表映射成类：
- Python：**SQLAlchemy**、Django ORM
- Node：**Prisma**、Drizzle

ORM 底层还是用上面这些驱动发 SQL。既然你会 SQL，建议先用原生驱动熟悉一遍，再决定要不要上 ORM。

## Postgres 相比 MySQL / SQLite 的一些特色

- `RETURNING`：插入、更新后直接返回行，不用再查一次
- `JSONB` 类型：可以存 JSON，还能按字段查询和建索引
- `SERIAL` / `GENERATED AS IDENTITY`：自增主键的写法
- 数组类型、`ON CONFLICT DO UPDATE`（upsert）
- 默认的 schema 叫 `public`，表的全名是 `public.users`

## 调试工具

- **psql**（命令行）：`psql postgresql://postgres:mypassword@localhost:5432/postgres`，进去后 `\dt` 列出表，`\d users` 看表结构
- **图形界面**：pgAdmin、DBeaver、DataGrip

---


# Postgres has pgvector to store vectors
pgvector is an extension of PostgreSQL that stores vectors in a relational database.

然后这边又谈到为什么现在 PostgreSQL 突然又开始火了。这是因为 AI 应用所需要的几样东西，PostgreSQL 数据库基本都能做，而且做得还挺好的。

最核心的原因就是它有 pgvector 这个 extension。AI 应用（尤其是 RAG）本身就离不开向量搜索：先把文档切成小块，用 embedding 模型转成 vector；用户提问的时候，再找到语义最接近的几段，最后交给大语言模型。

最早期的时候，大家做这事都是用专用向量库，比如 Pinecone、Milvus 之类的。但后来 PostgreSQL 提供了 pgvector 这个扩展，向量就直接变成了一种普通的列类型.

刚才不是说 Postgres 的好处，就是胜在能把向量跟其他数据放在同一个库里，因为它本来就是一个关系型数据库。

如果我们还像之前那样用专用数据库，那好，vector 存一个地方，用户的各种数据又存一个地方，事情就全乱了。这真的对人的身心健康很不好：多一样东西就多一样 troubleshooting 和维护的开销。万一出事了，你还得到处排查到底是哪边的问题，非常烦。

所以还不如就用 Postgres，最好一项直接给我全包了。刚才那个其实就是个简单的情况：多一项东西，就多一个认知的开销。

但最重要的原因，是用 PostgreSQL 可以直接带业务条件过滤。比如真正的需求往往是：在这个客户有权查看的、最近一年的文档里，找最新的某某信息。用 Postgres 直接在查询里就把权限控制做掉了，根本不会有越权这种鬼问题。

同样的情况下，如果是独立的向量数据库，就得把权限、租户和时间等信息同步一份过去，两边的数据还可能对不上，维护起来非常麻烦。

而且它还能通过事务保证一致性：删除一篇文档时，它的元数据和所有向量可以在同一个事务里一起删掉，绝不会出现文档删了、向量还在、RAG 里还能搜出来的情况。

这个我之前在 Twitter 上见有人聊过（具体什么产品我忘了）：有个用户不用某个 AI 应用了，把自己的数据全删了，结果一查才发现，自己之前上传的文档其实依然在向量空间里“永世存活”

对不起啊，刚才扯多了。现在总结一下，它主要有三个好处：

1. 少维护一套系统
2. 可以带业务条件过滤，一条 SQL 搞定
3. 事务保证一致性
   
### postgres 其他契合 AI 应用的能力
上述三个是它的主要原因。但它也有一些契合 AI 应用的能力：比如它支持 JSONB，可以用来存不规则的数据。

尤其是现在用 Agent 了，各种输出、参数、执行轨迹以及数据结构都在变。JSONB 可以直接存 JSON，还能按字段查询、建索引，不用每次都去改表结构

SELECT * FROM agent_runs WHERE result->>'status' = 'failed';

全文搜索 + 向量搜索 = 混合搜索（hybrid search）
光靠向量搜索，对专有名词、产品型号、人名这类精确匹配效果不好。Postgres 本身有全文搜索（tsvector），可以和向量搜索结合，在同一个库里做混合检索。

一库多用
聊天记录、用户数据、任务队列（SELECT ... FOR UPDATE SKIP LOCKED）、缓存、向量都可以放在 Postgres 里。社区里流行的说法叫 “Just use Postgres”：先全部用 Postgres，等真遇到瓶颈再拆出去。


用 pgvector 的话，全文检索并不需要另外搭。PostgreSQL 本身就自带全文检索：tsvector 加 GIN 索引。所以向量和全文完全可以放在同一个 Postgres 库里，一条 SQL 就能做混合检索。我上一条说"通常要另外搭"，这一点只对 Pinecone 成立：它主要存向量，要做关键词检索，得自己另外生成稀疏向量传进去。

### 不足之处

postgreSQL它也不是万能的:

向量规模非常大（比如几十亿条）或对延迟要求极高时，专门的向量数据库通常更合适

pgvector 的索引（HNSW、IVFFlat）需要调参，"过滤条件 + 近似搜索"组合时可能出现召回不全的问题，需要一定经验处理

分析型的海量数据（数仓类场景）不是它的强项

不过对绝大多数 AI 应用来说，数据量离这些上限还很远。

---
pgvector 和 Infinity 真正的区别在两个细节上：

1. Postgres 自带的全文检索不是 BM25

你理解得对：全文索引就是关键词检索，BM25 是给关键词匹配打分的标准算法。它看两件事：

这个词在这段里出现了几次；

这个词在整个语料里有多稀有（IDF）。比如"sparrow"这种词，比"the"要有分量得多。

Postgres 自带的打分函数 ts_rank 只看词在当前这一段里的情况，不看整个语料，也就没有 IDF。所以它能搜到包含关键词的段落，但排序质量不如 BM25。要在 Postgres 里用真正的 BM25，得另装扩展，比如 ParadeDB 的 pg_search。

<img width="926" height="190" alt="image" src="https://github.com/user-attachments/assets/c4bc08e5-e9c9-4909-b03a-68a821c48b6f" />

所以准确的说法是：pgvector 也能在一个库里做混合检索，只是 BM25 打分和结果合并要自己补；Infinity 把这两样都内置了。

另外顺带一提：阶段 1 那版检索里，BM25 是在 retrieval.py 里手写的，大约 30 行 numpy，合并用的也是手写的 RRF。所以这两种做法你其实都经历过：先自己手写，后来换成了 Infinity 的内置功能。面试时这正好能讲。

---
Pinecone is a cloud-hosted vector database.

---
In my (JesusSaysHi) project, I am using Ragflow's built-in Infinity engine. It is self-hosted and in the same category as Pinecone or Milvus, but it keeps dense and full-text indexes in one store, so hybrid search is a simple query. I did not need a managed service at this scale.

向量存在 RAGFlow 自带的 Infinity 数据库里。Infinity 是 InfiniFlow 自家的引擎，向量索引和全文索引存在同一个库里，所以混合检索是在一处完成的。

具体位置：

容器：jesussayshi-ragflow-infinity-1，由 JesusSaysHi/ragflow/docker/.env 里的 DOC_ENGINE=infinity 选定。
数据：在 Docker 卷 jesussayshi-ragflow_infinity_data 里，挂到容器内的 /var/infinity。这个卷在 Docker Desktop 的 WSL 虚拟机里，不在你项目的文件夹中，在 Windows 资源管理器里直接是找不到的。
配置：ragflow/docker/infinity_conf.toml，以 bind 方式挂进容器。

里面存了什么： 487 个 BSB 段落，按经节切成子块。每个子块有 text-embedding-3-small 生成的向量、全文索引，以及 LLM 生成的关键词和问题。其他东西分开存：文档和数据集的元数据在 MySQL 容器里，上传的原始文件在 MinIO 容器里。

<img width="941" height="276" alt="image" src="https://github.com/user-attachments/assets/f9d4a248-058e-49e0-a18e-6ca74ecbab46" />

MinIO 是什么：一个"对象存储"，专门存文件，接口和亚马逊的 S3 一样。你可以把它理解成"自己机器上的 S3"。RAGFlow 设计上要处理用户上传的大量 PDF、Word，原件需要放在一个可靠的文件仓库里，以后重新解析时还要从这里取。

为什么要 MySQL：管理信息需要"关系型"的东西：哪个文件属于哪个数据集，解析状态从"排队"改成"完成"时不能出错。这是传统数据库最擅长的，而 Infinity 是为搜索设计的，不适合干这个。

（另外还有两个配套容器：kvrocks 做缓存，nats 是解析任务的排队系统。所以 RAGFlow 一共起了 6 个容器。）

那全放 Infinity 不行吗？

对你来说，完全可以，而且更简单。阶段 1 就是证明：两个文件（向量数组加段落 JSON），没有任何数据库，检索照样能跑。

RAGFlow 要这么多组件，是因为它的定位是一个多用户的企业知识库平台：很多人上传各种格式的文档，在网页上管理，后台排队解析。三套存储是为那种场景准备的。

而你这个项目真正检索时用到的，几乎只有 Infinity。 MinIO 和 MySQL 只在导入、管理阶段起作用。

“RAGFlow brings a whole platform: MinIO for raw files, MySQL for bookkeeping, Infinity for the search index. At my scale that’s overkill. My phase-1 baseline was two files in numpy. I took on that weight for the production pipeline: structure-aware chunking, chunk enrichment, hybrid search and rerank behind one API. If this were only retrieval over 3,800 verses, I’d go back to pgvector or plain files.”

# RagFlow 是怎么设计想让用户用的

<img width="938" height="472" alt="image" src="https://github.com/user-attachments/assets/f75de7ca-38aa-41fd-94f1-d1426ad54704" />

可以把 RAGFlow 想成一家**图书馆**，6 个容器就是馆里的 6 个岗位。

## 6 个岗位分别干什么

| 容器 | 比喻 | 实际干什么 |
|---|---|---|
| **ragflow**（主程序） | 前台加馆员 | 网页界面、对外的 API（你的代码就是调它），以及真正干活的解析程序：读文件、切块、调 OpenAI 生成向量和关键词。其他 5 个都是它的帮手。 |
| **MinIO** | 档案室 | 存你上传的原始文件 |
| **MySQL** | 登记簿 | 记账：有哪些数据集、哪些文件、每个文件处理到哪一步、用哪个模型、谁有权限 |
| **Infinity** | 检索目录 | 切好的块、向量、关键词索引。真正"搜"的地方。 |
| **NATS** | 待办任务板 | 排队系统，下面细说 |
| **kvrocks** | 馆员手边的便签本 | 缓存，下面细说 |

## 两个"配套"容器，用大白话说

**NATS：任务排队**

你导入的时候，一下子丢进去 487 个文件。每个文件都要切块、生成向量、让 LLM 写关键词和问题，前后大约 4,300 次模型调用，花了半小时左右。

这么多活不能让前台一边接待一边干。所以前台只负责把"解析 Luke 15:11–32"这样一张工单贴到任务板上，后台的干活程序从板上一张张取下来做。好处是：

- 前台不会卡住；
- 某个任务失败了可以重试；
- 机器多的话，可以多派几个人同时干。

**kvrocks：便签本（缓存）**

它记那些"常用、要快、丢了也不要紧"的东西：某个任务进行到百分之几、谁正在处理哪个文件（防止两个人重复处理同一个）、刚查过的配置等等。

为什么不全记在 MySQL 登记簿里？因为登记簿是写在硬盘上、要求严谨的，读写比较慢；便签本放在内存里，快得多。kvrocks 和大名鼎鼎的 Redis 用法一样，可以理解成 Redis 的一个替代品。

## 你这个项目里，它们什么时候真正在干活

**导入时（`ingest_ragflow.py`）：6 个全在忙**
1. 文件上传 → 存进 MinIO，同时在 MySQL 里登记；
2. 解析任务贴到 NATS 任务板；
3. 后台取任务干活，进度记在 kvrocks；
4. 切好的块和向量写进 Infinity；MySQL 里的状态改成"完成"。

所以 NATS 和 kvrocks 不是没用，**它们在那半小时导入里是出了力的**。

**对话时（Jesus 检索经文）：基本只剩两个**
你的代码调用 ragflow 的检索接口 → ragflow 到 MySQL 核对一下数据集和权限 → 去 Infinity 搜 → 返回结果。此时 MinIO 和 NATS 基本不参与。

## 一句话总结

RAGFlow 是按"一个团队、成千上万份文档、天天有人上传"的规模设计的，所以分工这么细。你这个项目的数据量只有 487 个段落，而且导入一次就不再变，所以平时真正在用的就是 **ragflow 加 Infinity**，其余几个主要是为了那一次导入而存在的。

你说的 "workflow" 我理解为 RAGFlow，下面三个问题按顺序回答。

# Other questions

## 1. RAGFlow 在工业界真的有人用吗？

有人用，但它**不是业界默认的选择**。先要分清 RAG 工具大致有三类：

| 类别 | 代表 | 特点 |
|---|---|---|
| **代码库**（自己拼） | LangChain / LangGraph、LlamaIndex、Haystack | 提供零件，工程师写代码组装。**大公司自研最常用这一类**，或者干脆只用几个底层组件自己写。 |
| **现成平台**（开箱即用） | **RAGFlow**、Dify 等 | 带网页界面、文档解析、知识库管理，部署起来就能用。 |
| **云厂商托管** | AWS Bedrock Knowledge Bases、Azure AI Search、Google Vertex AI RAG Engine、OpenAI 的 file search | 不用自己运维，数据放在云上。企业里非常常见，尤其是已经深度用某家云的公司。 |

RAGFlow 是 InfiniFlow 开源的，GitHub 上非常火。它的强项是**文档解析**（PDF、表格、版面识别），在**需要私有化部署**、数据不能上云的场景用得比较多，国内企业尤其多。但在北美的大公司里，更常见的是"LangChain 或 LlamaIndex 加一个向量库自己搭"，或者直接用云厂商的托管服务。

---
你记忆里那个"可拖动的可视化组件"，很可能不是 LangChain 本身，而是 **Langflow**（或者类似的 Flowise）。它们是建在 LangChain 之上的图形界面：把 LangChain 的零件做成方块，拖拽连线就能拼出流程。名字很像，很容易混。**LangChain 本身是一个代码库**，要写 Python 或 JS 代码来用。

## LangChain：通用的"LLM 应用积木"

核心思路是把做 LLM 应用常用的东西做成标准零件，用代码串起来：

- **模型接口**：OpenAI、Anthropic、本地模型，换一行代码就能切换；
- **提示词模板**、**输出解析**；
- **检索相关**：文档加载器、切块器、embedding、向量库（Pinecone、pgvector 等几十种都有现成适配）、检索器；
- **工具调用和 agent**。

后来又衍生出两个兄弟产品：
- **LangGraph**：把复杂的 agent 流程写成"状态图"，比如先查资料、判断够不够、不够就再查，还能循环；
- **LangSmith**：一个网页平台，用来追踪每一步调用、做评测。

**定位：什么 LLM 应用都能搭，RAG 只是其中一种。** 优点是适配器多、生态大；缺点是抽象层多，出了问题不好查，老版本的 API 也改过好几次。

## LlamaIndex：专门做"数据接入加检索"

它的核心关注点是**怎么把你的数据喂给 LLM**，所以在 RAG 这条链上做得更深：

- **数据接入**：LlamaHub 上有几百种连接器（Notion、Slack、Google Drive、数据库……），还有专门解析复杂 PDF 的 **LlamaParse**；
- **切块和索引**：把文档切成节点（node），支持父子块、层级摘要等多种索引结构；
- **查询引擎**：封装好了"检索 → 重排 → 合成答案"的整条流程，几行代码就能搭出一个 RAG；
- 后来也加了 agent 和 workflow 的能力。

**定位：以数据和检索为中心的 RAG 专用框架。**

## 放到你的项目里对比

| 环节 | 你现在用的 | 用 LangChain | 用 LlamaIndex |
|---|---|---|---|
| 读 BSB、按段落切块 | 自己写的 `pericopes.py`，加 RAGFlow 解析 | 文本切分器 | 节点解析器（父子块是现成的） |
| 向量和全文索引 | Infinity（RAGFlow 自带） | 接一个向量库，比如 pgvector | 同样接向量库，或用它自带的 |
| 混合检索和重排 | RAGFlow 内置，加 Voyage | 组合检索器，加 Voyage 重排组件 | 检索器加后处理器（重排） |
| 语音 agent 和工具调用 | OpenAI Agents SDK | 也能做，但你已经有了 | 不是它的强项 |

三者的本质区别：

- **RAGFlow 是一个"成品平台"**：要起服务，有网页界面，零件都内置好了。
- **LangChain 和 LlamaIndex 是"代码库"**：直接 `import` 到你的 Python 程序里用，不用起一堆容器。像你这个规模，用 LlamaIndex 写的话，很可能就是一个 Python 文件，加一个向量库，或者干脆连向量库都不要，只用本地文件。

**面试一句话版本**：

> "LangChain is the general toolkit for LLM apps, with LangGraph for agent flows. LlamaIndex is data-first: ingestion, indexing and query engines for RAG. RAGFlow is a deployable RAG platform with its own UI and storage. I used RAGFlow, but the pipeline stages map one-to-one onto either library."

---



**对你面试的意义**：工具名字不是重点，面试官关心的是你懂不懂每个环节，这些概念换哪个框架都通用：
- 结构化切块和父子块
- 混合检索
- 重排
- 拒答阈值
- 分阶段评测

你还有一个很好的讲法："我先手写了一版基线，再换成框架，用评测数据对比了两者。"

## 2. 为什么用 MySQL，不用 SQLite？

两者都是关系型数据库，区别在于**谁来访问它**：

- **SQLite 就是一个文件**，嵌在一个程序里用。同一时刻基本只能有一个写入者，写的时候整个库都被锁住。它也没法让别的容器通过网络来连。
- **MySQL 是一个独立运行的服务**。很多程序可以同时通过网络连上来读写，它负责协调，还支持备份、扩容。

RAGFlow 的网页界面、API、多个后台解析程序都可能**同时**改状态，比如"文件 A 解析完成""文件 B 开始解析"，而且它们分在不同的容器里。SQLite 应付不了这种多方同时写、还跨容器的情况，所以 RAGFlow 选了服务型数据库。从它的配置看，好像也可以换成 PostgreSQL，但没有 SQLite 这个选项。

**对你这个单人项目来说**，SQLite 本来就够用了，MySQL 确实是大材小用。不过这是框架的设计决定的，不是你能选的，这也是用框架要付的代价。

## 3. LLM Wiki 和正经 RAG，什么时候用哪个？

两者的根本区别在于：**知识是提前整理好，还是提问时现查**。

- **LLM Wiki**：先让 LLM 把所有资料读一遍，**提前写成**一套整理好、互相链接的笔记，相当于一本手册。提问时，agent 去翻这本手册里相关的几页来回答。
- **RAG**：资料**原样保留**，只是切块、建好索引。每次提问时，**现场检索**原文片段交给模型。

| 情况 | 更适合 |
|---|---|
| 资料不多（几百份文档以内）、变化慢 | **Wiki** |
| 答案需要跨多份资料综合归纳（"我们的部署流程是怎样的"） | **Wiki**：提前整理过，回答更连贯 |
| 希望人也能直接读、能检查的知识 | **Wiki** |
| 资料量很大（几万份以上），或者天天更新 | **RAG**：每次更新都重新编一遍手册太贵 |
| **必须原文引用、给出准确出处**（法律、合规、医疗、**经文**） | **RAG**：手册是 LLM 改写过的，原文已经丢了 |
| 不同人能看到的文档不一样（权限控制） | **RAG**：可以按文档过滤 |
| 问的是零散的细节（某份合同第几条） | **RAG**：手册在整理时很可能把这些细节省略掉 |

Wiki 有两个固有的风险：
1. **整理时出的错会一直留着**：LLM 在整理阶段写错一句，之后所有回答都会沿用这个错。
2. **资料会过时**：原始资料变了，手册不会自动跟着改。

所以实际中常见的是**两者结合**：手册负责把握全局，RAG 负责查原文和细节。

**回到你这个项目**，经文必须逐字引用并给出处，所以 RAG 是绕不开的。这也解释了你之前为什么觉得 Wiki 那版"太死板"：它把经文提前整理成了固定的说法，失去了按当下这个问题现场去查的那股新鲜感。
