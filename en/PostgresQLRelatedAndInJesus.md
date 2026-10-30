> English version of [PostgresQLRelatedAndInJesus.md](../PostgresQLRelatedAndInJesus.md). Original notes are written in Chinese/English mix.

# PostgresQL itslef

Let me summarize for now, starting with what Postgres is. I already knew MySQL, right? So for Postgres, the SQL part, including schemas, CREATE TABLE, and so on, is all the same; nothing to worry about there. But it's different from SQLite: Postgres isn't a file, it's a service process that's always running, listening on port 5432 by default (that part is like MySQL, but unlike SQLite). Connection info is usually written as a connection string.
postgresql://username:password@host:port/dbname
postgresql://postgres:mypassword@localhost:5432/mydb

## A few concepts you must know

**1. Parameterized queries (most important)**

```python
# ❌ SQL injection risk
cur.execute(f"SELECT * FROM users WHERE name = '{name}'")
# ✅ let the driver escape it for you
cur.execute("SELECT * FROM users WHERE name = %s", (name,))
```

Placeholder syntax varies by driver: psycopg uses `%s`, Node `pg` and Go use `$1, $2`.

**2. Transactions / commit**

Postgres drivers open a transaction by default. **The data you write is only really saved after commit**. A `with` block handles this automatically; if you manage it manually:

```python
conn = psycopg.connect(DSN)
cur = conn.cursor()
cur.execute("UPDATE ...")
conn.commit()      # forget this line and the changes are lost
conn.close()
```

"The program finished but the data never made it in": nine times out of ten, it's because there was no commit.

**3. Connection pool**

Establishing a connection is fairly expensive. A web service shouldn't `connect()` on every request; instead, open a **connection pool** shared by all requests. Node's `pg.Pool` is a connection pool; in Python you can use `psycopg_pool.ConnectionPool`.

Let me add something here, since this used to confuse me too.

Suppose Postgres has 3 connection pools attached, each with 10 connections, so at most 30 connections can access Postgres at the same moment. It's not FIFO in the sense that at each moment only the first in line in each of the 3 pools gets to connect; as long as the pools have room, all 30 connections can start at once. In this setup, it saves the overhead of things like the SSL handshake between the pool and the database, and saves some process-related resources.

(Important addition: the biggest overhead is actually that Postgres forks a new process for each connection.

With a connection pool, there's still a trade-off:
• Without a pool: clean at first; when something connects, it forks a process; when the connection ends, the process is closed
• With a pool: the process for each connection has to stay open the whole time)

Now, why was it designed as one process per connection in the first place?

Because PostgreSQL was born around 1986, when multithreading wasn't mature yet, so this kind of performance issue wasn't considered at all. Also, process isolation was seen as safer and more stable back then: if one connection's backend process crashed, it wouldn't take down the others. And the code really was simpler that way.

Changing that legacy mess now: PostgreSQL's code already has tons of global variables, all assuming each process serves only one connection. Refactoring now would be really hard, so better not to touch the legacy mess. Just solve the problem with connection pools rather than reworking the program from its very foundations to fit modern times; use it as is. And even if Postgres were made multithreaded, connection pools also solve other problems, like the overhead before a connection is established: TCP handshake, TLS handshake, password authentication, session initialization. Even with multithreading, those costs would still be there.

Also, each connection has to maintain a set of state on the database side anyway, like memory caches, transaction info, and lock info. Process or thread, 1000 connections means 1000 sets of state.

On top of that, concurrency is limited by the number of CPU cores. With too much concurrency and too few cores, they start competing for CPU time, which actually hurts I/O efficiency.

So it's fine as it is; no need to change it.

One more benefit: Postgres has an internal limit of 100. If there are 150 requests without a pool, the 50 that can't connect get errors right away. But with a pool as an extra layer in between, those 50 requests queue outside the pool waiting to get in, instead of erroring immediately. A much better experience. One more note: in these requests, the "user" we're talking about isn't a real person; in this context it usually means backend programs.

Queue order: common implementations like psycopg_pool and Node's pg.Pool are basically first-come, first-served (FIFO). But in practice you rarely need to care about order, because:

A simple query usually holds a connection for only a few milliseconds, so 10 connections can handle over a thousand requests per second
If queuing happens often, the pool is too small, some queries are too slow, or some code borrowed a connection and never returned it

