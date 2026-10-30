> English version of [OpenSearchRelated.md](../OpenSearchRelated.md). Original notes are written in Chinese/English mix.

In this age of AI-assisted learning, isn't everything a one-night job? If one night isn't enough, two nights; if two isn't enough, three. Three nights really is the limit; that's about it.


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
