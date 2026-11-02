# Changelog

All notable changes to onnx-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## Unreleased

### Changed

- Internal: the `elif` ladders over a protobuf field number are `match` expressions, and the cycle search holds its node as an optional. No public signature changed.

## 0.1.0 — 2026-10-12

**Every body is implemented, and every test is green.** The interface
is the one 0.0.1 published, unchanged; this release is that interface
made to work, against the suite in `tests/`, which was written first
and asserts `onnx.proto`'s own field numbers and the operator
specification's own version history.

### Added

- **The reader**: `onnxio.read`, `onnxio.read_header` and
  `onnxio.structure_only`. It is ONE reader with three settings.
  `read_header` decodes the header and steps over the graph,
  `structure_only` decodes the graph and steps over the tensor data,
  and `read` decodes everything. Fields are addressed by field number
  over protobuf-nv's low-level surface — the tag arithmetic, the event
  walk, the set of unknown fields — and never through a compiled
  descriptor set, because ONNX's message shapes are fixed and a graph
  can hold a hundred thousand nodes.
- **The writer**: `onnxio.write`, with `encoded_len` and
  `exceeds_protobuf_limit` beside it. A field of `ModelProto` this
  package does not model is KEPT when the file is read and written back
  out, so a model from a newer ONNX — a `TrainingInfoProto`, a metadata
  entry, a field that did not exist when this was written — survives a
  round trip with those fields intact.
- **The external-data path, end to end.** A tensor whose bytes live
  beside the model is refused as `external_data` with its location
  kept, and `onnxio.attach_external` takes the bytes the caller read
  and gives back the model with that tensor's data in it. Bytes whose
  length disagrees with the tensor's own declaration are refused as
  `tensor_size_mismatch`, so the length the model states is checked
  against the length supplied rather than trusted.
- **`tests/onnx_edge_tests.nv`**, the supplement: it builds protobuf by
  hand — one repeated field in all the spellings the wire allows,
  packed and unpacked, fixed32 and fixed64 and varint — and asserts the
  reader against the wire rather than against the writer, which is what
  catches a reader and a writer that agree with each other and with
  nothing else.

### Changed

- The `novo` floor moves from `>= 0.9.0` to `>= 0.20.2`, the compiler
  this release was built and tested with. A floor under a `todo()`
  proves nothing about a body, and the bodies are what changed.
- The version moves to 0.1.0 and the release is recorded
  `stability = "experimental"`, which is what a first working release
  is. The API suite's header now states that the tests are green and
  that they came first.
