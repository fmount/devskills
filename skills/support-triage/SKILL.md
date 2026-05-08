---
name: support-triage
description: Triage RHOSO must-gather reports using OMC and the Core Operators Support Enablement guide
argument-hint: "<must-gather-path>"
user-invocable: true
allowed-tools: ["Bash", "Read"]
context: fork
---

# RHOSO Support Triage

Analyze a RHOSO must-gather report offline using OMC, following the diagnostic hierarchy from the RHOSO Core Operators Support Enablement guide. The must-gather is treated as a read-only cluster snapshot.

## General Rules

- The must-gather report is **read-only**. Never modify any file in it.
- Do **not** use any command whose purpose is to communicate over the network.
- Use `omc` commands wherever possible to mirror the live-cluster experience. Fall back to direct file reads (via the Read tool or `grep`) for logs, events, and content OMC does not surface well.
- When reporting findings, always include the `omc` command (or file path) used so the user can independently verify.
- Classify every finding by severity:
  - **CRITICAL**: Actively broken (pods crashing, controlplane status False, degraded cluster operators, nodes NotReady).
  - **WARNING**: Misconfiguration that will likely cause problems (wrong targetNamespace, interface mismatches, StorageClass misalignment).
  - **INFO**: Observations worth noting (non-default settings, missing optional components, items that could not be verified offline).

## Phase 1: Setup

1. **Check for OMC.** Run `which omc`. If not found, tell the user:

   > OMC is required but not found in PATH. Install it with:
   >
   > ```
   > go install github.com/gmeghnag/omc@latest
   > ```
   >
   > Or download a binary from <https://github.com/gmeghnag/omc/releases>

   Then stop.

2. **Locate the must-gather.** The argument is a path to either:
   - An extracted directory
   - A `.tar.xz` archive (extract it first with `tar xf`)

   If no argument was provided, ask the user for the path.

3. **Load into OMC.** Run `omc use <path>`. The path should be the innermost must-gather directory (the one that contains `cluster-scoped-resources/`, `namespaces/`, etc.). If `omc use` is given a parent directory, it may not work. Look inside the provided path for the actual must-gather root:

   ```
   find <path> -maxdepth 3 -name "cluster-scoped-resources" -type d
   ```

   Use the parent of whichever directory is found.

4. **Sanity check.** Run `omc get nodes`. If it returns data, OMC is loaded. If it errors, report the problem and stop.

5. **Detect namespaces.** The default OpenStack namespaces are `openstack` (services) and `openstack-operators` (operators). Confirm they exist:

   ```
   omc get ns openstack
   omc get ns openstack-operators
   ```

   If they have different names, ask the user or attempt to detect them from the must-gather directory structure (look under `namespaces/`).

## Phase 2: Area Selection

Present these diagnostic categories to the user and ask which to run. The user may select one, several, or "all":

1. **Cluster Health** - OCP cluster stability, node status
2. **RHOSO Installation** - Subscription, OpenStack resource, operator pods
3. **Networking** - NNCPs, NADs, MetalLB, DNS
4. **Storage** - StorageClass alignment, PVCs, LVM
5. **Control Plane** - OpenStackControlPlane status, operators, webhooks, events
6. **Data Plane** - Metal3 provisioning, compute connectivity, provision server

Then run only the selected sections.

## Phase 3: Diagnostics

### 1. Cluster Health

**Commands:**

```
omc get clusterversion
omc get co
omc get nodes
omc describe node <name>    # for any node with issues
```

**What to check:**

- `clusterversion`: AVAILABLE should be True, PROGRESSING should be False. If not, note the status message.
- `co` (cluster operators): Every operator should have AVAILABLE=True, PROGRESSING=False, DEGRADED=False. Flag any that deviate. If cluster operators are degraded, note that this is an OCP-level issue, not RHOSO-specific.
- `nodes`: All nodes should be Ready. Check for MemoryPressure, DiskPressure, PIDPressure conditions. If any node shows pressure, flag it.

### 2. RHOSO Installation

**Commands:**

```
omc get subscription -n openstack-operators
omc get subscription -n openstack-operators -o yaml
omc get operatorgroup -n openstack-operators
omc get operatorgroup -n openstack-operators -o yaml
omc get openstack -n openstack-operators
omc get pods -n openstack-operators
omc get csv -n openstack-operators
```

