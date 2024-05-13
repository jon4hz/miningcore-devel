# Nexa

## Build docker image
Use this [`Dockerfile`](./Dockerfile)
```bash

NEXA_SHORT_VERSION=1.4.0.1
NEXA_VERSION=1.4.0.1

docker build -t localhost:5000/nexa:$NEXA_SHORT_VERSION . \
    --build-arg=NEXA_SHORT_VERSION=$NEXA_SHORT_VERSION \
    --build-arg=NEXA_VERSION=$NEXA_VERSION
    
docker push localhost:5000/nexa:$NEXA_SHORT_VERSION
```

## Deploy nexad
1. Create the argocd application `configs/nexa.yaml`:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: nexa
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: git@github.com:jon4hz/miningcore-devel-configs
       path: ./src/nexa/
       targetRevision: HEAD
     destination:
       server: https://kubernetes.default.svc
       namespace: miningcore
     syncPolicy:
       automated:
         prune: true
       syncOptions:
         - CreateNamespace=true
   ```
> [!IMPORTANT]  
> Make sure to adjust the `spec.source.repoURL` accordingly.

2. Create `src/nexa/manifest.yaml`:  
   Unfortunately we dont have a helm chart for nexa. So we'll use raw kubernetes resources instead.
   <details>
   <summary><code>manifest.yaml</code></summary>

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: nexa
     namespace: miningcore
   data:
     nexa.conf: |
       printtoconsole=1
       testnet=1
       server=1
       rpcuser=test
       rpcpassword=test
       rpcallowip=0.0.0.0/0
       prune=1000
       rpcbind=0.0.0.0
   ---
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: nexa
     namespace: miningcore
   spec:
     selector:
       matchLabels:
         app: nexa
     serviceName: nexa
     replicas: 1
     template:
       metadata:
         labels:
           app: nexa
       spec:
         containers:
         - name: nexa
           image: localhost:5000/nexa:1.4.0.1
           ports:
           - containerPort: 7229
             name: rpc
           volumeMounts:
           - name: nexadata
             mountPath: /nexa/.nexa/
           - name: nexaconfig
             mountPath: /nexa/.nexa/nexa.conf
             subPath: nexa.conf
         volumes:
           - name: nexaconfig
             configMap:
               name: nexa
     volumeClaimTemplates:
     - metadata:
         name: nexadata
         namespace: miningcore
       spec:
         accessModes: [ "ReadWriteOnce" ]
         resources:
           requests:
             storage: 20Gi
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: nexa
     namespace: miningcore
   spec:
     selector:
       app: nexa
     ports:
     - name: rpc
       port: 7229
       protocol: TCP
       targetPort: rpc
   ```
   </details>

  After deploying the nexa node, we'll have to wait a while until the full blockchain is synced.

## Create the pool wallet
```bash
# once for the pool wallet
k exec -it services/nexa -- nexa-cli getnewaddress

# and once for the fee wallet
k exec -it services/nexa -- nexa-cli getnewaddress "fee_wallet"
```

## Create database partition
```bash
# get the db password
k get secrets postgresql -o yaml | yq -r '.data.password' | base64 -d; echo

k exec -it postgresql-0 -- psql -h localhost -U miningcore --password -p 5432 -d miningcore -c "CREATE TABLE shares_nexa1 PARTITION OF shares FOR VALUES IN ('nexa1');"
```

## Deploy miningcore pool
```bash
```