What happens if it waits too long? Pools usually have a timeout setting; e.g. psycopg_pool's timeout defaults to 30 seconds. After the timeout, the request errors out instead of waiting forever.

### A few common connection pool pitfalls

Create the pool only once: make it global or create it at app startup. Don't new Pool() in every request; that's the same as having no pool.

Always return what you borrow: use with / try...finally. The classic symptom of a connection leak is that after running for a while, all requests hang.

A transaction must run on the same connection: if BEGIN and COMMIT borrow different connections, the transaction doesn't hold.

Bigger isn't better: every Postgres connection is a process, so too many connections actually make things slower. 10–20 is usually enough. Note: if you deploy 5 instances with a pool size of 20 each, that's 100 total, exactly hitting the default limit.

When you don't need a pool: one-off scripts and small CLI tools; just connect(), use it, and close it.



**4. ORM (optional)**

If you don't want to hand-write SQL, you can use an ORM, which maps tables to classes:
- Python: **SQLAlchemy**, Django ORM
- Node: **Prisma**, Drizzle

Under the hood, the ORM still uses these drivers to send SQL. Since you know SQL, I suggest getting familiar with a raw driver first, then deciding whether to adopt an ORM.

## Some Postgres features compared with MySQL / SQLite

- `RETURNING`: return rows directly after an insert or update, no second query needed
- `JSONB` type: store JSON, and query and index by field
- `SERIAL` / `GENERATED AS IDENTITY`: how to write auto-increment primary keys
- Array types, `ON CONFLICT DO UPDATE` (upsert)
- The default schema is called `public`; a table's full name is `public.users`

## Debugging tools

- **psql** (command line): `psql postgresql://postgres:mypassword@localhost:5432/postgres`; once inside, `\dt` lists tables, `\d users` shows table structure
- **GUI**: pgAdmin, DBeaver, DataGrip

---


# Postgres has pgvector to store vectors
pgvector is an extension of PostgreSQL that stores vectors in a relational database.

This also gets at why PostgreSQL has suddenly become popular again. The things AI applications need, PostgreSQL can basically do, and do pretty well.

The core reason is the pgvector extension. AI applications (especially RAG) can't do without vector search: first cut documents into small chunks and turn them into vectors with an embedding model; when the user asks a question, find the few semantically closest passages, then hand them to the large language model.

Early on, everyone used dedicated vector databases for this, like Pinecone and Milvus. But then PostgreSQL got the pgvector extension, and vectors became just an ordinary column type.

As I just said, Postgres wins by putting vectors and other data in the same database, since it's a relational database to begin with.

If we kept using a dedicated database like before, vectors would live in one place and all the user data in another, and everything gets messy. That's genuinely bad for your mental and physical health: every extra component is extra troubleshooting and maintenance overhead. If something breaks, you have to hunt everywhere for which side the problem is on. Very annoying.

So you might as well just use Postgres; ideally one thing covers everything. That was the simple case: one more component means one more cognitive overhead.

But the most important reason is that with PostgreSQL you can filter by business conditions directly. For example, the real requirement is often: among the documents from the past year that this customer is allowed to see, find the latest info on X. With Postgres, access control is handled right in the query, so there's no way for those darn unauthorized-access problems to happen.

In the same situation with a standalone vector database, you'd have to sync permissions, tenant, time, and other info over there, the data on the two sides might not match, and maintaining it is a huge hassle.

It also guarantees consistency through transactions: when deleting a document, its metadata and all its vectors can be deleted together in the same transaction, so you never get a case where the document is deleted but its vectors remain and can still be found by RAG.

I saw someone talk about this on Twitter before (I forget which product): a user stopped using some AI app and deleted all their data, only to find out the documents they'd uploaded were still "living forever" in the vector space.

Sorry, I rambled. To summarize, there are three main benefits:

1. One less system to maintain
2. Filter by business conditions, all in one SQL statement
3. Transactions guarantee consistency
   
### Other Postgres capabilities that fit AI applications
Those three are the main reasons. But it also has other capabilities that fit AI apps: for example, it supports JSONB, which can store irregular data.

