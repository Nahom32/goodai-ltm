# Session Q&A Export

This file captures the user-visible questions and final assistant answers from this Codex CLI session. Intermediate progress updates are omitted.

## 2. Main Project Flow

> What is the main flow of the things in this folder? Start with what the project is and how it starts execution and then the entire flow. This should just be an introduction.

This repository is primarily a Python library, not a single executable app. Its purpose is to provide long-term text memory for LLM-based agents: store text, chunk it, embed it, index it, retrieve relevant passages later, and optionally wrap that in a chat agent (`LTMAgent`).

There are really two main ways execution starts:

1. Library usage inside Python code, such as:

```python
from goodai.ltm.mem.auto import AutoTextMemory
mem = AutoTextMemory.create()
```

or:

```python
from goodai.ltm.agent import LTMAgent
agent = LTMAgent(model="...")
response = agent.reply("...")
```

2. Utility scripts in `scripts/` and examples in `examples/`, which are just thin entry points that exercise the library.

So the actual “main flow” starts in the objects under `goodai/ltm/`, not in a CLI bootstrap.

At the center of the project is the memory stack:

1. `AutoTextMemory.create()` in `goodai/ltm/mem/auto.py` builds a memory instance.
2. It chooses defaults for tokenizer, embedding model, device, and vector DB.
3. It returns `DefaultTextMemory`, implemented in `goodai/ltm/mem/default.py`.
4. `DefaultTextMemory` manages:
   - chunking incoming text into memory units
   - embedding those chunks
   - storing vectors in either a simple DB or FAISS
   - retrieving and expanding relevant chunks later

When text goes in, the flow is roughly:

`add_text()` -> chunk queue updates -> embeddings generated -> vector index updated

When a query comes in, the flow is roughly:

`retrieve()` / `retrieve_multiple()` -> query embedding -> vector search -> dedupe/overlap filtering -> optional reranking -> final `RetrievedMemory` results

`LTMAgent` in `goodai/ltm/agent.py` sits on top of that memory layer. Its `reply()` flow is:

1. Read the current chat session.
2. Build an LLM prompt from recent conversation plus retrieved memory excerpts.
3. Call the LLM through `litellm.completion(...)`.
4. Save the user message and assistant reply back into conversation memory.
5. Reuse that memory on future turns.

So, conceptually: this repo is a memory engine first, and an agent wrapper second. The core path is `input text -> chunk/embed/index -> query -> retrieve passages -> optionally inject into agent prompt -> LLM reply`.

## 3. How This Creates LTM

> How does this memory create an LTM? Walk me through an example scenario and how this package could be at play.

In this package, “long-term memory” is not a separate reasoning system. It is a retrieval-backed text store that lets an agent keep prior information, turn it into searchable chunks, and pull back the most relevant passages later.

The flow starts when you create a memory through `AutoTextMemory.create()`. That factory picks a tokenizer, an embedding model, a device, and a vector index, then returns a `DefaultTextMemory`. Internally, that memory owns two important things: a `ChunkQueue`, which stores text as overlapping token chunks, and a vector DB, which stores embeddings for those chunks.

A concrete example:

```python
from goodai.ltm.agent import LTMAgent

agent = LTMAgent(model="gpt-4o-mini")
agent.add_knowledge("The user is allergic to peanuts.")
agent.reply("I usually buy oat milk and bananas.")
agent.reply("What should I remember about my groceries?")
```

Under the hood:

1. `add_knowledge(...)` or stored conversation text goes into `add_text()`.
2. The text is tokenized and inserted into the `ChunkQueue`, which breaks the stream into chunks with overlap.
3. New chunks are embedded in `_ensure_keys_added()` and written into the vector index.
4. Later, when a query arrives, the query text is embedded and searched against that vector index in `retrieve_multiple()`.
5. The memory removes duplicates, expands chunks into more readable passages, optionally reranks them, and returns `RetrievedMemory` objects.
6. If you are using `LTMAgent`, `reply()` builds the prompt, injects those retrieved excerpts, calls the LLM, and then stores the new user and assistant messages back into conversation memory.

