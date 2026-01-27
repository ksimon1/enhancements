# VEP: Probe Proxy for Dynamic Probe Control

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue created, which links to VEP dir in [kubevirt/enhancements] (not the initial VEP PR)
- [ ] (R) Target version is explicitly mentioned and approved
- [ ] (R) Graduation criteria filled

## Overview

This proposal introduces a lightweight, ConfigMap-based mechanism to dynamically control GuestAgentPing probes at runtime. By utilizing VMI annotations, administrators can pause these probes during maintenance operations such as live migration, backup, or guest OS updates. virt-handler reconciles these annotations into a ConfigMap that is mounted within the virt-launcher pod. A background goroutine in virt-launcher periodically reads this configuration and instructs the existing virt-probe utility to return synthetic success responses when paused. This prevents probe failures from triggering unwanted Pod restarts without requiring a complex proxy server or intercepting network traffic.

## Motivation

Currently, KubeVirt probes are directly translated into Pod probes and executed by the Kubernetes kubelet against the VM or guest agent. This creates several challenges during standard administrative operations:

1. **Live Migration**: During live migration, the VM may be temporarily unresponsive. Probe failures during this window can cause Kubernetes to restart the Pod or mark the VM as not ready, disrupting the migration.

2. **Backup Operations**: When taking VM snapshots or backups, the guest may need to be quiesced or frozen, causing probes to fail.

3. **Maintenance Windows**: Administrators need a way to temporarily disable health checks during planned maintenance without modifying the VMI spec.

4. **No Centralized Control**: Different probe types (HTTP, TCP, Exec) are handled differently, making it difficult to implement consistent control policies.

## Goals

- Allow dynamic pausing and unpausing of GuestAgentPing probes via VMI annotations without requiring a Pod restart.
- Return synthetic success responses (exit 0) from the virt-probe utility when probes are paused.
- Enable runtime control exclusively via VMI annotations to maintain a single source of truth.
- Maintain strict backward compatibility by leaving existing Kubernetes Pod probe specifications completely untouched.
- Deliver a lightweight solution that avoids introducing new HTTP proxy servers, API server polling from virt-launcher, or additional RBAC privileges.

## Non Goals

