# Extension Authoring Guide

## Status

**Non-normative implementation guidance**

This guide documents the repository convention for writing Runtime Conditions extension definitions. The core profile draft defines the extension document shape. This guide explains how extension authors should use that shape so extensions compose cleanly. For a keyword-by-keyword vocabulary reference, see [Extension Vocabulary Keywords](extension-vocabulary-keywords.md).

---

# 1. Adapter-Actionable Minimum

The adapter-actionable minimum is the highest-priority design constraint for extension authors. Begin with what a downstream Adapter must decide, then work backward to the smallest portable Condition vocabulary that supports that decision. Do not begin with everything an API model, SDK, library, framework, configuration file, or code analyzer can expose.

An extension is not a Service Operations Inventory, SDK reference, API description, or source-analysis evidence file. It is a semantic contract between a workload requirement and consumers that can fulfill, validate, govern, or report on that requirement. Every additional distinction increases profile size, authoring burden, validation complexity, versioning pressure, and the amount of meaning adopters must understand.

## 1.1 Inclusion Test

Before adding a kind, interface type, field, field value, operation coordinate, or validation rule, answer these questions:

1. Does the detail describe an external runtime requirement rather than internal application behavior?
2. Can a credible Adapter use it to make a materially different provisioning, service-selection, authorization, network-policy, configuration, deployment, validation, or governance decision?
3. Is the distinction portable across environments and independent of one SDK's naming or implementation?
4. Is this the least detailed stable representation that supports the decision?
5. Is there an authoritative semantic source or a clearly documented extension-owner decision for the distinction?

If the answer to the first three questions is not yes, omit the detail from the extension. If a coarser representation supports the same Adapter outcome, prefer the coarser representation. Do not add vocabulary for possible future consumers without documenting the concrete decision that vocabulary is expected to enable.

Static detectability is not an inclusion requirement. An extension may need to represent a valid requirement that a particular profiler or SDK mapping cannot infer. In that case, an application developer may use extension-provided declarative bindings or a project-local override. Conversely, easy detection does not justify adding a detail that is not adapter-actionable.

## 1.2 Appropriate Granularity

The appropriate minimum differs by integration:

- A relational datastore may need `engine: postgres` because it changes the service or runtime selected by an Adapter. Individual SQL client methods usually do not belong in the extension unless they support a concrete, portable database-access policy that the extension intentionally owns.
- An HTTP API may need only an endpoint requirement. Method and path are appropriate when an Adapter uses them to configure an API gateway, egress policy, mock, or allowlist. Request and response schemas should be included only when an Adapter actually consumes them for fulfillment or validation; their presence in OpenAPI is not sufficient justification.
- An environment-configuration extension may identify that an application expects a URL, hostname, token, or other property through a source-proven environment variable. It should not invent a vendor's conventional variable names merely because an SDK supports them.
- An S3 extension may distinguish `PutObject`, `GetObject`, and other canonical service operations because those distinctions can change authorization. A source/destination role may be justified for an operation such as `CopyObject` when the two resources require different permissions. SDK method signatures, paginator classes, retry behavior, and HTTP request structures remain outside the extension.
- A Kubernetes extension may retain verb, API group, API version, plural resource, scope, and optional subresource because those coordinates directly affect RBAC and policy. Generated Python class names, watch-wrapper implementation, discovery cache behavior, and OpenAPI Generator identifiers do not belong in the extension.

Operation-level vocabulary is therefore optional, not a maturity level that every extension must reach. Include operations only when different operations lead to different Adapter behavior. A service with hundreds of API operations may legitimately expose a handful of adapter-facing capabilities, while a permission-sensitive service may need canonical operation names. One detailed extension is not a template requiring equivalent depth elsewhere.

## 1.3 SDK and Generated-Model Context

An SDK mapping aligns SDK-owned public behavior to an existing extension release. It does not define the extension's semantic depth.

SDK and service-model integrations commonly contain three different amounts of information:

1. The authoritative service or protocol model may enumerate every operation and request shape.
2. The SDK mapping may enumerate many language symbols, aliases, factories, wrappers, generated methods, and state transitions needed to recognize application usage.
3. The extension should retain only the canonical distinctions that an Adapter needs, and the generated profile should contain only those Conditions proven for the application.

These layers need not be similar in size. Hundreds of SDK methods may collapse into a few extension capabilities. A generated SDK mapping may be large while its extension and application profiles remain compact. Generated artifact size should be measured for packaging and performance, but it is not by itself evidence that the extension vocabulary is too deep.

