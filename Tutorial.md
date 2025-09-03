# OpenShift Disconnected Installation Guide

This guide provides step-by-step instructions for installing OpenShift in a disconnected environment using agent-based installation with a mirror registry.

## Prerequisites

- Red Hat OpenShift subscription
- Bastion host with internet access for mirroring
- Target environment without internet access
- Minimum 500GB storage for mirror registry
- Valid SSL certificates or ability to create self-signed certificates

## Required Tools

Download and install the following tools on your bastion host:

```bash
dnf install nmstate wget tar ifconfig podman acl -y
```

```bash
# Download OpenShift CLI tools
mkdir softwares
cd softwares
version=4.19.0
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${version}/openshift-client-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${version}/openshift-install-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${version}/oc-mirror.tar.gz

# Extract tools
tar -xzf openshift-client-linux.tar.gz
tar -xzf openshift-install-linux.tar.gz

# Move to PATH
sudo mv oc kubectl openshift-install /usr/local/bin/

# Download mirror registry
wget https://developers.redhat.com/content-gateway/file/pub/openshift-v4/clients/mirror-registry/1.3.9/mirror-registry.tar.gz
mkdir -p ~/mirror-registry && tar -xzvf mirror-registry.tar.gz -C ~/mirror-registry/
```

Add pull secret in JSON https://console.redhat.com/openshift/downloads#tool-pull-secret 

cat ./pull-secret.json | jq . > auth-pull-secret.json

```
echo -n 'admin:password' | base64 -w0
```
```
"mirror.rebonta-registry.com": {
"auth": "YWRtaW46cGFzc3dvcmQ=",
"email": "rebontadeb@gmail.com"
}
```

## Step 1: Mirror Registry Creation with SSL

### 1.1 Prepare SSL Certificates

Create SSL certificates for your mirror registry:

```bash
# Create directory for certificates
mkdir -p /opt/registry/certs

# Generate private key
openssl genrsa -out /opt/registry/certs/domain.key 2048

# Create certificate signing request
openssl req -new -key /opt/registry/certs/domain.key -out /opt/registry/certs/domain.csr \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=mirror.example.com"

# Generate self-signed certificate (valid for 365 days)
openssl x509 -req -in /opt/registry/certs/domain.csr \
  -signkey /opt/registry/certs/domain.key \
  -out /opt/registry/certs/domain.crt -days 365

# Create certificate bundle
cp /opt/registry/certs/domain.crt /opt/registry/certs/ca-bundle.crt


--------------------------------------------------------------------------------------------------------------------
openssl genrsa -out rootCA.key 2048
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.pem -subj "/C=BD/ST=Dhaka/L=Gulshan/O=TestOrg/OU=OrgUnit/CN=mirror.rebonta-registry.com"
openssl genrsa -out ssl.key 2048
openssl req -new -key ssl.key -out ssl.csr -subj "/C=BD/ST=Dhaka/L=Gulshan/O=TestOrg/OU=OrgUnit/CN=mirror.rebonta-registry.com"

cat > openssl.cnf << EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
countryName = BD
stateOrProvinceName = Dhaka
localityName = Gulshan
organizationName = TestOrg
organizationalUnitName = OrgUnit
commonName = mirror.rebonta-registry.com
commonName_max = 64
[v3_req]
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]

DNS.1 = mirror.rebonta-registry.com
DNS.2 = localhost
IP.1 = 172.31.71.88
IP.2 = 127.0.0.1
EOF

openssl x509 -req -in ssl.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out ssl.crt -days 356 -extensions v3_req -extfile openssl.cnf
sudo mkdir -p /etc/containers/certs.d/mirror.rebonta-registry.com/
sudo cp rootCA.pem /etc/containers/certs.d/mirror.rebonta-registry.com/ca.crt
sudo cp rootCA.pem /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust extract
trust list | grep mirror.rebonta-registry.com
--------------------------------------------------------------------------------------------------------------------

```

### 1.2 Install Mirror Registry

```bash
# Install mirror registry with SSL
./mirror-registry install \
  --quayHostname mirror.example.com \
  --quayRoot /opt/registry \
  --sslCert /opt/registry/certs/domain.crt \
  --sslKey /opt/registry/certs/domain.key \
  --initUser admin \
  --initPassword redhat123

# Verify installation
curl -k https://mirror.example.com:8443/health/instance
```

### 1.3 Configure Trust

```bash
# Copy CA certificate to system trust store
sudo cp /opt/registry/certs/ca-bundle.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

# Configure podman/docker to trust the registry
mkdir -p ~/.config/containers/
cat > ~/.config/containers/registries.conf << EOF
[[registry]]
location = "mirror.example.com:8443"
insecure = false
EOF
```

## Step 2: Mirror OpenShift Images

### 2.1 Set Environment Variables

```bash
export OCP_RELEASE="4.14.8"
export LOCAL_REGISTRY="mirror.example.com:8443"
export LOCAL_REPOSITORY="ocp4/openshift4"
export PRODUCT_REPO="openshift-release-dev"
export RELEASE_NAME="ocp-release"
export ARCHITECTURE="x86_64"
export REMOVABLE_MEDIA_PATH="/opt/registry/mirror"
```

### 2.2 Create Pull Secret

