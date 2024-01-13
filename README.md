# vault-k8s-setup
This project shows how to setup vault in HA mode with raft storage in Kubernetes cluster with automated unseal.

This project used bank-vaults. But since support of this utility for integrated raft storage is not working in a stable manner I don't encourage anyone for using this setup. You may of course try. 

## Prerequisites

1. Kind cluster (any other kubernetes cluster should be fine as well)
2. [cert-manager](https://cert-manager.io/docs/installation/) installed
3. Ingress controller (for example kong)

## Installation

### Kind cluster

Create cluster with exposed ingress:

```shell
kind create cluster --config kind-config.yml
```

### Cert manager

```shell
helm repo add cert-manager https://charts.jetstack.io
helm repo update
helm upgrade --install cert-manager cert-manager/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true \
  --wait
```

### Selfsigned cert issuer

```shell
kubectl apply -f manifests/cert-issuer.yml
```

### Install kong ingress controller

```shell
helm repo add kong https://charts.konghq.com 
helm repo update
helm upgrade --install kong kong/kong \
  --namespace kong --create-namespace \
  --values kong/values.yml \
  --wait
kubectl apply -f manifests/kong-cert.yml
```

### Install vault prerequisites

This step installs:
* RBAC role and role binding needed for bank-vaults operator to create secret holding unseal keys and root token.
* certificate for internal vault communication

```shell
kubectl create namespace vault
kubectl apply -f manifests/vault-cert.yml
kubectl apply -f manifests/bank-vaults-rbac.yml
```

### Install vault

```shell
helm repo add hashicorp https://helm.releases.hashicorp.com/
helm repo update
helm upgrade --install --namespace vault \
  vault hashicorp/vault \
  --values vault/values.yml \
  --wait               
```

## Access vault
Add to your `/etc/hosts` file:
```
127.0.0.1 vault.cluster.local
```

Set access parameters:
```shell
export VAULT_ADDR=https://vault.cluster.local:32443
export VAULT_TOKEN=`kubectl get secret -n vault vault-unseal-keys -ojson | jq -r '.data["vault-root"]' | base64 -d`
```

Execute commands:
```shell
vault secrets list -tls-skip-verify
```

## Cleanup

```shell
kind delete cluster
```