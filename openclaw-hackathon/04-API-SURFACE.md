# 04 — API Surface

> The exact wire contract between cartorio-backend (the unique writer +
> canonical store) and every consumer (cartorio-web, lacre, sekiban,
> openclaw-scanner, cartorio-verify, third parties). cartorio-web
> compiles against this. New gates implementing this contract drop in
> without changes to cartorio.

## 0. Frame

Cartorio exposes one API on port 8082, three protocols (REST, GraphQL,
WebSocket) over the same handler set:

| Protocol | Path | Role |
|---|---|---|
| REST | `/api/v1/*` | classic mutating + listing surface; **the gating endpoint** `/by-digest/{digest}` lives here |
| GraphQL | `POST /graphql` | unified query + mutation surface; the cartorio-web crate's primary client |
| WebSocket | `WS /graphql` (subprotocol `graphql-transport-ws`) | live subscriptions for cartorio-web's animations |
| Health | `GET /health`, `GET /health/ready` | k8s liveness + readiness |
| Metrics | `GET /metrics` (port 9090) | Prometheus text format |

REST and GraphQL share the same data model; pick the one that fits the
caller. Mutations are available on both; subscriptions are GraphQL-only.

## 1. Auth + versioning

| Concern | M0 demo | Production (M1+) |
|---|---|---|
| Auth model | open | x-user-* headers from Hanabi BFF (lilitu pattern) |
| TLS | terminated at Ingress (selfsigned-issuer) | same; cracha SSO at the edge |
| Path versioning | `/api/v1/*` | bump to `/api/v2/*` on breaking changes |
| Deprecation | none | RFC 9745 `Sunset` + `Deprecation` headers |
| Content-Type | `application/json; charset=utf-8` everywhere except `/metrics` and WebSocket | same |

GraphQL schema is published at `GET /graphql/schema.graphql` (read-only).
Cartorio's CI lints for drift between the published schema and the
generated one.

## 2. Common types

Used across endpoints.

### 2.1 Hash strings (TypeScript-flavored type aliases for clarity)

```
type Blake3Hash    = string;  // 64-char lowercase hex (256 bits)
type ContentDigest = string;  // OCI-compatible: "sha256:<hex>" or "blake3:<hex>"
type Signature     = string;  // 128-char lowercase hex (Blake3 keyed-HMAC; 64 bytes)
```

### 2.2 Identifiers

```
type ListingId    = string;  // e.g. "operator@pleme.io/openclaw-publisher-pki@0.1.0"
type EventId      = string;  // UUID v7 (time-ordered)
type RunId        = string;  // UUID v7
type LedgerIndex  = number;  // i64 in JSON
type SignerId     = string;  // e.g. "operator@pleme.io" or "scanner@pleme.io"
```

### 2.3 Enums (snake_case in JSON; serde rename)

```
ArtifactKind ::=
  "skill"
  | "agent_config"
  | "agent_binary"
  | "agent_skills"
  | "agent_guardrails"
  | "agent_models"
  | "agent_runtime"
  | "agent_mcp_servers"
  | "agent_dependencies"
  | "agent_certificates"
  | "platform_connector"
  | "runtime_policy"
  | "helm_chart"
  | "oci_image"

ListingStateVariant  ::= "active" | "revoked" | "quarantined"

EventKind ::= "admit" | "revoke" | "quarantine" | "reactivate" | "reattest" | "supersede"

ComplianceStatus ::= "passed" | "passed_with_waiver" | "failed"

ComplianceFramework ::=
  "nist_800_53"
  | "fedramp"
  | "soc2"
  | "hipaa"
  | "pci_dss_v4"
  | "iso_27001"
  | "eu_ai_act"
  | "nist_ai_rmf"
  | "cis_kubernetes"
  | "slsa_build"

CrossReferenceType ::=
  "deploys"        // chart → image
  | "embeds"       // image → skill
  | "tests"        // ComplianceRun → tested artifact
  | "has_sbom"     // listing → its SBOM (implicit; the SBOM is embedded)
  | "depends_on"   // chart/listing → sibling listing
  | "publisher_of" // publisher_enrollment → its listings (or vice versa)

SignatureAlgorithm ::= "blake3" | "ed25519" | "akeyless_dfc"  // M0 emits "blake3"
```

### 2.4 Timestamps

ISO 8601 UTC with millisecond precision: `"2026-05-05T14:32:18.421Z"`.

### 2.5 Common nested objects

```typescript
type CertificationArtifact = {
  artifact_hash: Blake3Hash;
  control_hash:  Blake3Hash;
  intent_hash:   Blake3Hash;
  composed_root: Blake3Hash;   // = blake3_3leaf(artifact, control, intent)
};

type SignedRoot = {
  root:        Blake3Hash;
  signature:   Signature;
  algorithm:   SignatureAlgorithm;
  signer_id:   SignerId;
  signed_at:   string;          // ISO timestamp
};

type SigstoreBundle = {
  fulcio_cert_hash: Blake3Hash;
  rekor_entry_id:   string;     // M0 default: "offline-bundle-phase1"
  rekor_log_index:  number;     // M0 default: 0
  bundle_hash:      Blake3Hash;
};

type AttestedPublisher = {
  publisher_id:         SignerId;
  public_key_hex:       string;          // 64-char hex (32 bytes)
  enrollment_chain_hex: string[];        // 0..N intermediate certs
  enrolled_root:        Blake3Hash;      // org root
};

type ControlVerdict = {
  control_id:    string;          // "NIST.SI-7", "OWASP.LLM01", "MITRE.ATLAS.AML.T0019", "SLSA.L1"
  status:        ComplianceStatus;
  evidence_hash: Blake3Hash;
};

type ComplianceVerdict = {
  framework:     ComplianceFramework;
  summary_hash:  Blake3Hash;
  controls:      ControlVerdict[];     // non-empty (NonEmpty<T> on the Rust side)
};

type SbomComponent = {
  name:    string;
  version: string;
  hash:    Blake3Hash;
};

type Sbom = {
  components: SbomComponent[];   // sorted by (name, version) for determinism
  sbom_hash:  Blake3Hash;
};

type PillarEvidence = {
  baseline:             string;
  primitives_satisfied: number;
  attested_root:        string;   // base64 attestation seal
};

type LedgerInclusionProof = {
  leaf_hash:    Blake3Hash;
  merkle_path:  Blake3Hash[];     // bottom-up siblings
  global_head:  Blake3Hash;       // = ledger_root
  ledger_index: LedgerIndex;
};
```

