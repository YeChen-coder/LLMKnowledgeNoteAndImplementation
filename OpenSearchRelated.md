在现在这么一个 AI 辅助学习的年代，什么事不都是一晚上的事吗？一晚上搞不定就两晚上，两晚上搞不定就三晚上，三晚上真的就到头了，差不多了。


# 本地先跑个OpenSearch Docker

### 为什么要先定 mapping

OpenSearch 存一个字段之前，要知道**这个字段该怎么处理**。同样是一串字，处理方式可能完全不同：

| 字段 | 内容 | 希望怎么用 | 该用的类型 |
|---|---|---|---|
| 经文正文 | `Jesus wept.` | **拆成单词**，按关键词搜索、BM25 打分 | `text` |
| 书名 | `John` | **整个当成一个值**，用来精确筛选："只要约翰福音" | `keyword` |
| 第几章 | `11` | 当成数字，能比大小："第 5 到 7 章" | `integer` |
| 向量 | `[0.12, -0.03, ...]` 1024 个数字 | 建向量索引，按意思找 | `knn_vector` |

mapping 就是提前告诉它：**哪个字段是什么类型**。跟数据库里 `CREATE TABLE` 时要写每一列的类型是一回事。

### 不定会怎样

OpenSearch 会**自己猜**，这叫 **dynamic mapping**（动态映射）。猜得大多还行，但有两个坑：

1. **向量会被猜错。** 一串数字它会当成普通的数字数组，而不是 `knn_vector`，那就**做不了向量检索**。
2. **类型定了就不能改。** 字段类型一旦确定，就改不了，只能新建一个 index，把数据全部重新导一遍（这叫 **reindex**）。

所以正式项目里，都是先写好 mapping，再放数据。

### 建 index 的命令

把下面整段贴进 Dev Tools 左边，点 ▶：

```
PUT /gospels
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "index.knn": true
  },
  "mappings": {
    "properties": {
      "reference": { "type": "keyword" },
      "book":      { "type": "keyword" },
      "chapter":   { "type": "integer" },
      "verse":     { "type": "integer" },
      "text":      { "type": "text", "analyzer": "english" }
- 这个其实还蛮有必要说一下的：Analyzer 就是把一段文字变成搜索用的词的规则，拆出来的每个词就是 Token。

这件事情发生在两个时候：

1. 存的时候：文档进来，把 Text 字段拆成 Token，再去建立 Index。
2. 搜的时候：搜的关键词也是用同一套规则去拆开（当然得同一套规则，不同规则怎么匹配得上）。

Analyzer 主要分几种：

1. 默认的（Standard）：简单粗暴，按单词分开，转小写并去除标点，是非常基础的做法。
2. English（我们这里用的）：在 Standard 的基础上去掉了无意义的常见词（比如 the、and），并做词根还原，比如把 loved 改成 love，worried 改成 worry。
3. Whitespace：只按空格切分，不转小写，标点也会保留。
4. Simple：跟 Standard 差不多，按非字母的位置切分，但不转小写。
5. Keyword：完全不拆，整句话直接作为一个 Token。

刚才提到的那些需要去掉的常见词，正式名字叫 Stop Words（停用词）；词根还原叫做 Stemming

因为中文根本就不是按照空格来区分的，所以对于中文项目，一般都是装 SmartCN 或者 IK 这类分词插件。

可是 AWS 托管的 OpenSearch 不能随便装插件，只能从它支持的列表里边选，就没办法。本地的 OpenSearch 其实可以随便搞，那个无所谓，但有些时候它就是没办法嘛
    }
  }
}
```

逐行解释：

**`PUT /gospels`**：建一个叫 `gospels` 的 index。

**`settings`**：index 本身的设置。
- `number_of_shards: 1`：1 个 primary shard。数据很少，不用拆。- 可以先补充一下，因为它涉及到了 shard 的概念。

其实它跟其他大数据组件的思路都是一样的，主要有几个目的：

