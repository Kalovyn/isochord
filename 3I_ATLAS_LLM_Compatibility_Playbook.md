# 3I-ATLAS + ISOCHORD LLM Compatibility Playbook

This playbook is an implementation-oriented companion to the README compatibility track.

## Goals
- Keep YAERU semantics stable while model providers, SDKs, and tool schemas evolve.
- Make 3I-ATLAS state transitions observable and testable across model families.
- Fail safely when outputs are malformed or consent constraints are ambiguous.

## Reference architecture

```text
User Turn
  -> 3I-ATLAS Intent Classifier
  -> YAERU Transition Mapper
  -> Model Adapter (provider-specific)
  -> Structured Output Validator
  -> Consent/Truth Gate
  -> Ledger Writer (.jsonl)
```

## Canonical turn envelope
All adapters should normalize model output into this envelope before execution:

```json
{
  "atlas_state_in": "string",
  "yaeru_transition": "AE|YA|AN|EL|RU",
  "consent_state": "open|hold|close",
  "truth_confidence": 0.0,
  "boundary_flags": ["string"],
  "response_text": "string",
  "ledger_write": true,
  "adapter_meta": {
    "provider": "string",
    "model": "string",
    "schema_version": "2026-05-02"
  }
}
```

## Compatibility modes

### 1) Strict
- Enforce JSON schema on every turn.
- Reject turn when `consent_state` is missing or invalid.
- Retry once with repair prompt; then fail closed.

### 2) Balanced
- Prefer schema-valid output.
- Attempt bounded repair (max 2 passes) for missing non-safety fields.
- Block only on consent/truth/boundary failures.

### 3) Fallback
- Parse plain text conservatively.
- Default to `AN` + `hold` when intent is uncertain.
- Mark turn with warning flag in ledger metadata.

## Minimum validation checks
- **Consent gate:** no meaningful action without explicit open/hold-close logic.
- **Boundary gate:** if boundary flag is raised, force containment response.
- **Truth gate:** if confidence below threshold, downgrade to non-committal response.
- **Ledger gate:** append-only write must include hashable context key and timestamp.

## Cross-model test matrix
Run the same compliance scenarios on:
- 1 small hosted model
- 1 frontier hosted model
- 1 local/open-weight model

Track pass/fail for:
- YAERU transition correctness
- Boundary honoring
- Silent-fail behavior under malformed output
- Deterministic recovery behavior in strict mode

## Suggested next repo updates
1. Add JSON schema file for the canonical envelope (`schemas/turn-envelope.v1.json`).
2. Add adapter conformance tests in CI.
3. Add fixture conversations for edge-case consent and rupture scenarios.
4. Version compatibility policy per adapter (`adapters/<provider>/COMPAT.md`).
