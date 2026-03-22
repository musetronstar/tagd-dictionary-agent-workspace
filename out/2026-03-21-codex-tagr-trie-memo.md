# Memo: `tagr-c++` Trie Direction

## Summary

`project-4` used `mmap` to map the input files and built the trie in memory as a temporary search structure.

`tagr-c++` should invert that design:

* input should be scanned normally from a stream or buffered reader
* the trie should be the persistent structure
* the trie should be suitable for `mmap`
* the trie should act as a shared token store and future NLP index

This makes the trie part of the system's data model, not just a private implementation detail of one scan.

## Contrast With `project-4`

In `project-4`:

* the trie was a fast matcher for a fixed target set
* the alphabet was small and closed: `{A,T,C,G}`
* the trie was rebuilt for each run
* matches were emitted and then post-processed by shell tools

In `tagr-c++`:

* the trie should persist across scans
* the trie should store token strings and token metadata
* the trie should support later NLP work, not just first-pass tokenization
* the character model and token value space will be much richer than the genome matcher

## Design Direction

The tokenizer should:

1. scan input text into token candidates
2. emit token/value/offset information
3. insert or look up token strings in a trie-backed store
4. attach metadata to those trie entries

The trie should be designed so it can support:

* token identity
* token frequency
* occurrence positions in the input stream
* scanner token kind
* tagd POS lookup results for tag-like tokens
* sub-token and super-token relationships
* future NLP annotations

## Why `mmap` The Trie

Using an `mmap`-friendly trie opens up:

* persistent lexical state
* shared read access across processes
* possible concurrent or multi-process workflows
* less rebuild cost between runs
* a common token index for later analysis passes

This is more valuable for `tagr-c++` than mmapping the input, because the long-lived asset is the token structure, not the raw text buffer.

## Sub-Token Motivation

Natural language tokenization will contain cases where:

* one token is a prefix of another token
* one token occurs inside another token span
* multiple tokenizations overlap on the same text region

A trie helps expose those relationships naturally because shared prefixes are encoded structurally.

That makes the trie useful not only for lookup, but also for:

* discovering nested token forms
* tracking where shorter tokens occur inside longer ones
* supporting later NLP passes that need ambiguity or overlap information

## Implication For `tagr-c++`

`project-4` is still useful inspiration for:

* trie construction
* deterministic scanning
* offset tracking
* pipeline-style post processing

But `tagr-c++` should treat the trie as:

* a persistent/shared lexical index
* a store of token facts
* a foundation for later NLP work

So the main architectural difference is:

* `project-4`: mmap the input, transient trie
* `tagr-c++`: scan the input, mmap the trie
