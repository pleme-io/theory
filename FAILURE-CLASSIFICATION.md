# Failure Classification — typed reconciler-loop META

> **Frame.** Companion to [`SHIGOTO.md`](./SHIGOTO.md) (the typed work-graph
> primitive). This doc owns the typed-failure-classification META —
> the answer to "should this reconciler keep retrying?" expressed as a
> compile-time invariant of every long-running pleme-io daemon.
>
> **Status.** Live. shigoto-types v0.1.0 ships the canonical primitive
> at `shigoto-types::failure::{FailureKind, Failure, classify, signature}`.
> kikai is the bootstrap consumer; magma / tatara-reconciler /
> pangea-operator / engenho-controllers are next.
>
> **Companion verification:** [`FAILURE-CLASSIFICATION-VERIFICATION.md`](./FAILURE-CLASSIFICATION-VERIFICATION.md)
> — the typed-primitive audit per `TYPESCAPE-METHODOLOGY.md`.

---

## I. The repeating pattern

Every long-running pleme-io reconciler has the same loop:

```
loop:
  outcome = attempt()
  if outcome.ok: continue
  log("retrying after backoff")
  sleep(backoff)
```

Without typed failure classification, this conflates two unrelated
classes of error:

| Class | Example | Right behaviour |
|---|---|---|
| **Transient** | builder unreachable, DNS blip, dependency not yet ready | retry per policy — conditions may clear |
| **Declarative** | missing flake attribute, NixOS module eval error, SOPS reference missing, schema mismatch | give up + surface to operator — won't fix itself |

The naive loop treats both the same: retry forever. The reconciler
spams logs, burns CPU + nix-eval cycles, and the operator's only signal
that something is wrong is the cluster never reaching Healthy. The
typed mental model — "permanent vs transient" — exists in every
operator's head but isn't in the substrate.

This was the exact failure mode that surfaced engenho-local's
cross-module wiring blocker on 2026-05-18: kikai retried `nix build
.#packages.aarch64-linux.engenho-local-image` 8+ times in 5-minute
backoff before any signal that "this declaration is broken."

---

## II. The destination

A single typed primitive in `shigoto-types::failure` that every
long-running pleme-io daemon consumes. Adding a new daemon (or
extending an existing one to handle a new declarative-failure class)
becomes one of two operations:

1. **Add a Declarative pattern**: one edit in
   `shigoto-types/src/failure.rs`'s `DECLARATIVE_PATTERNS` array.
   Every consumer picks up the new pattern on next `cargo update -p shigoto-types`.

2. **Adopt the primitive in a new daemon**: import `Failure::from_raw`,
   classify each failure, route Declarative → Deadletter via
   `shigoto-retry::RetryPolicy::decide` (which short-circuits
   automatically — no per-daemon retry-budget logic needed).

The primitive itself is small + dense:

```rust
pub enum FailureKind {
    Transient,    // retry per policy
    Declarative,  // operator must fix; route to Deadletter
}

pub struct Failure {
    pub kind: FailureKind,
    pub message: String,    // truncated 256 chars
    pub signature: String,  // stable for "same error twice" detection
}

pub fn classify(raw: &str) -> FailureKind;
pub fn signature(raw: &str) -> String;
```

`Default for FailureKind` is `Transient` — the conservative choice.
Unknown error shapes get the "keep trying" path; permanent classification
is opt-in via documented pattern match.

---

## III. The Declarative pattern set

The patterns documented as of v0.1.0 (`shigoto-types/src/failure.rs`):

### Nix evaluation / flake-attribute failures

- `does not provide attribute`
- `does not exist`
- `evaluating the attribute`
- `infinite recursion encountered`
- `syntax error`
- `attribute set is missing the attribute`
- `value is null while a set was expected`
- `value is a function while a set was expected`
- `is not allowed to refer to a store path`
- `cannot coerce`
- `while evaluating definitions from`

