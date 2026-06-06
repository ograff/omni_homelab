# tailscale-exporter

Prometheus exporter for tailnet-level Tailscale metrics
([adinhodovic/tailscale-exporter](https://github.com/adinhodovic/tailscale-exporter)),
plus the Grafana dashboards and Prometheus alerts from its `tailscale-mixin`.

What this app deploys:

- The exporter (`tailscale-exporter` upstream Helm chart) + a `ServiceMonitor`
  (auto-scraped by the kube-prometheus-stack Prometheus in `apps/monitoring`).
- A `PrometheusRule` with the tailnet + tailscaled-machine alerts
  (`templates/prometheusrule.yaml`).
- The mixin dashboards as `grafana_dashboard`-labelled ConfigMaps
  (`dashboards/*.json` + `templates/dashboards.yaml`), auto-imported by the
  Grafana sidecar in `apps/grafana` into a "Tailscale" folder.

## Required secret (create out-of-band)

The exporter reads its credentials from a Secret named `tailscale-exporter` in
this namespace. Create a Tailscale OAuth client at
<https://login.tailscale.com/admin/settings/oauth> with these read scopes:
`devices:core:read`, `devices:posture_attributes:read`, `devices:routes:read`,
`users:read`, `dns:read`, `auth_keys:read`, `feature_settings:read`,
`policy_file:read`. Then:

```sh
kubectl -n tailscale-exporter create secret generic tailscale-exporter \
  --from-literal=TAILSCALE_OAUTH_CLIENT_ID='<client-id>' \
  --from-literal=TAILSCALE_OAUTH_CLIENT_SECRET='<client-secret>' \
  --from-literal=TAILSCALE_TAILNET='<tailnet>'   # e.g. example.com or your-tailnet.ts.net
```

(The namespace is created by ArgoCD on first sync; create the secret after.)

## Updating the mixin (dashboards + alerts)

These are vendored, so Renovate won't bump them. To refresh against upstream:

```sh
cd apps/tailscale-exporter
for d in tailscale-overview tailscale-machine; do
  curl -fsSL "https://raw.githubusercontent.com/adinhodovic/tailscale-exporter/main/tailscale-mixin/dashboards_out/$d.json" -o "dashboards/$d.json"
done
# then reconcile templates/prometheusrule.yaml against:
# https://github.com/adinhodovic/tailscale-exporter/blob/main/tailscale-mixin/prometheus_alerts.yaml
```
