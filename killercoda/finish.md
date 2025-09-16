# Finished

You deployed the custom ingress-nginx controller image and enabled Arxignis without Helm, then validated routing and configuration.

What you accomplished:
- Deployed static manifests and provided a webhook cert secret
- Switched controller image to `ghcr.io/arxignis/ingress-nginx:v1.0.1`
- Enabled Arxignis via ConfigMap and environment variables
- Tested routing with a demo app

Next steps:
- Toggle `ARXIGNIS_MODE` between monitoring/blocking and observe behavior
- Integrate with real API credentials
- Optionally add the admission webhook certgen jobs once image pulls are available
