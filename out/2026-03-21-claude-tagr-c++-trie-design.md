# tagr-c++ Trie Design

## Purpose

`tagr-c++` is the intake and structural indexing layer of the `tagr` pipeline.
It reads raw input, tokenizes it using the TAGL scanner, and builds a
persistent mmap'd trie that serves as a fast multi-tagspace POS index for
downstream translation.

## Pipeline Overview

`tagr` operates in two passes.

### Pass 1 — Pre-scan (build the index)

```
raw input stream
  → TAGL scanner: tokenize
  → for each token: lookup_pos() against each loaded tagspace
      (tagd-simple-english, Penn Treebank, Universal Dependencies, ...)
  → write into mmap'd trie:
      key   = token string (byte path through trie nodes)
      value = { stream positions[], tagd POS per tagspace[], freq }
```

### Pass 2 — Translation (use the index)

```
for each token in input:
  → trie lookup (fast, no tagdb query)
  → get: all POS roles across all tagspaces + sub-token relationships
  → multiplex: route token to relevant handlers based on tagspace matches
  → emit TAGL
```

## The mmap'd Trie

### Why mmap the trie (not the input)

`project-4` (`~/sandbox/project-4`) mmap'd the *input* corpus for fast
byte-level scanning. `tagr-c++` inverts this: the *trie* is mmap'd, the input
is streamed through a libevent buffer.

Benefits:

* **Persistence**: the trie survives across runs; pre-scanned data accumulates
* **Shared memory**: multiple `tagr` processes share the same trie via
  file-backed mmap — build once, read many
* **Speed**: one mmap offset lookup replaces N sqlite queries across N tagspaces
* **Fixed stride**: flat array of fixed-size node records — cache-friendly,
  no pointer chasing, predictable buffer sizing for stream-oriented I/O

### Concurrency model

* Single writer holds a lock over the file during node insertion
* Multiple concurrent readers require no lock (fixed-size records make
  per-node writes reasonably atomic)
* File-backed mmap doubles as shared memory (`MAP_SHARED`)

## Node and Value Layout

All offsets are into a flat array of `node` records. Index 0 is the root.
No raw pointers — children are stored as `int32_t` offsets; `-1` means no child.

```cpp
// Max positions tracked per token in a single scan pass
static constexpr size_t MAX_TOKEN_POSITIONS = 64;

// Max tagspace matches per token (simple-english, PTB, UD, ...)
static constexpr size_t MAX_TAGSPACE_MATCHES = 8;

struct tagspace_match {
    uint32_t tagspace_id;       // which tagspace this match came from
    tagd::part_of_speech pos;   // tagd POS for this token in that tagspace
    int tok;                    // TAGL scanner token int (TOK_TAG, TOK_RELATOR, ...)
};

struct trie_value {
    size_t freq;
    size_t pos_count;
    size_t pos[MAX_TOKEN_POSITIONS];         // stream byte offsets where token was seen

    size_t match_count;
    tagspace_match matches[MAX_TAGSPACE_MATCHES];
};

struct node {
    int32_t children[256];   // child offsets into flat node array; -1 = no child
    bool term;
    trie_value val;          // meaningful only when term == true
};
```

Fixed-size arrays are intentional. `tagr` is stream-oriented (libevent
non-blocking buffers), so sensibly-sized fixed records map cleanly to
chunk-based buffer processing. Variable-length regions can be added later
when we have empirical evidence that the caps are insufficient.

## Sub-token Relationships

The trie encodes token containment naturally. If `run` and `running` are both
tokens, the path for `run` is a prefix of the path for `running`. Every
intermediate terminal node on the way to a longer token is a sub-token of that
longer token.

This gives pass 2:

* prefix detection (is token A a sub-token of token B?)
* longest-match vs. shortest-match disambiguation
* morphological decomposition and compound word analysis — structural, not
  heuristic

## Multi-Tagspace Multiplexing

A token like `bark` may resolve differently across tagspaces:

* `tagd-simple-english`: `POS_TAG`, `TOK_TAG`
* Universal Dependencies: verb sense
* Penn Treebank: noun or verb depending on context

All matches are stored at the same trie node during pass 1. Pass 2 reads
`trie_value.matches[]` and routes the token to whichever tagspace handlers
apply — one lookup, many possible interpretations, all captured.

## Tagspace Relationship

The mmap'd trie is a fast pre-scan index *over* tagspaces, not a replacement
for tagdb. Planned tagspaces include:

* `tagd-simple-english` (VOA Wordbook — initial target)
* Penn Treebank tagged tagspace
* Universal Dependencies tagspace
* Other dictionaries and taxonomies as needed

Each tagspace contributes a `tagspace_id` and its `tagd::part_of_speech`
classification to matching trie nodes.

## Departure from project-4

`tagr.cc` was derived from `trie-seq.cc` (Ling 473, Project 4 — Human Genome
Sequence Matcher). The key structural differences:

| | project-4 | tagr-c++ |
|---|---|---|
| mmap'd | input corpus | **the trie** |
| node children | `data[4]` (A/T/C/G only) | `children[256]` (any byte) |
| trie populated from | pre-known target sequences | tokens seen in input stream |
| node references | raw `node*` pointers | `int32_t` offsets into flat array |
| trie role | search index over known targets | persistent key/value token store |
| lifetime | process lifetime | persistent across runs |
| sharing | none | multi-process via file-backed mmap |
| `matches()` direction | scan corpus for pre-loaded targets | **not applicable** — flow is inverted |

The `mmap_t` wrapper, `file_exists()`, and the `trie::add()` loop shape are
worth preserving. The genomic `namespace sequence`, `base` struct, and
`matches()` method are conceptually obsolete and should be removed during
the refactor.

## Task Sequence

### Task 1 — Log emitted tokens

Instrument the TAGL scanner so every emitted token (token name + value) is
logged to console. This is the observability foundation for all subsequent
work.

### Task 2 — Reverse frequency table

Over a corpus (e.g. VOA Wordbook text), accumulate `token_type → count` while
scanning, then print sorted by descending frequency. Gives empirical data on
which token types dominate the input, informing tagspace lookup priorities.

### Task 3 — Trie with position and scanner token

Build the mmap'd trie as described above. Each terminal node stores the stream
positions where the token was seen and the TAGL scanner token type resolved via
`lookup_pos()`. This is the core structure that pass 2 will query.

## Design Principles

* Stream-oriented: libevent non-blocking buffers; fixed-size records suit
  chunk-based processing
* Simple first: fixed-size arrays now, variable-length regions later when
  caps prove insufficient
* Small pure functions, deterministic translation, composable pipeline
* Preserve the author's style — minimal diffs, no unnecessary reformatting
* TDD: write test, implement, pass, report
