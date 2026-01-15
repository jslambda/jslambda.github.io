# An Offline Semantic Search for tldr-pages Console Commands

The [TLDR project](https://github.com/tldr-pages/tldr) is one of the most useful tools in a daily terminal workflow. Small, example-focused command explanations help you get real work done — without scrolling through long man pages. But while TLDR is brilliant, it has one major limitation:

> You must already *know the name* of the command before you can retrieve its help page.

<!--If you don’t remember `pgrep` or `jq`, but you *do* remember what you want to achieve… the [official tldr.sh](https://tldr.sh) site can’t help. Its search only works by command name — not by meaning.-->

If you remember what you want to achieve but not the name (pgrep, jq, kill, etc.), the [official tldr.sh](https://tldr.sh) search can’t help — it’s lexical and command-name based.

That limitation motivated building an alternative:
**[tldr semantic search](https://jslambda.github.io/tldr-vsearch/)** — enabling search by intent instead of by exact command names.

---

## Why Semantic Search?

We usually don’t think in exact command names. We think in tasks and goals:

* *"search inside a PDF"*
* *"terminate a running process"*
* *"convert JSON to YAML"*
* *"rename files recursively"*

Lexical text search works only if your query contains the *same words* that appear in the page you’re seeking — which may not be true.

Motivating examples:

Searching for **"convert json to yaml"**.
* On the official TLDR: no results — because that phrase isn’t the command name.
* With lexical text search project ([lexical search](https://jslambda.github.io/tldr-search/)): top results doesn't seem to be relevant. 
* With semantic search ([semantic search](https://jslambda.github.io/tldr-vsearch/)): the top result `yq` is completely relevant.

Searching for **"terminate a running process"**.

* On the official TLDR: no results — because that phrase isn’t the command name.
* With lexical text search project ([tldr-search](https://jslambda.github.io/tldr-search/)): it shows relevant commands — but only at result #3 or #4.
* With semantic search ([tldr-vsearch](https://jslambda.github.io/tldr-vsearch/)):
  the correct matches — variants of the `kill` command — appear *at the top*.

In practice, semantic search improves precision and recall, and tends to surface more relevant candidates earlier.

---

## The Design: Offline-First Principles

Before diving into technical details, here’s the idea I anchored this project on:

> **Developer tools should be offline-first.**
> If `grep`, `make`, `docker`, `ripgrep`, and `cargo` work offline,
> then search tools shouldn’t require an internet connection either.

This led to the following constraints:

✔ **No backend**
✔ **No servers**
✔ **No Pinecone / Elasticsearch**
✔ **Just a static site + client-side JavaScript**

If TLDR works offline, then **semantic TLDR must also run offline.**

---

## Architecture — In-Memory Vector Search

* [`markdown-indexer`](https://github.com/jslambda/markdown-indexer) reads TLDR pages and generates a `data.json` corpus.
* [`python-cli` vector search](https://github.com/jslambda/vector-search/tree/main/python-cli) reads that corpus and generates a **vectorized search index**.

The embedding model is MiniLM-L6-v2, and cosine similarity is used in the search.

You can query it in two ways:

* CLI: [**python vector-search**](https://github.com/jslambda/vector-search/tree/main/python-cli). See the instructions
in [the last section](#try-it-out).
* Browser: **semantic search web UI** —
  👉 [https://jslambda.github.io/tldr-vsearch/](https://jslambda.github.io/tldr-vsearch/)

Search is performed by a **simple linear scan over pre-computed embeddings**.
Even with ~6000 TLDR entries, this is trivial in practice — a query takes about half a second. I only included entries in `linux` and `common` as they seem to cover most of the useful commands. This leads to a relatively small memory footprint enabling a lightweight fully offline experience.

After initial load, the demo runs entirely client-side with no network requests.

---

## Why Offline Local Search Matters

Running semantic search locally leads to valuable consequences:

* Can be embedded directly into DevTools
  (VS Code extensions, Zed, Neovim, browser sidebar, etc.)
* Your queries **never leave the device**
* **No** API keys, cloud failures, outages, or rate limits
* Works even on a plane, on a train, or during a network outage


---

## Looking Forward

This prototype sparked a few next ideas:

* GPU-accelerated cosine indexing (WebGPU)
* HNSW graph indexing for larger offline knowledgebases
  *(Imagine: offline StackOverflow…)*
* Improving performance by using variants techniques like Word2Vec

If there is one takeaway:

> Don’t assume semantic search requires cloud infrastructure.

---

## Try It Out

Demo
👉 [https://jslambda.github.io/tldr-vsearch/](https://jslambda.github.io/tldr-vsearch/)

Toolset
👉 [https://github.com/jslambda/vector-search](https://github.com/jslambda/vector-search)

If you like the idea, adapt it to make your own offline semantic tools.

Below are the full instructions for trying out the full stack on your machine!

```bash
# 1) Clone the TLDR pages repo somewhere temporary (or wherever you prefer)
git clone https://github.com/tldr-pages/tldr.git /tmp/tldr

# 2) Clone and build markdown-indexer (Rust)
git clone https://github.com/jslambda/markdown-indexer.git /tmp/markdown-indexer
cd /tmp/markdown-indexer
cargo build

# Parse + jsonify the TLDR pages (common + linux) into a single JSON file
# Output: /tmp/tldr-data.json
cargo run -- /tmp/tldr/pages/common /tmp/tldr/pages/linux > /tmp/tldr-data.json

# 3) Clone vector-search and use the Python CLI
git clone https://github.com/jslambda/vector-search.git /tmp/vector-search
cd /tmp/vector-search/python-cli

# 4) Create + activate a virtual environment, then install deps
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 5) Vectorize the JSON data
# Output: /tmp/vectorized-data.json
python app.py --verbose --output /tmp/vectorized-data.json /tmp/tldr-data.json

# 6) Query the vectorized data
python app.py --query "copy files recursively" /tmp/vectorized-data.json
python app.py --query "terminate programs" /tmp/vectorized-data.json
```