1. 存储容量：万一数据太多，一台机器、一个 node 里面装不下，那就分片。把整个数据切成几块，一个 node 装一块，这样就能放得下了，也方便集群做横向扩展。
2. 提高检索速度：也就是 scatter-gather。当数据量很大的时候，让每个机器各自去搜自己持有的 shard 里面相关的内容，然后再把结果合并返回。这种并行处理肯定比一台机器单独一点一点去查要快得多。
3. 容灾备份：比如我现在有两个 shard，两台机器 node A 和 node B。node A 在持有 shard 0 的同时，也会持有 shard 1 作为 node B 的 replica；对于 node B 也是同理，在持有 shard 1 作为 primary 的同时，也会持有 node A 那边 shard 0 的 replica。
- `number_of_replicas: 0`：不要副本，因为只有一个 node，副本没地方放（就是你刚才在表里看到的 `rep` 那一列）。
- `index.knn: true`：**允许这个 index 做向量检索**。这个开关建好以后就不能再改，所以现在先打开。向量字段第 4 步再加，后加字段是允许的。

**`mappings.properties`**：每个字段的类型。
- `reference`：比如 `"John 11:35"`，`keyword` 类型，整个当成一个值。
- `book`：比如 `"John"`，`keyword` 类型，用来筛选。
- `chapter`、`verse`：`integer`，数字。
- `text`：经文正文，`text` 类型，会被拆成单词。
  - `"analyzer": "english"`：用英文规则来拆词。**analyzer**（分析器）决定一段文字怎么拆成搜索用的词。english 分析器会把 `loved`、`loves` 都还原成 `love`，所以搜 love 也能搜到 loved。

这边的 index 可以约等于数据库里的一张表，mapping 就可以按照建表时写的表结构（也就是 schema）去理解。

我这里一般指的是关系型数据库，因为我对 MySQL 比较熟所以拿来比照。对应关系差不多是这样：
• MySQL 的 table 就是我们这边谈的 index（表）
• schema 就是我们谈的 mapping
• 一行就是我们这边的一个 document
• 一列（它有很多列）就是我们这边的一个 field

因为 OpenSearch 其实是在我本地开的一个 localhost 网页，在关系型数据库那边往里面插数据是 insert，而在 OpenSearch 里面，作为一个网页存在，你正常去 PUT 就行。



{

  "acknowledged": true,
  
  "shards_acknowledged": true,
  
  "index": "gospels"
  
}

就这个样子，就是给他写入一行，然后这些什么 reference 啊、what book、什么 chapter 啊、什么 text 呀、什么 title 呀，就是它里边的这些 field

<img width="825" height="477" alt="image" src="https://github.com/user-attachments/assets/3efa50ee-b56f-4811-ac37-c433ca7378f1" />

实际上真用起来一般都是用 bulk API，就像这样的：第一行那个 index 说明要往哪个 document 里面写，第二行就是具体要写的内容

<img width="1236" height="567" alt="image" src="https://github.com/user-attachments/assets/45ba982d-9166-4f01-8399-b49f32497c63" />

## OpenSearch match = BM25

<img width="1056" height="755" alt="image" src="https://github.com/user-attachments/assets/05ebd34f-35be-48f9-9435-e3382efbfef0" />

以上图为例：

OpenSearch 里的 match 默认就是 BM25。它给我匹配出来三个结果，分数从高到低排，这里的 score 就是 BM25 的分数，source 就是它的原始内容。

BM25 就是 TF-IDF 再加点其他配置什么的，在别的文章里写过无数次了。虽然我每次都忘，但大概知道是什么意思就行了。抓大放小，我又不是搞科研的.

它的坏处就是它不懂 semantic。因为我们刚才说了，我们这边的 analyzer 用的是 English 的，它能做到的是把 worried 和 worry 识别成同一件事，但并不知道 tired 和 worry 也是同一件事，因为它不是基于语义的

## embedding