- The README's implementation-status section is gone, and the two rules
  the implementation settled are stated instead: which unknown fields
  survive a round trip (the model's own), and what a sparse-tensor
  attribute is here (bytes that round trip, not a tensor anything
  interprets).

### Fixed

Two bugs the new edge tests found in this release's own implementation
before it shipped.

- **The main graph's slot.** Subgraphs are pushed onto the flat table
  as they are met while reading, so the main graph — pushed when its
  own read finished — landed behind them, and `OnnxModel.main` (`0`)
  named a subgraph. The main graph's place is now taken before its
  nodes are read.
- **`onnxgraph.nodes_in_order`** called a graph sorted when a node's
  input was produced anywhere at all, so a node consuming a name
  produced LATER was called sorted and `topological_order` returned an
  order that was not one. It now uses the same first-producer-wins rule
  `producer_of` does, and the two agree.

### Known

- **Reading does not validate the graph.** A node that consumes a name
  nothing produces is read without complaint, because deciding that is
  a question about the whole model and the reader answers questions
  about one message at a time. `onnxcheck.check` reports it — the
  finding's code is `dangling_input` and `onnxcheck.dangling_inputs`
  lists the names — and that report is an approximation by
  construction: the flat subgraph table does not record which graph
  encloses which, so a subgraph is allowed its own names and the MAIN
  graph's, and a name supplied by an outer SUBGRAPH is reported as
  dangling.
- **The unknown-field set is the model's own.** A field this package
  does not model inside a node, a graph, an opset import, an attribute
  or a tensor is stepped over when read and is not kept. Only
  `ModelProto`'s unknown fields survive a round trip.
- **`read_header` keeps neither the graph nor the unknown fields.**
  That is what makes it cheap: it answers a model with a blank main
  graph and an empty unknown set, and nothing refuses a write of one,
  so a model read this way and written back is a model with an empty
  graph.
- **`structure_only` keeps no tensor bytes.** The tensors it produces
  carry their names, shapes and types and no data, and a write of such
  a model emits tensors with no data. Nothing refuses it, and nothing
  can: the model does not record how it was read.
- **The operator table is a snapshot** — the default domain to opset
  21 and `ai.onnx.ml` to opset 5. A model importing a newer one still
  READS; `onnxcheck` is what reports that the table cannot speak for
  it.
- **An attribute's declared `type` above 12 is dropped.** ONNX's
  `AttributeType` numbers `SPARSE_TENSORS` 12, `TYPE_PROTO` 13 and
  `TYPE_PROTOS` 14; this package's table stops at 12, so an attribute
  declaring one of those reads with its declaration `UNDEFINED` and the
  `type` field is not written back. The value field itself —
  `tp`/`type_protos`/`sparse_tensor`/`sparse_tensors`, key and length
  included — is kept byte for byte as `OnnxAttrRaw`.
- **A string that contains a NUL is truncated when it is read as
  text.** The reader refuses bytes that are not UTF-8 in a message's
  text fields, but `bytes.to_str` stops at the first NUL, and a
  `string_data` element is kept raw either way.

## 0.0.3 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- protobuf-nv: `^0.0.1` to `^0.1.0`.

## 0.0.2 — 2026-09-25

The package builds with novo 0.11.  Every body is still `todo()`.

- The lock file moves leb128-nv 0.1.5 to 0.1.7, varint-nv 0.1.4 to 0.1.6
  and zigzag-nv 0.1.4 to 0.1.5.  leb128-nv 0.1.5 and varint-nv 0.1.4
  write into lists through names that are not declared `var`, which novo
  0.11 refuses (E2038), so this package did not build with novo 0.11
  against them.  No requirement in the manifest changed.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `onnxtype` — the load-bearing interface. `OnnxDim` has THREE arms,
  because a dimension is a number, a NAME, or genuinely absent, and
  the name is what a serving stack matches on: two dimensions called
  `batch` are the same unknown, and an integer cannot carry that.
  `element_count` answers `None` and not zero when any dimension is
  symbolic, because a zero-element tensor is a real thing. A tensor's
  bytes are kept RAW, with `element_bytes` and `tensor_byte_length`
  beside them, because a four-gigabyte weight decoded into a `[Float]`
  is four gigabytes of boxed numbers.
- `onnxgraph` — subgraphs in ONE FLAT TABLE, referred to by index. A
  graph holds nodes, a node holds attributes, and an attribute can
  hold a graph; that is a directly recursive value type, which cannot
  be written. The arena also makes `all_nodes` a fold, which is the
  walk that does not miss the branches of an `If`. A node's empty
  input name is KEPT, because it means an omitted optional whose
  position matters — `Clip`'s `min` empty with its `max` present — and
  a reader that filtered it would change what the model computes.
  `true_inputs` exists because before IR version 4 an initializer had
  to appear among the inputs too.
- `onnxopset` — the table keyed by domain, op type AND version, because
  an operator is a sequence of versions and not one thing: `Squeeze`
  took its axes as an attribute to opset 12 and as an input from 13.
  It carries arity, attribute names and since-versions, and NOT type
  constraints, shape inference or semantics — those are most of the
  specification's bulk and a package that shipped them would be
  shipping a runtime.
- `onnxattr` — the declared kind is carried BESIDE the value, not
  inferred from it. An empty `INTS` and an empty `FLOATS` are
  byte-identical except for the `type` field, which is exactly
  `Constant`'s `value` against `value_floats`. An accessor of the
  wrong type answers `None` rather than a default, because an
  attribute's default belongs to the operator and changes between
  opset versions.
- `onnxcheck` — findings, not a verdict. A checker that answered
  `Result<(), Error>` would stop at the first problem, and the first
  problem in a converted model is almost never the interesting one.
  Each finding names its graph, its node index, the node's name — which
  ONNX allows to be empty — and a stable code.
- `onnxio` — read by FIELD NUMBER over protobuf-nv's low-level
  surface, not through `pbdyn`. `pbdyn` would need `onnx.proto`'s
  descriptor carried as a blob and a schema lookup per field of a
  graph that can hold a hundred thousand nodes. `pbunknown` keeps what
  is not modelled, so a round trip through a newer ONNX keeps its newer
  fields. `structure_only` skips the weights, which is most of a
  model's bytes and none of what an inspection needs.
- `onnxfault` — thirteen refusals, with `needs_external_file` singled
  out: a tensor whose bytes live beside the model is not corruption,
  it is every large model, and a caller that treated the refusal as
  damage would reject exactly the models people care about.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  onnx-nv.<module>.<fn>`.
- **Execution is not declared at all.** `libonnxruntime-sys` is layer
  `sys` and a `core` package may not depend on one, so the division
  the README describes is also a rule the compiler checks.
- **Shape inference, type constraints and function inlining are not
  declared.** Each is a package on top of this one, and each needs
  more of the ONNX specification than a reader does.
- **The operator table is a snapshot.** It knows the default domain up
  to the opset named by `ONNX_MAX_OPSET`, and a model importing a
  newer one still READS — only the check reports that the table cannot
  speak for it.
