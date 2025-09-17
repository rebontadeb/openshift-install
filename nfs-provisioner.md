```yaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: null
  name: nfs-client-provisioner
spec: {}
status: {}
```


```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: nfs-provisioner
parameters:
  archiveOnDelete: "false" 
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
```

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
  - apiGroups: ["apps"]
    resources: ["statefulsets"]
    verbs: ["get", "list", "watch"]
```

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: nfs-client-provisioner
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
```
----
Role and Role Binding
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: test-vm   # Replace with your actual namespace
  name: nfs-client-provisioner-role
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "create", "update", "watch"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: nfs-client-provisioner-rolebinding
  namespace: test-vm  # Replace with your actual namespace
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner  # The name of your service account
    namespace: test-vm            # Namespace of the service account
roleRef:
  kind: Role
  name: nfs-client-provisioner-role  # Role created above
  apiGroup: rbac.authorization.k8s.io
```

---
```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: nfs-provisioner-scc
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegedContainer: false
allowedCapabilities: null
defaultAddCapabilities: null
fsGroup:
  type: RunAsAny
groups: []
priority: null
readOnlyRootFilesystem: false
requiredDropCapabilities:
- KILL
- MKNOD
- SETUID
- SETGID
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
users: []
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
- nfs  # This allows NFS volumes
```




```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  namespace: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: gcr.io/k8s-staging-sig-storage/nfs-subdir-external-provisioner:v4.0.0:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: nfs-provisioner
            - name: NFS_SERVER
              value: 10.3.103.251
            - name: NFS_PATH
              value: /OPEN_SHIFT      
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.3.103.251 
            path: /OPEN_SHIFT   
```


`oc adm policy add-scc-to-user nfs-provisioner-scc -z nfs-client-provisioner`

`oc adm policy add-scc-to-user privileged -z nfs-client-provisioner -n test-vm`

`oc adm policy add-scc-to-user hostmount-anyuid system:serviceaccount:test-vm:nfs-client-provisioner`

