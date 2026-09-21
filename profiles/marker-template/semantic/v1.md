# MARKER Metadata Template Specification — Semantic V1 Scope

> **Status:** Production-stable V1, declared final as `v1.0.0`. This document
> is normative for this release and defines the boundary, binding vocabulary,
> component JSON Schema, cross-component validation rules, deterministic
> projection algorithm, and diagnostic contracts of the lean Semantic V1
> profile.

Semantic V1 is an optional profile for attaching machine-interpretable meaning
to fields in a Core V1 MARKER metadata-template package and deterministically
projecting a validated metadata instance to expanded JSON-LD. It is deliberately
small: the raw JSON instance remains authoritative, while JSON-LD is a derived
representation that can be regenerated from the exact template version,
semantic bindings, and response.

Semantic V1 uses the profile URI already reserved by Core V1:

```text
https://staplescience.com/profiles/marker-template/semantic/v1
```

The `semantics` component is optional for a Core V1 package, but required for a
package that claims Semantic V1 conformance. When a package contains the
top-level `semantics` component, Core V1 requires this URI in `conformsTo`, and
declaring the URI requires the component. Bindings address fields with Core V1
field pointers rooted at `form.schema`. Semantic annotations MUST NOT be
embedded as invented keywords inside the form schema.

## Processing model

Projection takes:

1. a Core V1-conformant template package with a valid Semantic V1 component;
2. a metadata instance that validates against the package's `form.schema`; and
3. an optional absolute root instance IRI supplied by the runtime.

It produces an offline, deterministic expanded JSON-LD representation. If the
runtime does not supply a root instance IRI, projection uses a blank root node.
Projection never changes the stored metadata response.

```text
Core template + Semantic V1 bindings + validated response + optional root IRI
                                      |
                                      v
                         expanded JSON-LD projection
```

## Field resolution and node ownership

Semantic V1 field pointers retain the RFC 6901 token syntax and instance-field
addressing rules defined by Core V1. Resolution MUST also traverse Core V1
local `$ref` values deterministically. A local reference is transparent while
following an instance-bearing path: the resolver dereferences it against the
root `form.schema` and continues resolving the remaining path in the referenced
schema.

For example, given:

```json
{
  "properties": {
    "contributors": {
      "type": "array",
      "items": { "$ref": "#/definitions/contributor" }
    }
  },
  "definitions": {
    "contributor": {
      "type": "object",
      "properties": {
        "name": { "type": "string" },
        "orcid": { "type": "string", "format": "uri" }
      }
    }
  }
}
```

the instance-bearing field pointers are:

```text
/properties/contributors
/properties/contributors/items/properties/name
/properties/contributors/items/properties/orcid
```

They are not pointers into `/definitions`. A resolver encountering the `$ref`
at `contributors/items` dereferences `#/definitions/contributor` before
resolving `properties/name` or `properties/orcid`. Resolution MUST be offline,
MUST use only Core-conformant local references, and MUST fail with a diagnostic
for an unresolved reference or a reference cycle that prevents resolution.

Every binding MAY contain `parentNodePointer`. The property establishes which
`node` binding owns the subject from which that binding's predicate is emitted:

- a binding without `parentNodePointer` is emitted from the root instance node;
- a binding for a field nested within a bound object or object-array node MUST
  set `parentNodePointer` to the field pointer of its nearest containing `node`
  binding;
- `parentNodePointer` MUST resolve to an existing binding whose value kind is
  `node`;
- the child field MUST be inside the object represented by that node after
  local-reference resolution; and
- self-parenting and parent cycles are invalid.

A nested `node` binding follows the same rule, allowing deterministic trees of
related nodes. Parentage MUST NOT be inferred solely from textual pointer
prefixes.

## Included in Semantic V1

Semantic V1 includes:

- a Core-optional, Semantic-required top-level `semantics` component;
- an optional class IRI for the root node;
- partial annotation, with no requirement to bind every form field;
- one absolute predicate IRI for every binding;
- exactly one binding for a given field pointer;
- explicit node-child ownership through `parentNodePointer`;
- three value kinds: `literal`, `iri`, and `node`;
- literal projection for Core scalar types and supported date, date-time, and
  time formats;
- an optional datatype IRI compatible with the bound field;
- an optional fixed BCP 47 language tag for a compatible literal binding;
- direct absolute IRI values;
- exact, local mappings from stored values to absolute IRIs;
- scalar arrays projected as repeated values;
- object and object-array fields projected as related nodes;
- an optional class IRI for each nested node binding; and
- deterministic, offline expanded JSON-LD projection without dereferencing
  remote resources.

