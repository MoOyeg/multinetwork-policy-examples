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
│   └── multinetwork-policies/       # MultiNetworkPolicy from ConfigMap policy
├── environments/
│   ├── dev/                         # Dev environment overlay
│   └── prod/                        # Prod environment overlay
├── kustomize-configs/               # Shared Kustomize transformers
├── local-cluster/                   # Hub cluster configuration
└── examples/                        # Example ConfigMaps for testing
```

## Scenario 1: MultiNetworkPolicy from ConfigMap

An ACM policy that reads from ConfigMaps on the hub cluster (labeled with a NAD name) and dynamically creates MultiNetworkPolicies on managed clusters in every namespace that contains a matching NetworkAttachmentDefinition (NAD).

### How It Works

1. Create ConfigMaps in the `multinetpolicy-configs` namespace on the hub, labeled with:
   - `multinetpolicy-nad: <nad-name>` — identifies which NAD this rule targets
   - `multinetpolicy-type: ingress|egress` — determines rule direction
2. The ACM policy discovers all NADs on managed clusters
3. For each NAD matching a ConfigMap label, a `MultiNetworkPolicy` is created in that namespace
4. All ingress/egress rules from matching ConfigMaps are merged into a single policy per NAD

### ConfigMap Format

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tenant-blue-ingress-allow-web
  namespace: multinetpolicy-configs
  labels:
    multinetpolicy-nad: tenant-blue
    multinetpolicy-type: ingress
data:
  cidr: "10.0.0.0/24"
  except: "10.0.0.5/32,10.0.0.6/32"   # optional
  port: "443"                           # optional
  protocol: "TCP"                       # optional, defaults to TCP
```

### Quick Start

```bash
# 1. Grant subscription-admin privileges
oc adm policy add-cluster-role-to-user open-cluster-management:subscription-admin $(oc whoami)

# 2. Apply prerequisites (creates multinetpolicy-configs namespace)
oc apply -k requirements/

# 3. Label managed clusters into the secure-clusterset ClusterSet
oc label managedcluster <cluster-name> cluster.open-cluster-management.io/clusterset=secure-clusterset

# 4. Apply example ConfigMaps
oc apply -k examples/configmaps/

# 5. Deploy policies to dev environment
kustomize build --enable-alpha-plugins environments/dev/ | oc apply -f -

# 6. Verify policy created
oc get policy -n multinetwork-policies-dev multinetworkpolicy-from-configmap

# 7. Check MultiNetworkPolicies on managed clusters
oc get multi-networkpolicy -A -l managed-by=acm-multinetpolicy
```

### Environment Promotion

Policies are deployed per environment using Kustomize overlays:

- **Dev**: `kustomize build --enable-alpha-plugins environments/dev/`
- **Prod**: `kustomize build --enable-alpha-plugins environments/prod/`

Each environment has its own namespace (`multinetwork-policies-dev`, `multinetwork-policies-prod`), ManagedClusterSet, and PolicySet suffix. See the [acm-policy-samples](https://github.com/bry-tam/acm-policy-samples) reference for the full release management workflow.

## Adding New Rules

To add a new MultiNetworkPolicy rule:

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
  cidr: "10.1.0.0/16"
  port: "5432"
  protocol: "TCP"
EOF
```

## References

- [acm-onboarding-examples](https://github.com/MoOyeg/acm-onboarding-examples) — ACM onboarding patterns
- [acm-policy-samples](https://github.com/bry-tam/acm-policy-samples) — Production ACM policy structure
- [MultiNetworkPolicy docs](https://docs.openshift.com/container-platform/latest/networking/multiple_networks/configuring-multi-network-policy.html)
- [ACM PolicyGenerator](https://github.com/stolostron/policy-generator-plugin)