### 2.6 ListingState envelope (the wire shape of `ListingState<K>`)

```typescript
type ListingState =
  | { variant: "active";       listing:    CompliantListing }
  | { variant: "revoked";      revocation: RevocationCertificate }
  | { variant: "quarantined";  quarantine: QuarantineEntry };

type CompliantListing = {
  kind:           ArtifactKind;
  listing_id:     ListingId;
  name:           string;
  version:        string;
  publisher:      AttestedPublisher;
  certification:  CertificationArtifact;
  signed_root:    SignedRoot;
  sigstore:       SigstoreBundle;
  ledger:         LedgerInclusionProof;
  verdicts:       ComplianceVerdict[];      // non-empty
  sbom:           Sbom;
  pillar_seal:    PillarEvidence;
  cross_refs_out: CrossReferenceEdge[];     // outbound (deploys/embeds/depends_on)
  cross_refs_in:  CrossReferenceEdge[];     // inbound (publisher_of, tests)
};

type RevocationCertificate = {
  listing_id: ListingId;
  reason:     string;
  revoked_by: SignerId;
  revoked_at: string;
  signature:  Signature;
};

type QuarantineEntry = {
  listing_id:    ListingId;
  reason:        string;
  detected_by:   SignerId;            // typically "scanner@pleme.io"
  detected_at:   string;
  drift_summary: {
    expected_artifact_hash: Blake3Hash;
    observed_artifact_hash: Blake3Hash;
    diff_bytes_sampled:     number;
  };
};

type CrossReferenceEdge = {
  id:                string;          // UUID v7
  source_listing_id: ListingId;
  target_listing_id: ListingId;
  reference_type:    CrossReferenceType;
  source_field_path: string;          // e.g. ".image" or ".values.image.repository"
  recorded_at:       string;
};
```

### 2.7 Event (a leaf in the event tree)

```typescript
type Event = {
  event_id:                  EventId;
  event_index:               LedgerIndex;
  listing_id:                ListingId;
  kind:                      EventKind;
  prior_state_hash:          Blake3Hash | null;     // null for Admit
  posterior_state_hash:      Blake3Hash;
  cosigners:                 SignerId[];            // who signed this transition
  event_data:                JsonObject;            // kind-specific payload
  leaf_hash:                 Blake3Hash;            // hash of canonical_json(this Event)
  created_at:                string;
  ledger_root_after:         Blake3Hash;            // root after this event was appended
};
```

### 2.8 ComplianceRun (a leaf, but ALSO a testing-artifact citizen of the state tree)

```typescript
type ComplianceRun = {
  run_id:                       RunId;
  tested_artifact_listing_id:   ListingId;
  pack_hash:                    Blake3Hash;       // identifies the provas pack version
  pack_name:                    string;           // "fedramp_high_openclaw_image"
  pack_version:                 string;           // "v2"
  run_hash:                     Blake3Hash;       // hash of canonical run output
  run_status:                   ComplianceStatus;
  executed_at:                  string;
  cosigners:                    SignerId[];       // typically [publisher, scanner]
  verdicts:                     ComplianceVerdict[];
};
```

## 3. Error envelope

Every 4xx / 5xx response carries:

```json
{
  "error": {
    "kind":                            "<ErrorKind>",
    "message":                         "<human-readable>",
    "details":                         { "...": "kind-specific structured fields" },
    "request_id":                      "01J1Q...",
    "ledger_root_at_time_of_error":    "abcd1234...",
    "trace_id":                        "<W3C trace-id, optional>"
  }
}
```

### 3.1 ErrorKind variants

```
AdmissionError::PublisherUnknown       → 401
AdmissionError::PublisherUnenrolled    → 401
AdmissionError::PublisherRevoked       → 401
AdmissionError::CertificationRoot      → 422  ◀── THE TAMPER GATE
AdmissionError::SignatureInvalid       → 422
AdmissionError::ComplianceMismatch     → 422
AdmissionError::InsufficientControls   → 422
AdmissionError::FailedVerdict          → 422
AdmissionError::ListingExists          → 409
AdmissionError::ListingNotFound        → 404
AdmissionError::InvalidStateTransition → 409   (e.g. revoke an already-Revoked listing)
NotFound                               → 404
BadRequest                             → 400
RateLimited                            → 429
Internal                               → 500
ServiceUnavailable                     → 503   (db down, tree state inconsistent)
```

### 3.2 Worked example — `AdmissionError::CertificationRoot` (the headline tamper case)

Request body has `certification.artifact_hash` mutated post-signing:

```http
POST /api/v1/artifacts HTTP/1.1
Content-Type: application/json

{
  "kind": "oci_image",
  "publisher": { ... },
  "certification": {
    "artifact_hash": "9999999999999999999999999999999999999999999999999999999999999999",
    "control_hash":  "bbbbbbbb...",
    "intent_hash":   "cccccccc...",
    "composed_root": "dddddddd..."   ← claimed root (signed)
  },
  "signed_root":   { "root": "dddddddd...", "signature": "1111...", ... },
  ...
}
```

Server recomputes `composed_root` from the (mutated) leaves: result is
`eeeeeeee...`, which ≠ `dddddddd...`. Response:

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json