So in the shopping example, the package is doing three jobs for the agent:

- turning raw text into indexed memory
- finding the right past facts when a new question arrives
- feeding those facts back into the LLM prompt so the model appears to “remember”

That is the practical meaning of LTM here: persistent, searchable context stitched into LLM responses.

## 4. Can Stored Facts Be Removed?

> Does it have any way of removing the LTM stored fact?

Yes, but it depends on which layer you use.

At the raw memory level, individual stored text can be removed. `DefaultTextMemory.delete_text()` deletes by `text_key`, and internally it just replaces that stored sequence with an empty string. You can also update a stored fact with `replace_text()`.

Example:

```python
mem = AutoTextMemory.create()
key = mem.add_text("The user is allergic to peanuts.")
mem.delete_text(key)
```

At the `LTMAgent` level, the API is more limited. `add_knowledge()` does not return the `text_key`, so the agent wrapper does not give you a built-in “remove this one fact” operation. What it does expose is bulk removal:

- `clear_knowledge()` clears the whole knowledge base
- `clear_conversation_memory()` clears conversational LTM
- `reset()` clears everything

So the practical answer is:

- Yes, the memory system supports deleting a specific stored item.
- No, the high-level `LTMAgent.add_knowledge()` flow does not currently make single-fact deletion convenient unless you manage the underlying memory keys yourself.

## 5. Importance, Knowledge Storage, and Working Memory

> Does it have any sort of importance being calculated when a fact is stored into the knowledge base? Is that necessary since we are also embedding it in a vector db? Where does the immediate working memory get stored?

By default, no importance score is calculated when the agent stores a fact in the knowledge base.

`LTMAgent` creates `kb_mem` and `convo_mem` with `AutoTextMemory.create(...)`, but it does not pass an `importance_model` there, so stored chunks get `importance=None` in the default setup. The fact is still tokenized, chunked, embedded, and inserted into the vector DB, so semantic retrieval works without importance.

Importance is a separate signal, not a replacement for embeddings. Embeddings answer “what is semantically similar to this query?” Importance answers “even if several items are relevant, which ones matter more?” This repo supports that as an optional extension through `importance_model`, and there is a built-in `StanfordImportanceModel` plus `StanfordReranker` that combine relevance, recency, and importance.

For “immediate working memory,” there are really three stores:

- Current chat/session history is kept in `LTMAgentSession.message_history`, which is just an in-memory Python list, not automatically a file or vector DB.
- `user_info` and `wm_scratchpad` are also plain in-memory fields on the agent, not vectorized.
- Long-term conversation/knowledge memory lives in `convo_mem` and `kb_mem`, which use both a chunk store and a vector DB.

Nothing is automatically written to disk. If you want persistence, you must call `state_as_text()` and save that string yourself. That serialized state includes the chunk queue, vector DB state, and agent fields like `user_info` and `wm_scratchpad`.

## 6. Is the Vector DB Also in Memory? Is Saved State Serialized?

> Wait is the vector db also in memory? And is the saved state serialized?

Yes. By default, the vector DB is also in memory.

`AutoTextMemory.create()` defaults to `VectorDbType.SIMPLE`, and that creates a `SimpleVectorDb` object that just holds `numpy` arrays in RAM: `all_vectors` and `all_ids`. If you switch to `FAISS_FLAT_L2`, that is still an in-process memory index by default, just using FAISS instead of raw numpy. So out of the box, this is not a separate server-backed vector database.

And yes, the saved state is serialized.

`DefaultTextMemory.state_as_text()` returns a JSON string produced with `json.dumps(...)`, containing the config, chunk queue, tokenizer name, embedding model info, and vector DB state. `LTMAgent.state_as_text()` also returns a serialized JSON string that includes both memories plus `user_info` and `wm_scratchpad`.