Especially now with Agents, all kinds of outputs, parameters, execution traces, and data structures keep changing. JSONB can store JSON directly and supports querying and indexing by field, without changing the table structure every time.

SELECT * FROM agent_runs WHERE result->>'status' = 'failed';

Full-text search + vector search = hybrid search
Vector search alone doesn't do well on exact matches like proper nouns, product model numbers, and names. Postgres has built-in full-text search (tsvector), which can be combined with vector search to do hybrid retrieval in the same database.

One database, many uses
Chat history, user data, task queues (SELECT ... FOR UPDATE SKIP LOCKED), cache, and vectors can all go in Postgres. The popular saying in the community is "Just use Postgres": use Postgres for everything first, and split things out only when you actually hit a bottleneck.


With pgvector, full-text search doesn't need a separate setup. PostgreSQL has full-text search built in: tsvector plus a GIN index. So vectors and full text can live in the same Postgres database, and one SQL statement can do hybrid retrieval. When I said last time that it "usually needs a separate setup," that only applies to Pinecone: it mainly stores vectors, and for keyword search you have to generate sparse vectors yourself and pass them in.

### Ecosystem tailwinds
LLMs write Postgres SQL especially well: there's a huge amount of Postgres material and code online, so AI coding tools generate high-quality SQL. This also makes AI lean toward Postgres by default when writing code and building apps.

Managed services are very convenient: Supabase (Postgres + auth + API + pgvector, the whole package), Neon (Serverless Postgres; spin up a database in seconds, with git-like branching). Many AI app generators and agent platforms use them as the default database, because an agent can spin up a new database whenever it wants.

Capital is betting on it too: in 2025 Databricks acquired Neon and Snowflake acquired Crunchy Data, both Postgres companies.

Fully open source with a permissive license: no lock-in to one vendor, and none of the license controversies of MongoDB or Redis.

### Shortcomings

PostgreSQL isn't a cure-all either:

When vector scale is very large (e.g. billions) or latency requirements are extreme, a dedicated vector database is usually a better fit

pgvector's indexes (HNSW, IVFFlat) need tuning, and combining "filter conditions + approximate search" can lead to incomplete recall, which takes some experience to handle

Massive analytical data (data warehouse scenarios) isn't its strength

But for the vast majority of AI applications, data volume is nowhere near these limits.

---
The real differences between pgvector and Infinity come down to two details:

1. Postgres's built-in full-text search is not BM25

You've got it right: a full-text index is keyword search, and BM25 is the standard algorithm for scoring keyword matches. It looks at two things:

how many times the word appears in this passage;

how rare the word is across the whole corpus (IDF). A word like "sparrow" carries much more weight than "the."

Postgres's built-in scoring function ts_rank only looks at the word within the current passage, not the whole corpus, so there's no IDF. It can find passages containing the keywords, but the ranking quality isn't as good as BM25. To use real BM25 in Postgres, you need to install an extension, like ParadeDB's pg_search.

<img width="926" height="190" alt="image" src="https://github.com/user-attachments/assets/c4bc08e5-e9c9-4909-b03a-68a821c48b6f" />

So the accurate statement is: pgvector can also do hybrid retrieval in one database, but you have to add BM25 scoring and result merging yourself; Infinity has both built in.

Also, by the way: in the stage 1 retrieval, BM25 was hand-written in retrieval.py, about 30 lines of numpy, and the merge was a hand-written RRF too. So you've actually experienced both approaches: hand-writing it first, then switching to Infinity's built-in features. That's a great interview talking point.

---
Pinecone is a cloud-hosted vector database.

---
In my (JesusSaysHi) project, I am using Ragflow's built-in Infinity engine. It is self-hosted and in the same category as Pinecone or Milvus, but it keeps dense and full-text indexes in one store, so hybrid search is a simple query. I did not need a managed service at this scale.

The vectors are stored in the Infinity database bundled with RAGFlow. Infinity is InfiniFlow's own engine; the vector index and full-text index live in the same store, so hybrid retrieval happens in one place.

Specific locations:

Container: jesussayshi-ragflow-infinity-1, selected by DOC_ENGINE=infinity in JesusSaysHi/ragflow/docker/.env.
Data: in the Docker volume jesussayshi-ragflow_infinity_data, mounted at /var/infinity in the container. This volume is inside Docker Desktop's WSL VM, not in your project folder, so you can't find it directly in Windows Explorer.
Config: ragflow/docker/infinity_conf.toml, bind-mounted into the container.