{
  "error": {
    "kind":    "AdmissionError::CertificationRoot",
    "message": "composed_root recomputed from certification leaves does not match signed_root.root",
    "details": {
      "listing_id":           "operator@pleme.io/openclaw-publisher-pki@0.1.0",
      "expected_root":        "dddddddd...",
      "recomputed_root":      "eeeeeeee...",
      "offending_leaf":       "artifact_hash",
      "leaf_value_in_request": "99999999...",
      "leaves_recomputed_from_request": {
        "artifact_hash": "99999999...",
        "control_hash":  "bbbbbbbb...",
        "intent_hash":   "cccccccc..."
      }
    },
    "request_id":                   "01J1Q3K8RWS4...",
    "ledger_root_at_time_of_error": "abcd1234...",
    "trace_id":                     null
  }
}
```

The state tree is **untouched** — no leaf inserted, no event appended,
ledger root unchanged. Cartorio-web's tamper-overlay component
highlights `details.offending_leaf` in the cert tree visualization.

## 4. REST endpoints

### 4.1 Health + metrics

```
GET /health
GET /health/ready
GET /metrics       (port 9090, Prometheus text format)
```

`/health/ready` returns 200 only when:
- DB pool is healthy (`SELECT 1` succeeds)
- Tree state is consistent (latest ledger_snapshot's roots recompose to the cached tree state)

Otherwise 503 with details.

### 4.2 POST /api/v1/artifacts — admit (the 6-gate chain)

The substrate's only entry point for new attestation. Runs the 6 gates
from `01-FINDINGS.md` § 2.2. Single Postgres transaction. Atomic.

**Request:**

```json
{
  "kind": "oci_image",
  "name": "openclaw-publisher-pki",
  "version": "0.1.0",
  "publisher": {
    "publisher_id":         "operator@pleme.io",
    "public_key_hex":       "aaaa...",
    "enrollment_chain_hex": [],
    "enrolled_root":        "ffff..."
  },
  "certification": {
    "artifact_hash": "1111...",
    "control_hash":  "2222...",
    "intent_hash":   "3333...",
    "composed_root": "4444..."
  },
  "signed_root": {
    "root":      "4444...",
    "signature": "abcd1234...",
    "algorithm": "blake3",
    "signer_id": "operator@pleme.io",
    "signed_at": "2026-05-05T14:32:18.421Z"
  },
  "sigstore": {
    "fulcio_cert_hash": "5555...",
    "rekor_entry_id":   "offline-bundle-phase1",
    "rekor_log_index":  0,
    "bundle_hash":      "6666..."
  },
  "verdicts": [
    {
      "framework":    "fedramp",
      "summary_hash": "7777...",
      "controls": [
        { "control_id": "NIST.SI-7", "status": "passed", "evidence_hash": "8888..." },
        { "control_id": "NIST.AC-3", "status": "passed", "evidence_hash": "9999..." }
      ]
    },
    {
      "framework":    "slsa_build",
      "summary_hash": "aaaa...",
      "controls": [
        { "control_id": "SLSA.L1", "status": "passed", "evidence_hash": "bbbb..." }
      ]
    }
  ],
  "sbom": {
    "components": [
      { "name": "tokio",   "version": "1.40.0", "hash": "cccc..." },
      { "name": "axum",    "version": "0.8.1",  "hash": "dddd..." }
    ],
    "sbom_hash": "eeee..."
  },
  "pillar_seal": {
    "baseline":             "fedramp-high",
    "primitives_satisfied": 47,
    "attested_root":        "<base64 seal>"
  }
}
```

**Success — 200 OK:**

```json
{
  "listing_id":      "operator@pleme.io/openclaw-publisher-pki@0.1.0",
  "ledger_root":     "abcd1234...",
  "ledger_index":    42,
  "event_id":        "01J1Q3K8RWS4...",
  "inclusion_proof": {
    "leaf_hash":     "ffff...",
    "merkle_path":   ["....", "...."],
    "global_head":   "abcd1234...",
    "ledger_index":  42
  },
  "admitted_at":     "2026-05-05T14:32:18.523Z"
}
```

**Failure responses:** see § 3.

### 4.3 GET /api/v1/artifacts — paginated list

```
GET /api/v1/artifacts?limit=50&offset=0&kind=oci_image&state=active&publisher=operator@pleme.io
```

```json
{
  "data": [
    {
      "listing_id":   "operator@pleme.io/openclaw-publisher-pki@0.1.0",
      "kind":         "oci_image",
      "state":        "active",
      "name":         "openclaw-publisher-pki",
      "version":      "0.1.0",
      "publisher_id": "operator@pleme.io",
      "summary_hashes": {
        "artifact_hash": "1111...",
        "composed_root": "4444..."
      },
      "admitted_at":  "2026-05-05T14:32:18.523Z"
    },
    ...
  ],
  "total":                1234,
  "next_offset":          50,
  "ledger_root_at_query": "abcd..."
}
```

Query params:
- `limit` (default 50, max 200)
- `offset` (default 0)
- `kind` (filter by ArtifactKind; multi-value via repeating param)
- `state` (`active` | `revoked` | `quarantined`; multi-value)
- `publisher` (publisher_id)
- `since` (ISO timestamp)

### 4.4 GET /api/v1/artifacts/{listing_id}

Returns full `ListingState` envelope. URL-encode the listing_id (it
contains `@`, `/`, `:`).

```
GET /api/v1/artifacts/operator%40pleme.io%2Fopenclaw-publisher-pki%400.1.0
```

```json
{
  "variant": "active",
  "listing": {
    "kind":       "oci_image",
    "listing_id": "operator@pleme.io/openclaw-publisher-pki@0.1.0",
    "name":       "openclaw-publisher-pki",
    "version":    "0.1.0",
    "publisher":  { ... },
    "certification": { ... },
    "signed_root":   { ... },
    "sigstore":      { ... },
    "ledger":        { ... },
    "verdicts":      [ ... ],
    "sbom":          { ... },
    "pillar_seal":   { ... },
    "cross_refs_out": [
      {
        "id":                "01J1Q...",
        "source_listing_id": "operator@pleme.io/openclaw-publisher-pki@0.1.0",
        "target_listing_id": "operator@pleme.io/charts/openclaw-publisher-pki@0.1.0",
        "reference_type":    "deploys",
        "source_field_path": ".values.image.repository",
        "recorded_at":       "2026-05-05T14:32:18.523Z"
      }
    ],
    "cross_refs_in": [
      {
        "id":                "01J1R...",
        "source_listing_id": "fedramp_high_openclaw_image@2/operator@pleme.io/openclaw-publisher-pki@0.1.0",
        "target_listing_id": "operator@pleme.io/openclaw-publisher-pki@0.1.0",
        "reference_type":    "tests",
        "source_field_path": ".tested_artifact_hash",
        "recorded_at":       "2026-05-05T14:32:19.001Z"
      }
    ]
  }
}
```

`variant: "revoked"` and `variant: "quarantined"` swap in `revocation`
or `quarantine` instead of `listing` per § 2.6.

### 4.5 GET /api/v1/artifacts/{id}/verify — re-verify server-side

Recomputes everything from stored leaves and emits the verification trail:

```json
{
  "listing_id": "operator@pleme.io/openclaw-publisher-pki@0.1.0",
  "verified":   true,
  "checks": [
    { "check": "publisher_enrollment", "passed": true,  "details": { "enrolled_root": "ffff..." } },
    { "check": "certification_root",    "passed": true,  "details": { "recomputed": "4444..." } },
    { "check": "signature",             "passed": true,  "details": { "algorithm": "blake3" } },
    { "check": "compliance_packs",      "passed": true,  "details": { "pack_hashes_verified": 2 } },
    { "check": "framework_coverage",    "passed": true,  "details": { "frameworks": ["fedramp", "slsa_build"] } },
    { "check": "ledger_inclusion",      "passed": true,  "details": { "ledger_root": "abcd..." } }
  ],
  "ledger_root_at_verify": "abcd..."
}
```

### 4.6 GET /api/v1/artifacts/{id}/history

```json
{
  "listing_id": "operator@pleme.io/openclaw-publisher-pki@0.1.0",
  "events": [
    { ...Event with kind="admit"... },
    { ...Event with kind="reattest"... },
    { ...Event with kind="reattest"... }
  ],
  "total":      8,
  "next_offset": null
}
```

### 4.7 GET /api/v1/artifacts/{id}/compliance-runs

```json
{
  "listing_id": "operator@pleme.io/openclaw-publisher-pki@0.1.0",
  "runs": [
    { ...ComplianceRun fedramp_high_openclaw_image@2... },
    { ...ComplianceRun fedramp_high_openclaw_helm_content@1... }
  ],
  "total": 2
}
```

### 4.8 Lifecycle mutations — POST /api/v1/artifacts/{id}/{action}

Five mutating endpoints, all share the shape: payload contains the
required cosignatures, response is the new `Event` plus updated
`ListingState`.

```
POST /api/v1/artifacts/{id}/revoke
POST /api/v1/artifacts/{id}/quarantine
POST /api/v1/artifacts/{id}/reactivate
POST /api/v1/artifacts/{id}/reattest
POST /api/v1/artifacts/{id}/supersede
```

Example — revoke:

**Request:**

```json
{
  "reason":     "compromised signing key — immediate revocation",
  "revoked_by": "operator@pleme.io",
  "revoked_at": "2026-05-05T15:00:00.000Z",
  "signature":  "<128-char hex over canonical_json(this body minus signature)>"
}
```

**Response:**

```json
{
  "event":         { ...Event with kind="revoke"... },
  "listing_state": { "variant": "revoked", "revocation": { ... } },
  "ledger_root":   "bcde...",
  "ledger_index":  43
}
```

Cosigner requirements per action:

| Action | Required cosigners |
|---|---|
| `revoke` | publisher (or PKI on revocation cert) |
| `quarantine` | scanner |
| `reactivate` | operator + scanner (cosigned) |
| `reattest` | scanner |
| `supersede` | publisher + new listing's publisher (cosigned) |

### 4.9 GET /api/v1/artifacts/by-digest/{digest}

**The most-called endpoint.** lacre + sekiban consult this on every
push / Pod-create. Index-backed; <1ms p99 expected.

```
GET /api/v1/artifacts/by-digest/sha256:DEADBEEFDEADBEEFDEADBEEFDEADBEEFDEADBEEFDEADBEEFDEADBEEFDEADBEEF
```

**Found, Active:**

```json
{
  "found":              true,
  "listing_id":         "operator@pleme.io/openclaw-publisher-pki@0.1.0",
  "state":              "active",
  "as_of_ledger_root":  "abcd..."
}
```

**Found, but not Active (revoked / quarantined):**

```json
{
  "found":              true,
  "listing_id":         "operator@pleme.io/openclaw-publisher-pki@0.1.0",
  "state":              "quarantined",
  "as_of_ledger_root":  "bcde..."
}
```

**Not found:**

```json
{
  "found":              false,
  "as_of_ledger_root":  "abcd..."
}
```

lacre allows the push iff `found && state == "active"`. sekiban
allows the Pod create iff the same.

### 4.10 GET /api/v1/events + /api/v1/events/{event_id}

**List** (paginated):

```
GET /api/v1/events?limit=50&offset=0&listing_id=...&kind=admit
```

```json
{
  "data":  [ { ...Event... }, ... ],
  "total": 234,
  "next_offset": 50,
  "ledger_root_at_query": "abcd..."
}
```

**Get one:**

```
GET /api/v1/events/01J1Q3K8RWS4...
```

Returns the full `Event` object.

### 4.11 GET /api/v1/artifacts/{id}/proof

Inclusion proof for the listing's leaf in the state tree.

```json
{
  "leaf_hash":           "ffff...",
  "merkle_path":         ["aaaa...", "bbbb...", "cccc..."],
  "state_root":          "1234...",
  "ledger_root_at_proof": "abcd...",
  "ledger_index":         42,
  "verifier_logic":       "blake3(domain_separator_internal || left || right) at each step; leaf domain is domain_separator_state_leaf"
}
```

cartorio-verify uses this to check inclusion locally without trusting
cartorio.

### 4.12 GET /api/v1/events/{event_id}/proof

Same shape, but for the event tree.

### 4.13 GET /api/v1/merkle/root

```json
{
  "state_root":   "1234...",
  "event_root":   "5678...",
  "ledger_root":  "abcd...",
  "ledger_index": 1234,
  "signed_at":    "2026-05-05T14:32:18.421Z",
  "signed_by":    "cartorio@pleme.io",
  "tree_sizes": {
    "active_listings":      6,
    "revoked_listings":     0,
    "quarantined_listings": 0,
    "events_total":         234
  }
}
```

### 4.14 GET /api/v1/merkle/consistency — NEW M0

Range proof that ledger root @to extends ledger root @from. The
standard append-only-log invariant; without this, a malicious cartorio
could rewrite history between snapshots.

```
GET /api/v1/merkle/consistency?from=abcd1234...&to=ef567890...
```

```json
{
  "from_root":          "abcd1234...",
  "from_ledger_index":  100,
  "to_root":            "ef567890...",
  "to_ledger_index":    234,
  "consistency_proof":  ["...", "...", "..."],
  "verifier_logic":     "RFC 6962-style; blake3 over domain-separated nodes; see cartorio/merkle/consistency.rs",
  "verified_by_cartorio": true
}
```

cartorio-verify recomputes the proof locally against `from_root` →
`to_root`. If the proof doesn't reduce to the expected root, the
verifier emits `✗ break-at-event-id <id>`.

### 4.15 GET /api/v1/artifacts/{id}/tree-fragment — NEW M0

Returns the leaf, its parents up to root, and each parent's siblings
— ready for d3-tree-style render. cartorio-web's `tree_view.rs`
component compiles directly against this shape.

```
GET /api/v1/artifacts/operator%40pleme.io%2Fopenclaw-publisher-pki%400.1.0/tree-fragment
```

```json
{
  "listing_id": "operator@pleme.io/openclaw-publisher-pki@0.1.0",
  "leaf": {
    "leaf_hash": "ffff...",
    "type":      "state_leaf",
    "depth":     0
  },
  "path": [
    {
      "depth":          0,
      "node_hash":      "ffff...",
      "is_leaf":        true,
      "left_child":     null,
      "right_child":    null,
      "sibling_hash":   "1212..."
    },
    {
      "depth":          1,
      "node_hash":      "3434...",
      "is_leaf":        false,
      "left_child":     "ffff...",
      "right_child":    "1212...",
      "sibling_hash":   "5656..."
    },
    {
      "depth":          2,
      "node_hash":      "7878...",
      "is_leaf":        false,
      "left_child":     "3434...",
      "right_child":    "5656...",
      "sibling_hash":   null
    }
  ],
  "state_root":  "9abc...",
  "ledger_root": "abcd..."
}
```

### 4.16 GET /api/v1/cross-references/source/{listing_id} — NEW M0

Outbound edges. cartorio-web's `cross_ref_graph.rs` component renders
these.

```json
{
  "listing_id": "operator@pleme.io/charts/openclaw-skill-store@0.1.0",
  "edges_out": [
    {
      "id":                "01J1Q...",
      "source_listing_id": "operator@pleme.io/charts/openclaw-skill-store@0.1.0",
      "target_listing_id": "operator@pleme.io/openclaw-skill-store@0.1.0",
      "reference_type":    "deploys",
      "source_field_path": ".values.image.repository",
      "recorded_at":       "2026-05-05T14:32:18.523Z"
    },
    {
      "id":                "01J1R...",
      "source_listing_id": "operator@pleme.io/charts/openclaw-skill-store@0.1.0",
      "target_listing_id": "operator@pleme.io/charts/openclaw-publisher-pki@0.1.0",
      "reference_type":    "depends_on",
      "source_field_path": ".values.upstream.pki",
      "recorded_at":       "2026-05-05T14:32:18.523Z"
    }
  ],
  "ledger_root_at_query": "abcd..."
}
```

### 4.17 GET /api/v1/cross-references/target/{listing_id} — NEW M0

Inbound edges. Same shape, `edges_in` instead of `edges_out`.

## 5. GraphQL schema

Full SDL. Ships at `cartorio/services/rust/backend/src/api/graphql/`.

```graphql
schema {
  query:        Query
  mutation:     Mutation
  subscription: Subscription
}

