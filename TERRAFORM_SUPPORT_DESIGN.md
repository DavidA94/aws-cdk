# Design Proposal: Terraform/OpenTofu Output Support in aws-cdk-lib

**Status:** Draft / POC  
**Date:** 2025-05-15  
**Authors:** kornherm

## 1. Goal

Allow users to take their **existing** CDK app (same constructs, same code) and synthesize it to Terraform JSON instead of CloudFormation, by changing only the synthesizer (and optionally using a convenience App/Stage wrapper).

```ts
// Before (CloudFormation output)
const app = new cdk.App();
const stack = new cdk.Stack(app, 'MyStack');
new s3.Bucket(stack, 'Bucket');

// After (Terraform output) — Option A: explicit synthesizer
const app = new cdk.App();
const stack = new cdk.Stack(app, 'MyStack', {
  synthesizer: new TerraformStackSynthesizer(),
});
new s3.Bucket(stack, 'Bucket');

// After (Terraform output) — Option B: convenience App
const app = new TerraformApp();
const stack = new cdk.Stack(app, 'MyStack');
new s3.Bucket(stack, 'Bucket');
```

## 2. How cdk-terrain Does It (and Why We Can't Copy It)

cdk-terrain (formerly CDKTF) is a **completely separate framework**. It does NOT reuse aws-cdk-lib's `Stack`, `App`, `CfnResource`, or synthesis pipeline at all:

- `App` extends `Construct` directly (not `cdk.App`)
- `TerraformStack` extends `Construct` directly (not `cdk.Stack`)
- Resources are `TerraformElement` subclasses with `toTerraform()` methods
- The synthesizer calls `stack.toTerraform()` which collects all TerraformElements

**Key insight:** cdk-terrain resources are purpose-built for Terraform. They don't go through CloudFormation at all. For our POC, we must work within aws-cdk-lib's existing model where resources are `CfnResource` instances that produce CloudFormation JSON.

## 3. Architecture: Pluggable Resource Serialization

### 3.1 The Problem with `_toCloudFormation()`

Today, every `CfnElement` has `_toCloudFormation()` which produces a CFN JSON fragment. The Stack collects these fragments and merges them into a template. This is hardcoded — there's no way to ask a resource "give me your data in a different format."

But the **raw data is already format-neutral**:
- `CfnResource.cfnResourceType` → `"AWS::S3::Bucket"` (a type identifier)
- `CfnResource.cfnProperties` → `{ bucket: "my-bucket", tags: [...] }` (CDK property names, not CFN names)
- `CfnResource.cfnOptions` → condition, dependsOn, deletionPolicy (lifecycle metadata)
- `CfnResource.logicalId` → unique identifier

The CFN-specific part is `renderProperties()` which maps CDK property names → CFN property names (e.g., `bucket` → `Bucket`). And `_toCloudFormation()` which wraps everything in `{ Resources: { [logicalId]: { Type, Properties } } }`.

### 3.2 Pluggable Serialization via `ResourceData`

Resources expose a `serialize(serializer)` method that pushes a **format-neutral data struct** to the serializer. The serializer receives resolved, neutral data and maps it to the target format.

```ts
/**
 * Format-neutral representation of a resource.
 * These are generic IaC concepts, not CFN-specific.
 */
export interface ResourceElementData extends ElementData {
  readonly kind: 'cdk:resource';
  readonly resourceType: string;
  readonly properties: Record<string, any>;
  /** Construct paths of dependencies */
  readonly dependencies: string[][];
  /** Construct path segments of condition */
  readonly condition?: string[];
  readonly removalPolicy?: RemovalPolicy;
  readonly metadata?: Record<string, any>;
}

export interface OutputElementData extends ElementData {
  readonly kind: 'cdk:output';
  readonly value: any;
  readonly description?: string;
  readonly exportName?: string;
}

export interface ParameterElementData extends ElementData {
  readonly kind: 'cdk:parameter';
  readonly type: string;
  readonly default?: any;
  readonly description?: string;
  readonly allowedValues?: any[];
}

export interface ConditionElementData extends ElementData {
  readonly kind: 'cdk:condition';
  readonly expression: any;
}

export interface MappingElementData extends ElementData {
  readonly kind: 'cdk:mapping';
  readonly mapping: Record<string, Record<string, any>>;
}
```

