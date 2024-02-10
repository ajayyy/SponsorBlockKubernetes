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

To setup replication, connect use:

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

#### Make sure secrets have been applied

```bash
kubectl apply -f kubernetes-config/secrets
```

### Then use kustomize

Might have to run a second time to make sure cert works (will error if it fails)

```bash
kubectl apply -k kubernetes-config
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

# Other useful things

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