There are also explicit persistence helpers if you want files on disk:

- JSON-based persistence
- MsgPack-based persistence
- Hybrid formats that store vectors in `.npz`

So the practical model is:

- runtime storage: in-memory chunk queue + in-memory vector index
- persistence: serialized state string or saved files if you explicitly call the persistence layer

## 7. What Importance Model Is Proposed?

> Back to the importance model, what kind of importance model has they proposed usage even if it is not a default importance model?

Yes. The built-in “proposed” importance path is the Stanford-style one in `goodai/ltm/reranking/stanford.py`.

The intended pattern is:

- score each stored memory for `importance` with `StanfordImportanceModel`
- combine that score with `relevance` and `recency` at retrieval time using `StanfordReranker`

`StanfordImportanceModel` is based on the Generative Agents paper and asks an LLM to rate how “poignant” a memory is on a 1-10 scale, then normalizes that to `0..1`. So this is not a learned vector-space importance model. It is an LLM-scored salience signal computed when the fact is stored.

The reranker then computes a combined score from:

- relevance
- recency
- importance

The repo’s example usage is in `examples/custom_reranker.py`:

```python
reranker = StanfordReranker(half_life=1.0, use_importance=True, time_fn=custom_timestamp)
importance_model = StanfordImportanceModel()
mem = AutoTextMemory.create(reranker=reranker, importance_model=importance_model, config=config)
```

A few practical notes:

- The example file currently sets `_use_importance_model = False`, so it demonstrates the wiring but does not enable it by default.
- The baked-in default for `StanfordImportanceModel` uses `text-davinci-003` in code, which is dated and would likely need replacement in real use.
- The system also supports fully custom importance models by implementing `BaseImportanceModel`.

So the main proposed usage is: use embeddings to find candidate memories, then use importance plus recency to prioritize which relevant memories should surface first.

## 8. How Importance and Recency Work

> How does the importance model calculate the importance of something out of nowhere? What is the process? In addition what is the recency decay and what does that even mean.

It does not calculate importance “out of nowhere.” In this repo, the built-in importance model calculates it by asking another language model to judge the memory text when the memory is stored.

The flow is:

1. You add text to memory.
2. If an `importance_model` is attached, `DefaultTextMemory._replace_or_add_text()` calls `importance_model.get_importance(text)`.
3. The built-in `StanfordImportanceModel` builds a prompt like: “Rate how poignant this memory is from 1 to 10.”
4. It sends that prompt to a text-generation model.
5. It parses the model’s numeric answer, clamps it to `1..10`, then normalizes it to `0..1`.
6. That score is stored on the chunk as `importance`.

So the importance is not derived from the embedding vector itself. It is an extra LLM judgment about how significant the memory seems.

Recency decay is a separate idea. It means: older memories should count less than newer ones, unless they are very relevant or important.

In `StanfordReranker`, recency is computed from:

`elapsed = current_time - memory.timestamp`

Then that elapsed time is converted into a score between near `1` and near `0`:

- Exponential decay: `exp(-(elapsed * coeff))`
- Inverse decay: `half_life / (half_life + elapsed)`

The `half_life` means “after this much time, recency should drop to 0.5.”

Then retrieval ranking combines three signals:

- `relevance`
- `importance`
- `recency`

This lets the system prefer something that is not only similar, but also recent or important.

## 9. What Context Does the Importance Model Get? What Is the Timestamp?

> What other context does the importance model get to rank the poignancy? For the recency decay what is the `memory.timestamp`, and do we store long-term memory anywhere else than the vector database?

By default, the built-in importance model gets almost no extra context.

`StanfordImportanceModel` builds a prompt containing only:

- the rating instruction
- two example categories in the instruction
- the raw memory text itself as `Memory: {mem_text}`

It does not pass neighboring memories, conversation history, user profile, or retrieval context. So the score is a standalone LLM judgment of the memory text.