**The serializer interface:**

```ts
export interface IElementSerializer {
  /**
   * Serialize a single stack element.
   * `kind` is an open string — serializers handle known kinds and skip/warn on unknown ones.
   */
  serializeElement(data: ElementData): void;
}

/**
 * Format-neutral element data, discriminated by `kind`.
 *
 * Kind prefixes:
 *   "cdk:"  — generic IaC concepts (resource, output, parameter, condition, mapping)
 *   "cfn:"  — CloudFormation-specific (cfn:rule, cfn:hook, cfn:include, cfn:transform)
 *   "tf:"   — Terraform-specific (future: tf:backend, tf:provider-config, etc.)
 */
export interface ElementData {
  /** Discriminator. Open string — any value allowed. */
  readonly kind: string;
  /** Construct path segments — e.g., ['MyStack', 'MyBucket', 'Resource']. Serializer derives target-specific ID from this. */
  readonly path: string[];
}

// Generic IaC kinds ("cdk:" prefix)

**On `CfnElement` subclasses:**

```ts
// CfnResource
public serialize(serializer: IElementSerializer): void {
  serializer.serializeElement({
    kind: 'cdk:resource',
    path: this.node.path.split('/'),
    resourceType: this.cfnResourceType,
    properties: this.cfnProperties,
    dependencies: [...(this.dependsOn ?? [])].map(d => d.node.path.split('/')),
    condition: this.cfnOptions.condition?.node.path.split('/'),
    removalPolicy: cfnDeletionPolicyToRemovalPolicy(this.cfnOptions.deletionPolicy),
    metadata: this.cfnOptions.metadata,
  });
}

