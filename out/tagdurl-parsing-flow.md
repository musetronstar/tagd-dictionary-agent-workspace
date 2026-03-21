# tagdurl Parsing: Hierarchy, Organization, and Flow

## Overview

A **tagdurl** is a URL-based syntax for encoding TAGL commands into HTTP paths.
The `httagd` web service translates incoming HTTP requests with tagdurls into
TAGL statements, which are parsed and executed by the TAGL driver.

---

## File Hierarchy

```
tagd/
├── httagd/
│   ├── include/httagd.h             # htscanner, httagl, request, response, callback
│   ├── src/httagd.cc                # tagdurl scanning & translation (main impl)
│   ├── tests/
│   │   ├── Tester.h                 # Unit test suite with tagdurl examples
│   │   └── tester.exp               # Integration tests via curl
│   └── architecture.md              # MVC architecture diagram
│
├── tagl/
│   ├── include/
│   │   ├── scanner.h                # Scanner interface & re2c macros
│   │   ├── parser.h                 # Token definitions (TOK_CMD_GET, etc.)
│   │   └── tagl.h                   # TAGL driver class
│   ├── src/
│   │   ├── scanner.re.cc            # re2c lexical scanner (source)
│   │   ├── scanner.out.cc           # Generated scanner output
│   │   ├── scanner.cc               # Scanner runtime implementation
│   │   ├── parser.cc                # Lemon parser implementation
│   │   └── tagl.cc                  # Driver: init, execute, callbacks
│   └── tests/Tester.h               # TAGL parser test suite
│
└── tagd/
    ├── include/tagd.h               # Main tag header
    ├── include/tagd/
    │   ├── codes.h                  # Error codes
    │   └── hard-tags.h              # Hard tag constants
    └── src/
        ├── url.cc                   # URL parsing/encoding utilities
        └── tagd.cc                  # Tag implementation
```

---

## Key Classes

### `request` / `response` — `httagd/include/httagd.h:235-296`

```cpp
class request {
    http_method  method;
    std::string  _path;           // URL path: /dog, /mammal/, etc.
    url_query_map_t _query_map;   // ?v=view &c=context &q=search

    path()               // Returns URL path
    query_opt_search()   // ?q=...
    query_opt_view()     // ?v=...
    query_opt_context()  // ?c=...
};
```

### `htscanner` — `httagd/include/httagd.h`, impl `httagd/src/httagd.cc:141`

```cpp
class htscanner : public TAGL::scanner {
    void scan_tagdurl_path(int cmd, const request&);
    // Parses URL path segments and emits TAGL tokens to the driver
};
```

### `httagl` — `httagd/src/httagd.cc:300`

```cpp
class httagl : public TAGL::driver {
    tagd::code execute(transaction&);   // Routes HTTP method → tagdurl parser
    tagd::code tagdurl_get(request&);   // line 361
    tagd::code tagdurl_put(request&);   // line 369
    tagd::code tagdurl_del(request&);   // line 377
};
```

### `TAGL::driver` — `tagl/include/tagl.h:102-199`

```cpp
class driver : public tagd::errorable {
    scanner  *_scanner;
    void     *_parser;   // Lemon parser context
    int       _cmd;      // TOK_CMD_GET / PUT / DEL / QUERY

    tagd::code execute(const std::string&);  // Parse TAGL statement
    tagd::code execute(struct evbuffer*);    // Stream-based parse
    void       parse_tok(int tok, std::string* val);  // Inject token
    void       do_callback();                // Dispatch to callback
};
```

### `callback` — `httagd/include/httagd.h:298-324`, impl `httagd/src/httagd.cc:670+`

```cpp
class callback : public TAGL::callback {
    void cmd_get(const tagd::abstract_tag&);    // line 670
    void cmd_put(const tagd::abstract_tag&);    // line 713
    void cmd_del(const tagd::abstract_tag&);    // line 719
    void cmd_query(const tagd::interrogator&);  // line 726
    void finish();                               // line 429 — send HTTP reply
};
```

---

## Data Flow

```
HTTP Request (evhtp)
  GET /dog
  PUT /mammal  --data "_is_a animal"
  DELETE /turtle
        │
        ▼
main_cb()                               httagd/src/httagd.cc:877
  Creates: request, response, transaction, callback, httagl driver
        │
        ▼
httagl::execute(transaction&)           httagd/src/httagd.cc:300
  Maps htp_method → HTTP_* enum
  HEAD/GET  → tagdurl_get()
  PUT/POST  → tagdurl_put()  [then execute(evbuffer) for body]
  DELETE    → tagdurl_del()
        │
        ▼
httagl::tagdurl_get/put/del(request&)   httagd/src/httagd.cc:361-383
  1. this->init()   — reset parser state
  2. htscanner::scan_tagdurl_path(cmd, req)
        │
        ▼
htscanner::scan_tagdurl_path(cmd, req)  httagd/src/httagd.cc:141
  Parses URL path segments, emits TAGL tokens:
  /dog            → TOK_CMD_GET, TOK_TAG("dog")
  /mammal/        → TOK_CMD_QUERY, TOK_INTERROGATOR, TOK_SUB_RELATOR, TOK_TAG("mammal")
  /mammal/legs    → ... + TOK_RELATOR, TOK_TAG("legs")
  /mammal/?q=...  → ... + TOK_RELATOR("_terms"), TOK_QUOTED_STR("...")
  Token injection via: driver::parse_tok(TOK_*, string*)
        │
        ▼
TAGL::driver::parse_tok(tok, val)       tagl/src/tagl.cc
  Parse(_parser, tok, val, &_tag)       — Lemon parser call
  Builds tagd::abstract_tag:
    _tag->id()           = subject
    _tag->super_object() = subordinate
    _tag->relations[]    = {relator, object, modifier}
        │
        ▼
TAGL::driver::do_callback()             tagl/src/tagl.cc:194
  TOK_CMD_GET   → callback->cmd_get(*_tag)
  TOK_CMD_PUT   → callback->cmd_put(*_tag)
  TOK_CMD_DEL   → callback->cmd_del(*_tag)
  TOK_CMD_QUERY → callback->cmd_query(*_tag)
        │
        ▼
httagd::callback handlers
  cmd_get:   look up tag in tagdb → get view → expand template → buffer
  cmd_put:   insert tag into tagdb via session
  cmd_del:   delete tag from tagdb
  cmd_query: query tagdb with interrogator → view → buffer
  finish():  set Content-Type, send HTTP reply
        │
        ▼
HTTP Response (evhtp)
  200 OK / 400 / 404 / 409 / 500
```

