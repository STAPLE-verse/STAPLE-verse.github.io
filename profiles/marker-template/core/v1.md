# MARKER Metadata Template Specification — Core V1

> **Status:** Production-stable V1, declared final as `v1.0.0`. This document
> is normative for this release.

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, **SHOULD NOT**,
and **MAY** in this document are to be interpreted as described by RFC 2119 and
RFC 8174 when, and only when, they appear in all capitals.

Core V1 is the ontology-agnostic interoperability contract for a MARKER
metadata-template package. Semantic bindings and JSON-LD projection are an
optional, separately validated profile and are not defined by this document.

## 1. Identifiers

The Core V1 profile URI is:

```text
https://staplescience.com/profiles/marker-template/core/v1
```

The validating package schema `$id` is:

```text
https://staplescience.com/schemas/marker-template/core/v1/package.schema.json
```

These are the target V1 identifiers. During the release-candidate period,
consumers MUST pin the exact tagged candidate that defines the contract they
implement. An incompatible candidate change requires a new immutable
prerelease tag and release asset and MUST be reported in the changelog. After
final `v1.0.0` is declared, these identifiers MUST NOT be reassigned to an
incompatible contract. Editorial corrections that do not change conformance
MAY be published at the same locations.

## 2. Package structure and conformance declaration

A Core V1 package is one ordinary JSON object with exactly these top-level
properties:

| Property | Required | Meaning |
| --- | --- | --- |
| `conformsTo` | yes | Profile URIs implemented by the package; it MUST contain the Core V1 URI. |
| `metadata` | yes | Template-family, version, lifecycle, credit, and descriptive metadata. |
| `form` | yes | The canonical form artifacts. |
| `semantics` | no | The separately specified Semantic V1 payload. |

The package MUST NOT contain top-level `$schema` or `$id` properties. It is a
JSON instance, not a JSON Schema resource. `conformsTo` identifies the broader
normative profile. That profile identifies the machine-readable package schema
that validates the package.

`conformsTo` is an array because a package may conform to both Core V1 and
Semantic V1. It MUST contain unique absolute URIs. A package containing
`semantics` MUST also declare:

```text
https://staplescience.com/profiles/marker-template/semantic/v1
```

Declaring that Semantic V1 URI also requires the `semantics` property. Core
reserves the location and verifies this declaration pairing; the Semantic V1
specification owns the payload shape and behavior.

The canonical form-schema location is `/form/schema`. The canonical RJSF
UI-schema location is `/form/uiSchema`. Locations in this specification are
RFC 6901 JSON Pointers evaluated against the package unless stated otherwise.
Implementations MUST NOT infer either component from another location.

The package validator is written in JSON Schema 2020-12. `form.schema` is a
separate JSON Schema draft-07 document and MUST contain this exact declaration:

```json
"$schema": "http://json-schema.org/draft-07/schema#"
```

The two schemas are validated separately. This deliberately isolates the
modern package contract from the form dialect currently interoperable across
RJSF, Form Studio, and STAPLE.

## 3. Template identity and lifecycle metadata

### 3.1 Family and version identity

`metadata.familyId` is the stable URI of the template family. It MUST remain
the same for every version in that family. `metadata.versionId` is the stable
URI of the exact version represented by the package. The two values MUST be
different.

Neither identifier is an internal database primary key. Applications MAY use
UUID URNs for working drafts and SHOULD use persistent, resolvable HTTPS
identifiers for published resources.

`metadata.version` is a non-empty, human-facing label unique within the family.
Published MARKER templates MUST use a three-component semantic version such as
`1.2.0`. Draft labels MAY follow an application's checkpoint convention, such
as `draft-3`.

A published `versionId` identifies an immutable package. Changing `form`,
`semantics`, or identity-bearing metadata requires a new `versionId` and
version label. Drafts are mutable; after a draft change, `updatedAt` MUST
change.

### 3.2 Required fields

Every package requires these properties in `metadata`:

