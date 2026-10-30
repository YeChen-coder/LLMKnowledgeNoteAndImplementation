> English version of [LangChainRelated.md](../LangChainRelated.md). Original notes are written in Chinese/English mix.

I take the "workflow" you mentioned to mean RAGFlow. Answering the three questions below in order.

## 1. Does anyone in industry actually use RAGFlow?

Yes, but it's **not the industry default**. First, RAG tools fall roughly into three categories:

| Category | Examples | Characteristics |
|---|---|---|
| **Code libraries** (assemble it yourself) | LangChain / LangGraph, LlamaIndex, Haystack | Provide parts; engineers write code to assemble them. **This is what big companies building in-house use most**, or they just use a few low-level components and write the rest themselves. |
| **Ready-made platforms** (out of the box) | **RAGFlow**, Dify, etc. | Come with a web UI, document parsing, and knowledge base management; deploy and go. |
| **Cloud-managed** | AWS Bedrock Knowledge Bases, Azure AI Search, Google Vertex AI RAG Engine, OpenAI file search | No ops on your side; data lives in the cloud. Very common in enterprises, especially companies already deep into one cloud. |

RAGFlow is open-sourced by InfiniFlow and is very popular on GitHub. Its strength is **document parsing** (PDFs, tables, layout recognition). It's used more where **private deployment is required** and data can't go to the cloud, especially by companies in China. But at big North American companies, it's more common to "build it yourself with LangChain or LlamaIndex plus a vector store," or just use a cloud provider's managed service.

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

# LlamaIndex

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

Let's go one by one.

## 1. Splitters / node parsers: not a black box, just ordinary rule-based code

Both libraries are open source; you can click into the chunking code and read it. The logic isn't mysterious. The most common kinds:

- **Split by character count**: LangChain's most-used `RecursiveCharacterTextSplitter`. You give it a limit, say 500 characters; it first tries to split on blank lines; if pieces are still too long, it splits on newlines; still too long, on spaces, until every chunk is under the limit, with a bit of overlap between adjacent chunks. **It doesn't understand the content at all**; it only looks at characters.
- **Split by heading**: e.g. `MarkdownHeaderTextSplitter` (LangChain) or `MarkdownNodeParser` (LlamaIndex), which cut at headings like `#` and `##`.
- **Split hierarchically (parent-child chunks)**: LlamaIndex's `HierarchicalNodeParser` cuts big chunks first, then small chunks within them, remembering which is whose parent. In your project, "pericope is the parent, verse is the child" is exactly this idea.

**The key point**: none of them understand the BSB format. BSB's section headings live in USFM files as markers like `\s1`, not Markdown `#`. So whichever library you use, you have to write a bit of code yourself first (that's the current `pericopes.py`) to split BSB into 487 pericopes, then hand them to the library for the rest. A library saves you the work after chunking; the first step, "understanding your data format," is always on you.

## 2. LlamaIndex only does the first half? Mostly right, with one addition

| | Does LlamaIndex handle it? |
|---|---|
| Reading data, chunking, generating vectors, building the index | ✅ This is its strength |
| Retrieval, reranking | ✅ |
| Taking retrieval results to call the LLM and generate an answer | ✅ It's called a query engine; you can build Q&A in a few lines of code |
| Your whole app: voice, persona, safety checks, faithfulness checks, avatar | ❌ Still written by you |

So to be precise: **it can handle the whole RAG chain, but only RAG**. The application logic around it you still write yourself; you understood that correctly.

## 3. Where does a "code library" actually run?

This is the most important point. **A library runs inside your own Python process**, just like numpy or requests.

Usage looks like this:

```python
# after pip install llama-index
from llama_index.core import VectorStoreIndex, Document

index = VectorStoreIndex.from_documents(docs)   # chunk, call OpenAI for embeddings, build the index
results = index.as_retriever().retrieve("I feel so alone")
```

When you run `python jesus.py`, the chunking and retrieval code is **function calls inside your process**, using your process's CPU and memory. No other program runs in the background, and no containers need to be started.

So where is the data stored? Three options:

1. **In memory**: by default, vectors live in your process's memory. Call `index.storage_context.persist()` and it saves to a few JSON files on disk, which are read back at next startup. **This is exactly what you did in stage 1**: `retrieval.py` plus `embeddings.npy` and `passages.json`, no services at all.
2. **Embedded database**: e.g. Chroma or LanceDB, which also run as libraries inside your process, just writing data to a local folder. Conceptually the same as SQLite.
3. **Connecting to an external service**: if you configure Pinecone or pgvector, the library acts as a **client** connecting to that database over the network. The database is then another process or a cloud service, but LlamaIndex itself is still just code in your program.

**Comparing the two architectures:**

```
Library (LlamaIndex / LangChain):
  your python process = your code + retrieval code + in-memory vectors
                       └─ HTTP → OpenAI (embeddings, model calls)

Platform (RAGFlow):
  your python process ──HTTP──→ RAGFlow service (another program)
                               ├─ Infinity (chunks and vectors)
                               ├─ MySQL / MinIO / NATS / kvrocks
                               └─ HTTP → OpenAI, Voyage
```

In a sentence: **a library is a "function you call," a platform is "another server you access."** Your stage 1 was the "library" approach, except the library was hand-written by you; stage 4 is the "platform" approach.

---

There's a key misconception here: **when you use a library, you usually don't need to wrap another HTTP service around it at all**.

## Back to your own project first

Your program is already a single Python process (`jesus.py`). When it needs retrieval:

- **With a library**: call a function directly in the process, e.g. `index.retrieve(...)`, and you get the result immediately. **Not a single extra layer.**
- **With RAGFlow**: your process has to reach another service over HTTP, and that service has 6 containers behind it.

So **the extra layer is actually RAGFlow**, not the library. RAGFlow being "already wrapped as an HTTP service" is only an advantage when "retrieval needs to be shared by many different programs." For your single app, it actually adds a network round trip and 6 containers to maintain. You've also personally felt another cost: its API doesn't match the official docs.

## Why do many enterprises choose libraries?

Enterprises usually **already have their own backend services**, e.g. a "customer support Q&A service" written in FastAPI or Java, with its own auth, monitoring, deployment pipeline, and database. What they need is to **add retrieval to that existing service**, not stand up a separate platform next to it.

| What enterprises care about | Library | Platform (RAGFlow / Dify) |
|---|---|---|
| Plugging into existing systems (auth, logging, monitoring) | Written directly in their own code | Have to adapt to the platform's way |
| Custom flows (special chunking, custom scoring, multi-step retrieval) | Change it however you want | Only the options the platform provides |
| Where data lives | Use the company's existing database, e.g. the Postgres they already have | Platform bundles MySQL, MinIO, Infinity; one more stack to operate |
| Testing and versioning | It's just ordinary code: unit tests, code review | Lots of config is clicked in a web UI, hard to track |
| Scaling | Scales with their own service | Platform scales separately |
| Lock-in risk | Low; switching libraries isn't costly | High; data and flows all live in the platform |

When they actually expose a service, enterprises wrap **their whole application** (e.g. a "document Q&A API"); LangChain is just part of the code, not something they wrap a layer around specifically.

## So when does a platform fit?

- You want to **build it fast** without writing much code;
- The users **aren't engineers**, e.g. a business team uploading documents and managing the knowledge base in a web UI;
- Documents are in complex formats (scanned PDFs, tables), and you value the platform's strong built-in parsing;
- One knowledge base needs to be shared by **many different applications**.

## Your last question

LangChain and LlamaIndex aren't against being deployed as services; both offer a "deploy as a service" option: LangChain has LangGraph Platform (earlier called LangServe), LlamaIndex has LlamaDeploy and LlamaCloud. But these are **optional**; the default usage is still embedding them in your code as a library.

**One-line summary: library = put the capability into your program; platform = your program asks someone else for the capability.** Enterprises already have their own programs, so most pick the former; if you want to stand up a knowledge base quickly with zero code, pick the latter.

---

