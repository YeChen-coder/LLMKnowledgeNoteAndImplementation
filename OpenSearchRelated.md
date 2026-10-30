在现在这么一个 AI 辅助学习的年代，什么事不都是一晚上的事吗？一晚上搞不定就两晚上，两晚上搞不定就三晚上，三晚上真的就到头了，差不多了。

这个年代天天东奔西跑，很多东西要是没来得及学，那就真的不用学了，最后直接拿人家的来用就行。尤其对于不是做科研而是做应用侧的人来说，各方面该干什么就干什么

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

它的坏处就是它不懂 semantic。因为我们刚才说了，我们这边的 analyzer 用的是 English 的，它能做到的是把 worried 和 worry 识别成同一件事，但并不知道 care 和 worry 也是同一件事，因为它不是基于语义的

## embedding

在本地装了一个很小的模型，只输出 384 维度的向量。

干的事也没什么特别的：

1. 因为它本身是个新的 field，所以先得在 index 里面多加一列
2. 加完之后，得加一个 processor 在 ingest pipeline，把我们现在用什么引擎、向量具体怎么算都放进去

这样每次新文档 ingest 进来之后（现在先不管旧的，processor 不管旧文档），就会自动跑一遍，把这个向量给搞出来;那些他还得手动搞。你得让他先把他们自己的 text embedding 字段（也就是那个 384 维的数据）导出来，到这就结束了。