What's stored: 487 BSB sections, split into child chunks by verse. Each child chunk has a vector generated by text-embedding-3-small, a full-text index, and LLM-generated keywords and questions. Other things are stored separately: document and dataset metadata in the MySQL container, uploaded original files in the MinIO container.

<img width="941" height="276" alt="image" src="https://github.com/user-attachments/assets/f9d4a248-058e-49e0-a18e-6ca74ecbab46" />

What MinIO is: an "object store," dedicated to storing files, with the same interface as Amazon S3. Think of it as "S3 on your own machine." RAGFlow is designed to handle lots of user-uploaded PDFs and Word files; the originals need to live in a reliable file repository and are fetched from there when re-parsing later.

Why MySQL is needed: managing information needs something "relational": which file belongs to which dataset, and status changes from "queued" to "done" must not go wrong. That's what traditional databases do best, while Infinity is designed for search and isn't suited to this.

(There are also two supporting containers: kvrocks for caching, and nats as the queue for parsing tasks. So RAGFlow starts 6 containers in total.)

So can't everything just go in Infinity?

For you, absolutely, and it would be simpler. Stage 1 proved it: two files (a vector array plus passage JSON), no database at all, and retrieval still worked.

RAGFlow needs so many components because it's positioned as a multi-user enterprise knowledge base platform: many people upload documents in all kinds of formats, manage them in a web UI, and parsing is queued in the background. The three stores are there for that scenario.

What your project actually uses at retrieval time is almost only Infinity. MinIO and MySQL only matter during import and management.

“RAGFlow brings a whole platform: MinIO for raw files, MySQL for bookkeeping, Infinity for the search index. At my scale that’s overkill. My phase-1 baseline was two files in numpy. I took on that weight for the production pipeline: structure-aware chunking, chunk enrichment, hybrid search and rerank behind one API. If this were only retrieval over 3,800 verses, I’d go back to pgvector or plain files.”

# How RAGFlow is designed to be used

<img width="938" height="472" alt="image" src="https://github.com/user-attachments/assets/f75de7ca-38aa-41fd-94f1-d1426ad54704" />

Think of RAGFlow as a **library**, and the 6 containers as 6 roles in it.

## What each of the 6 roles does

| Container | Analogy | What it actually does |
|---|---|---|
| **ragflow** (main program) | Front desk plus librarian | The web UI, the external API (which your code calls), and the parsers that do the real work: reading files, chunking, calling OpenAI to generate vectors and keywords. The other 5 are its helpers. |
| **MinIO** | Archive room | Stores the original files you upload |
| **MySQL** | Ledger | Bookkeeping: which datasets and files exist, how far each file has been processed, which model is used, who has access |
| **Infinity** | Search catalog | The chunks, vectors, and keyword index. Where the actual "search" happens. |
| **NATS** | To-do task board | Queue system, explained below |
| **kvrocks** | The librarian's notepad | Cache, explained below |

## The two "supporting" containers, in plain words

**NATS: task queue**

When you imported, you dropped in 487 files at once. Each file had to be chunked, embedded, and have the LLM write keywords and questions: about 4,300 model calls in all, taking roughly half an hour.

You can't have the front desk do that much work while also serving visitors. So the front desk just posts a ticket like "parse Luke 15:11–32" on the task board, and background workers take tickets off the board one by one. Benefits:

- The front desk doesn't get stuck;
- A failed task can be retried;
- With more machines, you can assign more workers at once.

**kvrocks: notepad (cache)**

It records things that are "frequently used, need to be fast, and fine to lose": what percent a task is at, who's processing which file (so two workers don't process the same one), recently looked-up config, and so on.

Why not record it all in the MySQL ledger? Because the ledger is written to disk and must be rigorous, so reads and writes are slower; the notepad lives in memory and is much faster. kvrocks is used the same way as the famous Redis; think of it as a Redis replacement.

## When they're actually working in your project