For recency, `memory.timestamp` is the timestamp attached when that text was stored in the chunk queue. If you do not pass one, it defaults to `time.time()` at insertion time. That timestamp is written onto each created chunk. So yes, conceptually it is “when this memory was created/stored,” unless you explicitly provide a custom timestamp or replace the text later with a new one.

For long-term storage, the memory is not stored only in the vector DB.

There are two coordinated stores:

- The chunk store (`ChunkQueue`) keeps the actual tokenized text stream, chunk boundaries, timestamps, metadata, importance scores, and key mappings.
- The vector DB keeps embeddings plus chunk IDs for semantic search.

The vector DB cannot reconstruct the memory text by itself. It only knows “this vector corresponds to chunk id 37.” The actual text, timestamps, metadata, and importance live in the chunk queue.

## 10. What Does the Chunk Queue Look Like?

> What does the chunk queue look like, is it a file or how is the stuff stored there? If the same concept is mentioned in different contexts, how does the chunk id store that?

The `ChunkQueue` is not a file by default. It is an in-memory Python data structure.

Concretely, it holds:

- `token_ids`: one long list of token IDs for the stored text stream
- `chunks`: a list of `Chunk` objects
- `chunk_map`: `chunk_id -> Chunk`
- `sequence_map`: `text_key -> (from_token_seq_id, to_token_seq_id)`
- `separator_seq_ids`: boundaries between sections/documents

A `Chunk` itself is just metadata about a window into that token stream: start offset, end offset, timestamp, importance, metadata, and associated `text_key`s.

So the long-term memory is stored in two places at runtime:

- `ChunkQueue`: actual text content and structure
- vector DB: embeddings plus chunk IDs for search

For “the same concept mentioned in different contexts,” this system does not maintain a shared concept object. It does not know that two chunks are “about the same entity” in a symbolic sense. It only knows:

- this text was stored here
- this chunk has this `chunk_id`
- this chunk’s embedding is similar to some query

So if “Paris” appears in two different stored passages, the system will usually create separate chunks for those occurrences. Those chunks get different `chunk_id`s. If the passages are semantically similar, their embeddings may land near each other in the vector space, but they are still separate stored memories.

Retrieval removes duplicate `chunk_id`s and overlapping passages, but it does not merge semantically equivalent concepts into one shared representation.

## 11. What Is a Token ID, Why Multiple top_k Values, How Replacement Works

> What is a token id? What is the point of having three different `top_k` variables? Why is replacement necessary? What triggers it? Does replacement update the timestamp? How does overlap removal work? Can you give me a concrete example?

### Token IDs

A token ID is the integer form of a token.

- A token is the tokenizer’s text piece, such as a whole word, part of a word, punctuation, or a whitespace-marked piece.
- A token ID is that piece’s numeric vocabulary index.

In this repo, the default tokenizer is `distilroberta-base`. The flow is:

`raw text -> tokenizer -> token strings / token IDs -> stored in ChunkQueue.token_ids`

The actual stored stream is the integer list `ChunkQueue.token_ids`, not raw strings. When the code needs readable text, it slices those IDs and decodes them back to text.

### Why 3 top-k values

The system intentionally retrieves more than the final answer count.

1. `k`
   This is what the caller asked for: “give me 3 memories.”

2. `reranking_top_k`
   This is the candidate count after initial search but before final trimming.
   If a reranker or matching model exists, it becomes:
   `round(k * reranking_k_factor)`
   Otherwise it stays `k`.

3. `expansion_top_k`
   This is even larger because chunk expansion and overlap removal may throw candidates away.

4. `coverage_top_k`
   This is what gets sent to the vector DB search.
   It is larger still because one chunk may have multiple storage embeddings.

The reason is simple: the vector DB returns embedding hits, but the final result should be `k` non-redundant readable passages.

### Why replacement exists

Replacement is for editing or deleting an already stored text sequence.

Examples:

- A stored fact was wrong: “The user lives in Nairobi.”
  Replace with: “The user lives in Addis Ababa.”