scalar Blake3Hash
scalar ContentDigest
scalar DateTime
scalar JSON

# ─── Query root ─────────────────────────────────────────────────────

type Query {
  "Look up a listing by id."
  listing(id: ID!): ListingState

  "Paginated listing search."
  listings(filter: ListingFilter, page: PageInput): ListingPage!

  "Look up an event by id."
  event(id: ID!): Event

  "Paginated event log."
  events(filter: EventFilter, page: PageInput): EventPage!

  "The current merkle root triple + signed snapshot metadata."
  merkleRoot: MerkleRoot!

  "Range proof that toRoot extends fromRoot. Verified server-side; client may re-verify."
  consistencyProof(fromRoot: Blake3Hash!, toRoot: Blake3Hash!): ConsistencyProof!

  "Inclusion proof for a listing's leaf in the state tree."
  inclusionProof(listingId: ID!): InclusionProof!

  "Viz-friendly tree-fragment (parents + siblings) for d3-style render."
  treeFragment(listingId: ID!): TreeFragment!

  "Outbound cross-reference edges from a listing."
  crossReferencesOut(listingId: ID!): [CrossReferenceEdge!]!

  "Inbound cross-reference edges into a listing."
  crossReferencesIn(listingId: ID!): [CrossReferenceEdge!]!

  "A single ComplianceRun by id."
  complianceRun(id: ID!): ComplianceRun

  "Compliance runs that tested a given listing."
  complianceRunsForListing(listingId: ID!): [ComplianceRun!]!

  "Lookup by content digest. The hot-path endpoint lacre + sekiban use."
  artifactByDigest(digest: ContentDigest!): DigestLookup!
}

