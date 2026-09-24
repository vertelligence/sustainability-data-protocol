# A Proposal for a Shared Geo-Query Specification

**Spatial intersection queries for climate risk assessment**

Status: **Draft v0.2**
Published as part of the **Open Sustainability Data** initiative.

This document specifies GEO QUERY: an HTTP contract that lets a client ask *"what
regulatory, environmental, and organizational context surrounds this location?"* and get
back a structured, typed answer — without the client needing to know anything about how
the server stores or computes spatial data.

It is a draft. The wire format (Sections 5–7) is the part other implementers should treat
as load-bearing; some naming details may still carry the reference implementation's own
internal vocabulary.

---

## 1. Motivation

Climate risk assessment starts with a question every organization asks the same way:
*given a physical asset at a coordinate, what applies to it?* Which jurisdiction is it
in, which laws that jurisdiction has signed onto are in force, which hazard or risk
layers overlap it, and what else nearby might be relevant.

Every organization that tries to answer this ends up building the same three pieces:
a way to describe *where* an asset is, a way to ask *what's there*, and a way to get
back an answer a UI can render without bespoke parsing per data source. GEO QUERY
standardizes the middle and the last of those, and reuses an existing, universal
standard for the first.

## 2. Design principles

- **GeoJSON in.** Assets are described using unmodified [RFC 7946](https://datatracker.ietf.org/doc/html/rfc7946)
  `Feature` and `FeatureCollection` objects. No proprietary geometry format.
- **Typed, attributed results out.** A response is a flat list of typed resources, each
  carrying enough to render directly — a name, a description, the query point that found
  it, and which input asset it belongs to.
- **Read-only.** A GEO QUERY request never creates or mutates data. Submitted coordinates
  are used to query and are discarded; they are not a backdoor ingestion path.
- **Server-defined precision.** The spec does not mandate a spatial engine. A server MAY
  answer with exact geometric intersection (e.g. PostGIS) or with a coarser bounding-box
  test; it MUST document which, and MUST NOT claim precision it doesn't have.

## 3. Terminology

| Term | Meaning |
|---|---|
| **Asset** | A location a client wants context for — a facility, a site, anything with a coordinate. Submitted as a GeoJSON `Feature`. |
| **Resource** | Something a server knows about that can spatially intersect an asset — a jurisdiction, a hazard layer, a piece of legislation, another asset. Returned as an **assessment feature**. |
| **Server / provider** | An HTTP service implementing this spec. |
| **Client** | Whatever submits assets and consumes the response — a UI, a batch job, another service. |
| **Assessment feature** | One resource match in the response: a typed, attributed record tied back to the asset that found it. |

## 4. Protocol summary

```
Client                                  Server
  |--- POST /geo_queries -------------->|
  |     FeatureCollection of Assets     |  for each Asset:
  |                                     |    find intersecting Resources
  |<-- AssessmentCollection ------------|    (filtered, deduped, attributed)
```

One request MAY carry many assets; the response is the union of all their matches, each
one tagged with which asset produced it.

---

## 5. Request

### 5.1 Endpoint

```
POST <base-url>/geo_queries
```

The exact path is deployment-specific — mount it wherever fits your routing. For
interoperability, implementers are encouraged to expose it at a path ending in
`/geo_queries`. The reference implementation serves it at `POST /api/v1/geo_queries`.

### 5.2 Authentication

`Authorization: Bearer <token>`, over HTTPS. This spec does not mandate a token issuance
mechanism — the reference implementation uses one shared secret per deployment, the same
pattern it uses for its other query endpoints, checked with a constant-time comparison.
An unset server-side secret MUST reject every request rather than accept unauthenticated
ones.

### 5.3 Body

A single JSON object, `Content-Type: application/json`:

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | `"FeatureCollection"` | Yes | Per RFC 7946. |
| `features` | array of `Feature` | Yes | The assets to query. May be empty (returns an empty result, not an error). |
| `exclusions` | array of string | No, default `[]` | `type` values to leave out of the response (e.g., internal model name for incoming asset type), matched case-insensitively against the vocabulary in [§6.2](#62-assessment-feature). Unrecognized values are ignored rather than rejected, so this extends automatically to any custom `type` a server adds.|

Each `Feature`:

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | `"Feature"` | Yes | Per RFC 7946. |
| `id` | string | Recommended | The client's own identifier for this asset. Echoed back on every match as `matched_feature_id` — omit it and you lose the ability to tell which asset a result belongs to when submitting more than one. |
| `geometry` | `Point` geometry | Yes | `{"type": "Point", "coordinates": [lon, lat]}` or `[lon, lat, alt]`. **v0.1 supports `Point` only** — see [§10](#10-roadmap). |
| `properties` | object | No | Passed through by the client for its own use. The server does not read it. |

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "id": "asset-001",
      "properties": { "name": "San Diego Office" },
      "geometry": { "type": "Point", "coordinates": [-117.1611, 32.7157] }
    }
  ],
  "exclusions": []
}
```

---

## 6. Response

### 6.1 Envelope

```json
{
  "type": "AssessmentCollection",
  "features": [ "...assessment features, see 6.2" ],
  "errors": [ "...per-asset errors, see 6.3" ],
  "citation": "Vertelligence_20260923T201530Z"
}
```

`citation` identifies the data source and moment of the query — `"<source>_<UTC
timestamp>"`. It exists so a client can show provenance ("this assessment came from
Vertelligence's data as of ...") without hardcoding the source name, since a client may
query more than one GEO QUERY-compliant provider over time.

