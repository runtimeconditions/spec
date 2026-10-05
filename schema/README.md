# Core profile schema

[`runtimeconditions.profile.schema.yaml`](runtimeconditions.profile.schema.yaml)
is the JSON Schema Draft 2020-12 core structural contract for the profile in
[`sixth-draft.md`](../docs/sixth-draft.md). It is independent of any extension
and must be distributed with an installed profiler before that profiler accepts
a generated profile. A workload must not need a checkout of this repository or
a separate core-schema package.

| Identity field | Value |
| --- | --- |
| `$id` | `https://runtimeconditions.io/schemas/profile/0.2.0/runtimeconditions.profile.schema.yaml` |
| Version | `0.2.0` |
| Semantic SHA-256 | `a090a8016d045f9c3fa872a67f8df293b77ca2809a1bea5ae9fa31a27a06109a` |
| Source-byte SHA-256 | `342bf20bce479f5012b9fc2c6238dc1fb0935e327ecb0fbca6e647362563c73c` |

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

The schema is released with a `v`-prefixed Git tag. Each release contains this
file unchanged and a `SHA256SUMS` file that verifies its exact source bytes.
Earlier schema versions remain available from their tagged commits and
releases. Version `0.2.0` requires HTTPS URI/version identifiers; exact catalog
path and identity checks are enforced by the resolver.