// CfnRule (CFN-specific)
public serialize(serializer: IElementSerializer): void {
  serializer.serializeElement({
    kind: 'cfn:rule',
    path: this.node.path.split('/'),
    assertions: this.assertions,
    // ...
  });
}
```

**How serializers handle this:**

- Known kinds → process them
- Unknown kinds → skip with a warning (e.g., TF serializer sees `cfn:rule` → warns "unsupported element")
- Future: TF serializer could emit `tf:backend` kinds that CFN serializer ignores

This is fully open — no enum, no closed union. New kinds can be added without breaking existing serializers.

### 3.3 Existing `_toCloudFormation()` — Migration Path

**POC:** `_toCloudFormation()` remains unchanged. The TF synthesizer calls `serialize()` on each element directly. Two parallel paths coexist.

**Post-POC:** Refactor `_toCloudFormation()` to use a `CloudFormationSerializer` internally:

```ts
// On Stack (future):
protected _toCloudFormation() {
  const serializer = new CloudFormationSerializer(this);
  for (const element of cfnElements(this)) {
    element.serialize(serializer);
  }
  return serializer.toTemplate();
}
```

This unifies both paths — `_toCloudFormation()` becomes just one consumer of the same `serialize()` data. The `CloudFormationSerializer` handles `cdk:*` kinds (resource, output, parameter, condition, mapping) plus `cfn:*` kinds (rule, hook, include) natively.

**Why not now:** `_toCloudFormation()` is the most exercised code path in the CDK. Refactoring it requires full regression coverage across the entire test suite. Do it as a separate, focused effort.

### 3.4 Token Resolution Per Serializer

Since `properties` contains unresolved tokens, each serializer provides its own resolution strategy:

| CDK Token | CFN resolution | TF resolution |
|---|---|---|
| `CfnReference(resource, 'Ref')` | `{ "Ref": "LogicalId" }` | `aws_s3_bucket.logical_id.id` |
| `CfnReference(resource, 'Arn')` | `{ "Fn::GetAtt": ["Id", "Arn"] }` | `aws_s3_bucket.logical_id.arn` |
| `Fn.join("-", [...])` | `{ "Fn::Join": ["-", [...]] }` | `join("-", [...])` |
| `Stack.account` (pseudo) | `{ "Ref": "AWS::AccountId" }` | `data.aws_caller_identity.current.account_id` |
| `Stack.region` (pseudo) | `{ "Ref": "AWS::Region" }` | `data.aws_region.current.name` |

The serializer resolves tokens by providing a custom `IResolveContext` when processing the properties.

### 3.5 Comparison with cdktf-aws-cdk Approach

| | cdktf-aws-cdk (`AwsTerraformAdapter`) | Our approach |
|---|---|---|
| Reads data via | `_toCloudFormation()` then reverse-engineers | `serialize()` pushes neutral `ResourceData` |
| Mapping input | CFN property names (PascalCase) | CDK property names (camelCase) |
| Token handling | Resolved to CFN first, then re-parsed | Resolved directly to target format |
| Requires | Both aws-cdk-lib + cdktf | Only aws-cdk-lib |
| Extensibility | None — hardcoded adapter | Any serializer can be plugged in |

## 4. Components

### 4.1 `TerraformStackSynthesizer` (implements `IStackSynthesizer`)

The core new class. Orchestrates synthesis by iterating elements and collecting serialized output.

```ts
export class TerraformStackSynthesizer extends StackSynthesizer {
  bind(stack: Stack): void;
  addFileAsset(asset: FileAssetSource): FileAssetLocation;
  addDockerImageAsset(asset: DockerImageAssetSource): DockerImageAssetLocation;
  synthesize(session: ISynthesisSession): void;
  supportsCrossStackReferences(producer: IStackSynthesizer): boolean; // returns false for now
}
```

**`synthesize()` flow:**
1. Create a `TerraformSerializer` (implements `IElementSerializer`)
2. Iterate `cfnElements(stack)`, call `element.serialize(serializer)` on each
3. The serializer accumulates Terraform JSON fragments internally (handles `cdk:*` kinds, warns on `cfn:*` kinds)
4. Finalize: add `required_providers`, `provider` block, pseudo-parameter data sources
5. Write `main.tf.json` to `{outdir}/stacks/{stackName}/`
6. Emit a `terraform:stack` artifact to the cloud assembly

**`addFileAsset()` / `addDockerImageAsset()`:**
- For the POC, return placeholder locations (assets are a later concern)

### 4.2 Resource Type Mapping: `aws_cloudcontrolapi_resource`

**Target: `aws_cloudcontrolapi_resource` from `hashicorp/aws` provider**

Instead of mapping each CFN type to a specific TF resource, we use the generic Cloud Control API resource. This is a single TF resource type that can manage *any* AWS resource via CCAPI:

```hcl
resource "aws_cloudcontrolapi_resource" "my_bucket" {
  type_name = "AWS::S3::Bucket"

  desired_state = jsonencode({
    BucketName = "my-bucket"
    VersioningConfiguration = { Status = "Enabled" }
  })
}
```

**Why this approach:**
- **Zero per-resource mapping needed** — every CFN resource type works immediately
- **Same property names** — CCAPI uses CFN property schema (PascalCase)
- **Same semantics** — create/update/delete lifecycle matches CFN exactly
- **Ships in `hashicorp/aws`** — no extra provider dependency
- **Full coverage** — any resource in the CloudFormation registry works

**The serializer just needs to:**
1. Convert `cfnProperties` from CDK camelCase → CFN PascalCase (using existing `renderProperties()`)
2. JSON-encode as `desired_state`
3. Set `type_name` to the CFN resource type string

```ts
// For every cdk:resource, emit:
{
  resource: {
    aws_cloudcontrolapi_resource: {
      [terraformId]: {
        type_name: "AWS::S3::Bucket",
        desired_state: jsonencode(cfnProperties)  // PascalCase
      }
    }
  }
}
```

**Reading attributes (Ref/GetAtt):**
`aws_cloudcontrolapi_resource` exposes `properties` as a JSON string output. We can read attributes via `jsondecode(resource.properties).Arn`.

**Post-POC upgrade path:**
Users can later migrate to native TF resources (e.g., `aws_s3_bucket`) for higher-fidelity features. This requires `terraform state rm` + `terraform import` per resource — a well-understood TF migration pattern. We could emit migration scripts to assist.

### 4.3 Cloud Assembly Artifact Type

We introduce a **new artifact type**: `terraform:stack`.

```ts
// New enum value in @aws-cdk/cloud-assembly-schema
TERRAFORM_STACK = 'terraform:stack'

