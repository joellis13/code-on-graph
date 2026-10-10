# ADR-0016: Secret redaction before egress, in logs, and in storage

- **Status:** Accepted
- **Date:** 2026-10-08

## Context

Code-OG sends repository content to third-party model providers and records prompts, responses, and events locally. Code and issues sometimes contain credentials. The Planning reference left the redaction rules open (open decision 11). Full detail is in [Policy and sandbox](../explanation/policy-and-sandbox.md), Section 7.

## Decision

- A `Redactor` interface takes text and returns redacted text plus findings (kind, location, rule id; never the secret value).
- Detection has three layers: **known secrets** (the exact values of credentials cog itself holds), **well-known token formats**, and **heuristics** (high-entropy values assigned to secret-like names, connection strings with embedded credentials).
- Replacement token: `[REDACTED:kind:xxxx]`, where `xxxx` is a short **salted, per-run** hash, so repeats are recognizable without being guessable.
- Redaction runs at the **context packer**, on **tool results**, as a **last-line filter in every LLM adapter's outbound payload**, and in the **event and log writers**. No path bypasses the adapter filter.
- Context items that are themselves secret files or credential blocks are **dropped whole**, with a warning.
- The blob store keeps outbound prompts **as sent** (already redacted). Raw fetched GitHub JSON is stored unredacted for replay because it never leaves the machine; the data directory must be on an encrypted disk and outside synced folders.
- **v1 uses cog's own ruleset.** Gitleaks and detect-secrets may be added as optional adapters. **TruffleHog's live verification is not used,** because it would transmit secrets to third parties.
- Redaction is **biased toward redacting** in outbound code. Allow-patterns loosen it and therefore live only in user-level config.
- Redaction counts are recorded per run and shown as a warning.

## Alternatives considered

- **Rely on the deny-list alone:** misses secrets embedded in ordinary files.
- **Adopt one external scanner as the only detector:** adds an install or binary dependency on Windows and cedes false-positive tuning.
- **Redact only at the packer:** misses tool results, error messages, and logs.

## Consequences

- False positives will occur; fixtures and measurements tune them.
- Rule provenance and licenses must be checked before seeding rules from open-source scanners **(verify)**.

## Revisit when

False-positive rates harm plan quality, or an optional scanner proves clearly better on fixture corpora.
