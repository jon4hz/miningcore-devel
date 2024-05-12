# Installation

- [Installation](#installation)
  - [Install dependencies](#install-dependencies)
  - [Start minikube](#start-minikube)
    - [Enable the registry plugin](#enable-the-registry-plugin)
  - [Deploy argocd](#deploy-argocd)
    - [Start port-forwarding to access argocd](#start-port-forwarding-to-access-argocd)
    - [Fetch the admin password](#fetch-the-admin-password)
    - [Login using argocd's cli tool](#login-using-argocds-cli-tool)
    - [Setup appConfigs](#setup-appconfigs)
      - [Create "master" application](#create-master-application)
  - [Deploy sealed-secrets application](#deploy-sealed-secrets-application)
  - [Next steps](#next-steps)


## Install dependencies
```bash
yay -S minikube kubectl kubectx kubeseal k9s argocd github-cli jq
```
> [!NOTE]
> This is guide is written for arch linux (btw). Adjust the dependency installation according to your distro.

## Start minikube
```bash
minikube start --driver=docker
```

### Enable the registry plugin
```bash
minikube addons enable registry

# make sure registry is available on 127.0.0.1
docker run --rm -it --network=host alpine ash -c "apk add socat && socat TCP-LISTEN:5000,reuseaddr,fork TCP:$(minikube ip):5000"
```

## Deploy argocd
```bash
alias k=kubectl

k create ns argocd
k apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# verify
k get all -n argocd
```

### Start port-forwarding to access argocd
```bash
k port-forward svc/argocd-server -n argocd 8080:443
```
> Argocd will be available at https://localhost:8080

### Fetch the admin password
```bash
k -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

### Login using argocd's cli tool
```bash
argocd login localhost:8080
```

### Setup appConfigs
Create a repo to store the appConfigs:
```bash
gh repo create --private miningcore-devel-configs
cd ..; gh repo clone miningcore-devel-configs; cd -;
```

#### Create "master" application
We'll use an argocd application to deploy all the other appsets.  

1. Switch to the miningcore-devel-configs repo:
    ```bash
    cd ../miningcore-devel-configs/
    ```

2. Create `application.yaml`:
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: "all-configs"
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: git@github.com:jon4hz/miningcore-devel-configs
        targetRevision: HEAD
        path: ./configs
      destination:
        server: https://kubernetes.default.svc
        namespace: argocd
      syncPolicy:
        automated:
          prune: true
    ```
    > [!IMPORTANT]  
    > Make sure to adjust the `spec.source.repoURL` accordingly.

3. Push the changes:
    ```bash
    git add .
    git commit -m "initial commit"
    git push
    ```

4. Add repo to argocd
    Create a new deploy key for that repo and add it to argocd:
    ```bash
    # create a new ssh key
    ssh-keygen -t ed25519 -f ~/.ssh/miningcore-devel-configs -N ""

    # store your github username
    gh_user=$(gh api 'https://api.github.com/user' | jq -r .login)

    # upload the deployment key to github
    gh repo -R $gh_user/miningcore-devel-configs deploy-key add ~/.ssh/miningcore-devel-configs.pub

    # add the repo to argocd
    argocd repo add ssh://git@github.com/$gh_user/miningcore-devel-configs --ssh-private-key-path ~/.ssh/miningcore-devel-configs
    ```

5. Deploy the application on k8s
   ```
   k apply -f application.yaml
   ```

## Deploy sealed-secrets application
1. Make sure the `configs` folder exists:
   ```bash
   mkdir configs
   ```

2. Create the sealed-secrets application `./configs/sealed-secrets.yaml`:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
    name: sealed-secrets-controller
    namespace: argocd
    spec:
    project: default
    source:
      helm:
        releaseName: sealed-secrets-controller
      repoURL: https://bitnami-labs.github.io/sealed-secrets
      chart: sealed-secrets
      targetRevision: "*"
    destination:
      server: https://kubernetes.default.svc
      namespace: kube-system
    syncPolicy:
      automated:
        prune: true
      syncOptions:
        - CreateNamespace=true
   ```

3. Create application to deploy secrets `./config/secrets.yaml`:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
   name: secretes
     namespace: argocd
   spec:
     project: default
     source:
       path: secrets/
       repoURL: git@github.com:jon4hz/miningcore-devel-configs
       targetRevision: HEAD
       directory:
       recurse: true
     destination:
       server: https://kubernetes.default.svc
     syncPolicy:
       automated:
         prune: true
       syncOptions:
         - CreateNamespace=true
   ```
    > [!IMPORTANT]  
    > Make sure to adjust the `spec.source.repoURL` accordingly.

5. Create secrets directory:
   ```bash
   mkdir -p secrets/raw
   touch secrets/raw/.gitkeep
   cat <<EOF >> .gitignore
   secrets/raw/*
   !secrets/raw/.gitkeep
   EOF
   ```

6. Commit changes:
    ```bash
    git add .
    git commit -m "deploy sealed-secrets"
    git push
    ```
    Our master application should now automatically deploy the sealed-secrets app and any encrypted secrets stored in the `./secrets` directory.

## Next steps
1. [Setup postgresql database](./postgresql.md)