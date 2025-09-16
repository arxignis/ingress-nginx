# Step 1: Setup and deploy controller

1) Namespace and webhook secret

```bash
kubectl create ns ingress-nginx
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/tls.key -out /tmp/tls.crt \
  -subj "/CN=ingress-nginx-controller-admission"
kubectl -n ingress-nginx create secret generic ingress-nginx-admission \
  --from-file=key=/tmp/tls.key --from-file=cert=/tmp/tls.crt
```

2) Apply static manifests (bare-metal flavor)

```bash
kubectl apply -f deploy/static/provider/baremetal/deploy.yaml
```

3) Point controller to your image and enable Arxignis

```bash
kubectl -n ingress-nginx set image deploy/ingress-nginx-controller \
  controller=ghcr.io/arxignis/ingress-nginx:v1.0.1

kubectl -n ingress-nginx set env deploy/ingress-nginx-controller \
  ARXIGNIS_API_KEY="test-api-key" \
  ARXIGNIS_API_URL="https://api.arxignis.com" \
  ARXIGNIS_CAPTCHA_PROVIDER="recaptcha" \
  ARXIGNIS_CAPTCHA_SITE_KEY="test-site-key" \
  ARXIGNIS_CAPTCHA_SECRET_KEY="test-secret-key" \
  ARXIGNIS_MODE="monitoring"

kubectl -n ingress-nginx patch configmap ingress-nginx-controller \
  --type merge -p '{"data":{"enable-arxignis":"true"}}'

kubectl -n ingress-nginx rollout restart deploy/ingress-nginx-controller
kubectl -n ingress-nginx rollout status deploy/ingress-nginx-controller --timeout=180s
```

Troubleshooting: If admission Jobs are Pending/ImagePullBackOff, ignore or delete them; the pre-created secret is sufficient for the controller to start.

```bash
kubectl -n ingress-nginx delete job ingress-nginx-admission-create ingress-nginx-admission-patch --ignore-not-found
```
