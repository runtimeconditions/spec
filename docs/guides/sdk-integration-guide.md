# SDK Integration Guide

## Status

**Non-normative implementation guidance**

This guide describes how SDK authors can package Runtime Conditions metadata so workloads that import those SDKs can generate accurate Runtime Conditions Profiles without adding application-specific configuration files.

The core Runtime Conditions Profile specification defines the profile document and extension artifacts. This guide defines a packaging convention for SDKs and production libraries that want their internal runtime integrations to surface into generated workload profiles.

---

# 1. Goal

Many workloads access external runtime integrations through SDKs rather than direct application code.

Examples:

- A Go service calls `s3.Client.PutObject`.
- A Python service calls `boto3.client("s3").put_object`.
- A TypeScript service calls `new S3Client(...).send(new PutObjectCommand(...))`.

Without SDK-provided metadata, a generator can see the imported SDK or production library and the method call, but it cannot reliably know which Runtime Conditions extension owns that integration vocabulary or how the call maps to a Condition.

The SDK package convention solves this by allowing SDK authors to ship:

- A Runtime Conditions package manifest
- A reference to the extension definition owned by that SDK, vendor, or ecosystem
- Language-specific symbol mappings from SDK calls to canonical service operations and Conditions

---

# 2. Service Semantic Authority

SDK authorship begins with service semantics, not language symbols. Before mapping any SDK class, method, function, or command, the integration must identify the language-neutral source that defines the service operations and the reviewed translation from those operations to an exact Runtime Conditions extension release.

Use an authoritative machine-readable service description when one adequately identifies the operation surface. Smithy models, OpenAPI descriptions, protobuf service definitions, and comparable provider-maintained specifications belong in this category. Reference the exact authoritative artifact needed for reproducible generation and do not copy its operation inventory into another hand-maintained file.

When no adequate authoritative machine-readable operation source exists, service stakeholders SHOULD maintain a **Service Operations Inventory**. The standard filename is `service-operations-inventory.yaml`. This inventory becomes the reviewed, cohesive service-level result set for Runtime Conditions SDK authorship; it is not inferred from one language SDK and it is not recreated for every language or SDK version.

A Service Operations Inventory records:

- A stable service identity
- Stable, language-neutral operation names
- Service-owned operation inputs and input requiredness
- Service-domain value shapes and constraints needed to interpret those inputs

A Service Operations Inventory does not contain Runtime Conditions extension coordinates, Condition semantics, Adapter behavior, package paths, class names, method names, argument positions, generated model types, wrapper call graphs, or language-specific state flow. Runtime Conditions semantics belong in a Service Operations Semantic Bridge; language facts belong in SDK mappings.

A **Service Operations Semantic Bridge**, using the standard filename `service-operations-semantic-bridge.yaml`, references the authoritative Smithy, OpenAPI, protobuf, or other provider model when one exists; otherwise it references the exact fallback Service Operations Inventory. The bridge answers the authoring question “What adapter demand does a particular service operation prove?” It owns only the reviewed translation from source operations and inputs to extension Condition semantics. It does not duplicate a complete authoritative model, contain SDK symbols, or become part of the runtime contract consumed by an Adapter.

The authoritative service description or fallback inventory and its semantic bridge compile into a deterministic extension definition and `RuntimeConditionsServiceMapping`. The extension remains the standalone vocabulary and validation contract. The generated service mapping records exact extension coordinates, canonical operation-to-Condition translations, and semantic digests. Every language-specific SDK mapping references operations from that service mapping and records its digest. The maintenance relationship is therefore one service semantic authority and bridge to many SDK languages and releases.

Source, extension, and SDK release changes have separate maintenance outcomes. A new source operation that maps to existing Condition vocabulary changes the bridge or generated service mapping without necessarily changing the extension. Every affected SDK mapping must still regenerate or be updated when its SDK release exposes that operation, so its public method can reference the canonical service operation and existing Condition. An SDK that does not expose the operation does not invent a mapping for it. If the translation requires new adapter-facing vocabulary, the extension must be revised and versioned before SDK mappings target it.

