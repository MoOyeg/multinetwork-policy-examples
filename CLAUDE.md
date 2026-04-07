# CLAUDE.md — NFS Policy Examples

## Project Purpose

This repository contains Red Hat Advanced Cluster Management (RHACM/ACM) policy examples for managing NFS-related configurations across OpenShift/Kubernetes clusters. It follows patterns and best practices from two reference repositories.

## Reference Repositories

### 1. acm-onboarding-examples (github.com/MoOyeg/acm-onboarding-examples)

Demonstrates progressive ACM onboarding patterns across three examples:

- **Example 1**: Hierarchical namespace management using Kustomize overlays. A `global-teams/` base defines shared resources (ClusterResourceQuota, LimitRange, secrets). Team overlays (`team-a/`, `team-b/`) customize via patches. A `PolicyGenerator` wraps manifests into ACM Policies targeting clusters by label (`team-a-cluster=true`).
- **Example 2**: Builds on Example 1, replacing static Kustomize `secretGenerator` with ACM template functions (`range`, `lookup`, `fromSecret`) to dynamically inject secrets into every namespace labeled `deployer=kustomize-global`.
- **Example 3**: Full enterprise onboarding — htpasswd OAuth users, groups (`team-a`, `team-b`), operator installation (DevSpaces, GitOps) via ACM policies, and per-user DevWorkspace provisioning using nested template lookups checking group membership.

Key patterns: PolicyGenerator plugin, Placement/PlacementBinding, enforce remediation, policy dependencies, ACM hub templates, Kustomize overlays.

### 2. acm-policy-samples (github.com/bry-tam/acm-policy-samples)

Production-grade ACM policy repository with 50+ policies organized for multi-environment GitOps workflows (Dev → QA → Prod):

- **Structure**: `policies/` holds all definitions (acm-configs, cluster-configs, cluster-health, cluster-maintenance, cluster-validations, cluster-version, gatekeeper, operators, application-defaults, multicluster-data, security). `environments/` holds Kustomize overlays per environment. `kustomize-configs/` holds shared transformers.
- **PolicyGenerator**: Every policy uses `generator.yml` files. Policies are grouped into PolicySets. Three reusable placements: `env-bound-placement` (all clusters), `env-bound-hub-placement` (hub only), `env-bound-nohub-placement` (all except hub).
- **Feature flags**: Dynamic placement creation via naming convention `ft-<LabelName>--<LabelValue>` (e.g., `ft-logging-type--Loki`). The `feature-flags-placement` policy auto-generates these Placements.
- **Environment promotion**: Single policy source, namespace/PolicySet suffix per environment, ClusterSet scoping via Kustomize transformers. Managed via tags or branches with ArgoCD ApplicationSets.
- **Operators covered**: ACM, ACS, AAP, cert-manager, cluster-logging, compliance, ODF, Developer Hub, external-secrets, file-integrity, Gatekeeper, GitOps, Kiali, Loki, network-observability, OpenTelemetry, secondary-scheduler, service-mesh, TALM, Tekton, Tempo, workload-availability (23 total).
- **CI/CD**: GitHub Actions validate PolicyGenerator builds, YAML schema (kubeconform), trailing whitespace, `---` doc markers, and newline EOF.

## ACM Concepts Quick Reference

| Concept | Description |
|---|---|
| **Policy** | Desired-state definition enforced or monitored on target clusters |
| **ConfigurationPolicy** | Policy type for Kubernetes object state management with template support |
| **OperatorPolicy** | Policy type for OLM operator lifecycle management |
| **PolicySet** | Logical grouping of policies for placement binding |
| **PolicyGenerator** | Kustomize plugin that generates Policy/PolicySet from YAML manifests |
| **Placement** | Selects target clusters by label, ClusterSet, or claim |
| **PlacementBinding** | Links Policy/PolicySet to a Placement |
| **ManagedClusterSet** | Groups of clusters for scoped access |
| **ManagedClusterSetBinding** | Namespace-scoped binding to a ClusterSet |
| **Remediation** | `enforce` (auto-correct) or `inform` (report only) |
| **Dependencies** | Policy ordering — wait for another policy to be `Compliant` |
| **Template functions** | `lookup`, `range`, `fromSecret`, `contains`, `base64enc/dec` for dynamic policies |

## Policy File Conventions

Follow the standards from acm-policy-samples:

1. All YAML files must start with `---`
2. No trailing whitespace
3. Files must end with a blank line
4. OperatorPolicy subscriptions must specify complete definitions including a versioned channel (e.g., `stable-1.23`)
5. Every policy needs a README with: description, dependencies, ACM minimal version, documentation links, implementation notes
6. Each `generator.yml` policy must have a `description` field
7. Manifest names must be unique across all policies
8. Include NIST SP 800-53 compliance annotations:
   ```yaml
   categories:
     - "CM Configuration Management"
   controls:
     - "CM-2 Baseline Configuration"
   standards:
     - "NIST SP 800-53"
   ```
9. Include `policyLabels` with a `policy_gen.name` identifier

## PolicyGenerator Template

```yaml
---
apiVersion: policy.open-cluster-management.io/v1
kind: PolicyGenerator
metadata:
  name: gen-policy-generator-<category>
policyDefaults:
  namespace: <policy-namespace>
  remediationAction: enforce
  consolidateManifests: false
  policySets:
    - <policyset-name>
  categories:
    - "CM Configuration Management"
  controls:
    - "CM-2 Baseline Configuration"
  standards:
    - "NIST SP 800-53"
  policyLabels:
    policy_gen.name: <label>
  severity: medium
placementBindingDefaults:
  name: "<category>-binding"
policies:
  - name: <policy-name>
    description: "<description>"
    manifests:
      - path: <manifest-file.yml>
        name: <unique-manifest-name>
        complianceType: musthave
policySets:
  - name: <policyset-name>
    placement:
      placementName: "env-bound-placement"
```