# ─── Mutation root ──────────────────────────────────────────────────

type Mutation {
  admit(input: AdmitArtifactInput!):                                                 AdmitResult!
  revoke(listingId: ID!, certificate: RevocationCertificateInput!):                  MutationResult!
  quarantine(listingId: ID!, entry: QuarantineEntryInput!):                          MutationResult!
  reactivate(listingId: ID!, cosignatures: [CosignInput!]!):                         MutationResult!
  reattest(listingId: ID!, scannerSignature: SignatureInput!):                       MutationResult!
  supersede(listingId: ID!, successor: SuccessorInput!):                             MutationResult!
}

# ─── Subscription root ─────────────────────────────────────────────

type Subscription {
  "Emits whenever a new event causes the ledger root to advance."
  ledgerRootChanged: MerkleRoot!

  "Emits each new event matching filter."
  eventAdded(filter: EventFilter): Event!

  "Emits when a listing's state transitions (admit/revoke/quarantine/reactivate)."
  listingStateChanged(listingId: ID): ListingState!
}

# ─── Object types ──────────────────────────────────────────────────

union ListingState = ActiveListing | RevokedListing | QuarantinedListing

type ActiveListing {
  variant:    String!  # always "active"
  listing:    CompliantListing!
}

type RevokedListing {
  variant:    String!  # "revoked"
  revocation: RevocationCertificate!
}

