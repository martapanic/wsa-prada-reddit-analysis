# Web and Social Media Search and Analysis — Bluesky Project

**Topic:** *The Devil Wears Prada 2*
**Platform:** Bluesky (AT Protocol via `atproto` SDK)
**Course:** Web and Social Media Search and Analysis, A.Y. 2025/2026
**University:** Università degli Studi di Milano‑Bicocca

---

## Authors

<!-- TODO: fill in names -->
- *Author 1*
- *Author 2*

## Research question

> > How did Bluesky users engage with the release of *The Devil Wears Prada 2*, and how do discussion communities, sentiment and dominant themes differ across film-, fashion-, celebrity-culture- and box-office-related narratives?

The question is articulated into four sub‑questions:

1. **Network structure.** What is the shape of the conversation graph and what reply communities emerge?
2. **Sentiment and emotion.** Which affective tones dominate the discussion, and how do they differ across communities?
3. **Temporal dynamics.** How does the conversation evolve across the press‑tour and post‑release window?
4. **Entities and narratives.** Which people, brands and works are most frequently mentioned, and which entities anchor which community?

---

## Repository structure

```
wsa-prada-project/
├── notebooks/
│   ├── 00_pilot_bluesky.ipynb              ← pilot collection (12 queries, ~900 posts)
│   ├── 01_full_collection_bluesky.ipynb    ← full collection (16 queries) + thread expansion
│   ├── 02_network_and_communities.ipynb    ← reply graph, centrality, 4 community algorithms
│   └── 03_content_analysis_and_results.ipynb  ← EDA, sentiment, emotion, NER, cross-analysis
├── data/
│   ├── raw/        ← outputs of notebooks 00 and 01 (immutable, from Bluesky API)
│   ├── processed/  ← outputs of notebooks 02 and 03 (derived analyses)
│   └── figures/    ← all figures produced by the notebooks (.png, dpi=300)
├── credentials.json    ← Bluesky app password (gitignored, see Setup below)
├── requirements.txt    ← Python dependencies
├── .gitignore
└── README.md           ← this file
```

The notebooks are designed to be executed **in order**. Each notebook reads files produced by the previous one and writes its own outputs to disk, so the pipeline is fully reproducible from the raw collection onwards.

---

## Setup

### 1. Python environment

Tested on Python 3.12. From the project root:

```bash
python3 -m venv venv
source venv/bin/activate          # macOS / Linux
# .\venv\Scripts\activate         # Windows PowerShell
pip install --upgrade pip
pip install -r requirements.txt
```

### 2. Bluesky credentials

