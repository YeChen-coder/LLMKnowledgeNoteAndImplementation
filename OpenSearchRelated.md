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
    }
  }
}
```

逐行解释：

**`PUT /gospels`**：建一个叫 `gospels` 的 index。

**`settings`**：index 本身的设置。
- `number_of_shards: 1`：1 个 primary shard。数据很少，不用拆。
- `number_of_replicas: 0`：不要副本，因为只有一个 node，副本没地方放（就是你刚才在表里看到的 `rep` 那一列）。
- `index.knn: true`：**允许这个 index 做向量检索**。这个开关建好以后就不能再改，所以现在先打开。向量字段第 4 步再加，后加字段是允许的。

**`mappings.properties`**：每个字段的类型。
- `reference`：比如 `"John 11:35"`，`keyword` 类型，整个当成一个值。
- `book`：比如 `"John"`，`keyword` 类型，用来筛选。
- `chapter`、`verse`：`integer`，数字。
- `text`：经文正文，`text` 类型，会被拆成单词。
  - `"analyzer": "english"`：用英文规则来拆词。**analyzer**（分析器）决定一段文字怎么拆成搜索用的词。english 分析器会把 `loved`、`loves` 都还原成 `love`，所以搜 love 也能搜到 loved。

成功的话，右边会显示 `"acknowledged": true`。跑完把结果贴给我，下一步往里放 document。