### NixOS module assertion + option type-checks

- `in the condition of the assert statement` *(added 2026-05-18 from
  in-the-wild engenho-local trip)*
- `assertion failed`
- ``The option ` ``
- ``is missing the attribute ` ``

### SOPS / secret resolution failures

- `missing secret`
- `could not decrypt`

### Cargo / build-time

- `could not find Cargo.toml`

### Schema mismatches (Terraform / Crossplane / K8s admission)

- `schema validation failed`
- `unknown attribute`
- `field required`

### Typed-wrapper markers

When a consumer classifies internally and bubbles up via `anyhow!`, the
wrapper marker preserves the classification through re-classification:

- `preflight failed (Declarative)`
- `[Declarative]`

The pattern set is **append-only** — never remove a documented pattern,
only add. Each addition is a substrate-wide hardening that every
consumer inherits on next dep update.

---

## IV. The retry-policy short-circuit

`shigoto-retry::RetryPolicy::decide(attempt, history)` consults the
latest `FailureRecord.kind` BEFORE evaluating the policy's
attempts/backoff:

```rust
if let Some(latest) = history.last() {
    if latest.kind == FailureKind::Declarative {
        return RetryDecision::Deadletter;
    }
}
// ... existing policy logic ...
```

Same `RetryPolicy::Exponential { attempts: 100, ... }` declaration,
DIFFERENT behaviour:

- Transient failures: get the full 100-retry budget.
- Declarative failures: stop on first attempt.

The decision is made once, in one place. No per-daemon copy-paste.

---

## V. Verifiability surface

| L | Surface | Where |
|---|---|---|
| 0 | Type proofs (compiler) | `FailureKind` is `#[non_exhaustive]` + `Default = Transient` — invalid composition is a build error |
| 1 | Unit tests | `shigoto-types::failure::tests` — 19/19 (every pattern, default-is-Transient, serde round-trip) |
| 2 | Property tests | `classify_is_deterministic` + `signature_is_stable_across_runs` — determinism contract |
| 3 | META invariants | `shigoto-retry::tests` — 4 cases asserting `Declarative ⇒ Deadletter regardless of policy` |
| 4 | Integration | `kikai/src/daemon.rs` consumes the primitive; the operator's `kikai status` reads persisted typed state and surfaces remediation hints |
| 5 | E2E (proven 2026-05-18) | engenho-local hit a real declarative failure → daemon classified → transitioned to `BlockedDeclarative` → persisted to disk → `kikai status` surfaced typed `[BLOCKED]` block |

See [`FAILURE-CLASSIFICATION-VERIFICATION.md`](./FAILURE-CLASSIFICATION-VERIFICATION.md).

---

## VI. Consumers

| Consumer | Status | Failure classes it owns |
|---|---|---|
| **kikai** daemon | LIVE (bootstrap consumer) | nix build / NixOS eval / SOPS / preflight |
| **magma** apply engine | queued | Terraform schema mismatch (Declarative) / provider 5xx (Transient) |
| **tatara-reconciler** | queued | `intent.nix` eval error (Declarative) / FluxCD pending (Transient) |
| **pangea-operator** | queued | IAM permission denied (Declarative) / AWS 5xx (Transient) |
| **engenho** controllers | queued (post-M0.4) | CRD admission rejection (Declarative) / kubelet not yet ready (Transient) |
| **caixa Supervisor** | queued (post-M2) | M2 typed slot violation (Declarative) / pod scheduling timeout (Transient) |

Each consumer's adoption is ~30 LoC of integration + their own
Declarative pattern contributions to extend the shared set.

---

## VII. Anti-patterns

- **Per-daemon copy of the classifier.** Forbidden — every consumer
  imports from `shigoto-types::failure`. Pattern additions land
  upstream once.
- **Removing a Declarative pattern.** Forbidden — once documented,
  patterns are append-only. Failures that were Declarative yesterday
  are still Declarative today.