- `familyId`;
- `versionId`;
- `version`;
- `status` (`draft` or `published`);
- `resourceType`, with the constant value `MetadataTemplate`;
- `title`;
- `createdAt`; and
- `updatedAt`.

`createdAt` and `updatedAt` describe the exact template version represented by
the package, not the creation time of the family. `updatedAt` MUST NOT be
earlier than `createdAt`.

A draft MAY contain incomplete catalog metadata and MUST NOT contain
`publishedAt`.

A published package additionally requires:

- a non-empty `description`;
- a BCP 47 `language` tag describing its labels and documentation;
- a non-empty `contributors` array;
- `publisher`;
- `license`; and
- `publishedAt`.

`publishedAt` MUST NOT be earlier than `createdAt`. `keywords` and `domain` are
RECOMMENDED for discovery but are not Core publication requirements.
`releaseNotes` is optional.

`metadata.title` and `metadata.description` are the canonical catalog and
citation values. Root `form.schema.title` and `form.schema.description` are
form-presentation annotations and MAY differ.

### 3.3 Contributor credit

Core uses one contributor collection rather than separate creator and
contributor collections. A contributor MUST have a non-empty `name`. Every
contributor in a published package MUST have one or more unique, non-empty
`roles`. A contributor MAY perform multiple roles.

`Creator` is the only role with special Core V1 meaning, when a contributor
uses it. It identifies people or organizations primarily responsible for the
intellectual content and lets publication exporters derive creator lists
required by external catalog standards. Core V1 does not require any
contributor to hold this role; an application or a later profile MAY require
it for its own purposes. Other role strings are preserved; a downstream
exporter maps known roles to its vocabulary and handles unknown roles
explicitly.

`nameType` (`Personal` or `Organizational`), `givenName`, `familyName`,
identifiers, and affiliations are optional. Contributor identifiers are
absolute URIs paired with a non-empty scheme name, for example an ORCID URI
with scheme `ORCID`. An affiliation requires a name. When an affiliation
identifier is present, its identifier scheme is also required, and vice versa.

Contributor array order is preserved. Among contributors with the `Creator`
role, it expresses the intended citation order. MARKER ownership,
collaboration permissions, and importing a template MUST NOT automatically
confer contributor credit.

### 3.4 Publisher and license

`publisher.name` is required. Its optional `identifier` is an absolute URI and
must be paired with `identifierScheme`. Publisher describes the organization
making the published template available; it is not inferred to be a creator.

`license` describes reuse terms for the template package. It MUST contain at
least one of:

- `identifier`, such as the SPDX identifier `CC-BY-4.0`; or
- `uri`, such as `https://creativecommons.org/licenses/by/4.0/`.

The template license does not declare the license of metadata instances later
collected with the template, the research objects those instances describe, or
an external source from which a MARKER template was derived.

## 4. Form Schema Profile

The form schema MUST be valid JSON Schema draft-07 and MUST have an object root
with `type: "object"` and a `properties` object. It MUST use only the Core V1
keyword and format subset enforced by the conformance suite.

Core V1 supports scalar string, number, integer, and boolean fields; nested
objects; homogeneous arrays; constraints on those values; enumerations and
constants; `allOf`, `anyOf`, `oneOf`, `not`, and `if`/`then`/`else` composition;
draft-07 `dependencies`; annotations; and local references into `definitions`.

External references are not portable and MUST NOT be used. Every `$ref` MUST
be a local fragment beginning `#/definitions/` and MUST resolve within the
same form schema. Tuple validation (`items` as an array), custom formats, and
the legacy non-standard `format: "textarea"` MUST NOT be used. Presentation
belongs in `form.uiSchema`.

The supported string formats are `date`, `date-time`, `time`, `email`, `uri`,
`uri-reference`, `hostname`, `ipv4`, `ipv6`, `uuid`, and `regex`.

## 5. UI Schema Profile and cross-component integrity

`form.uiSchema` is an RJSF `uiSchema` object. Field-named entries MUST address
fields present at the corresponding location in `form.schema`. For arrays,
the reserved `items` entry addresses the item schema. Keys beginning `ui:` are
directives, not field names.

Core V1 permits the portable RJSF directives exercised by the conformance
suite. Widget names are restricted to `text`, `textarea`, `password`, `email`,
`uri`, `date`, `date-time`, `alt-date`, `alt-datetime`, `checkbox`, `select`,
`radio`, `hidden`, `updown`, and `range`. An implementation MAY preserve other
UI directives, but a package using an unknown widget is not Core V1 conformant.

Every string in `ui:order`, other than the single wildcard `*`, MUST name a
field available at that object location. A field MUST NOT occur more than once
in an order array.

## 6. Field addresses

Semantic V1 addresses a field with an RFC 6901 JSON Pointer evaluated against
`form.schema`, not against the outer package. A field pointer MUST resolve to a
schema object that describes a metadata-instance location. Examples:

```text
/properties/title
/properties/contact/properties/email
/properties/contributors/items/properties/orcid
```

Pointer tokens use RFC 6901 escaping (`~0` for `~`, `~1` for `/`). A definition
such as `/definitions/identifier` is a reusable schema, not itself an instance
field, and MUST NOT be used as a field address. A property whose schema is a
`$ref` is addressed at its property location, not at the referenced definition.

Core validates the field-address boundary but does not define semantic
resolution behavior. Semantic V1 may retain this instance-bearing pointer
syntax while defining deterministic traversal through Core-conformant local
`$ref` values; that traversal remains part of Semantic V1 rather than Core.

Field addresses are version-local. Producers of a new template version MUST
update semantic pointers when fields move or are renamed. Consumers MUST reject
a semantic binding whose pointer does not resolve in that package.

## 7. Semantic boundary

Semantic bindings MUST NOT add keywords to `form.schema`. The optional
`semantics` object is a sibling of `metadata` and `form`. Core validation treats
its content as opaque after checking the package shape and the corresponding
Semantic V1 declaration in `conformsTo`.

Complete validation MUST dispatch `semantics` to the Semantic V1 validator. A
processor that does not recognize Semantic V1 may still establish Core
conformance, but MUST report that complete semantic validation was not
performed. A known-invalid semantic payload fails Semantic V1 conformance
without changing the meaning or Core conformance of `form.schema`.

Core V1 defines no general extension registry. A future independent component
must demonstrate a concrete use case before Core adds another top-level
component or a generalized extension mechanism.

## 8. Validation order and conformance

Validators MUST run these stages in order and stop dependent stages after a
failure:

1. Validate the outer package with the Core V1 package schema.
2. Validate `form.schema` as a JSON Schema draft-07 schema.
3. Validate the Form Schema Profile, UI Schema Profile, and cross-component
   integrity rules in this specification.
4. Run RJSF rendering, Form Studio preservation/authoring, and STAPLE deployment
   compatibility checks against their declared version coordinates.
5. If present, validate `semantics` using Semantic V1.

The numbered order is a complete-processing sequence, not a dependency from
Semantic V1 validation on stage 4. Semantic validation depends only on stages
1–3 and MAY run independently of application compatibility. A stage-4 failure
MUST NOT prevent Semantic V1 validation or invalidate Semantic V1 conformance.

A package is **Core V1 conformant** when stages 1–3 pass. Compatibility results
from stage 4 MUST be reported separately because they are tied to application
versions. Stage 4 passes when RJSF rendering, Form Studio lossless preservation,
and STAPLE deployment pass. Visual authoring MAY be reported as `unsupported`
when Form Studio deliberately exposes a preserved construct read-only; that is
not data loss and does not fail stage 4.

A package is **fully validated** for a processor only when stages 1–4 pass and,
when `semantics` is present, the processor recognizes and successfully validates
Semantic V1.

Diagnostics MUST identify the stage, a stable code, and an RFC 6901 pointer
into the package. The fixtures in `fixtures/v1/` are the normative examples of
those diagnostics.
