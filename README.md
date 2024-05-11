# Miningcore Development

This is a guide to run the miningcore fork from 1oopio in k8s for development purposes.

I'll use the following technologies:

- minikube -> to spin up a kubernetes cluster
- argocd -> to deploy miningcore using helm charts
- kube-prometheus-stack -> to monitor miningcores performance and detect anomalies

> [!Important]
> No part of this guide is written with security in mind. So please be more careful with secrets like password and seedphrases in production!

---

- [Miningcore Development](#miningcore-development)
  - [Installation](#installation)
    - [Install dependencies](#install-dependencies)
    - [Start minikube](#start-minikube)
      - [Enable the registry plugin](#enable-the-registry-plugin)
    - [Deploy argocd](#deploy-argocd)
      - [Start port-forwarding to access argocd](#start-port-forwarding-to-access-argocd)
      - [Fetch the admin password](#fetch-the-admin-password)
      - [Login using argocd's cli tool](#login-using-argocds-cli-tool)
    - [Create a repo to store the argocd appConfigs](#create-a-repo-to-store-the-argocd-appconfigs)
  - [Build miningcore](#build-miningcore)

---

## Installation
### Install dependencies
```bash
yay -S minikube kubectl kubectx k9s argocd github-cli
```
> This is guide is written for arch linux (btw). Adjust the dependency installation according to your distro.

### Start minikube
```bash
minikube start --driver=docker
```

#### Enable the registry plugin
```bash
minikube addons enable registry

# make sure registry is available on 127.0.0.1
docker run --rm -it --network=host alpine ash -c "apk add socat && socat TCP-LISTEN:5000,reuseaddr,fork TCP:$(minikube ip):5000"
```

### Deploy argocd
```bash
alias k=kubectl

k create ns argocd
k apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# verify
k get all -n argocd
```

#### Start port-forwarding to access argocd
```bash
k port-forward svc/argocd-server -n argocd 8080:443
```

> Argocd will be available at https://localhost:8080

#### Fetch the admin password
```bash
k -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

#### Login using argocd's cli tool
```bash
argocd login localhost:8080
```


### Create a repo to store the argocd appConfigs
```bash
gh repo create --private miningcore-devel-configs
cd ..; gh repo clone miningcore-devel-configs; cd -;
```

## Build miningcore
```bash
docker build -t localhost:5000/miningcore:devel

# push the image to the local registry
docker push localhost:5000/miningcore:devel
```