- **Treating unknown errors as Declarative.** Forbidden — default is
  `Transient` (conservative; retry might work). Permanent classification
  is opt-in via explicit pattern match.
- **Adding a third FailureKind variant without need.** Forbidden — the
  binary distinction (retry vs surface) is sufficient. Sub-classifications
  belong in the `Failure.signature` field (free-form), not in the enum.
- **Bypassing the typed retry policy.** Forbidden — every daemon's
  retry loop goes through `RetryPolicy::decide`, which honours the
  classification. No per-daemon "ignore the policy and just retry"
  short-circuits.

---

## VIII. Operating principles applied

| Principle | How this primitive honours it |
|---|---|
| #0 path-of-least-resistance forbidden | The destination ("one classifier, every reconciler") was named before the path. The path is: promote to shigoto-types, refactor kikai to consume, document, repeat for next consumer. |
| #1 solve once, in one place | One classifier in shigoto-types; every consumer imports. Pattern extensions land once. |
| #2 load-bearing fix over local fix | When kikai daemon's loop hit infinite retry on a declarative failure, the fix wasn't "add a max-retry cap to kikai" — it was "make every daemon classify failures." |
| #3 idiom-first | The `RetryDecision::{Retry, Deadletter}` shape already existed in shigoto. The new primitive RIDES the existing typed surface; doesn't reinvent. |
| #4 models stay current | This doc + the verification companion + the in-code comments in shigoto-types + kikai + daemon-supervision.md all cite each other. |
| #5 direction beats velocity | The session that landed this primitive declined to commit a kikai-local copy of `FailureKind` ("we'll promote later") in favour of promoting upstream first. |
| #6 single goals are anti-goals | "Make engenho-local stop retrying forever" became "make every reconciler in the fleet handle this class of bug correctly." |
| #7 acquire + contextualize | The Nix-evaluation Declarative patterns came from observing engenho-local's real failures, not from reading nix source. Each in-the-wild observation hardened the substrate. |

---

## IX. Open questions

### IX.1. ProtoBuf / gRPC structured errors

Today's classifier operates on raw strings (anyhow / nix stderr). For
consumers that produce structured errors (gRPC status codes, Terraform
provider tfplugin5/6 protocol errors), the classifier could consult
the structured fields directly + skip the string-match step. Not
blocking; documents an extension surface.

### IX.2. Severity dimensions

`FailureKind` is binary. Some failures have shades — e.g., "Declarative
but auto-fixable" (the operator might want a CRD remediation
controller to handle some Declarative errors automatically). For now
that's out of scope; the binary distinction is the only one consumers
need.

### IX.3. Time-windowed declarative repeat

A daemon could detect "Declarative failure now, but it might be
transient if it was Transient five minutes ago" — i.e., signature
flapping. Today the substrate is signature-equality-based, not
windowed. If we see flapping in the wild, extend.

### IX.4. Distributed classification

When magma's apply runs against a provider that returns N errors
across N resources, each one might be Declarative or Transient. The
classifier as-is operates on a single string; for parallel apply, we
need a `Vec<Failure>` aggregation. Easy extension.

---

## X. Relationship to other primitives

- **shigoto** — typed work-graph (Job, JobPhase, Scheduler). `Failure`
  is the typed failure record that flows through every Job's outcome.
- **tameshi** — BLAKE3 attestation. `Failure.signature` could attach
  to a tameshi receipt to attest "this artifact failed Declaratively
  with signature X" — typed-failure provenance.
- **kensa** — compliance verification. A compliance violation IS a
  Declarative failure (operator must fix the manifest). Routes through
  the same primitive.
- **kikai daemon** — bootstrap consumer; persists Failure to disk + surfaces in status.
- **magma / tatara-reconciler / pangea-operator** — queued consumers.

---

*One typed primitive; every reconciler shares the same answer to "should I keep trying?".*
