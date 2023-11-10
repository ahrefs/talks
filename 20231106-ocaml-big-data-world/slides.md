---
view: https://github.com/maaslalani/slides
theme: dark.json
date: Nov 10, 2023
---

# OCaml in the Big Data World

at Ahrefs

---

# Ahrefs - who we are

* Founded in 2010
* Bootstrapped, with no outside investors
* ~120 people now
* ~70 developers (20+ backend, ~30 frontend, data science, devops)
* HQ in Singapore
* Two thirds work remotely from 24 countries all over the world (but nobody in Japan, yet)
* spoiler: most of our code is in OCaml

---

# What we do

* SaaS for web intelligence, primarily used by SEO and marketing people and agencies
* Processing the whole web and storing it in compressed form into our storage
* Web tools to visualize and extract useful indicators out of that data
    * links between sites (Site Explorer)
    * keywords that people search for (Keywords Explorer)
    * on-demand crawling (Site Audit)
    * site positions and movements in serch results (Rank Tracker)
    * blog and news search (Content Explorer)
    * multi-criteria web search (Web Explorer)

## Ahrefs dashboard

```bash
xdg-open "https://app.ahrefs.com/v2-site-explorer/overview?mode=subdomains&target=fos.kuis.kyoto-u.ac.jp"
```

---

# Yep

