> English version of [GraphRag+RagFlow+AWSBedrockChatbot.md](../GraphRag+RagFlow+AWSBedrockChatbot.md). Original notes are written in Chinese/English mix.

# A knowledge graph is a data structure; GraphRAG is a RAG method that uses that structure

A knowledge graph stores information as "entity—relation—entity" triples, for example:

(Jesus) —meets→ (Zacchaeus)

(Zacchaeus) —located in→ (Jericho)

(Zacchaeus) —role→ (chief tax collector)

Connect many triples and you get a graph.

It was hot for a while after Google launched the Knowledge Graph in 2012 (the info card on the right of search results). Then people gradually stopped talking about it, mainly because building the graph was too expensive: either manual curation, or a complex NLP pipeline for entity recognition and relation extraction, which wasn't accurate anyway.

GraphRAG: Microsoft made it popular in 2024. The core idea: let a large model automatically extract entities and relations from documents and build the graph. Building the graph, once the most expensive part, is now cheap, so knowledge graphs are back thanks to LLMs.

It adds two capabilities over ordinary RAG:

Multi-hop queries. For example, asking "Who did Jesus meet in Jericho?" can walk the graph: Jericho → people located here → relation to Jesus. Ordinary vector retrieval can only find passages that are "textually similar"; it doesn't follow relations.

Global questions. Group tightly connected entities in the graph into a "community" and write a summary for each community in advance. For questions that require reading the whole book, like "What kinds of conflict between Jesus and the Pharisees are there in the Gospels?", you can answer directly from those summaries.

In short, the knowledge graph is the material and GraphRAG is the method; LLMs made the material affordable again.

GraphRAG isn't really a good fit here. I'm already using RAGFlow, which has knowledge graph capability built in. If I wanted to do it, I'd just add some processing on top of the existing vectors. But its default template doesn't work: it's designed for companies, basically all regulations, predicates, and so on, so that would need changing. -To be specific, the knowledge_graph.yaml in the container extracts entity types person, organization, product, regulation, and location by default, with relation types "owns, belongs to, causes, regulates, located in," designed for enterprise documents. For the Gospels, "product," "regulation," and "regulates" are basically useless, while what really matters, "event," "group (Pharisees, disciples)," "meets, heals, teaches," isn't there at all.

I was trying to force a use case for it: could I put it on my personal site, deploying the personal site's RAG to AWS to show off? That led to the monthly cost question:

1. AWS offers Neptune Analytics, but it's too expensive. It charges by how long the graph is running, regardless of whether anyone visits. The official price for a 256GB graph is about $7.68 per hour. But for a personal site showcasing my experience, who needs 256GB? Even reaching 1GB would mean I'm frantically padding it. Hourly billing is just too expensive; this option is out.
   
2. Then, while I was at it, I looked for another use case: could I set up a Knowledge Base on my personal site and do hybrid search? One look showed the only wallet-friendly, practical option is a Knowledge Base using S3 Vectors for search. But S3 Vectors is basically just BM25; hybrid isn't possible at all. If you want hybrid search, you have to use OpenSearch, with a floor of at least $150 a month.

A bit of context here: to process documents in an AWS Knowledge Base, you have to configure a vector store for it, and the real issue is that vector store's cost. (- A side rant: AWS's Hybrid retrieval weights aren't public; it's a complete black box. This has always really bugged me: how can weights like this be a black box? Or does it use some especially complex algorithm underneath that just can't be understood in terms of a weight? Either way it's the out-of-the-box, use-it-and-done kind, all about being simple, direct, and easy.)

The main vector store options you can configure:

1. S3 Vector
Billed by storage and number of queries. For my small amount of data it's basically negligible, a few cents a month.
2. OpenSearch
• OpenSearch Serverless: the classic version starts at $175 a month
• OpenSearch managed cluster: since it's a cluster, at least one machine has to be running all month, and the docs say it doesn't support Hybrid Search
3. Aurora PostgreSQL (pgvector)
I wrote a primer on it in another doc. With Serverless v2, it can scale down to 0 when idle. The problem is a cold start takes a dozen-plus seconds, so chat stalls at first. But for cost and wallet reasons, I think it's fine to just note in the docs that cold start takes a dozen-plus seconds; it might actually work. Also, PostgreSQL can do hybrid retrieval. (- To add: I think this is doable. It's billed by time after startup. The cold start is only a dozen-plus seconds anyway, it's only slow on the first question, and later I can apply some engineering tricks to optimize it. I think this is doable, totally show-off-worthy, and adds something to my resume.)
4. Neptune Analytics (GraphRAG)
Very expensive, billed by the hour.

The rest are third-party options, not natively supported by AWS:

• Pinecone: has a free tier, but doesn't support hybrid retrieval.
• MongoDB Atlas: has a free cluster, but whether it can be used with Knowledge Base still needs to be verified.
• Redis Enterprise Cloud: burns money even when idle, and can't do hybrid retrieval at all.

Looking at the whole thing, this scenario shouldn't use this stuff in the first place; it's mostly a cost problem. And I think I'm pretty smart; I'm not a sucker. Why would I pay for it when it might not get used once a month and the site might barely get visited? It's a bit like those emotional-companion projects I ranted about before: possibly not a single real user a day, barely a few log lines, yet they insist on a Redis cluster. I feel the same way here. Isn't this using a dragon-slaying sword to peel potatoes? My head hurts thinking about it. Putting all this stuff on a scenario like this is so annoying.

# RAGFlow Vs AWS

With RAGFlow, good news: **Elasticsearch is wrapped inside RAGFlow, so you never touched it directly.**

Among the containers RAGFlow starts with Docker Compose, one is Elasticsearch (`es01`), and it's the default document engine. There's also a `DOC_ENGINE` setting in `docker/.env` that can be switched to `opensearch` or RAGFlow's own `infinity`.

This comparison has two layers: **RAGFlow as a whole** maps to a set of managed services on AWS, and **the ES inside RAGFlow** maps to OpenSearch Service.

---

## Layer 1: what each RAGFlow container becomes on AWS

| RAGFlow local container | What it does | AWS equivalent |
|---|---|---|
| **es01 (Elasticsearch)** | Stores the chunks: full-text index + vectors | **OpenSearch Service** |
| **MinIO** | Stores the original files you upload | **S3** (MinIO is S3-compatible to begin with) |
| **MySQL** | Stores metadata like knowledge bases, users, conversations | **RDS / Aurora MySQL** |
| **Redis / Valkey** | Parsing task queue, cache | **ElastiCache** (or replace the queue with SQS) |
| **ragflow-server** (API + parsing task processes) | Business logic, DeepDoc parsing | **ECS + Fargate** |
| nginx + frontend | Web entry point | **ALB** + Fargate, or S3 + CloudFront |
| The LLM / Embedding models you configured | Generate answers, compute vectors | **Bedrock** |
| Container logs | Troubleshooting | **CloudWatch** |


## Layer 2: which OpenSearch and Bedrock features RAGFlow's concepts map to

The AWS service most like RAGFlow as a whole is actually **Bedrock Knowledge Bases**: it also does the whole "upload documents → parse → chunk → embed → store → retrieve → generate" flow, and **its default vector store is OpenSearch Serverless**.

| What you know from RAGFlow | In OpenSearch | In Bedrock Knowledge Bases |
|---|---|---|
| Knowledge base | An **index** | A Knowledge Base |
| Uploading files | Not this layer's job | Data source points to an **S3 bucket** |
| Parsing method (General, Q&A, Paper, Laws, Table...) + DeepDoc | Not this layer's job | Parsing options (default parsing or foundation-model parsing) + chunking strategy (fixed-size, hierarchical, semantic) |
| Embedding model | `knn_vector` field (dimension must match the model) | Pick a Bedrock embedding model |
| **"Keyword similarity weight"** in retrieval testing | **weights in the hybrid query + normalization pipeline** | Choose hybrid search at retrieval time |
| Similarity threshold, Top N | `min_score`, `k`, `size` | Parameters like `numberOfResults` |
| Rerank model | Rerank processor in the search pipeline | Rerank model in the retrieval config |
| Chat assistant | Not this layer's job | `RetrieveAndGenerate` API |
| Agent canvas | Not this layer's job | **Bedrock Agents / Bedrock Flows** |

As you can see, OpenSearch only handles the "store and query" part. The other RAGFlow things you know (parsing, chunking, chat, Agents) on AWS are either handled by Bedrock or written by you in Fargate or Lambda.

---
# Start from RagFlow

<img width="371" height="347" alt="image" src="https://github.com/user-attachments/assets/4a808a39-e81b-44d2-9b79-352db7e1cee3" />

This figure shows the 6 containers RAGFlow runs in my local project. From here, let's map things properly.

First, by default RAGFlow still uses self-hosted Infinity. Fully default, with no parameters changed, it doesn't use OpenSearch.

Setting aside what the other things do, the only thing in there that's really about search is Infinity. Infinity occupies the same niche as OpenSearch (i.e. ElasticSearch). When I tuned it before, didn't I set keyword similarity weight = 0.3? That means keyword similarity counts for 30% and semantic for 70%. That's all inside Infinity. In OpenSearch, it can also do hybrid retrieval, and this ratio has a corresponding setting there too. -A rant: of course it has to exist. Once hybrid is involved, what do you do without a weight? Force a 50/50 split? That's very easy logic to arrive at.

Now about the processed data.

