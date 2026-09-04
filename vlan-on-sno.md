# SNO OpenShift Virtualization — VLAN-Backed Linux Bridge for VM Static IP

**Platform:** Single Node OpenShift (SNO) on bare-metal  
**Goal:** Add a VLAN-tagged secondary bridge (`br1`) via NMState so OpenShift Virtualization VMs get a static IP reachable via SSH from bastion  
**Prerequisite:** Physical NIC already enslaved to `br-ex` by OVN-Kubernetes — VLAN goes on top of the NIC, not on `br-ex`

---

## Architecture Overview

```
Bastion VM
    |
    | (VLAN 100 — 192.168.100.0/24)
    |
Physical NIC (e.g. enp2s0)
    ├── br-ex         ← OVN-Kubernetes (cluster traffic) — DO NOT TOUCH
    └── enp2s0.100    ← VLAN subinterface (id: 100)
            └── br1   ← Linux bridge (NMState managed)
                  └── VM eth1 → 192.168.100.50/24
```

---

## Step 0 — Verify Prerequisites

```bash
oc whoami                 # must be cluster-admin
oc get nodes              # SNO node must be Ready
ssh core@<SNO_IP>         # must connect from bastion
```

---

## Step 1 — Install Kubernetes NMState Operator

> NMState is **not** installed by default. Multus is. Install NMState first.

```bash
# Create namespace
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-nmstate
EOF

# Create OperatorGroup
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-nmstate
  namespace: openshift-nmstate
spec:
  targetNamespaces:
  - openshift-nmstate
EOF

# Subscribe to operator
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: kubernetes-nmstate-operator
  namespace: openshift-nmstate
spec:
  channel: stable
  installPlanApproval: Automatic
  name: kubernetes-nmstate-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

Wait for CSV:

```bash
oc get csv -n openshift-nmstate -w
```

**Expected:** `PHASE: Succeeded`

---

## Step 2 — Create NMState Instance

> Must be named `nmstate` — singleton for entire cluster.

```bash
cat <<EOF | oc apply -f -
apiVersion: nmstate.io/v1
kind: NMState
metadata:
  name: nmstate
spec: {}
EOF
```

Verify daemonset pods running:

```bash
oc get pods -n openshift-nmstate
```

**Expected:** `nmstate-handler-*` pods in `Running` state.

---

## Step 3 — Find Physical NIC Name

```bash
ssh core@<SNO_IP> "ip link show | grep -E '^[0-9]+:'"
```

Or inspect node network state:

```bash
oc get nns -o yaml | grep -A5 "type: ethernet" | head -40
```

**Expected:** Identify NIC name (`enp2s0`, `ens3`, etc.)  
**Note:** The NIC will already show as enslaved to `br-ex` — this is expected and correct.

---

## Step 4 — Create NNCP: VLAN Subinterface + Linux Bridge

> Replace `enp2s0` with actual NIC name. Replace `100` with actual VLAN ID.

```yaml
# nmstate-vlan-bridge.yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: vlan100-bridge
spec:
  nodeSelector:
    node-role.kubernetes.io/master: ""
  desiredState:
    interfaces:
      - name: enp2s0.100
        type: vlan
        state: up
        vlan:
          base-iface: enp2s0
          id: 100
      - name: br1
        description: Linux bridge for VM secondary network
        type: linux-bridge
        state: up
        ipv4:
          enabled: false
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: enp2s0.100
```

```bash
oc apply -f nmstate-vlan-bridge.yaml
oc get nncp vlan100-bridge -w
```

**Expected:**

```
NAME              STATUS      REASON
vlan100-bridge    Available   SuccessfullyConfigured
```

If `Degraded`, inspect:

```bash
oc get nnce -A
oc describe nnce -A | grep -A10 "Message"
```

---

## Step 5 — Verify Bridge on Node

```bash
ssh core@<SNO_IP> "ip link show br1 && bridge link show"
```

**Expected:** `br1` UP, `enp2s0.100` listed as bridge port.

---

## Step 6 — Create NetworkAttachmentDefinition

> `type: "bridge"` is the correct CNI plugin name for Linux bridge.

```yaml
# nad-vlan100.yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vlan100-network
  namespace: default
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "vlan100-network",
      "type": "bridge",
      "bridge": "br1",
      "macspoofchk": true,
      "ipam": {}
    }
