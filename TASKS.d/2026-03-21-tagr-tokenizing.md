# tagr-c++ Scanner Tokenizer Tasks

Develop `tagr/tagr-c++` as a *tokenizer*:
+ use `tagd` logging facilities
+ use `tagd:error` reporting facilities
+ use a TAGL inspired/derived scanner
+ lookup tagd POS for tag-like patterns

## Task 1

Log <matched tokens> and <token type> string as a TSV line.

Use C++ to use logging facility to output `<TOKEN_VALUE>\t<TOKEN_STR>` for each token match

where `<TOKEN_VALUE>` is the literally bytes matched and `<TOKEN_STR>` is the string representation of that token type.

### Example (no -f <tagl file>, so no lookup facility):
```bash
$ echo "one any of r@nd0m" | ./tagr -n
one TOK_UNKNOWN
any TOK_UNKNOWN
of  TOK_UNKNOWN
r@nd0m  TOK_UNKNOWN
```

### Example (using -f <tagl file> for lookup facility):
```bash
$ echo "one any of" | ./tagr -n -f tests/bootstrap.tagl
one TOK_TAG
any TOK_TAG
of  TOK_SUB_RELATOR
r@nd0m  TOK_UNKNOWN
```

## Task 2 Remove frequency table from C++

User completed these changes (see `git log` to verify understanding):
* Removed any code, comments, fragments or otherwise leftover from the former desire
to print token frequency in C++. Make it as if that idea never existed.
* Keep `freqs` in the `trie_value` for now - see TODO comment - it will be replaced later 

### Small refactor according to TODO comments

* Do not change any code.
* Read all TODO comments in `tagr.h`, `tagr.cc`
* Verify to the user the agent understands our goals.
* Only upon validation from user - proceed

## Task 3 Tokens Reverse Frequency Table in Bash

Print tagr token reverse frequency table of tagr output tokens for an input stream

I created bash script `bin/tagr-rev-freq-tokens.sh`.
+ Follow usage example from `TODO` comment 
+ Modify and fix it to read from STDIN

### Test Reverse Frequency Table 

1. Add a `test_rev_freqs` target to `tests/Makefile`
   which will call a shell script `test-rev-freqs.sh`
   exit code 0 indicates success - don't hide output
2. Create test input: `simple-text.txt` to contain simple declaritive sentences according to examples we have seen in our conversations.
3. Create test output: `output-simple-text-rev-freqs.txt` to contain the exact correct output of a reverse frequency table for `simple-text.txt`
3. Create test script: `test-rev-freqs.sh` to
   + Run: `bin/tagr-rev-freq-tokens.sh`
     STDIN: `simple-text.txt`
     STDOUT: <redirect STDOUT to /tmp file - use robust method - follow the way `~bin/vmd` uses `mktemp` as inspiration>
   + diff the temp file with `output-simple-text-rev-freqs.txt`
   + If has difference, let diff output be displayed
     exit 1
   + No difference - no output - success - exit 0


## Task 4 - Print Trie (duplicate in input)

* construct trie of tokens and their
  + position within the input stream
  + rename `print_targets()` to `print_trie()`
    modify to do what its's comment says: 
	+  TODO prints to the trie to the output stream (defalut STDOUT).
	   the output bytes should match exactly the normalized input stream bytes (TODO: normalize input bytes)

## Task 5 (tagr architecture report)

Analysis & Generate Feasibility Report to accomplish:
* remove tokenization being done in the `tagr_tokenizer:scan()` method
* create `tagr-scanner.re.cc`
* do the tokenization using re2c
* populate a tag with matched tokens according to type and use the TAGL callback (the way TAGL does it)
* When in down follow `tagd/tagl` or `tagd/tagsh` for inspiration.

## Design Principles

+ Interative & TDD - Think small
  1. Write test according to user requirements
  2. Implement code according to user desired change of behaviour or constrainsts
     Write deep code according to specification, not shallow code just to pass a test
  3. Pass test
  4. Report back to user
  5. Provide a concise git commit message for the completed task/iteration
  6. Ask to proceed with next iteration (unless already instructino to proceed).
+ small pure functions
+ deterministic translation
+ composable pipeline
+ clear error behavior

## Scope

Files/Dirs which can be changed:
`tagd-workspace/out`
`tagr/tagr-c++/`

## Doctrine

Follow `docs/ai-assisted-dev-doctrine.md`

## Constraints

* preserve behavior
* keep diff small
* no new dependencies
* follow priority
  1. user prompts
  2. user specified task `.md` file
  3. `README.md` and `AGENTS.md`
  4. files references encountered when processing
     the files above should be folowed as needed

## Language & Style

* follow `STYLE.md`
* speak TAGL `TAGL-README.md`

## Tests

* update tests as needed
* modified code must be tested
* tests must pass before task is complete
* `make all` in `tagr-c++/` must run the full required suite
* follow the `tagd/` testing style:
  + model unit tests on `tagd/tagl/tests/Tester.h`
  + use CXXTest suites in a dedicated `tests/` directory for unit tests
    and in-process integration tests
  + prefer small in-process tests over shell `diff` tests in `Makefile`
  + use mocks/stubs or small in-memory fixtures where practical
  + keep tests fast, deterministic, and easy to extend iteratively
  + prefer repo-owned local TAGL fixtures under `tagr-c++/tests/`
  + prefer in-memory tagspace loading from `.tagl` fixtures over building on-disk `.db` files
* if external CLI behavior or process I/O needs verification, model system tests
  on `tagd/tagsh/tests/tester.exp`
  + use `expect` style system tests in `tests/*.exp`
  + use them the way `tagsh` tests `bin/tagsh` externally
  + keep them focused on CLI/process behavior, not unit logic
  + verify the built `../tagr` binary rather than re-implementing tokenizer logic in the test

## Acceptance criteria

* within set boundaries 
* behavior
* tests pass

## Deliverable: Concise Report

1. summary
2. test results
3. open issues, concerns or interesting observations


## Observations

### Claude: Open issues / observations

- libtagl.a's $(LIB) Make rule has no prerequisites — it won't auto-rebuild when .o files change.
  This burned us during testing. Worth noting for future changes to tagl.
- The tagsh::_driver (inherited by tagr) still runs for TAGL command execution (args.interpret).
  The tagr_driver is a separate instance used only for the scan pass. These two drivers share _tdb but have independent sessions.