type QuarantinedListing {
  variant:    String!  # "quarantined"
  quarantine: QuarantineEntry!
}

type CompliantListing {
  kind:           ArtifactKind!
  listingId:      ID!
  name:           String!
  version:        String!
  publisher:      AttestedPublisher!
  certification:  CertificationArtifact!
  signedRoot:     SignedRoot!
  sigstore:       SigstoreBundle!
  ledger:         LedgerInclusionProof!
  verdicts:       [ComplianceVerdict!]!         # non-empty
  sbom:           Sbom!
  pillarSeal:     PillarEvidence!
  crossRefsOut:   [CrossReferenceEdge!]!
  crossRefsIn:    [CrossReferenceEdge!]!
}

type CertificationArtifact {
  artifactHash: Blake3Hash!
  controlHash:  Blake3Hash!
  intentHash:   Blake3Hash!
  composedRoot: Blake3Hash!
}

type SignedRoot {
  root:      Blake3Hash!
  signature: String!         # 128-char hex
  algorithm: SignatureAlgorithm!
  signerId:  String!
  signedAt:  DateTime!
}

enum SignatureAlgorithm { BLAKE3 ED25519 AKEYLESS_DFC }

type SigstoreBundle {
  fulcioCertHash: Blake3Hash!
  rekorEntryId:   String!
  rekorLogIndex:  Int!
  bundleHash:     Blake3Hash!
}

type AttestedPublisher {
  publisherId:        String!
  publicKeyHex:       String!
  enrollmentChainHex: [String!]!
  enrolledRoot:       Blake3Hash!
}

type ControlVerdict {
  controlId:    String!
  status:       ComplianceStatus!
  evidenceHash: Blake3Hash!
}

enum ComplianceStatus { PASSED PASSED_WITH_WAIVER FAILED }

type ComplianceVerdict {
  framework:   ComplianceFramework!
  summaryHash: Blake3Hash!
  controls:    [ControlVerdict!]!         # non-empty
}

enum ComplianceFramework {
  NIST_800_53   FEDRAMP   SOC2   HIPAA   PCI_DSS_V4
  ISO_27001     EU_AI_ACT NIST_AI_RMF    CIS_KUBERNETES   SLSA_BUILD
}

type SbomComponent {
  name:    String!
  version: String!
  hash:    Blake3Hash!
}

type Sbom {
  components: [SbomComponent!]!
  sbomHash:   Blake3Hash!
}

type PillarEvidence {
  baseline:             String!
  primitivesSatisfied:  Int!
  attestedRoot:         String!
}

type LedgerInclusionProof {
  leafHash:    Blake3Hash!
  merklePath:  [Blake3Hash!]!
  globalHead:  Blake3Hash!
  ledgerIndex: Int!
}

type RevocationCertificate {
  listingId: ID!
  reason:    String!
  revokedBy: String!
  revokedAt: DateTime!
  signature: String!
}

type QuarantineEntry {
  listingId:    ID!
  reason:       String!
  detectedBy:   String!
  detectedAt:   DateTime!
  driftSummary: DriftSummary!
}

type DriftSummary {
  expectedArtifactHash: Blake3Hash!
  observedArtifactHash: Blake3Hash!
  diffBytesSampled:     Int!
}

type CrossReferenceEdge {
  id:                String!
  sourceListingId:   ID!
  targetListingId:   ID!
  referenceType:     CrossReferenceType!
  sourceFieldPath:   String!
  recordedAt:        DateTime!
}

enum CrossReferenceType { DEPLOYS EMBEDS TESTS HAS_SBOM DEPENDS_ON PUBLISHER_OF }

type Event {
  eventId:             String!
  eventIndex:          Int!
  listingId:           ID!
  kind:                EventKind!
  priorStateHash:      Blake3Hash
  posteriorStateHash:  Blake3Hash!
  cosigners:           [String!]!
  eventData:           JSON!
  leafHash:            Blake3Hash!
  createdAt:           DateTime!
  ledgerRootAfter:     Blake3Hash!
}

enum EventKind { ADMIT REVOKE QUARANTINE REACTIVATE REATTEST SUPERSEDE }

