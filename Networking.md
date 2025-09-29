# How OpenShift Virtualization VMs Get IPs from VLANs

Let me explain the complete process of how VMs obtain IP addresses from VLANs when using NMState and OpenShift Virtualization.

## Architecture Overview

```
Physical Network (VLAN) → Physical NIC → Node → Bridge/VLAN Interface → CNI → VM
```

## Step-by-Step: VLAN Configuration and IP Assignment

### Step 1: Configure VLAN Interface on Nodes Using NMState

```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: vlan100-bridge-policy
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  desiredState:
    interfaces:
    # Create VLAN interface
    - name: ens3.100  # Format: physical-interface.vlan-id
      type: vlan
      state: up
      vlan:
        base-iface: ens3  # Your physical interface
        id: 100           # VLAN ID
      ipv4:
        enabled: false    # No IP on VLAN interface itself
        dhcp: false
    
    # Create Linux bridge attached to VLAN
    - name: br-vlan100
      type: linux-bridge
      state: up
      ipv4:
        enabled: false
        dhcp: false
      bridge:
        options:
          stp:
            enabled: false
        port:
        - name: ens3.100  # Attach VLAN interface to bridge
```

Apply the configuration:
```bash
oc apply -f vlan-bridge-policy.yaml

# Verify on all nodes
oc get nncp  # NodeNetworkConfigurationPolicy
oc get nnce  # NodeNetworkConfigurationEnactment
```

### Step 2: Create NetworkAttachmentDefinition with DHCP/Static IP

#### Option A: Using DHCP from VLAN Network

```yaml
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
      "type": "cnv-bridge",
      "bridge": "br-vlan100",
      "ipam": {
        "type": "dhcp"
      }
    }
```

**For DHCP to work, you need a DHCP server on VLAN 100 network.**

#### Option B: Using Static IP Assignment

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vlan100-network-static
  namespace: default
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "vlan100-network",
      "type": "cnv-bridge",
      "bridge": "br-vlan100",
      "ipam": {
        "type": "static"
      }
    }
```

#### Option C: Using Whereabouts IPAM (IP Pool Management)

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vlan100-network-pool
  namespace: default
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "vlan100-network",
      "type": "cnv-bridge",
      "bridge": "br-vlan100",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.100.10-192.168.100.200",
        "gateway": "192.168.100.1",
        "routes": [{
          "dst": "0.0.0.0/0",
          "gw": "192.168.100.1"
        }]
      }
    }
```

### Step 3: Create VM with VLAN Network Attachment

#### VM with DHCP from VLAN

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-vlan100-dhcp
  namespace: default
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-vlan100-dhcp
    spec:
      domain:
        devices:
          disks:
          - name: containerdisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virtio
          interfaces:
          # Primary pod network (for k8s communication)
          - name: default
            masquerade: {}
          # VLAN network (gets IP from VLAN DHCP)
          - name: vlan100
            bridge: {}
        resources:
          requests:
            memory: 2Gi
            cpu: 1
      networks:
      - name: default
        pod: {}
      - name: vlan100
        multus:
          networkName: vlan100-network
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/kubevirt/fedora-cloud-container-disk-demo
      - name: cloudinitdisk
        cloudInitNoCloud:
          networkData: |
            version: 2
            ethernets:
              eth1:  # Second interface (VLAN)
                dhcp4: true
          userData: |
            #cloud-config
            password: fedora
            chpasswd: { expire: False }
```

#### VM with Static IP on VLAN

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-vlan100-static
  namespace: default
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-vlan100-static
      annotations:
        k8s.v1.cni.cncf.io/networks: |
          [
            {
              "name": "vlan100-network-static",
              "namespace": "default",
              "ips": ["192.168.100.50/24"],
              "gateway": ["192.168.100.1"]
            }
          ]
    spec:
      domain:
        devices:
          disks:
          - name: containerdisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virtio
          interfaces:
          - name: default
            masquerade: {}
          - name: vlan100
            bridge: {}
        resources:
          requests:
            memory: 2Gi
      networks:
      - name: default
        pod: {}
      - name: vlan100
        multus:
          networkName: vlan100-network-static
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/kubevirt/fedora-cloud-container-disk-demo
      - name: cloudinitdisk
        cloudInitNoCloud:
          networkData: |
            version: 2
            ethernets:
              eth1:
                addresses:
                - 192.168.100.50/24
                gateway4: 192.168.100.1
                nameservers:
                  addresses:
                  - 8.8.8.8
          userData: |
            #cloud-config
            password: fedora
            chpasswd: { expire: False }
```

## Complete Flow: How VMs Get IPs from VLAN

### Flow Diagram

```
1. Physical Switch (VLAN 100)
   ↓ Tagged VLAN Traffic
2. Physical NIC (ens3) on OpenShift Node
   ↓
3. VLAN Interface (ens3.100) - Created by NMState
   ↓
4. Linux Bridge (br-vlan100) - Created by NMState
   ↓
5. CNI Plugin (cnv-bridge) - Configured in NetworkAttachmentDefinition
   ↓
6. veth pair to virt-launcher pod
   ↓
7. TAP device in VM
   ↓
8. VM Network Interface (eth1)
   ↓
9. DHCP Request OR Static IP Configuration
   ↓
10. IP Address from VLAN 100 subnet
```

## Multiple VLANs Configuration

```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: multi-vlan-bridge-policy
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  desiredState:
    interfaces:
    # VLAN 100
    - name: ens3.100
      type: vlan
      state: up
      vlan:
        base-iface: ens3
        id: 100
      ipv4:
        enabled: false
    
    - name: br-vlan100
      type: linux-bridge
      state: up
      ipv4:
        enabled: false
      bridge:
        port:
        - name: ens3.100
    
    # VLAN 200
    - name: ens3.200
      type: vlan
      state: up
      vlan:
        base-iface: ens3
        id: 200
      ipv4:
        enabled: false
    
    - name: br-vlan200
      type: linux-bridge
      state: up
      ipv4:
        enabled: false
      bridge:
        port:
        - name: ens3.200
    
    # VLAN 300
    - name: ens3.300
      type: vlan
      state: up
      vlan:
        base-iface: ens3
        id: 300
      ipv4:
        enabled: false
    
    - name: br-vlan300
      type: linux-bridge
      state: up
      ipv4:
        enabled: false
      bridge:
        port:
        - name: ens3.300
```

Create NetworkAttachmentDefinitions for each VLAN:

```bash
# VLAN 100
cat <<EOF | oc apply -f -
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vlan100-network
  namespace: default
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "vlan100",
      "type": "cnv-bridge",
      "bridge": "br-vlan100",
      "ipam": {"type": "dhcp"}
    }
EOF

# VLAN 200
cat <<EOF | oc apply -f -
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vlan200-network
  namespace: default
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "vlan200",
      "type": "cnv-bridge",
      "bridge": "br-vlan200",
      "ipam": {"type": "dhcp"}
    }
EOF
```

## Verification Commands

```bash
# 1. Verify VLAN interfaces on nodes
oc debug node/<node-name>
chroot /host
ip link show | grep vlan
ip link show | grep br-vlan
bridge link show

# 2. Check VM IP addresses
oc get vmi -A
virtctl guestosinfo <vm-name> -n <namespace>

# 3. Check network attachments
oc get network-attachment-definitions -A

# 4. Verify VM network configuration
virtctl console <vm-name> -n <namespace>
# Inside VM:
ip addr show
ip route show

# 5. Check if DHCP is working (if using DHCP)
oc get pods -A | grep dhcp
oc logs -n openshift-multus <dhcp-daemon-pod>

# 6. Test connectivity from VM
virtctl ssh user@<vm-name> -n <namespace>
ping <gateway-ip>
ping <external-ip>
```

## Troubleshooting VLAN IP Assignment

```bash
# 1. Check NMState configuration
oc get nncp
oc get nnce
oc describe nnce <node-name>

# 2. Check if VLAN tagged traffic reaches the node
tcpdump -i ens3 -nn vlan 100

# 3. Check bridge on node
brctl show br-vlan100

# 4. Check VM pod annotations
oc get pod virt-launcher-<vm-name> -o yaml | grep -A 20 "k8s.v1.cni.cncf.io/networks-status"

# 5. Check Multus CNI logs
oc logs -n openshift-multus <multus-pod>

# 6. Enable DHCP daemon if needed
oc patch networks.operator.openshift.io cluster --type=merge \
  -p '{"spec":{"additionalNetworks":[{"name":"dhcp-shim","namespace":"openshift-multus","type":"DHCP"}]}}'
```

## Key Points

1. **NMState creates the infrastructure**: VLAN interfaces and bridges on nodes
2. **NetworkAttachmentDefinition**: Defines how VMs connect to the bridge
3. **IPAM (IP Address Management)**: Three options:
   - **DHCP**: VM gets IP from external DHCP server on VLAN
   - **Static**: IP manually assigned via annotations
   - **Whereabouts**: OpenShift manages IP pool
4. **VM must be configured**: Cloud-init or guest OS must request/configure the IP
5. **Physical switch must be configured**: VLAN trunk port to the OpenShift node

Would you like me to provide examples for any specific VLAN scenario or IPAM configuration?