interface TerraformStackProperties {
  templateFile: string;  // e.g., "stacks/MyStack/main.tf.json"
  workingDirectory: string;
}
```

**The CDK CLI will never deploy `terraform:stack` artifacts.** Users run `terraform init && terraform apply` themselves. The artifact exists so that:
- `cdk synth` produces discoverable output
- Tooling can enumerate what's in the assembly
- Future third-party tools can consume the assembly

**Assembly schema pluggability:** The current `ArtifactType` enum is closed. We need to either:
1. Add `TERRAFORM_STACK` to the enum (requires cloud-assembly-schema release), or
2. Make the schema accept unknown artifact types gracefully (more pluggable)

For the POC we add the enum value. Longer-term, making the schema open to extension (unknown types are ignored rather than rejected) is the right call for a pluggable ecosystem.

### 4.5 `TerraformStack`

A `Stack` subclass that is the primary user-facing primitive:

```ts
export class TerraformStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, {
      ...props,
      synthesizer: props?.synthesizer ?? new TerraformStackSynthesizer(),
    });
  }

  /** Type guard */
  public static isTerraformStack(x: any): x is TerraformStack {
    return x instanceof TerraformStack;
  }
}
```

Beyond presetting the synthesizer, `TerraformStack` will:
- Suppress CFN-specific validations (template size limits, CDKMetadata resource)
- Adjust logical ID generation if needed for Terraform naming constraints
- Serve as the type-check target for blocking cross-stack references

### 4.6 Cross-Stack Reference Blocking (Dynamic Discovery)

**Decision: Cross-stack reference support is discovered dynamically via the synthesizer.**

Rather than hardcoding "if TerraformStack → throw", we make the synthesizer declare what it supports. This is fully extensible — any future synthesizer (Pulumi, Crossplane, etc.) just implements the method.

**New method on `IStackSynthesizer`:**

```ts
interface IStackSynthesizer {
  // ... existing ...

  /**
   * Whether this synthesizer supports being the consumer of a
   * cross-stack reference from the given producer synthesizer.
   *
   * Called during reference resolution. If not implemented, defaults
   * to `true` (backwards compatible — existing CFN↔CFN works unchanged).
   */
  supportsCrossStackReferences?(producerSynthesizer: IStackSynthesizer): boolean;
}
```

**Implementation in `resolveValue()` (private/refs.ts):**

```ts
// After determining producer !== consumer:
const canReference = consumer.synthesizer.supportsCrossStackReferences?.(producer.synthesizer) ?? true;
if (!canReference) {
  throw new Error(
    `Cross-stack references between "${consumer.node.path}" and "${producer.node.path}" ` +
    `are not supported by the stack synthesizer "${consumer.synthesizer.constructor.name}".`
  );
}
```

**Synthesizer implementations:**

| Synthesizer | `supportsCrossStackReferences(producer)` |
|---|---|
| `DefaultStackSynthesizer` | not implemented → defaults to `true` (no change) |
| `TerraformStackSynthesizer` | returns `false` for all (POC); later returns `true` when producer is CFN (can read outputs via `data.aws_cloudformation_stack`) |

**Why this design:**
- Zero breaking changes — method is optional, defaults to `true`
- Each synthesizer owns its own compatibility story
- No central allowlist to maintain
- Fully dynamic — works for any future output format

### 4.7 `TerraformApp` / `TerraformStage` (Optional Convenience Wrappers)

```ts
export class TerraformApp extends App {
  constructor(props?: AppProps) {
    super({
      ...props,
      defaultStackSynthesizer: new TerraformStackSynthesizer(),
    });
  }
}

