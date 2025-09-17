# Step 2: Configure Arxignis Integration

## Update to Arxignis-Enabled Image

```bash
kubectl -n ingress-nginx set image deploy/ingress-nginx-controller \
  controller=ghcr.io/arxignis/ingress-nginx:v1.0.3

kubectl -n ingress-nginx describe deploy ingress-nginx-controller | grep Image:
```

## Enable Arxignis with Real API Key(REPACE PLACEHOLDER WITH REAL API KEY)

```bash
kubectl -n ingress-nginx patch configmap ingress-nginx-controller \
  --type merge -p '{"data":{"enable-arxignis":"true"}}'

kubectl -n ingress-nginx set env deploy/ingress-nginx-controller \
  ARXIGNIS_API_URL="https://api.arxignis.com" \
  ARXIGNIS_API_KEY="API KEY HERE" \
  ARXIGNIS_MODE="monitor" \
  ARXIGNIS_CAPTCHA_PROVIDER="recaptcha"
```

## Configure Ports for Sandbox

```bash
cat > /tmp/fix-ports.json << 'EOF'
[{
  "op": "replace",
  "path": "/spec/template/spec/containers/0/args",
  "value": [
    "/nginx-ingress-controller",
    "--election-id=ingress-nginx-leader",
    "--controller-class=k8s.io/ingress-nginx",
    "--ingress-class=nginx",
    "--configmap=$(POD_NAMESPACE)/ingress-nginx-controller",
    "--validating-webhook=:8443",
    "--validating-webhook-certificate=/usr/local/certificates/cert",
    "--validating-webhook-key=/usr/local/certificates/key",
    "--http-port=8080",
    "--https-port=8444"
  ]
}]
EOF

cat > /tmp/fix-container-ports.json << 'EOF'
[
  {"op": "replace", "path": "/spec/template/spec/containers/0/ports/0/containerPort", "value": 8080},
  {"op": "replace", "path": "/spec/template/spec/containers/0/ports/1/containerPort", "value": 8444}
]
EOF

kubectl -n ingress-nginx patch deploy ingress-nginx-controller \
  --type='json' --patch-file /tmp/fix-ports.json

kubectl -n ingress-nginx patch deploy ingress-nginx-controller \
  --type='json' --patch-file /tmp/fix-container-ports.json
```

## Restart and Wait for Deployment

```bash
kubectl -n ingress-nginx rollout restart deploy/ingress-nginx-controller
kubectl -n ingress-nginx rollout status deploy/ingress-nginx-controller --timeout=180s
kubectl -n ingress-nginx get pods -l app.kubernetes.io/component=controller
```


## Verify Arxignis Configuration

```bash
kubectl -n ingress-nginx get configmap ingress-nginx-controller -o yaml | grep arxignis
kubectl -n ingress-nginx get deploy ingress-nginx-controller -o yaml | grep -A5 -B5 ARXIGNIS
kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller | grep -i arxignis
```
