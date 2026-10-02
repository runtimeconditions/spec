# Core profile schema

[`runtimeconditions.profile.v0.1.0.schema.yaml`](runtimeconditions.profile.v0.1.0.schema.yaml)
is the JSON Schema Draft 2020-12 core structural contract for the profile in
[`sixth-draft.md`](../docs/sixth-draft.md). It is independent of any extension
and must be distributed with an installed profiler before that profiler accepts
a generated profile. A workload must not need a checkout of this repository or
a separate core-schema package.

| Identity field | Value |
| --- | --- |
| `$id` | `https://runtimeconditions.io/schemas/profile/0.1.0/runtimeconditions.profile.schema.yaml` |
| Version | `0.1.0` |
| Semantic SHA-256 | `49890a0f3e7276d1e480d654176672d977df9c63094f3a24983b0a8102e1a3e3` |
| Source-byte SHA-256 | `ad101336b676b468ec975aff45c21749df22fddf422371156c586ed62abf3223` |

The semantic digest is SHA-256 of RFC 8785 canonical JSON for the parsed schema.
It preserves array order and includes `$id` and
`x-runtimeconditions-version`. The source-byte digest covers the exact YAML
file. A change to the parsed schema requires a new version and `$id`; a
presentation-only YAML change requires a new source-byte digest.

The schema checks core shape, types, required values, unique extension IDs, and
the required `interface.type` for each Condition. Validators must separately
reject duplicate YAML mapping keys and duplicate Condition names; resolve the
complete extension closure; verify vocabulary ownership; and apply every
applicable extension JSON Schema. URI `format` assertion must be enabled, or
extension identifiers must be checked with an equivalent absolute-URI parser.
Checks for secrets and concrete target-environment values also remain outside
this structural schema.
The normalized binding model's `coreProfileSchema` identity must match the
actual installed schema before profile validation.