在本地装了一个很小的模型，只输出 384 维度的向量。

干的事也没什么特别的：

1. 因为它本身是个新的 field，所以先得在 index 里面多加一列
2. 加完之后，得加一个 processor 在 ingest pipeline，把我们现在用什么引擎、向量具体怎么算都放进去

这样每次新文档 ingest 进来之后（现在先不管旧的，processor 不管旧文档），就会自动跑一遍，把这个向量给搞出来;那些他还得手动搞。你得让他先把他们自己的 text embedding 字段（也就是那个 384 维的数据）导出来，到这就结束了。


---
# AWS OPENSEARCH Service

<img width="1008" height="467" alt="image" src="https://github.com/user-attachments/assets/dfc1c29a-bb87-4da8-82d9-bfd9c0a97a49" />

# AWS OpenSearch 一些名词解释

---

## 1. 集群里的几种 node，以及高可用

| 角色 | 正式名字 | 干什么 |
|---|---|---|
| 存数据、跑查询 | **data node** | 你现在那台 t3.small 就是 |
| 管集群 | **dedicated cluster manager node**（以前叫 master node） | 只管元数据：有哪些 index、shard 放在哪。不存数据 |
| 只分发请求 | **coordinator node** | 大集群里专门做 scatter-gather 的合并工作 |

**cluster manager 一般开 3 台。** 选主要过半数（**quorum**）同意，3 台挂掉 1 台还剩 2 台，仍然过半。如果只有 2 台，一断网两边各自以为自己是主，就会出现 **split brain（脑裂）**。所以都是奇数台。


补充一下：在小集群里（比如像我这种情况），不一定有专门的 cluster manager。像我现在这里边只用了一个 t3.small，它就相当于一个人兼两份活，既是 data node，也是 cluster manager。

上面第二行写的 dedicated，指的就是专门拿几台机器来做这件事情，一般是集群大的时候才这么搞

**Multi-AZ with Standby**：数据分布在 3 个机房，其中 1 个机房的节点平时只当备用，不接流量，出事时秒级切过去。这是 AWS 的 99.99% SLA 方案。你刚才选的 "without standby" 就是不要这个，所以也用不了 t3。

## 2. Blue/green 部署（很容易踩坑）

在 AWS 上改很多配置，比如**换机器类型、升级版本**，AWS 不会原地改，而是：

> 新建一整套集群（green）→ 把数据全部复制过去 → 切换流量 → 删掉旧的（blue）

后果：
- 数据量大时可能要**好几个小时**
- 期间**两套集群同时存在**，CPU 和网络压力大
- 集群本来就很忙的时候做，可能会卡住

面试常问："生产环境怎么安全地升级 OpenSearch？" 答案就围绕这个：挑低峰期、先确认 JVM (补充：JVM 是 Java Virtual Machine，就是 Java 虚拟机。因为 OpenSearch 是 Java 写的，Java 程序不直接跑在 Operating System 上，而是跑在 JVM 里面) 和存储有余量、先打快照。

再补充一下，既然谈到了 JVM，那就顺手提几个跟它相关的名词：

1. Heap（堆内存）：
指的是 JVM 管理的那块内存，Java 程序的对象就放在这里。比如说这里我开的是 -Xms2G -Xmx2G，意思就是 Heap 最小是 2G，最大也是 2G。这个之所以重要，是因为 OpenSearch 的瓶颈是在内存而不是存储，所以必须关注。
2. JVM Memory Pressure：
意思是 Heap 用了百分之几。如果只用了 1G，那就是 50%，很简单。
3. Garbage Collection（GC）：
JVM 定期清理不用的对象，Heap 越满 GC 越频繁。它会把不用的那些块清掉，是很常规的 operating system 概念。但每次清理程序就会卡一下，不少人吐槽过 Java 的这个特性，说每次要干点什么事程序就卡了。

