# Runtime Conditions Profile Specification (Draft)

## Status

**Draft - Request for Comments**

This document defines the core Runtime Conditions Profile format.

First-party extension drafts define common integration vocabulary separately:

- `https://runtimeconditions.io/extensions/common-integrations/v1alpha1/runtimeconditions.extension.yaml`
- `https://runtimeconditions.io/extensions/env-configuration/v1alpha1/runtimeconditions.extension.yaml`

---

# 1. Purpose

A Runtime Conditions Profile declares the external runtime integrations required by one application workload.

The profile describes requirements. It does not describe implementations, provisioning actions, deployment topology, credentials, secret values, or concrete target-environment values.

---

# 2. Scope

This specification defines:

- The Runtime Conditions Profile document shape
- Workload identity fields
- Profile metadata labels
- The core Condition object shape
- Extension declaration and resolution rules
- Extension definition structure
- Validation layers
- Conformance requirements

This specification does not define concrete Condition vocabulary. Condition kinds, interface types, field values, and type-specific fields are defined by extensions.

Concerns specifically beyond the scope of this specification include:

- Infrastructure provisioning behavior
- Platform-specific resource models
- Deployment topology
- Runtime configuration values
- Secret material
- Internal workload behavior
- Observability or code-analysis mechanisms used to generate a profile

---

# 3. Profile Document

Profiles MAY be serialized as YAML or JSON.

First-party Runtime Conditions tooling MUST use YAML as its serialization standard and MUST emit YAML for generated profiles, extension artifacts, SDK integration metadata, indexes, manifests, state, and evidence. A consumer MAY accept a JSON representation of a profile for interoperability, but first-party generators MUST NOT create a parallel JSON artifact contract. JSON Schema used inside an extension definition is a validation vocabulary and does not change the YAML serialization requirement.

Serialized profiles are invalid if:

- A mapping contains duplicate keys
- A required field is missing
- A required field is present with `null`
- A field has a type other than the type defined by this specification

Optional fields SHOULD be omitted when unused.

## 3.1 Complete Profile Example

```yaml
apiVersion: runtimeconditions.io/v1alpha1
kind: RuntimeConditionsProfile

metadata:
  name: checkout-service
  labels:
    owner.example.com/team: payments
    lifecycle.example.com/stage: production

workload:
  uri: https://github.com/example-org/checkout-service
  version: v1.2.3

extensions:
  - https://runtimeconditions.io/extensions/common-integrations:v1alpha1
  - https://runtimeconditions.io/extensions/env-configuration:v1alpha1

conditions:
  - name: primary-db
    kind: datastore
    interface:
      type: relational
      engine: postgres
    configuration:
      env:
        - property: hostname
          name: POSTGRES_HOST
        - property: port
          name: POSTGRES_PORT
        - property: database
          name: POSTGRES_DATABASE
        - property: username
          name: POSTGRES_USERNAME
        - property: password
          name: POSTGRES_PASSWORD
          sensitive: true

  - name: payments-api
    kind: api
    interface:
      type: http
      operations:
        - method: POST
          path: /charge
    configuration:
      env:
        - property: baseUrl
          name: PAYMENTS_API_URL
        - property: token
          name: PAYMENTS_API_TOKEN
          sensitive: true
```

## 3.2 Top-Level Fields

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `apiVersion` | string | YES | Runtime Conditions Profile API version |
| `kind` | string | YES | Document kind |
| `metadata` | object | YES | Profile metadata |
| `workload` | object | YES | Workload identity |
| `extensions` | array | YES | Extension identifier strings required by the profile |
| `conditions` | array | YES | Runtime Conditions declared by the workload |

`apiVersion` MUST be:

```text
runtimeconditions.io/v1alpha1
```

`kind` MUST be:

```text
RuntimeConditionsProfile
```

`extensions` MAY be empty if and only if the profile uses no extension-defined vocabulary.

`conditions` MAY be empty if and only if the profile is intended to explicitly declare that the workload has no external runtime integration requirements.

## 3.3 Metadata

```yaml
metadata:
  name: checkout-service
  labels:
    owner.example.com/team: payments
    lifecycle.example.com/stage: production
```

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `name` | string | YES | Human-readable profile name |
| `labels` | object | NO | Machine-readable profile and workload classification labels |

`metadata.name` MUST be a non-empty string.

