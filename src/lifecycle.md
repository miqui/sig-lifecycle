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

**Scope.** The Lifecycle Object applies at the Operation Object level only. An
earlier draft explored a root-level, document-wide object (see
[Appendix A](#appendix-a--revision-history)), but SIG discussion concluded
that a single OpenAPI document does not necessarily describe exactly one API,
so a document-wide deprecation signal does not generalize; see
[sig-lifecycle discussion #5](https://github.com/OAI/sig-lifecycle/discussions/5).
Applying the same lifecycle metadata uniformly across many operations remains
the job of [Overlays](https://spec.openapis.org/overlay/latest.html), not of
this object.

**Non-goal.** This SAF does not replace `deprecated: true`, which
remains the coarse signal tooling already recognizes (for example, to gray out
an operation in Swagger UI). The Lifecycle Object is additional detail used
alongside it.

## Relationship to the OpenAPI Specification

The Lifecycle Specification is a companion specification defining a
[Standardized API Feature](https://spec.openapis.org/oas/v3.3.0.html#standardized-api-features)
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

Because the default value of `features` is `{"security": "1.0.0"}`, a
document that sets `features` explicitly MUST also list `security` to keep
the OAS 3.2-compatible security behavior. As required by the OAS, all
documents in a multi-document OpenAPI Description MUST declare the same SAFs.

When `lifecycle` is declared in `features`, the Operation Object gains the
`lifecycle` field defined in [Operation Object Fields](#operation-object-fields).
The Lifecycle Object is scoped to individual operations (see Introduction):

```yaml
paths:
  /flights:
    get:
      deprecated: true
      lifecycle:
        # Lifecycle Object fields here
```

Compatibility between versions of this specification and versions of the OpenAPI
Specification is determined by publication date. The rules for determining
compatible version pairs are documented in [TODO: link or section].

## Definitions

### Operation Object Fields

When a document declares `lifecycle` in its `features` field, the following
field is added to the [Operation Object](https://spec.openapis.org/oas/v3.3.0.html#operation-object):

| Field Name | Type | Description |
| ---- | ---- | ---- |
| `lifecycle` | [Lifecycle Object](#lifecycle-object) | Deprecation and sunset details for this operation. |

A `lifecycle` field on an Operation Object in a document that does not
declare `lifecycle` in `features` is not governed by this specification.

### Lifecycle Object

A **Lifecycle Object** describes the deprecation and sunset status of a
single operation: when the deprecation took effect, why, where to find a
migration guide and a replacement, and when the operation will stop accepting
requests. It is optional, and is meant to complement — not replace —
`deprecated: true` on the same Operation Object.

#### Lifecycle Object Fixed Fields

| Field Name | Type | Description |
| ---- | ---- | ---- |
| `date` | `string` (date-time) | **REQUIRED.** The date the deprecation takes or took effect. |
| `reason` | `string` | A human-readable explanation of why the operation is deprecated. |
| `migration` | [External Documentation Object](https://spec.openapis.org/oas/latest.html#external-documentation-object) | A description of, and link to, a migration guide. |
| `replacement` | [External Documentation Object](https://spec.openapis.org/oas/latest.html#external-documentation-object) | A description of, and link to, the operation or API that replaces this one. |
| `sunset` | [Sunset Object](#sunset-object) | Sunset details for this operation. |

`migration` and `replacement` reuse the OpenAPI Specification's existing
External Documentation Object rather than introducing new object types, since
both are fundamentally a described link — the same shape already used
throughout the OAS for this purpose.

#### Example

```yaml
paths:
  /v1/flights:
    get:
      operationId: listFlights_v1
      summary: List all flights
      deprecated: true
      lifecycle:
        date: "2025-01-15T00:00:00Z"
        reason: "Replaced by v2 API with improved performance."
        migration:
          description: Migration guide
          url: https://docs.example.com/migrate
        replacement:
          description: Flights API v2
          url: https://api.example.com/v2/openapi.yaml
        sunset:
          date: "2025-07-01T00:00:00Z"
          policy:
            url: https://docs.example.com/sunset-policy
```

#### Complete OpenAPI Description Example

This example is **informative**. It shows the Lifecycle SAF in a complete
OpenAPI 3.3 document, alongside the `externalLinks` field from the
[Docs Links proposal](https://github.com/OAI/OpenAPI-Specification/blob/main/proposals/2026-08-01-Docs-Links.md).
That proposal replaces the single `externalDocs` field with an
`externalLinks` array and adds optional `summary` and `kind` fields to the
External Documentation Object. Because `migration`, `replacement`, and
`sunset.policy` reuse the External Documentation Object, they gain `summary`
and `kind` without any change to this specification. The `kind` values shown
are illustrative.

The Lifecycle Object stays the structured source for deprecation and sunset
data. `externalLinks` holds other resources that help consumers through the
transition, such as a changelog or an announcement.

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
  - url: https://docs.example.com/changelog
    summary: Flights API changelog
    kind: changelog
paths:
  /v1/flights:
    get:
      operationId: listFlights_v1
      summary: List all flights
      deprecated: true
      externalLinks:
        - url: https://docs.example.com/changelog#2025-01-15
          summary: Changelog entry announcing the v1 deprecation
          kind: changelog
        - url: https://status.example.com/announcements/flights-v1-sunset
          summary: Sunset announcement
          description: |
            Timeline, support contacts, and FAQ for the retirement of
            `GET /v1/flights`.
          kind: announcement
      lifecycle:
        date: "2025-01-15T00:00:00Z"
        reason: "Replaced by v2 API with improved performance."
        migration:
          summary: Migrating from Flights v1 to v2
          description: |
            Step-by-step guide covering renamed query parameters and the
            new cursor-based pagination.
          url: https://docs.example.com/migrate
          kind: migration-guide
        replacement:
          summary: Flights API v2
          description: Use `listFlights_v2` (`GET /v2/flights`) instead.
          url: https://api.example.com/v2/openapi.yaml
          kind: openapi
        sunset:
          date: "2025-07-01T00:00:00Z"
          policy:
            summary: API sunset policy
            url: https://docs.example.com/sunset-policy
            kind: policy
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

Only Lifecycle Object fields participate in the
[Header Mapping](#header-mapping); entries in `externalLinks` do not generate
`Deprecation`, `Sunset`, or `Link` headers. A server would emit the same
headers for `GET /v1/flights` here as for the [example](#example) above.

### Sunset Object

A **Sunset Object** describes when an operation is expected to stop accepting
requests entirely.

#### Sunset Object Fixed Fields

| Field Name | Type | Description |
| ---- | ---- | ---- |
| `date` | `string` (date-time) | **REQUIRED.** The date after which the operation is expected to stop accepting requests. |
| `policy` | [External Documentation Object](https://spec.openapis.org/oas/latest.html#external-documentation-object) | A link to a human-readable sunset policy. |

### Schema Object

A **Schema Object** is a JSON Schema schema as defined by the OpenAPI Specification
in use. Within this document, Schema Objects behave according to the OAS dialect
in effect in the containing OpenAPI document.

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

When validating a Lifecycle Object within an OpenAPI document, the Schema Object
schema in effect is determined by the OAS dialect declared in that document.
Standalone validation of Lifecycle Objects uses the standard JSON Schema
2020-12 dialect for Schema Object positions.

### Header Mapping

This section is **informative**. The Lifecycle Object describes deprecation
and sunset *behavior*; the HTTP headers below are a byproduct a server MAY
choose to generate from that description at runtime. An OpenAPI description
alone does not cause a server to emit these headers — that requires runtime
support in the API implementation or gateway.

Tools that do generate headers from a Lifecycle Object SHOULD use the
following mapping:

| Lifecycle Object field | Generated header | RFC |
| ---- | ---- | ---- |
| `date` | `Deprecation: @<epoch-seconds>` | [RFC 9745 §2.1](https://www.rfc-editor.org/rfc/rfc9745#section-2.1), timestamp format per [RFC 9651 §3.3.7](https://www.rfc-editor.org/rfc/rfc9651#section-3.3.7) |
| `migration.url` | `Link: <url>; rel="deprecation"; type="text/html"` | [RFC 9745 §3.1](https://www.rfc-editor.org/rfc/rfc9745#section-3.1) + [RFC 8288](https://www.rfc-editor.org/rfc/rfc8288) |
| `replacement.url` | `Link: <url>; rel="successor-version"` | [RFC 8288](https://www.rfc-editor.org/rfc/rfc8288) |
| `sunset.date` | `Sunset: <IMF-fixdate>` | [RFC 8594 §3](https://www.rfc-editor.org/rfc/rfc8594#section-3), date format per [RFC 9110 §5.6.7](https://www.rfc-editor.org/rfc/rfc9110#section-5.6.7) |
| `sunset.policy.url` | `Link: <url>; rel="sunset"` | [RFC 8288](https://www.rfc-editor.org/rfc/rfc8288) |

`reason`, `migration.description`, and `replacement.description` have no
header equivalent — they are documentation-only fields, intended for
doc-gen and SDK-gen tools (for example, as the body of a generated
`@deprecated` annotation).

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
Operation Object's `lifecycle` field, and enabled through the root `features`
declaration per the [SAF](https://github.com/OAI/OpenAPI-Specification/pull/5447)
convention) is the primary object, with `sunset` nested inside it, and
`gracePeriod` was dropped as derivable from `deprecation.date` and
`sunset.date` together. See
[sig-lifecycle discussion #5](https://github.com/OAI/sig-lifecycle/discussions/5)
for the full history.

## Appendix B — Usage with AI Agents and MCP Tooling

This section is **informative**.

Servers that expose OpenAPI operations as tools (for example, MCP servers)
can use the Lifecycle Object in two distinct phases:

1. **Tool registration (design-time, from the spec).** When registering an
   operation as a tool, a server can fold `reason`, and the `migration` and
   `replacement` links, into the tool's description — for example, prefixing
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

Naming the `replacement` link and its `description` clearly (in language an
LLM would recognize as migration- or deprecation-related) is what makes phase
1 effective — the mechanism is ordinary metadata, but the wording carries the
signal.