总之，如果 JVM Memory Pressure 那边的压力很高，也就是占用的百分比快满了，整个集群就会变慢。因为老是卡一下卡一下的，那确实就慢了，很正常

## 3. 权限有三层，403 错误常出在第三层

请求进来依次过三道门：

1. **网络**：公开访问，还是 VPC + 安全组
2. **Access policy**：基于资源的 IAM 策略，决定哪些 AWS 身份能碰这个 domain
3. **Fine-grained access control（FGAC）**：OpenSearch 自己的用户和角色系统，能细到某个 index、某个字段、某些文档（**document-level / field-level security**）

**最常见的坑**：IAM 已经允许了，还是报 **403**。因为 FGAC 那层不认识你的 IAM 身份，要在 OpenSearch 里把 IAM ARN **映射（role mapping）**到一个 OpenSearch 角色上。 - 这边补充一下，就是不用了。

我用的那个 root 账号，就是用 SigV4（AWS CLI）登录的。我确定我什么都没干，但它其实已经在这边的 All Access 权限（也就是 map_user）里边有了。

就看下图吧。

<img width="1090" height="598" alt="image" src="https://github.com/user-attachments/assets/c5d0b864-3881-487c-84f6-96be0e0ff514" />

再补充一下：因为我这个 IAM 身份的 root 是创建域名的账号，所以讲道理，我刚才的话可能说得太绝对了。

顺手说一下，这里面其实有两种身份：

1. user：精确到某一个具体的 IAM 身份
2. backend role：常用来映射一个 IAM role，所有扮演这个 role 的程序都会获得对应权限

## 4. 要监控的指标，以及会把集群"锁死"的几个阈值

在 CloudWatch 里最常看的：

| 指标 | 看什么 |
|---|---|
| `ClusterStatus.red / yellow` | 健康颜色 |
| `JVMMemoryPressure` | JVM 内存压力。建议保持在 75% 以下。**到 92% 并持续 30 分钟，AWS 会拒绝所有写入** |
| `FreeStorageSpace` | 剩余磁盘 |
| `ThreadpoolWriteRejected` / `SearchRejected` | 请求排不上队、被拒绝了 |

磁盘有三条**水位线（watermark）**，默认值：
- **85%（low）**：不再往这个 node 分配新 shard
- **90%（high）**：开始把 shard 搬走
- **95%（flood stage）**：**index 变成只读**，所有写入失败

node 是一台独立的计算资源，也就是一个 instance，一个 node 上可以有很多个 shard。固定的是每个 index 有几个 primary shard，而不是每一个 node 上放哪几个 shard。

shard 放在哪台 node 上是由 cluster manager 决定的，所以它是可以挪动的，这种挪动就叫 shard allocation 和 rebalancing，目的就是把负载平均分配一下。常见的触发场景有这么几种：

1. 加节点：比如原来有 5 台 node，现在预算批下来加了台 node，就会把原来 5 台 node 上的部分 shard 挪过去，让大家负载更平均
2. 节点故障：如果一个 node 挂了，它上面的 replica 就会升级成 primary，并在别的 node 上补一份新的 replica (补充一下：shard 是 primary shard 还是 replica shard，跟 node 没有关系，node 从来不拥有任何数据。

Primary 和 replica 是针对 index 的具体某块数据来说的：
• Primary shard：某块数据的本体，写入时先写它。
• Replica shard：针对 primary shard 同一份数据的副本。

正本和副本必须放在不同 node 上，不然容灾就没法做了；但这跟哪份数据归哪个 node 同样没有从属关系。

举个例子：比如有三个 node（Node A、Node B、Node C），上面分别拥有的正本是 P0、P1 和 P2。

<img width="681" height="237" alt="image" src="https://github.com/user-attachments/assets/306b981c-d150-41aa-bf73-4ca5e40f5d90" />


现在 Node A 突然没了（也就是 Primary 0 没了）：
处在 Node B 上的 Replica 0 必须立刻升级成正本 Primary 0，以保证这块数据依然可读可写、没有丢失。

但这样会直接导致两件事： Replica 0 升级成 Primary 0 之后，数据 0 就没有副本了。+ 原本在 Node A 上的 Replica 2 随着 Node A 挂掉，导致处在 Node C 上的 Primary 2 也没有副本了。

也就是说，现在必须补上的是 Replica 0 和 Replica 2。当前能用的只有 Node B 和 Node C，在给它们补齐副本的同时，还要遵循正本和副本不能放在同一台设备上的原则，否则就无法容灾了。就这个意思)