`metadata.labels`, when present, MUST be a string-to-string mapping.

Label keys MUST be non-empty strings.

Label values MUST be strings.

Label keys SHOULD be namespaced:

```text
<namespace>/<name>
```

Examples:

- `compliance.example.com/hipaa`
- `owner.example.com/team`
- `lifecycle.example.com/stage`
- `risk.example.com/criticality`

The `runtimeconditions.io/` label namespace is reserved for the Runtime Conditions core specification and first-party Runtime Conditions extensions.

Extensions MAY document label key conventions. Label keys are metadata. They are not Condition vocabulary and do not participate in extension resolution unless a future version of this specification defines otherwise.

Labels MAY be used for selection, filtering, policy, ownership, reporting, and lifecycle workflows.

Labels MUST NOT define runtime dependency requirements or change the meaning of Conditions.

Labels MUST NOT contain secrets, protected data, personal data, customer data, or concrete target-environment values.

## 3.4 Workload

```yaml
workload:
  uri: https://github.com/example-org/checkout-service
  version: v1.2.3
```

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `uri` | string | YES | Stable workload identifier |
| `version` | string | NO | Workload version described by the profile |

`workload.uri` MUST be a non-empty string.

`workload.version`, when present, MUST be a non-empty string.

A Runtime Conditions Profile MUST describe exactly one workload identity.

---

# 4. Condition Model

Each Condition represents one external runtime dependency requirement.

## 4.1 Condition Fields

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `name` | string | NO | Unique Condition name within the profile |
| `optional` | boolean | NO | Whether the Condition is optional. Defaults to `false` |
| `kind` | string | YES | Extension-defined integration classification |
| `extension` | string | NO | Extension identifier selecting the extension defining this Condition's `kind`, when ambiguous |
| `interface` | object | YES | Workload-facing interface requirement |

Condition shape:

```yaml
conditions:
  - name: primary-db
    optional: false
    kind: datastore
    interface:
      type: relational
```

`conditions[].name`, when present, MUST be a non-empty string and MUST be unique within the profile.

`conditions[].optional`, when omitted, MUST be interpreted as `false`.

`conditions[].kind` MUST be a non-empty string and MUST be defined by at least one resolved extension.

`conditions[].extension`, when present, MUST be a non-empty string exactly equal to one item in the profile's `extensions` array, and that extension MUST define `conditions[].kind`.

When more than one resolved extension defines `conditions[].kind`, `conditions[].extension` selects which definition this Condition resolves against. When `conditions[].extension` is absent, the Condition resolves against the first matching definition in extension resolution order.

`conditions[].interface` MUST be an object.

Extensions MAY define additional Condition fields.

## 4.2 Interface Fields

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `type` | string | YES | Extension-defined interface type for the declared `kind` |

`conditions[].interface.type` MUST be a non-empty string and MUST be defined by exactly one resolved extension for the declared `kind`.

Extensions MAY define additional `interface` fields.

---

# 5. Extensions

Profiles that use extension-defined vocabulary MUST declare the required extensions.

First-party extensions are extensions. They are not core vocabulary and are not implicit.

Implementations MAY bundle support for first-party extensions, but generated profiles MUST still declare the vocabulary they use.

## 5.1 Extension Identifiers and Versions

An extension identifier MUST be a non-empty string. It SHOULD be resolvable by tooling, using a form such as an HTTPS URI, a file URI, or an OCI reference. It SHOULD identify immutable content, but immutability is not required.

An extension reference MAY append a version to its identifier in the form `<identifier>:<version>`. The version MAY use any non-empty string form, including `v1alpha`, `v1.1.1`, or a SHA digest. The version suffix is optional. Structural validation MUST treat each reference as a string and MUST NOT require URI syntax or assign meaning to a suffix.

Extension reference strings are case-sensitive and MUST NOT be trimmed, case-folded, or URI-normalized for equality. Tooling MAY resolve an identifier through its URI, file, OCI, package, or catalog mechanisms.

Examples:

- `https://runtimeconditions.io/extensions/common-integrations:v1alpha1`
- `https://runtimeconditions.io/extensions/env-configuration`
- `file:///opt/runtimeconditions/extensions/common-integrations/v1alpha1/runtimeconditions.extension.yaml`
- `oci://ghcr.io/example/runtimeconditions/extensions/common-integrations:sha256-abc123`
- `example.common-integrations` (resolved through a configured catalog or mapping)