- Replacing Kubernetes native probe mechanisms
- Providing custom health check logic beyond pause/unpause
- Modifying probe results based on complex conditions (beyond pause state)
- Exposing the probe proxy externally (it's only accessible within the Pod)
- Supporting probe pausing across multiple VMIs simultaneously via a single control plane
- Providing HTTP control endpoints for runtime management from within the Pod (annotation-only control)

## Definition of Users

- **VM User**: A person who runs VMs on KubeVirt and wants their VMs to remain stable during maintenance operations
- **Cluster Administrator**: A person who manages the KubeVirt infrastructure and needs to control probe behavior during cluster-wide operations

## User Stories

- As a VM user, I want my VM to not be restarted during live migration due to probe failures
- As a VM user, I want probes to automatically be paused when I add an annotation to my VMI
- As a VM user, I want my VM to not be forcefully restarted by KubeVirt during the reboot of the guest upgrade process, so that I can install guest OS updates on VMs with liveliness probes enabled
- As a cluster administrator, I want to pause probes on specific VMs during scheduled maintenance windows
- As a cluster administrator, I want to be able to query the current probe pause status of a VM via its annotation

## Use Cases

### Supported Use Cases

1. **Live Migration**: Automatically pause probes during live migration to prevent probe failures from interfering with the migration process
2. **Backup/Snapshot**: Pause probes before freezing the guest filesystem for backup, unpause after backup completes
3. **Planned Maintenance**: Add annotation to VMI before maintenance, remove after completion
4. **Troubleshooting**: Temporarily pause probes while investigating VM health issues

### Future Possible Use Cases

1. **Integration with VirtualMachineSnapshot**: Automatically pause probes when taking snapshots
2. **Integration with Migration Controller**: Automatically pause probes during migration phases
3. **Conditional Probe Responses**: Return different probe results based on VM lifecycle state

### Unsupported Use Cases

1. **Permanent probe disabling**: This feature is meant for temporary pausing, not permanent disabling
2. **Custom probe responses**: Only success/failure responses are supported, not custom response bodies
3. **Probe modification**: The feature doesn't modify probe behavior, only intercepts and optionally bypasses

## Repos

kubevirt/kubevirt

## Design

All alternatives below use the same user-facing interface: VMI annotations control probe pausing.

### Pausing Probes via Annotation

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: my-vm
  annotations:
    kubevirt.io/probes-paused: "true"  # Probes will return success without executing
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: containerdisk
    resources:
      requests:
        memory: 1Gi
  readinessProbe:
    tcpSocket:
      port: 22
    initialDelaySeconds: 10
    periodSeconds: 5
  livenessProbe:
    guestAgentPing: {}
    initialDelaySeconds: 120
    periodSeconds: 20
  volumes:
  - containerDisk:
      image: quay.io/containerdisks/fedora
    name: containerdisk
```

### Dynamically Pausing Probes

```bash
# Add annotation to pause probes on running VMI
kubectl annotate vmi my-vm kubevirt.io/probes-paused=true

# Check probe pause status by inspecting the VMI annotation
kubectl get vmi my-vm -o jsonpath='{.metadata.annotations.kubevirt\.io/probes-paused}'
# Output: true

# Remove annotation to unpause probes
kubectl annotate vmi my-vm kubevirt.io/probes-paused-

# Verify probes are active again (annotation should be absent)
kubectl get vmi my-vm -o jsonpath='{.metadata.annotations.kubevirt\.io/probes-paused}'
# Output: (empty - annotation removed)
```

## Alternatives

The preferred alternative is number 1, the other alternatives (Alternatives 2, 3, and 4) are included in this document to demonstrate the full spectrum of options we explored. While these alternatives offer broader technical capabilities—such as intercepting all probe types (HTTP, TCP, Exec) or polling the Kubernetes API directly—they introduce significant complexity, potential API server strain, and higher resource overhead.

### Alternative 1: ConfigMap-Based Guest Agent Probe Update (No Proxy Server)

This alternative targets only the **GuestAgentPing** probe type and eliminates the need for a proxy server inside `virt-launcher` entirely. `virt-handler` writes probe maintenance configuration to a ConfigMap, which is mounted into `virt-launcher`. A lightweight function inside `virt-launcher` periodically reads the mounted file and adjusts probe behavior accordingly.

#### Design

1. **ConfigMap Creation**: `virt-handler` creates and maintains a ConfigMap per VMI containing probe maintenance configuration. When the user sets the `kubevirt.io/probes-paused` annotation on the VMI, `virt-handler` reconciles the annotation into the ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: vmi-probe-maintenance-<vmi-name>
  namespace: <namespace>
  ownerReferences:
  - apiVersion: kubevirt.io/v1
    kind: VirtualMachineInstance
    name: <vmi-name>
data:
  probe-maintenance.yaml: |
    probesPaused: true
    pausedSince: "2026-02-16T10:00:00Z"
```

2. **Volume Mount**: The ConfigMap is mounted read-only into `virt-launcher`:

```yaml
volumes:
- name: probe-maintenance
  configMap:
    name: vmi-probe-maintenance-<vmi-name>
containers:
- name: compute
  volumeMounts:
  - name: probe-maintenance
    mountPath: /etc/kubevirt/probe-maintenance
    readOnly: true
```

3. **Periodic File Check**: Inside `virt-launcher`, instead of running an HTTP proxy server, a simple function is started as a goroutine that:
   - Periodically reads the mounted file at `/etc/kubevirt/probe-maintenance/probe-maintenance.yaml` (e.g., every 5 seconds)
   - Parses the `probesPaused` field
   - Updates an in-memory atomic boolean that the existing guest agent ping probe logic checks before executing

4. **Guest Agent Probe Modification**: The existing guest agent ping probe handler (the exec-based `virt-probe` command) is updated to check the paused state before executing:
   - If paused: return success immediately without contacting the guest agent
   - If not paused: execute the guest agent ping as normal

This means the Kubernetes Pod probe definition does **not** change — it still uses the existing exec-based `virt-probe` command. The only change is that `virt-probe` now checks the mounted maintenance file before deciding whether to actually ping the guest agent.

5. **Control Flow**:

```
┌───────────────────────────────────────────────────────────────┐
│                    virt-handler                               │
│                                                               │
│   VMI annotation:                                             │
│   kubevirt.io/probes-paused: "true"                           │
│           │                                                   │
│           ▼                                                   │
│   Reconcile → Update ConfigMap                                │
│               vmi-probe-maintenance-<name>                    │
└───────────────────────────────────────────────────────────────┘
                         │
                         │ (Kubernetes ConfigMap propagation)
                         ▼
┌───────────────────────────────────────────────────────────────┐
│                    virt-launcher Pod                          │
│                                                               │
│   /etc/kubevirt/probe-maintenance/probe-maintenance.yaml      │
│           │                                                   │
│           ▼                                                   │
│   ┌─────────────────────────────────────────────────────┐     │
│   │  Periodic File Reader (goroutine, every 5s)         │     │
│   │  → Reads mounted ConfigMap file                     │     │
│   │  → Updates atomic bool: probesPaused                │     │
│   └─────────────────────────────────────────────────────┘     │
│                                                               │
│   ┌─────────────────────────────────────────────────────┐     │
│   │  virt-probe (GuestAgentPing)                        │     │
│   │  → Checks probesPaused flag                         │     │
│   │  → If paused: return success (exit 0)               │     │
│   │  → If not paused: ping guest agent as normal        │     │
│   └─────────────────────────────────────────────────────┘     │
│                                                               │
│   ┌─────────────────────────────────────────────────────┐     │
│   │  VM / Guest Agent                                   │     │
│   └─────────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────────┘
```

**Pros:**
- No new server or listener inside `virt-launcher` — significantly reduces complexity and attack surface
- No changes to Pod probe specifications — existing probe definitions remain untouched
- No Kubernetes API calls from `virt-launcher` — `virt-handler` handles all API interactions
- Extremely lightweight — a single goroutine reading a local file
- Simple to implement and reason about — minimal moving parts
- Lower risk of regressions — does not modify how kubelet invokes probes

**Cons:**
- Only covers the GuestAgentPing probe type — HTTP, TCP, and generic Exec probes are not supported
- ConfigMap propagation delay (10-120 seconds depending on kubelet configuration)
- Additional Kubernetes resource (ConfigMap) per VMI
- `virt-handler` must manage the ConfigMap lifecycle (create, update, delete will be done via ownerReference)
- Less extensible for future probe control features that may need to cover all probe types

### Alternative 2: HTTP Proxy Server with ConfigMap Approach

This alternative introduces a Probe Proxy HTTP server in `virt-launcher` that intercepts **all** probe types (HTTP, TCP, Exec, GuestAgentPing). The proxy reads its pause configuration from a locally mounted ConfigMap file, which is managed by `virt-handler` based on VMI annotations.

#### Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│  User: kubectl annotate vmi my-vm kubevirt.io/probes-paused=true  │
└───────────────────────┬───────────────────────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────────────────────┐
│                      virt-handler                                 │
│                                                                   │
│   VMI watch detects annotation change                             │
│           │                                                       │
│           ▼                                                       │
│   Reconcile → Update ConfigMap vmi-probe-config-<name>            │
│               data: probesPaused: true                            │
└───────────────────────────────────────────────────────────────────┘
                        │
                        │ (Kubernetes ConfigMap propagation)
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Kubernetes Kubelet                       │
│                                                                 │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐    │
│   │ Liveness     │   │ Readiness    │   │ Startup          │    │
│   │ Probe        │   │ Probe        │   │ Probe            │    │
│   └──────┬───────┘   └──────┬───────┘   └───────┬──────────┘    │
└──────────┼──────────────────┼───────────────────┼───────────────┘
           │                  │                   │
           ▼                  ▼                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                     virt-launcher Pod                            │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Mounted ConfigMap: /etc/kubevirt/probe-config/            │  │
│  └──────────────────────────┬─────────────────────────────────┘  │
│                             │                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    Probe Proxy (:9500)                     │  │
│  │                                                            │  │
│  │   ┌─────────────────────────────────────────────────────┐  │  │
│  │   │  File Watcher (every 5s)                            │  │  │
│  │   │  → Reads mounted ConfigMap file                     │  │  │
│  │   │  → Updates atomic bool: probesPaused                │  │  │
│  │   └─────────────────────────────────────────────────────┘  │  │
│  │                                                            │  │
│  │   Endpoints:                                               │  │
│  │   - /probe/liveness   → HTTP probe handler                 │  │
│  │   - /probe/readiness  → HTTP probe handler                 │  │
│  │   - /probe/tcp        → TCP probe handler                  │  │
│  │   - /probe/exec       → Exec probe handler                 │  │
│  │                                                            │  │
│  │                                                            │  │
│  │   ┌────────────────────────────────────────────────────┐   │  │
│  │   │  If paused:                                        │   │  │
│  │   │    → Return HTTP 200 immediately                   │   │  │
│  │   │  If not paused:                                    │   │  │
│  │   │    → Forward to actual probe target                │   │  │
│  │   └────────────────────────────────────────────────────┘   │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                       VM / Guest Agent                     │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

#### Components

**1. Probe Proxy Server**: An HTTP server that listens e.g. on port 9500 within the `virt-launcher` Pod, intercepts all probe requests, routes them to appropriate handlers based on probe type, and reads its pause configuration from a locally mounted ConfigMap file.

**2. ConfigMap File Watcher**: A background goroutine that periodically reads the mounted ConfigMap file (every 5 seconds), parses the `probesPaused` field, and updates the proxy's internal pause state. Requires no Kubernetes API calls and no service account permissions.

**3. Annotation-to-ConfigMap Sync**: `virt-handler` manages the ConfigMap lifecycle — creates a ConfigMap per VMI during VMI creation (with `probesPaused: false` as default), watches VMI annotations, reconciles `kubevirt.io/probes-paused` into the ConfigMap, and sets `ownerReference` for automatic garbage collection.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: vmi-probe-config-<vmi-name>
  namespace: <namespace>
  ownerReferences:
  - apiVersion: kubevirt.io/v1
    kind: VirtualMachineInstance
    name: <vmi-name>
data:
  probe-config.yaml: |
    probesPaused: true
    pausedSince: "2026-02-16T10:00:00Z"
```

The ConfigMap is mounted read-only into `virt-launcher` at Pod creation time.

When `virt-handler` updates the ConfigMap, Kubernetes automatically propagates the change to the mounted file in the Pod. The propagation delay depends on the kubelet sync period (default 1 minute) and ConfigMap cache TTL (default 1 minute), with a worst-case delay of ~2 minutes.

**4. Probe Transformation**: Modified probe rendering that transforms all VMI probe definitions to target the probe proxy, passes original probe configuration via HTTP headers, and maintains backward compatibility with existing probe specifications.

#### Probe Transformation Details

**HTTP Probes** — Original VMI probe:
```yaml
readinessProbe:
  httpGet:
    port: 8080
    path: /health
```

Transformed Pod probe:
```yaml
readinessProbe:
  httpGet:
    port: 9500
    path: /probe/readiness
    httpHeaders:
    - name: X-KubeVirt-Probe-Type
      value: http
    - name: X-KubeVirt-Original-Host
      value: 127.0.0.1
    - name: X-KubeVirt-Original-Port
      value: "8080"
    - name: X-KubeVirt-Original-Path
      value: /health
```

**TCP Probes** — Original VMI probe:
```yaml
readinessProbe:
  tcpSocket:
    port: 22
```

Transformed Pod probe:
```yaml
readinessProbe:
  httpGet:
    port: 9500
    path: /probe/tcp
    httpHeaders:
    - name: X-KubeVirt-Probe-Type
      value: tcp
    - name: X-KubeVirt-Original-Host
      value: 127.0.0.1
    - name: X-KubeVirt-Original-Port
      value: "22"
```

**Exec Probes** — Transformed to call the proxy's exec endpoint, which then executes the original `virt-probe` command:
```yaml
readinessProbe:
  httpGet:
    port: 9500
    path: /probe/exec
    httpHeaders:
    - name: X-KubeVirt-Probe-Type
      value: exec
    - name: X-KubeVirt-Exec-Command
      value: /usr/bin/virt-probe
    - name: X-KubeVirt-Exec-Args
      value: "--exec\x00cat\x00/tmp/healthy"
    - name: X-KubeVirt-Exec-Timeout
      value: "5"
```

**Pros:**
- Covers all probe types (HTTP, TCP, Exec, GuestAgentPing)
- Zero Kubernetes API calls from `virt-launcher` — reads only a local mounted file
- No service account permissions needed in `virt-launcher` — reduced attack surface
- Uses native Kubernetes ConfigMap propagation — well-understood, reliable mechanism
- ConfigMap can be extended to hold additional probe configuration beyond pause state
- `virt-handler` already watches VMIs, so adding ConfigMap sync is minimal additional work

**Cons:**
- New HTTP server inside `virt-launcher` — increases complexity and attack surface
- All probe definitions must be rewritten to target the proxy — complex transformation logic
- ConfigMap propagation delay (10-120 seconds depending on kubelet configuration)
- Additional Kubernetes resource (ConfigMap) per VMI
- `virt-handler` must manage ConfigMap lifecycle (create, update, garbage collect via ownerReference)
- Two-step propagation (annotation → ConfigMap → mounted file) is harder to debug
- Higher resource overhead

### Alternative 3: Probe Reconfiguration During Live Migration

This alternative avoids runtime probe interception entirely. Instead, it leverages the fact that during KubeVirt live migration a **new target Pod is created** on the destination node. The migration controller  adjusts probe parameters at Pod creation time for the target and manages the source Pod's probe tolerance through the existing migration lifecycle.


#### Design

1. **Target Pod**: When new virt-launcher is created the probe configuration is propagated from VM/VMI object.

2. **Migration Workflow Integration**:

```
┌───────────────────────────────────────────────────────────────┐
│                    virt-controller                            │
│                                                               │
│   1. Migration requested for VMI                              │
│           │                                                   │
│           ▼                                                   │
│   2. Create target Pod with new probe parameters              │
│           │                                                   │
│           ▼                                                   │
│   3. Proceed with migration (memory transfer, switchover)     │
│           │                                                   │
│           ▼                                                   │
│   4. Migration complete                                       │
│      - Source Pod terminated                                  │
│      - Target Pod VM is now running                           │
└───────────────────────────────────────────────────────────────┘
```

**Pros:**
- No new components in `virt-launcher` — no proxy server, no file watcher, no annotation polling
- Uses standard Kubernetes probe mechanisms — no custom interception layer
- Target Pod probes are configured correctly from the start — no race condition between Pod start and probe pause
- No additional Kubernetes resources (no ConfigMaps, no new annotations)

**Cons:**
- **Does not protect the source Pod** — if the migration takes longer than the source Pod's probe tolerance window, probes will fail on the source and kubelet may restart it, aborting the migration
- Does not provide a generic mechanism for users to pause probes on demand

### Alternative 4: HTTP Proxy Server with Direct Annotation Watcher

This alternative is similar to Alternative 2 (HTTP Proxy Server) but instead of using a ConfigMap for configuration delivery, the probe proxy in `virt-launcher` **directly polls the Kubernetes API** to read the VMI's `kubevirt.io/probes-paused` annotation. This is the simplest full-proxy approach — no ConfigMap intermediary, no `virt-handler` sync logic.

#### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kubernetes Kubelet                       │
│                                                                 │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐    │
│   │ Liveness     │   │ Readiness    │   │ Startup          │    │
│   │ Probe        │   │ Probe        │   │ Probe            │    │
│   └──────┬───────┘   └──────┬───────┘   └───────┬──────────┘    │
└──────────┼──────────────────┼───────────────────┼───────────────┘
           │                  │                   │
           ▼                  ▼                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                     virt-launcher Pod                            │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    Probe Proxy (:9500)                     │  │
│  │                                                            │  │
│  │   ┌─────────────────────────────────────────────────────┐  │  │
│  │   │  Annotation Watcher (every 10s)                     │  │  │
│  │   │  → GET VMI from Kubernetes API                      │  │  │
│  │   │  → Checks kubevirt.io/probes-paused annotation      │  │  │
│  │   │  → Updates atomic bool: probesPaused                │  │  │
│  │   └─────────────────────────────────────────────────────┘  │  │
│  │                                                            │  │
│  │   Endpoints:                                               │  │
│  │   - /probe/liveness   → HTTP probe handler                 │  │
│  │   - /probe/readiness  → HTTP probe handler                 │  │
│  │   - /probe/tcp        → TCP probe handler                  │  │
│  │   - /probe/exec       → Exec probe handler                 │  │
│  │                                                            │  │
│  │   ┌────────────────────────────────────────────────────┐   │  │
│  │   │  If paused:                                        │   │  │
│  │   │    → Return HTTP 200 immediately                   │   │  │
│  │   │  If not paused:                                    │   │  │
│  │   │    → Forward to actual probe target                │   │  │
│  │   └────────────────────────────────────────────────────┘   │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                       VM / Guest Agent                     │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

#### Components

**1. Probe Proxy Server** (`pkg/virt-launcher/probeproxy/probeproxy.go`): Same HTTP server as Alternative 2 — listens on port 9500, intercepts all probe requests, routes them to appropriate handlers.

**2. Annotation Watcher** (`pkg/virt-launcher/probeproxy/watcher.go`): A background goroutine that periodically (every 10 seconds) fetches the VMI from the Kubernetes API, checks for the `kubevirt.io/probes-paused` annotation, and updates the proxy's internal pause state.

**3. Probe Transformation** (`pkg/virt-controller/services/template.go`): Same probe rewriting as Alternative 2 — transforms all VMI probe definitions to target the proxy on port 9500 with original probe configuration passed via HTTP headers.

#### Pause State Management

The proxy's pause state is controlled directly through VMI annotations:

1. **Initial State**: The `--probes-paused` flag is set on `virt-launcher` startup based on the VMI annotation at launch time
2. **Dynamic Updates**: The annotation watcher polls the VMI every 10 seconds and updates the pause state when the annotation changes

This annotation-only approach ensures that all probe pause state is reflected in the VMI resource (single source of truth) and changes are auditable through standard Kubernetes mechanisms.

**Pros:**
- Covers all probe types (HTTP, TCP, Exec, GuestAgentPing) — same scope as Alternative 2
- Faster propagation (~10 seconds) compared to ConfigMap-based approaches (10-120 seconds)
- Simpler architecture — no ConfigMap intermediary, no `virt-controller` sync logic
- No additional Kubernetes resources — no ConfigMaps to manage
- Single source of truth (VMI annotation) with direct reading — no two-step propagation

**Cons:**
- `virt-launcher` makes 1 GET request per VMI every 10 seconds — at 1000+ VMs this adds ~100 req/s to the API server
- Requires service account in `virt-launcher` with VMI read permissions — increased attack surface
- New HTTP server inside `virt-launcher` — same complexity as Alternative 2
- All probe definitions must be rewritten to target the proxy — same transformation complexity as Alternative 2
- May not work in restricted environments where `virt-launcher` API access is limited
- Higher resource overhead

### Alternatives Comparison

| Aspect | Alt 1: Guest Agent + ConfigMap | Alt 2: HTTP Proxy + ConfigMap | Alt 3: Migration Reconfig | Alt 4: HTTP Proxy + Annotation |
|--------|--------------------------------|-------------------------------|---------------------------|-------------------------------|
| **Probe scope** | GuestAgentPing only | All types | All types (migration only) | All types |
| **Use case scope** | All (migration, backup, maintenance) | All | Live migration only | All |
| **virt-launcher changes** | 1 goroutine (file reader) | HTTP server + file watcher | None | HTTP server + API watcher |
| **virt-controller changes** | None | ConfigMap sync + probe transform | Migration-aware Pod template | Probe transformation |
| **virt-handler changes** | ConfigMap lifecycle | None | None | None |
| **Pod probe spec changes** | None | All probes rewritten | Parameters adjusted at creation | All probes rewritten |
| **API calls from virt-launcher** | 0 | 0 | 0 | 1 GET/10s per VMI |
| **ConfigMaps per VMI** | 1 | 1 | 0 | 0 |
| **Propagation delay** | 10-120s | 10-120s | N/A | ~10s |
| **Protects source Pod** | Yes | Yes | No | Yes |
| **Complexity** | Low | High | Low-Moderate | High |
| **Scalability (1000+ VMs)** | Good (0 API calls) | Good (0 API calls) | Good (0 API calls) | Poor (100 req/s) |

## Update/Rollback Compatibility

- **Upgrade**: The feature is strictly backward compatible. Running VMs created prior to the update will not have the ConfigMap mounted and will simply continue executing `GuestAgentPing` probes normally. Newly created or restarted VMs will have the ConfigMap mounted and will immediately support dynamic pausing.

- **Rollback**: 
  - If rolling back to a KubeVirt version without this feature, `virt-handler` will stop managing the probe ConfigMaps.
  - Running VMs that already have the ConfigMap mounted will continue to function safely; the downgraded `virt-probe` binary will simply ignore the mounted file and execute probes as it did before.
  - To completely clear the ConfigMap mounts and underlying resources, VMs must be stopped and recreated.

- **Feature Flag**: A feature gate `GuestAgentProbePause` (or `DynamicProbeControl`) controls this functionality:
  - **When disabled**: Probes work exactly as before. `virt-handler` does not generate the ConfigMap, and `virt-probe` contacts the guest agent directly without checking for a pause state.
  - **When enabled**: `virt-handler` reconciles the VMI annotation into a ConfigMap, mounts it into the `virt-launcher` pod, and `virt-probe` checks this file to determine if it should return a synthetic success response.

## Functional Testing Approach

### Unit Tests

1. Depends on selected alternative

### Integration Tests

1. Depends on selected alternative

### E2E Tests

1. Depends on selected alternative

## Implementation History

- 2026-02-16: Initial VEP proposal created

## Graduation Requirements

### Alpha

- [ ] Feature gate `ProbeProxy` guards all code changes
- Depends on selected alternative

### Beta

- Depends on selected alternative

### GA

- Depends on selected alternative


## References

- [Kubernetes Probes Documentation](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [KubeVirt Probes Documentation](https://kubevirt.io/user-guide/user_workloads/liveness_and_readiness_probes/)
- [KubeVirt Live Migration](https://kubevirt.io/user-guide/compute/live_migration/)
