# tagd Agent Workspace

## Symlinks

This workspace contains symlinks to these repositories:
* `tagd/` tagd semantic-relational database and TAGL languge
* `tagd-simple-english/` VOA Wordbook Simple English dictionary to be translated into TAGL and httagd web app
* `tagr/` natural language to TAGL translator

## TAGL

Understand the TAGL language in `TAGL-README.md`

## Definitions

tagspace: a tagd semantic-relational database using TAGL
TAGLize: to translate (or tranform) input into TAGL

## Goals

TAGLize various forms of input and construct meaninful tagspaces.

## Tasks

Task `.md` files are stored in `TASKS.d`.
Only follow the task specified in the user prompt.

## Workflow

### One small step at a time

Do not make sweeping changes. Each edit should do exactly one thing.
Break large tasks into small steps and iterate.

Prefer the smallest change possible:
* write the test first
* make the minimal code change to pass it
* build and test after each step

### Preserve the author's style

When editing existing code:
* do not reformat lines that do not need changing
* do not change whitespace, indentation, or brace style on untouched lines
* do not alter comments unless explicitly asked
* do not remove commented-out code blocks without being explicitly asked

Comments and TODOs are design intent, not clutter.

### Minimal diffs

Produce the smallest possible diff that achieves the goal.
If a line does not need to change to make the feature work, do not touch it.

Assume the author may review changes with `git diff` and will care about
unnecessary edits.

### Reviewability

For non-trivial changes, prefer a patch the author can review rather than a
large rewrite.

### Verification

Build and test after every change.
Before reporting success, make sure the current step is complete and verified.