The three value kinds have these roles:

- **`literal`** projects a scalar value as an RDF literal. A compatible fixed
  datatype or language may refine its representation.
- **`iri`** projects either a stored absolute IRI or the result of an exact
  local value-to-IRI mapping as an identified RDF node.
- **`node`** projects an object, or each member of an object array, as a related
  node whose child fields may have their own bindings. Nested nodes are blank
  nodes in V1 and may carry an explicitly declared class IRI.

### Literal projection

In the absence of an explicit datatype or language tag, Semantic V1 uses these
datatype IRIs:

| Core field | Default RDF datatype |
| --- | --- |
| `type: "string"` | `http://www.w3.org/2001/XMLSchema#string` |
| `type: "boolean"` | `http://www.w3.org/2001/XMLSchema#boolean` |
| `type: "integer"` | `http://www.w3.org/2001/XMLSchema#integer` |
| `type: "number"` | `http://www.w3.org/2001/XMLSchema#double` |
| string with `format: "date"` | `http://www.w3.org/2001/XMLSchema#date` |
| string with `format: "date-time"` | `http://www.w3.org/2001/XMLSchema#dateTime` |
| string with `format: "time"` | `http://www.w3.org/2001/XMLSchema#time` |

Other Core string formats project with the XSD string datatype shown above.
Projection preserves the validated JSON value: it MUST NOT trim, case-fold,
parse, or otherwise rewrite strings. Boolean, integer, and number values remain
JSON scalars in expanded JSON-LD with the datatype shown above.

An explicit datatype MUST be an absolute IRI. For boolean, integer, number,
date, date-time, and time fields, V1 permits only the corresponding datatype
shown in the table. An unformatted string field MAY declare another absolute
datatype IRI; Semantic V1 preserves its lexical string but does not dereference
the datatype or verify a custom datatype's lexical space.

A fixed language tag is permitted only for an unformatted string field and
MUST be a structurally valid BCP 47 tag. A literal binding MUST NOT declare both
a datatype and a language tag. A language-tagged value is represented as an RDF
language-tagged string rather than with the XSD string datatype.

Every predicate, class, datatype, mapped value, and supplied instance IRI is
used exactly as written. Semantic V1 does not expand prefixes or compact IRIs:
for example, `https://schema.org/name` identifies the Schema.org predicate,
while `schema:name` would be a different IRI with the scheme `schema` and MUST
NOT be used as shorthand for the Schema.org IRI.

Unbound fields are omitted from JSON-LD and retained unchanged in the raw JSON
instance. Missing values and `null` are omitted. Empty strings, `false`, and
`0` are projected when they are valid response values. Scalar arrays produce
repeated RDF values; V1 assigns no semantic significance to array order.

Projection MUST fail with a diagnostic rather than silently omit or rewrite a
bound value when an IRI is invalid or a value requiring a local mapping is
unmapped. Conformance is based on expanded JSON-LD or RDF-equivalent output,
not JSON property order, formatting, or implementation-assigned blank-node
labels.

Local value-to-IRI mappings use exact equality of parsed JSON scalar type and
value. Implementations MUST NOT coerce types, trim strings, fold case, or parse
string values before matching. Consequently, `"1"` does not match `1`, and
`"true"` does not match `true`; `1` and `1.0` are the same JSON number value.
Mapping entries cannot target `null`, because missing values and `null` are
omitted from projection.

## Normative projection algorithm

### Preconditions and result

Projection requires:

1. a Core V1-conformant template package;
2. a Semantic V1-valid `semantics` component;
3. an object response that has already validated against `form.schema`; and
4. an optional runtime `rootInstanceIri` that is an absolute IRI.

For conforming input, the result is an expanded JSON-LD array containing one
root node object. It contains no `@context`: predicates, classes, datatypes, and
identified nodes use their full IRIs. Projection MUST NOT mutate the template
or response.

If `rootInstanceIri` is supplied, the root node receives it as `@id`. Otherwise
the root is an unnamed blank node and the expanded object omits `@id`. If
`semantics.root.classIri` is present, the root receives an `@type` array
containing that class IRI.

### Binding traversal

Bindings are processed in their array order. A binding without
`parentNodePointer` emits its predicate from the root node. A binding with
`parentNodePointer` emits from each response object projected by that exact
parent `node` binding.

The projector translates the binding's schema field pointer into a response
path:

- each `properties/<name>` step reads the object property `<name>`; and
- each `items` step visits every array member in response order.

Local `$ref` traversal is completed during Semantic validation; `/definitions`
never becomes part of the response path. For a child of an object-array node,
the leading `items` step after the parent's field pointer addresses the current
array member and is not traversed a second time. This preserves the association
between every nested response object and its own child values.

A missing property or `null` value emits nothing. Empty strings, `false`, and
`0` are values and MUST be emitted. An empty array emits no value for its
predicate. Scalar-array and object-array members are processed in response
order, but that order has no RDF meaning in V1.

### Value projection

For each non-null value:

- A `literal` binding emits a JSON-LD value object. It contains `@value` and
  either `@language` or `@type`. A fixed language tag is normalized to
  lowercase. Otherwise the explicit datatype is used, or the default datatype
  is selected from the effective Core field type and format using the literal
  table above.
- A direct `iri` binding emits `{ "@id": value }` after confirming that the
  response string is an absolute IRI.
- A mapped `iri` binding performs exact JSON scalar matching and emits
  `{ "@id": mappedIri }`. It never falls back to interpreting an unmapped value
  as an IRI or literal.
- A `node` binding emits one related blank node for an object, or one related
  blank node for every object-array member. Its optional `classIri` becomes an
  `@type` array. Its child bindings are then projected into that same node.

When multiple bindings use the same predicate from the same subject, their
values are appended to one predicate array in binding order. Nested blank nodes
do not receive generated identifiers in V1.

### Failure and determinism

Projection is atomic. If any projection diagnostic is produced, no expanded
JSON-LD result is returned; implementations MUST NOT return a silently partial
graph. Response-value diagnostics use RFC 6901 pointers into the response.
Runtime-option diagnostics use pointers into the projection options.

For a fixed valid template, response, and root IRI, an implementation MUST
produce the same RDF graph. The reference serialization additionally preserves
binding order for properties and response-array order for values. Conformance
comparison ignores JSON object-property order and implementation-assigned blank
node labels.

## Exact-template association boundary

Every collected response MUST remain durably associated with the exact Core V1
template package used to collect it. That package already supplies
`metadata.versionId`. Semantic V1 therefore does not copy `versionId` into the
response, add it to the field-derived JSON-LD graph, or define a metadata-
instance envelope. Response storage and export packaging are responsibilities
of the collecting application.

## Deferred beyond V1

Semantic V1 does not include:

- concept-scheme constraints that do not affect projection;
- expected classes for externally referenced IRI values;
- remote ontology verification or controlled-vocabulary membership checks;
- ontology lookup, browsing, registry, or provider integrations;
- multiple predicates for one field;
- conditional or branch-specific semantic mappings;
- calculated, concatenated, parsed, or otherwise transformed values;
- RDF lists or RDF ordering semantics;
- reverse conversion from JSON-LD into a STAPLE response;
- user-authored or remotely fetched JSON-LD contexts;
- stable identifiers for nested nodes derived from particular response fields;
- dynamic language selection from another response field;
- OWL reasoning, SHACL validation, ontology alignment, or inferred equivalence;
- reverse properties, named graphs, reification, RDF-star, or arbitrary RDF
  graph authoring;
- automatic migration of legacy `ontologyId` annotations; or
- JSON-LD describing the template package itself.

Legacy `ontologyId` values MAY be preserved and presented as migration hints,
but they do not establish Semantic V1 conformance and MUST NOT be converted
without an exact, user-reviewed interpretation.

## Validation and compatibility independence

Semantic V1 validation depends on successful Core V1 package, form-component,
and cross-component validation. It does not depend on RJSF rendering, Form
Studio authoring, or STAPLE deployment compatibility.

```text
Core V1 conformance
|-- Semantic V1 validation
`-- application compatibility assessment

Core V1 + Semantic V1 + valid metadata instance
`-- JSON-LD projection
```

An application-compatibility failure MUST NOT prevent Semantic V1 validation
or invalidate Semantic V1 conformance. Conversely, Semantic V1 conformance does
not claim that a particular application can author, preserve, preview, or
deploy the semantic component. These are separate, versioned capability
assessments.

## Validation diagnostic contract

A validation diagnostic contains `stage`, `code`, `pointer`, and a human-facing
`message`. Consumers MUST use `code`, not `message`, for program logic. The
`pointer` is an RFC 6901 pointer into the outer template package.

