# MultiNetworkPolicy from Kustomize Manifests

Creates MultiNetworkPolicies on managed clusters using rules defined directly in
git-tracked YAML files. For each NetworkAttachmentDefinition (NAD) found on
managed clusters whose name matches a policy manifest, a MultiNetworkPolicy is
created in that NAD's namespace.

This is Scenario 1, an alternative to Scenario 2 (ConfigMap-based) that eliminates
hub templates, ConfigMaps, and hub RBAC — rules are embedded in the policy
manifests themselves and managed entirely through Kustomize and git.

## Dependencies

- NetworkAttachmentDefinitions must exist on managed clusters

## How It Works

1. Each NAD has a directory under `nad-policies/` containing one or more rule files
2. Each rule file targets the **same** MultiNetworkPolicy (by name) but only
   specifies its own subset of rules using `complianceType: musthave`
3. ACM enforces each rule file independently — `musthave` ensures the specified
   rules exist in the MultiNetworkPolicy without removing rules from other files
4. The result is a single MultiNetworkPolicy per NAD with all rules merged
5. Managed-cluster templates (`{{...}}`) discover local NADs dynamically

### Rule Merging with `musthave`

Unlike Scenario 2 (which merges rules in a hub template), Scenario 1 relies on
ACM's `musthave` compliance type. Each rule file enforces that its rules are
present in the MultiNetworkPolicy — ACM adds any missing rules without removing
existing ones from other files.

This means:
- **Adding rules**: Create a new rule file, add it to `generator.yml`, commit
- **Modifying rules**: Edit the rule file, commit
- **Removing rules**: Delete the rule file and remove from `generator.yml`.
  Then delete the MultiNetworkPolicy on managed clusters so ACM recreates it
  with only the remaining rules:
  ```bash
  oc delete multinetworkpolicy acm-mnp-<nad-name> -n <namespace>
  ```

## Directory Structure

```
multinetwork-policies-kustomize/
├── kustomization.yaml
├── generator.yml                          # PolicyGenerator
├── nad-policies/                          # One directory per NAD
│   ├── tenant-blue/                       # Rules for tenant-blue NAD
│   │   ��── ingress-allow-web.yml          # TCP 443, 8080 from specific CIDRs
│   │   ├─�� ingress-allow-api.yml          # TCP 9090 from 10.1.0.0/16
│   │   └── egress-allow-dns.yml           # UDP/TCP 53 to 0.0.0.0/0
│   └── tenant-red/                        # Rules for tenant-red NAD
│       ├── ingress-deny-all.yml           # Deny all ingress
│       └── egress-deny-all.yml            # Deny all egress
└── nad-ns-policies/                       # Namespace-scoped overrides
    └── tenant-blue-web-frontend.yml       # TCP 4430 in web-frontend only
```

## Adding Rules to an Existing NAD

1. Create a new rule file in the NAD's directory (e.g., `nad-policies/tenant-blue/egress-allow-db.yml`)
2. Add a manifest entry to the NAD's policy in `generator.yml`
3. Commit and sync via ArgoCD

### Rule File Template

```yaml
---
object-templates-raw: |
  {{- range $nad := (lookup "k8s.cni.cncf.io/v1" "NetworkAttachmentDefinition" "" "").items }}
  {{- if eq $nad.metadata.name "<nad-name>" }}
  - complianceType: musthave
    objectDefinition:
      apiVersion: k8s.cni.cncf.io/v1beta1
      kind: MultiNetworkPolicy
      metadata:
        name: acm-mnp-<nad-name>
        namespace: '{{ $nad.metadata.namespace }}'
        annotations:
          k8s.v1.cni.cncf.io/policy-for: '{{ $nad.metadata.namespace }}/<nad-name>'
        labels:
          managed-by: acm-multinetpolicy
      spec:
        podSelector: {}
        policyTypes:
          - Ingress
        ingress:
          - from:
              - ipBlock:
                  cidr: '<cidr>'
            ports:
              - port: <port>
                protocol: <TCP|UDP|SCTP>
  {{- end }}
  {{- end }}
```

## Adding a New NAD

1. Create a new directory under `nad-policies/<nad-name>/`
2. Add one or more rule files using the template above
3. Add a new policy entry in `generator.yml` with a manifest per rule file
4. Commit and sync via ArgoCD

### Deny-All Pattern

To deny all traffic for a direction, include the `policyTypes` entry but omit
the corresponding `ingress` or `egress` field:

```yaml
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

With `musthave`, this ensures the policyType is present. If other rule files add
ingress allow rules, they create exceptions to the deny-all baseline — matching
standard Kubernetes NetworkPolicy union semantics.

### Namespace-Scoped Rules

To restrict rules to a specific namespace, place the file under `nad-ns-policies/`
and change the lookup to target that namespace:

```yaml
{{- range $nad := (lookup "k8s.cni.cncf.io/v1" "NetworkAttachmentDefinition" "web-frontend" "").items }}
```

Kubernetes NetworkPolicy union semantics apply — global and namespace-scoped
policies are additive.

## Comparison with Scenario 2 (ConfigMap)

| Dimension | Scenario 1 (Kustomize) | Scenario 2 (ConfigMap) |
|---|---|---|
| Rule source | Git YAML files | ConfigMaps on hub |
| Rule merging | ACM `musthave` enforcement | Hub template aggregation |
| Runtime changes | Git commit + sync | Update ConfigMap, auto-detected |
| Rule removal | Remove file + delete MNP on clusters | Remove ConfigMap, auto-detected |
| Hub RBAC | Not needed | Required |
| Template complexity | Low (managed only) | High (hub + managed) |
| Git auditability | Full git history | ConfigMaps may drift |
| Adding a rule | Add YAML + update generator.yml | Create ConfigMap |

## Details

ACM Minimal Version: 2.12

Documentation:
- [MultiNetworkPolicy](https://docs.openshift.com/container-platform/latest/networking/multiple_networks/configuring-multi-network-policy.html)
- [ACM Policy Templates](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.12/html/governance/governance#hub-cluster-templates)

---

**Notes:**
- All rule files for a NAD target the same MultiNetworkPolicy name (`acm-mnp-<nad-name>`)
- `musthave` compliance ensures rules accumulate additively across files
- Removing a rule file requires deleting the MNP on managed clusters for cleanup
- Uses `podSelector: {}` to apply to all pods in the namespace
- No hub templates or ServiceAccount RBAC required
