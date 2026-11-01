> English version of [OpenSearchLocal+AWSRelated.md](../OpenSearchLocal+AWSRelated.md). Original notes are written in Chinese/English mix.

In this age of AI-assisted learning, isn't everything a one-night job? If one night isn't enough, two nights; if two isn't enough, three. Three nights really is the limit; that's about it. — Done.


# First, run an OpenSearch Docker locally

### Why define the mapping first

Before OpenSearch stores a field, it needs to know **how that field should be handled**. The same string of characters might be handled completely differently:

| Field | Content | How we want to use it | Type to use |
|---|---|---|---|
| Verse text | `Jesus wept.` | **Split into words**, keyword search, BM25 scoring | `text` |
| Book name | `John` | **Treat the whole thing as one value** for exact filtering: "only the Gospel of John" | `keyword` |
| Chapter | `11` | Treat as a number, comparable: "chapters 5 to 7" | `integer` |
| Vector | `[0.12, -0.03, ...]` 1024 numbers | Build a vector index, search by meaning | `knn_vector` |

The mapping tells it up front: **what type each field is**. It's the same as writing each column's type in a database `CREATE TABLE`.

### What happens if you don't define it

OpenSearch **guesses on its own**, which is called **dynamic mapping**. The guesses are mostly OK, but there are two pitfalls:

1. **Vectors get guessed wrong.** It treats a string of numbers as an ordinary number array rather than `knn_vector`, so **you can't do vector search**.
2. **Types can't be changed once set.** Once a field's type is set, it can't be changed. You can only create a new index and re-import all the data (this is called **reindex**).

So in real projects, you write the mapping first, then load the data.

### Command to create the index

Paste the whole block below into the left side of Dev Tools and click ▶:

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
- This is actually worth explaining: an Analyzer is the set of rules that turns a piece of text into words for search, and each word it produces is a Token.

This happens at two points:

1. At storage time: when a document comes in, the Text field is split into Tokens, then the Index is built.
2. At search time: the search keywords are split with the same rules (it has to be the same rules, of course; different rules couldn't match).

The main kinds of Analyzer:

1. Default (Standard): simple and blunt; splits by word, lowercases, and strips punctuation. A very basic approach.
2. English (what we use here): on top of Standard, removes meaningless common words (like the, and) and does root reduction, e.g. loved to love, worried to worry.
3. Whitespace: splits only on spaces, no lowercasing, punctuation kept.
4. Simple: similar to Standard, splits at non-letter positions, but doesn't lowercase.
5. Keyword: doesn't split at all; the whole sentence is one Token.

Those common words to be removed are formally called Stop Words; root reduction is called Stemming.

Chinese isn't separated by spaces at all, so Chinese projects usually install a tokenizer plugin like SmartCN or IK.

But AWS-managed OpenSearch doesn't let you install arbitrary plugins; you can only pick from its supported list, so there's no way around it. Local OpenSearch lets you do whatever you want, no problem there, but sometimes it just isn't possible.
    }
  }
}
```

Line by line:

**`PUT /gospels`**: create an index called `gospels`.

**`settings`**: settings for the index itself.
- `number_of_shards: 1`: 1 primary shard. There's very little data, no need to split it. - Let me add a note here, since this involves the concept of a shard.

It's the same idea as in other big data components, with a few main goals:

1. Storage capacity: if there's too much data to fit on one machine, one node, you shard it. Cut the data into pieces, one node holds one piece, and then it fits. This also makes horizontal scaling of the cluster easy.
2. Faster retrieval: this is scatter-gather. When there's a lot of data, each machine searches the relevant content in the shards it holds, and the results are merged and returned. This parallel processing is definitely much faster than one machine searching bit by bit on its own.
3. Disaster recovery / backup: say I have two shards and two machines, node A and node B. While node A holds shard 0, it also holds shard 1 as a replica for node B; likewise, node B holds shard 1 as primary and also holds a replica of node A's shard 0.
- `number_of_replicas: 0`: no replicas, because there's only one node and nowhere to put them (that's the `rep` column you just saw in the table).
- `index.knn: true`: **allows this index to do vector search**. This switch can't be changed after creation, so turn it on now. The vector field gets added in step 4; adding fields later is allowed.

**`mappings.properties`**: the type of each field.
- `reference`: e.g. `"John 11:35"`, `keyword` type, treated as a single value.
- `book`: e.g. `"John"`, `keyword` type, used for filtering.
- `chapter`, `verse`: `integer`, numbers.
- `text`: the verse text, `text` type, split into words.
  - `"analyzer": "english"`: split words using English rules. The **analyzer** decides how a piece of text is split into words for search. The english analyzer reduces `loved` and `loves` to `love`, so searching love also finds loved.

An index here is roughly a table in a database, and the mapping can be understood as the table structure (i.e. schema) you write when creating the table.

I generally mean relational databases here, since I know MySQL well and use it for comparison. The correspondence is roughly:
• A MySQL table is what we call an index here
• The schema is what we call the mapping
• A row is a document here
• A column (there are many) is a field here

Since OpenSearch here is a localhost web page running on my machine: in a relational database, putting data in is an insert, while in OpenSearch, as a web service, you just do a normal PUT.



{

  "acknowledged": true,
  
  "shards_acknowledged": true,
  
  "index": "gospels"
  
}

That's what it looks like: write one row to it, and things like reference, book, chapter, text, title are its fields.

<img width="825" height="477" alt="image" src="https://github.com/user-attachments/assets/3efa50ee-b56f-4811-ac37-c433ca7378f1" />

In real use you'd generally use the bulk API, like this: the index on the first line says which document to write to, and the second line is the actual content to write.

<img width="1236" height="567" alt="image" src="https://github.com/user-attachments/assets/45ba982d-9166-4f01-8399-b49f32497c63" />

## OpenSearch match = BM25

<img width="1056" height="755" alt="image" src="https://github.com/user-attachments/assets/05ebd34f-35be-48f9-9435-e3382efbfef0" />

Taking the figure above as an example:

In OpenSearch, match is BM25 by default. It matched three results for me, sorted by score from high to low. The score here is the BM25 score, and source is the original content.

BM25 is TF-IDF plus some extra configuration and such; I've written about it countless times in other notes. I forget it every time, but knowing roughly what it means is enough. Focus on the big picture; I'm not doing research.

Its downside is that it doesn't understand semantics. As we just said, our analyzer is English, so it can recognize worried and worry as the same thing, but it doesn't know that tired and worry are related, because it isn't based on meaning.

## embedding

I installed a very small model locally that outputs only 384-dimensional vectors.

What it does is nothing special:

1. Since it's a new field, first add a column to the index
2. After that, add a processor to the ingest pipeline, specifying which engine we use and how exactly the vectors are computed

Then every time a new document is ingested (ignore old ones for now; the processor doesn't handle old documents), it automatically runs and produces the vector; the old ones still have to be done manually. You have it export their own text embedding field (that 384-dimensional data), and that's it.


---
# AWS OPENSEARCH Service

<img width="1008" height="467" alt="image" src="https://github.com/user-attachments/assets/dfc1c29a-bb87-4da8-82d9-bfd9c0a97a49" />

# AWS OpenSearch: Some Terms Explained

---

## 1. Node types in a cluster, and high availability

| Role | Official name | What it does |
|---|---|---|
| Stores data, runs queries | **data node** | Your current t3.small is one |
| Manages the cluster | **dedicated cluster manager node** (formerly master node) | Only manages metadata: which indexes exist, where shards live. Stores no data |
| Only distributes requests | **coordinator node** | In large clusters, dedicated to the scatter-gather merging work |

**Cluster managers usually come in threes.** Electing a leader needs a majority (**quorum**): with 3, if 1 goes down, 2 remain — still a majority. With only 2, a network split makes each side think it's the leader, causing **split brain**. That's why it's always an odd number.


A note: small clusters (like mine) don't necessarily have a dedicated cluster manager. I'm only using one t3.small, so it does two jobs at once: it's both the data node and the cluster manager.

"Dedicated" in the second row means using a few machines just for this job, which is usually only done for large clusters.

**Multi-AZ with Standby**: data is spread across 3 data centers, and the nodes in 1 of them are normally standby only, taking no traffic; on failure, it fails over within seconds. This is AWS's 99.99% SLA option. The "without standby" you just picked means opting out, which is also why t3 isn't available.

## 2. Blue/green deployment (an easy trap)

For many config changes on AWS, like **changing instance type or upgrading the version**, AWS doesn't change things in place. Instead it:

> Creates a whole new cluster (green) → copies all the data over → switches traffic → deletes the old one (blue)

Consequences:
- With lots of data it can take **several hours**
- **Both clusters exist at the same time**, putting heavy load on CPU and network
- Doing it while the cluster is already busy can get it stuck

A common interview question: "How do you safely upgrade OpenSearch in production?" The answer revolves around this: pick an off-peak window, first make sure the JVM (note: JVM is the Java Virtual Machine. OpenSearch is written in Java, and Java programs don't run directly on the operating system but inside the JVM) and storage have headroom, and take a snapshot first.

Since the JVM came up, here are a few related terms:

1. Heap:
The memory the JVM manages, where a Java program's objects live. For example, I set -Xms2G -Xmx2G here, meaning the heap's minimum is 2G and its maximum is also 2G. This matters because OpenSearch's bottleneck is memory, not storage, so you have to watch it.
2. JVM Memory Pressure:
The percentage of the heap in use. If only 1G is used, that's 50% — simple.
3. Garbage Collection (GC):
The JVM periodically cleans up unused objects; the fuller the heap, the more often GC runs. It clears out unused blocks — a very standard operating system concept. But each cleanup pauses the program briefly, and plenty of people complain about this Java trait: every time it does something, the program stutters.

In short, if JVM Memory Pressure is high — the percentage is nearly full — the whole cluster slows down. It keeps stuttering, so of course it's slow. Perfectly normal.

## 3. Permissions have three layers, and 403 errors usually come from the third

A request passes through three gates in order:

1. **Network**: public access, or VPC + security groups
2. **Access policy**: a resource-based IAM policy deciding which AWS identities can touch this domain
3. **Fine-grained access control (FGAC)**: OpenSearch's own user and role system, granular down to a specific index, field, or set of documents (**document-level / field-level security**)

**The most common trap**: IAM already allows it, but you still get a **403**. That's because the FGAC layer doesn't recognize your IAM identity; you have to **map (role mapping)** the IAM ARN to an OpenSearch role inside OpenSearch. — A note here: turns out I didn't need to.

The root account I use logs in via SigV4 (AWS CLI). I'm sure I didn't do anything, yet it was already in the All Access permission here (i.e. map_user).

Just see the image below.

<img width="1090" height="598" alt="image" src="https://github.com/user-attachments/assets/c5d0b864-3881-487c-84f6-96be0e0ff514" />

One more note: since the root of my IAM identity is the account that created the domain, strictly speaking, what I just said may have been too absolute.

By the way, there are actually two kinds of identities here:

1. user: a specific IAM identity
2. backend role: commonly used to map an IAM role; every program assuming that role gets the corresponding permissions

## 4. Metrics to monitor, and the thresholds that can "lock up" the cluster

The most commonly watched in CloudWatch:

| Metric | What to look at |
|---|---|
| `ClusterStatus.red / yellow` | Health color |
| `JVMMemoryPressure` | JVM memory pressure. Keep it below 75%. **At 92% for 30 minutes, AWS rejects all writes** |
| `FreeStorageSpace` | Remaining disk |
| `ThreadpoolWriteRejected` / `SearchRejected` | Requests that couldn't get queued and were rejected |

Disk has three **watermarks**, with defaults:
- **85% (low)**: no new shards are allocated to this node
- **90% (high)**: starts moving shards away
- **95% (flood stage)**: **the index becomes read-only**, and all writes fail

A node is an independent compute resource, i.e. an instance, and one node can hold many shards. What's fixed is how many primary shards each index has, not which shards sit on each node.

Which node a shard lives on is decided by the cluster manager, so shards can move. This movement is called shard allocation and rebalancing, and the goal is to spread the load evenly. Common triggers:

1. Adding nodes: say there were 5 nodes and the budget got approved for one more; some shards on the original 5 get moved over so the load is more even
2. Node failure: if a node goes down, its replicas get promoted to primaries, and new replicas are created on other nodes (note: whether a shard is a primary or replica has nothing to do with the node; a node never owns any data.

Primary and replica refer to a specific piece of an index's data:
• Primary shard: the original of a piece of data; writes go to it first.
• Replica shard: a copy of the same data as the primary shard.

The primary and its replica must sit on different nodes, otherwise there's no fault tolerance; but again, there's no ownership relationship between data and nodes.

Example: say there are three nodes (Node A, Node B, Node C), holding primaries P0, P1, and P2 respectively.

<img width="681" height="237" alt="image" src="https://github.com/user-attachments/assets/306b981c-d150-41aa-bf73-4ca5e40f5d90" />


Now Node A suddenly disappears (i.e. Primary 0 is gone):
Replica 0 on Node B must immediately be promoted to Primary 0, so this piece of data stays readable and writable with nothing lost.

But this directly causes two things: after Replica 0 becomes Primary 0, data 0 has no replica. + Replica 2, which was on Node A, went down with it, so Primary 2 on Node C has no replica either.

So Replica 0 and Replica 2 must now be recreated. Only Node B and Node C are available, and while restoring the replicas, the rule that a primary and its replica can't be on the same machine still applies — otherwise there's no fault tolerance. That's the idea.)

3. Creating a new index: e.g. in logging scenarios, a new index is created every day, so every day there are new shards that need a home (note: an index is real data, not a schema. The schema in OpenSearch is called the mapping, while an index is the mapping plus settings plus all the documents in it. It's a more complete thing that actually holds data)
4. Disk watermark limits: when a node's disk is nearly full, the watermark mechanism kicks in and no more shards are allocated to it

The whole allocation is a dynamic process, not something fixed at the start where each spot is filled to 70% and left alone. Data keeps growing (e.g. logs grow every day); it isn't a fixed volume. Also, merging segments in the background temporarily takes up an extra copy's worth of disk, so you normally need to leave enough headroom.

## 5. Why vector search is expensive on AWS: memory

An HNSW index is only fast if it's **entirely in memory**. The estimation formula from the AWS docs:

> Memory ≈ 1.1 × (4 × dimensions + 8 × M) × number of vectors (bytes)

M is how many neighbors each HNSW point connects to; default 16. Plugging in your data:
- 4,255 × 1,536 dims ≈ **29 MB**, no big deal
- **10 million** of the same vectors ≈ **69 GB**, needing several high-memory machines

So the common cost-saving tricks for large-scale vector search:
- **Quantization**: compress each number from 4 bytes to 2 bytes (fp16), 1 byte (byte), or even 1 bit (binary), cutting memory several to dozens of times with a little loss of precision

I'd always been a bit confused about quantization, so I'll cover it here:

The number of dimensions doesn't change. A 1,536-dim vector is still 1,536 numbers marking its point in 1,536-dimensional space. Even after quantization it's still 1,536 numbers; each number is just stored with fewer bits and becomes less precise.

The dimensions stay the same. Taking an X/Y axis as an example: originally X is 1.734 and Y is 3.528; after quantization, X might be 1 and Y 3.

The dimensions don't change at all, but the precision of the position they mark does drop.

That raises another question: why does it still work after this compression?

Because retrieval looks at who's closest — it cares about ranking, not the exact distance values. Small rounding like this rarely changes the ranking; what was close is still close. The most common approach is to quickly find the top 100 with the compressed version, then re-score those 100 at original precision; this step is called rescoring. After all, once you've taken the top 100, the rest are unlikely to have much potential as candidates.

The same goes for running a large language model locally. With quantization, the model gets smaller, so the GPU stops screaming Out of Memory. But in that case, inference runs fine while training doesn't. Training involves backpropagation, which nudges the weights bit by bit. When precision is too low, such tiny changes are as good as nothing — they get rounded straight to 0. For example, 3 plus 0.0001: if each number can only represent so many digits, the 0.0001 can't be stored at all, and adding it does nothing.

One more thing: why can methods like QLoRA train on an already-quantized model? Because they freeze the quantized base model and add a small block of high-precision parameters to train. During backpropagation, the changes land on those high-precision parameters, so the weights can actually update.

- **Disk-based vector search**: the main index lives on disk, with only a compressed version in memory

This is a weighty point when discussing vector databases in interviews: **the cost bottleneck of vector search is memory, not disk.**

## 6. Storage tiers: the core of logging scenarios

| Tier | Where it lives | Characteristics |
|---|---|---|
| **Hot** | Data node EBS | Fast, expensive |
| **UltraWarm** | S3, with a cache in front | Read-only, much cheaper |
| **Cold** | S3 | Must be attached before querying, cheapest |

Pair this with an **ISM (Index State Management)** policy that automatically does "move to warm after 7 days, to cold after 30 days, delete after 90 days." This is standard in logging scenarios.

The **OpenSearch Optimized (OR / OM / OI series)** instances you just saw in the console take a different approach: data is durably written to S3, and nodes sync via **segment replication**, giving higher write throughput — suited to large-scale writes.

## 7. Domain or Serverless

| | Domain (what you just created) | Serverless |
|---|---|---|
| What you manage | Choose instances, shards, version yourself | Nothing; auto-scales by OCU (OpenSearch Compute Unit) |
| Minimum cost | t3.small about $27/month | Starts at over a hundred dollars a month |
| Limitations | Few | Some APIs and plugins unsupported |
| Good for | Steady load, fine-tuning | Unpredictable load, no ops; **the default vector store for Bedrock Knowledge Bases** |

What Serverless creates isn't called an index but a **collection**, in three types: search, time series, vector search.

OCU (OpenSearch Compute Unit): Serverless's billing unit. One OCU is roughly 6 GB of memory plus corresponding CPU and temporary storage. The more you use, the more OCUs AWS adds automatically, billed as "OCU × hours."

## 8. Surrounding integrations (just know the names)

- **Automated snapshot**: hourly, stored in AWS-managed S3, kept for 14 days, **free**. For migration or disaster recovery, use manual snapshots stored in your own S3
- **OpenSearch Ingestion (OSI)**: a managed data ingestion pipeline (built on the open-source Data Prepper) that can pour in data from S3, Kafka, DynamoDB
- **ML connector**: covered earlier — use ML Commons to call embedding and rerank models on Bedrock

While we're here, one more thing.

In local OpenSearch (e.g. all-MiniLM-L6-v2, one of the pre-trained models OpenSearch officially provides), there are officially several other embedding models, sparse embedding models, and cross-encoder models (a cross-encoder is used for reranking, while sparse embedding is a retrieval approach between keywords and vectors). These official models can all run directly inside the cluster, without calling any external API.

The AWS managed version doesn't allow this. A managed domain generally won't let users install models to run in the cluster; you have to use their connector to call external models, like Bedrock, SageMaker, or third parties such as OpenAI.

If you don't even want to wire up a connector yourself, Serverless has an Automatic Semantic Enrichment feature: once enabled, it does semantic search on its own without connecting a model. That's what the Semantic Search OCU charge is for.

---
# AWS OpenSearch Results

<img width="877" height="470" alt="image" src="https://github.com/user-attachments/assets/f38c20b2-3bc8-448c-8b23-a0aed7463c7e" />


The evaluation results on AWS and locally are in. The accuracy is identical (as it should be: OpenSearch on AWS didn't do any embedding this time; I sent over vectors already computed earlier. In other words, the documents sent over weren't human-readable text but the corresponding 1,000-plus-dimensional vectors. Search uses the same k-NN query too, so of course it's the same — it'd be weird if it weren't). But speed differs: 10 ms locally, 200–300 ms on AWS.

Nothing to be done about that; it's mainly network latency, since requests go from me to the US East data center and back.

But in a real deployment, if the application and OpenSearch are in the same zone, or even the same VPC, that network overhead naturally goes away. Fine.

# AWS ML connector to Bedrock

The model is Amazon Titan Text Embeddings V2 on Bedrock (1024-dim output).

The result: Titan's performance is clearly lower than OpenAI's. This model really is weaker on this task.

The image below is a bit misleading: "OpenAI 3-small" there still means the vectors I had already exported with it on my computer were sent over as documents; I didn't re-run an OpenAI 3-small embedding model on the OpenSearch service. Also, when testing with method A, since I already had an OpenAI API key locally, I first sent the query to OpenAI's embedding model locally to get the vector, then sent that vector to AWS for matching.

Why go through all this? Because although many OpenAI models are on Bedrock, the embedding models aren't. The ones on Bedrock are basically all text-generation models whose output type is text.

The only embedding models available on Bedrock are Amazon's own (e.g. the Titan series, Multimodal Embeddings), Cohere's, and Twelve Labs (that one's for video). So there's no convenient direct connection, hence this pointlessly roundabout detour.

Doesn't matter — it's only a test anyway. Cohere's Embed v4 actually has one of the best reputations for text embedding on Bedrock. I figured I didn't need to test it, close enough — oh well, might as well run it while I'm at it.

<img width="900" height="457" alt="image" src="https://github.com/user-attachments/assets/8038446a-16f8-4d27-8c39-a26d9a2c579c" />

<img width="605" height="221" alt="image" src="https://github.com/user-attachments/assets/47969caf-c6c8-4ad8-9a5b-a6224ea7a5ed" />

<img width="886" height="456" alt="image" src="https://github.com/user-attachments/assets/4eadd446-6f9e-4398-ae55-70bfdc621577" />

After the whole round, switching engines back and forth (from RAGFlow to local OpenSearch, then to AWS OpenSearch) had a modest effect, not much; pure vector stayed around 84% throughout.

But switching embedding models caused a big drop, which shows how important the embedding model is.

Also, rewriting the question can make up for a weak model. From this chart, after rewriting, the gap between Titan and OpenAI shrank from 19 points to 7.

## embedding model: openAI 3 small VS Titan V2 VS Cohere Embed v4

<img width="881" height="455" alt="image" src="https://github.com/user-attachments/assets/72b0985c-83a8-4255-b2fe-9fe67e0a040f" />

None of this uses rerank. Look at the second row: pure vector plus rewriting reaches 95%, without even using rerank; whereas RAGFlow's 95% needed a Voyage rerank stacked on top.

So this shows Cohere Embed v4 is really good.

But honestly, whether adding the rerank layer helps is debatable. Doing it in the project was pure showing off — truly just showing off. In practice it comes down to actual results, so don't bother.

<img width="882" height="416" alt="image" src="https://github.com/user-attachments/assets/b8d29bab-77f6-47bd-ab17-a5eb92180db4" />

Cohere matched it even without enrichment and rerank.

Rerank: yes. Add a rerank processor to the search pipeline that calls a remote rerank model via a connector, e.g. Cohere Rerank 3.5 or Amazon Rerank on Bedrock.

Chunk enrichment: yes. Add an ml_inference processor to the ingest pipeline that calls an LLM (e.g. Claude on Bedrock) via a connector, having it generate keywords and questions for each chunk at ingestion time, stored as new fields, then search those fields with BM25.

# AWS OpenSearch Pipeline

A pipeline is a sequence of processors executed in order.

OpenSearch has two main kinds:

1. Ingest pipeline: runs before a document is written to the index. In the experiments above, this is mainly what Titan embedding and Cohere embedding did: send the text to the model to compute a vector, then store it in the embedding field.
2. Search pipeline: runs at search time, in three main stages (processing the request/query, combining results, processing the response). For the latter part of handling a user's question, its main job is normalizing the BM25 and vector scores and adding them with weights — the hybrid search logic.


Note that in OpenSearch, the AI search workflow in the dashboard and the pipeline are two different things; the pipeline is what actually does the work. (On the main AWS dashboard, the "pipeline" on the left side of the OpenSearch Service page refers to the data ingestion pipeline — open it and you'll see S3 and the like, i.e. channels for incoming data. It has nothing to do with the ingest and search pipelines we're discussing; it's further upstream.)

<img width="1163" height="738" alt="image" src="https://github.com/user-attachments/assets/03e5f281-93de-4b05-a2c1-31f98668d2ff" />

<img width="676" height="528" alt="image" src="https://github.com/user-attachments/assets/2aee2a70-1a14-4741-b2ea-81c17444e349" />

Anyway, the above lists all the pipelines. Two of them have only one step, which can be confusing, but it doesn't matter.

The logic is similar to CloudFormation: it goes down line by line, like normal code executing sequentially.

For example:

<img width="582" height="577" alt="image" src="https://github.com/user-attachments/assets/a179675d-63f8-4560-a4ee-b147abb12e4d" />


Step 1 set: add a field recording the ingestion time
→ adds "ingested_at": "2026-10-31T10:15:02Z"

Step 2 ml_inference: give the text to an LLM to generate questions, stored in questions
→ adds "questions": "I'm exhausted, where can I find rest? ..."

Step 3 text_embedding: give the text to Cohere to compute a vector, stored in embedding
→ adds "embedding": [0.012, -0.034, ...] (1536 numbers)

---

One more thing. From the earlier experiments, Cohere's Embed v4 is far better than Amazon's own Titan v2. So I wondered about the Bedrock Chatbot on my personal site, which originally used Titan — would switching to Cohere be better? I ran an experiment.

It turned out both are terrible at Chinese. That's a trade-off of effort, not technology: I built it mostly with English-speaking visitors in mind, so I only put real effort into English and didn't do much for Chinese.

But none of that matters. Without proper Chinese handling, both give terrible results with no real difference; and as the image below shows, the English results are exactly the same too. Add that Cohere v4 costs 6× as much as Titan v2 — why bother? Just stick with Titan v2.

<img width="988" height="330" alt="image" src="https://github.com/user-attachments/assets/585dc253-6752-4517-bff7-e03ee260e532" />