enum ArtifactKind {
  SKILL  AGENT_CONFIG  AGENT_BINARY  AGENT_SKILLS  AGENT_GUARDRAILS
  AGENT_MODELS  AGENT_RUNTIME  AGENT_MCP_SERVERS  AGENT_DEPENDENCIES
  AGENT_CERTIFICATES  PLATFORM_CONNECTOR  RUNTIME_POLICY
  HELM_CHART  OCI_IMAGE
}

type ComplianceRun {
  runId:                    String!
  testedArtifactListingId:  ID!
  packHash:                 Blake3Hash!
  packName:                 String!
  packVersion:              String!
  runHash:                  Blake3Hash!
  runStatus:                ComplianceStatus!
  executedAt:               DateTime!
  cosigners:                [String!]!
  verdicts:                 [ComplianceVerdict!]!
}

type MerkleRoot {
  stateRoot:   Blake3Hash!
  eventRoot:   Blake3Hash!
  ledgerRoot:  Blake3Hash!
  ledgerIndex: Int!
  signedAt:    DateTime!
  signedBy:    String!
  treeSizes:   TreeSizes!
}

type TreeSizes {
  activeListings:      Int!
  revokedListings:     Int!
  quarantinedListings: Int!
  eventsTotal:         Int!
}

type ConsistencyProof {
  fromRoot:           Blake3Hash!
  fromLedgerIndex:    Int!
  toRoot:             Blake3Hash!
  toLedgerIndex:      Int!
  consistencyProof:   [Blake3Hash!]!
  verifiedByCartorio: Boolean!
}

type InclusionProof {
  leafHash:           Blake3Hash!
  merklePath:         [Blake3Hash!]!
  stateRoot:          Blake3Hash!
  ledgerRootAtProof:  Blake3Hash!
  ledgerIndex:        Int!
}

type TreeFragment {
  listingId:  ID!
  leaf:       TreeNode!
  path:       [TreeNode!]!
  stateRoot:  Blake3Hash!
  ledgerRoot: Blake3Hash!
}

type TreeNode {
  depth:        Int!
  nodeHash:     Blake3Hash!
  isLeaf:       Boolean!
  leftChild:    Blake3Hash
  rightChild:   Blake3Hash
  siblingHash:  Blake3Hash
}

type DigestLookup {
  found:           Boolean!
  listingId:       ID
  state:           String      # "active" | "revoked" | "quarantined"
  asOfLedgerRoot:  Blake3Hash!
}

# ─── Pagination ────────────────────────────────────────────────────

input PageInput {
  limit:  Int = 50
  offset: Int = 0
  desc:   Boolean = false
}

type ListingPage {
  data:               [ListingState!]!
  total:              Int!
  nextOffset:         Int
  ledgerRootAtQuery:  Blake3Hash!
}

type EventPage {
  data:               [Event!]!
  total:              Int!
  nextOffset:         Int
  ledgerRootAtQuery:  Blake3Hash!
}

input ListingFilter {
  kind:        [ArtifactKind!]
  state:       [String!]
  publisherId: String
  since:       DateTime
}

input EventFilter {
  listingId:  ID
  kind:       [EventKind!]
  fromIndex:  Int
  toIndex:    Int
  since:      DateTime
}

# ─── Mutation inputs ───────────────────────────────────────────────

input AdmitArtifactInput {
  kind:          ArtifactKind!
  name:          String!
  version:       String!
  publisher:     AttestedPublisherInput!
  certification: CertificationArtifactInput!
  signedRoot:    SignedRootInput!
  sigstore:      SigstoreBundleInput!
  verdicts:      [ComplianceVerdictInput!]!
  sbom:          SbomInput!
  pillarSeal:    PillarEvidenceInput!
}

# (the *Input shapes mirror the output object types 1:1; omitted for
#  brevity. cargo will generate them from #[derive(InputObject)] on
#  serde-mirror'd structs.)

type AdmitResult {
  listingId:       ID!
  ledgerRoot:      Blake3Hash!
  ledgerIndex:     Int!
  eventId:         String!
  inclusionProof:  InclusionProof!
  admittedAt:      DateTime!
}

type MutationResult {
  event:        Event!
  listingState: ListingState!
  ledgerRoot:   Blake3Hash!
  ledgerIndex:  Int!
}
```

## 6. WebSocket protocol

Subprotocol: `graphql-transport-ws` (https://github.com/enisdenjo/graphql-ws/blob/master/PROTOCOL.md). Endpoint: `WS /graphql` (same path as POST mutations; protocol negotiated via `Sec-WebSocket-Protocol`).

### 6.1 Lifecycle

```
1. Client → Server: ConnectionInit
   {"type": "connection_init", "payload": {}}

2. Server → Client: ConnectionAck
   {"type": "connection_ack"}

3. Client → Server: Subscribe
   {"type": "subscribe", "id": "<op_id>", "payload": {
     "operationName": "LedgerRootChanged",
     "query": "subscription LedgerRootChanged { ledgerRootChanged { ledgerRoot ledgerIndex signedAt } }",
     "variables": {}
   }}

4. Server → Client: Next (repeat per emission)
   {"type": "next", "id": "<op_id>", "payload": { "data": { "ledgerRootChanged": { ... } } }}

5. Client → Server: Complete (to end one subscription)
   {"type": "complete", "id": "<op_id>"}

