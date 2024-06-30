---
archetype: lecture-cg
title: "Cluster-API Demo"
menuTitle: "Cluster-API Demo"
author: "Leon Welchert"
weight: 6
tldr: |
  A short practical intro to deploying Kubernetes with Cluster API
---

## Prerequisites
- Linux with basic tools
- helm
- kind
- kubectl
- k9s
- clusterctl
- Hetzner Cloud account

## Setup Hetzner Cloud Project
- visit `https://console.hetzner.cloud`
- create new project
  - Click on project, select "Security" in left navigation bar
  - Add SSH Key, set default
  - Create RW API token (save for future use)

## Setup Management Cluster

The following commands use the default namespace for demonstration purposes
```{.bash size="scriptsize"}
kind create cluster
kubectl config use-context kind-kind

# this will install resources to the currently selected kubectl context
clusterctl init --core cluster-api --bootstrap kubeadm --control-plane kubeadm --infrastructure hetzner
# take a look at the created namespaces and the deployments within 
# to see the various Cluster API components
kubectl get ns
kubectl get pod -A
```

Some variables need to be set for the following steps:
```{.bash size="scriptsize"}
export HCLOUD_SSH_KEY="your-throwaway-ssh-key" \
export CLUSTER_NAME="my-cluster" \
export HCLOUD_REGION="fsn1" \
export CONTROL_PLANE_MACHINE_COUNT="3" \
export WORKER_MACHINE_COUNT="2" \
export KUBERNETES_VERSION="1.29.4" \
export HCLOUD_CONTROL_PLANE_MACHINE_TYPE="cpx31" \
export HCLOUD_WORKER_MACHINE_TYPE="cpx31" \
export HCLOUD_TOKEN="<TOKEN_GOES_HERE>" \
```

## Applying the Cluster-API Custom Resources
```{.bash size="scriptsize"}
kubectl create secret generic hetzner --from-literal=hcloud=$HCLOUD_TOKEN

clusterctl generate cluster my-cluster --kubernetes-version v1.29.4 \ 
    --control-plane-machine-count=3 --worker-machine-count=1  > my-cluster.yaml
# search the file for HCloudMachineTemplate to see what the machines will be based on
# search the file for preKubeadmCommands to see the cloud init setup of k8s
k9s --readonly --headless -c machine
kubectl apply -f my-cluster.yaml
```

```{.bash size="scriptsize"}
kubectl get cluster
kubectl describe cluster my-cluster 
kubectl get kubeadmcontrolplane
kubectl get machinedeployment
```

Afterwards, get the kubeconfig of the workload cluster and set up k9s to watch for nodes being created
```{.bash size="scriptsize"}
clusterctl get kubeconfig my-cluster > /tmp/workload-kubeconfig
k9s --readonly --headless --kubeconfig /tmp/workload-kubeconfig -c node
```

##  Installing the CNI
Our cluster still needs a CNI for networking, let's deploy one
```{.bash size="scriptsize"}
# Install CNI
helm repo add cilium https://helm.cilium.io/

# grab templates from syself for hetzner-cloud-cilium setup
# the CAPH repo by Hetzner assumes you want to use Flanel
# but there is another one that provides templates for Cilium
git clone https://github.com/syself/cluster-api-provider-hetzner.git
cd cluster-api-provider-hetzner


# will be applied to workload cluster because of the KUBECONFIG env var
KUBECONFIG=$CAPH_WORKER_CLUSTER_KUBECONFIG helm upgrade --install cilium cilium/cilium --version 1.14.4 \
-f templates/cilium/cilium.yaml
```


## Installing - CCM
Without a Cloud Controller Manager deployment in the workload cluster, nodes will not be correctly labelled with the provider IDs and Cluster API will not properly scale the cluster
```{.bash size="scriptsize"}
# will be applied to hetzner cluster because of the KUBECONFIG env var
kubectl --kubeconfig $CAPH_WORKER_CLUSTER_KUBECONFIG -n kube-system \
    create secret generic hcloud --from-literal=token=$HCLOUD_TOKEN \
    --from-literal=network=my-cluster-net

helm repo add hcloud https://charts.hetzner.cloud
helm repo update hcloud
KUBECONFIG=$CAPH_WORKER_CLUSTER_KUBECONFIG helm install hccm hcloud/hcloud-cloud-controller-manager \
	--namespace kube-system \
	--set secret.name=hetzner \
	--set secret.tokenKeyName=hcloud \

```

## Playing around
Scale the cluster
```{.bash size="scriptsize"}
kubectl --kubeconfig /tmp/workload-kubeconfig get node
# set replicas to two
kubectl edit  machinedeployment my-cluster-md-0
kubectl --kubeconfig /tmp/workload-kubeconfig get node
```

Delete machines
```{.bash size="scriptsize"}
# watch using k9s in another tab
kubectl get machine
kubectl delete machine
```

## Links and Literature
- https://cluster-api.sigs.k8s.io/user/quick-start
- https://github.com/syself/cluster-api-provider-hetzner/blob/main/docs/topics/quickstart.md
- https://github.com/cncf/tag-security/blob/main/security-whitepaper/v2/cloud-native-security-whitepaper.md
- https://github.com/kubernetes-sigs/cluster-api
- https://github.com/derailed/k9s
- https://github.com/helm/helm
- https://github.com/kubernetes-sigs/kind
- https://github.com/hetznercloud/hcloud-cloud-controller-manager
- https://github.com/cilium/cilium
