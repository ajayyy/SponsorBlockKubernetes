### Create cluster

```bash
cd terraform
terraform apply
```

Destroy:

```bash
cd terraform
terraform destroy
```


### Install nginx ingress

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  -f kubernetes-config/ingress/nginx-ingress-values.yaml
```

### Install redis

Docker compose file

```yaml
version: '3.8'
services:
  dragonfly:
    container_name: dragonfly
    image: 'docker.dragonflydb.io/dragonflydb/dragonfly'
    ulimits:
      memlock: -1
    #ports:
    #  - "6379:6379"
    # For better performance, consider `host` mode instead `port` to avoid docker NAT.
    # `host` mode is NOT currently supported in Swarm Mode.
    # https://docs.docker.com/compose/compose-file/compose-file-v3/#network_mode
    network_mode: "host"
    entrypoint: dragonfly --cache_mode=true --port=32773 --snapshot_cron "* * * * *"
    restart: always
    volumes:
      - ./dragonflydata:/data
```

To setup regulation, connect use:

```bash
dragonfly --cache_mode=true --port=32773 --replicaof=10.2.0.2:32773
```

### Setup postgres

#### Ensure diff plugin is installed

Needed for `helmfile`

```bash
helm plugin install https://github.com/databus23/helm-diff
```

#### Install Grafana

```bash
helmfile -f kubernetes-config/grafana/helmfile.yaml apply
```

#### Install Operator

```bash
helmfile -f kubernetes-config/stackgres/production/operator/helmfile.yaml apply
```

#### Make sure secrets have been applied

```bash
kubectl apply -f kubernetes-config/secrets
```

#### Create all the configurations

```bash
kubectl apply -k kubernetes-config/stackgres/production/configurations
```


#### Create SGCluster

```bash
kubectl apply -k kubernetes-config/stackgres/production/cluster
```

### Then use kustomize

Might have to run a second time to make sure cert works (will error if it fails)

```bash
kubectl apply -k kubernetes-config
```

# Postgres

```bash
kubectl apply -k kubernetes-config/kustomize/install/namespace
kubectl apply --server-side -k kubernetes-config/kustomize/install/default
```

To check on the status of your installation, you can run the following command:

```bash
kubectl -n postgres-operator get pods \
  --selector=postgres-operator.crunchydata.com/control-plane=postgres-operator \
  --field-selector=status.phase=Running
```

Then install postgres itself

```bash
kubectl apply -k kubernetes-config/kustomize/postgres
```


To check on the status of your installation, you can run the following command:

```bash
kubectl describe postgresclusters.postgres-operator.crunchydata.com pgsb
```

Then fix the affinity by running

```bash
kubectl edit deployments.apps cluster1-repl<number>
```

And adding the following to `spec.affinity`

```yaml
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: largerdb
                operator: In
                values:
                - "true"
```

And adding the following to `spec.affinity.podAntiAffinity` for only the repl ones

```yaml
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                - key: deployment-name
                  operator: In
                  values:
                  - cluster3
              topologyKey: "kubernetes.io/hostname"
```

# Install other configs

```bash
terraform apply -auto-approve

kubectl apply -f kubernetes-config/secrets
kubectl apply -f kubernetes-config/certs
kubectl apply -f kubernetes-config/*
```

## For setup from scratch

### Set default kustomize

```bash
KUBECONFIG=<full file path to kubeconfig.yaml>
```

### Prep Cert Manager

Pre-setup (already done)
```bash
cmctl x install --dry-run > cert-manager.custom.yaml
```

The cert manager yaml has already been created and is in kubernetes-config/certs

## Restoring DB from backup

Create a new cluster yml with

```yaml
initialData:
    restore:
      fromBackup:
        uid: d7e660a9-377c-11ea-b04b-0242ac110004
      downloadDiskConcurrency: 1
```

Also check runbooks https://stackgres.io/doc/latest/runbooks/

# Other useful things

#### Notes

If the main switches to a replica, there might be an issue due to replica service and pgbouncer now pointing to the same pod, removing any load balancing. A solution would be to delete the replica pod to cause it to failover to the main one.

#### Changing config

```bash
kubectl edit configmap cluster1-custom-config
```

setting `pause: true` and then `kubectl apply -f kubernetes-config/postgres/cr.yaml` is required after many config changes to get it to update. `pause: false` once they shutdown

### Queries

#### Get active queries

```sql
SELECT pid, age(clock_timestamp(), query_start), usename, query 
FROM pg_stat_activity 
WHERE query != '<IDLE>' AND query NOT ILIKE '%pg_stat_activity%' AND state != 'idle'
ORDER BY query_start asc;
```

#### Get blocked queries

```sql
SELECT blocked_locks.pid     AS blocked_pid,
         blocked_activity.usename  AS blocked_user,
         blocking_locks.pid     AS blocking_pid,
         blocking_activity.usename AS blocking_user,
         blocked_activity.query    AS blocked_statement,
         blocking_activity.query   AS current_statement_in_blocking_process
   FROM  pg_catalog.pg_locks         blocked_locks
    JOIN pg_catalog.pg_stat_activity blocked_activity  ON blocked_activity.pid = blocked_locks.pid
    JOIN pg_catalog.pg_locks         blocking_locks 
        ON blocking_locks.locktype = blocked_locks.locktype
        AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
        AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
        AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
        AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
        AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
        AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
        AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
        AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
        AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
        AND blocking_locks.pid != blocked_locks.pid

    JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
   WHERE NOT blocked_locks.granted;
```

# Old stuff

### Setup old postgres


```bash
kubectl apply -f kubernetes-config/secrets/cluster1-backrest-repo-config-secret.yaml
kubectl apply -f kubernetes-config/postgres/operator.yaml

# Then wait a little bit for it to startup and install the CRDs

kubectl apply -f kubernetes-config/postgres
```

Get password

```bash
kubectl get secret cluster1-pguser-secret -o go-template='{{.data.password | base64decode}}'
```

### Restoring database from backup

See https://www.percona.com/doc/kubernetes-operator-for-postgresql/standby.html

- Make new cluster with standby and old repoPath
- Copy over the secrets (script there doesn't work, just edit the file manually)
- Make sure s3 secrets are there for that cluster

Deleting clusters can be done with:

```bash
kubectl delete perconapgclusters.pg.percona.com clustername
```

When scaling down, make sure everything is still working before deleting volumes

### Recovering from wal failure causing out of date replicas

Check to see if backrest is out of space. If so, raise it in cr.yml and it will auto resize.

Then, follow the steps below to fix wal being out of date

#### If only one is failing

Raise storage size in backup pvc

Kill that one and delete the pvc, then recreate the pvc.

Then wait for the automated full backup to finish or use the method below

Then reinit that node

Then delete the old backups to save space

Then lower the storage claim? Is the only way to lower it making a new cluster based on the old one?

#### Option 1

start a full backup with the repo (see sample-backup.yaml)

Delete replica pods and pvc and restart them or use `patronictl reinit clusterX` and choose the specific pod.

#### Option 2

Untested

https://dba.stackexchange.com/questions/242722/archive-command-repeatly-failing-for-one-particular-file


```bash
kubectl edit configmap cluster1-pgha-config
```

Set archive_command = /bin/true

Delete replica pods and pvc and restart them or use `patronictl reinit clusterX` and choose the specific pod.

Restore archive_command to default

```
source /opt/crunchy/bin/postgres-ha/pgbackrest/pgbackrest-archive-push-local-s3.sh %p
```

Make sure to trigger a full backup after instead of incremental

then need to start a full backup with the repo (see sample-backup.yaml)