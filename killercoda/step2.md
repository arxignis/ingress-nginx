# Step 2: Deploy demo app and Ingress

1) Deploy echo app and service
```bash
kubectl create ns demo || true
cat <<'YAML' | kubectl -n demo apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: { name: echo }
spec:
  replicas: 1
  selector: { matchLabels: { app: echo } }
  template:
    metadata: { labels: { app: echo } }
    spec:
      containers:
        - name: http-echo
          image: hashicorp/http-echo:0.2.3
          args: ["-text=hello-from-echo"]
          ports: [{ containerPort: 5678 }]
---
apiVersion: v1
kind: Service
metadata: { name: echo }
spec:
  selector: { app: echo }
  ports:
    - name: http
      port: 80
      targetPort: 5678
YAML
```

2) Create an Ingress using ingressClassName nginx
```bash
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

3) Access via port-forward (works on any service type)
```bash
kubectl -n ingress-nginx port-forward svc/ingress-nginx-controller 8080:80 >/dev/null 2>&1 &
curl -i -H "Host: demo.local" http://127.0.0.1:8080/
```
