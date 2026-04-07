# multinetwork-policy-examples

ACM (Red Hat Advanced Cluster Management) policy examples for managing NetworkPolicies and MultiNetworkPolicies across OpenShift clusters.

## Requirements

- Red Hat Advanced Cluster Management 2.12+ installed
- User with cluster-admin and subscription-admin privileges
- Kustomize with [PolicyGenerator plugin](https://github.com/stolostron/policy-generator-plugin) installed
- Managed clusters with [MultiNetworkPolicy support enabled](https://docs.openshift.com/container-platform/latest/networking/multiple_networks/configuring-multi-network-policy.html)

## Repository Structure

```
.
├── requirements/                    # Prerequisites (ConfigMap namespace)
├── policies/
│   ├── acm-placements/              # Reusable placement definitions
│   ├── multinetwork-policies/       # Scenario 2: MultiNetworkPolicy from ConfigMap
│   └── multinetwork-policies-kustomize/  # Scenario 1: MultiNetworkPolicy from Kustomize
├── environments/
│   ├── dev/                         # Dev environment overlay
│   └── prod/                        # Prod environment overlay
├── kustomize-configs/               # Shared Kustomize transformers
├── local-cluster/                   # Hub cluster configuration
├── examples/                        # Example ConfigMaps for testing
└── blog/                            # Blog post on the approach
```

## Scenario 1: MultiNetworkPolicy from Kustomize Manifests

Defines MultiNetworkPolicy rules directly in git-tracked YAML files. Each NAD gets its own directory with separate rule files that merge via ACM's `musthave` compliance type. No hub templates, ConfigMaps, or hub RBAC required.

### How It Works

1. Define rules in YAML files under `policies/multinetwork-policies-kustomize/nad-policies/<nad-name>/`
2. Each file uses managed-cluster templates to discover local NADs by name
3. Multiple rule files target the same MultiNetworkPolicy with `musthave` — ACM merges them additively
4. No hub templates, ConfigMaps, or hub RBAC required

See [policies/multinetwork-policies-kustomize/README.md](policies/multinetwork-policies-kustomize/README.md) for full details, templates, and examples.

### Quick Start

```bash
# 1. Grant subscription-admin privileges
oc adm policy add-cluster-role-to-user open-cluster-management:subscription-admin $(oc whoami)

# 2. Apply prerequisites (creates multinetpolicy-configs namespace)
oc apply -k requirements/

# 3. Label managed clusters into the secure-clusterset ClusterSet
oc label managedcluster <cluster-name> cluster.open-cluster-management.io/clusterset=secure-clusterset

# 4. Deploy policies to dev environment
kustomize build --enable-alpha-plugins environments/dev/ | oc apply -f -

# 5. Verify policies created
oc get policy -n multinetwork-policies-dev -l policy_gen.name=multinetwork-policies-kustomize

# 6. Check MultiNetworkPolicies on managed clusters
oc get multi-networkpolicy -A -l managed-by=acm-multinetpolicy
```

### Environment Promotion

Policies are deployed per environment using Kustomize overlays:

- **Dev**: `kustomize build --enable-alpha-plugins environments/dev/`
- **Prod**: `kustomize build --enable-alpha-plugins environments/prod/`

Each environment has its own namespace (`multinetwork-policies-dev`, `multinetwork-policies-prod`), ManagedClusterSet, and PolicySet suffix. See the [acm-policy-samples](https://github.com/bry-tam/acm-policy-samples) reference for the full release management workflow.

## Scenario 2: MultiNetworkPolicy from ConfigMap

An ACM policy that reads from ConfigMaps on the hub cluster (labeled with a NAD name) and dynamically creates MultiNetworkPolicies on managed clusters in every namespace that contains a matching NetworkAttachmentDefinition (NAD).

### How It Works

1. Create ConfigMaps in the `multinetpolicy-configs` namespace on the hub, labeled with:
   - `multinetpolicy-nad: <nad-name>` — identifies which NAD this rule targets
   - `multinetpolicy-type: ingress|egress` — determines rule direction
2. The ACM policy discovers all NADs on managed clusters
3. For each NAD matching a ConfigMap label, a `MultiNetworkPolicy` is created in that namespace
4. All ingress/egress rules from matching ConfigMaps are merged into a single policy per NAD

### ConfigMap Format

Each ConfigMap contains rules as a JSON array in the `rules` data key. Labels identify the target NAD and direction.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: vlan-10-ingress-allow-web
  namespace: multinetpolicy-configs
  labels:
    multinetpolicy-nad: vlan-10
    multinetpolicy-type: ingress
data:
  rules: |
    [
      {
        "protocol": "TCP",
        "port": "443",
        "cidr": "10.0.0.0/24",
        "except": ["10.0.0.5/32", "10.0.0.6/32"]
      },
      {
        "protocol": "TCP",
        "port": "8080",
        "cidr": "172.16.0.0/16"
      }
    ]
```

| Rule Field | Required | Description |
|---|---|---|
| `port` | Yes | Port number (string) |
| `protocol` | Yes | `TCP`, `UDP`, or `SCTP` |
| `cidr` | No | IP CIDR block for source/destination filtering |
| `except` | No | JSON array of CIDR exclusions (only valid with `cidr`) |

An empty rules array (`"[]"`) with a `multinetpolicy-type` label creates a deny-all policy for that direction. Adding the optional `multinetpolicy-namespace` label restricts a ConfigMap to a single namespace instead of applying globally.

See [policies/multinetwork-policies/README.md](policies/multinetwork-policies/README.md) for full details.

### Key Differences from Scenario 1

| Dimension | Scenario 1 (Kustomize) | Scenario 2 (ConfigMap) |
|---|---|---|
| Rule source | Git YAML files | ConfigMaps on hub |
| Runtime changes | Git commit + ArgoCD sync | Update ConfigMap, auto-detected |
| Hub RBAC | Not needed | Required |
| Template complexity | Low (managed only) | High (hub + managed) |

## Adding New Rules

### Scenario 1 (Kustomize)

1. Create a rule file in the NAD's directory under `policies/multinetwork-policies-kustomize/nad-policies/<nad-name>/`
2. Add a manifest entry in `generator.yml`
3. Commit and sync via ArgoCD

### Scenario 2 (ConfigMap)

1. Create a ConfigMap in `multinetpolicy-configs` with the appropriate labels
2. The policy re-evaluates automatically and creates/updates MultiNetworkPolicies

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-nad-egress-allow-db
  namespace: multinetpolicy-configs
  labels:
    multinetpolicy-nad: my-nad-name
    multinetpolicy-type: egress
data:
  rules: |
    [{"protocol": "TCP", "port": "5432", "cidr": "10.1.0.0/16"}]
EOF
```

## References

- [acm-onboarding-examples](https://github.com/MoOyeg/acm-onboarding-examples) — ACM onboarding patterns
- [acm-policy-samples](https://github.com/bry-tam/acm-policy-samples) — Production ACM policy structure
- [MultiNetworkPolicy docs](https://docs.openshift.com/container-platform/latest/networking/multiple_networks/configuring-multi-network-policy.html)
- [ACM PolicyGenerator](https://github.com/stolostron/policy-generator-plugin)