* And a search engine: [yep.com](https://yep.com)
* AI
    * summarization
    * image search
    * writing tools
    * content grader
    * content extractor

```bash
xdg-open "https://yep.com/web?q=kyoto+university"
```

---

# What we actually do

* Every minute we crawl 5M pages
* Process 600TB of HTML every day
* Backlinks index covers 200 million domains
* Top 5 most active web crawlers in the world

Here is the screenshot from https://radar.cloudflare.com/traffic/verified-bots as of this morning
```bash
geeqie -f bot.png
```

---

# Hardware

(mostly our own)

* 3000 servers
* 600K CPU cores
* 4PB RAM
* 33PB HDD
* 400PB SSD

More detailed and video at https://ahrefs.com/big-data

To put these numbers to perspective
* our standard workhorse servers have 2TB ram and 256 CPUs
* the GPU servers in our US datacenter are 200kg each
```bash
geeqie -f qts-nvidia/
```
* first shipment of SSD to our SG datacenter was 500kg, took us a week to install them into the servers

---

# SSD installation behind the scenes

```bash
geeqie -f cyxtera-ssd/
```

---

# HPC

* 504 H100 GPU
* 8th cluster in the world https://www.stateof.ai/

```bash
geeqie -f stateofai.png
```

* Scored 19.7 PFlops in linpack
* It would put us at ~30 place in the list of top 500 supercomputers as of June 2023
* https://www.top500.org/lists/top500/list/2023/06/

---

# Big Data

* ~300 billion pages in backlinks index
* ~800PB raw data
* ~600 trillion rows

```text
SELECT formatReadableSize(sum(data_uncompressed_bytes))
FROM cluster(L0L1, system.parts)

┌─formatReadableSize(sum(data_uncompressed_bytes))─┐
│ 788.40 PiB                                       │
└──────────────────────────────────────────────────┘

SELECT formatReadableQuantity(sum(rows))
FROM cluster(L0L1, system.parts)

┌─formatReadableQuantity(sum(rows))─┐
│ 642.81 trillion                   │
└───────────────────────────────────┘

```

---

# How we do it

```
~~~graph-easy --as=boxart
[ WWW ] -> [ Crawler ] -> {start:front,0} [ Storage ] { rows:4 } -> [ Web backend ]
[ Storage ] -> [ Data Science ] -> [ Web backend ] {rows:4} -> [ UI ]
~~~
```

---

# Crawler

```
~~~graph-easy --as=boxart
[ WWW ] -> [ Crawler ] {border:double} -> {start:front,0} [ Storage ] {rows:4} -> [ Web backend ]
[ Storage ] -> [ Data Science ] -> [ Web backend ] {rows:4} -> [ UI ]
~~~
```

* Crawler is actually many different interconnected components
    * scheduler
    * http downloader
    * renderer
    * offline batch processing
* Different types of data command different suitable storage approaches
* Performance puts many constraints on the way things can be done
* All the data is sharded between multiple hosts
* Distributed systems are hard, need to think about consistency and failures
* Internet is wild west, cannot trust anything about the input

---

# Storage

```
~~~graph-easy --as=boxart
[ WWW ] -> [ Crawler ] -> {start:front,0} [ Storage ] { rows:4; border:double } -> [ Web backend ]
[ Storage ] -> [ Data Science ] -> [ Web backend ] {rows:4} -> [ UI ]
~~~
```

* experimented with existing solutions (cassandra,hypertable,etc) - not good enough
* custom key value distributed storage (200 servers)
* mysql/redis for frontend
* elasticsearch
* clickhouse columnar database, geo-replicated

---

# Challenges

```
~~~graph-easy --as=boxart
[ WWW ] -> [ Crawler ] -> {start:front,0} [ Clickhouse || Elasticsearch || Mysql || Custom ] { basename: S; rows:2 } -> [ Web backend ]
[ S.0 ] -> [ Data Science ] -> [ Web backend ] {rows:7} -> [ UI ]
~~~
```

* Many components => many boundaries
* Need to ingest a lot of new data fast
* No maintenance downtime possible
* More and more different ways to read data: ahrefs.com, public API, search engine, 3rd party integrations
* Remote team, working across multiple time zones

---

# Use OCaml everywhere we can

```
~~~graph-easy --as=boxart
[ WWW ] -> {flow:down} [ Crawler (OCaml) ] -> {start:front,0} [ Clickhouse || Elasticsearch || Mysql || Custom (OCaml, C++) ] { basename: S; rows:2 } -> [ Web backend (OCaml) ]
[ S.0 ] -> [ Data Science (Python) ] -> [ Web backend (OCaml) ] {rows:7} -> {flow:down} [ UI (OCaml?) ]
~~~
```

---

# Why OCaml

* Expressive and flexible language, multi-paradigm (can be used as functional or imperative at the same time)
* Great performance without sacrificing ergonomics (GC)
* Static typing + type inference => Easy maintenance
* Native code binaries
* Available in many environments, including the browser
* Stability
* OCaml single core limitation (not anymore with OCaml 5) is not a problem, data is sharded anyway - just shard per process as well

=> small team can do a lot of things fast and maintain reliable codebase long term

iow productivity

---

# Web backend

```
~~~graph-easy --as=boxart
[ WWW ] -> [ Crawler ] -> {start:front,0} [ Storage ] {rows:4} -> [ Web backend ] {border:double}
[ Storage ] -> [ Data Science ] -> [ Web backend ] {rows:4} -> [ UI ]
~~~
```

* REST API server for frontend
* Single entry point to all the data sources for all the clients (ahrefs.com, api, extensions)
* Processes data from many different types of databases
* Define database schemas and communication apis once
* Generate data types and communication protocols from above schema descriptions
* Static definitions of all queries to databases (well typed)
* Typed communication payloads (request body + response value)

---

# UI / Web Frontend

```
~~~graph-easy --as=boxart
[ WWW ] -> [ Crawler ] -> {start:front,0} [ Storage ] {rows:4} -> [ Web backend ]
[ Storage ] -> [ Data Science ] -> [ Web backend ] {rows:4} -> [ UI ] {border:double}
~~~
```

* Multiple ReasonReact applications
* Calls to http endpoints in Web Backend
* Data types generated from database schema descriptions and Web backend routes
* Sharing code with Web backend
* All communications to REST API typed
* Written in Melange

---

# OCaml for Web story

* js_of_ocaml
* Ocsigen
* Bonsai
* vdom
* ReasonML
* Bucklescript
* Rescript
* Melange

---

# First there was js_of_ocaml

```
~~~graph-easy --as=boxart
[ OCaml parser ] {origin:OCaml typechecker; offset:-4,0} -> {end:back,0} [ OCaml typechecker ] {rows:8} -> [ ocamlc bytecode compiler ]
-> [ js_of_ocaml ] {border:double} -> [Ocsigen] {border:double}
[ OCaml typechecker ] -> [ ocamlopt native-code ]
~~~
```

---

# ReasonML + Bucklescript

```
~~~graph-easy --as=boxart
[ OCaml parser ] {origin:OCaml typechecker; offset:-4,0} -> {end:back,0} [ OCaml typechecker ] {rows:8} -> [ ocamlc bytecode compiler ]
-> [ js_of_ocaml ] -> [Bonsai], [vdom] {border:double}, [Ocsigen]
[ ReasonML ] {origin:OCaml parser;offset:0,2; border:double }-> [ OCaml typechecker ] ~~ fork ~~> {flow:down} [Bucklescript] {border:double}
[ OCaml typechecker ] -> [ ocamlopt native-code ]
~~~
```

---

# Rescript

```
~~~graph-easy --as=boxart
[ OCaml parser ] {origin:OCaml typechecker; offset:-4,0} -> {end:back,0} [ OCaml typechecker ] {rows:8} -> [ ocamlc bytecode compiler ]
-> [ js_of_ocaml ] -> [Ocsigen], [Bonsai], [vdom]
[ ReasonML ] {origin:OCaml parser;offset:0,2}-> [ OCaml typechecker ] -> [ ocamlopt native-code ]

[OCaml typechecker] ~~ fork ~~> {flow:down} [Bucklescript] .. rebranded ..> {flow:down} [Rescript typechecker]

( Rescript
[ Rescript parser ] {origin: ReasonML; offset:0,9} -> {minlen:3} [ Rescript typechecker ] -> [ Rescript codegen ]
) {border:double}
~~~
```

---

# Melange

```
~~~graph-easy --as=boxart
[ OCaml parser ] {origin:OCaml typechecker; offset:-4,0} -> {end:back,0} [ OCaml typechecker ] {rows:8} -> [ ocamlc bytecode compiler ]
-> [ js_of_ocaml ] -> [Ocsigen], [Bonsai], [vdom]
[ melange.ppx ]{origin:OCaml parser;offset:0,2;border:double} -> [ OCaml typechecker ]
[ ReasonML ] {origin:OCaml parser;offset:0,4}-> [ OCaml typechecker ] -> [Melange] {border:double}, [ ocamlopt native-code ]

( Rescript
[ Rescript parser ] {origin: OCaml parser; offset:0,10} -> {minlen:3} [ Rescript typechecker ] -> [ Rescript codegen ]
)
~~~
```

---

# Melange

https://melange.re/

* OCaml for JavaScript developers
* Keeps OCaml syntax and semantics
* Compatibile with the OCaml platform (type-checker, tooling, LSP, …)
* Interoperable with JavaScript
* Development supported by Ahrefs, shout-out to our frontend and tooling teams who made this possible

---

# OCaml across the whole stack


```
~~~graph-easy --as=boxart
[ WWW ] -> {flow:down} [ Crawler (OCaml) ] -> {start:front,0} [ Clickhouse || Elasticsearch || Mysql || Custom (OCaml, C++) ] { basename: S; rows:2 } -> [ Web backend (OCaml) ]
[ S.0 ] -> [ Data Science (Python) ] -> [ Web backend (OCaml) ] {rows:7} -> {flow:down} [ UI (OCaml?) ]
~~~
```

---

# OCaml across the whole stack

__Benefits__

* Shared tooling and code (monorepo), 600KLoC OCaml + 400KLoC ReasonML

$ tree
```bash
├── backend
├── ci
├── data_science
├── dune
├── dune-project
├── frontend
├── infra
├── Makefile
[...]

```

$ cat Makefile

```Makefile
fullinit: opam-global-setup
	rm -rf _opam _build
	opam init --bare --shell-setup --enable-shell-hook ${OPAM_INIT_ARGS}
	opam sw -y --repositories "ahrefs-${SLUG_REPO_PATH},mirror" create . --packages=ahrefs-setup,ahrefs-dev-tools-deps,$(OCAML_VERSION) --no-install
	opam install ahrefs-all-deps

build:
    dune build @install

```

---

# OCaml across the whole stack

__Benefits__

* Shared tooling and code (monorepo)
* Shared data schemas and core type definitions : all the way from crawler to UI
* Same semantics across domains (no impedance mismatch beween layers, e.g. variants everywhere)

```ocaml
type href = {
  nofollow : bool;
  alt : string option;
  anchor : string;
  context : (string * string) option;
  ugc : bool ;
  sponsored : bool
}

type link_info = [
   Redirect of int (* HTTP redirect *)
 | Href of href (* html a href *)
 | Frame (* html frame src *)
 | Form (* html form action *)
 | Canonical (* html link rel=canonical *)
 | Alternate (* html link rel=alternate *)
 | Rss (* rss link *)
] <ocaml repr="classic">

```

(this is not exactly ocaml - we will talk about it later)

---

# OCaml across the whole stack

__Benefits__

* Shared tooling and code (monorepo)
* Shared data schemas and core type definitions : all the way from crawler to UI
* Same semantics across domains (no impedance mismatch beween layers, e.g. variants everywhere)
* Reduced friction, less mental overhead, easier for developers to explore other parts of the stack
* More opportunities to automate workflows and integrate all the way down to infrastructure

```ocaml
  let keywords_api =
    reg keywords_api ~root:"keywords_api" Nodes.laksa 1 ~port:8092 {
      prod = {
        redis = redis_prod;
        billing_api = { base_url = billing_base_url; token = "###"; };
      };
      staging = {
        redis = redis_test;
        billing_api = { base_url = billing_staging_base_url; token = "###"; };
      };
    }

```

make gen demo


---

# OCaml across the whole stack

__Benefits__

* Shared tooling and code (monorepo)
* Shared data schemas and core type definitions : all the way from crawler to UI
* Same semantics across domains (no impedance mismatch beween layers, e.g. variants everywhere)
* Reduced friction, less mental overhead, easier for developers to explore other parts of the stack
* More opportunities to automate workflows and integrate all the way down to infrastructure

__Drawbacks__

none

---

# OCaml across the whole stack

__Benefits__

* Shared tooling and code (monorepo)
* Shared data schemas and core type definitions : all the way from crawler to UI
* Same semantics across domains (no impedance mismatch beween layers, e.g. variants everywhere)
* Reduced friction, less mental overhead, easier for developers to explore other parts of the stack
* More opportunities to automate workflows and integrate all the way down to infrastructure

__Drawbacks__

* Many newcomers (frontend,devops) for some reason get frustrated by syntax
* Less packages, need to write bindings ourselves, gets better

---

# Before

* OCaml - PHP - JavaScript
* All communication layers are written by hand and do validation on their own
* Some code is written in multiple languages (validation, encoding & decoding, etc)

---

# Now

* OCaml - OCaml - OCaml
* Communication layers are generated
* Types are shared between all layers
* Cross-team shared knowledge compounds over time

---

# Code generation and shared types

```
~~~graph-easy --as=boxart
[ WWW ] - html -> {flow:down} [ Crawler (OCaml) ] - api -> {start:front,0} [ Clickhouse || Elasticsearch || Mysql || Custom ] { basename: S; rows:2 }

[ S.0 ] - ppx_clickhouse -> [ Web backend (OCaml) ]
[ S.1 ] - esgg -> [ Web backend (OCaml) ]
[ S.2 ] - sqlgg -> [ Web backend (OCaml) ]
[ S.3 ] - extprot -> [ Web backend (OCaml) ]
[ S.0 ] - untyped :( -> [ Data Science (Python) ] - atdpy -> [ Web backend (OCaml) ] {rows:8} - atd -> {flow:down} [ UI (Melange) ]
~~~
```

---

# atd

```ocaml
type version = [ Old | New ]
type input = { v: version; n: string; u: string; }
```
__output:__
> OCaml
```ocaml
type version = [ `Old | `New ]
type input = { v: version; n: string; u: string }
val string_of_version : ?len:int -> version -> string
val version_of_string : string -> version
val string_of_input : ?len:int -> input -> string
val input_of_string : string -> input
```

---

# atd

```ocaml
type version = [ Old | New ]
type input = { v: version; n: string; u: string; }
```
__output:__

> OCaml
```ocaml
type version = [ `Old | `New ]
type input = { v: version; n: string; u: string }
val string_of_version : ?len:int -> version -> string
val version_of_string : string -> version
val string_of_input : ?len:int -> input -> string
val input_of_string : string -> input
```

> D
```D
alias Version = SumType!(Old, New);
@trusted Version fromJson(T : Version)(JSONValue x);
@trusted JSONValue toJson(T : Version)(T x);
struct Input { Version v; string n; string u; }
@trusted Input fromJson(T : Input)(JSONValue x);
@trusted JSONValue toJson(T : Input)(T obj);
```

> python
```python
class Version:
    value: Union[Old, New]
    def from_json_string(cls, x: str) -> 'Version'
    def to_json_string(self, **kw: Any) -> str
class Input:
    v: Version
    n: str
    u: str
```


---

# esgg

```json
{
  "query": {
    "bool": {
      "must": { "query_string": { "query": $query } },
      "should": [
        { "term": { "content": $content }},
        $should,
        $maybe_should?,
        { "terms": $terms }
      ],
      "filter": $filter
}}}
```

__output:__

> input.atd
```ocaml
type basic_json <ocaml module="Json" t="t"> = abstract
type input = {
  filter: basic_json;
  terms: basic_json;
  should: basic_json;
  content: string wrap <ocaml module="Content">;
  query: string;
  ?maybe_should: basic_json nullable
}
```

> output.atd
```ocaml
type _source = { content: string wrap <ocaml module="Content"> }
type hit = { _id: string; _source: _source }
type hits = { total: int; ~hits: hit list }
type result = { hits: hits }
```

---

# sqlgg

```sql
CREATE TABLE IF NOT EXISTS `companies` (
  `company_id` int(10) unsigned NOT NULL,
  `disable_intercom` tinyint(1) NOT NULL DEFAULT '0',
  PRIMARY KEY (`company_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- @create_company_if_not_exists
INSERT IGNORE INTO `companies` (`company_id`, `disable_intercom`) VALUES (@id, 0);
-- @is_intercom_disabled
SELECT disable_intercom <> 0 AS disable_intercom FROM companies where company_id = @id LIMIT 1;
```

__output:__

> OCaml

```ocaml
module Make (T : Sqlgg_traits.M_io) = struct

  let create_company_if_not_exists db ~id =
    let set_params stmt =
      let p = T.start_params stmt (1) in
      T.set_param_Int p id;
      T.finish_params p
    in
    T.execute db ("INSERT IGNORE INTO `companies` (`company_id`, `disable_intercom`) VALUES (?, 0)") set_params

  let is_intercom_disabled db ~id = (* dbd -> int -> bool *)
end

```

---

# ppx_clickhouse

```ocaml
(* defined query *)
let%select memory_trace =
{|
  SELECT
    sum(size) AS size,
    arrayMap(x -> tuple(demangle(addressToSymbol(x)), addressToLine(x)), trace) AS stack
  FROM system.trace_log AS t
  WHERE (trace_type IN @trace_types)
    AND (t.event_date >= toDate(@date_from)) AND (t.event_date <= toDate(@date_to))
    AND t.query_id = @query_id
  GROUP BY trace_type, trace
|}

    (* ... run query with typed parameters *)

    let trace_types = 
    let query = (Sql2.apply_query memory_trace)
        ~query_id:(Sql2.string query_id)
        ~date_from:(Sql2.date date_from)
        ~date_to:(Sql2.date date_to)
        ~trace_types:(Sql2.array @@ List.map Sql2.inject_enum [ trace_type_to_ch_type trace_type ])
    in

    (* ... inspect result *)

    (match result with
    | trace when trace.size > Int64.zero ->
      { value = trace.size; stack = Array.append trace.stack [| "allocate", "" |] }
    | trace when trace.size < Int64.zero -> { value = trace.size; stack = Array.append trace.stack [| "free", "" |] }
    | trace -> { value = trace.size; stack = trace.stack })
```

---

# Code generation and performance

> MetaOCaml
```ocaml
let make_must2 st1 st2 =
  Raw.merge_map_raw
    (C.cde_app1 .<(fun (kx, x) -> (kx, x, false))>.)
    (C.cde_app1 .<(fun (kx, x) -> (kx, x, false))>.)
    (C.cde_app2 .<fun (ka, a) (kb, b) ->
        if ka = kb then (true, true, (ka, a +. b, true))
        else if ka < kb then true, false, (ka, a, false)
        else false, true, (kb, b, false)>.)
    st1 st2
  |> Raw.map_raw ~linear:false (fun e ret ->
    C.if1 (C.get33 e) (ret (C.cde_app1 .<(fun (k,v,_) ->
    (k,v))>. e)))

let make_must streams =
  match streams with
  | [] -> failwith "Empty list of streams"
  | hd::tl -> List.fold_left make_must2 hd tl
```

__output:__

> OCaml
```ocaml
  (* generated code without closures *)
  (let v_141 = Stdlib.ref true in
   while ! v_141 do
     (if (((Tt.AnyDocStream.peek arg1_1) <> None) && ((Tt.AnyDocStream.peek arg2_2) <> None)) && (Stdlib.not (! v_54)) then
        (let v_169 = Stdlib.ref true in
         while ! v_169 do
           (let t_171 =
              let v_170 = Stdlib.Option.get (Tt.AnyDocStream.peek arg1_1) in
              Tt.AnyDocStream.junk arg1_1; v_170 in
  (* 200+ lines *)
```

---

# Future

More OCaml

* Infrastructure description
* Data Science team
* MetaOCaml for machine learning performance optimizations

---

# We are hiring

https://ahrefs.com/jobs


    █▀▀▀▀▀█ ▄ █ ▀▀▄▄█ █▀▀▀▀▀█
    █ ███ █ ▄▄▄ █▄▀ ▀ █ ███ █
    █ ▀▀▀ █ ▄█ ▄▄█▄▄█ █ ▀▀▀ █
    ▀▀▀▀▀▀▀ ▀▄█▄▀ █ ▀ ▀▀▀▀▀▀▀
    ███▀█▄▀▀▀▀▄▀▄█▀▀▄▀ █ ▀ █
    ▄▀ ▄▄█▀   ▀ ▄▀ ▀▄ ▄▀ ▀ ▀█
    ▄ ▀▄▄▄▀▄██▀ ▀█▀  █▀▄▀▄▀█▀
    █ ▀  █▀▀█ ▄█▀▀█▀▀ ▀██▀ ▀█
    ▀ ▀▀ ▀▀▀█ ▄▀▄▀▀▀█▀▀▀█▄▀
    █▀▀▀▀▀█ ▀▄█ ▄▀  █ ▀ █▄▀█▀
    █ ███ █ █▄▀ ▀▀█████▀█▄███
    █ ▀▀▀ █ █▀ █▀▀▄▄▄▄▄▄▄█▀ █
    ▀▀▀▀▀▀▀ ▀▀▀▀▀    ▀▀▀▀▀▀▀▀