- A memory is no longer wanted.
  Delete it by replacing it with empty text.

What triggers it:

- only an explicit API call
- the system does not auto-rewrite stored memories by default

During replacement, the code:

1. finds the old token span from `sequence_map`
2. removes that span
3. inserts the new token IDs
4. rebuilds affected boundary chunks
5. shifts later chunk offsets if length changed

Timestamp behavior:

- if you pass `timestamp=...`, that value is used
- if you omit it, the new inserted sequence gets `time.time()`
- neighboring preserved fragments keep their old timestamps

### How overlap removal works

This is not semantic deduplication. It is positional overlap filtering.

The code checks the token-index range of each candidate passage:

- `p_from`
- `p_to`

If too much of that range overlaps a previously accepted passage, it skips it.

So:

- same `chunk_id` repeated from multiple embeddings -> remove exact duplicate hit
- different `chunk_id`s but nearly same text span -> remove overlap duplicate

### Concrete example

Suppose you store these without a separator:

1. `text_key=10`: “Jake went to the hardware store.”
2. `text_key=11`: “He bought nails for Archie.”

The queue stores one shared `token_ids` stream containing both sentences. Chunks are sliding windows over that stream, for example:

- `chunk_id=100` -> “Jake went to the hardware”
- `chunk_id=101` -> “to the hardware store. He”
- `chunk_id=102` -> “store. He bought nails for”
- `chunk_id=103` -> “bought nails for Archie.”

`chunk_id=101` may reference both `text_key=10` and `text_key=11`. That does not mean the system merged two concepts. It only means one overlap window straddled the boundary.

Now query:

`"Where did Jake go?"`

Retrieval does this:

1. embed the query
2. ask vector DB for many candidate hits, not just 1
3. map returned chunk IDs back to chunks
4. expand chunks into readable passages
5. drop repeated chunk IDs
6. drop heavily overlapping passages
7. return top `k`

If later you discover sentence 2 was wrong and should be:

“He bought a hammer for Archie.”

You call:
`replace_text(11, "He bought a hammer for Archie.")`

The system:

- finds the token span for `text_key=11`
- swaps those token IDs
- rebuilds the chunks around that area
- shifts later offsets if length changed
- updates vector embeddings for affected chunks

If you omit `timestamp`, the replaced region gets a fresh current timestamp.

## 12. Add Importance Back Into the Example

> You left out the importance calculation in this example, can you add it back?

Yes. Add importance at the moment the text is stored.

Using the same example:

1. `text_key=10`: “Jake went to the hardware store.”
2. `text_key=11`: “He bought nails for Archie.”

If an `importance_model` is attached, then during `add_text(...)` the memory calls:

`importance_model.get_importance(text)`

With the built-in `StanfordImportanceModel`, the model sees only a prompt like:

- rate this memory from 1 to 10
- memory text: “Jake went to the hardware store.”

and separately:

- rate this memory from 1 to 10
- memory text: “He bought nails for Archie.”

Then it normalizes the answer to `0..1`.

So you might end up with something like:

- “Jake went to the hardware store.” -> importance `0.22`
- “He bought nails for Archie.” -> importance `0.44`

Those scores get attached to the affected chunks in the chunk queue, alongside timestamp and metadata.

Then later, when you query:

`"What did Jake do for Archie?"`

the retrieval flow is:

1. semantic search finds candidate chunk IDs from the vector DB
2. chunk IDs are resolved back to chunk/passage text
3. duplicate and overlap filtering runs
4. if a reranker is enabled, it combines:
   - relevance
   - recency
   - importance

So if two memories are both relevant, the reranker can prefer the one judged more significant.

If you later replace `text_key=11` with:

“He bought a hammer for Archie.”

then importance is recalculated for that new text if you pass it through `replace_text(...)`, because replacement goes through the same `_replace_or_add_text()` path that calls the importance model.

## 13. Where Recency Comes From

> Where did the recency comes from again?

