# drm-exporter

Prometheus exporter for Intel/AMD GPU metrics, deployed as a per-node DaemonSet.

Upstream: https://github.com/home-operations/drm-exporter

Pinned to `nas-worker` (the node with the Intel iGPU, same one Plex uses for
hardware transcoding). The bundled ServiceMonitor and Grafana dashboard are
enabled; Prometheus scrapes ServiceMonitors cluster-wide and the Grafana
sidecar imports dashboards from any namespace.

## Getting Intel iGPU engine metrics working (the fiddly part)

The exporter reads engine utilization from the **i915 perf PMU** via
`perf_event_open(2)`. On a default Talos node it crashes at startup with:

```
Error: initializing the GPU collector
Caused by:
    0: discovering DRM devices
    1: Operation not permitted (os error 1)
```

Three things are required, all verified empirically against this node's iGPU and
all configured in `values.yaml` — no node-level change is needed.

1. **`privileged: true`** on the container. The i915 perf PMU open needs the
   `card0` device node, gated by the container **device cgroup**, which
   Kubernetes only opens with `privileged: true`. Non-privileged fails at
   discovery no matter which capabilities are added — `PERFMON`/`SYS_ADMIN`
   alone are not enough. (This is how Plex/frigate reach the same iGPU.)

2. **`CAP_PERFMON` + `CAP_SYS_ADMIN`** on the container. Needed for the
   system-wide i915 perf stream. With these present and the container
   privileged, `CAP_PERFMON` also bypasses `kernel.perf_event_paranoid`, so the
   Talos default of `3` is fine — no sysctl patch required.

3. **`seccompProfile: Unconfined`** on the pod. The default `RuntimeDefault`
   profile blocks the `perf_event_open` syscall outright (EPERM) regardless of
   capabilities. (The namespace is labelled PSA `privileged`, so Unconfined is
   permitted — see `templates/namespace.yaml`.)

## Package temperature (`msr`) is intentionally disabled

Intel iGPU package-temperature readings need the host `msr` kernel module, which
Talos does not ship and treats as a userspace-MSR security risk. Per upstream
guidance we drop the `/dev/cpu` mount and `SYS_RAWIO` capability (see
`values.yaml`); all other metrics work via the perf PMU. Temperature is simply
unavailable on this cluster.