**During import (`ingest_ragflow.py`): all 6 are busy**
1. Files uploaded → stored in MinIO and registered in MySQL;
2. Parsing tasks posted to the NATS task board;
3. Background workers take tasks and do the work, with progress recorded in kvrocks;
4. Chunks and vectors written to Infinity; status in MySQL changed to "done."

So NATS and kvrocks aren't useless; **they pulled their weight during that half-hour import**.

**During conversation (Jesus retrieving scripture): basically just two**
Your code calls ragflow's retrieval API → ragflow checks the dataset and permissions in MySQL → searches Infinity → returns results. MinIO and NATS basically aren't involved.

## One-line summary

RAGFlow is designed for the scale of "a team, thousands of documents, uploads every day," which is why the division of labor is so fine-grained. Your project has only 487 sections and doesn't change after the one-time import, so day to day what's really in use is **ragflow plus Infinity**; the rest mainly exist for that one import.

I take the "workflow" you mentioned to mean RAGFlow. Answering the three questions below in order.

# Other questions

## 1. Does anyone in industry actually use RAGFlow?

Yes, but it's **not the industry default**. First, RAG tools fall roughly into three categories:

| Category | Examples | Characteristics |
|---|---|---|
| **Code libraries** (assemble it yourself) | LangChain / LangGraph, LlamaIndex, Haystack | Provide parts; engineers write code to assemble them. **This is what big companies building in-house use most**, or they just use a few low-level components and write the rest themselves. |
| **Ready-made platforms** (out of the box) | **RAGFlow**, Dify, etc. | Come with a web UI, document parsing, and knowledge base management; deploy and go. |
| **Cloud-managed** | AWS Bedrock Knowledge Bases, Azure AI Search, Google Vertex AI RAG Engine, OpenAI file search | No ops on your side; data lives in the cloud. Very common in enterprises, especially companies already deep into one cloud. |

RAGFlow is open-sourced by InfiniFlow and is very popular on GitHub. Its strength is **document parsing** (PDFs, tables, layout recognition). It's used more where **private deployment is required** and data can't go to the cloud, especially by companies in China. But at big North American companies, it's more common to "build it yourself with LangChain or LlamaIndex plus a vector store," or just use a cloud provider's managed service.

---
The "draggable visual component" you remember is most likely not LangChain itself but **Langflow** (or the similar Flowise). They're GUIs built on top of LangChain: LangChain's parts become blocks, and you drag and connect them to build a flow. The names are similar and easy to mix up. **LangChain itself is a code library**; you use it by writing Python or JS code.

## LangChain: general-purpose "LLM app building blocks"

The core idea is to turn the things commonly needed for LLM apps into standard parts, chained together with code:

- **Model interface**: OpenAI, Anthropic, local models; switch with one line of code;
- **Prompt templates**, **output parsing**;
- **Retrieval**: document loaders, splitters, embeddings, vector stores (ready-made adapters for dozens, like Pinecone and pgvector), retrievers;
- **Tool calling and agents**.

It later spun off two sibling products:
- **LangGraph**: writes complex agent flows as a "state graph," e.g. look up material, decide whether it's enough, look up more if not, with loops;
- **LangSmith**: a web platform for tracing every call and running evaluations.

**Positioning: you can build any LLM app with it; RAG is just one kind.** Pros: lots of adapters, big ecosystem. Cons: many abstraction layers, hard to debug when things go wrong, and the old APIs have changed several times.

## LlamaIndex: specialized in "data ingestion plus retrieval"

Its core focus is **how to feed your data to an LLM**, so it goes deeper on the RAG chain:

- **Data ingestion**: LlamaHub has hundreds of connectors (Notion, Slack, Google Drive, databases...), plus **LlamaParse**, dedicated to parsing complex PDFs;
- **Chunking and indexing**: splits documents into nodes, supports multiple index structures like parent-child chunks and hierarchical summaries;
- **Query engine**: wraps the whole "retrieve → rerank → synthesize answer" flow, so you can build a RAG in a few lines of code;
- It later added agent and workflow capabilities too.

**Positioning: a RAG-specific framework centered on data and retrieval.**

## Compared within your project