6. Either side: ConnectionClose (graceful shutdown)
```

### 6.2 cartorio-web's three subscriptions

Driven by the dashboard:

| Subscription | Frequency | Drives |
|---|---|---|
| `ledgerRootChanged` | every event | `ledger_banner.rs` updates |
| `eventAdded(filter)` | every matching event | `event_timeline.rs` appends |
| `listingStateChanged(listingId)` | on state transition | per-artifact view refresh |

### 6.3 Reconnection

If the WS drops, the client reconnects and sends:

```json
{
  "type": "subscribe",
  "id": "<op_id>",
  "payload": {
    "operationName": "EventAdded",
    "query": "subscription EventAdded($lastIndex: Int) { eventAdded(filter: {fromIndex: $lastIndex}) { ... } }",
    "variables": { "lastIndex": 234 }
  }
}
```

The server replays events from `fromIndex` so cartorio-web doesn't miss
any while disconnected. If `fromIndex` is older than the current event
log's window (M2+ may add archival), the server returns
`error: replay_window_exceeded` and the client falls back to a fresh
query of the page-paginated `events` list.

## 7. Pagination model

REST: `?limit=N&offset=N` with default 50, max 200. Response carries
`total` (i64) and `next_offset` (number or null when exhausted).

GraphQL: `PageInput { limit: Int = 50, offset: Int = 0, desc: Boolean = false }` and matching `XPage { data, total, nextOffset, ledgerRootAtQuery }`.

Stable ordering:
- Listings: `updated_at DESC` (default) or `updated_at ASC` with `desc: false`.
- Events: `event_index ASC` (default; chronological).
- Compliance runs: `executed_at DESC`.

`ledgerRootAtQuery` is the `ledger_root` snapshot at the moment of
the query. Pinning to it lets a verifier re-issue the query and confirm
the same view.

## 8. Determinism + ordering guarantees

- **Hashing**: blake3 throughout, with three domain separators
  (`state_leaf`, `event_leaf`, `internal`). Domains are disjoint per
  Kani harness `hash_proofs.rs::leaf_internal_domains_disjoint`.
- **Canonical JSON**: serialization for hashing uses
  [RFC 8785 JSON Canonicalization Scheme] (sorted keys, no
  whitespace, normalized floats). Implementations must match
  byte-for-byte.
- **UTC + millisecond precision**: timestamps in canonical JSON
  serialize as `YYYY-MM-DDTHH:MM:SS.sssZ`.
- **Listing IDs** are stable strings. Two admits with the same
  `listing_id` collide at gate 6 unless the prior is in
  `Revoked`/`Quarantined` state.
- **Event IDs** are UUID v7 (time-ordered); event_index is monotonic.
- **WS event order** matches event tree order. Clients can pin to
  `event_index` for replay.

## 9. Worked example flows

### 9.1 Admit a single openclaw OciImage

```bash
curl -k -XPOST https://openclaw.dev.use1.quero.lol/api/v1/artifacts \
  -H 'content-type: application/json' \
  -d @- <<'EOF'
{
  "kind": "oci_image",
  "name": "openclaw-publisher-pki",
  "version": "0.1.0",
  ...
}
EOF
```

Response 200, ledger_root advances, inclusion_proof returned. Browser
dashboard's `ledgerRootChanged` subscription fires; new event animates
onto timeline; new artifact card appears.

### 9.2 Lacre's gate consultation on a registry push

```bash
# inside lacre's reverse-proxy pipeline:
curl -fsS http://cartorio-backend.pleme-attestation:8082/api/v1/artifacts/by-digest/sha256:DEADBEEF...
# {"found": true, "state": "active", ...} → forward push
# {"found": false, ...} or {"state": "quarantined"} → 403 to client
```

### 9.3 cartorio-verify audit run

```
cartorio-verify audit \
  --pinned-root abcd1234... \
  --against https://openclaw.dev.use1.quero.lol

# 1. GET /api/v1/merkle/root → current = ef567890...
# 2. GET /api/v1/merkle/consistency?from=abcd1234...&to=ef567890...
# 3. Verify consistency proof locally (in-process; no network in math).
# 4. For each Active listing:
#    GET /api/v1/artifacts/{id}/proof → verify inclusion locally
# 5. Emit ✓ or ✗
```

### 9.4 cartorio-web's startup query

```graphql
query DashboardInitial {
  merkleRoot { ledgerRoot ledgerIndex signedAt treeSizes { activeListings eventsTotal } }
  listings(filter: { state: ["active"] }, page: { limit: 50 }) {
    data { ... on ActiveListing { listing { listingId kind name version certification { composedRoot } } } }
    total
    ledgerRootAtQuery
  }
  events(page: { limit: 20, desc: true }) {
    data { eventId kind listingId createdAt }
  }
}
```

Followed by three subscriptions opened simultaneously (per § 6.2).

## 10. cartorio-web's compile-time contract

cartorio-web (`crates/cartorio-app/`) generates Rust types from this
schema using `cynic`. The schema lives in
`cartorio-web/crates/cartorio-app/schema.graphql`, refreshed via:

```
cd cartorio-web
nix run github:pleme-io/cartorio#schema:cartorio  # writes the latest
git diff schema.graphql                            # review changes
cargo check -p cartorio-app                        # confirm no breakage
```

REST clients are hand-written in `crates/cartorio-app/src/api.rs`
against the JSON shapes documented above. Types are `#[derive(Serialize,
Deserialize)]` records mirroring § 2.5 and § 2.6 verbatim.

## 11. What's deferred (M1+)

| Feature | M0 status | M1+ shape |
|---|---|---|
| Cursor pagination | offset-based only | `?after=<opaque>` + Relay-style page info |
| ETag / If-None-Match | none | per-listing-id ETags from `updated_at` hash |
| HTTP/2 server push | none | for proof-bundle responses |
| Auth middleware | open | x-user-* (cracha/passaporte SSO via Hanabi) |
| Field-level deprecation | none | `@deprecated` directives on GraphQL fields |
| Persisted queries | none | hash-named query allowlist for prod |
| Rate limiting | none | per-publisher token bucket on POST endpoints |
| OpenAPI spec | none | machine-readable spec for non-GraphQL consumers |

These are real but out-of-scope for the demo.

## 12. The contract is the boundary

This document is the substrate's wire contract. cartorio-backend is the
unique writer; everything else is a reader, gated against the same
single signed posture.

> *If you can speak this contract, you can build a new gate. The
> substrate is open by design — through cartorio.*

`05-VISUAL-PLAN.md` ready next: Leptos route + component wireframes,
keyed against the GraphQL operations in § 5. Or push back / refine the
contract first?