Notebooks 00 and 01 connect to the Bluesky API and require an [app password](https://bsky.app/settings/app-passwords) (not the account password). Create one and save it as `credentials.json` in the project root:

```json
{
  "handle": "your-handle.bsky.social",
  "app_password": "xxxx-xxxx-xxxx-xxxx"
}
```

The file is listed in `.gitignore` and is never tracked.

### 3. NLTK data and SpaCy model

Notebook 03 needs additional resources (NLTK corpora and the SpaCy English model). They are downloaded automatically by the setup cell on the first run; no manual action is required. If your macOS Python ships without a configured CA bundle, the setup cell already includes a one‑session SSL workaround.

For a fully clean installation, you can pre-download manually:

```bash
python -m spacy download en_core_web_sm
python -c "import nltk; [nltk.download(p) for p in ['stopwords','wordnet','omw-1.4','punkt','punkt_tab']]"
```

---

## How to reproduce the analysis

Open the notebooks in the order below and run all cells. Estimated runtimes refer to a recent laptop with a normal home internet connection.

| # | Notebook | Reads from | Writes to | Approx. runtime |
|---|---|---|---|---|
| 00 | `00_pilot_bluesky.ipynb` | Bluesky API | `data/raw/bluesky_posts_pilot.*` | 5–10 min |
| 01 | `01_full_collection_bluesky.ipynb` | Bluesky API | `data/raw/bluesky_posts_full.*`, `data/raw/bluesky_replies_full.*` | 25–35 min |
| 02 | `02_network_and_communities.ipynb` | `data/raw/` | `data/processed/` (graph, centrality, communities) + `data/figures/02_*.png` | 1–2 min |
| 03 | `03_content_analysis_and_results.ipynb` | `data/raw/` + `data/processed/` | `data/processed/` (sentiment, emotion, NER) + `data/figures/03_*.png` | 3–5 min |

> **Time window for the analysis:** 1 April – 21 May 2026. The collection was performed on 21 May 2026 and the window is closed at that date even when the notebooks are re‑run later, so that the dataset reported in the project remains consistent.

---

## Methodology

### Data collection

The pilot (notebook 00) tests twelve search queries that target four overlapping audiences: the film itself, the two main characters (Miranda Priestly and Anna Wintour), the four returning cast members (Anne Hathaway, Meryl Streep, Emily Blunt, Stanley Tucci) and two press‑tour guests (Lady Gaga, Doechii). Cast queries are disambiguated by pairing each name with `prada` to suppress generic‑name noise.

The full collection (notebook 01) refines the query set in light of the pilot: queries that saturated the pilot cap are widened, low‑signal queries are dropped, and six new queries are added to broaden recall (`"dwp2"`, `#dwp2`, `"devil wears prada sequel"`, `"prada sequel"`, `"prada 2" movie`, `"devil wears prada" 2026`). It also performs **thread expansion**: for every root post with at least one declared reply, the full thread is fetched with `app.bsky.feed.get_post_thread(depth=8)` and recursively flattened, producing one row per reply with parent and root URIs.

Collection uses cursor‑based pagination with `sort="latest"`, server‑side `since` / `until` time filters, and an exponential‑backoff retry wrapper that honours `ratelimit-reset` and `retry-after` headers so that long runs recover from transient errors automatically.

### Network analysis (notebook 02)

The reply network is built on author handles: an edge connects the author of a reply to the author of the post being replied to. Self‑loops (chained self‑replies, which on Bluesky usually correspond to long thread‑style posts split across consecutive bubbles by the same author) are removed because the unit of analysis is the interaction *between* users; the affected authors are flagged with `is_likely_promo` for later interpretation but kept in the graph.

Two graphs are constructed from the same edge list: a **directed** graph used for in‑degree and out‑degree centrality, and an **undirected weighted** graph used for closeness, betweenness, eigenvector centrality and community detection.

Four community detection algorithms representing distinct families are compared on the giant component: **Louvain** (modularity‑maximising, baseline), **Greedy Modularity** (agglomerative), **Asynchronous Fluid Communities** (propagation‑based, requires `k`), and **Girvan–Newman** (divisive, applied only to the giant component for computational cost). Their partitions are compared with modularity, Adjusted Rand Index and Normalised Mutual Information. The **Louvain** partition is taken as the reference for the downstream content analysis.

### Content analysis (notebook 03)

Posts and replies are combined into a single corpus and standardised. Sentiment, emotion and NER are run on the **English sub‑corpus only**, because the lexicons and models used (VADER, AFINN, NRCLex, SpaCy `en_core_web_sm`) are English‑only and would silently score other languages as neutral.

Preprocessing produces lower-cased text without URLs, mentions or punctuation. Tokens are obtained from the cleaned text, English stopwords and project-specific collection keywords are removed, and words are lemmatised with WordNet.

**Sentiment** uses VADER (`compound` score, ±0.05 thresholds for the discrete label) cross‑checked with AFINN. **Emotion** uses the eight basic emotions of NRCLex (Plutchik wheel). **NER** uses SpaCy `en_core_web_sm` on the *original* text (not the cleaned version, because aggressive cleaning destroys the case and punctuation cues SpaCy needs) and keeps the labels `PERSON`, `ORG`, `GPE`, `LOC`, `WORK_OF_ART`, `EVENT`, `DATE`.

The final cross‑analysis joins community membership with sentiment, emotion and entities, producing a per‑community profile table (`community_profiles.csv`) that anchors the answers to the four sub‑questions.

---

## Key findings

- The Bluesky conversation about *The Devil Wears Prada 2* is **event‑driven and post‑release**. Daily volume is negligible before 15 April 2026 and rises sharply after 1 May; about 88 % of the corpus is post‑release.
- The reply network is **highly fragmented**: 1 555 authors form 371 disconnected components, the largest of which contains only 5.7 % of nodes. The conversation is a constellation of small reply trees around individual viral posts rather than a single sustained debate.
- Three of the four community detection algorithms (Louvain, Greedy Modularity, Girvan–Newman) converge on essentially the same 10–11 community partition with modularity around 0.42 (pairwise ARI ≥ 0.96). FluidC produces a different, coarser partition (ARI ≈ 0.45).The convergence between Louvain, Greedy Modularity and Girvan–Newman suggests that the giant component has a stable modular structure under different community-detection strategies.
- Sentiment is **polarised rather than uniformly positive**: about 43 % negative, 38 % positive, 19 % neutral (VADER ± 0.05). The NRCLex emotion profile is led by *anticipation* and *trust*, consistent with a release‑window discussion.
- Communities show distinct affective and topical profiles, which is what makes the cross‑analysis informative. Within the giant component, communities show distinct affective and topical profiles. These community-level findings concern only about 110 texts, corresponding to roughly 2% of the full corpus and about 2.6% of the English sub-corpus.

The full quantitative breakdown is in the final markdown cells of notebooks 02 and 03, in the figures under `data/figures/`, and in the project report.

---

## Files produced by the pipeline

`data/raw/` (from notebooks 00 and 01):

- `bluesky_posts_pilot.csv` / `.jsonl`, `bluesky_query_summary_pilot.csv`
- `bluesky_posts_full.csv` / `.jsonl`, `bluesky_replies_full.csv` / `.jsonl`, `bluesky_query_summary_full.csv`

`data/processed/` (from notebooks 02 and 03):

- Network: `reply_edges.csv`, `node_attributes.csv`, `reply_graph_directed.gexf`, `reply_graph_undirected.gexf`, `centrality_full.csv`, `centrality_giant.csv`, `community_assignments.csv`, `community_comparison.csv`, `community_ari.csv`, `community_nmi.csv`
- Content: `text_analysis_dataset.csv`, `sentiment_results.csv`, `emotion_by_community.csv`, `ner_entities.csv`, `community_profiles.csv`

`data/figures/` (PNG, dpi 300):

- `02_*` — network plots, component sizes, centrality bar chart, algorithm comparison, giant‑component community visualisation
- `03_*` — texts over time, query productivity, top words, word cloud, sentiment distribution / over time / by community, emotion heatmap, top entities overall, positive vs negative entities

---

## Limitations

- **Window.** The analysis covers 1 April – 21 May 2026. Discussion after 21 May is not included.
- **Language.** Sentiment, emotion and NER are computed on the English‑tagged sub‑corpus only (about 70 % of the texts). Portuguese, Spanish and French posts are described in aggregate but not analysed with their own pipelines.
- **Community coverage.** Louvain communities are defined only on the 88 authors of the giant component. As a result, community-level content analysis covers about 110 English texts, while the majority of the corpus is analysed only at the global level.
- **NER model.** `en_core_web_sm` is the small SpaCy model: it sometimes mislabels hashtags or short tokens (e.g. `ai`, `tucci`, `devilwearsprada2`) as locations. The errors do not affect the dominant ranks but should be acknowledged when reading individual entries of the entity tables.
- **Promotional accounts.** A small number of accounts (e.g. `@ourmovieguide`) repost template content. They are flagged with `is_likely_promo` and kept in the network for transparency rather than silently filtered out.

---

## Licence and data ethics

The data collected through the Bluesky API is publicly accessible content authored by individual users. It is used here only for academic analysis in the context of this course and is not redistributed in identifiable form outside of this repository. Author handles appear in the analysis where they are structurally relevant (e.g. hub authors in the reply graph) and are taken as published.

---