When an SDK investigation identifies a distinction absent from the extension, first ask whether that distinction changes a portable Adapter decision. If it does, revise and version the extension before mapping the SDK behavior. If it does not, keep the information as mapping-generation evidence or omit it from emitted Conditions. Never add extension vocabulary merely to make an SDK mapping lossless.

An SDK mapping should describe only behavior proven by an authoritative service model or pinned SDK source, and a profiler consumer should emit only Conditions proven by application source. When required coordinates are dynamic or available only through runtime discovery, the generator should emit no inferred Condition rather than substitute a broader requirement. The extension may still support explicit declarative use of that vocabulary.

### 1.3.1 Service Operation Sources and Semantic Bridges

SDK authors need one language-neutral operation authority before they can map language symbols to extension Conditions. When a provider publishes an adequate machine-readable service model such as Smithy, OpenAPI, protobuf, or another protocol definition, generators should reference and project that source rather than ask maintainers to reproduce its operation inventory.

When no adequate authoritative machine-readable operation source exists, service stakeholders should maintain one `service-operations-inventory.yaml`. The Service Operations Inventory records only a cohesive result set of stable service operations, service-owned inputs and requiredness, and service-domain value shapes. It is a language-neutral service artifact shared by all SDK languages and releases, not an SDK-specific overlay, and it must not contain Runtime Conditions semantics.

The extension repository maintains one `service-operations-semantic-bridge.yaml` for the extension. The bridge references the authoritative Smithy, OpenAPI, protobuf, or other provider artifact when one exists and references an exact Service Operations Inventory only as the fallback. It contains the minimum reviewed decisions needed to translate source operations and inputs into adapter-actionable Condition semantics. This separation keeps service facts reusable outside Runtime Conditions and keeps the published extension free of an authoring paper trail.

The operation source and semantic bridge compile into a deterministic extension and language-neutral service mapping with exact extension coordinates and semantic digests. SDK mappings reference that generated mapping and add only language-owned symbols, bindings, state, delegation, and release identity. The compiler can generate the operation-oriented portions of an extension when the bridge contains every adapter-facing semantic decision required for them; generation automates expansion and validation, not the human decision about what an operation proves.

A source operation added without a new adapter distinction may update the bridge and service mapping while leaving the extension vocabulary unchanged. Any SDK release that exposes the operation still requires its generated SDK method mapping to be updated or regenerated so the public symbol points to the canonical service operation and existing Condition. If a new operation requires a new Condition distinction, the extension must be revised and versioned first.

Source operations remain subject to the adapter-actionable minimum when translated by the bridge. A fallback inventory must not become a handwritten substitute for an exhaustive upstream API model, and the bridge must not reproduce source details that do not change an Adapter decision. Conversely, the bridge must retain separate translations or multiple Condition templates when collapsing them would lose authorization, provisioning, policy, or other adapter-actionable meaning.