**What to check:**

- **OperatorGroup targetNamespace**: This field should **not** exist. If it does, this is a WARNING. RHOSO must be installed at global cluster-scope. A targetNamespace prevents reconciliation in other namespaces and causes webhook failures when creating OpenStackControlPlane resources outside the targetNamespace.
- **Subscription channel**: Should be `stable-v1.0` for downstream RHOSO. Flag if something else is used.
- **OpenStack resource**: Must exist in the `openstack-operators` namespace (not `openstack` or any other namespace). Check `oc get openstack -n openstack-operators` shows DEPLOYED=True. If the OpenStack resource is in the wrong namespace, flag as CRITICAL.
- **Operator pods**: All should be Running (1/1). The `openstack-operator-controller-init-*` pod is the init controller. The `openstack-operator-controller-manager-*` pod is the main operator. There should be ~23 operator pods when fully deployed. Flag any that are not Running.
- **CSV**: Check the ClusterServiceVersion status. It should show `Succeeded`.

### 3. Networking

**Commands:**

```
omc get network cluster -o jsonpath='{.spec.networkType}'
omc get network.operator cluster -o jsonpath='{.spec.defaultNetwork.ovnKubernetesConfig.gatewayConfig}'
omc get nncp
omc get nnce
omc get nnce <name> -o yaml                    # for any failing NNCE
omc get net-attach-def -n openstack
omc get net-attach-def -n openstack -o yaml
omc get l2advertisement -n metallb-system
omc get l2advertisement -n metallb-system -o yaml
omc get ipaddresspool -n metallb-system
omc get metallb -n metallb-system -o yaml
omc get pods -n metallb-system
omc get dnsmasq -n openstack -o yaml
```

**What to check:**

- **Network type**: Must be `OVNKubernetes`, not `OpenShiftSDN`. Flag if wrong.
- **Global forwarding**: `ipForwarding` should be `Global`. Flag if missing or different.
- **NNCPs**: All should show `Available` / `SuccessfullyConfigured`. If any show `FailedToConfigure`, check the corresponding NNCEs for the error message. Also check for `NoMatchingNode` which indicates bad nodeSelector values.
- **NNCEs**: If an NNCP failed, the NNCE status.conditions will contain the error. Read the NNCE YAML and extract the failure message.
- **NAD interface alignment**: For each NAD in the openstack namespace, extract the `master` field from the JSON config. Compare it against the NNCP interface names. They must match. Mismatches are a common misconfiguration.
- **L2Advertisement interface alignment**: For each L2Advertisement, extract `spec.interfaces`. Compare against NNCP interface names. They must match.
- **MetalLB speaker pods**: Check if MetalLB has a nodeSelector configured. If OSP networks only exist on certain nodes but speakers run on all nodes, this can cause traffic routing problems.
- **DNS**: Check the dnsmasq resource's `spec.options` for the upstream server configuration. Note what upstream DNS servers are configured.

### 4. Storage

**Commands:**

```
omc get sc
omc get osctlplane -n openstack -o jsonpath='{.items[].spec.storageClass}'
omc get osctlplane -n openstack -o yaml | grep storageClass:
omc get pvc -n openstack
omc get lvmcluster -A
omc get nodes -o jsonpath='{.items[*].metadata.annotations.capacity\.topolvm\.io\/local-storage}'
```

**What to check:**

- **StorageClass alignment**: The default StorageClass from `omc get sc` and the storageClass values in the OpenStackControlPlane must refer to StorageClasses that actually exist. Any storageClass value that does not appear in `omc get sc` is a CRITICAL finding (causes Pending pods).
- **Pending PVCs**: Any PVC stuck in `Pending` state usually indicates a StorageClass problem. Cross-reference with the storageClass alignment check.
- **LVM specifics** (if LVM storage is in use):
  - LVMCluster should show `Ready` status, but also check the topolvm annotation on each node. If the annotation is missing, PVCs cannot bind even if LVMCluster says Ready.
  - Check if storage requests use binary format (`Gi`) vs decimal (`G`). Decimal format can cause CSI snapshot issues for future backup/restore.

### 5. Control Plane

**Commands:**

