# MultiNetworkPolicy from ConfigMap

Creates MultiNetworkPolicies dynamically based on ConfigMaps stored on the ACM
hub cluster. For each NetworkAttachmentDefinition (NAD) found on managed
clusters whose name matches a ConfigMap label, a MultiNetworkPolicy is created
in that NAD's namespace with all matching ingress and egress rules merged.

## Dependencies

- NetworkAttachmentDefinitions must exist on managed clusters
- ConfigMaps must be created in the `multinetpolicy-configs` namespace on the hub

## ConfigMap Format

Each ConfigMap contains one or more rules as a JSON array in the `rules` data key.
Labels on the ConfigMap determine which NAD is targeted and the rule direction.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <descriptive-name>
  namespace: multinetpolicy-configs
  labels:
    multinetpolicy-nad: <nad-name>          # matches NAD metadata.name
    multinetpolicy-type: ingress            # or "egress"
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
        "port": "8080"
      }
    ]
```

### Labels

| Label | Required | Values | Purpose |
|---|---|---|---|
| `multinetpolicy-nad` | Yes | NAD name (e.g., `tenant-blue`) | Identifies which NAD this rule targets |
| `multinetpolicy-type` | Yes | `ingress` or `egress` | Determines rule direction |
| `multinetpolicy-namespace` | No | Namespace name (e.g., `web-frontend`) | Restricts policy to a specific namespace |

### Rule Fields

| Field | Required | Description |
|---|---|---|
| `port` | Yes | Port number (string) |
| `protocol` | Yes | `TCP`, `UDP`, or `SCTP` |
| `cidr` | No | IP CIDR block — if omitted, rule matches all sources/destinations |
| `except` | No | JSON array of exception CIDRs (only valid when `cidr` is set) |

### Deny-All Pattern

To deny all ingress or egress traffic for a NAD, create a ConfigMap with an empty
rules list (`"[]"`). The presence of the ConfigMap causes the corresponding
`policyTypes` entry (`Ingress` or `Egress`) to be added to the MultiNetworkPolicy
without any allow rules, which blocks all traffic for that direction.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tenant-red-ingress-deny-all
  namespace: multinetpolicy-configs
  labels:
    multinetpolicy-nad: tenant-red
    multinetpolicy-type: ingress
data:
  rules: "[]"
```

### Namespace-Scoped Rules

By default, ConfigMaps apply globally — rules are enforced in every namespace where
the matching NAD exists. To restrict a rule to a specific namespace, add the
`multinetpolicy-namespace` label. This creates a separate MultiNetworkPolicy
(named `acm-mnp-ns-<configmap-name>`) that only applies in the target namespace.

Kubernetes NetworkPolicy union semantics apply: global and namespace-scoped policies
are additive. A global deny-all combined with a namespace-scoped allow rule results
in only the allowed traffic passing in that namespace.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tenant-blue-ns-web-ingress-allow-https
  namespace: multinetpolicy-configs
  labels:
    multinetpolicy-nad: tenant-blue
    multinetpolicy-type: ingress
    multinetpolicy-namespace: web-frontend
data:
  rules: |
    [
      {
        "protocol": "TCP",
        "port": "443",
        "cidr": "0.0.0.0/0"
      }
    ]
```

## How It Works

1. Hub templates read all ConfigMaps labeled `multinetpolicy-nad` from `multinetpolicy-configs`
2. Global ConfigMaps (without `multinetpolicy-namespace`) are grouped by NAD name and direction
3. Managed cluster templates discover local NADs matching those names
4. For each matching NAD, a global MultiNetworkPolicy is created in that namespace
5. All global ingress rules merge into the `ingress` field, all egress rules into `egress`
6. Namespace-scoped ConfigMaps each generate a separate MultiNetworkPolicy only in the target namespace

## Details

ACM Minimal Version: 2.12

Documentation:
- [MultiNetworkPolicy](https://docs.openshift.com/container-platform/latest/networking/multiple_networks/configuring-multi-network-policy.html)
- [ACM Hub Templates](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.12/html/governance/governance#hub-cluster-templates)

---

**Notes:**
- One global MultiNetworkPolicy per NAD per namespace is created
- Global policies are named `acm-mnp-<nad-name>`, namespace-scoped policies are named `acm-mnp-ns-<configmap-name>`
- Uses `podSelector: {}` to apply to all pods in the namespace
- The `k8s.v1.cni.cncf.io/policy-for` annotation is set to `<namespace>/<nad-name>`
- Hub templates re-evaluate periodically; new ConfigMaps are picked up automatically
- Requires `hubTemplateOptions.serviceAccountName` for cross-namespace ConfigMap reads
