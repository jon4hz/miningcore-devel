# Postgresql

To deploy the database we will install bitnami's postgresql helm chart via argocd.

> [!IMPORTANT]
> This guide assumes that the miningcore-devel-configs repo is the working directory.

## Create secret for credentials
```bash
# make sure miningcore namespace exists
k create ns miningcore

# create the unencrypted secret
k create -n miningcore -o yaml --dry-run=client secret generic postgresql \
    --from-literal=postgres-password=$(openssl rand -base64 32) \
    --from-literal=password=$(openssl rand -base64 32) > secrets/raw/postgresql.yaml

# seal it
kubeseal --secret-file secrets/raw/postgresql.yaml --sealed-secret-file secrets/postgresql.yaml

# push it
git add .
git commit -m "add postgresql secret"
git push
```

## Deploy postgresql

First of all we create an argocd app which will handle the deployment of the helm chart for us.

1. Create `configs/postgresql.yaml`:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
   name: postgresql
   namespace: argocd
   spec:
   project: default
   source:
     helm:
        releaseName: postgresql
        valueFiles:
          - values.yml
     repoURL: git@github.com:jon4hz/miningcore-devel-configs
     path: ./src/postgresql/
     targetRevision: "HEAD"
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

1. Prepare your helm configuration:
   ```bash
   mkdir -p src/postgresql
   ```

2. Create `src/postgresql/Chart.yaml`:
   ```yaml
   apiVersion: v2
   name: postgresql
   description: postgresql
   type: application
   version: 1.0.0
   appVersion: "1.0.0"

   dependencies:
     - name: postgresql
       version: '^15'
       repository: https://charts.bitnami.com/bitnami
   ```

