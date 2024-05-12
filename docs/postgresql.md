# Postgresql

To deploy the database we will install bitnami's postgresql helm chart via argocd.

> [!IMPORTANT]
> This guide assumes that the miningcore-devel-configs repo is the working directory.

## Create secret for credentials
```bash
# make sure miningcore namespace exists
k create ns miningcore

# create the unencrypted secret
k create -n miningcore -o yaml --dry-run=client secret generic postgres \
    --from-literal=username=miningcore \
    --from-literal=password=$(openssl rand -base64 32) > secrets/raw/postgresql.yaml

# seal it
kubeseal --secret-file secrets/raw/postgresql.yaml --sealed-secret-file secrets/postgres.yaml

# push it
git add .
git commit -m "add postgresql secret"
git push
```

## Create the app config