Recency comes from the memory’s `timestamp`.

Each stored chunk has a `timestamp` field. When text is added, if no timestamp is provided, the chunk queue sets it to `time.time()` at insertion time. If the caller passes a custom timestamp, that custom value is used instead.

Later, the `StanfordReranker` computes:

- `elapsed = current_time - memory.timestamp`
- then converts that elapsed time into a recency score, such as exponential or inverse decay

So recency is not stored separately. It is derived at retrieval time from:

- “when was this memory stored?”
- “what time is it now?”

In short:

`recency = decay_function(now - timestamp)`

The newer the memory, the higher the recency score.

## 14. What Changed on `develop` Relative to `main`

> I just changed the branch to develop and there seems to be a lot of commit changes, can you take a look at the changes specific to this branch and let me know what changed from what we have talked so far?

`develop` is behaviorally different from `main`, but not across the whole stack we discussed. The core low-level memory engine is mostly the same: `ChunkQueue`, vector-search retrieval, importance/recency logic, and duplicate/overlap filtering are unchanged. The biggest changes are a new higher-level memory API and a new default embedding/tokenizer path.

The largest addition is `goodai/ltm/memory.py`, which introduces two wrapper APIs:

- `LTMSystem`: a higher-level interface around `AutoTextMemory` with `add_content`, `retrieve`, `retrieve_from_keywords`, `temporal_retrieval`, and `replace_content`.
- `RealTimeLTMSystem`: a multiprocessing wrapper that pushes additions and embedding work through queues/subprocesses for lower-latency use.

This changes one important thing from our earlier discussion: at the high-level `LTMSystem` layer, every `add_content(...)` call automatically adds a separator right after storing the content. So while the underlying `ChunkQueue` still supports chunks that span multiple `text_key`s, the new wrapper is explicitly trying to keep each memory item isolated. That makes the “one chunk can belong to multiple text keys” point less common in the new intended API.

`develop` also adds a keyword path. There is a new keyword extractor in `goodai/ltm/keywords/keywords.py` using `spaCy` plus TF-IDF, and `LTMSystem` keeps a `keyword_index` so you can retrieve by explicit keywords instead of semantic similarity alone. That is new relative to what we discussed before.

There is also new temporal navigation by timestamp. `LTMSystem.temporal_retrieval(...)` lets you fetch neighboring memories by `lookback_hops` and `lookahead_hops` around a target timestamp, and `replace_content(...)` lets you replace a stored item by locating it through its timestamp rather than its `text_key`. That lines up with our earlier discussion of timestamps and recency, but now there is an explicit API that uses timestamps as an addressing mechanism, not just for reranking.

The default tokenizer/embedder assumptions changed in `goodai/ltm/mem/auto.py`. Instead of defaulting to `distilroberta-base` plus the repo’s older embedding model flow, `develop` now points at `avsolatorio/GIST-Embedding-v0` and introduces a `SentenceTransformerEmbeddingModel` default. There is also a new `RemoteEmbeddingModel` for out-of-process embedding calls. Requirements were updated accordingly: newer `transformers`, newer `FlagEmbedding`, and new `spacy` dependencies.

`DefaultTextMemory.set_state()` now normalizes `sequence_map` keys back to `int`, which fixes deserialization compatibility for saved state. That directly affects the persistence behavior we talked about earlier.

There is one branch-specific regression worth flagging: in `goodai/ltm/mem/auto.py`, the default `emb_model` assignment has a trailing comma, so it becomes a 1-tuple instead of a model object. If `AutoTextMemory.create()` is called with no `emb_model`, that path looks broken.

Also, `RealTimeLTMSystem` does not currently expose all of the new `LTMSystem` capabilities. Its allowed RPC methods only include `is_empty`, `clear`, `state_as_text`, `set_state`, `retrieve`, and `retrieve_from_keywords`, so `temporal_retrieval` and `replace_content` are not wired through that realtime wrapper yet.
