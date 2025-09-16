# Ingress NGINX + Arxignis (No Helm)

In this scenario you'll:

- Deploy the custom controller image published at `ghcr.io/arxignis/ingress-nginx:v1.0.1` without Helm
- Enable Arxignis via ConfigMap flag and environment variables
- Deploy a demo app and route traffic through the controller

You will execute simple `kubectl` commands, use the static manifests from this repo, and validate functionality.


