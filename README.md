# onnx-nv

ONNX is a file format for machine-learning models: a computation graph,
the weights it uses, and a declaration of which operators it needs. It
is specified at [onnx.ai](https://onnx.ai/onnx/), and its structure is
a [Protocol Buffers](https://protobuf.dev/) message defined in
[`onnx.proto`](https://github.com/onnx/onnx/blob/main/onnx/onnx.proto).
This package reads an ONNX model into typed values, checks it against a
carried operator-set table, and writes it back. It does not run one.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What an ONNX model is

A **model** carries an `ir_version`, a list of **operator-set imports**,
and a **graph**. The `ir_version` is the version of the file format.
The operator-set imports say which **domain** the model's operators
come from and which version of that domain it is written against. The
two numbers are different and are confused constantly: `ir_version` 8
with an opset import of 17 is an ordinary model from 2022.

A **graph** is a list of **nodes**, a list of declared **inputs** and
**outputs**, optional declared types for intermediate values, and the
**initializers** — the weights, as named tensors.

A **node** names an **operator** by its `op_type` and its domain, lists
the names of the values it consumes and the names of the values it
produces, and carries its **attributes**. There is no edge list in the
format: the graph structure is that one node's output name is another
node's input name.

An **attribute** is a named constant configuring an operator:
`kernel_shape`, `axis`, `epsilon`. An attribute can also hold a whole
**subgraph**, which is how `If`, `Loop` and `Scan` carry their branches
and bodies.

A **tensor** has an element type, a list of dimensions, and its data. A
**shape**, as declared on an input or an output, is different: each of
its dimensions is either a number or a **symbolic name**. `batch` is the
usual one, and a model exported for serving has it on every input.

An operator is not one thing. It is a sequence of versions, each with
its own inputs, outputs and attributes, each introduced at a **since
version**. `Squeeze` took its axes as an attribute up to opset 12 and
as an input from opset 13. `Softmax`'s `axis` defaulted to 1 before
opset 13 and to -1 after it. What a node means is decided by the opset
version its domain was imported at.

Protocol Buffers cannot encode a message larger than two gigabytes, and
model weights passed that years ago. ONNX's answer is **external
data**: a tensor's message names a file beside the model, and the bytes
live there.

## Install

```
novo pkg add onnx-nv
```

## Example

```novo
use std.bytes
use std.list
use onnxio
use onnxgraph
use onnxcheck

fn main() [io]
    // The model's bytes, which the caller read. This package opens
    // nothing.
    let src = bytes.zeros(0)

    // `structure_only` skips the weights, which are most of a model's
    // bytes and none of what an inspection needs.
    match onnxio.read(src, onnxio.structure_only())
        Err(e) => println("cannot read: ${e.message()}")
        Ok(m)  =>
            // Which operators the model needs, and whether the carried
            // table knows them all at the version it imported.
            println("${list.len(onnxgraph.operators_used(m))} operator(s)")

            // Everything wrong with it, not just the first thing.
            let findings = onnxcheck.check(m)
            println("${list.len(findings)} finding(s)")
            println("${onnxcheck.has_errors(findings)}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: onnx-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `onnxfault` | Why a model could not be read or written, including the external-data case. |
| `onnxtype` | Element types, shapes whose dimensions may be names, tensors and value infos. |
| `onnxattr` | Attributes, with the type the file declared kept beside the value. |
| `onnxgraph` | The model, the graph, the nodes, and the flat subgraph table. |
| `onnxopset` | The carried operator-set table: which version introduced what, and its arity. |
| `onnxcheck` | Validation, as a list of findings with severities. |
| `onnxio` | Reading and writing, over protobuf-nv. |

## How to choose an entry point

**`onnxio.read_header` reads the first few hundred bytes.** Use it to
decide whether a model can be loaded at all: the opset imports are what
that depends on, and they are at the front.

**`onnxio.structure_only` reads the graph and skips the weights.** Use
it for anything that inspects, checks, draws or converts. A four
gigabyte read becomes a four megabyte one.

**`onnxio.read` reads everything.** Use it when the weights are the
point.

**`onnxcheck.check` answers every finding.** Use `at_least` and
`has_errors` to decide what to do about them.

**`onnxgraph.all_nodes` walks the whole model.** Use it rather than the
main graph's node list, which misses everything inside `If`, `Loop` and
`Scan`.

## The rules a user needs

1. **A dimension can be a name, and the name matters.** Two dimensions
   called `batch` are the same unknown. `onnxtype.element_count`
   answers `None` rather than zero when any dimension is symbolic,
   because a zero-element tensor is a real thing.

2. **`ir_version` is not the opset version.** The first is the file
   format's; the second is the operator set's, per domain.

3. **An operator's meaning depends on the opset version.**
   `onnxopset.schema_for` takes the version and answers the schema in
   force, which is the one with the largest `since_version` no greater
   than it.

4. **An empty input name means an omitted optional input.** `Clip`'s
   `min` can be empty with its `max` present. The position matters, so
   the empty strings are kept and `supplied_input_count` counts the
   rest.

5. **Nodes inside `If`, `Loop` and `Scan` are nodes.** They live in
   subgraphs, which this package holds in one flat table and refers to
   by index. `onnxgraph.all_nodes` walks all of them.

6. **An initializer may also appear among the graph's inputs.** Models
   exported before IR version 4 list their weights there.
   `onnxgraph.true_inputs` is what a caller must actually supply.

7. **External data is not corruption.** A tensor whose bytes live in a
   separate file is how every large model is stored.
   `onnxfault.needs_external_file` identifies the refusal, and
   `onnxio.attach_external` is what a caller that read the file calls.

8. **An attribute's declared type is in the file, not inferred.** An
   empty list of integers and an empty list of floats are
   byte-identical except for that field. `onnxattr.is_consistent` says
   whether the file agrees with itself.

9. **An attribute accessor of the wrong type answers `None`, not a
   default.** An attribute's default belongs to the operator and
   changes between opset versions.

10. **Unknown fields survive a round trip.** Anything this package does
    not model is kept and written back, so a model from a newer ONNX
    comes out with its newer fields intact.

11. **A check that finds nothing is not a proof.** The carried table
    holds arity, attribute names and since-versions. It does not hold
    type constraints, shape inference or semantics.

## What is not included

- **Running a model.** That is `libonnxruntime-sys`, which wraps ONNX
  Runtime. It is a different layer of the registry and this package
  does not depend on it, which is the same division this sentence
  describes stated as a rule the compiler checks.
- **Shape inference.** Reading declared shapes is here; computing the
  shapes of intermediate values from the operators' rules is the
  majority of the ONNX specification's bulk and is a package on top of
  this one.
- **Type constraints.** The table carries arity, not which element
  types each operator accepts.
- **ONNX functions and their inlining.** A function is an operator
  defined as a subgraph. Reading one is here; expanding a call to it is
  not.
- **Training graphs.** `TrainingInfoProto` is read as an unknown field
  and written back unchanged.
- **Quantisation rewriting.** Reading a quantised model is here;
  producing one is a transformation with its own calibration data.
- **Sparse tensors.** `SparseTensorProto` is kept as bytes so a model
  that uses one round trips, and is not modelled.
- **Opening the external data file.** This package performs no input.
  The caller reads it and calls `attach_external`.

## Related packages

**protobuf-nv** is what an ONNX file is made of. This package uses its
low-level surface — the tag arithmetic, the event walk, and the set of
unknown fields — rather than its dynamic messages, because the ONNX
message shapes are fixed and a descriptor set would be a blob to keep
in step with every ONNX release.

**libonnxruntime-sys** runs a model. This package reads one.

**gguf-nv** is the other model container: llama.cpp's format, which
stores weights and hyperparameters and no graph.

**safetensors** in the standard library stores tensors with no graph at
all.

**tokenizers-nv** turns text into the integers a model's input tensor
holds.

## Test vectors

The suite asserts the specifications directly.

- **`onnx.proto`'s `TensorProto.DataType`** fixes the seventeen element
  type codes and their names. Each is asserted, and the element widths
  with them.
- **`AttributeProto.AttributeType`** fixes the thirteen attribute
  kinds. An empty `INTS` and an empty `FLOATS` are asserted to be
  different, which is the case a reader that infers the type gets
  wrong.
- **The operator specification's version history** fixes that `Squeeze`
  changed at opset 13 and that `Concat` is variadic. Both are asserted
  against the table.
- **`Clip`'s optional inputs** fix that an empty input name is a
  position and not an absence.
- The symbolic-dimension rules are asserted on a shape that uses
  `batch` twice: one symbol, and binding it fills both.

Today every one of those assertions reaches a `not implemented` panic.

## Implementation status

| Area | Status |
| --- | --- |
| Element types, shapes, tensors, value infos | declared, not implemented |
| Attributes, with their declared kinds | declared, not implemented |
| Model, graph, nodes, flat subgraph table | declared, not implemented |
| The operator-set table | declared, not implemented |
| Validation findings | declared, not implemented |
| Reading and writing over protobuf-nv | declared, not implemented |
| External data, both directions | declared, not implemented |
| Shape inference | not declared |
| Type constraints | not declared |
| Function inlining | not declared |
| Execution | not declared — libonnxruntime-sys's |

## Licence

Apache-2.0. See [LICENSE](LICENSE).