3. Create `src/postgresql/values.yml`:
   The `values.yml` contains your postgresql configuration and an sql script to initialize the miningcore database.
   <details>
   <summary><code>src/postgresql/values.yml</code></summary>

   ```yaml
   postgresql:
     enabled: true
     auth:
       database: miningcore
       username: miningcore
       existingSecret: postgresql
     serviceAccount:
       automountServiceAccountToken: false
     architecture: standalone
     primary:
       persistence:
         enabled: true
         storageClass: standard
         size: 5Gi
       initdb:
         scripts:
           init_miningcore.sql: |
             CREATE DATABASE miningcore OWNER miningcore;
             --
             -- create_tsdb.sql
             --
             SET ROLE miningcore;

             CREATE TABLE shares
             (
               poolid TEXT NOT NULL,
               blockheight BIGINT NOT NULL,
               difficulty DOUBLE PRECISION NOT NULL,
               networkdifficulty DOUBLE PRECISION NOT NULL,
               miner TEXT NOT NULL,
               worker TEXT NULL,
               useragent TEXT NULL,
               ipaddress TEXT NOT NULL,
                 source TEXT NULL,
               created TIMESTAMPTZ NOT NULL
             );

             CREATE INDEX IDX_SHARES_POOL_MINER on shares(poolid, miner);
             CREATE INDEX IDX_SHARES_POOL_CREATED ON shares(poolid, created);
             CREATE INDEX IDX_SHARES_POOL_MINER_DIFFICULTY on shares(poolid, miner, difficulty);

             CREATE TABLE blocks
             (
               id BIGSERIAL NOT NULL PRIMARY KEY,
               poolid TEXT NOT NULL,
               blockheight BIGINT NOT NULL,
               networkdifficulty DOUBLE PRECISION NOT NULL,
               status TEXT NOT NULL,
                 type TEXT NULL,
                 confirmationprogress FLOAT NOT NULL DEFAULT 0,
               effort FLOAT NULL,
               transactionconfirmationdata TEXT NOT NULL,
               miner TEXT NULL,
               reward decimal(28,12) NULL,
                 source TEXT NULL,
                 hash TEXT NULL,
               created TIMESTAMPTZ NOT NULL,

                 CONSTRAINT BLOCKS_POOL_HEIGHT UNIQUE (poolid, blockheight, type) DEFERRABLE INITIALLY DEFERRED
             );

             CREATE INDEX IDX_BLOCKS_POOL_BLOCK_STATUS on blocks(poolid, blockheight, status);

             CREATE TABLE balances
             (
               poolid TEXT NOT NULL,
               address TEXT NOT NULL,
               amount decimal(28,12) NOT NULL DEFAULT 0,
               created TIMESTAMPTZ NOT NULL,
               updated TIMESTAMPTZ NOT NULL,

               primary key(poolid, address)
             );

             CREATE TABLE balance_changes
             (
               id BIGSERIAL NOT NULL PRIMARY KEY,
               poolid TEXT NOT NULL,
               address TEXT NOT NULL,
               amount decimal(28,12) NOT NULL DEFAULT 0,
               usage TEXT NULL,
                 tags text[] NULL,
               created TIMESTAMPTZ NOT NULL
             );

             CREATE INDEX IDX_BALANCE_CHANGES_POOL_ADDRESS_CREATED on balance_changes(poolid, address, created desc);
             CREATE INDEX IDX_BALANCE_CHANGES_POOL_TAGS on balance_changes USING gin (tags);

             CREATE TABLE miner_settings
             (
               poolid TEXT NOT NULL,
               address TEXT NOT NULL,
               paymentthreshold decimal(28,12) NOT NULL,
               created TIMESTAMPTZ NOT NULL,
               updated TIMESTAMPTZ NOT NULL,

               primary key(poolid, address)
             );

             CREATE TABLE payments
             (
               id BIGSERIAL NOT NULL PRIMARY KEY,
               poolid TEXT NOT NULL,
               coin TEXT NOT NULL,
               address TEXT NOT NULL,
               amount decimal(28,12) NOT NULL,
               transactionconfirmationdata TEXT NOT NULL,
               created TIMESTAMPTZ NOT NULL
             );

             CREATE INDEX IDX_PAYMENTS_POOL_COIN_WALLET on payments(poolid, coin, address);

             CREATE TABLE poolstats
             (
               id BIGSERIAL NOT NULL PRIMARY KEY,
               poolid TEXT NOT NULL,
               connectedminers INT NOT NULL DEFAULT 0,
               poolhashrate DOUBLE PRECISION NOT NULL DEFAULT 0,
               sharespersecond DOUBLE PRECISION NOT NULL DEFAULT 0,
               networkhashrate DOUBLE PRECISION NOT NULL DEFAULT 0,
               networkdifficulty DOUBLE PRECISION NOT NULL DEFAULT 0,
               lastnetworkblocktime TIMESTAMPTZ NULL,
                 blockheight BIGINT NOT NULL DEFAULT 0,
                 connectedpeers INT NOT NULL DEFAULT 0,
               created TIMESTAMPTZ NOT NULL
             );

             CREATE INDEX IDX_POOLSTATS_POOL_CREATED on poolstats(poolid, created);

             CREATE TABLE minerstats
             (
               id BIGSERIAL NOT NULL PRIMARY KEY,
               poolid TEXT NOT NULL,
               miner TEXT NOT NULL,
               worker TEXT NOT NULL,
               hashrate DOUBLE PRECISION NOT NULL DEFAULT 0,
               sharespersecond DOUBLE PRECISION NOT NULL DEFAULT 0,
               created TIMESTAMPTZ NOT NULL
             );

             CREATE INDEX IDX_MINERSTATS_POOL_CREATED on minerstats(poolid, created);
             CREATE INDEX IDX_MINERSTATS_POOL_MINER_CREATED on minerstats(poolid, miner, created);
             CREATE INDEX IDX_MINERSTATS_POOL_MINER_WORKER_CREATED_HASHRATE on minerstats(poolid,miner,worker,created desc,hashrate);

             SELECT create_hypertable('shares','created');
             SELECT create_hypertable('blocks', 'created');
             SELECT create_hypertable('balances', 'created');
             SELECT set_chunk_time_interval('shares', INTERVAL '24 hours');

             --
             -- ext1_update_sql.sql
             --
             ALTER TABLE minerstats ADD COLUMN hashratetype VARCHAR;

             --
             -- ext2_connected_workers.sql
             --
             ALTER TABLE poolstats ADD COLUMN connectedworkers INT NOT NULL DEFAULT 0;

             --
             -- ext3_refactor_reported_hashrate.sql
             --
             -- create the new table
             CREATE TABLE reported_hashrate
             (
               id BIGSERIAL NOT NULL PRIMARY KEY,
               poolid TEXT NOT NULL,
               miner TEXT NOT NULL,
               worker TEXT NOT NULL,
               hashrate DOUBLE PRECISION NOT NULL DEFAULT 0,
               created TIMESTAMPTZ NOT NULL
             );

             -- create new indexes
             CREATE INDEX IDX_REPORTEDHASHRATE_POOL_MINER_CREATED on reported_hashrate(poolid, miner, created);

             -- timescale config for reported_hashrate
             SELECT create_hypertable('reported_hashrate', 'created');

             -- copy data from minerstats to reported_hashrate
             INSERT INTO reported_hashrate 
                 (poolid, miner, worker, hashrate, created)
             SELECT poolid, miner, worker, hashrate, created 
             FROM minerstats
             WHERE hashratetype = 'reported';


             -- delete data from minerstats
             DELETE FROM minerstats WHERE hashratetype = 'reported';


             -- remove hashratetype column from minerstats
             ALTER TABLE minerstats DROP COLUMN hashratetype;
   ```
   </details>

3. Commit changes
   ```bash
    git add .
    git commit -m "deploy sealed-secrets"
    git push
    ```
  