Since this uses Docker, the data lives in a volume managed by Docker itself. The volume is named jesussayshi-ragflow_infinity_data, about 200MB.

It mainly contains the chunks, the computed vectors, and the keyword index.

Since we're on this topic, let me sort out RAGFlow first. Once RAGFlow is clear, getting into AWS isn't hard.

The Bible's native structure is books, with chapters in books and verses in chapters; a verse is something like John 21:20.

On top of that, the translation I use adds section headings between verses, grouping verses about the same event, e.g. John 21:20-25.

So I actually split by section heading, cutting the whole source document into its 487 sections; this is called "structure-aware chunking" (chunking by the document's own structure).

Anyway, this is what a document looks like opened in RAGFlow. Since the project already does ingestion enrichment, each chunk was first run through an LLM individually to write 5 keywords and 3 questions for it.

<img width="1352" height="785" alt="image" src="https://github.com/user-attachments/assets/9899292d-a93e-462b-b31b-af4941492490" />

<img width="822" height="618" alt="image" src="https://github.com/user-attachments/assets/9392b46a-c20b-45e7-9492-0b2731a6dfe4" />

Sorry, I really didn't look closely before; things moved so fast that I couldn't keep up. But looking now, there's something about these darn keywords I don't get: why do some chunks treat the verse number as a keyword? - Because it's just a section heading, literally one line; forcing it to distill 5 keywords out of that is hard for it too. Take the figure below as an example.

<img width="762" height="592" alt="image" src="https://github.com/user-attachments/assets/283996c9-d5fe-4b9e-85e4-6490bf49ae2d" />

Further down are some ingestion enrichment settings.

A few parameters explained:

1. Candidate: how many to coarsely select first, for Rerank/Relevance sorting; my program sets 30
2. Metadata filter: filtering by tags. In V2 of the program it had to be Japanese, so it was set to yes back then; V3 doesn't use that at all, so it's currently unused
3. Top N: how many to show in the end; my program sets 3
4. Not shown on the web page, but the program sets min score = 0.3. If even the top result is under 0.3, treat it as not found and drop all three picks.

## Some configuration parameters

<img width="1271" height="783" alt="image" src="https://github.com/user-attachments/assets/1faf1ad3-5dc2-44fb-a69f-56c3ee2f5592" />

This window controls **the processing pipeline data goes through before entering the library**. Everything earlier was about "how to find books"; this is about "how books get shelved."

### Parse mode: pick a ready-made pipeline

**General (Built in)** means "general-purpose, built into the system." RAGFlow provides different pipelines for different document types, e.g. Q&A (question-answer pairs), Paper, Book, Laws, Table, etc. You can also assemble your own.

Your scripture is plain text, so it uses General, with a few settings changed inside.

### The four tabs below = the pipeline's four stages, run left to right

| Stage | What it does | Analogy | How your project sets it |
|---|---|---|---|
| **Parser** | Reads the file into plain text | Unpacking and opening the book | Default. Your files are already txt, nothing to process |
| **General Chunker** | Cuts the text into cards | Taking the book apart into cards | **Split on newlines, one line per card; the whole file is the parent, each line a child**. This is the "one verse, one card" from earlier |
| **Extractor** | Has the LLM add tags to each card | Writing keywords and "questions it might answer" on each card | **5 keywords and 3 questions per card**. What you see on the cards was generated by this step |
| **Indexer** | Builds indexes | Filing the cards into two catalogs | **Both: vector (search by meaning) + full-text (search by keyword)** |



**Enable vision enhancement on the Parser page**: when it meets images or scanned PDFs, it calls a vision-capable model to recognize the content. My files are plain text, so it's not needed.

### Mapping to AWS (noting it while I'm at it)

When creating a knowledge base in Bedrock Knowledge Bases, what you choose is almost exactly these four steps:
- **Parsing strategy**: maps to Parser
- **Chunking strategy**: maps to Chunker
- **Custom processing (Lambda)**: maps to Extractor. AWS doesn't have built-in "generate questions per chunk"; you write it yourself
- **Embedding model + vector store (e.g. OpenSearch)**: maps to Indexer


There's also a memory here. I checked what it means: it's memory about the user.

I really want to rant about "memory" in the Jesus project; it's so ambiguous. In my own Jesus project, memory means the chunks of Jesus's life taken from the Gospels. But generally, when people say memory system, they mean memories about the user.

On AWS, "remembering the user" is handled by Bedrock AgentCore Memory, split into short-term memory (this conversation) and long-term memory (user info across conversations). It's the same kind of thing as RAGFlow's Memory.

# Bedrock chatbot on my personal site

## hybrid research 
Focus on the big picture:

HNSW is a vector index; it and OpenSearch's KNN are the same kind of thing. Don't dig into the specific algorithm; just know it's a vector index over the semantic space.

GIN is a PostgreSQL inverted index that records which chunks each word appears in.

BM25 is a scoring method, not an index itself. Again, big picture: everyone has their own secret sauce for the specific algorithm. Not important; just know it's the keyword side.

I use Hybrid here, both sides, done. The weight parameter is a Bedrock black box: unknown, not tunable. AWS holds it and you can't see it.

## Bedrock overview

AWS Bedrock is a big platform, and Knowledge Base is just one feature. Besides Knowledge Base, Bedrock includes:

• Various large language models (e.g. Claude)
• Agents Workflow (managed Agent workflows)
• Guardrails (content safety filtering)
• Evaluations

In this project, although AWS Knowledge Base directly offers an all-in-one "retrieve and generate" solution, I didn't use it; I still split it apart.

That is, Knowledge Base only does retrieval, and for the retrieved chunks I separately have a Lambda call Claude, using Haiku.

The benefits:

1. More control over the prompt
2. More complete logs, easier troubleshooting
3. Easier to adjust and change later


### Evaluations

For this, there's Bedrock Evaluation, which can specifically evaluate a Knowledge Base. It has built-in metrics and also supports custom ones.

It evaluates by bringing in a model as judge: you prepare a question set (questions plus answers, stored in S3), it runs my RAG over them, and then a large model scores each one.

Built-in metrics cover two kinds of tasks:

Retrieval only (Retrieve only) - Context relevance: whether the retrieved chunks are relevant to the question. Context coverage: how much of the information the reference answer needs was retrieved (you must provide reference answers)

Retrieval + generation	- Correctness, completeness, faithfulness (whether it made things up beyond the material, i.e. hallucination), citation accuracy, helpfulness, logical coherence, plus safety metrics like harmful content and refusals

Bedrock's built-in metrics all rely on an LLM judge. The upside is you don't have to label "which chunk is correct" item by item; the downside is the judge can be wrong too, and you pay for the judge model on every run.

Custom: you can write your own scoring prompt and define new metrics, e.g. for your scenario, "did it make up contact information."


Any evaluation needs a set of reference answers; it's really a well-defined problem in itself. For plain text content, my description may not be very precise, but the point is that for scenarios with clear rules, where fixed text can determine right and wrong, there's already a very mature set of techniques.

There's no room to show off your cleverness here, and the more cleverness you apply, the more likely the result is no better than the standard best practice, and probably worse. So here I just do what they say. Things have developed this far; it's not even a matter of reinventing the wheel. If I did it myself, I most likely couldn't make a wheel at all, just a square polygon. This isn't a master's thesis, and I don't need a novel contribution. Just listen to them.

## Getting my material into the store

Pretty rough, but that's not AI's fault. I had Claude Code do it for me, and I actually understand why it turned out this way.

<img width="677" height="220" alt="image" src="https://github.com/user-attachments/assets/bfd7f96a-be6a-4de8-abfd-0f724161ff0b" />


Its chunking right now is in a very primitive, ancient state:
• At most 400 tokens per chunk
• 15% overlap between adjacent chunks
• Each chunk goes to Amazon Titan Text Embeddings V2, producing a 1024-dimensional vector
• Then written to Aurora's bedrock_integration table (one row per chunk)

What I want to rant about is the chunking quality. It took the READMEs from my public GitHub repos plus the original site copy, generating 42 .md files in total.

This really isn't its fault. My READMEs are written very erratically: not properly sectioned, mostly not technical architecture but dev logs and assorted rants; plus I have an extremely strong urge to express myself, so they pile up very long without proper sections. It really is garbage in, garbage out.

No problem. I'll tune it separately later and tell it to skip the READMEs and look at dedicated technical architecture docs. Actually, forget it, I'll have it do that now - doing

## After a visitor asks a question

<img width="675" height="405" alt="image" src="https://github.com/user-attachments/assets/fd0f11d7-655b-488c-b2b3-83c40dfb7df3" />

Lambda function, an old friend; I've done this before, so I won't cover it here.

But I had one question: why is DynamoDB involved here? - Because there's a cap of 80 per day, to keep my wallet from exploding. It's also to guard against DDoS, in case Cloudflare's DDoS protection wall (the one from the company whose boss is supposedly "pretty and kind-hearted") fails and things blow up on my side, so this extra layer was forcibly added.

It's also cheap; this is just implementing the requirement however works. This check has to happen before waking up PostgreSQL.

<img width="741" height="330" alt="image" src="https://github.com/user-attachments/assets/8596dc52-bbcc-4cf4-9a03-0d4781e12cef" />

<img width="927" height="252" alt="image" src="https://github.com/user-attachments/assets/f8555137-8a93-44a2-a26e-dff9f2dd1870" />
