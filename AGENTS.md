# tagd Agent Workspace

## Symlinks
This workspace contains symlinks to these repositories:
* `tagd/` tagd semantic-relational database and TAGL languge
* `tagd-simple-english/` VOA Wordbook Simple English dictionary to be translated into TAGL and httagd web app
* `tagr/` natural language to TAGL translator

## Goals

Tranform (TAGLize) the **commented TAGL entries** in `simple-english.tagl`
into TAGL **PUT** statements containing the semantic relations of the VOA defintion.
Use `tagr` as the first-pass structural translator for the VOA word + gloss,
then use `tagsh` validation plus repository rules to repair and complete the
candidate TAGL in `simple-english.tagl`.

### PUT Grammar

`tagd/README.md` contains BNF grammar examples for TAGL statements:

```
put_statement ::= ">>" subject_sub_relation relations
put_statement ::= ">>" subject_sub_relation
put_statement ::= ">>" subject relations

subject_sub_relation ::= subject SUB TAG

subject ::= TAG

relations ::= relations predicate_list
relations ::= predicate_list

predicate_list ::= relator object_list

relator ::= TAG

object_list ::= object_list ',' object
object_list ::= object

object ::= TAG EQUALS QUANTIFIER
object ::= TAG EQUALS MODIFIER
object ::= TAG EQUALS QUOTED_STR
object ::= TAG
```

However, prioritize the canonical productions, terminals, and parser terminology are defined in
`tagd/tagl/src/parser.y` and warn the user if it diverges from what is in `tagd/README.md`


## TAGL Generation Philosophy

- "Nothing exists in isolation - To be is to be related."
- TAGL is a formal language - use correct TAGL syntax/grammar according to `tagd_pos:` types
- Don't be creative; prefer TAGL statements that use only wording present in the VOA definition where possible.
- the tree heirarchy is constructed with `sub_relations`
- If the definitions provide an abstraction explicitly, implicitly, or by clear induction, it is acceptable to use that abstraction in the hierarchy.
- Part-of-speech matching is not required for every subordinate relation; prefer the hierarchy that best preserves and derives meaning consistently.
- Use the tree structure to benefit usable software; consistency is more important than finding a single perfect ontology.
- Prefer meaningful heirarchies using super ordinate relations from the definition, but fallback to simple, shallow hierarchies when sub relation unknown.
- Avoid metaphysical expansion.
- Prefix each VOA term definition id with a `VOA:` namespace
  so we can distinguish tags from the VOA Word Book from other tags

Use `sub_relations` to create tagspaces with distributed tree structures, not flat

When uncertain, choose the simpler structure.

---

## Imperative Rules

* Source of truth is the VOA Wordbook TSV data:.
   Prefer VOA definition wording in determinig tagd tag names.
  - ` tagd-dictionary/word-list-defs.tsv` - VOA words, English POS, definition (gloss)
* Prioritize definition of tagd `sub_relator`s and `relator`s especially noun and verb phrases according to:
  - reverse frequency of word/ngram occurence
  - reduced list of English POS from multiple senses where it was present in the definition
  In files:
  - ` tagd-dictionary/pos-spans.tsv` generated from `word-list-defs.tsv`
  - ` tagd-dictionarybuild/ngram-freqs.tsv`
* Avoid inventing new high-level ontology unless necessary.
* Use hyponymy strategically using `sub_relator`s (identity relations) to shape the tagspace heirarchy for better sense making.
* Use `tagr` to generate the initial TAGL-shaped candidate from the VOA word + gloss before manually repairing or refining it.
* If inference beyond the VOA definition is required:
  - In `.tagl`, add an inline `-- reasonable induction` comment on the inferred statement.
  - TODO comments are added by human, do not add unless instructed. Do not modify existing `-- TODO` unless instructed to do so.
* Do not refactor or modify the `tagd` library unless explicitly instructed.
* Work inside `tagd-simple-english/` unless library inspection is needed and asked for / verified.
* Keep definitions consistent with existing examples
* Do not delete existing valid entries without instruction
* Add prerequisite subordinate statements inline as needed while working from top to bottom; They should be defined above definitions requiring them as prerequite to work. (this is analogous to "forward declared" class names in C++.
  However, the commented generated VOA definitions do not contain sub relations; that can be either
  1) added in the TAGLized put statement for the VOA definition block
  2) or, defined where need as a prerequite so the TAGLized put statement will not contain the `sub_relator` identity relation

  Example:
    ```tagl
    -- increase forward declared
    >> increase _sub act;

    -- several lines later, `increase` is required as an `object`
    >> show:advertise _sub show
    has product, quality
    to increase, sales
    identified_as "VOA:advertise:0"
    represents word = "advertise"
    categorized_as verb
    has definition = "to show or present the qualities of a product to increase sales"

    -- TAGLized VOA definition
    >> increase
    to_make more
    in size, ammount
    identified_as "VOA:increase:0"
    represents word = "increase"
    categorized_as verb
    has definition = "to make more in size or amount"
  ```
* Define **next word** strictly as the first remaining commented TAGL put operator `-->>` selected with:
  `grep -A4 -w '\-\->>' simple-english.tagl | head -n 5`