The [Service Operation Authoring Guide](service-operation-authoring.md) provides the complete source-selection tutorial, end-to-end NATS example, artifact ownership boundaries, and change-propagation model. The [SDK Integration Guide](sdk-integration-guide.md#2-service-semantic-authority) defines the SDK-facing responsibilities and packaging relationship.

## 1.4 Non-SDK Sources

The same minimum applies to declarative code packages, no-op bindings, configuration-file analysis, infrastructure definitions, protocol schemas, API descriptions, and manual profile authoring. These sources can contain far more detail than an Adapter needs. Their generators should project source evidence into extension vocabulary rather than reproduce the source format in the profile.

For example, a connection string parser may observe username, password, host, port, database, query parameters, TLS options, driver flags, and pool settings. An extension should expose only the properties required for portable fulfillment. Secret values, concrete environment values, driver tuning, and application-local pool behavior remain outside the Condition even though the parser can observe them.

## 1.5 Review and Size Discipline

Human review should focus on the authored semantic contract and the Adapter decisions it enables. Exhaustive service operation inventories, generated validation expansions, and SDK symbol mappings should be deterministic artifacts accompanied by concise semantic summaries; extension maintainers should not be asked to approve them line by line.

Compact serialization is still desirable. Repeated schema branches or enumerations should be factored when equivalent validation can be expressed more simply. However, reducing file size is not a reason to discard a distinction that materially changes safe fulfillment, and a small file is not evidence that every field is necessary.

For every extension-owned field or operation dimension, its authoring documentation should state:

- The Adapter decision enabled by the distinction
- Why a coarser representation is insufficient
- The authoritative semantic source or ownership decision
- What related source, SDK, or protocol details are intentionally excluded

# 2. Ownership

An extension definition owns only the vocabulary it introduces.

An extension MAY define:

- Condition kinds
- Interface types
- Condition fields
- Interface fields
- Allowed field values
- JSON Schema validation rules

An extension MUST NOT redefine vocabulary owned by another extension. If it needs to build on that vocabulary, it must declare a dependency and reference the dependency-owned kind, interface type, or field in scoped definitions.

For first-party tooling support, the resolved extension set must contain exactly one definition for each vocabulary item in its scope. Two definitions conflict even if they are textually identical.

The scope of a vocabulary item is part of its identity:

- A kind is scoped by its `name`.
- An interface type is scoped by `targetKind` and `name`.
- An interface field is scoped by `targetKind`, `targetType`, and `name`.
- A condition field is scoped by `name`, `appliesToKinds`, and `appliesToInterfaceTypes`.
- A field value definition is scoped by `field`, `targetKind`, and `targetType`.

Condition fields with the same `name` can coexist only when their kind/type scopes do not overlap. A kind-wide condition field overlaps every narrower interface-type condition field for that kind. The same field name can be reused for unrelated kinds, or for distinct interface types under the same kind when neither definition is kind-wide.

---

# 3. Base Extensions

A base extension introduces vocabulary directly.

The Common Integrations extension owns common application integration vocabulary:

```yaml
apiVersion: runtimeconditions.io/v1alpha1
kind: RuntimeConditionsExtensionDefinition

metadata:
  id: https://runtimeconditions.io/extensions/common-integrations/v1alpha1/runtimeconditions.extension.yaml

spec:
  kinds:
    - name: api
    - name: datastore
    - name: cache

  interfaceTypes:
    - name: http
      targetKind: api
    - name: relational
      targetKind: datastore
    - name: document
      targetKind: datastore
    - name: key_value
      targetKind: cache
```

Because this extension owns `api`, `datastore`, `cache`, `http`, `relational`, `document`, and `key_value`, other extensions must not redefine them.

---

# 4. Additive Extensions

An additive extension builds on dependency-owned vocabulary without copying it.

The Environment Configuration extension adds the `configuration` field to common integration Conditions. It does not redefine common's kinds or interface types:

```yaml
apiVersion: runtimeconditions.io/v1alpha1
kind: RuntimeConditionsExtensionDefinition

metadata:
  id: https://runtimeconditions.io/extensions/env-configuration/v1alpha1/runtimeconditions.extension.yaml

spec:
  dependencies:
    - https://runtimeconditions.io/extensions/common-integrations/v1alpha1/runtimeconditions.extension.yaml

  conditionFields:
    - name: configuration
      appliesToKinds:
        - api
      appliesToInterfaceTypes:
        - http
    - name: configuration
      appliesToKinds:
        - datastore
      appliesToInterfaceTypes:
        - relational
        - document
    - name: configuration
      appliesToKinds:
        - cache
      appliesToInterfaceTypes:
        - key_value
```

The dependency makes the referenced common vocabulary available. The additive
extension owns `configuration` only in the listed kind and interface-type scopes,
plus the rules for values inside that field.

---

# 5. Field Values

Use `fieldValues` to define allowed values for extension-owned fields in a specific vocabulary scope.

```yaml
fieldValues:
  - field: configuration.env[].property
    targetKind: api
    targetType: http
    values:
      - url
      - baseUrl
      - hostname
      - port
      - scheme
      - username
      - password
      - token
      - tls

  - field: configuration.env[].property
    targetKind: cache
    targetType: key_value
    values:
      - url
      - hostname
      - port
      - scheme
      - username
      - password
      - database
      - token
      - tls
```

The field path is owned by the additive extension. The target kind and interface type may come from the same extension or from a declared dependency.

For first-party tooling support, a `fieldValues.field` path must resolve to a field that is valid in the declared `targetKind` and optional `targetType` scope:

- Paths beginning with `interface.` must target an `interfaceFields` entry for that exact `targetKind` and `targetType`, except `interface.type`, which targets an interface type value.
- Paths beginning with a condition field name, such as `configuration.` or `trust.`, must target a `conditionFields` entry with the same name whose scope includes the declared `targetKind` and `targetType`.
- `targetType` must identify an interface type that is valid for `targetKind` when `targetType` is present.
- Values inside one `fieldValues` entry must be unique.

This allows unusual field paths such as `access.identity.kind`, `validation.nameConstraints.mode`, `interface.revocation.method`, or `interface.flows[].grantType`, as long as the first path segment is owned and scoped by the extension or one of its dependencies.

---

# 6. Schemas

Extension schemas should validate the fields owned by that extension and leave unrelated fields open.

```yaml
schemas:
  - id: configuration-shape
    description: Validates the environment configuration field shape.
    schema:
      $schema: https://json-schema.org/draft/2020-12/schema
      type: object
      properties:
        configuration:
          type: object
          oneOf:
            - required:
                - env
            - required:
                - alternatives
      additionalProperties: true
```

Use `appliesToKind` and `appliesToInterfaceType` when a schema is valid only for one target scope:

```yaml
schemas:
  - id: configuration-properties-cache-key-value
    appliesToKind: cache
    appliesToInterfaceType: key_value
    description: Validates allowed environment configuration properties for key/value cache integrations.
    schema:
      type: object
      properties:
        configuration:
          type: object
      additionalProperties: true
```

Schemas from multiple resolved extensions apply additively. A Condition must satisfy every schema whose scope matches it.

For first-party tooling support, `schemas[].appliesToKind` must resolve to exactly one kind in the resolved extension set. If `schemas[].appliesToInterfaceType` is present, it must resolve to exactly one interface type for `schemas[].appliesToKind`.

---

# 7. Declarative Code Packages

Declarative code packages should mirror extension ownership. The examples in this section use Go because that is the current first-party implementation.

A base extension package exports the declarations and option types for the vocabulary it owns:

```go
package commonintegrations

type APIOption interface {
	CommonIntegrationsAPIOption()
}

type CacheOption interface {
	CommonIntegrationsCacheOption()
}

func API(name string, options ...APIOption) Declaration {
	return Declaration{}
}

func Cache(name string, options ...CacheOption) Declaration {
	return Declaration{}
}
```

An additive extension package exports only its own options. It may import a base package so its options can satisfy the base package's marker interfaces:

```go
package envconfiguration

import common "github.com/runtimeconditions/extensions/common-integrations/go"

type ConditionOption interface {
	common.APIOption
	common.DatastoreOption
	common.CacheOption
	envConfigurationConditionOption()
}

type conditionOption struct{}

func (conditionOption) CommonIntegrationsAPIOption()       {}
func (conditionOption) CommonIntegrationsDatastoreOption() {}
func (conditionOption) CommonIntegrationsCacheOption()     {}
func (conditionOption) envConfigurationConditionOption()   {}

func Env(property, name string, options ...EnvOption) ConditionOption {
	return conditionOption{}
}
```

Application code imports both packages when it uses both extensions:

```go
import (
	common "github.com/runtimeconditions/extensions/common-integrations/go"
	env "github.com/runtimeconditions/extensions/env-configuration/go"
)

var _ = common.Cache("request-cache",
	common.KeyValue(common.Redis),
	env.EnvAlternative(env.Env("url", "REDIS_URL")),
)
```

The generated profile lists both extensions because both packages directly contributed vocabulary:

```yaml
extensions:
  - https://runtimeconditions.io/extensions/common-integrations/v1alpha1/runtimeconditions.extension.yaml
  - https://runtimeconditions.io/extensions/env-configuration/v1alpha1/runtimeconditions.extension.yaml
```

If a workload imports only `common-integrations/go`, the profile lists only `common-integrations`. If it imports `env-configuration/go` but does not apply an env option to a Condition, the profile does not list `env-configuration`.

Adapters and validators still resolve transitive extension dependencies from extension definitions. Direct profile declarations and dependency resolution are separate steps.

---

# 8. Declarative Code Package Bindings

An extension-side declarative code package uses `runtimeconditions.bindings.yaml`.
That binding manifest maps source calls back to extension-owned profile
vocabulary.

For first-party tooling support:

- `runtimeconditions.bindings.yaml` must use `kind: RuntimeConditionsBinding`.
- `metadata.extension` must match the extension definition `metadata.id`.
- The package should include `runtimeconditions.extension.yaml` next to the binding manifest, unless the extension definition is intentionally vendored elsewhere in the same package artifact or supplied by a local development override.
- `metadata.extensionDefinition` is a vendored or local development override; when present, it must resolve to the extension definition file.
- `metadata.language` must identify the language section used by the binding.
- The language-specific package identity fields are required.
- At least one language-specific declaration or option mapping is required.

This Go example shows a base declaration package mapping source calls to Conditions. The same manifest structure is intended for future language sections, with the language-specific mapping stored under that language's section.

```yaml
apiVersion: runtimeconditions.io/v1alpha1
kind: RuntimeConditionsBinding

metadata:
  extension: https://runtimeconditions.io/extensions/common-integrations/v1alpha1/runtimeconditions.extension.yaml
  language: go

go:
  importPath: github.com/runtimeconditions/extensions/common-integrations/go
  package: commonintegrations

  declarations:
    - function: Cache
      nameArg: 0
      kind: cache
      options:
        - function: KeyValue
          target: interface.type
          value: key_value
          engineArg: 0
```

This Go example shows an option-only extension mapping source calls to fields that can augment compatible Conditions:

```yaml
apiVersion: runtimeconditions.io/v1alpha1
kind: RuntimeConditionsBinding

metadata:
  extension: https://runtimeconditions.io/extensions/env-configuration/v1alpha1/runtimeconditions.extension.yaml
  language: go

go:
  importPath: github.com/runtimeconditions/extensions/env-configuration/go
  package: envconfiguration

  options:
    - function: Env
      target: configuration.env[]
      appliesToKinds:
        - api
        - datastore
        - cache
      stringArgs:
        property: 0
        name: 1
```

Generators use language-specific option mappings only when an option call appears inside a compatible declaration call. Standalone option calls are ignored for profile emission.

A declaration entry must map to extension-resolved vocabulary:

- `kind` must resolve to exactly one kind.
- `interfaceType`, when present, must resolve to exactly one interface type for `kind`.
- `values` must target supported binding targets and defined field values.
- A declaration may name either a package-level `function` or a receiver `method`.

A package-level option entry must map to fields that are valid for the declaration scope or for the option's `appliesToKinds` and `appliesToInterfaceTypes` scope. For configuration-style targets, the target field and the configured property field must both be defined in that same scope.

The declarative code package must match the binding manifest. For Go packages, that means:

- The parsed Go package name must match `go.package`.
- Every manifest constant must exist in Go and have the same string value.
- Every package-level declaration, option, and constructor function named in the manifest must exist.
- Every receiver method named in the manifest must exist on the named receiver type.
- `nameArg` and entries in `stringArgs` must point to existing `string` parameters.
- `valueArg` and `engineArg` must point to existing parameters.
- `typeArg` must point to an existing type parameter.
- Nested options are validated recursively.

For Java packages, that means:

- The Java package declaration must match `java.package`.
- Every Java declaration and option entry must include the class that owns the static method.
- Every manifest constant must exist in the referenced Java class and have the same string value.
- Every declaration and option function named in the manifest must exist as a public static method on the referenced class.
- `nameArg` and entries in `stringArgs` must point to existing `String` parameters.
- `classArg` must point to an existing `Class` parameter.
- Nested options are validated recursively.

Use the static validator before treating an extension as first-party tooling-ready:

```sh
cd go-rc-profiler
go run . validate-extension -root ../extensions/env-configuration
go run . validate-extension -root ../extensions/env-configuration -language java
```

The validator loads the target extension definition, resolves dependency extension definitions from the provided package or development roots, checks the binding manifest against the resolved vocabulary, and checks that the declarative code package contains the language symbols, constants, and argument positions named by the binding manifest.

When validating a single extension directory in this repository, sibling extension directories are included as a local development convenience. Published packages should not depend on repository sibling layout; they should package their own extension definition and declare exact dependency identifiers.

---

# 9. Authoring Checklist

- Apply the adapter-actionable minimum before defining vocabulary or validation rules.
- Document the materially different Adapter decision enabled by each field or operation dimension.
- Prefer a coarser stable representation when it enables the same safe fulfillment behavior.
- Do not mirror an API model, SDK surface, library abstraction, configuration format, or analyzer output into extension vocabulary.
- Keep SDK symbols, wrapper structure, execution paths, retry behavior, and other recognition evidence in SDK mappings rather than Conditions.
- Allow service-specific operation detail only when it changes authorization, provisioning, policy, configuration, or another concrete Adapter outcome.
- Define only vocabulary your extension owns.
- Declare dependencies for vocabulary you reference but do not own.
- Scope additive fields to the dependency-owned kinds and interface types they augment.
- Avoid condition field scope overlap unless the definitions are intentionally the same owned definition.
- Use `fieldValues` for portable, adapter-visible enums.
- Make every `fieldValues.field` path resolve to a condition or interface field in the declared target scope.
- Use JSON Schema for machine-readable validation.
- Scope schemas only to kinds and interface types that resolve in the extension's dependency graph.
- Keep schemas focused on your extension's fields and allow unrelated properties.
- Export only declaration functions for vocabulary your package owns.
- For additive declarative packages, export helper APIs that are compatible with the base declaration package contracts.
- Describe additive options with binding-level language mappings such as `go.options`.
- Keep `runtimeconditions.bindings.yaml` identity, package names, symbols, constants, and argument indexes synchronized with each supported language package.
- Run `go run . validate-extension -root <extension-dir>` before publishing first-party tooling support.
- Do not encode secrets, concrete target-environment values, or provider-specific fulfillment choices.
