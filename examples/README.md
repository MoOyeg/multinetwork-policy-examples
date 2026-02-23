# Example ConfigMaps

These ConfigMaps demonstrate the format expected by the MultiNetworkPolicy ACM policy.
Apply them to the `multinetpolicy-configs` namespace on the ACM hub cluster.

## Usage

```bash
# Apply all example ConfigMaps
oc apply -k examples/configmaps/

# Or apply individually
oc apply -f examples/configmaps/tenant-blue-ingress-allow-web.yml
```

## What These Create

When managed clusters have NetworkAttachmentDefinitions named `tenant-blue`
or `tenant-red`, the following MultiNetworkPolicies will be created:

**acm-mnp-tenant-blue** - In every namespace where a `tenant-blue` NAD exists:
- Ingress: allows TCP/443 from 10.0.0.0/24 (except 10.0.0.5/32, 10.0.0.6/32)
- Ingress: allows TCP/8080 from 172.16.0.0/16
- Ingress: allows TCP/9090 from 10.1.0.0/16
- Egress: allows UDP/53 to 0.0.0.0/0
- Egress: allows TCP/53 to 0.0.0.0/0

**acm-mnp-tenant-red** - In every namespace where a `tenant-red` NAD exists:
- Ingress: allows TCP/0 from all sources (no CIDR restriction)
