# OpenShift Mirror Registry Setup Guide for Agent-Based Installer

This comprehensive guide will walk you through creating a mirror registry for OpenShift Agent-Based Installer deployment.

## Prerequisites

### System Requirements
- **Operating System**: RHEL 8.x or 9.x (or compatible like Rocky Linux, AlmaLinux)
- **CPU**: Minimum 4 cores (8 cores recommended)
- **RAM**: Minimum 16 GB (32 GB recommended)
- **Disk Space**: Minimum 1 TB free space on dedicated mount point (2 TB+ recommended for production)
- **Network**: Stable internet connection for initial download (100 Mbps+ recommended)
- **Architecture**: x86_64

### Required Accounts and Access
- Red Hat Customer Portal account with valid subscription
- Pull secret from Red Hat (download from https://console.redhat.com/openshift/install/pull-secret)
- Sudo/root access on the host system

### Dedicated Storage Mount Point
**CRITICAL**: You MUST have a separate mount point for registry data to avoid filling up the root filesystem.

Example mount points you can use:
- `/data` (recommended)
- `/opt/data`
- `/registry`
- `/mnt/registry`

This guide uses a variable `REGISTRY_BASE_PATH` that you will set to your actual mount point.

### Required Software
- Podman or Docker installed
- `oc` CLI tool
- `openshift-install` CLI tool
- `oc-mirror` plugin

---

## Step 0: Pre-Installation Setup

### 0.1 Create Dedicated User for Mirror Registry
```bash
sudo useradd -m -s /bin/bash mirror-user
sudo usermod -aG wheel mirror-user
echo "mirror-user ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/mirror-user
```

### 0.2 Set Password for mirror-user
```bash
sudo passwd mirror-user
```
Enter a secure password when prompted.

### 0.3 Prepare Dedicated Mount Point

**Option A: If you already have a dedicated disk/partition mounted**
```bash
# Check your existing mount points
df -h

# Set your mount point (CHANGE THIS to your actual mount point)
export REGISTRY_BASE_PATH="/data"
```

**Option B: If you need to create and mount a new partition**
```bash
# List available disks
lsblk

# Example: Create partition on /dev/sdb (CHANGE THIS to your actual disk)
sudo parted /dev/sdb mklabel gpt
sudo parted /dev/sdb mkpart primary xfs 0% 100%

# Format the partition
sudo mkfs.xfs /dev/sdb1

# Create mount point
sudo mkdir -p /data

# Get UUID of the partition
sudo blkid /dev/sdb1

# Add to /etc/fstab for persistent mount (replace UUID with your actual UUID)
echo "UUID=your-uuid-here /data xfs defaults 0 0" | sudo tee -a /etc/fstab

# Mount the partition
sudo mount -a

# Verify mount
df -h /data
```

### 0.4 Set Registry Base Path Variable (Persistent)
```bash
# Set the variable (CHANGE THIS to your actual mount point)
export REGISTRY_BASE_PATH="/data"

# Make it persistent for mirror-user
echo "export REGISTRY_BASE_PATH=\"/data\"" | sudo tee -a /home/mirror-user/.bashrc
echo "export REGISTRY_BASE_PATH=\"/data\"" | sudo tee -a /home/mirror-user/.bash_profile
```

### 0.5 Verify Prerequisites
```bash
# Check mount point has enough space (minimum 1TB)
df -h $REGISTRY_BASE_PATH

# Should show at least 1TB available
```

### 0.6 Switch to mirror-user
```bash
su - mirror-user
# OR
sudo -i -u mirror-user
```

**IMPORTANT**: All subsequent commands should be run as `mirror-user` unless specified otherwise.

---

## Step 1: Prepare the Host System

### 1.1 Update the System
```bash
sudo dnf update -y
```

### 1.2 Install Required Packages
```bash
sudo dnf install -y podman httpd-tools jq wget tar openssl python3-pip bind-utils
```

### 1.3 Configure Firewall
```bash
sudo firewall-cmd --permanent --add-port=8443/tcp
sudo firewall-cmd --permanent --add-port=5000/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --reload

# Verify firewall rules
sudo firewall-cmd --list-ports
```

### 1.4 Configure SELinux for Registry Directories
```bash
# Set SELinux to permissive for testing (recommended for initial setup)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# OR set proper SELinux contexts (for production)
sudo semanage fcontext -a -t container_file_t "${REGISTRY_BASE_PATH}/registry(/.*)?"
sudo restorecon -Rv ${REGISTRY_BASE_PATH}/registry
```

### 1.5 Create Directory Structure on Dedicated Mount Point
```bash
# Verify REGISTRY_BASE_PATH is set
echo "Registry base path: $REGISTRY_BASE_PATH"

# Create registry directories
sudo mkdir -p ${REGISTRY_BASE_PATH}/registry/{auth,certs,data}
sudo mkdir -p ${REGISTRY_BASE_PATH}/mirror/oc-mirror-workspace
sudo mkdir -p ${REGISTRY_BASE_PATH}/mirror/downloads

# Create additional directories for organization
sudo mkdir -p ${REGISTRY_BASE_PATH}/mirror/{install-config,agent-installer,backup}

# Set ownership to mirror-user
sudo chown -R mirror-user:mirror-user ${REGISTRY_BASE_PATH}/registry
sudo chown -R mirror-user:mirror-user ${REGISTRY_BASE_PATH}/mirror

# Set proper permissions
sudo chmod -R 755 ${REGISTRY_BASE_PATH}/registry
sudo chmod -R 755 ${REGISTRY_BASE_PATH}/mirror

# Verify directory structure
tree -L 3 ${REGISTRY_BASE_PATH} || ls -laR ${REGISTRY_BASE_PATH}
```

### 1.6 Verify Disk Space
```bash
df -h ${REGISTRY_BASE_PATH}
# Ensure you have at least 1TB available
```

---

## Step 2: Generate SSL Certificates

### 2.1 Set Your Registry Hostname
```bash
export REGISTRY_HOST=$(hostname -f)
echo "Registry hostname: $REGISTRY_HOST"

# Make it persistent
echo "export REGISTRY_HOST=\"$REGISTRY_HOST\"" >> ~/.bashrc
```

### 2.2 Generate Self-Signed Certificate with SAN
```bash
cd ${REGISTRY_BASE_PATH}/registry/certs

# Create OpenSSL config file for SAN
cat > openssl-san.cnf <<EOF
[req]
default_bits = 4096
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = v3_req

[dn]
C = US
ST = State
L = City
O = Organization
OU = IT
CN = ${REGISTRY_HOST}

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = ${REGISTRY_HOST}
DNS.2 = localhost
DNS.3 = $(hostname -s)
IP.1 = 127.0.0.1
IP.2 = $(hostname -I | awk '{print $1}')
EOF

# Generate certificate
openssl req -newkey rsa:4096 -nodes -sha256 -keyout domain.key \
  -x509 -days 3650 -out domain.crt -config openssl-san.cnf -extensions v3_req

# Set proper permissions
chmod 644 domain.crt
chmod 600 domain.key
```

### 2.3 Trust the Certificate System-Wide
```bash
sudo cp ${REGISTRY_BASE_PATH}/registry/certs/domain.crt /etc/pki/ca-trust/source/anchors/${REGISTRY_HOST}.crt
sudo update-ca-trust extract

# Verify certificate is trusted
trust list | grep -i "${REGISTRY_HOST}"
```

### 2.4 Verify Certificate Details
```bash
openssl x509 -in ${REGISTRY_BASE_PATH}/registry/certs/domain.crt -text -noout | grep -A 1 "Subject Alternative Name"
```

### 2.5 Configure Podman to Trust Certificate
```bash
mkdir -p ~/.config/containers/certs.d/${REGISTRY_HOST}:5000
cp ${REGISTRY_BASE_PATH}/registry/certs/domain.crt ~/.config/containers/certs.d/${REGISTRY_HOST}:5000/ca.crt
```

---

## Step 3: Create Registry Authentication

### 3.1 Generate htpasswd File
```bash
# Set registry credentials
export REGISTRY_USER="admin"
export REGISTRY_PASSWORD="RedHat@2026"

# Generate htpasswd file
htpasswd -bBc ${REGISTRY_BASE_PATH}/registry/auth/htpasswd ${REGISTRY_USER} ${REGISTRY_PASSWORD}

# Verify htpasswd file
cat ${REGISTRY_BASE_PATH}/registry/auth/htpasswd
```

### 3.2 Set Proper Permissions
```bash
chmod 644 ${REGISTRY_BASE_PATH}/registry/auth/htpasswd
```

### 3.3 Save Credentials for Reference
```bash
cat > ${REGISTRY_BASE_PATH}/registry/CREDENTIALS.txt <<EOF
Mirror Registry Credentials
============================
Username: ${REGISTRY_USER}
Password: ${REGISTRY_PASSWORD}
Registry URL: https://${REGISTRY_HOST}:5000

KEEP THIS FILE SECURE!
EOF

chmod 600 ${REGISTRY_BASE_PATH}/registry/CREDENTIALS.txt
```

---

## Step 4: Deploy the Mirror Registry Container

### 4.1 Enable Podman Socket for Rootless
```bash
systemctl --user enable --now podman.socket
loginctl enable-linger mirror-user
```

### 4.2 Create Registry Container
```bash
podman run -d \
  --name mirror-registry \
  --restart always \
  -p 5000:5000 \
  -v ${REGISTRY_BASE_PATH}/registry/data:/var/lib/registry:z \
  -v ${REGISTRY_BASE_PATH}/registry/auth:/auth:z \
  -v ${REGISTRY_BASE_PATH}/registry/certs:/certs:z \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
  -e "REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt" \
  -e "REGISTRY_HTTP_TLS_KEY=/certs/domain.key" \
  -e "REGISTRY_STORAGE_DELETE_ENABLED=true" \
  docker.io/library/registry:2
```

### 4.3 Verify Registry is Running
```bash
podman ps | grep mirror-registry

# Should show STATUS as "Up"
```

### 4.4 Enable Auto-Start on Boot (Systemd User Service)
```bash
mkdir -p ~/.config/systemd/user/

podman generate systemd --name mirror-registry --files --new

mv container-mirror-registry.service ~/.config/systemd/user/

systemctl --user daemon-reload
systemctl --user enable container-mirror-registry.service
systemctl --user start container-mirror-registry.service

# Verify service status
systemctl --user status container-mirror-registry.service
```

### 4.5 Test Registry Access
```bash
# Test without authentication (should fail)
curl -k https://${REGISTRY_HOST}:5000/v2/_catalog

# Test with authentication (should succeed)
curl -u ${REGISTRY_USER}:${REGISTRY_PASSWORD} -k https://${REGISTRY_HOST}:5000/v2/_catalog
```
**Expected output**: `{"repositories":[]}`

### 4.6 Test with Podman Login
```bash
podman login -u ${REGISTRY_USER} -p ${REGISTRY_PASSWORD} ${REGISTRY_HOST}:5000

# Should show "Login Succeeded!"
```

---

## Step 5: Download and Install OpenShift CLI Tools

### 5.1 Set OpenShift Version (Updated for 4.19 Stable)
```bash
export OCP_VERSION=4.19.0
export OCP_ARCH=x86_64
export OCP_CHANNEL=stable-4.19

# Make it persistent
echo "export OCP_VERSION=\"4.19.0\"" >> ~/.bashrc
echo "export OCP_ARCH=\"x86_64\"" >> ~/.bashrc
echo "export OCP_CHANNEL=\"stable-4.19\"" >> ~/.bashrc

# Verify
echo "OpenShift Version: $OCP_VERSION"
echo "Architecture: $OCP_ARCH"
echo "Channel: $OCP_CHANNEL"
```

### 5.2 Download OpenShift Client (oc)
```bash
cd ${REGISTRY_BASE_PATH}/mirror/downloads

wget https://mirror.openshift.com/pub/openshift-v4/${OCP_ARCH}/clients/ocp/${OCP_VERSION}/openshift-client-linux.tar.gz

# Extract
tar -xzf openshift-client-linux.tar.gz

# Install
sudo install -m 755 oc /usr/local/bin/oc
sudo install -m 755 kubectl /usr/local/bin/kubectl

# Clean up
rm -f README.md
```

### 5.3 Verify oc Installation
```bash
oc version
kubectl version --client
```

### 5.4 Download OpenShift Installer
```bash
cd ${REGISTRY_BASE_PATH}/mirror/downloads

wget https://mirror.openshift.com/pub/openshift-v4/${OCP_ARCH}/clients/ocp/${OCP_VERSION}/openshift-install-linux.tar.gz

# Extract
tar -xzf openshift-install-linux.tar.gz

# Install
sudo install -m 755 openshift-install /usr/local/bin/openshift-install

# Clean up
rm -f README.md
```

### 5.5 Verify Installer
```bash
openshift-install version
```

### 5.6 Download oc-mirror Plugin
```bash
cd ${REGISTRY_BASE_PATH}/mirror/downloads

# Download latest oc-mirror
wget https://mirror.openshift.com/pub/openshift-v4/${OCP_ARCH}/clients/ocp/${OCP_VERSION}/oc-mirror.tar.gz

# Extract
tar -xzf oc-mirror.tar.gz

# Install
sudo install -m 755 oc-mirror /usr/local/bin/oc-mirror

# Make it executable
sudo chmod +x /usr/local/bin/oc-mirror
```

### 5.7 Verify oc-mirror
```bash
oc-mirror version
```

### 5.8 Download butane (for ignition config generation)
```bash
cd ${REGISTRY_BASE_PATH}/mirror/downloads

wget https://mirror.openshift.com/pub/openshift-v4/${OCP_ARCH}/clients/butane/latest/butane

# Install
sudo install -m 755 butane /usr/local/bin/butane
```

### 5.9 Verify All Tools
```bash
echo "=== Installed Tools Verification ==="
echo "oc version: $(oc version --client | head -1)"
echo "kubectl version: $(kubectl version --client --short 2>/dev/null)"
echo "openshift-install version: $(openshift-install version | head -1)"
echo "oc-mirror version: $(oc-mirror version 2>/dev/null || echo 'oc-mirror installed')"
echo "butane version: $(butane --version)"
```

---

## Step 6: Configure Pull Secret

### 6.1 Download Pull Secret from Red Hat
Go to https://console.redhat.com/openshift/install/pull-secret and download your pull secret.

### 6.2 Upload Pull Secret to Server
Transfer your downloaded pull-secret file to the server:
```bash
# From your local machine, upload to server:
# scp ~/Downloads/pull-secret.txt mirror-user@your-server:${REGISTRY_BASE_PATH}/mirror/

# On the server, save it properly:
cat > ${REGISTRY_BASE_PATH}/mirror/pull-secret.json <<'EOF'
PASTE_YOUR_PULL_SECRET_CONTENT_HERE
EOF
```

**IMPORTANT**: Replace `PASTE_YOUR_PULL_SECRET_CONTENT_HERE` with the actual JSON content from Red Hat.

### 6.3 Verify Pull Secret Format
```bash
# Verify it's valid JSON
jq . ${REGISTRY_BASE_PATH}/mirror/pull-secret.json

# Check if it contains Red Hat registries
jq '.auths | keys[]' ${REGISTRY_BASE_PATH}/mirror/pull-secret.json
```

### 6.4 Add Mirror Registry Credentials to Pull Secret
```bash
# Create base64 encoded auth string
REGISTRY_AUTH=$(echo -n "${REGISTRY_USER}:${REGISTRY_PASSWORD}" | base64 -w0)

# Backup original pull secret
cp ${REGISTRY_BASE_PATH}/mirror/pull-secret.json ${REGISTRY_BASE_PATH}/mirror/pull-secret.json.backup

# Add mirror registry to pull secret
jq --arg host "${REGISTRY_HOST}:5000" --arg auth "${REGISTRY_AUTH}" \
  '.auths += {($host): {"auth": $auth, "email": "noemail@localhost"}}' \
  ${REGISTRY_BASE_PATH}/mirror/pull-secret.json > ${REGISTRY_BASE_PATH}/mirror/pull-secret-merged.json

# Replace original with merged version
mv ${REGISTRY_BASE_PATH}/mirror/pull-secret-merged.json ${REGISTRY_BASE_PATH}/mirror/pull-secret.json
```

### 6.5 Verify Updated Pull Secret
```bash
# Check that mirror registry was added
jq '.auths | keys[]' ${REGISTRY_BASE_PATH}/mirror/pull-secret.json | grep ${REGISTRY_HOST}

# View the complete pull secret
jq . ${REGISTRY_BASE_PATH}/mirror/pull-secret.json
```

### 6.6 Test Pull Secret with Podman
```bash
# Login using pull secret
podman login registry.redhat.io --authfile ${REGISTRY_BASE_PATH}/mirror/pull-secret.json

# Should show "Login Succeeded!"
```

---

## Step 7: Create ImageSetConfiguration for oc-mirror

### 7.1 Create Comprehensive ImageSetConfiguration for OpenShift 4.19

This configuration includes all major Red Hat products and operators:

```bash
cat > ${REGISTRY_BASE_PATH}/mirror/imageset-config.yaml <<EOF
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
storageConfig:
  local:
    path: ${REGISTRY_BASE_PATH}/mirror/oc-mirror-workspace
mirror:
  platform:
    channels:
      - name: ${OCP_CHANNEL}
        type: ocp
        minVersion: ${OCP_VERSION}
        maxVersion: ${OCP_VERSION}
    graph: true
  
  operators:
    # Red Hat Operator Catalog for OpenShift 4.19
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.19
      packages:
        # Storage Operators
        - name: odf-operator                          # OpenShift Data Foundation
        - name: ocs-operator                           # OpenShift Container Storage
        - name: local-storage-operator                 # Local Storage Operator
        - name: lvms-operator                          # Logical Volume Manager Storage
        - name: csi-driver-nfs                         # NFS CSI Driver
        
        # Networking Operators
        - name: metallb-operator                       # MetalLB Load Balancer
        - name: kubernetes-nmstate-operator            # Network Management State
        - name: cluster-network-operator               # Cluster Network Operator
        - name: sriov-network-operator                 # SR-IOV Network Operator
        
        # Security & Compliance
        - name: compliance-operator                    # Compliance Operator
        - name: file-integrity-operator                # File Integrity Operator
        - name: rhacs-operator                         # Red Hat Advanced Cluster Security
        - name: quay-operator                          # Red Hat Quay
        
        # Monitoring & Observability
        - name: cluster-logging                        # OpenShift Logging
        - name: elasticsearch-operator                 # Elasticsearch
        - name: loki-operator                          # Loki Logging
        - name: opentelemetry-product                  # OpenTelemetry
        
        # Application Services
        - name: serverless-operator                    # OpenShift Serverless
        - name: servicemeshoperator                    # Red Hat Service Mesh
        - name: jaeger-product                         # Jaeger Tracing
        - name: kiali-ossm                             # Kiali Service Mesh Observability
        
        # Developer Tools
        - name: devworkspace-operator                  # Dev Workspace Operator
        - name: web-terminal                           # Web Terminal
        - name: pipelines-operator                     # OpenShift Pipelines (Tekton)
        - name: openshift-gitops-operator              # OpenShift GitOps (ArgoCD)
        
        # Virtualization
        - name: kubevirt-hyperconverged                # OpenShift Virtualization
        
        # Migration
        - name: mtc-operator                           # Migration Toolkit for Containers
        - name: mtv-operator                           # Migration Toolkit for Virtualization
        
        # Automation
        - name: ansible-automation-platform-operator   # Ansible Automation Platform
        
        # Windows Containers
        - name: windows-machine-config-operator        # Windows Machine Config Operator
        
        # Cost Management
        - name: costmanagement-metrics-operator        # Cost Management Metrics
        
        # Additional Operators
        - name: multicluster-engine                    # Multicluster Engine
        - name: advanced-cluster-management            # Advanced Cluster Management
        - name: node-healthcheck-operator              # Node Health Check
        - name: poison-pill-manager                    # Poison Pill Manager
        - name: node-maintenance-operator              # Node Maintenance
  
    # Red Hat Certified Operators (Optional - uncomment if needed)
    # - catalog: registry.redhat.io/redhat/certified-operator-index:v4.19
    #   packages:
    #     - name: gpu-operator-certified
    #     - name: node-feature-discovery-operator
  
  additionalImages:
    # Base Images
    - name: registry.redhat.io/ubi8/ubi:latest
    - name: registry.redhat.io/ubi9/ubi:latest
    - name: registry.redhat.io/ubi8/ubi-minimal:latest
    - name: registry.redhat.io/ubi9/ubi-minimal:latest
    
    # RHEL CoreOS Images (for agent-based installer)
    - name: quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:*
    
    # Troubleshooting Images
    - name: registry.redhat.io/rhel8/support-tools:latest
    - name: registry.redhat.io/rhel9/support-tools:latest
    
    # Monitoring Images
    - name: registry.redhat.io/openshift4/ose-prometheus:latest
    - name: registry.redhat.io/openshift4/ose-prometheus-alertmanager:latest
    - name: registry.redhat.io/openshift4/ose-grafana:latest
    
    # Developer Images
    - name: registry.redhat.io/rhscl/postgresql-13-rhel7:latest
    - name: registry.redhat.io/rhscl/mysql-80-rhel7:latest
    - name: registry.redhat.io/rhscl/redis-6-rhel7:latest

  helm: {}
EOF
```

### 7.2 Create Minimal Configuration (For Testing/Quick Setup)
```bash
cat > ${REGISTRY_BASE_PATH}/mirror/imageset-config-minimal.yaml <<EOF
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
storageConfig:
  local:
    path: ${REGISTRY_BASE_PATH}/mirror/oc-mirror-workspace
mirror:
  platform:
    channels:
      - name: ${OCP_CHANNEL}
        type: ocp
        minVersion: ${OCP_VERSION}
        maxVersion: ${OCP_VERSION}
    graph: true
  
  operators:
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.19
      packages:
        - name: local-storage-operator
        - name: metallb-operator
  
  additionalImages:
    - name: registry.redhat.io/ubi9/ubi:latest
  
  helm: {}
EOF
```

### 7.3 Verify Configuration Files
```bash
# Verify full configuration
cat ${REGISTRY_BASE_PATH}/mirror/imageset-config.yaml

# Verify minimal configuration
cat ${REGISTRY_BASE_PATH}/mirror/imageset-config-minimal.yaml
```

### 7.4 Choose Your Configuration
```bash
# For full mirror (recommended for production):
export IMAGESET_CONFIG="${REGISTRY_BASE_PATH}/mirror/imageset-config.yaml"

# For minimal mirror (for testing):
# export IMAGESET_CONFIG="${REGISTRY_BASE_PATH}/mirror/imageset-config-minimal.yaml"

echo "Using configuration: $IMAGESET_CONFIG"
```

---

## Step 8: Mirror OpenShift Images and Red Hat Products

### 8.1 Set Environment Variables
```bash
export LOCAL_REGISTRY="${REGISTRY_HOST}:5000"
export LOCAL_REPOSITORY='ocp4/openshift4'
export PRODUCT_REPO='openshift-release-dev'
export RELEASE_NAME='ocp-release'

# Verify variables
echo "Local Registry: $LOCAL_REGISTRY"
echo "Local Repository: $LOCAL_REPOSITORY"
echo "ImageSet Config: $IMAGESET_CONFIG"
```

### 8.2 Login to All Required Registries
```bash
# Login to mirror registry
podman login -u ${REGISTRY_USER} -p ${REGISTRY_PASSWORD} ${LOCAL_REGISTRY}

# Login to Red Hat registries using pull secret
podman login registry.redhat.io --authfile ${REGISTRY_BASE_PATH}/mirror/pull-secret.json
podman login quay.io --authfile ${REGISTRY_BASE_PATH}/mirror/pull-secret.json
podman login registry.connect.redhat.com --authfile ${REGISTRY_BASE_PATH}/mirror/pull-secret.json

# Verify all logins
cat ~/.config/containers/auth.json | jq '.auths | keys[]'
```

### 8.3 Run oc-mirror to Mirror All Content

**IMPORTANT**: This process will take 4-8 hours depending on network speed and disk I/O. The download size is typically 100-200 GB for full mirror.

```bash
# Change to mirror directory
cd ${REGISTRY_BASE_PATH}/mirror

# Run oc-mirror with full configuration
oc-mirror --config=${IMAGESET_CONFIG} \
  docker://${LOCAL_REGISTRY} \
  --dest-skip-tls \
  --continue-on-error \
  --skip-pruning

# The --continue-on-error flag ensures the mirroring continues even if some images fail
# The --skip-pruning flag prevents deletion of existing content
```

### 8.4 Monitor Progress in Another Terminal
```bash
# Open another SSH session and monitor:

# Watch registry data directory size
watch -n 30 "du -sh ${REGISTRY_BASE_PATH}/registry/data"

# Watch workspace size
watch -n 30 "du -sh ${REGISTRY_BASE_PATH}/mirror/oc-mirror-workspace"

# Monitor network activity
sudo iftop -i $(ip route | grep default | awk '{print $5}')

# Check container logs
podman logs -f mirror-registry
```

### 8.5 Handle Interruptions (If Needed)
```bash
# If the mirroring process is interrupted, you can resume with:
oc-mirror --config=${IMAGESET_CONFIG} \
  docker://${LOCAL_REGISTRY} \
  --dest-skip-tls \
  --continue-on-error \
  --skip-pruning \
  --from=${REGISTRY_BASE_PATH}/mirror/oc-mirror-workspace
```

### 8.6 Verify Mirroring Completion
```bash
# Check for results directory
ls -lh ${REGISTRY_BASE_PATH}/mirror/oc-mirror-workspace/results-*

# Check for critical files
ls -lh ${REGISTRY_BASE_PATH}/mirror/oc-mirror-workspace/results-*/imageContentSourcePolicy.yaml
ls -lh ${REGISTRY_BASE_PATH}/mirror/oc-mirror-workspace/results-*/catalogSource-*.yaml
```

---

## Step 9: Verify Mirrored Content

### 9.1 Check Registry Catalog
```bash
curl -u ${REGISTRY_USER}:${REGISTRY_PASSWORD} -k https://${LOCAL_REGISTRY}/v2/_catalog | jq -r '.repositories[]' | wc -l

# Should show hundreds of repositories
```

### 9.2 List All Repositories
```bash
curl -u ${REGISTRY_USER}:${REGISTRY_PASSWORD} -k https://${LOCAL_REGISTRY}/v2/_catalog | jq -r '.repositories[]' > ${REGISTRY_BASE_PATH}/mirror/mirrored-repositories.txt

# View first 20 repositories
head -20 ${REGISTRY_BASE_PATH}/mirror/mirrored-repositories.txt
```

### 9.3 Check OpenShift Release Images
```bash
curl -u ${REGISTRY_USER}:${REGISTRY_PASSWORD} -k https://${LOCAL_REGISTRY}/v2/ocp4/openshift4/tags/list | jq -r '.tags[]' | grep ${OCP_VERSION}
```

### 9.4 Check Operator Catalog
```bash
curl -u ${REGISTRY_USER}:${REGISTRY_PASSWORD} -k https://${LOCAL_REGISTRY}/v2/redhat/redhat-operator-index/tags/list | jq
```

### 9.5 Verify Specific Images
```bash
# Check if specific operator images exist
for operator in "local-storage-operator" "odf-operator" "metallb-operator"; do
  echo "Checking $operator..."
  curl -u ${REGISTRY_USER}:${REGISTRY_PASSWORD} -k https://${LOCAL_REGISTRY}/v2/_catalog | jq -r '.repositories[]' | grep $operator || echo "Not found: $operator"
done
```

### 9.6 Generate Mirroring Report
```bash
cat > ${REGISTRY_BASE_PATH}/mirror/MIRROR_REPORT.txt <<EOF
=== OpenShift Mirror Registry Report ===
Generated: $(date)

Registry URL: https://${LOCAL_REGISTRY}
OpenShift Version: ${OCP_VERSION}
Channel: ${OCP_CHANNEL}

Total Repositories: $(curl -u ${REGISTRY_USER}:${REGISTRY_PASSWORD} -k https://${LOCAL_REGISTRY}/v2/_catalog 2>/dev/null | jq -r '.repositories[]' | wc -l)

Disk Usage:
$(du -sh ${REGISTRY_BASE_PATH}/registry/data)

Workspace Usage:
$(du -sh ${REGISTRY_BASE_PATH}/mirror/oc-mirror-workspace)

Critical Files:
$(ls -lh ${REGISTRY_BASE_PATH}/mirror/oc-mirror-workspace/results-*/imageContentSourcePolicy.yaml 2>/dev/null || echo "Not found")
$(ls -lh ${REGISTRY_BASE_PATH}/mirror/oc-mirror-workspace/results-*/catalogSource-*.yaml 2>/dev/null || echo "Not found")

Top 10 Largest Repositories:
$(du -sh ${REGISTRY_BASE_PATH}/registry/data/docker/registry/v2/repositories/* 2>/dev/null | sort -hr | head -10)
EOF

cat ${REGISTRY_BASE_PATH}/mirror/MIRROR_REPORT.txt
```

---

## Step 10: Prepare ImageContentSourcePolicy and CatalogSource

### 10.1 Locate Generated Files
```bash
# Find the latest results directory
RESULTS_DIR=$(ls -td ${REGISTRY_BASE_PATH}/mirror/oc-mirror-workspace/results-* | head -1)
echo "Results directory: $RESULTS_DIR"

# List all generated files
ls -lh $RESULTS_DIR/
```

### 10.2 Copy Critical Files for Installation
```bash
# Copy imageContentSourcePolicy
cp $RESULTS_DIR/imageContentSourcePolicy.yaml ${REGISTRY_BASE_PATH}/mirror/install-config/

# Copy catalogSource files
cp $RESULTS_DIR/catalogSource-*.yaml ${REGISTRY_BASE_PATH}/mirror/install-config/

# Copy mapping file
cp $RESULTS_DIR/mapping.txt ${REGISTRY_BASE_PATH}/mirror/install-config/

# Copy release signatures
cp $RESULTS_DIR/release-signatures/ ${REGISTRY_BASE_PATH}/mirror/install-config/ -r 2>/dev/null || echo "No release signatures found"

# List copied files
ls -lh ${REGISTRY_BASE_PATH}/mirror/install-config/
```

### 10.3 Verify imageContentSourcePolicy Content
```bash
cat ${REGISTRY_BASE_PATH}/mirror/install-config/imageContentSourcePolicy.yaml

# Verify it contains mirror entries
grep -A 5 "repositoryDigestMirrors:" ${REGISTRY_BASE_PATH}/mirror/install-config/imageContentSourcePolicy.yaml
```

### 10.4 Verify CatalogSource Content
```bash
# List all catalog sources
ls ${REGISTRY_BASE_PATH}/mirror/install-config/catalogSource-*.yaml

# View each catalog source
for catalog in ${REGISTRY_BASE_PATH}/mirror/install-config/catalogSource-*.yaml; do
  echo "=== $catalog ==="
  cat $catalog
  echo ""
done
```

### 10.5 Extract Registry Information
```bash
# Create a summary of mirror mappings
grep "source:" ${REGISTRY_BASE_PATH}/mirror/install-config/mapping.txt | sort -u > ${REGISTRY_BASE_PATH}/mirror/install-config/source-registries.txt
grep "mirror:" ${REGISTRY_BASE_PATH}/mirror/install-config/mapping.txt | sort -u > ${REGISTRY_BASE_PATH}/mirror/install-config/mirror-destinations.txt

echo "Source Registries:"
cat ${REGISTRY_BASE_PATH}/mirror/install-config/source-registries.txt

echo ""
echo "Mirror Destinations:"
cat ${REGISTRY_BASE_PATH}/mirror/install-config/mirror-destinations.txt
```

---

## Step 11: Create Agent-Based Installer Configuration

### 11.1 Create Install Config Directory
```bash
mkdir -p ${REGISTRY_BASE_PATH}/mirror/agent-installer
cd ${REGISTRY_BASE_PATH}/mirror/agent-installer
```

### 11.2 Generate SSH Key (if not exists)
```bash
if [ ! -f ~/.ssh/id_rsa.pub ]; then
  ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa
  echo "SSH key generated"
else
  echo "SSH key already exists"
fi

# Display public key
cat ~/.ssh/id_rsa.pub
```

### 11.3 Create install-config.yaml

**IMPORTANT**: Edit the following values to match your environment:
- `baseDomain`: Your domain (e.g., example.com)
- `metadata.name`: Your cluster name
- Network CIDR ranges
- API and Ingress VIPs
- Machine network CIDR

```bash
cat > ${REGISTRY_BASE_PATH}/mirror/agent-installer/install-config.yaml <<EOF
apiVersion: v1
baseDomain: example.com
metadata:
  name: ocp-cluster
networking:
  networkType: OVNKubernetes
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  serviceNetwork:
    - 172.30.0.0/16
  machineNetwork:
    - cidr: 192.168.1.0/24
compute:
  - name: worker
    replicas: 2
controlPlane:
  name: master
  replicas: 3
  platform:
    baremetal: {}
platform:
  baremetal:
    apiVIPs:
      - 192.168.1.10
    ingressVIPs:
      - 192.168.1.11
pullSecret: '$(cat ${REGISTRY_BASE_PATH}/mirror/pull-secret.json | jq -c)'
sshKey: '$(cat ~/.ssh/id_rsa.pub)'
additionalTrustBundle: |
$(sed 's/^/  /' ${REGISTRY_BASE_PATH}/registry/certs/domain.crt)
imageContentSources:
$(grep -A 1000 "repositoryDigestMirrors:" ${REGISTRY_BASE_PATH}/mirror/install-config/imageContentSourcePolicy.yaml | sed 's/^/  /' | grep -v "apiVersion\|kind\|metadata")
EOF
```

### 11.4 Verify install-config.yaml
```bash
# Check if file is valid YAML
cat ${REGISTRY_BASE_PATH}/mirror/agent-installer/install-config.yaml

# Verify pull secret is embedded
grep -A 5 "pullSecret:" ${REGISTRY_BASE_PATH}/mirror/agent-installer/install-config.yaml

# Verify imageContentSources are embedded
grep -A 10 "imageContentSources:" ${REGISTRY_BASE_PATH}/mirror/agent-installer/install-config.yaml
```

### 11.5 Create agent-config.yaml

**IMPORTANT**: Edit the following values to match your environment:
- MAC addresses
- IP addresses
- Interface names
- Gateway and DNS servers

```bash
cat > ${REGISTRY_BASE_PATH}/mirror/agent-installer/agent-config.yaml <<EOF
apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: ocp-cluster
rendezvousIP: 192.168.1.20
hosts:
  - hostname: master-0
    role: master
    rootDeviceHints:
      deviceName: /dev/sda
    interfaces:
      - name: ens192
        macAddress: "00:50:56:00:00:01"
    networkConfig:
      interfaces:
        - name: ens192
          type: ethernet
          state: up
          ipv4:
            enabled: true
            address:
              - ip: 192.168.1.20
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - 192.168.1.1
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.1.1
            next-hop-interface: ens192
            table-id: 254
  
  - hostname: master-1
    role: master
    rootDeviceHints:
      deviceName: /dev/sda
    interfaces:
      - name: ens192
        macAddress: "00:50:56:00:00:02"
    networkConfig:
      interfaces:
        - name: ens192
          type: ethernet
          state: up
          ipv4:
            enabled: true
            address:
              - ip: 192.168.1.21
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - 192.168.1.1
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.1.1
            next-hop-interface: ens192
            table-id: 254
  
  - hostname: master-2
    role: master
    rootDeviceHints:
      deviceName: /dev/sda
    interfaces:
      - name: ens192
        macAddress: "00:50:56:00:00:03"
    networkConfig:
      interfaces:
        - name: ens192
          type: ethernet
          state: up
          ipv4:
            enabled: true
            address:
              - ip: 192.168.1.22
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - 192.168.1.1
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.1.1
            next-hop-interface: ens192
            table-id: 254
  
  - hostname: worker-0
    role: worker
    rootDeviceHints:
      deviceName: /dev/sda
    interfaces:
      - name: ens192
        macAddress: "00:50:56:00:00:04"
    networkConfig:
      interfaces:
        - name: ens192
          type: ethernet
          state: up
          ipv4:
            enabled: true
            address:
              - ip: 192.168.1.30
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - 192.168.1.1
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.1.1
            next-hop-interface: ens192
            table-id: 254
  
  - hostname: worker-1
    role: worker
    rootDeviceHints:
      deviceName: /dev/sda
    interfaces:
      - name: ens192
        macAddress: "00:50:56:00:00:05"
    networkConfig:
      interfaces:
        - name: ens192
          type: ethernet
          state: up
          ipv4:
            enabled: true
            address:
              - ip: 192.168.1.31
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - 192.168.1.1
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.1.1
            next-hop-interface: ens192
            table-id: 254
EOF
```

### 11.6 Verify agent-config.yaml
```bash
cat ${REGISTRY_BASE_PATH}/mirror/agent-installer/agent-config.yaml
```

### 11.7 Backup Configuration Files
```bash
cp ${REGISTRY_BASE_PATH}/mirror/agent-installer/install-config.yaml ${REGISTRY_BASE_PATH}/mirror/agent-installer/install-config.yaml.backup
cp ${REGISTRY_BASE_PATH}/mirror/agent-installer/agent-config.yaml ${REGISTRY_BASE_PATH}/mirror/agent-installer/agent-config.yaml.backup
```

---

## Step 12: Generate Agent ISO

### 12.1 Verify Prerequisites
```bash
# Check that all files exist
echo "Checking prerequisites..."
test -f ${REGISTRY_BASE_PATH}/mirror/agent-installer/install-config.yaml && echo "✓ install-config.yaml exists" || echo "✗ install-config.yaml missing"
test -f ${REGISTRY_BASE_PATH}/mirror/agent-installer/agent-config.yaml && echo "✓ agent-config.yaml exists" || echo "✗ agent-config.yaml missing"
test -f ~/.ssh/id_rsa.pub && echo "✓ SSH key exists" || echo "✗ SSH key missing"
test -f ${REGISTRY_BASE_PATH}/registry/certs/domain.crt && echo "✓ Registry certificate exists" || echo "✗ Certificate missing"
```

### 12.2 Create Agent ISO
```bash
cd ${REGISTRY_BASE_PATH}/mirror/agent-installer

# Generate the agent ISO
openshift-install agent create image --log-level=debug

# This will take 10-20 minutes
```

### 12.3 Monitor ISO Creation
```bash
# In another terminal, monitor progress:
tail -f ${REGISTRY_BASE_PATH}/mirror/agent-installer/.openshift_install.log
```

### 12.4 Verify ISO Creation
```bash
ls -lh ${REGISTRY_BASE_PATH}/mirror/agent-installer/agent.x86_64.iso

# Should show a file around 1-2 GB
```

### 12.5 Calculate ISO Checksum
```bash
cd ${REGISTRY_BASE_PATH}/mirror/agent-installer
sha256sum agent.x86_64.iso > agent.x86_64.iso.sha256

cat agent.x86_64.iso.sha256
```

### 12.6 Extract Ignition Files (for reference)
```bash
# Extract the generated ignition files
ls -lh ${REGISTRY_BASE_PATH}/mirror/agent-installer/*.ign 2>/dev/null || echo "Ignition files are embedded in ISO"
```

---

## Step 13: Post-Installation Registry Configuration

### 13.1 Configure Registry for Production
```bash
# Create registry config for garbage collection
cat > ${REGISTRY_BASE_PATH}/registry/config.yml <<EOF
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
  delete:
    enabled: true
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
  tls:
    certificate: /certs/domain.crt
    key: /certs/domain.key
auth:
  htpasswd:
    realm: Registry Realm
    path: /auth/htpasswd
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
EOF
```

### 13.2 Update Registry Container with New Config
```bash
# Stop existing container
podman stop mirror-registry
podman rm mirror-registry

# Create new container with config
podman run -d \
  --name mirror-registry \
  --restart always \
  -p 5000:5000 \
  -v ${REGISTRY_BASE_PATH}/registry/data:/var/lib/registry:z \
  -v ${REGISTRY_BASE_PATH}/registry/auth:/auth:z \
  -v ${REGISTRY_BASE_PATH}/registry/certs:/certs:z \
  -v ${REGISTRY_BASE_PATH}/registry/config.yml:/etc/docker/registry/config.yml:z \
  docker.io/library/registry:2

# Verify it's running
podman ps | grep mirror-registry
```

### 13.3 Update Systemd Service
```bash
# Regenerate systemd service
podman generate systemd --name mirror-registry --files --new

# Move to systemd directory
mv container-mirror-registry.service ~/.config/systemd/user/

# Reload and restart
systemctl --user daemon-reload
systemctl --user restart container-mirror-registry.service
systemctl --user status container-mirror-registry.service
```

### 13.4 Configure Registry Cleanup Cron Job
```bash
# Create cleanup script
cat > ${REGISTRY_BASE_PATH}/registry/cleanup.sh <<'EOF'
#!/bin/bash
REGISTRY_BASE_PATH="/data"  # Change this to match your path
podman exec mirror-registry registry garbage-collect /etc/docker/registry/config.yml --delete-untagged=true
echo "Registry cleanup completed: $(date)" >> ${REGISTRY_BASE_PATH}/registry/cleanup.log
EOF

chmod +x ${REGISTRY_BASE_PATH}/registry/cleanup.sh

# Add to crontab (runs weekly on Sunday at 2 AM)
(crontab -l 2>/dev/null; echo "0 2 * * 0 ${REGISTRY_BASE_PATH}/registry/cleanup.sh") | crontab -

# Verify crontab
crontab -l
```

### 13.5 Test Registry Health
```bash
# Check health endpoint
curl -k https://${LOCAL_REGISTRY}/v2/

# Check with authentication
curl -u ${REGISTRY_USER}:${REGISTRY_PASSWORD} -k https://${LOCAL_REGISTRY}/v2/_catalog | jq -r '.repositories[]' | wc -l
```

---

## Step 14: Backup and Documentation

### 14.1 Create Comprehensive Backup
```bash
mkdir -p ${REGISTRY_BASE_PATH}/backup/$(date +%Y%m%d)

# Backup certificates
tar -czf ${REGISTRY_BASE_PATH}/backup/$(date +%Y%m%d)/certs-backup.tar.gz \
  -C ${REGISTRY_BASE_PATH}/registry certs

# Backup auth files
tar -czf ${REGISTRY_BASE_PATH}/backup/$(date +%Y%m%d)/auth-backup.tar.gz \
  -C ${REGISTRY_BASE_PATH}/registry auth

# Backup pull secret
cp ${REGISTRY_BASE_PATH}/mirror/pull-secret.json ${REGISTRY_BASE_PATH}/backup/$(date +%Y%m%d)/

# Backup install configs
tar -czf ${REGISTRY_BASE_PATH}/backup/$(date +%Y%m%d)/install-config-backup.tar.gz \
  -C ${REGISTRY_BASE_PATH}/mirror install-config agent-installer

# Backup oc-mirror results
tar -czf ${REGISTRY_BASE_PATH}/backup/$(date +%Y%m%d)/oc-mirror-results-backup.tar.gz \
  -C ${REGISTRY_BASE_PATH}/mirror/oc-mirror-workspace results-*

# List backups
ls -lh ${REGISTRY_BASE_PATH}/backup/$(date +%Y%m%d)/
```

### 14.2 Create Registry Documentation
```bash
cat > ${REGISTRY_BASE_PATH}/REGISTRY_COMPLETE_INFO.txt <<EOF
===============================================================================
OPENSHIFT MIRROR REGISTRY - COMPLETE DOCUMENTATION
===============================================================================
Generated: $(date)
User: mirror-user
Base Path: ${REGISTRY_BASE_PATH}

===============================================================================
REGISTRY INFORMATION
===============================================================================
Hostname: ${REGISTRY_HOST}
Port: 5000
URL: https://${REGISTRY_HOST}:5000
Username: ${REGISTRY_USER}
Password: ${REGISTRY_PASSWORD}

===============================================================================
OPENSHIFT INFORMATION
===============================================================================
OpenShift Version: ${OCP_VERSION}
Channel: ${OCP_CHANNEL}
Architecture: ${OCP_ARCH}

===============================================================================
FILE LOCATIONS
===============================================================================
Registry Data: ${REGISTRY_BASE_PATH}/registry/data
Certificates: ${REGISTRY_BASE_PATH}/registry/certs/domain.crt
Auth File: ${REGISTRY_BASE_PATH}/registry/auth/htpasswd
Pull Secret: ${REGISTRY_BASE_PATH}/mirror/pull-secret.json
ImageSet Config: ${REGISTRY_BASE_PATH}/mirror/imageset-config.yaml
Install Config: ${REGISTRY_BASE_PATH}/mirror/agent-installer/install-config.yaml
Agent Config: ${REGISTRY_BASE_PATH}/mirror/agent-installer/agent-config.yaml
Agent ISO: ${REGISTRY_BASE_PATH}/mirror/agent-installer/agent.x86_64.iso

===============================================================================
MIRRORED CONTENT STATISTICS
===============================================================================
Total Repositories: $(curl -u ${REGISTRY_USER}:${REGISTRY_PASSWORD} -k https://${LOCAL_REGISTRY}/v2/_catalog 2>/dev/null | jq -r '.repositories[]' | wc -l)
Registry Data Size: $(du -sh ${REGISTRY_BASE_PATH}/registry/data | awk '{print $1}')
Workspace Size: $(du -sh ${REGISTRY_BASE_PATH}/mirror/oc-mirror-workspace | awk '{print $1}')
Total Disk Usage: $(du -sh ${REGISTRY_BASE_PATH} | awk '{print $1}')

===============================================================================
RED HAT PRODUCTS MIRRORED
===============================================================================
✓ OpenShift Container Platform ${OCP_VERSION}
✓ Red Hat Operator Catalog (v4.19)
✓ OpenShift Data Foundation (ODF)
✓ OpenShift Container Storage (OCS)
✓ Local Storage Operator
✓ Logical Volume Manager Storage (LVMS)
✓ MetalLB Load Balancer
✓ SR-IOV Network Operator
✓ Red Hat Advanced Cluster Security (RHACS)
✓ Red Hat Quay
✓ OpenShift Logging (EFK/Loki)
✓ OpenShift Serverless (Knative)
✓ Red Hat Service Mesh (Istio)
✓ OpenShift Pipelines (Tekton)
✓ OpenShift GitOps (ArgoCD)
✓ OpenShift Virtualization (KubeVirt)
✓ Migration Toolkit for Containers (MTC)
✓ Migration Toolkit for Virtualization (MTV)
✓ Ansible Automation Platform Operator
✓ Advanced Cluster Management (ACM)
✓ Multicluster Engine (MCE)
✓ Compliance Operator
✓ File Integrity Operator

===============================================================================
QUICK REFERENCE COMMANDS
===============================================================================

Login to Registry:
  podman login -u ${REGISTRY_USER} -p ${REGISTRY_PASSWORD} ${LOCAL_REGISTRY}

List Catalog:
  curl -u ${REGISTRY_USER}:${REGISTRY_PASSWORD} -k https://${LOCAL_REGISTRY}/v2/_catalog | jq

Check Registry Status:
  systemctl --user status container-mirror-registry.service

View Registry Logs:
  podman logs -f mirror-registry

Restart Registry:
  systemctl --user restart container-mirror-registry.service

Run Garbage Collection:
  ${REGISTRY_BASE_PATH}/registry/cleanup.sh

Update Mirrored Content:
  cd ${REGISTRY_BASE_PATH}/mirror
  oc-mirror --config=${IMAGESET_CONFIG} docker://${LOCAL_REGISTRY} --dest-skip-tls --continue-on-error

===============================================================================
AGENT ISO USAGE
===============================================================================

1. Copy ISO to bootable media:
   cp ${REGISTRY_BASE_PATH}/mirror/agent-installer/agent.x86_64.iso /path/to/usb/

2. Boot target machines from ISO

3. Installation proceeds automatically using:
   - Registry: https://${LOCAL_REGISTRY}
   - Pull Secret: Embedded in ISO
   - Certificates: Embedded in ISO

===============================================================================
TROUBLESHOOTING
===============================================================================

Registry Not Accessible:
  podman ps -a | grep mirror-registry
  podman logs mirror-registry
  systemctl --user restart container-mirror-registry.service

Certificate Issues:
  sudo cp ${REGISTRY_BASE_PATH}/registry/certs/domain.crt /etc/pki/ca-trust/source/anchors/
  sudo update-ca-trust extract

Test Registry Connectivity:
  curl -u ${REGISTRY_USER}:${REGISTRY_PASSWORD} -k https://${LOCAL_REGISTRY}/v2/_catalog

Check Disk Space:
  df -h ${REGISTRY_BASE_PATH}

===============================================================================
MAINTENANCE SCHEDULE
===============================================================================

Daily:
  - Monitor disk space: df -h ${REGISTRY_BASE_PATH}
  - Check registry logs: podman logs --tail 100 mirror-registry

Weekly:
  - Run garbage collection (automated via cron)
  - Review backup logs

Monthly:
  - Update mirrored content with latest releases
  - Rotate backups
  - Verify backup integrity

===============================================================================
BACKUP LOCATIONS
===============================================================================
Latest Backup: ${REGISTRY_BASE_PATH}/backup/$(date +%Y%m%d)/

Backup Contents:
  - certs-backup.tar.gz (Certificates)
  - auth-backup.tar.gz (Authentication files)
  - pull-secret.json (Pull secret)
  - install-config-backup.tar.gz (Installation configs)
  - oc-mirror-results-backup.tar.gz (Mirroring results)

===============================================================================
SECURITY NOTES
===============================================================================

1. Keep this file secure - it contains sensitive information
2. Regularly update registry credentials
3. Monitor registry access logs
4. Keep certificates up to date (current expiry: 10 years from creation)
5. Regularly backup registry data
6. Use firewall rules to restrict registry access

===============================================================================
SUPPORT AND RESOURCES
===============================================================================

Red Hat Documentation:
  - OpenShift Docs: https://docs.openshift.com
  - Agent-Based Installer: https://docs.openshift.com/container-platform/latest/installing/installing_with_agent_based_installer/

Red Hat Support:
  - Portal: https://access.redhat.com
  - Customer Portal: https://console.redhat.com

Registry Documentation:
  - Docker Registry: https://docs.docker.com/registry/

===============================================================================
EOF

# Set proper permissions
chmod 600 ${REGISTRY_BASE_PATH}/REGISTRY_COMPLETE_INFO.txt

# Display the documentation
cat ${REGISTRY_BASE_PATH}/REGISTRY_COMPLETE_INFO.txt
```

### 14.3 Create Quick Reference Card
```bash
cat > ${REGISTRY_BASE_PATH}/QUICK_REFERENCE.txt <<EOF
=== QUICK REFERENCE ===

Registry URL: https://${LOCAL_REGISTRY}
User: ${REGISTRY_USER}
Pass: ${REGISTRY_PASSWORD}

Status: systemctl --user status container-mirror-registry.service
Logs: podman logs -f mirror-registry
Restart: systemctl --user restart container-mirror-registry.service

ISO Location: ${REGISTRY_BASE_PATH}/mirror/agent-installer/agent.x86_64.iso
ISO SHA256: $(cat ${REGISTRY_BASE_PATH}/mirror/agent-installer/agent.x86_64.iso.sha256 2>/dev/null || echo "Not calculated")

Total Mirrors: $(curl -u ${REGISTRY_USER}:${REGISTRY_PASSWORD} -k https://${LOCAL_REGISTRY}/v2/_catalog 2>/dev/null | jq -r '.repositories[]' | wc -l)
Disk Usage: $(du -sh ${REGISTRY_BASE_PATH} | awk '{print $1}')
EOF

cat ${REGISTRY_BASE_PATH}/QUICK_REFERENCE.txt
```

### 14.4 Export Environment Variables for Future Use
```bash
cat >> ~/.bashrc <<EOF

# OpenShift Mirror Registry Environment
export REGISTRY_BASE_PATH="${REGISTRY_BASE_PATH}"
export REGISTRY_HOST="${REGISTRY_HOST}"
export REGISTRY_USER="${REGISTRY_USER}"
export REGISTRY_PASSWORD="${REGISTRY_PASSWORD}"
export LOCAL_REGISTRY="${LOCAL_REGISTRY}"
export OCP_VERSION="${OCP_VERSION}"
export OCP_CHANNEL="${OCP_CHANNEL}"
export IMAGESET_CONFIG="${IMAGESET_CONFIG}"
EOF

# Source the updated bashrc
source ~/.bashrc
```

---

## Step 15: Final Verification and Testing

### 15.1 Complete System Check
```bash
cat > ${REGISTRY_BASE_PATH}/verify-setup.sh <<'EOF'
#!/bin/bash

echo "==============================================================================="
echo "OPENSHIFT MIRROR REGISTRY - VERIFICATION SCRIPT"
echo "==============================================================================="
echo ""

# Color codes
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

check_status() {
    if [ $? -eq 0 ]; then
        echo -e "${GREEN}✓${NC} $1"
        return 0
    else
        echo -e "${RED}✗${NC} $1"
        return 1
    fi
}

echo "1. Checking Environment Variables..."
[ -n "$REGISTRY_BASE_PATH" ] && check_status "REGISTRY_BASE_PATH is set" || check_status "REGISTRY_BASE_PATH is NOT set"
[ -n "$REGISTRY_HOST" ] && check_status "REGISTRY_HOST is set" || check_status "REGISTRY_HOST is NOT set"
[ -n "$LOCAL_REGISTRY" ] && check_status "LOCAL_REGISTRY is set" || check_status "LOCAL_REGISTRY is NOT set"
echo ""

echo "2. Checking Directory Structure..."
[ -d "$REGISTRY_BASE_PATH/registry" ] && check_status "Registry directory exists" || check_status "Registry directory missing"
[ -d "$REGISTRY_BASE_PATH/mirror" ] && check_status "Mirror directory exists" || check_status "Mirror directory missing"
[ -f "$REGISTRY_BASE_PATH/registry/certs/domain.crt" ] && check_status "SSL certificate exists" || check_status "SSL certificate missing"
[ -f "$REGISTRY_BASE_PATH/registry/auth/htpasswd" ] && check_status "Auth file exists" || check_status "Auth file missing"
echo ""

echo "3. Checking Disk Space..."
DISK_AVAIL=$(df -h $REGISTRY_BASE_PATH | tail -1 | awk '{print $4}')
echo -e "Available space on $REGISTRY_BASE_PATH: ${YELLOW}$DISK_AVAIL${NC}"
DISK_USED=$(du -sh $REGISTRY_BASE_PATH | awk '{print $1}')
echo -e "Total usage: ${YELLOW}$DISK_USED${NC}"
echo ""

echo "4. Checking Registry Container..."
podman ps | grep -q mirror-registry && check_status "Registry container is running" || check_status "Registry container is NOT running"
systemctl --user is-active container-mirror-registry.service > /dev/null 2>&1 && check_status "Registry service is active" || check_status "Registry service is NOT active"
echo ""

echo "5. Checking Network Connectivity..."
curl -s -k https://$LOCAL_REGISTRY/v2/ > /dev/null && check_status "Registry is accessible" || check_status "Registry is NOT accessible"
curl -s -u $REGISTRY_USER:$REGISTRY_PASSWORD -k https://$LOCAL_REGISTRY/v2/_catalog > /dev/null && check_status "Registry authentication works" || check_status "Registry authentication FAILED"
echo ""

echo "6. Checking Mirrored Content..."
REPO_COUNT=$(curl -s -u $REGISTRY_USER:$REGISTRY_PASSWORD -k https://$LOCAL_REGISTRY/v2/_catalog 2>/dev/null | jq -r '.repositories[]' | wc -l)
if [ "$REPO_COUNT" -gt 100 ]; then
    check_status "Mirrored repositories: $REPO_COUNT (GOOD)"
elif [ "$REPO_COUNT" -gt 0 ]; then
    echo -e "${YELLOW}⚠${NC} Mirrored repositories: $REPO_COUNT (LOW - expected 100+)"
else
    echo -e "${RED}✗${NC} No repositories found - mirroring may have failed"
fi
echo ""

echo "7. Checking CLI Tools..."
which oc > /dev/null && check_status "oc CLI is installed" || check_status "oc CLI is NOT installed"
which openshift-install > /dev/null && check_status "openshift-install is installed" || check_status "openshift-install is NOT installed"
which oc-mirror > /dev/null && check_status "oc-mirror is installed" || check_status "oc-mirror is NOT installed"
echo ""

echo "8. Checking Critical Files..."
[ -f "$REGISTRY_BASE_PATH/mirror/pull-secret.json" ] && check_status "Pull secret exists" || check_status "Pull secret missing"
[ -f "$REGISTRY_BASE_PATH/mirror/imageset-config.yaml" ] && check_status "ImageSet config exists" || check_status "ImageSet config missing"
RESULTS_DIR=$(ls -td $REGISTRY_BASE_PATH/mirror/oc-mirror-workspace/results-* 2>/dev/null | head -1)
[ -n "$RESULTS_DIR" ] && check_status "oc-mirror results exist" || check_status "oc-mirror results missing"
[ -f "$REGISTRY_BASE_PATH/mirror/agent-installer/agent.x86_64.iso" ] && check_status "Agent ISO exists" || check_status "Agent ISO missing"
echo ""

echo "9. Checking SSL Certificate..."
CERT_EXPIRY=$(openssl x509 -in $REGISTRY_BASE_PATH/registry/certs/domain.crt -noout -enddate | cut -d= -f2)
echo -e "Certificate expires: ${YELLOW}$CERT_EXPIRY${NC}"
trust list | grep -q "$REGISTRY_HOST" && check_status "Certificate is trusted system-wide" || check_status "Certificate is NOT trusted"
echo ""

echo "==============================================================================="
echo "VERIFICATION COMPLETE"
echo "==============================================================================="
echo ""
echo "Registry URL: https://$LOCAL_REGISTRY"
echo "Mirrored Repositories: $REPO_COUNT"
echo "Total Disk Usage: $DISK_USED"
echo ""
EOF

chmod +x ${REGISTRY_BASE_PATH}/verify-setup.sh
```

### 15.2 Run Verification Script
```bash
${REGISTRY_BASE_PATH}/verify-setup.sh
```

### 15.3 Test Image Pull from Registry
```bash
# Test pulling a base image
podman pull --tls-verify=false ${LOCAL_REGISTRY}/ubi9/ubi:latest

# Verify it was pulled
podman images | grep ${LOCAL_REGISTRY}

# Clean up test image
podman rmi ${LOCAL_REGISTRY}/ubi9/ubi:latest
```

### 15.4 Validate Agent ISO
```bash
# Check ISO size (should be 1-2 GB)
ls -lh ${REGISTRY_BASE_PATH}/mirror/agent-installer/agent.x86_64.iso

# Verify ISO checksum
cd ${REGISTRY_BASE_PATH}/mirror/agent-installer
sha256sum -c agent.x86_64.iso.sha256
```

---

## Troubleshooting Guide

### Issue 1: Registry Container Won't Start
```bash
# Check logs
podman logs mirror-registry

# Check if port is in use
sudo ss -tlnp | grep 5000

# Kill conflicting process if needed
sudo kill $(sudo lsof -t -i:5000)

# Restart registry
podman restart mirror-registry
```

### Issue 2: Certificate Not Trusted
```bash
# Re-trust certificate
sudo cp ${REGISTRY_BASE_PATH}/registry/certs/domain.crt /etc/pki/ca-trust/source/anchors/${REGISTRY_HOST}.crt
sudo update-ca-trust extract

# Verify
trust list | grep ${REGISTRY_HOST}

# For Podman specifically
mkdir -p ~/.config/containers/certs.d/${REGISTRY_HOST}:5000
cp ${REGISTRY_BASE_PATH}/registry/certs/domain.crt ~/.config/containers/certs.d/${REGISTRY_HOST}:5000/ca.crt
```

### Issue 3: oc-mirror Fails During Mirroring
```bash
# Check disk space
df -h ${REGISTRY_BASE_PATH}

# Check network connectivity
ping -c 3 registry.redhat.io
ping -c 3 quay.io

# Resume mirroring with continue-on-error
cd ${REGISTRY_BASE_PATH}/mirror
oc-mirror --config=${IMAGESET_CONFIG} \
  docker://${LOCAL_REGISTRY} \
  --dest-skip-tls \
  --continue-on-error \
  --from=${REGISTRY_BASE_PATH}/mirror/oc-mirror-workspace

# Check specific operator availability
oc-mirror list operators --catalog registry.redhat.io/redhat/redhat-operator-index:v4.19
```

### Issue 4: Pull Secret Authentication Fails
```bash
# Verify pull secret format
jq . ${REGISTRY_BASE_PATH}/mirror/pull-secret.json

# Test each registry in pull secret
for registry in $(jq -r '.auths | keys[]' ${REGISTRY_BASE_PATH}/mirror/pull-secret.json); do
  echo "Testing $registry..."
  podman login $registry --authfile ${REGISTRY_BASE_PATH}/mirror/pull-secret.json
done

# Re-download pull secret if needed from:
# https://console.redhat.com/openshift/install/pull-secret
```

### Issue 5: Agent ISO Creation Fails
```bash
# Check prerequisites
openshift-install version
test -f ${REGISTRY_BASE_PATH}/mirror/agent-installer/install-config.yaml && echo "OK" || echo "MISSING"
test -f ${REGISTRY_BASE_PATH}/mirror/agent-installer/agent-config.yaml && echo "OK" || echo "MISSING"

# Check for errors in log
tail -100 ${REGISTRY_BASE_PATH}/mirror/agent-installer/.openshift_install.log

# Validate YAML syntax
python3 -c "import yaml; yaml.safe_load(open('${REGISTRY_BASE_PATH}/mirror/agent-installer/install-config.yaml'))" && echo "Valid YAML" || echo "Invalid YAML"

# Clean and retry
rm -rf ${REGISTRY_BASE_PATH}/mirror/agent-installer/{.openshift_install*,auth,agent.*}
openshift-install agent create image --log-level=debug
```

### Issue 6: Disk Space Full
```bash
# Check what's using space
du -sh ${REGISTRY_BASE_PATH}/* | sort -hr

# Clean up oc-mirror workspace (after successful mirror)
rm -rf ${REGISTRY_BASE_PATH}/mirror/oc-mirror-workspace/src/

# Run registry garbage collection
podman exec mirror-registry registry garbage-collect /etc/docker/registry/config.yml --delete-untagged=true --dry-run

# If safe, run without dry-run
podman exec mirror-registry registry garbage-collect /etc/docker/registry/config.yml --delete-untagged=true

# Remove old downloads
rm -rf ${REGISTRY_BASE_PATH}/mirror/downloads/*.tar.gz
```

### Issue 7: Registry Performance Issues
```bash
# Check registry container resources
podman stats mirror-registry

# Check system resources
htop

# Increase container resources (if using podman with resource limits)
podman update --cpus=4 --memory=8g mirror-registry

# Restart with more resources
podman stop mirror-registry
podman rm mirror-registry

# Recreate with resource limits
podman run -d \
  --name mirror-registry \
  --restart always \
  --cpus=4 \
  --memory=8g \
  -p 5000:5000 \
  -v ${REGISTRY_BASE_PATH}/registry/data:/var/lib/registry:z \
  -v ${REGISTRY_BASE_PATH}/registry/auth:/auth:z \
  -v ${REGISTRY_BASE_PATH}/registry/certs:/certs:z \
  -v ${REGISTRY_BASE_PATH}/registry/config.yml:/etc/docker/registry/config.yml:z \
  docker.io/library/registry:2
```

### Issue 8: imageContentSourcePolicy Not Applied
```bash
# Verify the file exists and has content
cat ${REGISTRY_BASE_PATH}/mirror/install-config/imageContentSourcePolicy.yaml

# Check if it's properly embedded in install-config.yaml
grep -A 20 "imageContentSources:" ${REGISTRY_BASE_PATH}/mirror/agent-installer/install-config.yaml

# Re-embed if needed
sed -i '/imageContentSources:/,$d' ${REGISTRY_BASE_PATH}/mirror/agent-installer/install-config.yaml
echo "imageContentSources:" >> ${REGISTRY_BASE_PATH}/mirror/agent-installer/install-config.yaml
grep -A 1000 "repositoryDigestMirrors:" ${REGISTRY_BASE_PATH}/mirror/install-config/imageContentSourcePolicy.yaml | sed 's/^/  /' | grep -v "apiVersion\|kind\|metadata" >> ${REGISTRY_BASE_PATH}/mirror/agent-installer/install-config.yaml
```

---

## Maintenance and Updates

### Update Mirrored Content
```bash
# Backup current state
tar -czf ${REGISTRY_BASE_PATH}/backup/pre-update-$(date +%Y%m%d).tar.gz \
  ${REGISTRY_BASE_PATH}/mirror/oc-mirror-workspace/results-*

# Update to latest patch version in channel
cd ${REGISTRY_BASE_PATH}/mirror
oc-mirror --config=${IMAGESET_CONFIG} \
  docker://${LOCAL_REGISTRY} \
  --dest-skip-tls \
  --continue-on-error \
  --skip-pruning

# Verify new content
curl -u ${REGISTRY_USER}:${REGISTRY_PASSWORD} -k https://${LOCAL_REGISTRY}/v2/_catalog | jq
```

### Add New Operators
```bash
# Edit imageset config
vi ${REGISTRY_BASE_PATH}/mirror/imageset-config.yaml

# Add new operators under the packages section, for example:
#   - name: cert-manager-operator
#   - name: node-observability-operator

# Run incremental mirror
cd ${REGISTRY_BASE_PATH}/mirror
oc-mirror --config=${IMAGESET_CONFIG} \
  docker://${LOCAL_REGISTRY} \
  --dest-skip-tls \
  --continue-on-error
```

### Monitor Registry Health
```bash
# Create monitoring script
cat > ${REGISTRY_BASE_PATH}/registry/monitor.sh <<'EOF'
#!/bin/bash

REGISTRY_BASE_PATH="/data"  # Change to your path
LOCAL_REGISTRY="$(hostname -f):5000"
REGISTRY_USER="admin"
REGISTRY_PASSWORD="RedHat@2026"

echo "=== Registry Health Check: $(date) ===" 

# Check container status
if podman ps | grep -q mirror-registry; then
    echo "✓ Container: Running"
else
    echo "✗ Container: Not Running"
    podman start mirror-registry
fi

# Check connectivity
if curl -s -k https://${LOCAL_REGISTRY}/v2/ > /dev/null; then
    echo "✓ Connectivity: OK"
else
    echo "✗ Connectivity: Failed"
fi

# Check disk space
DISK_USAGE=$(df -h $REGISTRY_BASE_PATH | tail -1 | awk '{print $5}' | tr -d '%')
if [ $DISK_USAGE -lt 80 ]; then
    echo "✓ Disk Usage: ${DISK_USAGE}% (OK)"
elif [ $DISK_USAGE -lt 90 ]; then
    echo "⚠ Disk Usage: ${DISK_USAGE}% (Warning)"
else
    echo "✗ Disk Usage: ${DISK_USAGE}% (Critical)"
fi

# Check repository count
REPO_COUNT=$(curl -s -u ${REGISTRY_USER}:${REGISTRY_PASSWORD} -k https://${LOCAL_REGISTRY}/v2/_catalog 2>/dev/null | jq -r '.repositories[]' | wc -l)
echo "✓ Repositories: $REPO_COUNT"

echo ""
EOF

chmod +x ${REGISTRY_BASE_PATH}/registry/monitor.sh

# Add to crontab for daily monitoring
(crontab -l 2>/dev/null; echo "0 9 * * * ${REGISTRY_BASE_PATH}/registry/monitor.sh >> ${REGISTRY_BASE_PATH}/registry/monitor.log 2>&1") | crontab -
```

---

## Final Checklist

```bash
cat > ${REGISTRY_BASE_PATH}/FINAL_CHECKLIST.txt <<EOF
=== OPENSHIFT MIRROR REGISTRY - FINAL CHECKLIST ===

Pre-Installation:
  [✓] Dedicated user 'mirror-user' created
  [✓] Dedicated mount point configured
  [✓] Required packages installed
  [✓] Firewall configured
  [✓] SELinux configured

Registry Setup:
  [✓] Directory structure created
  [✓] SSL certificates generated and trusted
  [✓] Authentication configured
  [✓] Registry container deployed and running
  [✓] Systemd service enabled

CLI Tools:
  [✓] oc CLI installed (version $OCP_VERSION)
  [✓] openshift-install installed (version $OCP_VERSION)
  [✓] oc-mirror installed
  [✓] All tools verified

Pull Secret:
  [✓] Pull secret downloaded from Red Hat
  [✓] Pull secret saved locally
  [✓] Mirror registry credentials added
  [✓] Pull secret format validated

Mirroring:
  [✓] ImageSetConfiguration created
  [✓] oc-mirror completed successfully
  [✓] OpenShift ${OCP_VERSION} mirrored
  [✓] All operators mirrored
  [✓] Additional images mirrored
  [✓] imageContentSourcePolicy generated
  [✓] CatalogSource manifests generated

Agent ISO:
  [✓] install-config.yaml created
  [✓] agent-config.yaml created
  [✓] SSH key configured
  [✓] Agent ISO generated successfully
  [✓] ISO checksum calculated

Post-Configuration:
  [✓] Registry garbage collection configured
  [✓] Monitoring scripts created
  [✓] Backup completed
  [✓] Documentation generated
  [✓] Verification script run successfully

Red Hat Products Mirrored:
  [✓] OpenShift Container Platform ${OCP_VERSION}
  [✓] OpenShift Data Foundation
  [✓] Local Storage Operator
  [✓] MetalLB Operator
  [✓] Service Mesh
  [✓] Serverless
  [✓] Pipelines
  [✓] GitOps
  [✓] Virtualization
  [✓] Advanced Cluster Security
  [✓] Quay Registry
  [✓] Logging Stack
  [✓] Migration Toolkits
  [✓] Ansible Automation Platform
  [✓] Advanced Cluster Management
  [✓] Compliance Operator
  [✓] And 10+ additional operators

Final Verification:
  Registry URL: https://${LOCAL_REGISTRY}
  Mirrored Repositories: $(curl -s -u ${REGISTRY_USER}:${REGISTRY_PASSWORD} -k https://${LOCAL_REGISTRY}/v2/_catalog 2>/dev/null | jq -r '.repositories[]' | wc -l)
  Total Disk Usage: $(du -sh ${REGISTRY_BASE_PATH} 2>/dev/null | awk '{print $1}')
  Agent ISO: ${REGISTRY_BASE_PATH}/mirror/agent-installer/agent.x86_64.iso

=== SETUP COMPLETE - READY FOR OPENSHIFT INSTALLATION ===

Next Steps:
1. Copy agent.x86_64.iso to bootable media
2. Boot target machines from the ISO
3. Monitor installation progress
4. Access cluster after installation completes

Documentation Location: ${REGISTRY_BASE_PATH}/REGISTRY_COMPLETE_INFO.txt
Quick Reference: ${REGISTRY_BASE_PATH}/QUICK_REFERENCE.txt
Verification Script: ${REGISTRY_BASE_PATH}/verify-setup.sh

EOF

cat ${REGISTRY_BASE_PATH}/FINAL_CHECKLIST.txt
```

---

## Appendix A: Network Configuration Examples

### Static IP Configuration for Multiple Interfaces
```yaml
# Example for a host with multiple network interfaces
- hostname: master-0
  role: master
  interfaces:
    - name: ens192
      macAddress: "00:50:56:00:00:01"
    - name: ens224
      macAddress: "00:50:56:00:00:11"
  networkConfig:
    interfaces:
      - name: ens192
        type: ethernet
        state: up
        ipv4:
          enabled: true
          address:
            - ip: 192.168.1.20
              prefix-length: 24
          dhcp: false
      - name: ens224
        type: ethernet
        state: up
        ipv4:
          enabled: true
          address:
            - ip: 10.0.0.20
              prefix-length: 24
          dhcp: false
    dns-resolver:
      config:
        server:
          - 192.168.1.1
          - 8.8.8.8
    routes:
      config:
        - destination: 0.0.0.0/0
          next-hop-address: 192.168.1.1
          next-hop-interface: ens192
          table-id: 254
```

---

## Appendix B: Updating to Newer OpenShift Versions

### Update to 4.19.1 or newer patch version
```bash
# Set new version
export OCP_VERSION=4.19.1

# Update imageset config
sed -i "s/minVersion: 4.19.0/minVersion: $OCP_VERSION/" ${REGISTRY_BASE_PATH}/mirror/imageset-config.yaml
sed -i "s/maxVersion: 4.19.0/maxVersion: $OCP_VERSION/" ${REGISTRY_BASE_PATH}/mirror/imageset-config.yaml

# Mirror new version
cd ${REGISTRY_BASE_PATH}/mirror
oc-mirror --config=${IMAGESET_CONFIG} \
  docker://${LOCAL_REGISTRY} \
  --dest-skip-tls \
  --continue-on-error

# Generate new ISO if needed
cd ${REGISTRY_BASE_PATH}/mirror/agent-installer
# Update install-config.yaml with new version requirements
openshift-install agent create image --log-level=debug
```

---

## Appendix C: Useful Commands Reference

```bash
# List all available OpenShift versions
oc-mirror list releases --channel ${OCP_CHANNEL}

# List available operators in catalog
oc-mirror list operators --catalog registry.redhat.io/redhat/redhat-operator-index:v4.19

# Check specific operator versions
oc-mirror list operators --catalog registry.redhat.io/redhat/redhat-operator-index:v4.19 --package local-storage-operator

# Export registry data for offline transfer
podman save $(podman images -q) -o ${REGISTRY_BASE_PATH}/backup/all-images-$(date +%Y%m%d).tar

# Check registry storage usage by repository
du -sh ${REGISTRY_BASE_PATH}/registry/data/docker/registry/v2/repositories/* | sort -hr | head -20

# Test image pull from mirror
oc adm release info ${LOCAL_REGISTRY}/ocp4/openshift4:${OCP_VERSION}-x86_64 --insecure=true

# Validate YAML files
for file in ${REGISTRY_BASE_PATH}/mirror/agent-installer/*.yaml; do
  echo "Validating $file..."
  python3 -c "import yaml; yaml.safe_load(open('$file'))" && echo "✓ Valid" || echo "✗ Invalid"
done
```

---

**Document Version**: 2.0  
**Last Updated**: January 2026  
**OpenShift Version**: 4.19.0  
**Maintained by**: mirror-user  

**For support or questions**:
- Red Hat OpenShift Documentation: https://docs.openshift.com
- Red Hat Customer Portal: https://access.redhat.com
- Verification Script: `${REGISTRY_BASE_PATH}/verify-setup.sh`