The standard repository naming convention is:

| Artifact | Standard name | Role |
| --- | --- | --- |
| Maintained fallback source | `service-operations-inventory.yaml` | Reviewed, cohesive service operation authority when no adequate upstream machine-readable model exists |
| Reviewed semantic translation | `service-operations-semantic-bridge.yaml` | Maps authoritative or fallback service operations to adapter-actionable extension Conditions |
| Generated service projection | `<service>-service-mapping.yaml` | Deterministic language-neutral mapping from canonical operations to extension Conditions |
| Generated language projection | `runtimeconditions.sdk-mapping.yaml` | Deterministic mapping from one SDK release's public surface to canonical service operations |
| Published extension release | `runtimeconditions.extension.yaml` | Immutable Condition vocabulary and validation contract |

NATS is the current maintained-inventory example. AWS S3 instead begins with the public AWS Smithy model, and Kubernetes begins with the published OpenAPI description. These are different source workflows that converge on the same language-neutral service-mapping boundary.

The [Service Operation Authoring Guide](service-operation-authoring.md) provides a full tutorial covering source selection, the last-resort inventory path, semantic bridge authoring, extension and service-mapping generation, change propagation, and an end-to-end NATS operation.

---

# 3. SDK Author Responsibilities

An SDK or production library author SHOULD:

- Identify SDK operations that imply external runtime integration requirements.
- Reuse the authoritative service mapping for the selected extension release rather than defining service semantics in a language mapping.
- Participate in review of a `service-operations-inventory.yaml` only when no adequate authoritative machine-readable service model exists.
- Define or reference the Runtime Conditions extension vocabulary for those requirements.
- Ship a `runtimeconditions.package.yaml` manifest in the imported package.
- Package the extension definition as `runtimeconditions.extension.yaml` next to that manifest.
- Publish any SDK-owned extension definition as a standalone artifact that can be used without the SDK.
- Declare extension dependencies in the extension definition.
- Provide source fixtures that prove representative SDK calls generate the expected Conditions.

An SDK or production library author MUST NOT use package metadata to extract or emit:

- Secret values
- Credentials
- Customer data
- Protected data
- Concrete target-environment values
- Account-specific resource identifiers unless the profile extension explicitly permits them as non-secret requirements

The package metadata should describe workload requirements, not discovered deployment state.

## 3.1 Release-specific mapping input

The maintainer-facing input for a versioned SDK mapping should contain only facts owned by that SDK release: public factory and method symbols, the canonical service operation selected by each call, the source parameter or typed configuration field that supplies each extension binding, state returned by factories or methods, and whether that state starts a new dependency identity or inherits an existing one. Fixed Condition templates belong to the service mapping and MUST NOT be copied into each language overlay.

Language tooling SHOULD accept named source parameters and compile them into the calling forms required by its profiler. For example, the NATS Python projector validates `subject` against the pinned method signature and generates both its positional index and keyword name. Maintainers review the stable parameter name, not a fragile hand-counted integer. Repeated methods with identical semantics MAY use a clearly named group, but grouping MUST expand to independent method mappings and MUST NOT collapse distinct service operations into one operation with many parameter combinations.

SDK object flow can require mapped state even when a method does not itself emit a Condition. A connection factory can create a new dependency identity; a method returning a service context can inherit that identity; a method returning a bucket-bound object can retain a source-proven bucket value; and later calls can bind from that retained state. These constructs are language-level analysis capabilities. The profiler MUST receive every SDK-specific class, field, method, and operation name from version-aligned metadata rather than embedding those names in generic profiler code.

The generator and source validator are Runtime Conditions tooling responsibilities, not files SDK maintainers should rewrite per release. The maintainer reviews relevant semantic bridge or SDK annotation changes, classifications for newly introduced public behavior, and representative source-to-profile changes. The release ships one generated `runtimeconditions.sdk-mapping.yaml` plus a small `runtimeconditions/index.yaml` at the package's conventional metadata location. Static metadata MUST NOT add imported runtime code, initialization behavior, a Runtime Conditions runtime dependency, or application configuration.