```bash
# Download pull secret from Red Hat Customer Portal
# Save as pull-secret.json

# Create registry authentication
mkdir -p ~/.docker
cat > ~/.docker/config.json << EOF
{
  "auths": {
    "mirror.example.com:8443": {
      "auth": "$(echo -n 'admin:redhat123' | base64 -w0)"
    }
  }
}
EOF

# Merge with Red Hat pull secret
jq -s '.[0] * .[1]' ~/.docker/config.json pull-secret.json > merged-pull-secret.json
```

### 2.3 Mirror OpenShift Release Images

```bash
# Mirror OpenShift images
oc adm release mirror -a merged-pull-secret.json \
  --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} \
  --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
  --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE}

# Save image content source policy
oc adm release mirror -a merged-pull-secret.json \
  --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} \
  --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
  --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE} \
  --dry-run > imageContentSourcePolicy.yaml
```

## Step 3: Mirror Red Hat and Certified Operators

### 3.1 Download and Configure opm

```bash
# Download opm tool
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/opm-linux.tar.gz
tar -xzf opm-linux.tar.gz
sudo mv opm /usr/local/bin/

# Verify opm installation
opm version
```

### 3.2 Mirror Red Hat Operators

```bash
# Create operator mirror directory
mkdir -p /opt/operators/redhat

# Mirror Red Hat operators catalog
oc adm catalog mirror registry.redhat.io/redhat/redhat-operator-index:v${OCP_RELEASE%.*} \
  ${LOCAL_REGISTRY}/redhat/redhat-operator-index \
  -a merged-pull-secret.json \
  --index-filter-by-os='linux/amd64' \
  --manifests-only

# Apply the mirroring
oc adm catalog mirror registry.redhat.io/redhat/redhat-operator-index:v${OCP_RELEASE%.*} \
  ${LOCAL_REGISTRY}/redhat/redhat-operator-index \
  -a merged-pull-secret.json \
  --index-filter-by-os='linux/amd64'
```

### 3.3 Mirror Certified Operators

```bash
# Create certified operator mirror directory
mkdir -p /opt/operators/certified

# Mirror certified operators catalog
oc adm catalog mirror registry.redhat.io/redhat/certified-operator-index:v${OCP_RELEASE%.*} \
  ${LOCAL_REGISTRY}/certified/certified-operator-index \
  -a merged-pull-secret.json \
  --index-filter-by-os='linux/amd64' \
  --manifests-only

# Apply the mirroring
oc adm catalog mirror registry.redhat.io/redhat/certified-operator-index:v${OCP_RELEASE%.*} \
  ${LOCAL_REGISTRY}/certified/certified-operator-index \
  -a merged-pull-secret.json \
  --index-filter-by-os='linux/amd64'
```

### 3.4 Mirror Community Operators (Optional)

```bash
# Mirror community operators
oc adm catalog mirror registry.redhat.io/redhat/community-operator-index:v${OCP_RELEASE%.*} \
  ${LOCAL_REGISTRY}/community/community-operator-index \
  -a merged-pull-secret.json \
  --index-filter-by-os='linux/amd64'
```

## Step 4: Agent-Based Installation

### 4.1 Create Installation Directory

```bash
mkdir -p /opt/openshift-install/cluster
cd /opt/openshift-install/cluster
```

### 4.2 Create install-config.yaml

```yaml
cat > install-config.yaml << EOF
apiVersion: v1
baseDomain: example.com
metadata:
  name: ocp-cluster
compute:
- name: worker
  replicas: 3
  architecture: amd64
  hyperthreading: Enabled
controlPlane:
  name: master
  replicas: 3
  architecture: amd64
  hyperthreading: Enabled
networking:
  networkType: OVNKubernetes
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  serviceNetwork:
  - 172.30.0.0/16
  machineNetwork:
  - cidr: 192.168.1.0/24
platform:
  none: {}
pullSecret: |
  $(cat ../merged-pull-secret.json | jq -c .)
sshKey: |
  $(cat ~/.ssh/id_rsa.pub)
imageContentSources:
- mirrors:
  - mirror.example.com:8443/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - mirror.example.com:8443/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
additionalTrustBundle: |
  $(cat /opt/registry/certs/ca-bundle.crt | sed 's/^/  /')
EOF
```

### 4.3 Create agent-config.yaml

```yaml
cat > agent-config.yaml << EOF
apiVersion: v1alpha1
kind: AgentConfig
metadata:
  name: ocp-cluster
rendezvousIP: 192.168.1.10
hosts:
- hostname: master-0
  role: master
  interfaces:
  - name: ens3
    macAddress: 52:54:00:12:34:56
  networkConfig:
    interfaces:
    - name: ens3
      type: ethernet
      state: up
      ipv4:
        enabled: true
        address:
        - ip: 192.168.1.10
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
        next-hop-interface: ens3
- hostname: master-1
  role: master
  interfaces:
  - name: ens3
    macAddress: 52:54:00:12:34:57
  networkConfig:
    interfaces:
    - name: ens3
      type: ethernet
      state: up
      ipv4:
        enabled: true
        address:
        - ip: 192.168.1.11
          prefix-length: 24
        dhcp: false
    dns-resolver:
      config:
        server:
        - 192.168.1.1
    routes:
