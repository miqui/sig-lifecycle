# Lifecycle Specification

## Version History

| Version | Date | Notes |
| ---- | ---- | ---- |
| 1.0.0 | TBD | Initial release |

## Introduction

OpenAPI's only built-in signal for the end of an operation's life is the
`deprecated` boolean on the Operation Object. It says an operation is going
away, but nothing about *when*, *why*, where to migrate to, or what replaces
it. HTTP already has standard, structured ways to communicate this at
runtime — the `Deprecation` header ([RFC 9745](https://www.rfc-editor.org/rfc/rfc9745))
and the `Sunset` header ([RFC 8594](https://www.rfc-editor.org/rfc/rfc8594)) —
but there is no standard, design-time way to describe that information in an
OpenAPI description, so providers fall back to prose in `info.description`,
one-off `x-` extensions, or per-operation custom headers, none of which are
interoperable or reliably machine-readable.

This matters more as OpenAPI descriptions feed more than human-facing
documentation. Doc-gen and SDK-gen tools need a stable place to source
`@deprecated` annotations and migration links. AI agents and MCP servers that
register OpenAPI operations as tools need a design-time signal they can fold
into a tool's description *before* it is ever invoked, and a runtime signal
(from response headers) they can act on *after* invocation. A structured
Lifecycle Object serves both without requiring either kind of consumer to
parse prose.

This specification defines that object and a deterministic mapping from its
fields to the `Deprecation`, `Sunset`, and `Link` headers, so tooling can treat
the metadata as either documentation or a generation source for those
headers, per its needs.

**Scope.** The Lifecycle Object applies only to the Operation Object (see
[Feature Fields](#feature-fields)). To apply the same lifecycle metadata to
many operations, use [Overlays](https://spec.openapis.org/overlay/latest.html).

## HTTP Deprecation and Sunset Signals

The Lifecycle Object is a design-time description of signals that HTTP
already defines for runtime use. Each field that maps to an HTTP signal is
named after it:

| HTTP signal | Defined in | Meaning | Lifecycle Object field |
| ---- | ---- | ---- | ---- |
| `Deprecation` header | [RFC 9745 §2](https://www.rfc-editor.org/rfc/rfc9745#section-2) | The resource is, or will be, deprecated as of a date. | `deprecation.date` |
| `deprecation` link relation | [RFC 9745 §3](https://www.rfc-editor.org/rfc/rfc9745#section-3) | Human-readable information about the deprecation, such as a migration guide. | `deprecation.infoLink` |
| `successor-version` link relation | [RFC 5829](https://www.rfc-editor.org/rfc/rfc5829) | The resource that replaces this one. | `deprecation.successorLink` |
| `Sunset` header | [RFC 8594 §3](https://www.rfc-editor.org/rfc/rfc8594#section-3) | The resource is expected to become unavailable at a date. | `sunset.date` |
| `sunset` link relation | [RFC 8594 §6](https://www.rfc-editor.org/rfc/rfc8594#section-6) | Human-readable information about the sunset, such as a sunset policy. | `sunset.policyLink` |

Link relations are carried in the `Link` header defined by
[RFC 8288](https://www.rfc-editor.org/rfc/rfc8288). The
[Header Mapping](#header-mapping) gives the exact header each field
generates.

## Relationship to the OpenAPI Specification

The Lifecycle Specification is a companion specification defining a
[Standardized API Feature](https://github.com/OAI/OpenAPI-Specification/blob/v3.3-dev/src/oas.md#standardized-api-features)
(SAF) for OpenAPI 3.3 and later — see
[OAI/OpenAPI-Specification discussion #5310](https://github.com/OAI/OpenAPI-Specification/discussions/5310)
for background on the SAF concept, and
[OAI/OpenAPI-Specification#5447](https://github.com/OAI/OpenAPI-Specification/pull/5447)
for how SAFs are declared.

An OpenAPI document opts in to this specification by naming it in the
`features` field of the OpenAPI Object, which maps SAF names to the version
of each companion specification in use:

```yaml
openapi: 3.3.0
features:
  security: 1.0.0
  lifecycle: 1.0.0
```

See
[OpenAPI and SAF Specification Versions Across Multiple Documents](https://github.com/OAI/OpenAPI-Specification/blob/v3.3-dev/src/oas.md#openapi-and-saf-specification-versions-across-multiple-documents)
in the OAS for guidance on declaring SAFs, including the default value of
`features` and using SAFs across multiple OpenAPI Description documents.

SAFs do not add fields directly to core OAS objects. Instead, each SAF's
objects are placed under the `featureFields` field, keyed by the SAF's name.
When `lifecycle` is declared in `features`, the Lifecycle Object is placed
under `featureFields` on the Operation Object, as described in
[Feature Fields](#feature-fields). The Lifecycle Object is scoped to
individual operations (see Introduction):

```yaml
paths:
  /flights:
    get:
      deprecated: true
      featureFields:
        lifecycle:
          # Lifecycle Object fields here
```

**Editor's note.** The `featureFields` field is expected to be defined by the
OpenAPI Specification on the OpenAPI, Path Item, and Operation Objects. Until
it is, this specification describes only its use on the Operation Object.

This version of this specification is compatible with OAS 3.3, the first
version of the OAS to support SAFs, and with later OAS 3.x versions. Only the
major and minor version numbers determine compatibility, as described in
[OpenAPI and SAF Specification Versions Across Multiple Documents](https://github.com/OAI/OpenAPI-Specification/blob/v3.3-dev/src/oas.md#openapi-and-saf-specification-versions-across-multiple-documents).

## Definitions

### Feature Fields

When a document declares `lifecycle` in its `features` field, the following
key is defined in the `featureFields` of the
[Operation Object](https://spec.openapis.org/oas/latest.html#operation-object):

| Key | Type | Description |
| ---- | ---- | ---- |
| `lifecycle` | [Lifecycle Object](#lifecycle-object) | Deprecation and sunset details for this operation. |

The `lifecycle` key is defined only for the Operation Object. A `lifecycle`
key in the `featureFields` of any other object, such as the OpenAPI Object or
a Path Item Object, is invalid, and tools MUST report an error.

A `lifecycle` key in `featureFields` in a document that does not declare
`lifecycle` in `features` is not governed by this specification.

### Lifecycle Object

A **Lifecycle Object** describes the deprecation and sunset status of a
single operation. It is optional, and expands on the meaning of
`deprecated: true`: it applies only to an Operation Object that sets
`deprecated: true` (see
[Relationship to `deprecated`](#relationship-to-deprecated)).

#### Lifecycle Object Fixed Fields

| Field Name | Type | Description |
| ---- | ---- | ---- |
| `deprecation` | [Deprecation Object](#deprecation-object) | **REQUIRED.** When and why the operation is deprecated, and what replaces it. |
| `sunset` | [Sunset Object](#sunset-object) | When the operation is expected to stop accepting requests. |

#### Relationship to `deprecated`

The Operation Object's `deprecated` field is the authoritative signal of
whether an operation is deprecated. The Lifecycle Object only adds detail to
an operation that `deprecated` already marks as deprecated.

An Operation Object with a Lifecycle Object in `featureFields` MUST set
`deprecated: true`. If a Lifecycle Object is present and `deprecated` is
absent or `false`, the OpenAPI Description is invalid, and tools MUST report
an error rather than silently ignore the Lifecycle Object.

To announce a deprecation that takes effect in the future, set
`deprecated: true` and give the future date in `deprecation.date`.

This specification does not change the meaning of `deprecated: true` when
there is no Lifecycle Object. An operation can be marked deprecated without
this SAF, as before, which allows APIs to adopt the Lifecycle Object
gradually. The full value comes from using both: `deprecated: true` as the
signal that existing tools already recognize, and the Lifecycle Object for
the dates, reason, and links that tools need to explain the deprecation and
generate headers.

#### Lifecycle Link Relations in Response Headers

When a document declares `lifecycle` in its `features` field, the Lifecycle
Object is the only way to describe `Link` headers with the `deprecation`,
`successor-version`, or `sunset` relation. This applies to every operation in
the document, whether or not the operation has a Lifecycle Object.

A [Header Object](https://spec.openapis.org/oas/latest.html#header-object)
for the `Link` header in a Response Object's `headers` MUST NOT describe a
link with any of these relations, for example in its `description`,
`schema`, or examples. Tools MUST report an error if it does.

#### Example

```yaml
paths:
  /v1/flights:
    get:
      operationId: listFlights_v1
      summary: List all flights
      deprecated: true
      featureFields:
        lifecycle:
          deprecation:
            date: "2025-01-15T00:00:00Z"
            reason: "Replaced by v2 API with improved performance."
            infoLink:
              summary: Migration guide
              url: https://docs.example.com/migrate
            successorLink:
              summary: Flights API v2
              url: https://api.example.com/v2/openapi.yaml
          sunset:
            date: "2025-07-01T00:00:00Z"
            policyLink:
              summary: API sunset policy
              url: https://docs.example.com/sunset-policy
```

#### Complete OpenAPI Description Example

This example is **informative**. It shows the Lifecycle SAF in a complete
OpenAPI 3.3 document. Besides the named links that map to HTTP signals, the
Deprecation and Sunset Objects each carry documentation-only
`externalLinks`, and the root and Operation Objects keep their own
`externalLinks` for general documentation.

```yaml
openapi: 3.3.0
features:
  security: 1.0.0
  lifecycle: 1.0.0
info:
  title: Flights API
  version: 2.0.0
externalLinks:
  - url: https://docs.example.com/flights
    summary: Flights API documentation
    kind: api-docs
paths:
  /v1/flights:
    get:
      operationId: listFlights_v1
      summary: List all flights
      deprecated: true
      externalLinks:
        - url: https://docs.example.com/flights/v1
          summary: Flights API v1 reference
          kind: api-docs
      featureFields:
        lifecycle:
          deprecation:
            date: "2025-01-15T00:00:00Z"
            reason: "Replaced by v2 API with improved performance."
            infoLink:
              summary: Migrating from Flights v1 to v2
              description: |
                Step-by-step guide covering renamed query parameters and the
                new cursor-based pagination.
              url: https://docs.example.com/migrate
            successorLink:
              summary: Flights API v2
              description: Use `listFlights_v2` (`GET /v2/flights`) instead.
              url: https://api.example.com/v2/openapi.yaml
            externalLinks:
              - url: https://docs.example.com/changelog#2025-01-15
                summary: Changelog entry announcing the v1 deprecation
                kind: changelog
          sunset:
            date: "2025-07-01T00:00:00Z"
            policyLink:
              summary: API sunset policy
              url: https://docs.example.com/sunset-policy
            externalLinks:
              - url: https://status.example.com/announcements/flights-v1-sunset
                summary: Sunset announcement
                description: |
                  Timeline, support contacts, and FAQ for the retirement of
                  `GET /v1/flights`.
                kind: announcement
      responses:
        "200":
          description: A list of flights.
  /v2/flights:
    get:
      operationId: listFlights_v2
      summary: List all flights
      externalLinks:
        - url: https://docs.example.com/flights/v2
          summary: Flights API v2 reference
          kind: api-docs
      responses:
        "200":
          description: A page of flights.
```

Applying the [Header Mapping](#header-mapping), a server would emit the
following headers in responses to `GET /v1/flights`:

```http
Deprecation: @1736899200
Sunset: Tue, 01 Jul 2025 00:00:00 GMT
Link: <https://docs.example.com/migrate>; rel="deprecation"; type="text/html"
Link: <https://api.example.com/v2/openapi.yaml>; rel="successor-version"
Link: <https://docs.example.com/sunset-policy>; rel="sunset"
```

Each header comes from one Lifecycle Object field:

| Header | Source in the Lifecycle Object |
| ---- | ---- |
| `Deprecation: @1736899200` | `deprecation.date: "2025-01-15T00:00:00Z"` |
| `Sunset: Tue, 01 Jul 2025 00:00:00 GMT` | `sunset.date: "2025-07-01T00:00:00Z"` |
| `Link: <…/migrate>; rel="deprecation"` | `deprecation.infoLink` |
| `Link: <…/v2/openapi.yaml>; rel="successor-version"` | `deprecation.successorLink` |
| `Link: <…/sunset-policy>; rel="sunset"` | `sunset.policyLink` |

The `externalLinks` in the Deprecation and Sunset Objects, and the Operation
Object's own `externalLinks`, do not generate headers. `GET /v2/flights` has
no Lifecycle Object, so its responses carry none of these headers.

### Deprecation Object

A **Deprecation Object** describes the deprecation of an operation, as
signaled at runtime by the `Deprecation` header and the `deprecation` and
`successor-version` link relations (see
[HTTP Deprecation and Sunset Signals](#http-deprecation-and-sunset-signals)).

#### Deprecation Object Fixed Fields

| Field Name | Type | Description |
| ---- | ---- | ---- |
| `date` | `string` (date-time) | **REQUIRED.** The date the deprecation takes or took effect. Maps to the `Deprecation` header. |
| `reason` | `string` | A human-readable explanation of why the operation is deprecated. |
| `infoLink` | [Link Object](#link-object) | Human-readable information about the deprecation, typically a migration guide. Maps to a `Link` header with the `deprecation` relation. |
| `successorLink` | [Link Object](#link-object) | The operation or API that replaces this one. Maps to a `Link` header with the `successor-version` relation. |
| `externalLinks` | [[External Documentation Object](https://spec.openapis.org/oas/latest.html#external-documentation-object)] | Other resources about the deprecation, such as a changelog entry. Documentation only; these do not generate headers. |

### Sunset Object

A **Sunset Object** describes when an operation is expected to stop accepting
requests entirely, as signaled at runtime by the `Sunset` header and the
`sunset` link relation (see
[HTTP Deprecation and Sunset Signals](#http-deprecation-and-sunset-signals)).

#### Sunset Object Fixed Fields

| Field Name | Type | Description |
| ---- | ---- | ---- |
| `date` | `string` (date-time) | **REQUIRED.** The date after which the operation is expected to stop accepting requests. Maps to the `Sunset` header. |
| `policyLink` | [Link Object](#link-object) | A human-readable sunset policy. Maps to a `Link` header with the `sunset` relation. |
| `externalLinks` | [[External Documentation Object](https://spec.openapis.org/oas/latest.html#external-documentation-object)] | Other resources about the sunset, such as an announcement. Documentation only; these do not generate headers. |

### Link Object

A **Link Object** describes a link that maps to a `Link` header. The link
relation is determined by the field that holds the Link Object, not by the
Link Object itself.

#### Link Object Fixed Fields

| Field Name | Type | Description |
| ---- | ---- | ---- |
| `url` | `string` | **REQUIRED.** The URI reference of the linked resource. |
| `summary` | `string` | A short summary of the linked resource. |
| `description` | `string` | A description of the linked resource. [CommonMark syntax](https://spec.commonmark.org/) MAY be used for rich text representation. |

This object MAY be extended with
[Specification Extensions](https://spec.openapis.org/oas/latest.html#specification-extensions).

## Specification

### Version `1.0`

This document is the Lifecycle Specification version 1.0.0.

### Notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC2119](https://tools.ietf.org/html/rfc2119).

### Schema

Implementations that validate Lifecycle Objects MUST use the schema published at
`https://spec.openapis.org/lifecycle/1.0/schema/WORK-IN-PROGRESS`.

The Lifecycle Object contains no Schema Objects, so validation does not depend
on the OAS dialect in effect in the containing document. The schema includes
its own definitions of the Link Object and of the External Documentation
Object, as extended by the
[Docs Links proposal](https://github.com/OAI/OpenAPI-Specification/blob/main/proposals/2026-08-01-Docs-Links.md).

### Header Mapping

This section is **informative**. The Lifecycle Object describes deprecation
and sunset *behavior*; the HTTP headers below are a byproduct a server MAY
choose to generate from that description at runtime. An OpenAPI description
alone does not cause a server to emit these headers — that requires runtime
support in the API implementation or gateway. Headers are only generated for
operations with `deprecated: true`; see
[Relationship to `deprecated`](#relationship-to-deprecated).

Tools that generate headers from a Lifecycle Object should use the
following mapping:

| Lifecycle Object field | Generated header | RFC |
| ---- | ---- | ---- |
| `deprecation.date` | `Deprecation: @<epoch-seconds>` | [RFC 9745 §2.1](https://www.rfc-editor.org/rfc/rfc9745#section-2.1), timestamp format per [RFC 9651 §3.3.7](https://www.rfc-editor.org/rfc/rfc9651#section-3.3.7) |
| `deprecation.infoLink.url` | `Link: <url>; rel="deprecation"; type="text/html"` | [RFC 9745 §3](https://www.rfc-editor.org/rfc/rfc9745#section-3) + [RFC 8288](https://www.rfc-editor.org/rfc/rfc8288) |
| `deprecation.successorLink.url` | `Link: <url>; rel="successor-version"` | [RFC 5829](https://www.rfc-editor.org/rfc/rfc5829) + [RFC 8288](https://www.rfc-editor.org/rfc/rfc8288) |
| `sunset.date` | `Sunset: <IMF-fixdate>` | [RFC 8594 §3](https://www.rfc-editor.org/rfc/rfc8594#section-3), date format per [RFC 9110 §5.6.7](https://www.rfc-editor.org/rfc/rfc9110#section-5.6.7) |
| `sunset.policyLink.url` | `Link: <url>; rel="sunset"` | [RFC 8594 §6](https://www.rfc-editor.org/rfc/rfc8594#section-6) + [RFC 8288](https://www.rfc-editor.org/rfc/rfc8288) |

`deprecation.reason`, the `summary` and `description` of every Link Object,
and every `externalLinks` entry have no header equivalent — they are
documentation-only, intended for doc-gen and SDK-gen tools (for example, as
the body of a generated `@deprecated` annotation).

Applied to the [example](#example) above, a compliant server would emit:

```http
Deprecation: @1736899200
Sunset: Tue, 01 Jul 2025 00:00:00 GMT
Link: <https://docs.example.com/migrate>; rel="deprecation"; type="text/html"
Link: <https://api.example.com/v2/openapi.yaml>; rel="successor-version"
Link: <https://docs.example.com/sunset-policy>; rel="sunset"
```

## Appendix A — Revision History

Prior to April 2026, drafts of this specification explored a root-level,
document-wide `lifecycle` object with `stage`/`stageDate` fields and
`deprecation`/`sunset` as independent siblings, intended to describe the
lifecycle of an entire API in one place. SIG discussion raised two problems
with that shape:

* A single OpenAPI document does not necessarily describe exactly one API, so
  there is no reliable document-wide "this API" for such an object to
  describe.
* A global object competes with [Overlays](https://spec.openapis.org/overlay/latest.html)
  as the SIG's established answer for applying the same metadata across many
  operations, without the flexibility of choosing which operations it
  applies to.

From April 2026 onward, the design shifted to the operation-scoped model in
this document: `deprecation` (renamed `Lifecycle Object`, placed in the
Operation Object's `featureFields`, and enabled through the root `features`
declaration per the [SAF](https://github.com/OAI/OpenAPI-Specification/pull/5447)
convention) is the primary object, with `sunset` nested inside it, and
`gracePeriod` was dropped as derivable from `deprecation.date` and
`sunset.date` together.

In October 2026, the separate `migration`, `replacement`, and `sunset.policy`
link fields were briefly replaced by a single `externalLinks` array keyed by
`kind`. In November 2026, following TDC review, the links were regrouped:
links that map to HTTP link relations became named fields on new Deprecation
and Sunset Objects, and other resources moved to documentation-only
`externalLinks` arrays alongside them. See
[sig-lifecycle discussion #5](https://github.com/OAI/sig-lifecycle/discussions/5)
for the full history.

## Appendix B — Usage with AI Agents and MCP Tooling

This section is **informative**.

Servers that expose OpenAPI operations as tools (for example, MCP servers)
can use the Lifecycle Object in two distinct phases:

1. **Tool registration (design-time, from the spec).** When registering an
   operation as a tool, a server can fold `deprecation.reason`, and the
   `deprecation.infoLink` and `deprecation.successorLink` links, into the
   tool's description — for example, prefixing
   it with `"⚠ DEPRECATED — sunset 2025-07-01. Use listFlights_v2 instead."` A
   well-behaved LLM client that reads this before deciding whether to invoke
   the tool should prefer the replacement from the outset, without ever
   calling the deprecated one.
2. **Tool invocation (runtime, from response headers).** If the tool is
   invoked anyway, a server acting as a proxy can inspect the `Deprecation`,
   `Sunset`, and `Link` response headers (see [Header Mapping](#header-mapping))
   and wrap the result with a structured warning naming the sunset deadline
   and the successor tool, giving the LLM what it needs to relay the warning
   or retry with the replacement.

Writing the `successorLink`'s `summary` and `description` clearly (in
language an LLM would recognize as migration- or deprecation-related) is what makes phase
1 effective — the mechanism is ordinary metadata, but the wording carries the
signal.