The `semantic-schema` stage reports `SEMANTIC_COMPONENT_INVALID` for violations
of the component JSON Schema. The `semantic-profile` stage uses these stable
codes for cross-component rules:

| Code family | Codes |
| --- | --- |
| Validation prerequisite | `SEMANTIC_FORM_SCHEMA_REQUIRED` |
| IRI | `SEMANTIC_IRI_INVALID` |
| Field resolution | `SEMANTIC_FIELD_POINTER_INVALID`, `SEMANTIC_FIELD_POINTER_UNRESOLVED`, `SEMANTIC_FIELD_POINTER_REF_CYCLE`, `SEMANTIC_FIELD_POINTER_DUPLICATE`, `SEMANTIC_FIELD_TYPE_UNRESOLVED` |
| Binding compatibility | `SEMANTIC_BINDING_TYPE_INCOMPATIBLE`, `SEMANTIC_DATATYPE_INCOMPATIBLE`, `SEMANTIC_LANGUAGE_INVALID`, `SEMANTIC_LANGUAGE_INCOMPATIBLE` |
| Exact mappings | `SEMANTIC_MAPPING_VALUE_DUPLICATE`, `SEMANTIC_MAPPING_VALUE_TYPE`, `SEMANTIC_MAPPING_VALUE_NOT_ALLOWED`, `SEMANTIC_MAPPING_ENUM_UNCOVERED` |
| Node ownership | `SEMANTIC_PARENT_POINTER_INVALID`, `SEMANTIC_PARENT_REQUIRED`, `SEMANTIC_PARENT_NOT_FOUND`, `SEMANTIC_PARENT_AMBIGUOUS`, `SEMANTIC_PARENT_NOT_NODE`, `SEMANTIC_PARENT_OUTSIDE_NODE`, `SEMANTIC_PARENT_NOT_NEAREST`, `SEMANTIC_PARENT_SELF`, `SEMANTIC_PARENT_CYCLE` |

Projection uses the `semantic-projection` stage and these stable codes:

| Code | Meaning |
| --- | --- |
| `PROJECTION_PRECONDITION_FAILED` | The template or response does not meet the projection preconditions. |
| `PROJECTION_ROOT_IRI_INVALID` | The supplied root instance IRI is not absolute. |
| `PROJECTION_VALUE_TYPE_INVALID` | A bound response value has an incompatible runtime shape. |
| `PROJECTION_LITERAL_DATATYPE_AMBIGUOUS` | No single deterministic default datatype can be selected. |
| `PROJECTION_IRI_INVALID` | A direct IRI response value is not an absolute IRI. |
| `PROJECTION_MAPPING_MISSING` | A mapped binding has no exact mapping for the response value. |

Any projection diagnostic makes the projection result absent.

## Normative artifacts

The examples-first milestone established the binding vocabulary encoded by the
normative [Semantic V1 component schema](https://staplescience.com/schemas/marker-template/semantic/v1/semantics.schema.json).
The first five non-normative scenarios are maintained in
[`examples/semantic/v1/`](https://github.com/STAPLE-verse/marker-template-spec/tree/v1.0.0/examples/semantic/v1). They cover the title,
default date datatype, direct ORCID IRI, exact local value-to-IRI mapping, and
repeated contributors through a local `$ref`.

The optional TypeScript
[Semantic V1 validator](https://github.com/STAPLE-verse/marker-template-spec/blob/v1.0.0/implementations/typescript/src/semantic.ts) implements
component-shape validation, absolute-IRI and field-pointer checks, deterministic
local `$ref` traversal, effective field-type compatibility, exact mapping rules,
and node ownership with stable diagnostics. Its behavior is exercised by the
cross-component conformance tests.

The optional TypeScript
[Semantic V1 projector](https://github.com/STAPLE-verse/marker-template-spec/blob/v1.0.0/implementations/typescript/src/projector.ts) implements
the algorithm above. The design examples fix complete expected expanded
JSON-LD graphs, while the projector conformance tests cover number, boolean,
`false`, `0`, empty-string, missing, and `null` behavior; scalar arrays; inline
nested objects; fixed languages; explicit datatypes; invalid response IRIs;
unmapped values; and atomic failure.

The same edge cases are published as implementation-independent
[portable projection fixtures](https://github.com/STAPLE-verse/marker-template-spec/blob/v1.0.0/fixtures/semantic/v1/README.md). The
optional local `npm run validate:jsonld` command checks their expected and
projected graphs with an independent JSON-LD 1.1 processor.