```
omc get openstackcontrolplane -n openstack
omc get openstackversion -n openstack
omc get openstackversion -n openstack -o yaml
omc get pods -n openstack
omc get pods -n openstack | grep -v Running | grep -v Completed
omc get events -n openstack --sort-by=.lastTimestamp
omc logs -n openstack-operators <operator-pod>
omc get mutatingwebhookconfiguration
omc get validatingwebhookconfiguration
omc get mutatingwebhookconfiguration openstack-operator-mutating-webhook-configuration -o yaml
omc get validatingwebhookconfiguration openstack-operator-validating-webhook-configuration -o yaml
```

**What to check:**

- **OpenStackControlPlane status**: Should be `True` / `Setup complete`. If `False`, the MESSAGE field indicates the problem. If it mentions a specific service, that service needs investigation.
- **OpenStackVersion**: Its name must match the OpenStackControlPlane name. A mismatch will cause the validating webhook to reject the OpenStackControlPlane.
- **Pod status**: Look for pods not in Running or Completed state. Common problems:
  - `CrashLoopBackOff`: Check pod logs for the crash reason.
  - `Pending`: Usually a storage or scheduling issue.
  - `Init:CrashLoopBackOff`: For Galera pods after a node reboot, check for SSL certificate errors (`Unable to get certificate from '/etc/pki/tls/certs/galera.crt'`). Reference: <https://access.redhat.com/solutions/7140998>
  - `Terminating` (stuck): For RabbitMQ pods stuck terminating during updates, the resolution involves using `crictl stop` on the node and then force-deleting the pod.
- **Events**: Scan for Warning events. Pay attention to Unhealthy (probe failures), FailedScheduling, FailedMount, and BackOff events.
- **Operator logs**: For any problematic service identified above, check the corresponding operator's logs in openstack-operators namespace. Scan for ERROR and WARN level messages. The operator pod name follows the pattern `<service>-operator-controller-manager-*`.
- **Webhooks**:
  - All webhook services should point to the `openstack-operators` namespace.
  - `namespaceSelector` should be `{}` (empty). A non-empty namespaceSelector prevents webhooks from firing for all namespaces, which can cause defaulting failures and API rejections.
  - Each webhook's `path` should correspond to the correct `rules.resources` Kind.

### 6. Data Plane

**Commands:**

```
omc get provisioning -o yaml
omc get infrastructure cluster -o jsonpath='{.status.platform}'
omc get openstackdataplanenodeset -n openstack
omc get openstackdataplanenodeset -n openstack -o yaml
omc get openstackdataplanedeployment -n openstack
omc get openstackprovisionserver -n openstack
omc get openstackbaremetalset -n openstack
omc get baremetalhost -A
```

**What to check:**

- **Infrastructure platform**: Should be `BareMetal` if Metal3 provisioning is used.
- **Provisioning resource**:
  - `watchAllNamespaces` must be `true` for RHOSO to use Metal3.
  - For PXE: `provisioningNetwork` should be `Managed` and `provisioningInterface` must be set.
  - For Virtual Media: the `provisioning*` fields are not needed, but check `disableVirtualMediaTLS` and `virtualMediaViaExternalNetwork` if there are provisioning failures.
- **OpenStackDataPlaneNodeSet**: Check status conditions. If deployment is stuck, check the network_config and baremetalSetTemplate.ctlplaneInterface.
- **OpenStackProvisionServer**: If using Metal3 provisioning, this should exist and its pod should be Running. If using PXE, the provisioning interface on the OCP node must have an IP.

## Phase 4: Report

After running the selected diagnostics, present a structured report:

```
## Triage Report

### Critical Findings
- [finding with omc command used and guidance]

### Warnings
- [finding with omc command used and guidance]

### Informational
- [observations and things that could not be verified offline]

### Areas Not Checked
- [list any categories the user did not select]
```

## Phase 5: Interactive Follow-up

After presenting the report, tell the user they can:

- Ask to drill deeper into any finding
- Run additional diagnostic categories they skipped earlier
- Request specific `omc` commands against the must-gather
- Ask to examine specific log files or resource YAML from the must-gather
- Ask about any RHOSO troubleshooting topic using the Support Enablement guide knowledge

Remain available and use `omc` and direct file reads to answer follow-up questions.

## Reference

This skill's diagnostic checks are derived from the RHOSO Core Operators Support Enablement guide:
<https://redhat.atlassian.net/wiki/spaces/openstackk8s/blog/374749075/RHOSO+Core+Operators+Support+Enablement>