## 5.2 Extension Declarations

```yaml
extensions:
  - https://runtimeconditions.io/extensions/common-integrations:v1alpha1
  - https://runtimeconditions.io/extensions/env-configuration:v1alpha1
```

Each `extensions` item MUST be a non-empty string. The array MUST NOT contain duplicate strings. An item MAY be a bare identifier or an identifier with a version suffix.

Declared extensions MAY depend on other extensions.

Declared extensions and transitive dependencies form the resolved extension set.

Resolution, caching, dependency deduplication, and cycle detection MUST use the exact extension reference string. A resolver MAY use optional extension metadata to verify a resolved definition.

An identifier SHOULD be usable as a retrieval location when its format supports that. Tooling MAY also resolve it through configured catalogs, caches, package-local definitions, or explicit mappings. Resolution MUST report unsupported or unavailable identifiers.

Dependency resolution MUST be deterministic and MUST NOT depend on declaration order.

A profile is invalid if:

- A declared extension cannot be resolved
- A transitive dependency cannot be resolved
- Extension dependency resolution contains a cycle
- The resolved extension set contains a vocabulary definition conflict

---

# 6. Extension Definitions

Extensions are defined as independent artifacts.

## 6.1 Adapter-Actionable Minimum

The primary extension-authoring principle is the **adapter-actionable minimum**: an extension SHOULD define the smallest portable vocabulary that allows an Adapter to distinguish materially different fulfillment decisions.

A distinction is adapter-actionable when it can change how a target platform, catalog, policy system, configuration system, or deployment workflow fulfills a Condition. Examples include selecting a different class of external service, granting a different permission, applying a different network or gateway policy, or supplying a different category of workload configuration. Detail that merely describes how an application or library is implemented is not adapter-actionable.

An extension MUST NOT mirror an external API, protocol model, SDK surface, library abstraction, configuration format, or source-code structure solely because that detail is available. Each added kind, interface type, field, field value, operation dimension, and validation rule SHOULD be justified by a portable adapter decision that would be impossible, unsafe, or materially less precise without it.

The appropriate minimum is extension-specific. A datastore Condition may need only an engine distinction. An HTTP integration may need method and path when an Adapter uses them to configure an allowlist, but does not need them when endpoint access is the only actionable requirement. A service-specific extension may include canonical operations when they change authorization or provisioning behavior. One extension's valid operation-level depth does not require unrelated extensions to expose comparable detail.

SDK mappings, generated service mappings, static analyzers, declarative code packages, and other profile-generation inputs MAY contain substantially more detail than the extension vocabulary. They MUST NOT cause SDK class names, method names, wrapper structure, retry behavior, request-model fields, or other implementation details to become extension vocabulary. When such source detail reveals an adapter-actionable requirement, the extension MUST express that requirement using canonical service, protocol, or integration semantics independent of the originating SDK or tool. Multiple SDK methods MAY map to one extension-defined capability, while one SDK method MAY resolve to multiple Conditions when source evidence proves multiple external requirements.

SDK integration tooling SHOULD derive canonical service operations and adapter-actionable integration facts directly from an authoritative machine-readable service model when one exists. When no adequate authoritative operation model exists, service stakeholders SHOULD maintain one language-neutral Service Operations Inventory named `service-operations-inventory.yaml`. The inventory MUST form one cohesive service-operation result set and MUST NOT contain Runtime Conditions document concepts or language-specific SDK symbols.

An operation-oriented workflow MUST NOT require an additional translation document merely to restate facts that can be derived deterministically from the authoritative model or fallback inventory. When a specific fact needed to expose integration intent cannot be derived safely from that authority, extension and service stakeholders MAY maintain the smallest possible supplemental Service Operations Semantic Bridge named `service-operations-semantic-bridge.yaml`. Such a supplement MUST contain only the residual decisions or exceptions required for the projection, MUST NOT duplicate the source operation model, and MUST be omitted when there are no residual decisions. It is an authoring input, not Runtime Conditions vocabulary, runtime Adapter input, or SDK-maintainer packaging responsibility.