export class TerraformStage extends Stage {
  constructor(scope: Construct, id: string, props?: StageProps) {
    super(scope, id, {
      ...props,
      // Applies TerraformStackSynthesizer to all stacks in this stage
    });
  }
}
```

These simply preset the default synthesizer. No new behavior.

## 5. Refactoring Required in aws-cdk-lib

### 5.1 `CfnElement` — Add Public `serialize(serializer)` Method

Add `serialize(serializer: IElementSerializer)` to `CfnElement` subclasses. Each pushes its data with the appropriate `kind` prefix:
- `CfnResource` → `cdk:resource`
- `CfnOutput` → `cdk:output`
- `CfnParameter` → `cdk:parameter`
- `CfnCondition` → `cdk:condition`
- `CfnMapping` → `cdk:mapping`
- `CfnRule` → `cfn:rule`
- `CfnHook` → `cfn:hook`

This makes `cfnProperties` accessible indirectly — the resource reads its own protected data and pushes it in neutral form. No need to change visibility.

**Breaking change:** None — additive public method. `_toCloudFormation()` remains unchanged.

### 5.2 `addStackArtifactToAssembly` — Abstract the Artifact Type

Currently hardcoded to `ArtifactType.AWS_CLOUDFORMATION_STACK`. We need either:
- A new helper for Terraform artifacts, OR
- Make the existing helper accept an artifact type parameter

**Breaking change potential:** None — we add a new code path, don't modify the existing one.

### 5.3 `private/refs.ts` — Dynamic Cross-Stack Reference Gating

Add a check in `resolveValue()` that calls `consumer.synthesizer.supportsCrossStackReferences?.(producer.synthesizer)`. If it returns `false`, throw. This is a **non-breaking change** — the method is optional and defaults to `true`, so all existing synthesizers continue working unchanged.

### 5.4 `ISynthesisSession` — No Changes Needed

The session already provides `outdir` and `assembly` (CloudAssemblyBuilder). We can write Terraform files to `outdir` and add artifacts to `assembly` using the existing API.

### 5.5 Asset Handling — Deferred

For the POC, `addFileAsset()` and `addDockerImageAsset()` will return dummy locations. Assets in Terraform are handled differently (no S3 staging bucket pattern). This is a future concern.

## 6. Intrinsic Functions

Same pattern as elements: intrinsics have a `kind` (open string with prefix), and the serializer resolves them based on kind.

### 6.1 `IIntrinsic` Interface

```ts
/**
 * Represents a deferred computation (intrinsic function).
 * The serializer resolves it to the target format.
 */
