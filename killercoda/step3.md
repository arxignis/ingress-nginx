# Step 3: Validate Arxignis and traffic

1) Confirm controller image and readiness
```bash
kubectl -n ingress-nginx get deploy ingress-nginx-controller -o=jsonpath='{.spec.template.spec.containers[0].image}'; echo
kubectl -n ingress-nginx get pods -l app.kubernetes.io/component=controller
```

2) Check Arxignis configuration
```bash
kubectl -n ingress-nginx get cm ingress-nginx-controller -o yaml | grep -n 'enable-arxignis'
kubectl -n ingress-nginx get deploy ingress-nginx-controller -o yaml | grep -n 'ARXIGNIS_'
```

3) Validate NGINX has Arxignis modules
```bash
kubectl -n ingress-nginx exec deploy/ingress-nginx-controller -- nginx -T | grep -n 'arxignis_'
```

4) Generate traffic
```bash
curl -I -H "Host: demo.local" http://127.0.0.1:8080/
curl -I -H "Host: demo.local" -H "User-Agent: suspicious-bot/1.0" http://127.0.0.1:8080/
```

5) Inspect logs (optional)
```bash
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller | grep -i arxignis || true
```

Cleanup (optional)
```bash
kubectl delete ns demo
kubectl delete ns ingress-nginx
```