3. 新建 index：比如日志场景下，每天都会产生新的 index，每天都会有新的 shard 需要找地方分配 (补充一下：index 是真的数据，不是 schema。schema 在 OpenSearch 里叫 mapping，而 index 是 mapping 加上 settings，再加上里面所有的 document。它是一个更完整、里面真有数据的东西)
4. 磁盘水位线限制：当某台 node 的磁盘快满时，触发水位线机制，就不会再往这台 node 里分配 shard 了

整个分配是一个动态过程，并不是一开始定死每个位置填满 70% 就不管了。因为数据一直在增长（比如日志每天都在增量），并不是固定体量的数据。此外，后台合并 segments 时会临时多占一份磁盘空间，所以平时需要留足余量。

## 5. 向量检索在 AWS 上为什么贵：内存

HNSW 索引要**整个放在内存里**才快。AWS 文档给的估算公式：

> 内存 ≈ 1.1 × (4 × 维度 + 8 × M) × 向量数量（字节）

M 是 HNSW 每个点连多少邻居，默认 16。代入你的数据：
- 4255 个 × 1536 维 ≈ **29 MB**，小意思
- 换成 **1000 万个**同样的向量 ≈ **69 GB**，要好几台大内存机器

所以大规模向量检索的常用省钱手段：
- **量化（quantization）**：每个数字从 4 字节压到 2 字节（fp16）、1 字节（byte），甚至 1 个比特（binary），内存省几倍到几十倍，精度损失一点

因为之前一直对量化（quantization）这件事情有一点疑惑，就顺手在这儿说了吧：

它的维度数量其实是不变的。原本 1536 维的向量，仍然是 1536 个数字，用来标志它在 1536 维空间里所在的那个点。哪怕做了 quantization，它仍然是 1536 个数字，只是每个数字用更少的位数来存，变得没那么精确了。

维度仍然是这个维度。比如拿 X、Y 轴来说，原来我们说它的 X 轴是 1.734、Y 轴是 3.528；做了 quantization 之后，X 可能就是 1，Y 就是 3。

维度完全没有变，但它标志所在位置的精度确实变低了;

那就得牵扯到另外一个问题：为什么做了这个压缩还能用？

因为检索的时候看的是谁离得最近，它关心的是排名，而不是距离的精确数值。像这种比较小的四舍五入，其实很少会改变排名，离得近的依然离得近。这边最常见的做法是先用压缩版快速找到前 100 个，然后再用原始精度对这 100 个重新算分，这一步叫 rescoring。毕竟都取了前 100 个，剩下的那些其实就不太可能有多大潜力成为候选了。

把这个放到本地跑大语言模型（large language model）也是同理。用了量化（quantization）之后，模型的体量确实压下来了，能让 GPU 不再疯狂报 Out of Memory。可是在这种情况下，推理虽然能正常跑，训练却不行。因为训练涉及反向传播（backpropagation），需要把权重一点点微调。但当精度太低的时候，这么小的改动就跟没有一样，相当于没改，直接被四舍五入成 0 了。比如 3 加上 0.0001，如果限制了每个数字只能表示这么几位，后面的 0.0001 就根本存不下，加了等于没加。

再多说一句，为什么 QLoRA 这类方法能在已经量化的模型上训练？那是因为它冻结了量化后的原模型，另外加了一小块高精度的参数来训练。这样在做 backpropagation 时，改动落在高精度的这块参数上，权重就能够真正更新了