| Stage | What you use now | With LangChain | With LlamaIndex |
|---|---|---|---|
| Reading BSB, chunking by pericope | Your own `pericopes.py`, plus RAGFlow parsing | Text splitter | Node parser (parent-child chunks built in) |
| Vector and full-text index | Infinity (bundled with RAGFlow) | Hook up a vector store, e.g. pgvector | Also hook up a vector store, or use its built-in one |
| Hybrid retrieval and reranking | Built into RAGFlow, plus Voyage | Ensemble retriever, plus a Voyage rerank component | Retriever plus postprocessor (rerank) |
| Voice agent and tool calling | OpenAI Agents SDK | Can do it too, but you already have it | Not its strength |

The essential differences among the three:

- **RAGFlow is a "finished platform"**: you start services, it has a web UI, and the parts are all built in.
- **LangChain and LlamaIndex are "code libraries"**: you `import` them directly into your Python program, no pile of containers to start. At your scale, writing it with LlamaIndex would likely be one Python file plus a vector store, or no vector store at all, just local files.

**One-line interview version**:

> "LangChain is the general toolkit for LLM apps, with LangGraph for agent flows. LlamaIndex is data-first: ingestion, indexing and query engines for RAG. RAGFlow is a deployable RAG platform with its own UI and storage. I used RAGFlow, but the pipeline stages map one-to-one onto either library."

---



**What this means for your interviews**: tool names aren't the point. Interviewers care whether you understand each stage, and these concepts carry over to any framework:
- Structured chunking and parent-child chunks
- Hybrid retrieval
- Reranking
- Refusal threshold
- Stage-by-stage evaluation

You also have a great way to frame it: "I hand-wrote a baseline first, then switched to a framework, and compared the two using evaluation data."

## 2. Why MySQL and not SQLite?

Both are relational databases. The difference is **who accesses them**:

- **SQLite is just a file**, embedded in one program. Basically only one writer at a time, and the whole database is locked while writing. Other containers also can't connect to it over the network.
- **MySQL is a standalone running service**. Many programs can connect over the network and read/write at the same time; it coordinates them, and supports backups and scaling.

RAGFlow's web UI, API, and multiple background parsers may all change state **at the same time**, e.g. "file A finished parsing," "file B started parsing," and they live in different containers. SQLite can't handle multiple concurrent writers across containers, so RAGFlow chose a service database. From its config, it looks like it can also be switched to PostgreSQL, but SQLite isn't an option.

**For your solo project**, SQLite would have been enough; MySQL really is overkill. But that's a design decision of the framework, not something you can choose. That's part of the price of using a framework.

## 3. LLM Wiki vs. proper RAG: when to use which?

The fundamental difference: **is the knowledge organized in advance, or looked up at question time?**

- **LLM Wiki**: have the LLM read all the material first and **write it up in advance** as a set of organized, interlinked notes, like a handbook. When a question comes in, the agent flips to the relevant pages of this handbook to answer.
- **RAG**: the material is **kept as-is**, just chunked and indexed. For each question, the original passages are **retrieved on the spot** and handed to the model.

| Situation | Better fit |
|---|---|
| Not much material (a few hundred documents or fewer), changes slowly | **Wiki** |
| Answers need synthesis across multiple sources ("What's our deployment process?") | **Wiki**: organized in advance, answers are more coherent |
| You want knowledge that humans can also read and check directly | **Wiki** |
| Very large volume (tens of thousands of documents or more), or updated daily | **RAG**: recompiling the handbook on every update is too expensive |
| **Must quote the original verbatim and give exact sources** (legal, compliance, medical, **scripture**) | **RAG**: the handbook has been rewritten by the LLM, so the original wording is gone |
| Different people can see different documents (access control) | **RAG**: can filter by document |
| Questions are about scattered details (clause X of some contract) | **RAG**: the handbook likely dropped those details when it was organized |

A Wiki has two inherent risks:
1. **Errors made during organization stick around**: if the LLM writes one wrong sentence while organizing, every later answer inherits it.
2. **Material goes stale**: when the source material changes, the handbook doesn't update automatically.

So in practice it's common to **combine the two**: the handbook covers the big picture, RAG looks up the original text and details.

**Back to your project**: scripture must be quoted verbatim with sources, so RAG is unavoidable. That also explains why you felt the Wiki version was "too rigid": it turned scripture into fixed phrasings in advance and lost the freshness of looking things up on the spot for the question at hand.