---

# 4. Modeling Internal Conditions

SDK authors should model stable integration requirements, not every low-level method call.

For example, this SDK call:

```go
client := s3.NewFromConfig(s3.Config{})
_, err := client.PutObject(ctx, &s3.PutObjectInput{
	Bucket: &bucketName,
	Key:    &objectKey,
	Body:   body,
})
```

should normally produce one Condition:

```yaml
conditions:
  - name: s3-object-store
    kind: aws.object_store
    interface:
      type: aws.s3
      bucketClass: standard
```

It should not emit the runtime bucket name from the application variable. The bucket name is a concrete fulfillment choice unless the extension explicitly defines it as a requirement field.

## 4.1 Condition Granularity

SDK authors SHOULD prefer one Condition per required external integration surface.

Good examples:

- `aws.object_store` for S3 object storage usage
- `aws.rds` for RDS database usage
- `stripe.payments_api` for Stripe payment API usage
- `twilio.messaging_api` for Twilio messaging API usage

SDK authors SHOULD NOT create a separate Condition for every SDK method unless each method has materially different runtime requirements.

## 4.2 Stable Names

Package manifests MAY assign stable default Condition names.

Example:

```yaml
declarations:
  - receiver: Client
    method: PutObject
    name: s3-object-store
    kind: aws.object_store
    interfaceType: aws.s3
```

If an SDK supports multiple configured clients with different runtime requirements, the SDK author SHOULD provide a visible source-level naming mechanism that the generator can read statically.

---

# 5. Extension Ownership

An SDK author that introduces vendor-specific vocabulary should define an extension.

Example AWS object store extension:

```yaml
apiVersion: runtimeconditions.io/v1alpha1
kind: RuntimeConditionsExtensionDefinition

metadata:
  id: https://aws.example.com/runtimeconditions/object-store/v1alpha1/runtimeconditions.extension.yaml

spec:
  kinds:
    - name: aws.object_store

  interfaceTypes:
    - name: aws.s3
      targetKind: aws.object_store
```

An extension can be used directly by profiles and adapters without any SDK package. An SDK package that wants generation support must leverage an extension by referencing it from its package manifest; the package manifest does not define vocabulary by itself.

The SDK-owned extension definition should declare any first-party or third-party dependencies it relies on. For example, an AWS RDS extension that reuses common datastore vocabulary and environment configuration should declare dependencies on:

```yaml
spec:
  dependencies:
    - https://runtimeconditions.io/extensions/common-integrations/v1alpha1/runtimeconditions.extension.yaml
    - https://runtimeconditions.io/extensions/env-configuration/v1alpha1/runtimeconditions.extension.yaml
```

The SDK package does not need to vendor every dependency extension file. The SDK package is the source of the direct extension definition used for SDK extraction; that definition's dependency identifiers are then resolved by validators, generators, or adapters from their configured package, cache, registry, or development sources.

---

# 6. Package Manifest Role

The package manifest connects SDK source symbols to extension vocabulary.

Example:

```yaml
apiVersion: runtimeconditions.io/v1alpha1
kind: RuntimeConditionsPackage

metadata:
  package: github.com/runtimeconditions/spec/examples/sdks/aws-sdk-go-v2/service/s3
  language: go

extension:
  id: https://aws.example.com/runtimeconditions/object-store/v1alpha1/runtimeconditions.extension.yaml

go:
  importPath: github.com/runtimeconditions/spec/examples/sdks/aws-sdk-go-v2/service/s3
  package: s3

  constructors:
    - function: NewFromConfig
      receiver: Client

  declarations:
    - receiver: Client
      method: PutObject
      name: s3-object-store
      kind: aws.object_store
      interfaceType: aws.s3
      values:
        - target: interface.bucketClass
          value: standard
      configuration:
        env:
          - property: bucket
            name: AUDIT_LOG_BUCKET
          - property: region
            name: AWS_REGION
          - property: accessKeyId
            name: AWS_ACCESS_KEY_ID
            sensitive: true
          - property: secretAccessKey
            name: AWS_SECRET_ACCESS_KEY
            sensitive: true
```