---

## Tagdurl → TAGL Translation Examples

| HTTP Method | URL                   | Emitted TAGL                                      |
|-------------|-----------------------|---------------------------------------------------|
| GET         | `/dog`                | `<< dog`                                          |
| GET         | `/mammal/`            | `?? _interrogator _is_a mammal`                   |
| GET         | `/mammal/legs,tail`   | `?? _interrogator _is_a mammal * legs, tail`      |
| GET         | `/*/fins,hair`        | `?? _interrogator * fins, hair`                   |
| GET         | `/mammal/?q=can+bark` | `?? _interrogator _is_a mammal _has _terms = "can bark"` |
| PUT         | `/dog` + body         | `>> dog _is_a mammal _has legs`                   |
| DELETE      | `/turtle`             | `!! turtle`                                       |

---

## Token Types — `tagl/include/parser.h`

| Token           | Value | Meaning              |
|-----------------|-------|----------------------|
| TOK_CMD_GET     | 11    | `<<`                 |
| TOK_CMD_PUT     | 13    | `>>`                 |
| TOK_CMD_DEL     | 14    | `!!`                 |
| TOK_CMD_QUERY   | 16    | `??`                 |
| TOK_TAG         | 19    | user-defined tag id  |
| TOK_SUB_RELATOR | 20    | `_is_a`, `_sub`      |
| TOK_RELATOR     | 21    | `_has`, `_can`, etc. |
| TOK_WILDCARD    | 25    | `*`                  |
| TOK_INTERROGATOR| 18    | `_what`              |
| TOK_QUOTED_STR  |  9    | `"string"`           |

---

## Error Code → HTTP Status Mapping — `httagd/src/httagd.cc:444`

| tagd code           | HTTP Status               |
|---------------------|---------------------------|
| `TAGD_OK`           | 200 OK                    |
| `TAGL_ERR`          | 400 Bad Request           |
| `TS_MISUSE`         | 400 Bad Request           |
| `TS_NOT_FOUND`      | 404 Not Found             |
| `TS_DUPLICATE`      | 409 Conflict              |
| `TS_INTERNAL_ERR`   | 500 Internal Server Error |

---

## Existing Tagdurl Test Instances

### `httagd/tests/tester.exp` (curl integration tests)

```
GET  /dog                      → expect "dog _is_a mammal"
GET  /mammal/                  → expect children list
GET  /mammal/legs,tail         → expect filtered query results
GET  /mammal/?q=can+bark       → expect search results
GET  /doggy?c=simple_english   → expect context-translated result
PUT  /dolphin  body: ">> dolphin _is_a whale"
PUT  ""        → 400 Bad Request
PUT  /         → 400 Bad Request
PUT  /pigeon?q=oops            → 400 Bad Request (search on PUT)
DELETE /turtle
```

### `httagd/tests/Tester.h` (unit tests)

Fixture tags:
- `mammal _is_a animal`
- `dog _is_a mammal` (`_has legs, tail, fur`; `_can bark, bite`)
- `cat _is_a mammal` (`_has legs, tail, fur`; `_can meow, bite`)
- URL tag: `https://en.wikipedia.org/wiki/Dog` (about dog)

---

## Architecture (MVC)

```
         +-----------+     +--------------------+
         |  (model)  |     |        view        |
         |-----------|     |--------------------|
         |   tagdb   |     | handler | template |
         +-----------+     +--------------------+
             ^     \              ^      /
              \     \            /      /
               \     v          /      v
            +-------------------------------+
            |   tagl_callback (controller)  |
            |-------------------------------|
            |  route view   | call handler  |
            |---------------|---------------|
            |  tagdb CRUD   |expand template|
            |---------------|---------------|
            | http session  | add response  |
            +-------------------------------+
                 ^                    |
                 |                    v
            +-------------------------------+
            | scan/parse    |    write      |
            | method and    |   response    |
            | tagdurl       |               |
            |-------------------------------|
            |          HTTP server          |
            +-------------------------------+
                 ^                    |
                 |                    v
            +---------+          +----------+
            | request |          | response |
            +---------+          +----------+
```
