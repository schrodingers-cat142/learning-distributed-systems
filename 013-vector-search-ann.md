# Vector Search: Graph Indexes and Billion-Scale Inverted Indexes

**Date:** 2026-07-26
**Sources:**
- [Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs — Malkov & Yashunin (arXiv 2016, IEEE TPAMI 2020)](https://arxiv.org/pdf/1603.09320)
- [Revisiting the Inverted Indices for Billion-Scale Approximate Nearest Neighbors — Baranchuk, Babenko & Malkov (ECCV 2018)](https://arxiv.org/pdf/1802.02422)

**Related entries:**
- [003-data-models-what-goes-around.md](003-data-models-what-goes-around.md) — that entry called vector databases "a feature, not a data model"; this entry is the internals of that feature
- [004-data-store-selection-framework.md](004-data-store-selection-framework.md) — vector search is the retrieval half of "full-text search / similarity search" in the store-selection framework
- [008-lsm-trees.md](008-lsm-trees.md) — Bloom filters there are a probabilistic structure trading accuracy for space; ANN indexes make the same kind of bargain, trading recall for speed
- [009-bigtable.md](009-bigtable.md) — Bigtable's three-level tablet location hierarchy is the same "coarse-to-fine zoom-in" idea that both HNSW's layers and the inverted index's centroids use

---

## Why This Topic Exists

For decades, "search" meant the **inverted index**: a dictionary that maps each word to the list of documents containing it. If you search for "car," the engine looks up the posting list for "car" and returns those documents. This is the machinery behind full-text search, and it is superb at what it does — but it only finds *lexical* matches. A search for "car" will never surface a document that only ever says "automobile," because as far as the index is concerned those are two unrelated strings.

Vector search is the answer to that limitation. Instead of indexing words, we pass each document through an embedding model that turns it into a point in a high-dimensional space — a vector of, say, 96 to 1536 numbers. These models are trained so that *things which mean similar things land near each other*. "Car" and "automobile" end up as neighbors; a photo of a golden retriever lands near a photo of a labrador. Once your data lives in this geometric space, retrieval stops being about matching strings and becomes a purely geometric question: **given the query's vector, which stored vectors are closest to it?** This is the k-Nearest-Neighbor (k-NN) problem, and it is the foundation of semantic search, recommendations, image retrieval, and the retrieval step of nearly every modern LLM application.

The trouble is that answering this question *exactly* is brutally expensive. To find the true nearest neighbors you would have to compute the distance from your query to every single stored vector — an O(N) scan for each query, which is hopeless at billions of vectors. Worse, the old geometric tricks that make low-dimensional search fast (like kd-trees, which recursively cut space in half) fall apart in high dimensions. This is the famous **curse of dimensionality**: when you have hundreds of dimensions, everything is roughly equidistant from everything else, and space-partitioning trees end up visiting almost the whole dataset anyway.

The field's response was to give up on exactness. We accept **Approximate Nearest Neighbor (ANN)** search: return vectors that are *almost certainly* the nearest ones, and in exchange go orders of magnitude faster. The quality of an approximate index is measured by **recall** — if the true answer had 10 nearest neighbors and your index returned 9 of them, that's 90% recall. The entire discipline is about pushing the frontier of a three-way tradeoff:

```
                    RECALL
                   (accuracy)
                       ▲
                       │
                       │     Every ANN index picks a point
                       │     in this triangle. You cannot
                       │     maximize all three at once.
                       │
          ◄────────────┼────────────►
       LATENCY                    MEMORY
      (speed)                  (footprint)
```

This is the same flavor of tradeoff as the RUM conjecture from the LSM-tree notes (Entry #8) — you don't get read speed, write speed, and space efficiency for free; you choose. The two papers in this note attack this tradeoff from **two completely different corners of that triangle**, and — this is the satisfying part — the second one uses the first one as a building block.

---

## The Two Papers at a Glance

It helps to know up front how these two fit together, because they are not competitors. They solve the same underlying problem (find nearest vectors fast) but under **opposite constraints**.

The first paper, **HNSW**, assumes your vectors fit comfortably in RAM in their full, uncompressed form. Under that assumption it builds the fastest, highest-recall index the field has — but it is memory-hungry. The second paper, **Revisiting Inverted Indices**, assumes the opposite: you have a *billion* vectors that cannot possibly fit in memory uncompressed, and RAM is your binding constraint. It compresses everything aggressively and organizes the data into buckets. And crucially, to make its bucketing scheme work, it reaches for HNSW as an internal helper.

```
   IN-MEMORY, UNCOMPRESSED                 BILLION-SCALE, COMPRESSED
   (raw vectors fit in RAM)                (RAM is the bottleneck)
   ─────────────────────────               ──────────────────────────
   Paper 1: HNSW                           Paper 2: IVFOADC + Grouping + Pruning
   Structure: a proximity graph            Structure: inverted lists + compression
   Strength: best recall/latency           Strength: a billion vectors in modest RAM
   Weakness: high memory                   Weakness: some recall lost to compression
                    │                                        ▲
                    │                                        │
                    └───────── HNSW is used INSIDE ──────────┘
                               paper 2 as the "router" that
                               sends each query to the right bucket
```

Both papers, incidentally, share an author: **Yury Malkov**, who created HNSW and then co-authored the billion-scale paper that puts HNSW to work. That shared lineage is why the handshake between them is so clean.

---

## Paper 1: HNSW — Searching by Walking a Graph

### The Core Intuition

Imagine every vector in your dataset is a city, and you draw roads connecting each city to its nearby neighbors. Now someone drops you at a random city and asks you to reach the city closest to some target location. A simple strategy works remarkably well: **look at all the cities your current city connects to, drive to whichever one is closest to the target, and repeat.** Keep greedily moving toward the target until none of your neighbors is any closer than where you already are. You've arrived (approximately) at the nearest city.

This is **greedy routing on a proximity graph**, and it is the beating heart of HNSW. The graph *is* the index. Searching is just walking the graph, always stepping toward the query.

The problem is that a naive "connect each city to its nearest neighbors" graph has two failure modes. First, the number of hops you need grows badly as the dataset grows — you spend too long walking. Second, and more dangerously, if your data forms distinct clusters (which real data always does), the graph can fracture into islands with no roads between them. Greedy search gets *stuck* at the edge of an island, unable to cross to the cluster where the real answer lives. HNSW introduces two mechanisms to fix exactly these two problems.

### Mechanism One: A Hierarchy of Layers (the Skip-List Idea)

The best way to understand HNSW's layered structure is through the **skip list**, a classic data structure. A skip list is a sorted linked list with "express lanes" stacked on top. The bottom lane contains every element; each higher lane contains a sparse random sample of the one below. To find something, you ride the top express lane taking huge jumps until you overshoot, drop down a level for finer jumps, and keep dropping until you're on the bottom lane doing precise, single-step search. The express lanes let you *approach* the target region in a few big leaps before doing careful work.

HNSW is, quite literally, the skip list generalized from a 1-dimensional sorted list to a multi-dimensional proximity graph. Instead of express *lanes*, it has express *graphs*.

```
                        ENTRY POINT
                             │
   Layer 2   (very sparse)   ●───────────────●            ← a few long-range hops
                             │               │
                             ▼               │
   Layer 1   (sparse)    ●───●────●──────●───●            ← medium hops, zoom in closer
                         │   │    │      │   │
                         ▼   ▼    ▼      ▼   ▼
   Layer 0   (ALL nodes) ●─●─●─●─●─●─●─●─●─●─●─●─●─●─●     ← every vector lives here;
                                   ▲                         the precise search happens here
                                   │
                              query lands here
```

Every vector lives on **layer 0**, the ground floor. A random few also appear on layer 1, a rarer few on layer 2, and so on — each layer up holds an exponentially smaller sample, decided at insertion time by drawing from an exponentially-decaying distribution (the formula is `l = ⌊−ln(random)·mL⌋`, where the parameter `mL` sets how quickly the layers thin out). The upper layers, being sparse, have long-range connections that let a search cover huge distances quickly.

A search always **starts at the single entry point on the top layer** and greedily walks toward the query on that sparse layer until it can get no closer. Then it drops to the next layer down, using where it landed as the new starting point, and repeats. Each descent zooms in on a smaller neighborhood. Only when it reaches layer 0 does it do the wide, careful search. Because the upper layers collapse the distance so efficiently, the whole search runs in **O(log N)** time. HNSW's proud claim is that it needs *no separate index* to find a good starting point — the hierarchy is its own built-in zoom-in mechanism.

### Mechanism Two: The Neighbor-Selection Heuristic (the Subtle, Clever Part)

This is the piece that makes HNSW *robust* rather than merely fast, and it's worth slowing down for.

When you insert a new vector, you have to decide which existing vectors it should connect to. The obvious answer is "connect it to its M nearest neighbors." But consider what happens on clustered data. Suppose your new point sits at the edge of Cluster A, which is near Cluster B. Its M nearest neighbors are *all* going to be other points in Cluster A — because that's where its closest points are. So you build a dense web of connections inside Cluster A and **never create a single road to Cluster B.** Do this for every point and the two clusters become disconnected islands. Greedy search starting in A can never reach B.

```
        NAIVE "connect to M nearest"              HNSW's DIVERSIFYING heuristic
        ────────────────────────────             ─────────────────────────────
        Cluster A        Cluster B                Cluster A        Cluster B
        ● ● ●            ● ● ●                     ● ● ●───────────● ● ●
        ●[N]●            ● ● ●                     ●[N]●           ● ● ●
        ● ● ●            ● ● ●                     ● ● ●           ● ● ●
                                                        ▲
        [N]'s links all stay inside A.           [N] keeps one link that BRIDGES to B,
        The clusters are islands.                even though B's point isn't among its
        Greedy search gets trapped.              very closest — connectivity preserved.
```

HNSW's heuristic prevents this. When choosing neighbors, it considers candidates from nearest to farthest, but it **only accepts a candidate if that candidate is closer to the new point than it is to any already-accepted neighbor.** In plain terms: it refuses to add a new connection that points in a direction you've already covered. This forces connections to spread out in *diverse directions*, and it deliberately keeps a few longer "bridge" edges that link different regions together. Mathematically this approximates something called the Relative Neighborhood Graph — a minimal skeleton that stays connected. This one heuristic is why HNSW beat its own predecessor (the flat "NSW" graph) by up to **a thousandfold** on the hardest clustered, low-dimensional datasets.

### The Knobs You Actually Turn

HNSW exposes a handful of parameters, but in practice you think about very few of them:

| Parameter | What it controls | The tradeoff |
|---|---|---|
| **M** | how many neighbors each node links to (layer 0 gets `2M`) | higher M → better recall, more memory, slower build. Typical range **6–48**. |
| **efConstruction** | how hard the index looks for good neighbors *while building* | higher → better index quality, slower build (diminishing returns) |
| **efSearch** | how wide the search is *at query time* | **the main live knob** — higher → better recall, slower queries. Doesn't touch memory. |
| **mL** | how fast the layers thin out | default `1/ln(M)`; you almost never change it |

The practical mental model: **`M` is the one build-time decision that matters, and `efSearch` is the dial you turn at query time to trade recall against speed.** You can raise recall on a live system just by increasing `efSearch`, no rebuild required.

### The Cost, and the Catch

Search is O(log N); building the index is O(N log N) and can be done incrementally as data arrives. The results in the paper were dominant — HNSW beat tree methods (Annoy, FLANN), hashing methods (LSH/FALCONN), and even Facebook's product-quantization approach, usually by a wide margin.

But there is a real catch, and it's the reason the second paper exists: **memory.** HNSW stores explicit graph links — roughly 60 to 450 bytes per vector *on top of* the raw vectors themselves, which it keeps uncompressed. For a few million or even tens of millions of vectors, that's fine. For a *billion* high-dimensional vectors, storing the raw data plus a graph would run into terabytes of RAM. HNSW simply doesn't fit in that world. (The original paper also had no support for deleting vectors and was awkward to distribute across machines, since every search funnels through that single top-layer entry point — though production implementations have since patched the deletion gap with tombstones.)

---

## Paper 2: Revisiting Inverted Indices — A Billion Vectors on a RAM Budget

### A Different World With Different Rules

Now change the constraints entirely. You have **one billion vectors**, and they will not fit in memory uncompressed. RAM — and the cloud bill that comes with it — is the thing you are fighting. In this world, HNSW's "store everything plus a graph" approach is a non-starter. You need two things HNSW doesn't offer: you need to **compress** the vectors so they fit, and you need a way to **avoid looking at most of them** on every query. This paper's lineage is the classic inverted index, reborn for vectors.

### Building Block One: The Inverted File (IVF) — Bucketing by Centroid

The inverted index idea maps beautifully onto vectors. Instead of grouping documents by the words they contain, we group vectors by *which region of space they fall in*. Run k-means clustering to carve the space into K regions, each with a representative center called a **centroid**. Assign every vector to its nearest centroid. Now you have K "posting lists" — one per centroid — exactly like a full-text inverted index, except the key is a centroid instead of a word.

```
   The space, carved into regions by K centroids (× marks a centroid):

        ┌─────────┬─────────┬─────────┐
        │    ×    │    ×    │    ×    │   Every vector belongs to the
        │  • • •  │  • •    │   • • • │   posting list of its nearest ×.
        ├─────────┼─────────┼─────────┤
        │    ×    │   query │    ×    │   A query only needs to scan the
        │  •  •   │ ★  ×    │  • • •  │   posting lists of the few centroids
        ├─────────┼─────────┼─────────┤   nearest to it (the shaded ones),
        │    ×    │    ×    │    ×    │   not all K of them.
        │  • •    │ • • •   │   •  •  │
        └─────────┴─────────┴─────────┘
```

At query time you don't scan a billion vectors. You find the handful of centroids nearest the query and scan only *their* posting lists. This is the "probe a few buckets" strategy, and it is what makes the whole thing fast.

### Building Block Two: Product Quantization (PQ) — Making Vectors Tiny

Bucketing solves the *speed* problem but not the *memory* problem — a billion vectors is still a billion vectors. **Product Quantization** solves memory by compression. It chops each vector into several sub-vectors and replaces each sub-vector with the ID of the closest entry in a small learned codebook. The upshot is that a vector that was, say, 96 floating-point numbers (384 bytes) gets crushed down to an **8- or 16-byte code**. That is the compression that lets a billion vectors live in a modest amount of RAM. You lose some precision — the stored vector is now approximate — but you can re-rank the top candidates more carefully at the end to recover most of the lost accuracy.

### The Paper's Actual Argument

The prior state of the art was the **inverted multi-index (IMI)**, which gets a very fine-grained partition by splitting the vector into two halves, building a separate codebook for each half, and taking every combination — giving K² tiny cells from two K-sized codebooks. This is clever and works beautifully *when the two halves of the vector are statistically independent* — which happens to be true for old hand-crafted image features like SIFT.

But here's the crux: **modern deep-learning embeddings are not independent across their dimensions** — they're entangled, because a neural network learned them jointly. When IMI's independence assumption breaks, two ugly things happen. Many of the K² cells turn out **empty**, so queries waste time examining regions that contain nothing. And the quality **stops improving** once the codebook passes a certain size (around 2¹⁴ cells) — you can't buy more accuracy by making it finer.

The paper's thesis is a nice piece of "the old simple thing was underrated": the plain single inverted index (IVF) with a *very large* codebook actually beats the fancy multi-index at equal memory and build cost — **if** you can solve the two problems that a huge codebook creates. Their full system, named **IVFOADC + Grouping + Pruning**, does exactly that with three ideas:

**First, the huge codebook — and this is where HNSW enters.** To get a fine-grained partition, they want an enormous number of centroids: up to about 4 million (K = 2²²). But assigning a query to its nearest centroid out of 4 million is itself an O(K) nearest-neighbor problem — the very thing we're trying to avoid! Their solution is to **build an HNSW graph over the centroids themselves** and use it to route each query to its nearest centroid in well under a millisecond. HNSW here is not the main index over the billion data points; it is a fast *router* over the few million centroids. This is the handshake between the two papers:

```
                    ┌──────────────────────────────────────────┐
   query ★ ─────►   │  HNSW graph over ~4 million CENTROIDS     │
                    │  (Paper 1, used as a fast router)         │
                    └───────────────────┬──────────────────────┘
                                        │ "your nearest centroids
                                        │  are #418, #9022, #37"
                                        ▼
                    ┌──────────────────────────────────────────┐
                    │  Scan only those posting lists, which     │
                    │  hold PQ-COMPRESSED billion-scale vectors  │
                    │  (Paper 2's inverted index)               │
                    └──────────────────────────────────────────┘
```

A previous attempt used a kd-tree for this routing and it failed — the centroid search wasn't accurate enough. HNSW's near-perfect, sub-millisecond centroid lookup is what makes million-centroid inverted indexes practical at all. (Note the flip side: HNSW gives *no* benefit when the codebook is small — with only a few thousand centroids, approximate routing is no faster than just checking them all. HNSW only pays off precisely because this method leans on huge codebooks.)

**Second, Grouping.** Each posting list is further split into sub-regions around learned sub-centroids. Because each vector is now measured relative to a nearby sub-centroid rather than a distant main centroid, the leftover "residual" it has to compress is smaller — and smaller residuals compress more accurately with PQ. Better compression accuracy means better recall at the same byte budget.

**Third, Pruning.** Having computed distances to all the sub-regions, the search simply **skips the far half of them** — the paper found you can ignore roughly 50% of sub-regions with no measurable loss in accuracy, because they're too far to contain the answer anyway.

### The Results

The payoff is concrete. On DEEP1B (a billion deep-learning descriptors), the system matched the best competing methods' recall while running **several times faster** at the same memory footprint — for example, reaching R@1 of 0.452 in 3.3 milliseconds where a rival method (GNOIMI) needed 20 milliseconds for essentially the same recall. Memory scales as O(K) rather than IMI's O(K²), which matters at these sizes.

The paper is also honest about where it loses: on **SIFT1B** (the old hand-crafted features), IMI still wins for short candidate lists — precisely because SIFT's sub-vectors *are* independent, so IMI's core assumption holds there. The lesson is exactly the one you'd expect: the multi-index shines on independent features, the big-codebook single index shines on entangled deep embeddings, which is what everyone actually uses now.

---

## Putting It Together: When to Use Which

The two papers map cleanly onto the two regimes of real-world vector search, and the deciding question is almost always **"does my data fit in RAM uncompressed?"**

```
   How many vectors, and how much RAM?
                │
        ┌───────┴────────┐
        │                │
  Fits in RAM       Too big / RAM-bound
  uncompressed      (100s of millions → billions)
        │                │
        ▼                ▼
     HNSW            IVF + PQ/OPQ
  (Paper 1)          (Paper 2, with HNSW as its router)
  best recall/       compact footprint, billion-scale,
  latency, but       trades some recall for fitting in memory
  memory-hungry
```

- **Reach for HNSW** when your dataset fits in memory uncompressed — roughly up to tens of millions of vectors on a single machine, depending on dimension — and you want the best possible recall-versus-latency curve and can afford the memory. This is the default index in essentially every vector database today.

- **Reach for the inverted-index + product-quantization family** when you have hundreds of millions to billions of vectors that won't fit uncompressed, and memory (i.e. cost) is your binding constraint. You accept a little recall loss from compression in exchange for fitting a billion vectors into commodity RAM.

- **In practice, production systems rarely use vectors alone.** The strong default is **hybrid search**: run a classic lexical inverted index (BM25 keyword search) *and* a vector ANN search, then fuse the two ranked lists. Vectors capture meaning and paraphrase; lexical search catches exact terms, names, product codes, and rare words that embeddings blur together. The two are complementary, which brings this whole topic full circle back to the humble inverted index we started with.

---

## What's in the Industry Today

HNSW has become the **default in-memory ANN index almost everywhere** — it's in FAISS, Qdrant, Weaviate, Milvus, pgvector, Elasticsearch and OpenSearch, Redis, Vespa, and Lucene. Production implementations have since added the deletions and updates the original 2016 paper lacked, typically via tombstones and re-linking.

The **inverted-index-plus-quantization family remains the workhorse for billion-scale** deployments, and the specific idea from the second paper — a large codebook with an HNSW graph as the coarse quantizer — is now standard practice in FAISS's large-index recipes. Alongside it, **DiskANN** (Microsoft's graph index that lives mostly on SSD) attacked the same billion-scale problem from the graph side rather than the compression side, and quantization itself has moved on: **scalar (int8) and binary quantization** (one bit per dimension, ~32× smaller, with a full-precision re-rank at the end) are now common layers stacked on top of HNSW.

The demand driving all of this is **retrieval-augmented generation (RAG)**: nearly every LLM application follows the pattern *embed the query → ANN-retrieve relevant chunks → feed them to the model*. That single pattern is why every database vendor bolted on vector search between 2023 and 2025, and why the ideas in these two papers — one from 2016, one from 2018 — are suddenly infrastructure that a huge fraction of the software industry depends on.

---

## Key Takeaways

- **Vector search turns "find similar meaning" into "find nearest point,"** letting semantic queries work where lexical keyword matching fails.
- **Exact nearest-neighbor is too slow and the curse of dimensionality kills trees,** so everyone uses *approximate* nearest-neighbor and measures quality by recall.
- **HNSW searches by greedily walking a proximity graph,** using a skip-list-style hierarchy of layers to zoom in fast (O(log N)) and a diversifying neighbor-selection heuristic to stay connected across clusters. Its weakness is memory.
- **The inverted-index paper handles a billion vectors on a RAM budget** by bucketing vectors under k-means centroids and compressing them with product quantization — and it uses HNSW internally as a fast router over its millions of centroids.
- **The two are layers of one stack, not rivals:** HNSW dominates in-memory; IVF+PQ (routed by HNSW) dominates at billion-scale.
- **Real systems combine vector search with classic lexical inverted indexes (hybrid search),** because meaning and exact-term matching catch different things.
