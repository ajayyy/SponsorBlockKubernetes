


### Install nginx ingress

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  -f ingress/nginx-ingress-values.yaml
```

### Setup postgres


```bash
kubectl apply -f kubernetes-config/postgres/operator.yaml

# Then wait a little bit for it to startup and install the CRDs

kubectl apply -f kubernetes-config/postgres
```

### Then use kustomize

Untested

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