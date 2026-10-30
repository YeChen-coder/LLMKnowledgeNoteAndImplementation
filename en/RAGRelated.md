> English version of [RAGRelated.md](../RAGRelated.md). Original notes are written in Chinese/English mix.

Top K is a general term, as in Top 5, Top 10.

But top A and top B aren't general terms like that. Usually it's just paper authors using A and B as two different count parameters, e.g. A = 30, B = 50, not a universal, general item.

During search, instead of keeping only the single best result so far, it keeps the B most promising paths at the same time. This is so you know not only what result you ended up with, but also how you got there along the way, i.e. the reasoning/retrieval path. These are the Top-B paths in Beam Search, a concept from the knowledge graph side. -Honestly, I didn't get it. I only know it's graph-related, but it's really long and really academic. I'll come back to it when the project's complexity or requirements actually need it; it's not what I'm doing now.

# RAG has evolved

The old approach was: chunk documents, embedding, vector database, pick top k (k = 5 or 10, doesn't matter), then send it to the language model.

Now it's evolved. The core has shifted from vector search to Hybrid Retrieve → Rerank → Context Engineering → Generate → Evaluate.

To add: RAG is defined as a retrieval system, and its scope goes far beyond just a vector DB. Anything involving retrieval, whether via API or SQL, counts as RAG. - Oh, I get it now, so that's why interviews ask about RAG. Back then I thought about it for ages; my understanding of RAG was only embeddings and vector DBs. I thought, how much data could there be, is RAG even worth it? How much data is there that would justify this on a phone-call AI agent system? But if the definition already includes data from SQL and APIs, then I had it wrong, RAG really is needed, and failing that interview was fair.

<img width="338" height="332" alt="image" src="https://github.com/user-attachments/assets/06da8b5b-bb15-43ad-a87f-8286f58513de" />

Production recommendations from Microsoft, AWS, Google, and Databricks have now converged heavily: Hybrid Search, metadata filtering, reranking, structured chunks, and evaluation are the backbone; only complex questions get escalated to agentic retrieval. - Sorry, a rant. Since this is already such a super well-defined problem, and cloud providers and top tech companies have all published tutorials like this, who knows how many good, mature open-source frameworks there are on GitHub. Just use theirs; why bother? This has already reached the point of standardization; it's not something you need to write from scratch. Use their framework, understand what it does, then tweak it. Learn what you need as you need it. There's really no need to reinvent the wheel. For everything, find the simplest way and just grow with SOTA.

# https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-information-retrieval?utm_source=chatgpt.com

                2026 Production RAG / Enterprise RAG

──────────── OFFLINE / INGESTION ────────────

PDF / Word / HTML / DB / Wiki / Tickets
                │
                ▼
      Layout-aware Parsing
      title, paragraph, table, image, page number
                │
                ▼
       Clean + Deduplicate
                │
                ▼
   Structure-aware Chunking
                │
                ▼
     Chunk Enrichment
 add context / metadata to chunks
 title / section / date / ACL / version
                │
        ┌───────┴────────┐
        ▼                ▼
 Dense Embedding       Lexical Index
   vector index       BM25 / inverted index
        │                │
        └───────┬────────┘
                ▼
          Search Index
      enterprise knowledge retrieval layer


──────────── ONLINE / QUERY ───────────────────

User Question
     │
     ▼
Auth / ACL / Tenant
identity, permissions, tenant
     │
     ▼
Query Understanding
     │
     ├── Rewrite
     ├── Filter / extract conditions
     ├── Decompose / split the question
     └── Route / pick knowledge source
     │
     ▼
┌──────────── Hybrid Retrieval ──────────────┐
│                                             │
│   Dense Vector Search     BM25 Search       │
│     semantic search       keyword search    │
│          │                    │             │
└──────────┴─────────┬──────────┘
                     ▼
              RRF / Fusion
                     │
                     ▼
             Top 20~50 chunks
               broad recall
                     │
                     ▼
                RERANKER
                fine ranking
                     │
                     ▼
              Top 3~10 chunks
           high-quality evidence
                     │
                     ▼
     Dedup / Parent Expansion / Context Pack
                     │
                     ▼
                    LLM
                     │
                     ▼
       Answer + Citations + Abstention
     answer + citations + refuse when unsure
                     │
                     ▼
       Logging / Eval / Observability
         evaluation + monitoring + feedback


## Changes 

### Hybrid = ( dense retrieval / semantic retrieval +  BM25 / Lexical Search ）-> Reciprocal Rank Fusion

Before, it was vector search only: embed everything and search the vector DB. That's called dense retrieval / semantic retrieval.

But that has problems. For example, if the user searches only for a code name, and the code name has no semantic meaning at all, you should really use traditional search, i.e. BM25 (Lexical /ˈleksɪkl/ Search).

Since both are now used, merging the results from the two different searches is called Reciprocal /rɪˈsɪprəkl/ Rank Fusion. -This hasn't changed much; everyone was already using hybrid a year or two ago. ChatGPT recommended hybrid to me back then, so this idea really hasn't changed.

<img width="316" height="400" alt="image" src="https://github.com/user-attachments/assets/bcfd82e7-b6d0-4b7c-a63d-ca07bac32769" />

### Reranking = 50 candidates -> Reranker -> 5 really useful chunks ; Multi-stage Retrieval

This is different; another step was added.

The more primitive approach from long ago was to take the Top 5 or Top 10 directly and take those chunks straight to the last step, feeding them to the final LLM.

Not anymore. Now there's an extra step in the middle:

1. The first step fetches more: not just 5 or 10. To make sure no possibly-correct answer is missed, the first step fetches 50.
   
2. A filtering step in the middle: the problem is you can't feed all 50 chunks to the final LLM, so a step is added that compares the user's query against the retrieved chunks, re-selects the Top 5, and then sends those on.

<img width="826" height="235" alt="image" src="https://github.com/user-attachments/assets/912ac77d-6cb3-4350-9f9e-3a5087e1846e" />


Multi-stage Retrieval = Cheap + Fast retrieval (cheap, fast, wide search) -> Expensive + Accurate reranking (a bit pricier, but only processes a few dozen)

### Chunking: Layout-aware / Structure-aware Parsing

On chunking, nobody has used fixed lengths for a long time. Fixed lengths are bound to cause trouble: what do you do when a sentence gets cut in half? So there's not much to change here. It used to be chunk size = this much, chunk overlap = that much; nobody does that anymore.

What's emphasized now first is: Layout-aware / Structure-aware Parsing

Take the most troublesome case, PDFs: the crudest early version just converted PDF to text; now the approach is to turn the PDF's title, heading, paragraph, table, list, image, footnote, page number, etc. all into a structured representation.

Google explicitly recommends enabling the layout parser /'pɑ:sə/ for PDF/DOCX/PPTX/XLSX used in RAG; Microsoft also includes document structure and semantic/structure-aware chunking in current RAG design.

Done this way, a chunk keeps: a blob of text + what it is + where it came from + who it belongs to + who can see it.
<img width="822" height="428" alt="image" src="https://github.com/user-attachments/assets/3cec692f-2c4a-4870-92ca-3371a272759b" />

One more thing: the parent-child idea, which is also a widely accepted, mature approach.

For example, a document is the parent and its small paragraphs are children. Search generally targets children, because small chunks match more precisely. The current approach: once a small chunk is matched, instead of feeding just that chunk to the model, it finds the parent one level up, grabs the nearby context, and sends that to the model.

This is easy: when the chunks were cut, metadata like page number and paragraph was kept, so looking up and down is easy. It's basically a doubly linked list.

Because of this, one principle of the current approach is: small chunks are for precise matching, since they match easily, are easy to find, and score better on similarity. But finding it isn't enough. If it matched, the parent must have something useful, so you also pull out the slightly bigger parent block to help the model understand more. Azure even treats the chunk ↔ parent relationship as a mainstream RAG indexing pattern now.

### Metadata Filter; Filter first / Filter during retrieval

This is mainly because of authentication; it adds an extra gate.

But honestly, if permissions were sorted out during authentication early on, this step wouldn't be needed at all. But if they want to add it, fine.

Its main purpose is to make sure the user making the query doesn't see confidential stuff they shouldn't see.

AWS's current enterprise security guidance explicitly requires multi-layer protection across ingestion, storage, retrieval, and inference, with metadata filtering / RBAC at the retrieval stage. Azure also supports permission-aware retrieval like query-time ACL/RBAC. -That's more like it! Set up RBAC right from the start. How could anyone not get RBAC right at the start, let chunks already contain information the user shouldn't know, and then go filter it afterward? How can something so backwards happen?

### Contextual Embedding; ingestion enrichment

This is largely the same as saving metadata while cutting chunks as part of chunking, mentioned above, but this way is a bit better.

Because with the earlier way, the information available is still limited. Real contextual chunking lets the model understand the chunk in light of the whole document and its context (e.g. what "this company" actually refers to, which year's report this is), and generate a short description explaining what this chunk actually is in the original text.

It certainly takes a lot more compute, but the overall result will definitely be better. And since a document is usually only chunked once and embedded once, this one-time cost is worth it.

In short, the flow here is: cut chunks first, then do enrichment to add context or a description of the **whole** document, and finally send the enriched chunks to embedding.

### Query Rewrite / Query Decomposition /ˌdiːˌkɑːmpəˈzɪʃn/

As the old saying goes: users don't know what they want.

Embedding the user's query directly and retrieving with it really doesn't work well. So now it's processed and rewritten first. For example, if the user asks "What's the difference between A and B?", it's actually quite hard to find a comparison of A and B directly in the documents. Thinking like a person, you'd first need to know what A is and what B is, and only after getting both pieces of data talk about the comparison.

The system side follows the same logic:

1. First split into subqueries, separately looking at what A is and what B is
2. Run retrieval for each, getting evidence A and evidence B
3. Finally do the compare

Microsoft's current RAG Architecture Guide already puts query augmentation, rewrite, decomposition, multiple searches, and reranking into one complete retrieval pipeline.

The logic of modern RAG: after getting the user's query (i.e. question), first run a research planner, then go to the search engine.

Rather than embedding the query directly and then searching away with that embedding, a step of thinking and breaking down the question is added before that.

### Agentic RAG for complex questions, like multi-step reasoning

Agentic RAG adds a loop and brings in LLM-as-a-Judge; underneath it's still the same agent orchestration.

The problem it solves: the first pass's results are wrong, insufficient, or missing something. So the logic is still: first run the previous pipeline to get a research A result; if it's not enough after looking, go look up B; if after B it's still not right and something is missing (like a date), go look up C. -I keep feeling a very important reason this became possible is the big jump in AI intelligence. In the past, AI wasn't capable of judging whether the information it gathered was enough.

It's really just adding this loop in the middle.

Microsoft now defines it very clearly: suited for multi-step reasoning, dynamic source selection, query decomposition, iterative refinement, and retrieval + action. Conversely, if a question can be solved with one search, Microsoft directly recommends: use standard RAG; don't go agentic just for the sake of it.

### GraphRAG fits relationship-heavy / multi-hop questions

This is mainly for web-like network relationships, one-to-many, many-to-many, where everything is connected to everything.

For example, how a company's suppliers are doing, what happens if the company's upstream changes, or the downstream changes: that kind of networked graph structure.

GraphRAG hasn't become the default foundation for all enterprise RAG.

More interestingly, Microsoft's own official GraphRAG GitHub now explicitly says it's a research project, currently largely in maintenance mode, and won't keep being pushed as a fast-growing new platform.

Its development has been mediocre, honestly pretty mediocre; its future is uncertain, its prospects questionable. - I thought about it: is it because in real scenarios, people usually build this kind of RAG for one specific company? There really aren't many upstream/downstream relationships; two levels up and two levels down at most and that's it. What really complicated relationships could there be? Most companies aren't that intricate. So it doesn't have that much general applicability, and it makes sense that people don't see it as that important.

### ColBERT / Late Interaction

This one is genuinely new. It's for when the quality from the hybrid plus reranker setup still isn't good enough, and you invest more resources to use it.

With ordinary embedding, you take a whole chunk, regardless of what's in it or how it was processed, whether or not you did ingestion enrichment or put in metadata, doesn't matter, and make the whole chunk one vector. Think of it as compressing all of that chunk's information into a single point in vector space.

But the newer ColBERT method maps each token of the chunk to its own independent vector. At query time too, each token of the query finds its own vector, and then late interaction happens.

Instead of just comparing the overall similarity of the query and the chunk, different parts of the query (i.e. different tokens) each find the best-matching part of the document. Obviously that's more precise; it will definitely find very precise matches.

But the trade-off is that storage and compute are both noticeably higher:

1. Compute:
It's not a one-time cost at document processing; it's actually just as high on every query. You have to align each query token against each token vector in the chunk one by one. It's not a one-time overhead, and the compute cost really is big.

2. Storage:
Before, one chunk mapped to one vector, one string; now each token in the chunk has its own independent vector, so storage multiplies by however many tokens there are. It balloons a lot.

Weaviate's multi-vector / ColBERT-style retrieval is now production-ready/GA, not just a paper toy.

### SPLADE

This is new too. BM25 (BM = Best Match / Best Matching) finds exactly the words the user searched for; pure search. But SPLADE sits between pure-keyword BM25 and Dense Embedding.

Qdrant and others now officially support learned sparse retrieval like SPLADE.

But in reality: BM25 + Dense + Reranker is still the more conservative, more mature default.

SPLADE is an upgrade worth trying when you're doing retrieval tuning.



<img width="818" height="538" alt="image" src="https://github.com/user-attachments/assets/91f0a1a3-ea78-42d7-9f3c-2f6017212f55" />

A quick note: BM25 means that when the user searches (e.g. for cybersecurity), it checks whether the query term appears in the document, how many times, how rare the term is, and whether the document is especially long, to judge how relevant the current chunk is to the user's query. Main factors:
TF = Term Frequency: how many times the term appears in the document

IDF = Inverse Document Frequency: how rare the term is (-it looks at how rare the term is, to deal with words like "the" in the query that appear everywhere; a query term that only a few documents contain is highly distinctive, so it scores high); also, appearing 10 times doesn't score 10 times higher than appearing once; there's some processing. Probably some exponential-style function; anyway there's a ceiling, and after a certain point the extra score flattens out a lot. This is BM25's Term Frequency Saturation.

Document Length

SPLADE sits in the middle: it uses a neural network to understand meaning, but still represents the result in terms of "words." Let me explain that sentence: it uses some method to semantically understand words with the same meaning, which gives it the ability to recognize that car, automobile, and vehicle are about the same thing. But when representing it, it doesn't use the dense approach (a long vector); like BM25, it's still just a scalar per word. That's why it's said to understand meaning with a neural network but represent the result with a word-based data structure.

SPLADE stands for: SParse Lexical AnD Expansion

It uses a mechanism similar to a BERT masked-language-model head to learn weights and expansions for words in the vocabulary, then uses sparsity regularization to force the final representation to stay sparse. - I honestly didn't understand this sentence, but what it's explaining is what I described above.

I read an explanation. I'm not sure I can state it fully and accurately, so here's just my understanding:

First, it processes the tokens in the query in some way. Originally a token is like a small dot, but because of the expansion, it's no longer just a dot; there's now a fuzzy ring around it, which lets it overlap a lot with similar words.

For example, car and automobile were originally two separate dots, but after SPLADE processing each grows a blurred ring. So when they're compared, there's an area of overlap, which increases the ability to find it.

- That's enough on SPLADE for now. Just know SPLADE is better than BM25, and BM25 is a caveman-era antique.

### Multimodel RAG

Same old problem: for documents with images, tables, scanned pages, or flowcharts, the old plain-text TXT approaches just don't work.

So now there's multimodal RAG. The simplest case is taking a whole PDF page as an image for multimodal embedding, retrieving the whole page directly, and then giving it to a visual language model.

### Evaluation

This is standardizing evaluation: computing recall, precision, ranking, this and that.

This kind of redefinition and standardization of evaluation is actually quite common; for any AI system, everyone more or less looks at similar results.

<img width="830" height="565" alt="image" src="https://github.com/user-attachments/assets/76764e79-a866-4909-86d1-88a813817c12" />

AWS now also directly provides RAG metrics like correctness, completeness, faithfulness, citation precision, and citation coverage.

# In short, the flow is Parse → Chunk → Enrich → Hybrid Retrieve → Rerank → Pack Context → Generate → Cite → Evaluate

pack_context here means further organizing the 5 chunks that come out of reranking.

Specifically:

1. Dedup and version handling: e.g. deduplicate; if one of them comes from an old version, handle versions properly.
2. Add context: i.e. parent expansion. Sometimes titles need to be added too.
3. Reorder: if two chunks come from the same article, reorder them to sit together; if they're from the same section with nearby content, order them while you're at it.
4. Control token count: limit the token input to the large language model downstream.
5. Keep the sources, i.e. keep the evidence's metadata, like document ID, page, section, URL, and so on.

Once all of that is done, the context package is sent to the downstream model to generate the answer and translation. (Note in passing: • Context Packing happens before sending to the final LLM for generation; the goal is to make the already-retrieved chunks better suited for the model to read
• Contextual Embedding happens on the retrieval side (when making chunks); the goal is to make each chunk easier to find)


 Enterprise Data Sources
                         │
                         ▼
             Parse / OCR / Layout
                         │
                         ▼
             Normalize + Dedup
                         │
                         ▼
            Structure-aware Chunk
                         │
                         ▼
       Metadata + Parent + ACL + Version
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
          BM25 Index            Dense Index
       keyword inverted index  vector index
              │                     │
              └──────────┬──────────┘
                         │
──────────────── QUERY ──┼──────────────────
                         │
User
 │
 ▼
Authentication
 │
 ▼
Query Router / Planner
 │
 ├── simple → one query
 │
 └── complex → decomposition
                         │
                         ▼
                  ACL + Filters
              permission filtering
                         │
                         ▼
              ┌──────────┴──────────┐
              ▼                     ▼
             BM25                 Dense
              │                     │
              └──────────┬──────────┘
                         ▼
                        RRF
                         ▼
                   Top 20~50
                         ▼
                      Rerank
                         ▼
                    Top 3~10
                         ▼
       Dedup + Parent Context + Citations
                         ▼
                        LLM
                         ▼
            Grounded Answer / Refuse
     answer with evidence / no evidence, no answer
                         ▼
               Logs + Evaluation


- Final note: if you really need RAG to do real work, don't bother building it; just use the big companies' stuff. Their current pipeline frameworks are proven and have become mature standards, so don't piece it together yourself. Isn't it better to walk the road others have already paved? How exactly it's orchestrated inside isn't something developers need to agonize over. You just need to know what it means and how to use it, which methods are better than which, and what the trade-offs are.

---

For these design projects, I used RAGFlow to set up the RAG.

I figured out why, when I did the RAG project before, the whole process slid through my head like flowing water: it went so smoothly that I wasn't tortured at all, and in the end I had no impression of it. I didn't even know what I'd done.

It's because the framework is so well made now that they've already set everything up for you. All you do is fill in the model API, pick which default model you want, and you're done. No need to assemble anything; the whole pipeline is ready. No wonder it doesn't leave an impression. Oh well.

---