- "Process a word" means follow the **Workfown Instructions** iterative loop for one word only.

## Practical Rules

* Keep each VOA sense separate unless the source wording clearly supports a semantic parent-child relation between senses.
* TAGL rendering should follow the VOA wording;
  prioritize finding meaningful subordinate relations (hyponym, or identity relations) as most important and taglize into `sub_relator`
  then taglize the following predicate relations.
* Treat `tagr` output as a candidate shape, not final truth. Repair it using the file context, existing tags, and the rules in this workspace.
* Do not define a sub relation (`>> {relator} {sub_relation} _rel`) for a VOA word just because it is classified
  as adjective, adverb or preposition. The `_rel` is only appropriate as a fallback when the word
  functions primarily as a `relator` when no more specific semantic `super_object` available (e.g. `before`, `in`, `for`).
* When the VOA definition implies a `super_object` that has been previously defined, use that as the `super_object`
  [for temporal (`time`), spatial (`place`), action (`act`)], etc. — use that as the `super_object`.
  If there are multiple senses of a VOA word, use URI prefix style to disambiguate
  Example: `after` = "later" → `time:after _sub time`, not `after _sub _rel`.
* When the VOA definition use a word in the definition corresponding to  a previously defined tag use the previously defined tag.
* Define prerequisite tags before using them in a statement; TAGL requires valid subject-relator-object structure, not bare English fragments.
* Sparse subordinate definitions taken directly from the VOA gloss are acceptable when they preserve the source semantics and avoid unnecessary invented ontology.
* If a statement is not direct VOA wording or an exact normalization of recurring VOA wording, but is still a justified prerequisite or abstraction, mark that statement inline with `-- reasonable induction`.
* Prefer TAGL `object` relations over modifiers `<object> = <modifier`: when information can be represented as a meaningful subject-relator-object statement, model it as an object relation instead of a modifier assignment. Then it becomes a tag in the tree rather than just a "data property" Use modifiers mainly for scalar/literal metadata (quantities, strings) where no useful object node or relation is known or intended.
* In TAGLized VOA definitions
  - do not use hard tags directly when a VOA word can serve the tagd POS type.
  - first derive the VOA word from the hard tag, for example `>> has _sub _has;`, then use the VOA word in subsequent definition statements.
  - single line TAGL statements should not be preceed or succeeded by an empty newline.
  - multiline TAGL statements must be preceed and succeeded by an empty newline.
  - do not cut the multiline TAGLized VOA definition and insert in another location of the file; the VOA definitions were generated alphabetically - preserve that order
* When defining a namespaced or URI-style tag id with `:`, use the superordinate tag as the prefix and the word as the suffix, for example `event:accident` or `place:across`.
* Do not add predicates that are not present in the VOA definition unless TAGL requires a fallback.
* If a more specific subordinate relation is not part of the VOA definition, use `_sub`.
* If a more specific predicate relation is not part of the VOA definition, use `_rel`.
* Validate edited `.tagl` files with `tagsh -f <file> -n` after changes.

## Workflow Instructions

While the **next word** is found:
  1) uncomment the **next word** block
  2) pass the VOA word + gloss through `tagr` to generate a first-pass TAGL candidate.
     Prefer a subject hint when the VOA word should anchor the candidate, for example:
    `echo "<word> <definition>" | tagr/tagr.py --hint subject=<word>`

    Example commented VOA definition block:
    ```tagl
    -->>
    --identified_as "VOA:age:0"
    --represents word = "age"
    --categorized_as noun
    --has definition = "how old a person or thing is"
    ```

    We would execute

    echo "age how old a person or thing is" | tagr/tagr.py --hint subject=age


  3) repair and expand the TAGL definition from the `tagr` candidate as:
    either:
    one and only one sub relation (identity relation)
    or:
    zero sub relations in the TAGLized VOA definition block when the required subordinate statement was already defined earlier as a prerequisite for that same word
    zero or more prodicate relations 
    ```tagd-grammar
    >> <subject> <sub_relator> <object>
       [<relator> <object_list>]
    ```
  4) Test:
     **TAGL validation**:
    `tagd/tagsh/bin/tagsh -f simple-english.tagl -n`
  5) If validation fails, fix the reported issue(s) and repeat step 4 until validation succeeds.
  6) If the required tag corresponding to a VOA word cannot be directly match to a TAGL `tagd_pos:` type word, stop before forcing a hard tag into the definition and make recommendations to the user.
  7) Stop after validation succeeds for that one word; do not process additional words unless explicitly asked.
  8) Show the diff for the successful changes from this processed word.

- "Process the next N words" means batch mode with the same validation loop:
  1) Repeat the single-word loop N times in top-to-bottom order.
  2) For each word, generate a first-pass TAGL candidate with `tagr/`, then repair it according to the workspace rules.
  3) After each word translation, run **TAGL validations**.
    and fix issues until it succeeds before moving to the next word.
  4) Continue automatically until all N words are successfully processed.
  5) Then show one combined diff containing the successful changes for those N words.
* If the selected commented block is a duplicate of an already translated uncommented entry with the same `VOA:id`,
  remove the duplicate commented block, run validation, and then stop;
  do not continue to additional ids unless explicitly asked.

---
