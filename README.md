# Miningcore Development

This is a guide to run the miningcore fork from 1oopio in k8s for development purposes.

I'll use the following technologies:

- minikube -> to spin up a kubernetes cluster
- argocd -> to deploy miningcore using helm charts
- kube-prometheus-stack -> to monitor miningcores performance and detect anomalies

## 
- [Miningcore Development](#miningcore-development)
  - [](#)
  - [Installation](#installation)
    - [Install dependencies](#install-dependencies)
    - [Start minikube](#start-minikube)
    - [Install argocd](#install-argocd)
      - [Start portforwarding to access argocd](#start-portforwarding-to-access-argocd)
      - [Fetch the admin password](#fetch-the-admin-password)
    - [Create a repo to store the argocd appConfigs](#create-a-repo-to-store-the-argocd-appconfigs)


## Installation
### Install dependencies
```bash
yay -S minikube kubectl kubectx k9s kubeseal github-cli
```
> This is guide is written for arch linux (btw). Adjust the dependency installation according to your distro.

### Start minikube
```bash
minikube start
```

### Install argocd
```bash
alias k=kubectl

k create ns argocd
k apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# verify
k get all -n argocd
```

#### Start portforwarding to access argocd
```bash
k port-forward svc/argocd-server -n argocd 8080:443
```

Argocd will be available at https://localhost:8080

#### Fetch the admin password
```bash
k -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

### Create a repo to store the argocd appConfigs
```
gh repo create --private miningcore-devel-configs
cd ..; gh repo clone miningcore-devel-configs; cd -;
```

