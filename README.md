


### Install nginx ingress

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  -f ingress/nginx-ingress-values.yaml
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

# Outdated things

# Odd things

When initializing a new database, the pg-pool section needs to be commented out first, then it can be upgraded with it included.

\* Seems this was fixed thougn \*

### Getting credentials

```bash
export POSTGRES_PASSWORD=$(kubectl get secret --namespace default pg-ha-postgresql-ha-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode)

export REPMGR_PASSWORD=$(kubectl get secret --namespace default pg-ha-postgresql-ha-postgresql -o jsonpath="{.data.repmgr-password}" | base64 --decode)

export ADMIN_PASSWORD=$(kubectl get secret --namespace "default" pg-ha-postgresql-ha-pgpool -o jsonpath="{.data.admin-password}" | base64 --decode)
```

### Install db

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install -f postgres-values.yaml pg-ha bitnami/postgresql-ha
```

### Upgrading db

```bash
helm upgrade -f postgres-values.yaml pg-ha bitnami/postgresql-ha --set postgresql.repmgrPassword=$REPMGR_PASSWORD --set pgpool.adminPassword=$ADMIN_PASSWORD --set postgresql.password=$POSTGRES_PASSWORD
```

To restart db instances, set replicas to 1 and do

```bash
helm upgrade -f postgres-values.yaml pg-ha bitnami/postgresql-ha --set postgresql.repmgrPassword=$REPMGR_PASSWORD --set pgpool.adminPassword=$ADMIN_PASSWORD --set postgresql.password=$POSTGRES_PASSWORD --set postgresql.upgradeRepmgrExtension=true
```