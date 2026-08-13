# Step 1: Create the Provisioning Script

*Run this block to write the provision-namespace.sh script:*

```
cat <<'EOF' > provision-namespace.sh
#!/bin/bash
# Usage: ./provision-namespace.sh <namespace> <team> <tier>
NS=$1
TEAM=$2
TIER=$3

if [ -z "$NS" ] || [ -z "$TEAM" ] || [ -z "$TIER" ]; then
  echo "Usage: $0 <namespace> <team> <tier>"
  exit 1
fi

# Tier quotas
case $TIER in
  small)  CPU=4; MEM=8Gi; PODS=20 ;;
  medium) CPU=8; MEM=16Gi; PODS=50 ;;
  large)  CPU=20; MEM=40Gi; PODS=100 ;;
  *) echo "Invalid tier! Choose small, medium, or large."; exit 1 ;;
esac

echo "Provisioning namespace: $NS for team: $TEAM (tier: $TIER)"

# 1. Create namespace with PSA labels
kubectl create namespace "$NS"
kubectl label namespace "$NS" \
  team="$TEAM" \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest --overwrite

# 2. ResourceQuota
kubectl create resourcequota quota \
  --hard="requests.cpu=${CPU},requests.memory=${MEM},pods=${PODS}" \
  -n "$NS"

# 3. Default NetworkPolicy
cat <<NETPOL | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: $NS
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: $NS
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - ports:
    - port: 53
      protocol: UDP
NETPOL

# 4. RBAC
kubectl create role developer \
  --verb=get,list,watch,create,update,patch \
  --resource=pods,deployments,services,configmaps \
  -n "$NS"

kubectl create clusterrolebinding "${NS}-team-admin" \
  --clusterrole=admin \
  --group="${TEAM}-admins" \
  --dry-run=client -o yaml | kubectl apply -f -

echo "Namespace $NS provisioned successfully"
EOF

chmod +x provision-namespace.sh
```

## Step 2: Provision the Namespace

*Run the script to create the data-science-dev namespace for team data-science at the medium tier:*

```
./provision-namespace.sh data-science-dev data-science medium
```

**Expected Output:**

```
Provisioning namespace: data-science-dev for team: data-science (tier: medium)
namespace/data-science-dev created
namespace/data-science-dev labeled
resourcequota/quota created
networkpolicy.networking.k8s.io/default-deny created
networkpolicy.networking.k8s.io/allow-dns created
role.rbac.authorization.k8s.io/developer created
clusterrolebinding.rbac.authorization.k8s.io/data-science-dev-team-admin created
Namespace data-science-dev provisioned successfully
```

## Step 3: Verify the Provisioned Resources

*Verify that all baseline objects (Quota, NetworkPolicies, Role, RoleBinding) were properly applied to the new namespace:*

```
kubectl get quota,networkpolicy,role,rolebinding -n data-science-dev
```

**Expected Output:**

```
NAME                  REQUEST                                                  LIMIT   AGE
resourcequota/quota   pods: 0/50, requests.cpu: 0/8, requests.memory: 0/16Gi           14s

NAME                                           POD-SELECTOR   AGE
networkpolicy.networking.k8s.io/allow-dns      <none>         14s
networkpolicy.networking.k8s.io/default-deny   <none>         14s

NAME                                       CREATED AT
role.rbac.authorization.k8s.io/developer   2026-08-13T06:19:28Z
```

