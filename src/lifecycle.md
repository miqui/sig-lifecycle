# Lifecycle Specification

## Version History

| Version | Date | Notes |
| ---- | ---- | ---- |
| 1.0.0 | TBD | Initial release |

## Introduction

TODO: Introduce the extension and what problem it solves.

## Relationship to the OpenAPI Specification

The Lifecycle Specification defines the **Lifecycle Object**, which appears
within an OpenAPI document under the `features` field introduced in OpenAPI 3.3:

```yaml
features:
  lifecycle:
    deprecation:
      date: "2025-01-15T00:00:00Z"
      reason: "Replaced by v2 API with improved performance."
      migration:
        description: Migration guide
        url: https://docs.example.com/migrate
      replacement:
        description: User API v2
        url: https://api.example.com/v2/openapi.yaml

    sunset:
        date: "2025-07-01T00:00:00Z"
        policy:
          url: https://docs.example.com/sunset-policy
 
```

Compatibility between versions of this specification and versions of the OpenAPI
Specification is determined by publication date. The rules for determining
compatible version pairs are documented in [TODO: link or section].

## Definitions

### Lifecycle Object

A **Lifecycle Object** is ...

#### Fixed Fields

| Field Name | Type | Description |
| ---- | ---- | ---- |
| deprecation | [Deprecation Object](#deprecation-object) | Indicates that this API or API element is deprecated. |
| sunset | [Sunset Object](#sunset-object) | Information about when and how the API or API element will be removed. |

The `deprecation` object is itself OPTIONAL.

### Deprecation Object

A **Deprecation Object** describes the deprecation of an API or API element,
including when and why it was deprecated and how consumers can migrate away
from it.

#### Fixed Fields

| Field Name | Type | Description |
| ---- | ---- | ---- |
| date | `string` | The date and time, as defined by RFC 3339 `date-time`, at which the API or API element was deprecated. |
| reason | `string` | A human-readable explanation of why the API or API element was deprecated. |
| migration | External Documentation Object | Information describing how to migrate away from the deprecated API or API element. |
| replacement | External Documentation Object | Information describing the replacement for the deprecated API or API element. |

### Sunset Object

A **Sunset Object** describes when a deprecated API or API element will be
removed and where to find the policy governing that removal.

#### Fixed Fields

| Field Name | Type | Description |
| ---- | ---- | ---- |
| date | `string` | The date and time, as defined by RFC 3339 `date-time`, at which the API or API element is planned to be removed. |
| policy | External Documentation Object | Information describing the policy governing the removal. |

### Schema Object

A **Schema Object** is a JSON Schema schema as defined by the OpenAPI Specification
in use. Within this document, Schema Objects behave according to the OAS dialect
in effect in the containing OpenAPI document.

### External Documentation Object

An **External Documentation Object** is as defined by the OpenAPI Specification
in use.

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

## Appendix A — Revision History

TODO: Document notable changes between versions.

## Appendix B — HTTP Header Mapping

This appendix is non-normative. It illustrates how an implementation MAY
translate the fields of the Lifecycle Object into HTTP response headers, and
how documentation or SDK generators MAY consume them.

### Example

Given the example Lifecycle Object above:

| Field | Value | Header produced | SDK gen |
| ---- | ---- | ---- | ---- |
| `deprecation.date` | `2025-01-15T00:00:00Z` | `Deprecation: @1736899200` | `@deprecated` annotation with date |
| `deprecation.reason` | `Replaced by v2 API with improved performance` | — (docs only) | Deprecation message in doc comment |
| `deprecation.migration.description` | `Migration guide` | — (docs only) | Doc comment label for migration link |
| `deprecation.migration.url` | `https://docs.example.com/migrate` | `Link: <https://docs.example.com/migrate>; rel="deprecation"; type="text/html"` | `@see` / doc comment link to migration guide |
| `deprecation.replacement.description` | `User API v2` | — (docs only) | Doc comment label for replacement link |
| `deprecation.replacement.url` | `https://api.example.com/v2/openapi.yaml` | `Link: <https://api.example.com/v2/openapi.yaml>; rel="successor-version"` | `@see` / doc comment link to replacement API |
| `sunset.date` | `2025-07-01T00:00:00Z` | `Sunset: Tue, 01 Jul 2025 00:00:00 GMT` | Hard removal deadline in doc comment and changelog generation |
| `sunset.policy.url` | `https://docs.example.com/sunset-policy` | `Link: <https://docs.example.com/sunset-policy>; rel="sunset"` | `@see` / doc comment link to sunset policy |

**Generated headers:**

```http
Deprecation: @1736899200
Sunset: Tue, 01 Jul 2025 00:00:00 GMT
Link: <https://docs.example.com/migrate>; rel="deprecation"; type="text/html"
Link: <https://api.example.com/v2/openapi.yaml>; rel="successor-version"
Link: <https://docs.example.com/sunset-policy>; rel="sunset"
```

> `Deprecation` — RFC 9745 §2.1 + RFC 9651 §3.3.7. `Sunset` — RFC 8594 §3, IMF-fixdate per RFC 9110 §5.6.7. `Link` rels — RFC 9745 §3.1 + RFC 8288.

### Full Field → Header Mapping Reference

| OAS field | HTTP header | RFC | Docs only | SDK gen |
| ---- | ---- | ---- | ---- | ---- |
| `deprecation.date` | `Deprecation: @<epoch>` | RFC 9745 + RFC 9651 §3.3.7 | | `@deprecated` annotation with date |
| `deprecation.reason` | — | — | ✓ | Deprecation message in doc comment |
| `deprecation.migration.url` | `Link: <url>; rel="deprecation"; type="text/html"` | RFC 9745 §3.1 + RFC 8288 | | `@see` / doc comment link to migration guide |
| `deprecation.migration.description` | — | — | ✓ | Doc comment label for migration link |
| `deprecation.replacement.url` | `Link: <url>; rel="successor-version"` | RFC 8288 | | `@see` / doc comment link to replacement API |
| `deprecation.replacement.description` | — | — | ✓ | Doc comment label for replacement link |
| `sunset.date` | `Sunset: <IMF-fixdate>` | RFC 8594 + RFC 9110 §5.6.7 | | Hard removal deadline in doc comment and changelog generation |
| `sunset.policy.url` | `Link: <url>; rel="sunset"` | RFC 8288 | | `@see` / doc comment link to sunset policy |
