### Get the Device name from any node for br-ex

```bash
oc debug node/node1
```


```bash
ovs-vsctl get Open_vSwitch . external_ids:ovn-bridge-mappings
```


```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: tslocp-nad
  namespace: tslocprj
spec:
  config: |
    {
      "cniVersion": "0.4.0",
      "name": "physnet",
      "type": "ovn-k8s-cni-overlay",
      "topology": "localnet",
      "netAttachDefName": "tslocprj/tslocp-nad",
      "vlanID": 0,
      "mtu": 1400,
      "ipam": {}
    }
```


```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: bridge-network
  namespace: fp-devops
  annotations:
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/br-ex
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "bridge-network",
      "type": "cnv-bridge",
      "bridge": "br-ex"
    }
```