- **disk-based vector search**：主索引放磁盘，内存里只放压缩版

这是面试里聊向量库时很有分量的一点：**向量检索的成本瓶颈是内存，不是磁盘。**

## 6. 存储分层：日志场景的核心

| 层 | 存在哪 | 特点 |
|---|---|---|
| **Hot** | data node 的 EBS | 快，贵 |
| **UltraWarm** | S3，前面加缓存 | 只读，便宜很多 |
| **Cold** | S3 | 要先 attach 才能查，最便宜 |

再配一个 **ISM（Index State Management）** 策略，自动执行"7 天后挪到 warm，30 天后挪到 cold，90 天后删除"。日志场景里这是标配。

你刚才在控制台看到的 **OpenSearch Optimized（OR / OM / OI 系列）** 机器是另一个思路：数据持久地写在 S3 上，节点之间用**段复制（segment replication）**同步，写入吞吐更高，适合大规模写入。

## 7. Domain 还是 Serverless

| | Domain（你现在建的） | Serverless |
|---|---|---|
| 管什么 | 自己选机器、shard、版本 | 都不用管，按 OCU(OpenSearch Compute Unit，OpenSearch 计算单元) 自动伸缩 |
| 最低成本 | t3.small 一个月约 $27 | 一个月一百多美元起 |
| 限制 | 少 | 部分 API 和插件不支持 |
| 适合 | 稳定负载、要细调 | 负载不稳定、不想运维；**Bedrock Knowledge Bases 的默认向量库** |

Serverless 建的不叫 index，叫 **collection**，分三种类型：search、time series、vector search。

OCU（OpenSearch Compute Unit，OpenSearch 计算单元）：Serverless 的计费单位。一个 OCU 大约是 6 GB 内存加上相应的 CPU 和临时存储。用量越大，AWS 自动给你加的 OCU 越多，按"OCU × 小时"收费。

## 8. 周边集成（知道名字就行）

- **自动快照（automated snapshot）**：每小时一次，存在 AWS 管的 S3 里，保留 14 天，**免费**。要迁移或做灾备，用手动快照存到你自己的 S3
- **OpenSearch Ingestion（OSI）**：托管的数据导入管道（底层是开源的 Data Prepper），可以从 S3、Kafka、DynamoDB 往里灌数据
- **ML connector**：之前讲过，用 ML Commons 去调 Bedrock 上的 embedding 和 rerank 模型

谈到这儿，还有一个事情。

在本地的 OpenSearch 里（具体型号比如 all-MiniLM-L6-v2，它是 OpenSearch 官方提供的 pre-trained model 之一），官方本来就提供了其他几个 embedding 模型、sparse embedding 模型，以及 cross-encoder 模型（cross-encoder 是用来做重排序 rerank 的，而 sparse embedding 是一种介于关键词和向量之间的检索方式）。官方提供的这些模型，其实都可以直接在集群里面跑，不需要调用外部 API。

可是 AWS 托管版不能这么做。托管的 domain 一般不让用户把模型装进集群跑，而是需要用他们的 connector 去调用外部模型，比如 Bedrock、SageMaker，或者 OpenAI 等第三方。

如果连 connector 都不想自己连，Serverless 那边还有一个自动语义增强（Automatic Semantic Enrichment）的功能，开启之后它自己就能去做语义检索，就不用连模型了，这也就是那个 Semantic Search OCU 的收费项。

---
# AWS Opensearch 结果

<img width="877" height="470" alt="image" src="https://github.com/user-attachments/assets/f38c20b2-3bc8-448c-8b23-a0aed7463c7e" />


AWS 上和本地的测评结果出来了，其实准确度都一样（那确实得一样啊。这次在 AWS 上的 OpenSearch 根本就没有做 embedding，而是把之前已经算好的向量直接传过去的。也就是说，传过去的 document 不是人类阅读的文字text，而是直接对应的 1000 多个维度的向量。搜索的时候也都一样用 k-NN 查询嘛，确实得一样，不一样才见鬼了），但速度上有差别：本地是 10 毫秒，AWS 上面是 200 到 300 毫秒。