## Kustomize Build Commands

```bash
# Build policies with PolicyGenerator plugin
kustomize build --enable-alpha-plugins --enable-helm <path>/

# Apply directly
kustomize build --enable-alpha-plugins <path>/ | oc create -f -

# Apply with oc
oc apply -k <path>/
```

## Placement Strategies

- **All clusters in environment**: Use `env-bound-placement` with ClusterSet scoping
- **Hub only**: Use `env-bound-hub-placement` (matches `local-cluster`)
- **All except hub**: Use `env-bound-nohub-placement`
- **Feature flag subset**: Name placement `ft-<labelKey>--<labelValue>` and let the feature-flags-placement policy auto-generate it
- **Team-specific**: Use label selectors like `team-a-cluster=true` on ManagedCluster

## Directory Structure

```
.
├── CLAUDE.md                    # This file
├── README.md                    # Repo documentation
├── requirements/                # Prerequisites (multinetpolicy-configs namespace)
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   └── managedclustersetbinding.yaml
├── policies/
│   ├── kustomization.yaml
│   ├── acm-placements/          # Reusable placement definitions
│   │   ├── env-bound-placement.yml
│   │   ├── env-bound-hub-placement.yml
│   │   └── env-bound-nohub-placement.yml
│   ├── multinetwork-policies-kustomize/  # Scenario 1: Kustomize-only
│   │   ├── generator.yml        # PolicyGenerator (no hub templates)
│   │   ├── README.md
│   │   ├── nad-policies/        # One directory per NAD
│   │   │   ├── tenant-blue/     # Split rule files merged via musthave
│   │   │   └── tenant-red/      # Deny-all (ingress + egress)
│   │   └── nad-ns-policies/     # Namespace-scoped overrides
│   └── multinetwork-policies/   # Scenario 2: MultiNetworkPolicy from ConfigMap
│       ├── generator.yml        # PolicyGenerator with hubTemplateOptions
│       ├── README.md
│       ├── multinetworkpolicy-from-configmap.yml  # Core template
│       └── hub-template-auth/   # RBAC for hub ConfigMap reads
├── environments/
│   ├── dev/                     # Dev overlay (nfs-policies-dev namespace)
│   └── prod/                    # Prod overlay (nfs-policies-prod namespace)
├── kustomize-configs/           # Shared Kustomize transformers (Component)
├── local-cluster/               # Hub cluster ManagedCluster
└── examples/                    # Example ConfigMaps for testing
```

## Implemented Policies

### Scenario 1: MultiNetworkPolicy from Kustomize Manifests

**Policy directory**: `policies/multinetwork-policies-kustomize/`

Defines MultiNetworkPolicy rules directly in git-tracked YAML files instead of
ConfigMaps. Each NAD has a directory with separate rule files that merge via `musthave`.

**Manifest structure**: One directory per NAD under `nad-policies/`, one rule file per ingress/egress set, namespace-scoped overrides under `nad-ns-policies/`
**Rule merging**: Multiple files target the same MNP name with `complianceType: musthave` — ACM additively enforces each file's rules
**Template**: Managed cluster (`{{...}}`) templates only — no hub templates
**Placement**: `env-bound-nohub-placement` (all managed clusters, not hub)
**RBAC**: None required (no hub template lookups)
**Tradeoff**: Rules are GitOps-native but require a commit + sync to update (vs ConfigMap auto-detection in Scenario 2). Removing a rule file requires deleting the MNP on managed clusters for cleanup.

### Scenario 2: MultiNetworkPolicy from ConfigMap

**Policy**: `policies/multinetwork-policies/multinetworkpolicy-from-configmap.yml`

Reads ConfigMaps from `multinetpolicy-configs` namespace on hub, creates MultiNetworkPolicies
on managed clusters in namespaces containing matching NADs.

**ConfigMap labels**: `multinetpolicy-nad=<nad-name>`, `multinetpolicy-type=ingress|egress`
**ConfigMap data**: `rules` key containing a JSON array of rule objects
**Rule fields**: `port` (required), `protocol` (required), `cidr` (optional), `except` (optional JSON array)
**Template**: Mixed hub (`{{hub...hub}}`) + managed cluster (`{{...}}`) templates
**Placement**: `env-bound-nohub-placement` (all managed clusters, not hub)
**RBAC**: `hub-template-auth/` provides ServiceAccount for hub ConfigMap lookups

## Key Patterns to Follow

1. **Single source of truth**: Define policies once, deploy to multiple environments via Kustomize overlays — never duplicate policies across environments.
2. **Hierarchical configuration**: Use Kustomize bases and overlays for shared/team-specific resources.
3. **Dynamic templating**: Prefer ACM template functions over static manifests when resources depend on cluster state.
4. **Policy dependencies**: Use `dependencies` and `extraDependencies` to enforce ordering (e.g., operator installed before operand configured).
5. **Feature flags over directories**: Control policy targeting with ManagedCluster labels rather than separate policy copies.
6. **Enforce with care**: Use `enforce` for desired-state management, `inform` for health checks and auditing.
7. **Infrastructure awareness**: Use templates to conditionally configure resources based on cluster state (e.g., infra nodes present).
8. **Hub template auth**: When using `{{hub lookup ... hub}}` across namespaces, provide a ServiceAccount via `hubTemplateOptions.serviceAccountName` with appropriate RBAC.
