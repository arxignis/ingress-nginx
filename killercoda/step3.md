# Step 3: Deploy Test Application and Test Arxignis

## Deploy Echo Application

```bash
kubectl create ns demo

cat <<'YAML' | kubectl -n demo apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
        - name: http-echo
          image: hashicorp/http-echo:0.2.3
          args: ["-text=Hello from Arxignis-protected application!"]
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  selector:
    app: echo
  ports:
    - name: http
      port: 80
      targetPort: 5678
YAML
```

## Create Ingress Resource

```bash
kubectl delete validatingwebhookconfiguration ingress-nginx-admission --ignore-not-found

cat <<'YAML' | kubectl -n demo apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo
spec:
  ingressClassName: nginx
  rules:
    - host: demo.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: echo
                port:
                  number: 80
YAML
```

## Test Legitimate Traffic

```bash
kubectl -n ingress-nginx exec deploy/ingress-nginx-controller -- \
  curl -s -H "Host: demo.local" \
       -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
       http://127.0.0.1:8080/
```

Expected response:
```
Hello from Arxignis-protected application!
```

## Test Bot Traffic

```bash
kubectl -n ingress-nginx exec deploy/ingress-nginx-controller -- \
  curl -s -w "Status: %{http_code}\n" \
       -H "Host: demo.local" \
       -H "User-Agent: python-requests/2.28.0" \
       -H "X-Real-IP: 185.220.101.1" \
       http://127.0.0.1:8080/
```

Expected response (bot should be blocked):
```
{
  "success": false,
  "error": "Access denied by Arxignis",
  "reason": "API decision: block",
  "client_ip": "185.220.101.1"
}
Status: 403
```

## Monitor Arxignis Integration Status

```bash
kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=20
```

Look for Arxignis logs:
```
[lua] init_by_lua: Arxignis integration initialized successfully
[lua] init_worker_by_lua: Starting flush timers
[lua] access_by_lua: Arxignis: Analyzing request from 127.0.0.1 UA: python-requests/2.28.0
[lua] access_by_lua: Arxignis: API response status: 200
[lua] access_by_lua: Arxignis: Blocking request - API decision: block
```

The logs show active bot detection and blocking in real-time.

## Test Arxignis API Directly

```bash
kubectl -n ingress-nginx exec deploy/ingress-nginx-controller -- \
  curl -X POST -H "Authorization: Bearer /qrasi1m7pEdpZ/qW015agJMa4Jfb9czMpq5ThN7VnA=" \
       -H "Content-Type: application/json" \
       -d '{"timestamp":"2025-09-17T13:00:00Z","version":"1.0","clientIp":"192.168.1.100","hostName":"demo.local","http":{"method":"GET","url":"http://demo.local/","headers":{}},"tls":{"version":"TLSv1.3"}}' \
       "https://api.arxignis.com/v1/remediation/192.168.1.100"
```

Expected response:
```json
{"success":true,"remediation":{"ip":"192.168.1.100","action":"none","expired":60,"score":0,"ruleId":"none"}}
```