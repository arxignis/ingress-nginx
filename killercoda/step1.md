# Step 1: Setup Repository and Base Deployment

## Clone and Setup Repository

```bash
git clone https://github.com/arxignis/ingress-nginx.git
cd ingress-nginx
git checkout v1.0.3
git branch --show-current
```

## Create Namespace and TLS Secret

```bash
kubectl create ns ingress-nginx

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/tls.key -out /tmp/tls.crt \
  -subj "/CN=ingress-nginx-controller-admission"

kubectl -n ingress-nginx create secret tls ingress-nginx-admission \
  --key=/tmp/tls.key --cert=/tmp/tls.crt
```

## Deploy Base Ingress-NGINX

```bash
kubectl apply -f deploy/static/provider/baremetal/deploy.yaml
kubectl -n ingress-nginx get deploy,svc,pods
```