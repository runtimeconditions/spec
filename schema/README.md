# Runtime Conditions schemas

These self-contained JSON Schema Draft 2020-12 artifacts implement the current
contracts in [`sixth-draft.md`](../docs/sixth-draft.md).

## Profile structure

[`runtimeconditions.profile.schema.yaml`](runtimeconditions.profile.schema.yaml)
validates the core profile structure. It is independent of extension vocabulary
and must ship with an installed profiler before that profiler accepts a generated
profile. A workload must not need a specification checkout or separate schema package.

| Identity field | Value |
| --- | --- |
| `$id` | `https://runtimeconditions.io/schemas/profile/0.4.0/runtimeconditions.profile.schema.yaml` |
| Version | `0.4.0` |
| Semantic SHA-256 | `ed447dccefd7507d905b0b6177b1386d8ee96a1d053b731939a9a7ae973d5af1` |
| Source-byte SHA-256 | `96d430c7936fcf7334aa9613f63bb306592e07cf56fe4154af57933d2c1480e6` |

Version `0.4.0` requires Profile extension declarations and Condition extension
selectors to be non-empty strings. Entries may be identifiers alone or
identifiers followed by `:<version>`. The schema does not require URI syntax or
parse the optional suffix. Duplicate strings are invalid.

The schema checks core shape, types, required values, duplicate extension
strings, and the required `interface.type` for each Condition. Validators must
separately reject duplicate YAML mapping keys and duplicate Condition names;
resolve references according to configured tooling; verify vocabulary ownership;
and apply every applicable extension JSON Schema.

The normalized binding model's `coreProfileSchema` identity, version, and semantic
digest must match the actual installed schema before profile validation.

## Integrity and releases

The semantic digest is SHA-256 of RFC 8785 canonical JSON for the parsed schema.
It preserves array order and includes `$id` and `x-runtimeconditions-version`.
The source-byte digest covers the exact YAML file. A parsed-schema change requires
a new version and `$id`; a presentation-only YAML change requires a new source-byte
digest. Consumers must pin the exact schema identities and digests they implement.
They must not give different contracts the same schema `$id`.

The bundle is released with a `v`-prefixed Git tag matching both artifact versions.
Each release contains both files unchanged and a `SHA256SUMS` file covering both.
The release workflow verifies the tag, schema identities, documented digests, and
conformance cases before publishing.

Earlier schemas remain available from their tagged commits and releases. The
previous profile schema `0.2.0` required fifth-draft HTTPS URI/version strings;
its identity and contents must not be reused for this contract. Migrating a
consumer requires updating its schema pin and pair-based resolution together.
