# Pre-requisites

- Install k0rdent-enterprise-0.2.0-alpha4
- Install HCO (Hyperconverged Operator)
- Helm charts in `<custom registry>/k0rdent-enterprise` : 
 - oci://ghcr.io/dockerpac/kubevirt-oot/cluster-api-provider-kubevirt --version 0.3.0
- Images in `<custom registry>/k0rdent-enterprise` : 
 - quay.io/capk-manager:v0.1.9

# Install Kubevirt Provider

## Add to provider list

```sh
cat << 'EOF' | jq -Rs '{data:{"kubevirt.yml":.}}' | kubectl patch configmap providers -n kcm-system --type=merge --patch-file /dev/stdin
# SPDX-License-Identifier: Apache-2.0
name: kubevirt
clusterGVKs:
  - group: infrastructure.cluster.x-k8s.io
    version: v1alpha1
    kind: KubevirtCluster
clusterIdentityKinds:
  - Secret
EOF
```

## Add Permissions to manage Kubevirtclusters objects to kcm-controller
```sh
kubectl get clusterrole kcm-manager-role -o json | jq '.rules |= map(if .apiGroups[0] == "infrastructure.cluster.x-k8s.io" and .resources[0] == "awsclusters" then .resources += ["kubevirtclusters"] else . end)' | kubectl apply -f -
```

## Restart kcm-controller-manager

```sh
kubectl -n kcm-system rollout restart deploy/kcm-controller-manager
```

## Install ProviderTemplate (NEW)

```sh
kubectl apply -f - <<EOF
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: kubevirt-oot
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/managed: "true"
spec:
  type: oci
  url: 'oci://ghcr.io/dockerpac/kubevirt-oot'
  interval: 10m0s
---
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ProviderTemplate
metadata:
  name: cluster-api-provider-kubevirt-0-3-0
  annotations:
    helm.sh/resource-policy: keep
spec:
  helm:
    chartSpec:
      chart: cluster-api-provider-kubevirt
      version: 0.3.0
      interval: 10m0s
      sourceRef:
        kind: HelmRepository
        name: kubevirt-oot
EOF
```

## Add kubevirt provider to Release

```sh
kubectl patch release kcm-0-2-0-alpha4 --type=json -p='[{"op": "add", "path": "/spec/providers/-", "value": {"name": "cluster-api-provider-kubevirt", "template": "cluster-api-provider-kubevirt-0-3-0"}}]'
```

## Add kubevirt provider to Management

```sh
# change <custom registry>/k0rdent-enterprise
kubectl patch management kcm --type=json -p='[{"op": "add", "path": "/spec/providers/-", "value": {"name": "cluster-api-provider-kubevirt", "config": {"images": {"kubevirtManager": {"repo": "<custom registry>/k0rdent-enterprise"}}}}}]'
```