export interface IIntrinsic {
  /** Open string, prefixed: "cdk:", "cfn:", "tf:", etc. */
  readonly kind: string;
  /** Arguments to the intrinsic */
  readonly args: any;
}
```

### 6.2 Generic IaC Intrinsics (`cdk:` prefix)

These have equivalents in every IaC engine:

| Kind | Args | CFN output | TF output |
|---|---|---|---|
| `cdk:ref` | `{ target, attribute }` | `{ "Ref": "X" }` / `{ "Fn::GetAtt": [...] }` | `type.id.attr` |
| `cdk:join` | `{ separator, values }` | `{ "Fn::Join": [...] }` | `join(sep, list)` |
| `cdk:split` | `{ separator, value }` | `{ "Fn::Split": [...] }` | `split(sep, str)` |
| `cdk:select` | `{ index, values }` | `{ "Fn::Select": [...] }` | `element(list, idx)` |
| `cdk:sub` | `{ template, replacements }` | `{ "Fn::Sub": [...] }` | `"${...}"` interpolation |
| `cdk:base64` | `{ value }` | `{ "Fn::Base64": ... }` | `base64encode(...)` |
| `cdk:cidr` | `{ ipBlock, count, bits }` | `{ "Fn::Cidr": [...] }` | `cidrsubnets(...)` |
| `cdk:getAzs` | `{ region }` | `{ "Fn::GetAZs": ... }` | `data.aws_availability_zones.*.names` |
| `cdk:findInMap` | `{ map, key1, key2 }` | `{ "Fn::FindInMap": [...] }` | `local.map[key1][key2]` |
| `cdk:length` | `{ value }` | `{ "Fn::Length": ... }` | `length(...)` |
| `cdk:if` | `{ condition, ifTrue, ifFalse }` | `{ "Fn::If": [...] }` | `cond ? a : b` |
| `cdk:equals` | `{ left, right }` | `{ "Fn::Equals": [...] }` | `a == b` |
| `cdk:and` | `{ operands }` | `{ "Fn::And": [...] }` | `a && b && ...` |
| `cdk:or` | `{ operands }` | `{ "Fn::Or": [...] }` | `a \|\| b \|\| ...` |
| `cdk:not` | `{ operand }` | `{ "Fn::Not": [...] }` | `!a` |

### 6.3 CFN-Specific Intrinsics (`cfn:` prefix)

| Kind | Notes |
|---|---|
| `cfn:importValue` | Cross-stack. Blocked for TF stacks. |
| `cfn:transform` | CFN macros. No TF equivalent. |
| `cfn:valueOf` | Rule-specific. No TF equivalent. |

### 6.4 TF-Specific Intrinsics (`tf:` prefix, future)

| Kind | Notes |
|---|---|
| `tf:templatefile` | TF's `templatefile()` function |
| `tf:lookup` | TF's `lookup()` with default |

### 6.5 How It Works

Today, `Fn.join(...)` creates an `FnJoin` token (extends `Intrinsic`). When resolved, it produces `{ "Fn::Join": [...] }`. 

**New model:** The token carries its `kind` (`cdk:join`) and `args` (`{ separator, values }`). The resolve context asks the serializer how to render it:

```ts
// Simplified — the serializer provides a resolve strategy:
export interface IIntrinsicResolver {
  resolveIntrinsic(intrinsic: IIntrinsic): any;
}
```

- `CloudFormationResolver` → `{ "Fn::Join": [sep, values] }`
- `TerraformResolver` → `"${join(sep, values)}"`
- Unknown kind → throw with clear error

### 6.6 Migration Path

**POC:** The existing `FnJoin`, `FnSelect`, etc. classes gain a `kind` property. The TF serializer reads `kind` + `args` and resolves accordingly. The CFN path (`_toCloudFormation()`) continues using the existing `resolve()` machinery unchanged.

**Post-POC:** The existing `resolve()` machinery delegates to a `CloudFormationResolver` internally, unifying both paths.

## 7. Custom Resources → Terraform Actions

CDK uses custom resources extensively (~50 L2 constructs). These are side-effects at lifecycle points — not CRUD resources. Terraform's new **Actions** framework is the direct equivalent.

### 7.1 Element Kind: `cdk:action`

Custom resources serialize as `cdk:action`:

```ts
export interface ActionElementData extends ElementData {
  readonly kind: 'cdk:action';
  readonly serviceToken: string;       // Lambda ARN
  readonly resourceType: string;       // "Custom::S3AutoDeleteObjects"
  readonly properties: Record<string, any>;
  readonly removalPolicy?: RemovalPolicy;
}
```

### 7.2 TF Serializer Strategy

1. **Known actions with native TF equivalents** → map to resource attributes:
   - `Custom::S3AutoDeleteObjects` → `force_destroy = true`
   - `Custom::LogRetention` → `retention_in_days` on `aws_cloudwatch_log_group`
   - `Custom::VpcRestrictDefaultSG` → native SG rules

2. **Generic actions** → emit as Terraform Action invocation (requires a CDK TF provider that exposes these as actions)

3. **POC** → warn and skip unknown custom resources. Map the top ~5 known ones to native TF equivalents.

### 7.3 Future: CDK Terraform Provider

Long-term, we could ship a Terraform provider that exposes CDK's custom resource Lambdas as Terraform Actions — making the generic case work without per-resource mapping. OpenTofu will catch up to Actions eventually.

## 8. What's NOT in Scope for POC

- Nested stacks (Terraform has modules but different semantics)
- Generic custom resource → TF Action mapping (map top ~5 known ones only)
- CloudFormation-specific features: StackSets, Change Sets, Drift Detection
- Full asset pipeline (S3 staging, ECR push)
- `cdk deploy` integration (users run `terraform apply` manually)
- Cross-stack references (need Terraform remote state pattern)
- Complete resource type coverage (start with S3, Lambda, IAM, DynamoDB, SQS, SNS)

## 9. POC Scope

1. **`IElementSerializer` interface** — single `serializeElement(data)` method with open `kind` string
2. **`IIntrinsicResolver` interface** — resolves intrinsics per target format
3. **`TerraformStackSynthesizer`** — orchestrates synthesis, emits `aws_cloudcontrolapi_resource` for all resources
4. **`TerraformStack`** — convenience Stack subclass
5. **`serialize()` on CfnElement subclasses** — pushes neutral `ElementData` to serializer
6. **Generic CCAPI mapping** — all resources emit as `aws_cloudcontrolapi_resource` with `type_name` + `desired_state`
7. **Token/intrinsic resolution** for: `cdk:ref`, `cdk:join`, `cdk:select`, `cdk:split`, `cdk:sub`, pseudo-parameters
8. **Cross-stack ref gating** via `supportsCrossStackReferences()` on synthesizer
9. **Known custom resource mappings** for: S3 auto-delete, log retention (or skip with warning)
10. **Unit tests** proving the same CDK code produces valid Terraform JSON

## 10. File Structure

```
packages/aws-cdk-lib/core/lib/
├── stack-synthesizers/
│   ├── terraform-synthesizer.ts          # TerraformStackSynthesizer
│   ├── types.ts                          # IElementSerializer, ElementData, etc. added here
│   └── index.ts                          # re-export
├── terraform/                            # Internal implementation
│   ├── terraform-serializer.ts           # IElementSerializer impl — emits aws_cloudcontrolapi_resource
│   ├── terraform-intrinsic-resolver.ts   # IIntrinsicResolver impl for TF expressions
│   ├── custom-resource-mappings.ts       # Known cdk:action → native TF mappings
│   └── index.ts
├── cfn-element.ts                       # Modified: add serialize() method
├── cfn-resource.ts                      # Modified: add serialize() override
├── cfn-output.ts                        # Modified: add serialize() override
├── cfn-parameter.ts                     # Modified: add serialize() override
├── cfn-condition.ts                     # Modified: add serialize() override
├── terraform-stack.ts                   # TerraformStack class (public)
├── terraform-app.ts                     # TerraformApp convenience class (public)
├── private/
│   └── refs.ts                          # Modified: dynamic cross-stack ref gating
```

## 11. Open Questions

None — all resolved.

### Resolved

- **Logical ID mapping:** Serializer owns ID generation. `ElementData` carries the construct `path`; each serializer derives target-specific IDs from it (CFN → hashed alphanumeric, TF → snake_case from path segments). No sanitization problem.
- **Provider version pinning:** Synthesizer adds `required_providers` for POC. Long-term, TF-only constructs will ship their own provider requirements.
- **State management:** Out of scope for POC. Eventually a `TerraformBackend` construct will handle this.
- **Split resources:** Not an issue — we target `aws_cloudcontrolapi_resource` which maps 1:1 to CFN resource types.

## 12. Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| CFN↔TF mapping is lossy for complex resources | Start with simple resources; document unsupported patterns |
| Intrinsic translation is incomplete | Fail loudly on unsupported intrinsics with clear error messages |
| L2 constructs generate CFN-only resources (Custom Resources) | Skip/warn on custom resources; provide escape hatch |
| Breaking changes to IStackSynthesizer | We're only adding, not modifying the interface |
| Performance of template transformation | Template is already in memory; transformation is O(n) on resources |

## 13. Success Criteria for POC

- [ ] `cdk synth` with `TerraformStackSynthesizer` produces valid `main.tf.json`
- [ ] `terraform validate` passes on the output
- [ ] `terraform plan` shows expected resources (for supported types)
- [ ] Same CDK app code works with both default synthesizer (CFN) and Terraform synthesizer
- [ ] No changes required to existing CDK constructs or user code
