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
| `$id` | `https://runtimeconditions.io/schemas/profile/0.3.0/runtimeconditions.profile.schema.yaml` |
| Version | `0.3.0` |
| Semantic SHA-256 | `83be993b93f81561e405695143f873ef34af65297f32bc2266b6534444da1466` |
| Source-byte SHA-256 | `9f09d24054919e992f9c209cbd8098e2f0de382132fb06dbc3207e4e12c60e35` |

Version `0.3.0` requires extension declarations to be objects with exactly two
non-empty string fields: `id` and `version`. Condition `extension` selectors use
the same object. Duplicate pairs are invalid; different versions of one ID are
distinct references. A reference cannot be a bare string, including a concatenated
URI/version string.

IDs SHOULD use a format understood by a resolver, such as an HTTPS URI, a file
URI, or an OCI archive reference. URI syntax is not required, so the schema does
not impose a URI `format` or identifier pattern. Versions are exact strings and
are not inferred from IDs or interpreted as ranges.

The schema checks core shape, types, required values, duplicate extension pairs,
and the required `interface.type` for each Condition. Validators must separately
reject duplicate YAML mapping keys and duplicate Condition names; resolve the
complete extension closure by exact (`id`, `version`) pairs; check that a Condition
selector matches a declared pair; verify vocabulary ownership; and apply every
applicable extension JSON Schema. Checks for secrets and concrete target-environment
values also remain outside this structural schema.

The normalized binding model's `coreProfileSchema` identity, version, and semantic
digest must match the actual installed schema before profile validation.

## Extension metadata

[`runtimeconditions.extension-metadata.schema.yaml`](runtimeconditions.extension-metadata.schema.yaml)
validates the `metadata` object of an extension definition, not the entire
definition. It requires non-empty `id` and `version` strings and rejects the old
`uri` field, including when `id` is also present. Artifact and tooling conventions
may add non-identity metadata and must validate their own fields separately.
Neither an additional digest nor a retrieval location changes the release identity.

| Identity field | Value |
| --- | --- |
| `$id` | `https://runtimeconditions.io/schemas/extension-metadata/0.3.0/runtimeconditions.extension-metadata.schema.yaml` |
| Version | `0.3.0` |
| Semantic SHA-256 | `aa3680673bbed577c6ac701299c574b3a7a90f158a4be96a988ccf9d4e5a11db` |
| Source-byte SHA-256 | `4869d72b28662d3dd1390de0ed422e0e98c063405984b6a862ffda0ed2c58973` |

Both metadata fields must exactly equal the requested reference after resolution.
The metadata schema checks their shape; the resolver checks that equality and
the immutability of the resolved release.

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

## Local verification

```sh
python -m pip install -r requirements-dev.txt
python -m unittest discover -s tests
python tools/verify_schemas.py --release-tag v0.3.0
```
