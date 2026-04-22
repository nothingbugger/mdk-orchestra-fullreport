# 07 — Federated Memory Sharing — Design Document

**Status**: Design direction. Not implemented in the current prototype.

Implementation would be a natural extension of the memory layer once multiple mining operators run Orchestra independently.

---

## 7.1 Motivation

Memory patterns are interpretable artifacts that encode operational lessons. Every site that runs Orchestra independently discovers patterns through its own operations.

If two sites cannot share these patterns, each learns in isolation — inefficient and wasteful. If two sites can share raw telemetry, they expose competitive information (miner count, uptime, margins) and potentially customer-sensitive data.

**The goal**: enable pattern sharing while keeping raw telemetry strictly local.

## 7.2 Core properties required

| Property | Requirement |
|---|---|
| Privacy | Raw telemetry never leaves site perimeter |
| Authenticity | Each pattern is cryptographically attributable to its emitter |
| Decentralization | No trusted broker or single point of failure |
| Censorship resistance | Pattern distribution resistant to jurisdictional interference |
| Auditability | Every ingested pattern traceable to source with timestamp |
| Selective merge | Each site decides which external patterns to accept |
| Reversibility | Merged patterns can be reverted if later found harmful |

## 7.3 Proposed protocol: Nostr over Tor

### Why Nostr

- Native signed event model (authenticity solved by construction)
- Relay-based but stateless (no consensus overhead)
- Pub/sub by tag (natural pattern routing)
- Self-hostable relays
- Mature ecosystem, Tor-friendly

### Why Tor

- Operators may not want other operators to know which country, ISP, or data center they are in
- Censorship resistance (e.g., jurisdictions hostile to mining)
- Protects site metadata without adding protocol complexity

### Event model

Each memory pattern written by Maestro during curation can be optionally published as a Nostr event:

```json
{
  "kind": 30078,
  "pubkey": "<site_npub>",
  "created_at": 1712345678,
  "tags": [
    ["d", "<pattern_slug>"],
    ["t", "mining-predictive-maintenance"],
    ["t", "chip-instability"],
    ["site-class", "mid-size-apac"]
  ],
  "content": "<full pattern markdown>",
  "sig": "<signature>"
}
```

Fields:
- `kind: 30078` — parameterized replaceable event (Nostr convention for named updatable records)
- `d` tag — pattern slug (enables upsert: same slug from same pubkey overwrites)
- `t` tags — domain + subdomain for subscription filtering
- `content` — full pattern markdown (same format used locally)
- `sig` — Schnorr signature over canonical serialization

### Subscription

Sites subscribe to relays with filter:

```json
{
  "kinds": [30078],
  "#t": ["mining-predictive-maintenance"],
  "since": <last_sync_timestamp>
}
```

Received events are candidates for ingestion, not automatic imports.

### Merge protocol

Each site runs a merge agent (could be Maestro itself during a dedicated federated-curation cycle):

1. Fetch new events from subscribed relays
2. Filter by trust level of source pubkey (per-site policy)
3. For each event:
   a. Verify signature
   b. Check pattern format validity (schema compliance)
   c. Evaluate pattern against local decision log — would this have helped? Is it consistent with local patterns?
   d. Score: confidence if accepted, reason if rejected
4. Accepted patterns written to a quarantine file (not yet active memory)
5. Active memory promotion requires either:
   - Multi-source corroboration (N independent sites published the same or similar pattern)
   - Manual operator approval
   - Local decision log match (pattern predicts what local logs already confirmed)

### Reputation / trust layer

Each site maintains a per-pubkey reputation:
- Patterns from this pubkey historically accepted / rejected
- Patterns from this pubkey that caused bad decisions
- Patterns from this pubkey that corroborated locally-derived patterns

Trust is gossipable — sites can publish signed reputation events about other sites, enabling emergent consensus on which operators produce reliable patterns. Not required for v1 but natural extension.

## 7.4 Threat model

| Threat | Mitigation |
|---|---|
| Poisoned pattern from malicious pubkey | Multi-source corroboration required for auto-promotion; operator approval otherwise |
| Sybil attack (many fake pubkeys) | Reputation layer; initial N-source threshold raised if anomalous pubkey distribution detected |
| Relay censorship | Tor hidden services for critical relays; clients configured with multiple relays |
| Traffic analysis | Tor routing protects metadata |
| Private pattern leak | Site policy determines which patterns are published; sensitive patterns can be tagged local-only |
| Pattern supply chain (compromised site) | Signature audit trail enables post-hoc revocation |

## 7.5 What patterns should be shared

Not all memory is suitable for sharing. A rough taxonomy:

| Pattern class | Share by default? | Notes |
|---|---|---|
| General physics-of-mining patterns (e.g., DVFS instability) | Yes | Universal knowledge, low risk |
| Tariff-window patterns | No | Reveals site location / grid region |
| Fleet-composition patterns | No | Reveals hardware mix / business info |
| Specific miner model behaviors | Partial | Model is shared info; behaviors sometimes operationally sensitive |
| Failure signatures from specific firmware versions | Yes | Useful cross-site; firmware is public |
| Cross-domain reasoning patterns (Maestro memory) | Yes | General reasoning; rarely site-specific |

Operator explicitly tags patterns as `share` or `local_only` during curation.

## 7.6 Implementation path

### v0.1 (current — not yet built)

Memory files are already portable markdown with canonical format. Nothing to build yet.

### v0.2 — Export/import tooling

- Tool to serialize memory into Nostr events (signed, not yet published)
- Tool to import signed pattern files into quarantine

No network yet. Operators can exchange signed pattern files out-of-band (email, GitHub, etc.). Validates the format and signing flow.

### v0.3 — Relay subscription

Integrate Nostr client library (e.g., `nostr_python`).
Configure subscribed relays per site.
Background process pulls new events, writes quarantine file.
No merge yet — operator manually reviews quarantine.

### v0.4 — Merge automation

Add merge agent logic.
Multi-source corroboration threshold.
Reputation tracking (per-pubkey metadata file).
Auto-promote high-consensus patterns to active memory.

### v0.5 — Reputation gossip

Sites publish signed reputation events.
Transitive trust graph.
Emergent high-trust cohort.

## 7.7 Limitations and open questions

- **Pattern collision**: two sites may publish similar but slightly different patterns. Merge semantics are non-trivial.
- **Pattern stale-ness**: patterns may remain valid for years, but some depend on firmware versions, miner models, or market regimes that change. Versioning and expiration unclear.
- **Privacy audit**: even "patterns only" can leak operational information in aggregate. Formal information-theoretic analysis needed before wide deployment.
- **Anti-spam**: Nostr relays see abuse; rate limits and reputation filtering must be robust.
- **Cross-language**: patterns in natural language (English) may not translate across operator language preferences. Standardize on English as canonical for shared patterns?

## 7.8 Why this belongs in the architecture

Even unimplemented, the design is architecturally consequential:

- Memory format is **already portable** (markdown, signed-friendly)
- No runtime design changes needed to add sharing later
- Operators running Orchestra locally are already producing the raw material for a federated layer
- The value proposition scales with adoption: each new site makes every other site's Orchestra smarter

A future mining operator consortium adopting Orchestra would have a ready-made privacy-preserving learning network.
