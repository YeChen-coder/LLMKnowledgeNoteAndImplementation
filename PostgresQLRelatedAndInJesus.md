
pgvector is an extension of PostgreSQL that stores vectors in a relational database.

用 pgvector 的话，全文检索并不需要另外搭。PostgreSQL 本身就自带全文检索：tsvector 加 GIN 索引。所以向量和全文完全可以放在同一个 Postgres 库里，一条 SQL 就能做混合检索。我上一条说"通常要另外搭"，这一点只对 Pinecone 成立：它主要存向量，要做关键词检索，得自己另外生成稀疏向量传进去。

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