这个没办法，主要是因为网络的 latency，请求要从我这儿发到美国东部机房再回来，所以才导致的。

不过如果正式部署的时候，把应用程序和 OpenSearch 放在同一个 zone 里，或者甚至直接放在同一个 VPC 里边，那网络开销自然就没了，也行吧

# AWS ML connector to Bedrock

模型用 Bedrock 上的 Amazon Titan Text Embeddings V2（输出 1024 维）

出来的结果就是 Titan 明显 performance 要比 OpenAI 低一些哈，它这个模型在这个任务上确实弱一些哈

其实下图有点容易让人误解，但他这边那个 OpenAI 3-small 的意思，还是把我电脑上原来已经用它导出来的这波向量作为 document 送过去的，并没有在 OpenSearch service 上再重新找 OpenAI 3-small 的 embedding 模型搞一遍； 再补充一下哈，这里实际上用 A 方法测的时候，因为我本地本来就有 OpenAI 的 API key，所以是在本地先拿 query 送去 OpenAI 的 embedding 模型拿到向量，然后再把拿到的向量送去 AWS 那边去对。

为什么要这么折腾呢？是因为 OpenAI 的很多模型虽然上了 Bedrock，但 embedding 模型没上。上了 Bedrock 的基本上全是生成文字、输出类型是 text 的模型。

Bedrock 上能用的 embedding 模型只有 Amazon 自己的（比如 Titan 系列、Multimodal Embeddings）、Cohere 的，还有 Twelve Labs（那个是视频用的），所以没有直连这么方便，才非得脱裤子放屁折腾这么一趟。

不过无所谓，本来就只是个测试而已。Cohere 的 Embed v4 在 Bedrock 上口碑其实是最好的文字 embedding 之一，本来觉得不用测、差不多得了，算了，顺手干一遍吧

<img width="900" height="457" alt="image" src="https://github.com/user-attachments/assets/8038446a-16f8-4d27-8c39-a26d9a2c579c" />

<img width="605" height="221" alt="image" src="https://github.com/user-attachments/assets/47969caf-c6c8-4ad8-9a5b-a6224ea7a5ed" />

<img width="886" height="456" alt="image" src="https://github.com/user-attachments/assets/4eadd446-6f9e-4398-ae55-70bfdc621577" />

反正跑一圈下来，引擎换来换去（从 RAGFlow 换成本地 OpenSearch，又换到 AWS OpenSearch），其实影响一般，没那么大，纯向量一直都是 84% 左右。

但是换了 embedding 模型之后掉得非常大，这说明 embedding 模型非常重要。

另外，改写之后的问题是能够弥补弱模型的。从这张图来看，改写之后 Titan 和 OpenAI 的差距从 19 个点缩小到了 7 个点

## embedding model: openAI 3 small VS Titan V2 VS Cohere Embed v4

<img width="881" height="455" alt="image" src="https://github.com/user-attachments/assets/72b0985c-83a8-4255-b2fe-9fe67e0a040f" />

当前这边其实都没用 rerank 嘛。你看第二行，纯向量加改写就能达到 95% 的成绩，甚至都没用 rerank；而 RAGFlow 那边的 95%，甚至还得叠一个 Voyage rerank 才能达到。

所以这就说明，Cohere Embed v4 真的很不错了。

不过讲道理，rerank 这层加不加对效果的影响，还真的是两说。项目里搞那个纯属炫技，真的纯炫技。落到实处还是得看具体效果，就别折腾了

<img width="882" height="416" alt="image" src="https://github.com/user-attachments/assets/b8d29bab-77f6-47bd-ab17-a5eb92180db4" />

Cohere 这边是少了 enrichment 和 rerank 两样，还是追平了。