That decision wasn't made in this session; it was in the session you forked off. What I can see is the record in memory, which roughly says:

> 2026-10-21: switched retrieval to self-hosted RAGFlow 1.0. What you want is **a modern, resume-worthy retrieval pipeline built with an off-the-shelf framework**, no more hand-writing it piece by piece; accuracy gains are secondary.

So the motivation back then wasn't "RAGFlow is technically better than LangChain," but:

1. **Stage 1 retrieval was all hand-written**: your own BM25, RRF, and chunking. You wanted to switch to a mature industry solution, so your resume has an experience of "building a production-grade pipeline with a framework."
2. **RAGFlow has the production-grade stages built in**: structure-aware chunking, parent-child chunks, LLM auto-generating keywords and questions for each chunk, hybrid retrieval, hooking up a rerank model; basically all switches and parameters. With LangChain most of these can be done too, but features like "generate keywords and questions for each chunk" require writing your own code.
3. **Easy stage-by-stage comparison**: same interface; change parameters to switch between "pure vector," "hybrid," and "plus rerank," so that stage-by-stage comparison table was easy to produce.

## Looking back, was it the right choice?

**LangChain or LlamaIndex could do it all too**, and more lightly: no 6 containers, retrieval runs inside your process. At your scale, they're actually the better technical fit.

RAGFlow's value is mostly in the **interview narrative**, and the story you have now is actually more complete than using any single tool:

> "I built the baseline by hand first (BM25, RRF, windows), then moved to a full RAG platform, RAGFlow, to get production stages: structure-aware parent-child chunking, LLM chunk enrichment, hybrid search, rerank. I measured each stage. In hindsight, at 3,800 verses a library like LlamaIndex with an in-process index would be lighter. The platform's value is for multi-user, document-heavy deployments."

"Hand-wrote it, used a platform, and know when a library is the right call": you can speak to all three layers, which is more convincing than just saying "I used LangChain."

If the JDs for your target roles explicitly list LangChain or LlamaIndex, it's not too late to switch then: the retrieval layer's interface is thin, so swapping the implementation isn't a big deal. No need to touch it now.

---

RRF stands for **Reciprocal Rank Fusion**. The problem it solves: **two retrieval methods each produce a ranking; how do you merge them into one?**

## Why it's needed

Hybrid retrieval has two paths:

- **BM25 (keyword)**: scores like 12.7 or 8.3, with no upper bound;
- **Vector retrieval**: scores are cosine similarity, between 0 and 1, e.g. 0.82, 0.79.

The two scores are on completely different scales, so **you can't just add them**. 12.7 plus 0.82 is almost entirely determined by BM25.

## How RRF works: ignore scores, look only at rank

Each passage's final score is:

> compute `1 / (60 + rank)` in each ranking, then add them up

60 is a constant whose job is to keep the gap between rank 1 and rank 2 from being too large.

**For example**, with three passages:

| Passage | BM25 rank | Vector rank | RRF score |
|---|---|---|---|
| A | 1 | 5 | 1/61 + 1/65 = 0.0318 |
| B | 3 | 2 | 1/63 + 1/62 = **0.0320** |
| C | 2 | Not in top 50 | 1/62 + 0 = 0.0161 |

B ends up first. It's fairly high in both rankings, while A only took first in one. **RRF rewards results that "both sides agree on."**

## In your project

- **Stage 1**: I hand-wrote RRF in `retrieval.py`, about 5 lines of code. But back then hybrid retrieval was **actually worse than pure vector**. The reason: BM25 does poorly on conversational questions, and RRF gives both paths equal weight, dragging down the ranking of the good path. That was one of stage 1's findings.
- **Stage 4**: RAGFlow doesn't use RRF; it uses a **weighted merge**: normalize both scores first, then add them with weights "full-text 0.3, vector 0.7." The upside is you can give the more reliable path more weight.

If asked in an interview, you can say:

> "RRF merges rankings by rank instead of score, so you don't have to calibrate BM25 against cosine. It's the safe default for hybrid search. Its weakness is equal weighting, and in my baseline that let a weak BM25 drag the dense ranking down."
