# drm-exporter

Prometheus exporter for Intel/AMD GPU metrics, deployed as a per-node DaemonSet.

Upstream: https://github.com/home-operations/drm-exporter

Pinned to `nas-worker` (the node with the Intel iGPU, same one Plex uses for
hardware transcoding). The bundled ServiceMonitor and Grafana dashboard are
enabled; Prometheus scrapes ServiceMonitors cluster-wide and the Grafana
sidecar imports dashboards from any namespace.

## Intel iGPU package temperature requires the `msr` kernel module

Most distros autoload `msr`, but Talos does not. Without it, GPU/package
temperature metrics will be missing (the rest of the metrics still work).

Load it via a Talos machine config patch (`infra/patches/`):

```yaml
machine:
  kernel:
    modules:
      - name: msr
```

Apply with `talosctl patch mc` / re-apply the cluster config, then verify on the
node: `lsmod | grep msr`.
