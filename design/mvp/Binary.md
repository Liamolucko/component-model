# Component Model Binary Format Explainer

This document defines the binary format for the AST defined in the
[explainer](Explainer.md). The top-level production is `component` and the
convention is that a file suffixed in `.wasm` may contain either a
[`core:module`] *or* a `component`, using the `layer` field to discriminate
between the two in the first 8 bytes (see [below](#component-definitions) for
more details).

Note: this document is not meant to completely define the decoding or validation
rules, but rather merge the minimal need-to-know elements of both, with just
enough detail to create a prototype. A complete definition of the binary format
and validation will be present in the [formal specification](../../spec/).


## Component Definitions

(See [Component Definitions](Explainer.md#component-definitions) in the explainer.)
```
component ::= <preamble> s*:<section>*            => (component flatten(s*))
preamble  ::= <magic> <version> <layer>
magic     ::= 0x00 0x61 0x73 0x6D
version   ::= 0x0a 0x00
layer     ::= 0x01 0x00
section   ::=    section_0(<core:custom>)         => ϵ
            | m*:section_1(<core:module>)         => [core-prefix(m)]
            | i*:section_2(vec(<core:instance>))  => core-prefix(i)*
            | a*:section_3(vec(<core:alias>))     => core-prefix(a)*
            | t*:section_4(vec(<core:type>))      => core-prefix(t)*
            | c: section_5(<component>)           => [c]
            | i*:section_6(vec(<instance>))       => i*
            | a*:section_7(vec(<alias>))          => a*
            | t*:section_8(vec(<type>))           => t*
            | c*:section_9(vec(<canon>))          => c*
            | s: section_10(<start>)              => [s]
            | i*:section_11(vec(<import>))        => i*
            | e*:section_12(vec(<export>))        => e*
```
Notes:
* Reused Core binary rules: [`core:section`], [`core:custom`], [`core:module`]
* The `core-prefix(t)` meta-function inserts a `core` token after the leftmost
  paren of `t` (e.g., `core-prefix( (module (func)) )` is `(core module (func))`).
* The `version` given above is pre-standard. As the proposal changes before
  final standardization, `version` will be bumped from `0xa` upwards to
  coordinate prototypes. When the standard is finalized, `version` will be
  changed one last time to `0x1`. (This mirrors the path taken for the Core
  WebAssembly 1.0 spec.)
* The `layer` field is meant to distinguish modules from components early in
  the binary format. (Core WebAssembly modules already implicitly have a
  `layer` field of `0x0` in their 4 byte [`core:version`] field.)


## Instance Definitions

(See [Instance Definitions](Explainer.md#instance-definitions) in the explainer.)
```
core:instance       ::= ie:<instance-expr>                                 => (instance ie)
core:instanceexpr   ::= 0x00 m:<moduleidx> arg*:vec(<core:instantiatearg>) => (instantiate m arg*)
                      | 0x01 e*:vec(<core:export>)                         => e*
core:instantiatearg ::= n:<name> 0x12 i:<instanceidx>                      => (with n (instance i))
core:sortidx        ::= sort:<core:sort> idx:<u32>                         => (sort idx)
core:sort           ::= 0x00                                               => func
                      | 0x01                                               => table
                      | 0x02                                               => memory
                      | 0x03                                               => global
                      | 0x10                                               => type
                      | 0x11                                               => module
                      | 0x12                                               => instance
core:export         ::= n:<name> si:<core:sortidx>                         => (export n si)

instance            ::= ie:<instance-expr>                                 => (instance ie)
instanceexpr        ::= 0x00 c:<componentidx> arg*:vec(<instantiatearg>)   => (instantiate c arg*)
                      | 0x01 e*:vec(<export>)                              => e*
instantiatearg      ::= n:<name> si:<sortidx>                              => (with n si)
sortidx             ::= sort:<sort> idx:<u32>                              => (sort idx)
sort                ::= 0x00 cs:<core:sort>                                => core cs
                      | 0x01                                               => func
                      | 0x02                                               => value
                      | 0x03                                               => type
                      | 0x04                                               => component
                      | 0x05                                               => instance
export              ::= n:<name> si:<sortidx>                              => (export n si)
```
Notes:
* Reused Core binary rules: [`core:name`], (variable-length encoded) [`core:u32`]
* The `core:sort` values are chosen to match the discriminant opcodes of
  [`core:importdesc`].
* `type` is added to `core:sort` in anticipation of the [type-imports] proposal. Until that
  proposal, core modules won't be able to actually import or export types, however, the
  `type` sort is allowed as part of outer aliases (below).
* `module` and `instance` are added to `core:sort` in anticipation of the [module-linking]
  proposal, which would add these types to Core WebAssembly. Until then, they are useful
  for aliases (below).
* Validation of `core:instantiatearg` initially only allows the `instance`
  sort, but would be extended to accept other sorts as core wasm is extended.
* The indices in `sortidx` are validated according to their `sort`'s index
  spaces, which are built incrementally as each definition is validated.

## Alias Definitions

(See [Alias Definitions](Explainer.md#alias-definitions) in the explainer.)
```
core:alias       ::= sort:<core:sort> target:<core:aliastarget> => (core alias target (sort))
core:aliastarget ::= 0x00 i:<core:instanceidx> n:<name>         => export i n

alias            ::= sort:<sort> target:<aliastarget>           => (alias target (sort))
aliastarget      ::= 0x00 i:<instanceidx> n:<name>              => export i n
                   | 0x01 ct:<u32> idx:<u32>                    => outer ct idx
```
Notes:
* Reused Core binary rules: (variable-length encoded) [`core:u32`]
* For `export` aliases, `i` is validated to refer to an instance in the
  instance index space that exports `n` with the specified `sort`.
* For `outer` aliases, `ct` is validated to be *less or equal than* the number
  of enclosing components and `i` is validated to be a valid
  index in the `sort` index space of the `i`th enclosing component (counting
  outward, starting with `0` referring to the current component).
* For `outer` aliases, validation restricts the `sort` of the `aliastarget`
  to one of `type`, `module` or `component`.


## Type Definitions

(See [Type Definitions](Explainer.md#type-definitions) in the explainer.)
```
core:type       ::= dt:<core:deftype>                      => (type dt)        (GC proposal)
core:deftype    ::= ft:<core:functype>                     => ft               (WebAssembly 1.0)
                  | st:<core:structtype>                   => st               (GC proposal)
                  | at:<core:arraytype>                    => at               (GC proposal)
                  | mt:<core:moduletype>                   => mt
core:moduletype ::= 0x50 md*:vec(<core:moduledecl>)        => (module md*)
core:moduledecl ::= 0x00 i:<core:import>                   => i
                  | 0x01 t:<core:type>                     => t
                  | 0x02 a:<core:alias>                    => a
                  | 0x03 e:<core:exportdecl>               => e
core:importdecl ::= i:<core:import>                        => i
core:exportdecl ::= n:<name> d:<core:importdesc>           => (export n d)
```
Notes:
* Reused Core binary rules: [`core:import`], [`core:importdesc`], [`core:functype`]
* Validation of `core:moduledecl` (currently) rejects `core:moduletype` definitions
  inside `type` declarators (i.e., nested core module types).
* As described in the explainer, each module type is validated with an
  initially-empty type index space.
* Validation of `alias` declarators only allows `outer` `type` aliases.
  Validation of these aliases cannot see beyond the enclosing core type index
  space. Since core modules and core module types cannot nest in the MVP, this
  means that the maximum `ct` in an MVP `alias` declarator is `1`.

```
type          ::= dt:<deftype>                         => (type dt)
deftype       ::= dvt:<defvaltype>                     => dvt
                | ft:<functype>                        => ft
                | ct:<componenttype>                   => ct
                | it:<instancetype>                    => it
primvaltype   ::= 0x7f                                 => unit
                | 0x7e                                 => bool
                | 0x7d                                 => s8
                | 0x7c                                 => u8
                | 0x7b                                 => s16
                | 0x7a                                 => u16
                | 0x79                                 => s32
                | 0x78                                 => u32
                | 0x77                                 => s64
                | 0x76                                 => u64
                | 0x75                                 => float32
                | 0x74                                 => float64
                | 0x73                                 => char
                | 0x72                                 => string
defvaltype    ::= pvt:<primvaltype>                    => pvt
                | 0x71 field*:vec(<field>)             => (record field*)
                | 0x70 case*:vec(<case>)               => (variant case*)
                | 0x6f t:<valtype>                     => (list t)
                | 0x6e t*:vec(<valtype>)               => (tuple t*)
                | 0x6d n*:vec(<name>)                  => (flags n*)
                | 0x6c n*:vec(<name>)                  => (enum n*)
                | 0x6b t*:vec(<valtype>)               => (union t*)
                | 0x6a t:<valtype>                     => (option t)
                | 0x69 t:<valtype> u:<valtype>         => (expected t u)
field         ::= n:<name> t:<valtype>                 => (field n t)
case          ::= n:<name> t:<valtype> 0x0             => (case n t)
                | n:<name> t:<valtype> 0x1 i:<u32>     => (case n t (refines case-label[i]))
valtype       ::= i:<typeidx>                          => i
                | pvt:<primvaltype>                    => pvt
functype      ::= 0x40 param*:vec(<param>) t:<valtype> => (func param* (result t))
param         ::= 0x00 t:<valtype>                     => (param t)
                | 0x01 n:<name> t:<valtype>            => (param n t)
componenttype ::= 0x41 cd*:vec(<componentdecl>)        => (component cd*)
instancetype  ::= 0x42 id*:vec(<instancedecl>)         => (instance id*)
componentdecl ::= 0x03 id:<importdecl>                 => id
                | id:<instancedecl>                    => id
instancedecl  ::= 0x00 t:<core:type>                   => t
                | 0x01 t:<type>                        => t
                | 0x02 a:<alias>                       => a
                | 0x04 ed:<exportdecl>                 => ed
importdecl    ::= n:<name> ed:<externdesc>             => (import n ed)
exportdecl    ::= n:<name> ed:<externdesc>             => (export n ed)
externdesc    ::= 0x00 0x11 i:<core:typeidx>           => (core module (type i))
                | 0x01 i:<typeidx>                     => (func (type i))
                | 0x02 t:<valtype>                     => (value t)
                | 0x03 b:<typebound>                   => (type b)
                | 0x04 i:<typeidx>                     => (instance (type i))
                | 0x05 i:<typeidx>                     => (component (type i))
typebound     ::= 0x00 i:<typeidx>                     => (eq i)
```
Notes:
* The type opcodes follow the same negative-SLEB128 scheme as Core WebAssembly,
  with type opcodes starting at SLEB128(-1) (`0x7f`) and going down,
  reserving the nonnegative SLEB128s for type indices.
* Validation of `valtype` requires the `typeidx` to refer to a `defvaltype`.
* Validation of `instancedecl` (currently) only allows `outer` `type` `alias`
  declarators.
* As described in the explainer, each component and instance type is validated
  with an initially-empty type index space. Outer aliases can be used to pull
  in type definitions from containing components.
* Validation of `externdesc` requires the various `typeidx` type constructors
  to match the preceding `sort`.
* Validation of record field names, variant case names, flag names, and enum case
  names requires that the name be unique for the record, variant, flags, or enum
  type definition.
* Validation of the optional `refines` clause of a variant case requires that
  the case index is less than the current case's index (and therefore
  cases are acyclic).


## Canonical Definitions

(See [Canonical Definitions](Explainer.md#canonical-definitions) in the explainer.)
```
canon    ::= 0x00 0x00 f:<core:funcidx> opts:<opts> ft:<typeidx> => (canon lift f opts type-index-space[ft])
           | 0x01 0x00 f:<funcidx> opts:<opts>                   => (canon lower f opts (core func))
opts     ::= opt*:vec(<canonopt>)                                => opt*
canonopt ::= 0x00                                                => string-encoding=utf8
           | 0x01                                                => string-encoding=utf16
           | 0x02                                                => string-encoding=latin1+utf16
           | 0x03 m:<core:memidx>                                => (memory m)
           | 0x04 f:<core:funcidx>                               => (realloc f)
           | 0x05 f:<core:funcidx>                               => (post-return f)
```
Notes:
* The second `0x00` byte in `canon` stands for the `func` sort and thus the
  `0x00 <u32>` pair standards for a `func` `sortidx` or `core:sortidx`.
* Validation prevents duplicate or conflicting `canonopt`.
* Validation of `canon lift` requires `f` to have type `flatten(ft)` (defined
  by the [Canonical ABI](CanonicalABI.md#flattening)). The function being
  defined is given type `ft`.
* Validation of `canon lower` requires `f` to be a component function. The
  function being defined is given core function type `flatten(ft)` where `ft`
  is the `functype` of `f`.
* If the lifting/lowering operations implied by `lift` or `lower` require
  access to `memory` or `realloc`, then validation requires these options to be
  present. If present, `realloc` must have core type
  `(func (param i32 i32 i32 i32) (result i32))`.
* The `post-return` option is only valid for `canon lift` and it is always
  optional; if present, it must have core type `(func (param ...))` where the
  number and types of the parameters must match the results of the core function
  being lifted and itself have no result values.


## Start Definitions

(See [Start Definitions](Explainer.md#start-definitions) in the explainer.)
```
start ::= f:<funcidx> arg*:vec(<valueidx>) => (start f (value arg)*)
```
Notes:
* Validation requires `f` have `functype` with `param` arity and types matching `arg*`.
* Validation appends the `result` types of `f` to the value index space (making
  them available for reference by subsequent definitions).

In addition to the type-compatibility checks mentioned above, the validation
rules for value definitions additionally require that each value is consumed
exactly once. Thus, during validation, each value has an associated "consumed"
boolean flag. When a value is first added to the value index space (via
`import`, `instance`, `alias` or `start`), the flag is clear. When a value is
used (via `export`, `instantiate` or `start`), the flag is set. After
validating the last definition of a component, validation requires all values'
flags are set.


## Import and Export Definitions

(See [Import and Export Definitions](Explainer.md#import-and-export-definitions)
in the explainer.)
```
import ::= n:<name> ed:<externdesc> => (import n ed)
export ::= n:<name> si:<sortidx>    => (export n si)
```
Notes:
* Validation requires all import and export `name`s are unique.
* Validation requires any exported `sortidx` to have a valid `externdesc`
  (which disallows core sorts other than `core module`).


[`core:u32`]: https://webassembly.github.io/spec/core/binary/values.html#integers
[`core:section`]: https://webassembly.github.io/spec/core/binary/modules.html#binary-section
[`core:custom`]: https://webassembly.github.io/spec/core/binary/modules.html#custom-section
[`core:module`]: https://webassembly.github.io/spec/core/binary/modules.html#binary-module
[`core:version`]: https://webassembly.github.io/spec/core/binary/modules.html#binary-version
[`core:name`]: https://webassembly.github.io/spec/core/binary/values.html#binary-name
[`core:import`]: https://webassembly.github.io/spec/core/binary/modules.html#binary-import
[`core:importdesc`]: https://webassembly.github.io/spec/core/binary/modules.html#binary-importdesc
[`core:functype`]: https://webassembly.github.io/spec/core/binary/types.html#binary-functype

[type-imports]: https://github.com/WebAssembly/proposal-type-imports/blob/master/proposals/type-imports/Overview.md
[module-linking]: https://github.com/WebAssembly/module-linking/blob/main/proposals/module-linking/Explainer.md
