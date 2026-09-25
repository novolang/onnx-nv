# Changelog

All notable changes to onnx-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