This manifest says:

- The Go import path owns a package manifest.
- The manifest uses the AWS object store extension.
- `NewFromConfig` creates a `Client`.
- Calls to `Client.PutObject` imply an `aws.object_store` Condition with interface type `aws.s3`.
- The generated Condition should include `interface.bucketClass: standard`.
- The generated Condition should declare the environment variable names the workload expects for S3 connection and credential properties.

The manifest does not provide values for `AUDIT_LOG_BUCKET`, `AWS_REGION`, `AWS_ACCESS_KEY_ID`, or `AWS_SECRET_ACCESS_KEY`. It only maps workload-facing environment variable names to extension-defined properties. A platform adapter is responsible for satisfying those properties from provider outputs such as ConfigMaps, Secrets, service bindings, cloud identity mechanisms, or generated credentials.

---

# 7. Source Fixtures

SDK authors SHOULD include source fixtures that demonstrate the expected mapping.

Example fixture:

```go
package fixture

import (
	"context"

	"github.com/runtimeconditions/spec/examples/sdks/aws-sdk-go-v2/service/s3"
)

func write(ctx context.Context) error {
	client := s3.NewFromConfig(s3.Config{})
	_, err := client.PutObject(ctx, &s3.PutObjectInput{})
	return err
}
```

Expected generated profile fragment:

```yaml
extensions:
  - https://aws.example.com/runtimeconditions/object-store/v1alpha1/runtimeconditions.extension.yaml

conditions:
  - name: s3-object-store
    kind: aws.object_store
    interface:
      type: aws.s3
      bucketClass: standard
    configuration:
      env:
        - property: bucket
          name: AUDIT_LOG_BUCKET
        - property: region
          name: AWS_REGION
        - property: accessKeyId
          name: AWS_ACCESS_KEY_ID
          sensitive: true
        - property: secretAccessKey
          name: AWS_SECRET_ACCESS_KEY
          sensitive: true
```

These fixtures are important because package manifests are executable only through generator behavior. A manifest that validates structurally but never maps real SDK source code is not useful.

---

# 8. SDK Author Checklist

Before publishing Runtime Conditions metadata, SDK authors SHOULD verify:

- The package includes `runtimeconditions.package.yaml`.
- The package includes `runtimeconditions.extension.yaml` next to the manifest.
- The mapping references the selected service mapping and exact semantic digest.
- Service operations come from a pinned authoritative model or one reviewed `service-operations-inventory.yaml`, never from duplicated per-language tables.
- The extension identifier in the manifest matches `metadata.id` in the extension file.
- The extension declares all vocabulary dependencies.
- Any manifest `configuration` shape is defined by a declared extension dependency.
- The manifest maps real SDK symbols, not internal implementation details that users never call.
- Generated Conditions do not contain secrets or concrete target-environment values.
- Repeated calls deduplicate or aggregate into stable Conditions according to the generator's documented behavior.
- Source fixtures generate the expected profile fragments.
- Unsupported SDK features fail silently only when documented, or produce actionable diagnostics when a manifest is malformed.

---

# 9. Current Demo

This repository contains a minimal example SDK package at:

```text
examples/sdks/aws-sdk-go-v2/service/s3/
```

It includes:

```text
client.go
runtimeconditions.package.yaml
```

The manifest references the canonical example extension at:

```text
examples/extensions/aws-object-store/aws-object-store-v1alpha1.yaml
```

An example workload can import the SDK normally and call `Client.PutObject`. The Go generator discovers the package manifest from that import, loads the referenced extension definition, and emits an `aws.object_store` Condition into the generated Runtime Conditions Profile.
