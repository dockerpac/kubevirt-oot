# Pre-requisites

- Install k0rdent-enterprise-0.2.0-alpha4
- Install HCO (Hyperconverged Operator)

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
  name: oot-repo-pac
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/managed: "true"
spec:
  type: oci
  url: 'oci://registry.k8s.pac.dockerps.io/oot'
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
        name: oot-repo-pac
EOF
```

## Add kubevirt provider to Release

```sh
kubectl patch release kcm-0-2-0-alpha4 -n k0rdent --type=json -p='[{"op": "add", "path": "/spec/providers/-", "value": {"name": "cluster-api-provider-kubevirt", "template": "cluster-api-provider-kubevirt-0-3-0"}}]'
```

## Add kubevirt provider to Management

```sh
kubectl patch management kcm -n k0rdent --type=json -p='[{"op": "add", "path": "/spec/providers/-", "value": {"name": "cluster-api-provider-kubevirt", "config": {"images": {"kubevirtManager": {"repo": "registry.k8s.pac.dockerps.io/oot"}}}}}]'
```

## Install ClusterTemplate (Existing OOT ClusterTemplate from catalog)

```sh
kubectl apply -f - <<EOF
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: oot-repo
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/managed: "true"
spec:
  type: oci
  url: 'oci://ghcr.io/k0rdent/catalog/charts'
  interval: 10m0s
---
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterTemplate
metadata:
  name: kubevirt-standalone-cp-0-2-0
  namespace: kcm-system
  annotations:
    helm.sh/resource-policy: keep
spec:
  helm:
    chartSpec:
      chart: kubevirt-standalone-cp
      version: 0.2.0
      interval: 10m0s
      sourceRef:
        kind: HelmRepository
        name: oot-repo
EOF
```

# Create Child Cluster

```sh
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: Secret
metadata:
  name: kubevirt-config
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/component: "kcm"
---
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: Credential
metadata:
  name: kubevirt-cluster-identity-cred
  namespace: kcm-system
spec:
  description: KubeVirt credentials
  identityRef:
    apiVersion: v1
    kind: Secret
    name: kubevirt-config
    namespace: kcm-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubevirt-config-resource-template
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/component: "kcm"
  annotations:
    projectsveltos.io/template: "true"
EOF
```

```sh
kubectl apply -f - <<EOF
---
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: kubevirt-demo
  namespace: kcm-system
spec:
  template: kubevirt-standalone-cp-0-2-0
  credential: kubevirt-cluster-identity-cred
  config:
    controlPlaneNumber: 1
    workersNumber: 1
    controlPlane:
      preStartCommands:
        - passwd -u root
        - echo "root:root" | chpasswd
    worker:
      preStartCommands:
        - passwd -u root
        - echo "root:root" | chpasswd
EOF
```