A request with zero matches still returns `200` with an empty `features` array — an
empty result is not an error.

### 6.2 Assessment feature

```json
{
  "type": "Jurisdiction",
  "public_id": "jur_8Xk2mQ1zR9pLxAe",
  "matched_feature_id": "asset-001",
  "geo_query": { "type": "Point", "coordinates": [-117.1611, 32.7157] },
  "properties": {
    "name": "San Diego County",
    "description": "San Diego County, California."
  }
}
```

| Field | Description |
|---|---|
| `type` | The resource's kind. See the vocabulary table below. |
| `public_id` | The resource's stable identifier on the server. Opaque — clients should not parse it. |
| `matched_feature_id` | The `id` of the *input* asset that produced this match. Required for correlating results back to assets when a request submits more than one. |
| `geo_query` | The asset's own query point, echoed back — **not** the resource's geometry. Two different assets intersecting the same resource produce two different assessment features, each with its own `geo_query`. |
| `properties` | Resource attributes. Shape depends on `type` — see below. |


### 6.3 Errors

Per-asset problems (e.g. unsupported geometry) do not fail the whole request — they're
collected instead, and every other valid asset in the same request is still processed:

```json
{ "feature_id": "asset-002", "errors": ["Feature geometry must be a GeoJSON Point with [x, y] coordinates"] }
```

### 6.4 Status codes

| Code | Meaning |
|---|---|
| `200` | Query executed. Check `errors` for any assets that couldn't be processed. |
| `401` | Missing or invalid bearer token. |
| `422` | Malformed request body (e.g. `features` is not an array). |

---

## 7. Matching semantics

These are the rules a compliant server's matches must satisfy, independent of how it
computes spatial intersection internally:

1. **Only active resources match, where activity is tracked.** If a resource type has a
   concept of being active/inactive, inactive resources are never returned. Resource
   types with no such concept are unaffected by this rule.
2. **A resource never matches itself.** If a result's `public_id` would equal the
   querying asset's own `id`, it is excluded.
3. **Results are deduplicated per asset.** The same resource is never returned twice for
   the same input asset, even if the server's internal representation would find it
   through multiple paths.

What these rules deliberately leave open is *how* "intersects" is computed. An intersection may include interacting with calculations and computed data or weightings, such as a hazzard assessment for flood risk, fire risk, etc.

**A server MUST document its own precision characteristics** so clients can
judge how much to trust a boundary-adjacent result.

---

## 8. Full worked example

**Request:**
```json
POST /api/v1/geo_queries
Authorization: Bearer <token>
Content-Type: application/json

{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "id": "asset-001",
      "properties": { "name": "San Diego Office" },
      "geometry": { "type": "Point", "coordinates": [-117.1611, 32.7157] }
    },
    {
      "type": "Feature",
      "id": "asset-002",
      "properties": { "name": "Phoenix Warehouse" },
      "geometry": { "type": "Point", "coordinates": [-112.074, 33.4484] }
    }
  ]
}
```

**Response `200`:**
```json
{
  "type": "AssessmentCollection",
  "features": [
    {
      "type": "Jurisdiction",
      "public_id": "jur_8Xk2mQ1zR9pLxAe",
      "matched_feature_id": "asset-001",
      "geo_query": { "type": "Point", "coordinates": [-117.1611, 32.7157] },
      "properties": { "name": "San Diego County", "description": "San Diego County, California." }
    },
    {
      "type": "Law",
      "public_id": "agr_3nF7wQeR2sTkLpZ",
      "matched_feature_id": "asset-001",
      "geo_query": { "type": "Point", "coordinates": [-117.1611, 32.7157] },
      "properties": {
        "name": "Climate Action Plan",
        "description": "San Diego County's adopted climate action plan.",
        "entry_into_force": "2021-01-01T00:00:00Z"
      }
    },
    {
      "type": "Layer",
      "public_id": "lay_daf3kjfasR7xPmQ",
      "matched_feature_id": "asset-001",
      "geo_query": { "type": "Point", "coordinates": [-117.1611, 32.7157] },
      "properties": { "name": "Biodiversity Hotspot", "description": "Coastal sage scrub habitat zone." }
    }
  ],
  "errors": [],
  "citation": "Vertelligence_20260923T201530Z"
}
```

`asset-002` (Phoenix) produced no matches in this example and contributes nothing to
`features` — absence, not a null entry, is how "no context found" is represented.

---


Implementers should treat the **envelope shape** (§6.1), the **assessment feature
shape** (§6.2 minus the exact `type` string set), and the **matching semantics** (§7) as
the load-bearing contract, and treat the demonstration server's exact type names as the part still most likely to be revised without changing the overall shape of a compliant response.

New resource `type` values may be added freely by any implementation for its own domain
— a compliant client should not assume the four listed in §6.2 are exhaustive, and
should render unrecognized types gracefully (at minimum, `name` + `description` are
always present).

## 10. Roadmap

Not yet part of the spec, but anticipated:

- **Non-`Point` geometry.** Polygon and LineString assets (e.g. a pipeline route, a
  facility boundary) instead of a single coordinate.
- **Computed resources.** A server returning a *derived* value for an asset — a computed
  risk score, an interpolated datapoint — rather than only resources that already exist
  in its store.
- **True-distance tolerance.** Replacing degree-based padding with a real distance unit
  (meters), independent of latitude.

## 11. Reference implementation

See https://github.com/vertelligence