```

```bash
oc apply -f nad-vlan100.yaml
oc get net-attach-def -n default
```

**Expected:** `vlan100-network` listed.

---

## Step 7 — Create VM with Static IP via cloud-init

> `version: 2` must be top-level in `networkData`. Secondary interface is `eth1`.

```yaml
# vm-static.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-static
  namespace: default
spec:
  running: true
  template:
    spec:
      domain:
        cpu:
          cores: 2
        memory:
          guest: 2Gi
        devices:
          disks:
            - name: rootdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
            - name: default
              masquerade: {}
            - name: vlan100-nic
              bridge: {}
      networks:
        - name: default
          pod: {}
        - name: vlan100-nic
          multus:
            networkName: vlan100-network
      volumes:
        - name: rootdisk
          containerDisk:
            image: quay.io/containerdisks/fedora:40
        - name: cloudinitdisk
          cloudInitNoCloud:
            networkData: |
              version: 2
              ethernets:
                eth1:
                  addresses:
                    - 192.168.100.50/24
                  gateway4: 192.168.100.1
            userData: |
              #cloud-config
              user: fedora
              password: redhat123
              chpasswd: { expire: False }
              ssh_authorized_keys:
                - <PASTE_BASTION_PUBKEY_HERE>
```

```bash
oc apply -f vm-static.yaml
```

---

## Step 8 — Verify VM Running and IP Assigned

```bash
oc get vmi vm-static -n default
```

**Expected:** `PHASE: Running`

Check interface IPs:

```bash
oc get vmi vm-static -n default -o jsonpath='{.status.interfaces}' | python3 -m json.tool
```

**Expected:** Two interfaces:
- `eth0` → pod network IP (cluster internal)
- `eth1` → `192.168.100.50` (VLAN 100)

---

## Step 9 — SSH from Bastion

```bash
# Get bastion pubkey (if not done already)
cat ~/.ssh/id_rsa.pub

# SSH to VM via static IP
ssh fedora@192.168.100.50
```

**Expected:** Shell inside VM.

If unreachable from bastion:

```bash
ping 192.168.100.50
ip route show | grep 192.168.100
```

> Bastion must be on VLAN 100 subnet (`192.168.100.0/24`) or have an L3 route to it. Confirm switch port is tagged for VLAN 100.

---

## Troubleshooting Reference

| Symptom | Command |
|---|---|
| NNCP Degraded | `oc describe nnce -A` |
| `br1` not created on node | `ssh core@<SNO_IP> ip link` — check NIC name typo |
| VM stuck Pending | `oc describe vmi vm-static -n default` |
| `eth1` has no IP in VM | `virtctl console vm-static` → `ip addr show eth1` |
| cloud-init failed | `virtctl console vm-static` → `journalctl -u cloud-init` |
| Bastion cannot reach VM | Confirm VLAN 100 tagged on bastion NIC and upstream switch port |
| `net-attach-def` not found | Check NAD is in same namespace as VM |

---

## Common Mistakes

| Mistake | Effect |
|---|---|
| Using `type: "cnv-bridge"` in NAD | CNI plugin not found — VM fails to start |
| Skipping NMState instance creation | NNCP never enacted |
| `version: 2` not at top of `networkData` | cloud-init ignores network config |
| VLAN on `br-ex` instead of physical NIC | Breaks cluster networking |
| NAD in different namespace from VM | Multus cannot attach secondary NIC |
| Switch port not tagged for VLAN 100 | VM unreachable from bastion even with correct config |

---

## File Reference

| File | Purpose |
|---|---|
| `nmstate-vlan-bridge.yaml` | NNCP — creates `enp2s0.100` VLAN and `br1` Linux bridge |
| `nad-vlan100.yaml` | NetworkAttachmentDefinition — exposes `br1` to VMs |
| `vm-static.yaml` | VirtualMachine with static IP on secondary NIC via cloud-init |




```yaml
# nmstate-vlan-bridge.yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br1-bridge
spec:
  nodeSelector:
    node-role.kubernetes.io/master: ""
  desiredState:
    interfaces:
      - name: br1
        description: Linux bridge on enp113s0f1 for VM secondary network
        type: linux-bridge
        state: up
        ipv4:
          enabled: false
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: enp113s0f1
```
---
```yaml
# nad-br1.yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: br1-network
  namespace: default
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "br1-network",
      "type": "bridge",
      "bridge": "br1",
      "macspoofchk": true,
      "ipam": {}
    }
```