A service operation that maps to existing Condition vocabulary MAY change the generated service mapping without requiring a new extension release. An optional semantic supplement changes only when the affected projection depends on one of its residual decisions. Every SDK mapping for an SDK release that exposes the operation MUST still be updated or regenerated so its public symbol references the canonical service operation and existing Condition. A projection that requires new adapter-facing vocabulary MUST revise and version the extension before an SDK mapping targets it.

This principle applies equally when a profile is produced without an SDK mapping. Manually authored declarations, no-op bindings, configuration analysis, protocol specifications, API models, and other sources SHOULD project only the adapter-actionable requirement into a Condition. The detection mechanism does not determine the semantic depth of the extension.

The non-normative [Extension Authoring Guide](guides/extension-authoring.md#1-adapter-actionable-minimum) provides the detailed inclusion test, SDK and non-SDK examples, and review guidance for applying this principle. The [Service Operation Authoring Guide](guides/service-operation-authoring.md) explains direct projection from an authoritative source, the last-resort Service Operations Inventory, the narrowly optional semantic supplement, extension and service-mapping outputs, ownership boundaries, and change propagation. The [SDK Integration Guide](guides/sdk-integration-guide.md#2-service-semantic-authority) defines the SDK-facing authoring and packaging responsibilities.

## 6.2 Extension Definition Fields

```yaml
apiVersion: runtimeconditions.io/v1alpha1
kind: RuntimeConditionsExtensionDefinition

metadata:
  id: https://runtimeconditions.io/extensions/common-integrations/v1alpha1/runtimeconditions.extension.yaml
  version: v1alpha1

spec:
  kinds: []
```

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `apiVersion` | string | YES | Runtime Conditions API version |
| `kind` | string | YES | MUST be `RuntimeConditionsExtensionDefinition` |
| `metadata` | object | YES | Extension identity |
| `spec` | object | YES | Extension vocabulary, dependencies, and validation schemas |

Extension metadata has these identity fields:

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | YES | Non-empty extension identifier, as defined in Section 5.1 |
| `version` | string | NO | Optional non-empty extension version |

`metadata.id` MUST be a non-empty string. `metadata.version`, when present, MUST be a non-empty string.

## 6.3 Extension Spec Fields

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `dependencies` | array | NO | Extension reference strings required by this extension |
| `kinds` | array | NO | Condition kinds defined by this extension |
| `interfaceTypes` | array | NO | Interface types defined by this extension |
| `conditionFields` | array | NO | Condition-level fields defined by this extension |
| `interfaceFields` | array | NO | Interface-level fields defined by this extension |
| `fieldValues` | array | NO | Field values defined by this extension |
| `schemas` | array | NO | JSON Schema validation schemas defined by this extension |

An extension MUST define at least one vocabulary item or validation schema.

Each `dependencies` item MUST be a non-empty string using the same format as Profile `extensions` entries. Duplicate strings are invalid.

## 6.4 Vocabulary Definition Fields

Each `kinds` entry MUST include:

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `name` | string | YES | Condition kind defined by the extension |

Each `interfaceTypes` entry MUST include:

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `name` | string | YES | Interface type defined by the extension |
| `targetKind` | string | YES | Condition kind for which the interface type is valid |

Each `conditionFields` entry MUST include:

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `name` | string | YES | Condition-level field defined by the extension |
| `appliesToKinds` | array | YES | Kinds to which the field applies |
| `appliesToInterfaceTypes` | array | NO | Interface types to which the field applies |

Each `conditionFields` entry MUST declare a non-empty `appliesToKinds` list.

Each `interfaceFields` entry MUST include:

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `name` | string | YES | Interface-level field defined by the extension |
| `targetKind` | string | YES | Condition kind for which the field is valid |
| `targetType` | string | YES | Interface type for which the field is valid |

Each `fieldValues` entry MUST include:

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `field` | string | YES | Field path whose values are being defined |
| `targetKind` | string | YES | Condition kind scope |
| `targetType` | string | NO | Interface type scope |
| `values` | array | YES | Non-empty list of defined values |

Each `fieldValues.values` entry MUST be unique within that `fieldValues` entry.

Each `schemas` entry MUST include:

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | YES | Schema identifier unique within the extension |
| `description` | string | YES | Human-readable summary |
| `appliesToKind` | string | NO | Condition kind scope |
| `appliesToInterfaceType` | string | NO | Interface type scope |
| `schema` | object | YES | JSON Schema 2020-12 schema |

Each `schemas[].schema` object MUST be a valid JSON Schema using the JSON Schema 2020-12 dialect.

Each extension schema validates a Condition object after the profile has been converted to the JSON data model.

A schema applies to a Condition when all declared scope fields match the Condition. Omitted scope fields do not restrict applicability.

## 6.5 Extension Definition Example

```yaml
apiVersion: runtimeconditions.io/v1alpha1
kind: RuntimeConditionsExtensionDefinition

metadata:
  id: https://aws.example.com/runtimeconditions/object-store/v1alpha1/runtimeconditions.extension.yaml
  version: "v1alpha1"

spec:
  kinds:
    - name: aws.object_store

  interfaceTypes:
    - name: aws.s3
      targetKind: aws.object_store

  interfaceFields:
    - name: bucketClass
      targetKind: aws.object_store
      targetType: aws.s3

  fieldValues:
    - field: interface.bucketClass
      targetKind: aws.object_store
      targetType: aws.s3
      values:
        - standard
        - archive

  schemas:
    - id: aws-s3-interface
      appliesToKind: aws.object_store
      appliesToInterfaceType: aws.s3
      description: Validates the aws.s3 interface shape.
      schema:
        $schema: https://json-schema.org/draft/2020-12/schema
        type: object
        required:
          - kind
          - interface
        properties:
          kind:
            const: aws.object_store
          interface:
            type: object
            required:
              - type
            properties:
              type:
                const: aws.s3
              bucketClass:
                enum:
                  - standard
                  - archive
            additionalProperties: true
```

---

# 7. Extension Vocabulary Rules

Extensions MAY define:

- New Condition kinds
- New interface types
- New Condition fields
- New interface fields
- New field values
- JSON Schema validation schemas

Extensions MUST NOT:

- Redefine core fields
- Change core field meaning
- Define additional top-level profile fields
- Define additional fields within `metadata` or `workload`
- Encode secrets or concrete target-environment values

Core-reserved top-level fields:

- `apiVersion`
- `kind`
- `metadata`
- `workload`
- `extensions`
- `conditions`

Core-reserved Condition fields:

- `name`
- `optional`
- `kind`
- `interface`

Core-reserved interface field:

- `type`

## 7.1 Vocabulary Definition Scope

Vocabulary definition scope is determined by:

- Vocabulary category
- Field path, for field values
- Target Condition `kind`, when applicable
- Target `interface.type`, when applicable

A profile is invalid if the resolved extension set contains more than one definition for the same vocabulary item in the same scope.

Two definitions conflict even when they are identical.

Condition `kind` is exempt from this rule: more than one resolved extension MAY define the same `kind`. A Condition referencing an ambiguous `kind` resolves using `conditions[].extension`, or the first matching definition in extension resolution order when `conditions[].extension` is absent.

An extension that uses vocabulary defined by another extension MUST declare a dependency on that extension. It MUST NOT redefine that vocabulary in the same definition scope.

An extension MAY define vocabulary with overlapping purpose when it uses a distinct name or definition scope.

## 7.2 Namespacing

Extension-defined vocabulary SHOULD use namespaced identifiers when the vocabulary is vendor-specific, platform-specific, experimental, or likely to conflict.

Namespacing applies to:

- `kind` values
- `interface.type` values
- Condition field names
- Interface field names
- Field values that are not intended to be shared vocabulary

Examples:

```yaml
kind: aws.object_store
interface:
  type: aws.s3
```

```yaml
kind: api
interface:
  type: acme.soap
```

Unqualified vocabulary MAY be used when it resolves to exactly one definition in the resolved extension set.

---

# 8. Validation

Validation occurs in this order:

1. Core structural validation
2. Extension declaration resolution
3. Extension dependency resolution
4. Vocabulary definition and conflict validation
5. Extension JSON Schema validation

## 8.1 Structural Validation

A profile is structurally invalid if:

- `apiVersion` is missing or is not `runtimeconditions.io/v1alpha1`
- `kind` is missing or is not `RuntimeConditionsProfile`
- `metadata` is missing or is not an object
- `metadata.name` is missing or is not a non-empty string
- `metadata.labels` is present and is not a string-to-string mapping
- `workload` is missing or is not an object
- `workload.uri` is missing or is not a non-empty string
- `workload.version` is present and is not a non-empty string
- `extensions` is missing or is not an array
- An `extensions` item is not a non-empty string
- The `extensions` array contains a duplicate string
- `conditions` is non-empty and `extensions` is empty
- `conditions` is missing or is not an array
- Any required field is present with `null`
- Any mapping contains duplicate keys

A Condition is structurally invalid if:

- It is not an object
- `kind` is missing or is not a non-empty string
- `interface` is missing or is not an object
- `interface.type` is missing or is not a non-empty string
- `name` is present and is not a non-empty string
- `optional` is present and is not a boolean
- `extension` is present and is not a non-empty string

Condition names, when present, MUST be unique within the profile.

## 8.2 Vocabulary Validation

A profile is invalid if:

- A Condition `kind` is not defined by at least one resolved extension, or its `extension` does not resolve to exactly one definition
- An `interface.type` is not defined by exactly one resolved extension for the declared `kind`
- An extension-defined Condition field is not defined by exactly one resolved extension for its scope
- An extension-defined interface field is not defined by exactly one resolved extension for its scope
- An extension-defined field value is not defined by exactly one resolved extension for its scope
- Any resolved extension dependency is missing
- A resolved definition does not match the requested reference under the configured resolution rules
- Resolved extensions contain a dependency cycle
- Resolved extensions contain a vocabulary definition conflict
- Any applicable extension JSON Schema validation fails

## 8.3 Validity Levels

Validators SHOULD distinguish:

| Level | Description |
| ----- | ----------- |
| **Structural validity** | The document satisfies the core shape and type rules |
| **Extension-resolved validity** | All extensions resolve and all vocabulary has exactly one definition in scope, except Condition `kind`, which may resolve to more than one definition and is disambiguated by `extension` |
| **Semantic validity** | The profile satisfies all applicable extension JSON Schema validations |

A core-only profile with an empty `conditions` array can be structurally valid.

A profile with non-empty `conditions` is not extension-resolved valid unless every Condition `kind` resolves to exactly one definition (directly, or disambiguated by `extension`), and every interface type, extension-defined field, and extension-defined value resolves to exactly one definition.

## 8.4 Validator Diagnostics

Validation errors SHOULD identify:

- Error category
- Location within the profile
- Offending field or value
- Relevant extension, when applicable
- Expected valid type or vocabulary, when practical

Suggested error categories:

- `structural`
- `unknown-extension`
- `extension-dependency`
- `unresolved-vocabulary`
- `vocabulary-conflict`
- `semantic`

When a Condition's `kind` resolves against more than one definition, the validator SHOULD report whether that resolution was explicit or fallback.

---

# 9. Conformance

## 9.1 Profile Conformance

A conforming profile MUST:

- Satisfy core structural requirements
- Declare all extensions required to interpret its vocabulary
- Avoid unresolved vocabulary
- Avoid vocabulary definition conflicts
- Satisfy all semantic validation schemas for resolved extensions
- Avoid secret values and concrete target-environment values

## 9.2 Extension Conformance

A conforming extension MUST:

- Declare a non-empty `metadata.id` string
- Provide a valid extension definition artifact
- Apply the adapter-actionable minimum to its vocabulary and validation semantics
- Avoid mirroring API, SDK, library, protocol, configuration, or source-model detail that does not enable a materially different portable Adapter decision
- Identify all vocabulary it defines
- Declare exact-version dependencies on vocabulary defined by other extensions
- Avoid redefining vocabulary defined by another resolved extension
- Respect core field placement and reserved-name rules
- Express semantic validation with JSON Schema 2020-12 schemas

## 9.3 Generator Conformance

A conforming generator MUST emit structurally valid profiles.

A conforming generator MUST NOT emit secret values or concrete target-environment values.

A conforming generator SHOULD fail before emitting a profile with unresolved vocabulary, missing extensions, dependency errors, or known vocabulary conflicts.

## 9.4 Validator Conformance

A conforming validator MUST implement the validation layers in Section 8.

A conforming validator MUST reject structurally invalid profiles, unresolved extensions, dependency cycles, vocabulary conflicts, unresolved vocabulary, and JSON Schema validation failures.

## 9.5 Adapter Conformance

An Adapter consumes a Runtime Conditions Profile for a target platform, catalog, policy system, or deployment workflow.

A conforming Adapter MUST locate declared extension definitions and transitive extension dependencies before interpreting extension-defined vocabulary.

A conforming Adapter MUST NOT treat a structurally valid but extension-unresolved profile as semantically valid.

A conforming Adapter MUST preserve the distinction between profile requirements and target-environment fulfillment choices.

A conforming Adapter MAY map valid Conditions to platform resources, services, policies, configuration inputs, or deployment workflow steps. Such mappings are outside the profile data model.

---

# 10. Examples

## 10.1 Core-Only Structural Profile

```yaml
apiVersion: runtimeconditions.io/v1alpha1
kind: RuntimeConditionsProfile

metadata:
  name: structural-profile
  labels:
    owner.example.com/team: platform

workload:
  uri: https://github.com/example-org/example-service
  version: v1.2.3

extensions: []

conditions: []
```

## 10.2 Unresolved Conditions

This profile is structurally valid but not extension-resolved valid when its configured resolver cannot find the declared release. The opaque ID can be valid even when resolution fails.

```yaml
apiVersion: runtimeconditions.io/v1alpha1
kind: RuntimeConditionsProfile

metadata:
  name: unresolved-profile

workload:
  uri: https://github.com/example-org/example-service

extensions:
  - example.unresolved:1

conditions:
  - name: primary-db
    kind: relational-database
    interface:
      type: connection
```

## 10.3 Extension-Backed Profile

```yaml
apiVersion: runtimeconditions.io/v1alpha1
kind: RuntimeConditionsProfile

metadata:
  name: checkout-service
  labels:
    owner.example.com/team: payments
    lifecycle.example.com/stage: production

workload:
  uri: https://github.com/example-org/checkout-service
  version: v1.2.3

extensions:
  - https://runtimeconditions.io/extensions/common-integrations:v1alpha1

conditions:
  - name: primary-db
    kind: datastore
    interface:
      type: relational
      engine: postgres

  - name: payments-api
    kind: api
    interface:
      type: http
      operations:
        - method: POST
          path: /charge
```

## 10.4 Extension-Backed Profile With Configuration

```yaml
apiVersion: runtimeconditions.io/v1alpha1
kind: RuntimeConditionsProfile

metadata:
  name: checkout-service
  labels:
    owner.example.com/team: payments

workload:
  uri: https://github.com/example-org/checkout-service
  version: v1.2.3

extensions:
  - https://runtimeconditions.io/extensions/common-integrations:v1alpha1
  - https://runtimeconditions.io/extensions/env-configuration:v1alpha1

conditions:
  - name: primary-db
    kind: datastore
    interface:
      type: relational
      engine: postgres
    configuration:
      env:
        - property: hostname
          name: POSTGRES_HOST
        - property: port
          name: POSTGRES_PORT
        - property: database
          name: POSTGRES_DATABASE
        - property: username
          name: POSTGRES_USERNAME
        - property: password
          name: POSTGRES_PASSWORD
          sensitive: true

  - name: request-cache
    kind: cache
    interface:
      type: key_value
      engine: redis
    configuration:
      alternatives:
        - env:
            - property: url
              name: REDIS_URL
        - env:
            - property: hostname
              name: REDIS_HOST
            - property: port
              name: REDIS_PORT

  - name: payments-api
    kind: api
    interface:
      type: http
      operations:
        - method: POST
          path: /charge
    configuration:
      env:
        - property: baseUrl
          name: PAYMENTS_API_URL
        - property: token
          name: PAYMENTS_API_TOKEN
          sensitive: true
```

## 10.5 Profile With Ambiguous Kind

Two independently authored extensions both define `kind: cache`. The profile disambiguates the first Condition with `extension`; the second Condition omits it and resolves against the first matching definition in extension resolution order.

```yaml
apiVersion: runtimeconditions.io/v1alpha1
kind: RuntimeConditionsProfile

metadata:
  name: checkout-service

workload:
  uri: https://github.com/example-org/checkout-service
  version: v1.2.3

extensions:
  - https://redis-vendor.example.com/extensions/redis:0.1.0
  - https://memcached-vendor.example.com/extensions/memcached:0.1.0

conditions:
  - name: primary-cache
    kind: cache
    extension: https://redis-vendor.example.com/extensions/redis:0.1.0
    interface:
      type: redis_protocol

  - name: fallback-cache
    kind: cache
    interface:
      type: memcached_protocol
```
