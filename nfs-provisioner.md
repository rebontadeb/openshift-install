# Complete NFS Provisioner Configuration for OpenShift
# Choose your target namespace - update all instances of 'test-vm' if needed

---
# 1. Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test-vm
  labels:
    name: test-vm
```
---
# 2. ServiceAccount
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: test-vm
```
---
# 3. StorageClass
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: nfs-provisioner
parameters:
  archiveOnDelete: "false"
reclaimPolicy: Delete
allowVolumeExpansion: true
```
---
# 4. ClusterRole
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["volumeattachments"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
```
---
# 5. ClusterRoleBinding
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: test-vm
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
```
---
# 6. Role (for namespace-specific permissions)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: test-vm
  name: nfs-client-provisioner-role
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "create", "update", "watch"]
```
---
# 7. RoleBinding
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: nfs-client-provisioner-rolebinding
  namespace: test-vm
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: test-vm
roleRef:
  kind: Role
  name: nfs-client-provisioner-role
  apiGroup: rbac.authorization.k8s.io
```
---
# 8. SecurityContextConstraints (OpenShift specific)
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
users:
  - system:serviceaccount:test-vm:nfs-client-provisioner
volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
  - nfs
```
---
# 9. Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  namespace: test-vm
  labels:
    app: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
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
          image: gcr.io/k8s-staging-sig-storage/nfs-subdir-external-provisioner:v4.0.0
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: nfs-provisioner
            - name: NFS_SERVER
              value: "10.3.103.251"
            - name: NFS_PATH
              value: "/OPEN_SHIFT"
          resources:
            limits:
              cpu: 500m
              memory: 256Mi
            requests:
              cpu: 100m
              memory: 128Mi
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.3.103.251
            path: /OPEN_SHIFT
```
---
# 10. Test PVC (Optional - for testing purposes)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-nfs-pvc
  namespace: test-vm
  annotations:
    volume.beta.kubernetes.io/storage-class: "nfs-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs-storage
```

`oc adm policy add-scc-to-user nfs-provisioner-scc -z nfs-client-provisioner`

`oc adm policy add-scc-to-user privileged -z nfs-client-provisioner -n test-vm`

`oc adm policy add-scc-to-user hostmount-anyuid system:serviceaccount:test-vm:nfs-client-provisioner`

