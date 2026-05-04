# OpenShift Virtualization GitOps - Baremetal ACM Deployment Assessment

**Assessment Date:** April 28, 2026  
**Repository:** openshift-virtualization-gitops  
**Assessor:** Technical Analysis

---

## Executive Summary

This repository was originally designed assuming an **existing** OpenShift Hub cluster is already running. The recently added Ansible playbooks (`baremetal-acm.yml`, `deploy-acm-mce.yml`, `lvmcluster-config.yml`) successfully fill the critical gap by providing Phase 1 baremetal cluster deployment capabilities.

**Key Findings:**
- ✅ **Hub Deployment Viable**: End-to-end automation from powered-off servers to ACM-managed hub is possible with minor fixes
- ✅ **Managed Cluster Deployment Viable**: GitOps-based cluster provisioning is well-architected and functional
- ⚠️ **Storage Gap**: No default storage class configuration; ACM observability requires object storage (not configured)
- ⚠️ **Hardware Configuration Fragmentation**: Requires editing 8+ locations per hardware type
- ⚠️ **Per-Host File Explosion**: 3 YAML files per baremetal host (9 files for 3-node cluster)
- ⚠️ **Security Gap**: BMC credentials in plaintext, no secrets management

---

## Table of Contents

1. [Repository Architecture Overview](#1-repository-architecture-overview)
2. [Baremetal ACM Hub Cluster Deployment](#2-baremetal-acm-hub-cluster-deployment)
3. [Managed Child Cluster Deployment](#3-managed-child-cluster-deployment)
4. [Gap Analysis](#4-gap-analysis)
5. [Usability Assessment](#5-usability-assessment)
6. [Recommendations](#6-recommendations)
7. [Appendix: File Reference](#7-appendix-file-reference)

---

## 1. Repository Architecture Overview

### 1.1 Original Design Intent

The repository implements a **GitOps-based multi-cluster OpenShift deployment** with:
- **Hub Cluster**: Advanced Cluster Management (ACM), Ansible Automation Platform (AAP), Migration Toolkit for Virtualization (MTV)
- **Managed Clusters**: OpenShift Virtualization, OADP (backup/restore)

### 1.2 GitOps Structure

```
.
├── .bootstrap/                 # Initial ArgoCD deployment manifests
├── components/                 # Reusable operator/configuration components (28 components)
├── groups/                     # Shared configurations for cluster groups
│   └── all/                    # Universal configurations (cert-manager, nmstate, metallb, etc.)
├── clusters/                   # Cluster-specific configurations
│   ├── hub/                    # Hub cluster ArgoCD applications
│   ├── etl4/                   # Example managed cluster 1
│   └── etl6/                   # Example managed cluster 2
├── .helm-charts/               # Helm charts for cluster provisioning
│   ├── argocd-app-of-app/      # App-of-apps pattern
│   ├── bm-cluster-agent-install/ # ACM agent-based cluster creation
│   └── cluster-registration/   # ManagedCluster registration
└── ansible/                    # **NEW** Baremetal deployment automation
    ├── baremetal-acm.yml       # End-to-end hub provisioning
    ├── deploy-acm-mce.yml      # ACM/MCE deployment
    ├── lvmcluster-config.yml   # LVM storage automation
    └── templates/
        ├── install-config.yaml.j2
        └── agent-config.yaml.j2
```

### 1.3 Key Technologies

| Technology | Purpose | Sync Wave |
|------------|---------|-----------|
| **ArgoCD** | GitOps continuous delivery | - |
| **Kustomize** | Configuration templating | - |
| **Helm** | Cluster provisioning charts | - |
| **ACM** | Multi-cluster management | Wave 5 (operator), 15 (instance), 25 (config) |
| **Agent-Based Installer** | Baremetal cluster creation | - |
| **NMState** | Network configuration | Wave 5 (operator), 6 (instance) |
| **MetalLB** | LoadBalancer services | Wave 5 (operator), 6 (config) |

---

## 2. Baremetal ACM Hub Cluster Deployment

### 2.1 What Was Created (New Ansible Playbooks)

#### 2.1.1 `baremetal-acm.yml` - End-to-End Hub Provisioning

**Purpose**: Provision baremetal hub cluster from powered-off servers to GitOps-managed ACM deployment

**Workflow**:
```
Phase 1: Standalone Baremetal Installation
1. Generate install-config.yaml from template
2. Generate agent-config.yaml with host definitions
3. Create Agent-Based Installer ISO
4. Host ISO via HTTP server
5. Mount ISO to BMCs via Redfish VirtualMediaInsert
6. Power cycle hosts to boot from ISO
7. Wait for installation completion (30-60 minutes)

Phase 2: GitOps & ACM Bootstrap
8. Apply GitOps operator subscription
9. Deploy ArgoCD instance with custom plugins
10. Deploy root Application targeting clusters/hub/
11. ArgoCD auto-deploys: ACM operator → MultiClusterHub → assisted-service
```

**Key Variables**:
```yaml
pull_secret: "OpenShift pull secret (JSON string)"
ssh_public_key: "ecdsa-sha2-nistp521 AAAA..."
rendezvous_ip: "10.9.48.100"  # Primary master IP for install coordination
cluster_name: "hub"
bm_hosts:
  - name: hub-master-0
    bmc_address: "10.9.48.100"
    bmc_user: "admin"
    bmc_password: "funkychunkymonkey"
    macAddress: ff:ff:ff:ff:ff:ff
gitops_repo: "https://github.com/<your-org>/openshift-virtualization-gitops.git"
```

#### 2.1.2 `deploy-acm-mce.yml` - ACM/MCE Deployment

**Purpose**: Deploy Advanced Cluster Management or MultiCluster Engine on existing OpenShift cluster

**Features**:
- Auto-detects cluster architecture (x86_64, aarch64, s390x, ppc64le)
- Discovers OCP version and RHCOS bootimage details
- Queries default StorageClass
- Creates AgentServiceConfig with discovered settings
- Enables assisted-service for agent-based cluster provisioning
- Creates ClusterImageSet for discovered OCP version

**Deployment Options**:
```yaml
deploy_target: "acm"  # or "mce"
acm_channel: ""       # Auto-detects default channel if empty
storage_class: ""     # Auto-detects default SC if empty
```

#### 2.1.3 `lvmcluster-config.yml` - LVM Storage Automation

**Purpose**: Automatically discover and configure LVM storage on OpenShift worker nodes

**Features**:
- Uses `oc debug` to inspect each worker node
- Identifies OS disk via `/boot` mount point
- Discovers non-OS disks using `/dev/disk/by-id` paths
- Generates LVMCluster CR with per-node device classes
- Applies configuration to `openshift-storage` namespace

**Prerequisites**:
- LVM Storage Operator already installed from OperatorHub
- Worker nodes have additional drives beyond OS disk

### 2.2 GitOps Components Deployed via ArgoCD

Once the playbooks complete, ArgoCD deploys (from `clusters/hub/values.yaml`):

| Component | Sync Wave | Purpose |
|-----------|-----------|---------|
| **acm-operator** | 5 | ACM operator subscription |
| **acm-instance** | 15 | MultiClusterHub CR |
| **acm-configuration** | 25 | AgentServiceConfig, Provisioning, HiveConfig |
| **gitops-boostrap-policy** | 15 | GitOps propagation to managed clusters |
| **aap-operator** | 5 | Ansible Automation Platform |
| **mtv-operator** | 5 | Migration Toolkit for Virtualization |
| **cert-manager-operator** | 5 | Certificate automation |
| **external-dns-operator** | 5 | Automatic DNS record management |
| **metallb-operator** | 5 | LoadBalancer service provisioning |
| **nmstate-operator** | 5 | Network configuration management |

### 2.3 Current Gaps for Hub Deployment

#### 2.3.1 Storage Class Configuration (CRITICAL)

**Problem**:
- Repository explicitly states "This repo does not setup storage" (`storage.md`)
- LVM playbook creates `topolvm-provisioner` StorageClass but doesn't set it as default
- ACM requires object storage for observability (S3-compatible: MinIO, ODF, NooBaa)
- No storage operator components in `components/` directory

**Impact**:
- PVCs may fail if no default StorageClass exists
- ACM observability component cannot deploy (requires S3 bucket)
- Hub monitoring/metrics unavailable

**Required Annotations**:
```yaml
storageclass.kubernetes.io/is-default-class: "true"           # Cluster default
storageclass.kubevirt.io/is-default-virt-class: "true"       # OpenShift Virtualization default
```

#### 2.3.2 Template Variables Require Manual Configuration

**install-config.yaml.j2 Limitations**:
```yaml
compute:
  - replicas: 0          # Hardcoded: no workers
controlPlane:
  - replicas: 3          # Hardcoded: 3 masters
networking:
  clusterNetwork:
    - cidr: 10.128.0.0/14    # Fixed CIDR
  serviceNetwork:
    - 172.30.0.0/16          # Fixed CIDR
platform:
  none: {}               # No cloud provider integration
```

**agent-config.yaml.j2 Issues**:
- **MAC Address Field Mismatch**: Playbook uses `macAddress` but template expects `mac_address` (line 11)
- **Interface Name Hardcoded**: `eth0` on line 10 (should be `ens192`, `eno1`, etc. depending on hardware)
- **DHCP Only**: No static IP configuration (only comment suggesting NMState)
- **Single Interface**: No multi-NIC support

#### 2.3.3 Hardware-Specific Configuration Fragmentation

**Requires editing in 8+ locations** for different hardware:

| Configuration Item | File Location | Line(s) |
|-------------------|---------------|---------|
| BMC IP addresses | `ansible/baremetal-acm.yml` | 19 |
| BMC credentials | `ansible/baremetal-acm.yml` | 20-21 |
| MAC addresses | `ansible/baremetal-acm.yml` | 23 |
| MAC addresses (duplicate) | `ansible/templates/agent-config.yaml.j2` | 11 |
| Interface names | `ansible/templates/agent-config.yaml.j2` | 10 |
| Rendezvous IP | `ansible/baremetal-acm.yml` | 10 |
| ISO hosting URL | `ansible/baremetal-acm.yml` | 76 |
| Cluster pod CIDR | `ansible/templates/install-config.yaml.j2` | 17 |
| Service CIDR | `ansible/templates/install-config.yaml.j2` | 20 |
| Base domain | `ansible/templates/install-config.yaml.j2` | 2 |

**Example Problem**: Deploying on Dell R640 vs HPE DL380 requires:
- Different interface names (`eno1` vs `eth0`)
- Possibly different BMC addresses schemes
- Different disk layouts (affects LVM playbook)
- No abstraction layer to handle these differences

#### 2.3.4 No Validation or Pre-Flight Checks

**Missing Validations**:
- ✗ `openshift-install` binary exists and is executable
- ✗ Pull secret is valid JSON format
- ✗ SSH key is valid public key format
- ✗ BMC addresses are reachable (ping/curl test)
- ✗ HTTP server is running on control node (for ISO hosting)
- ✗ DNS entries exist for cluster ingress/API
- ✗ DHCP server configured (if using DHCP for rendezvous IP)

**Impact**: Failures discovered late in process (e.g., after 30 minutes of installation)

#### 2.3.5 GitOps Bootstrap Assumptions

**Hardcoded Dependencies**:
- Playbook assumes `.bootstrap/` files exist at specific relative path
- `gitops_repo` variable must be manually updated to point to your fork
- No mechanism to customize target cluster folder (always `clusters/hub/`)
- Environment variable substitution requires specific naming (`${cluster_name}`, `${cluster_base_domain}`)

**ArgoCD Plugin Requirements**:
- Requires custom `setenv-cmp-plugin` for environment variable substitution
- Requires RHACM PolicyGenerator plugin for policy propagation
- Both configured in `.bootstrap/argocd.yaml` (lines 475-520)

---

## 3. Managed Child Cluster Deployment

### 3.1 Architecture (Works Well)

The GitOps approach for managed clusters is **well-designed** and follows ACM Zero Touch Provisioning (ZTP) patterns.

#### 3.1.1 Deployment Workflow

```
1. Hub Admin: Create BareMetalHost CRs with BMC details
   ├── Example: clusters/hub/overlays/cluster-etl4/x240m5-11-baremetal-host.yaml
   └── Defines: BMC address, MAC address, credentials

2. Hub Admin: Create NMStateConfig CRs for network configuration (optional)
   ├── Example: clusters/hub/overlays/cluster-etl4/x240m5-11-nmstate-config.yaml
   └── Defines: Static IPs, VLANs, bonding, routes

3. Hub Admin: Deploy bm-cluster-agent-install Helm chart
   ├── Chart creates: InfraEnv, AgentClusterInstall, ClusterDeployment
   ├── Parameters: VIPs, CIDRs, imageSet, SSH key, worker count
   └── Example: clusters/hub/overlays/cluster-etl4/kustomization.yaml (lines 20-35)

4. ACM Operator: Creates discovery ISO with agent
   └── Hosts boot from ISO, agents report back to ACM

5. ACM Operator: Matches agents to BareMetalHosts via MAC
   └── Waits for all hosts to be discovered

6. ACM Operator: Validates cluster configuration
   └── Checks: Network connectivity, disk space, CPU/memory

7. ACM Operator: Installs OpenShift via agent-based installer
   └── 30-60 minute installation

8. Hub Admin: Deploy cluster-registration Helm chart
   ├── Chart creates: ManagedCluster, KlusterletAddonConfig
   └── Cluster appears in ACM console

9. ArgoCD: Applies day-2 configuration from clusters/<name>/
   └── Example: clusters/etl4/ (storage, networking, operators)
```

#### 3.1.2 Example Cluster Definition

**clusters/hub/overlays/cluster-etl4/kustomization.yaml**:
```yaml
resources:
  - namespace.yaml
  - x240m5-11-baremetal-host.yaml     # Host 1
  - x240m5-12-baremetal-host.yaml     # Host 2
  - x240m5-13-baremetal-host.yaml     # Host 3
  - x240m5-11-nmstate-config.yaml     # Host 1 networking
  - x240m5-12-nmstate-config.yaml     # Host 2 networking
  - x240m5-13-nmstate-config.yaml     # Host 3 networking
  - x240m5-11-fqdn.yaml               # Host 1 DNS
  - x240m5-12-fqdn.yaml               # Host 2 DNS
  - x240m5-13-fqdn.yaml               # Host 3 DNS

helmCharts:
  - name: bm-cluster-agent-install
    releaseName: etl4
    namespace: etl4
    valuesInline:
      clusterSet: default
      workerNumber: 0
      imageSet: img4.17.2-x86-64-appsub
      mastersSchedulable: true
      sshKey: ssh-rsa AAAA...
      networking:
        podCIDR: 10.136.0.0/14
        serviceCIDR: 172.32.0.0/16
        ingressVIP: "10.9.51.154"
        apiVIP: "10.9.51.155"

  - name: cluster-registration
    releaseName: etl4
    namespace: etl4
```

**Registered in clusters/hub/values.yaml**:
```yaml
applications:
  etl4:
    annotations:
      argocd.argoproj.io/sync-wave: '25'
    destination:
      namespace: etl4
    source:
      path: clusters/hub/overlays/cluster-etl4
```

### 3.2 Current Gaps for Managed Cluster Deployment

#### 3.2.1 Per-Host Configuration Explosion (CRITICAL)

**Problem**: For **each baremetal host**, must create **3 separate YAML files**:

```
x240m5-11-baremetal-host.yaml       # BMC, MAC, credentials (20 lines)
x240m5-11-nmstate-config.yaml       # Static IP, VLAN, bonding (50-100 lines)
x240m5-11-fqdn.yaml                 # external-dns annotations (15 lines)
```

**Scale Impact**:
| Cluster Size | Files Required | Manual Edits |
|--------------|----------------|--------------|
| 3 nodes | 9 files | 270+ lines |
| 5 nodes | 15 files | 450+ lines |
| 10 nodes | 30 files | 900+ lines |

**No Templating**: `kustomization.yaml` lists each file explicitly, no loop/generator mechanism

**Error-Prone**: 
- MAC address typos
- BMC credential mismatches
- NMState YAML syntax errors (difficult to debug)
- Duplicate IP assignments

#### 3.2.2 BMC Credentials Management (SECURITY)

**Current Approach**:
```yaml
# Plain-text Secret referenced by all BareMetalHosts
apiVersion: v1
kind: Secret
metadata:
  name: bmc-credentials
  namespace: etl4
type: Opaque
data:
  username: YWRtaW4=      # base64("admin")
  password: ZnVua3k=      # base64("funky")
```

**Problems**:
- ✗ Credentials committed to Git repository
- ✗ No encryption at rest (beyond base64 encoding)
- ✗ No credential rotation mechanism
- ✗ Shared credentials across all hosts (security boundary violation)

**Compliance Violations**:
- PCI-DSS: Requires encrypted credential storage
- SOC 2: Requires access controls on privileged credentials
- FedRAMP: Requires FIPS 140-2 validated encryption

**Missing Integrations**:
- No Sealed Secrets controller
- No External Secrets Operator
- No HashiCorp Vault integration
- No cert-manager for credential lifecycle

#### 3.2.3 Network Configuration Complexity

**NMStateConfig Example** (simplified):
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: x240m5-11-nmstate
spec:
  nodeSelector:
    kubernetes.io/hostname: x240m5-11.etl4.ocp.rht-labs.com
  desiredState:
    interfaces:
      - name: bond0
        type: bond
        state: up
        ipv4:
          enabled: true
          address:
            - ip: 10.9.51.11
              prefix-length: 24
          dhcp: false
        link-aggregation:
          mode: 802.3ad
          options:
            miimon: '100'
          port:
            - ens192
            - ens224
      - name: bond0.100      # VLAN 100 for storage
        type: vlan
        state: up
        vlan:
          base-iface: bond0
          id: 100
        ipv4:
          enabled: true
          address:
            - ip: 10.9.100.11
              prefix-length: 24
    routes:
      config:
        - destination: 0.0.0.0/0
          next-hop-interface: bond0
          next-hop-address: 10.9.51.1
```

**Challenges**:
- 50-100 lines of YAML per host
- No validation tool to test before deployment
- Errors discovered during host provisioning (30+ minutes wasted)
- No examples for:
  - Multi-NIC configurations (management + storage + VM networks)
  - SR-IOV configurations
  - OVS bridge configurations
  - Mixed bond + single interface setups

**Reference Gap**: `networking.md` mentions examples but provides minimal guidance:
```
For the networks that are configured at day2 look at the 
nmstate-configuration overlay as an example.
```

#### 3.2.4 ImageSet Management

**Current Approach**: Hardcoded imageSet names
```yaml
imageSet: img4.17.2-x86-64-appsub    # Where is this defined?
```

**Problems**:
- No documented process for creating ClusterImageSets
- No version pinning strategy (how to prevent auto-upgrades?)
- No multi-version support (QA on 4.16, Prod on 4.17)
- No promotion workflow (Dev → Test → Prod)

**Missing Documentation**:
- How to create imageSet for new OCP version
- Where ClusterImageSet CRs are stored (not in repo)
- Relationship to `deploy-acm-mce.yml` auto-created imageSet

#### 3.2.5 Day-2 Configuration Gaps

**Managed Cluster Structure** (clusters/etl4/):
```
clusters/etl4/
├── kustomization.yaml              # Imports groups/all
├── values.yaml                     # Cluster-specific apps
└── overlays/
    ├── metallb-configuration/      # IP pools
    ├── nmstate-configuration/      # NNCPs, NADs
    └── openshift-config/           # ClusterVersion, OAuth
```

**Present**:
- ✓ MetalLB IP pool configuration
- ✓ NMState day-2 networking (post-install VLANs)
- ✓ ClusterVersion pinning
- ✓ OAuth integration

**Missing** (despite architecture diagram showing them):
- ✗ Storage operator deployment (ODF, LVM, NFS)
- ✗ OpenShift Virtualization operator (repo name implies this is core!)
- ✗ OADP (backup/restore) configuration
- ✗ Monitoring/alerting customization
- ✗ Node autoscaler configuration
- ✗ Machine health check configuration

**Example Missing Component**: OpenShift Virtualization
```
Expected:
  components/openshift-virtualization-operator/    # ✓ EXISTS
  components/openshift-virtualization-instance/    # ✓ EXISTS
  clusters/etl4/values.yaml:
    openshift-virtualization-operator:             # ✗ MISSING
    openshift-virtualization-instance:             # ✗ MISSING

Result: Managed clusters don't have virtualization enabled!
```

---

## 4. Gap Analysis

### 4.1 Critical Gaps (Blocking Production Use)

| # | Gap | Impact | Affected Component |
|---|-----|--------|-------------------|
| 1 | **No default StorageClass configured** | PVC failures, ACM observability unavailable | Hub cluster |
| 2 | **No object storage (S3) for ACM** | ACM observability component fails | Hub cluster |
| 3 | **BMC credentials in plaintext** | Security/compliance violations (PCI, SOC2, FedRAMP) | All clusters |
| 4 | **Per-host file explosion** | 3N files for N hosts; error-prone, unscalable | Managed clusters |
| 5 | **No hardware abstraction** | Must edit 8+ locations per hardware type | Hub cluster |
| 6 | **MAC address field name mismatch** | Playbook fails: `macAddress` vs `mac_address` | Hub cluster |

### 4.2 High Priority Gaps (Usability Issues)

| # | Gap | Impact | Workaround |
|---|-----|--------|------------|
| 7 | **Network templates incomplete** | Manual YAML for bonds, VLANs, static IPs | Copy/modify examples |
| 8 | **No pre-flight validation** | Late failure discovery (after 30+ min) | Manual verification |
| 9 | **ISO hosting mechanism unclear** | httpd setup not documented | Start httpd manually |
| 10 | **GitOps repo URL hardcoded** | Can't easily fork/customize | Edit playbook vars |
| 11 | **No NMState validation tool** | Syntax errors discovered during provisioning | Trial and error |
| 12 | **ImageSet management undocumented** | Unclear how to update OCP versions | Use deploy-acm-mce.yml |

### 4.3 Medium Priority Gaps (Nice-to-Have)

| # | Gap | Impact | Workaround |
|---|-----|--------|------------|
| 13 | **No cluster lifecycle automation** | Manual upgrades, scaling, decommissioning | Manual `oc` commands |
| 14 | **No disaster recovery** | Hub loss = total system loss | External backups |
| 15 | **No federation support** | Single hub limitation | N/A |
| 16 | **No ArgoCD notification config** | Deployment failures not alerted | Check ArgoCD UI |
| 17 | **No cost tracking/metering** | No resource utilization visibility | External monitoring |
| 18 | **No GPU/accelerator support** | Can't deploy AI/ML workloads | Manual config |

### 4.4 Documentation Gaps

| # | Gap | Impact |
|---|-----|--------|
| 19 | **No hub deployment guide** | Users don't know Ansible playbooks exist |
| 20 | **No hardware compatibility matrix** | Unknown supported hardware |
| 21 | **No troubleshooting guide** | Installation failures difficult to debug |
| 22 | **No upgrade procedures** | Cluster version management unclear |
| 23 | **No security hardening guide** | CIS benchmarks not addressed |
| 24 | **No disaster recovery runbook** | Hub recovery procedures unknown |

---

## 5. Usability Assessment

### 5.1 Can You Deploy a Hub Cluster Today?

**Answer: YES, with caveats**

#### 5.1.1 Deployment Procedure

```bash
# Step 1: Prerequisites
# - Baremetal servers with IPMI/Redfish BMCs
# - DHCP server configured for rendezvous IP
# - DNS entries for *.apps.<cluster>.<domain> and api.<cluster>.<domain>
# - HTTP server running on Ansible control node
# - openshift-install and oc binaries in PATH

# Step 2: Fix MAC address field name mismatch
# Edit ansible/baremetal-acm.yml line 23:
#   macAddress → mac_address
# OR edit ansible/templates/agent-config.yaml.j2 line 11:
#   mac_address → macAddress

# Step 3: Customize variables in ansible/baremetal-acm.yml
# - pull_secret (get from console.redhat.com)
# - ssh_public_key
# - rendezvous_ip
# - bm_hosts (BMC addresses, credentials, MACs)
# - gitops_repo (your fork of this repository)

# Step 4: Customize templates
# Edit ansible/templates/install-config.yaml.j2:
# - baseDomain (line 2)
# - networking CIDRs if needed (lines 17-20)
# Edit ansible/templates/agent-config.yaml.j2:
# - Interface name if not eth0 (line 10)

# Step 5: Start HTTP server for ISO hosting
sudo systemctl start httpd
# Update baremetal-acm.yml line 76 with your control node IP

# Step 6: Run Phase 1 - Base cluster installation
ansible-playbook ansible/baremetal-acm.yml

# Expected runtime: 45-60 minutes
# Watch for:
# - ISO creation success
# - BMC mount success (Redfish errors if credentials wrong)
# - Installation progress (tails install logs)

# Step 7: Setup storage (LVM)
ansible-playbook ansible/lvmcluster-config.yml

# Step 8: Set default StorageClass
oc patch storageclass topolvm-provisioner \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Step 9: Verify GitOps deployment
oc get applications -n openshift-gitops
# Should show: acm-operator, acm-instance, acm-configuration, etc.

# Step 10: Wait for ACM deployment
oc get multiclusterhub -n open-cluster-management -w
# Wait for STATUS: Running (10-20 minutes)

# Step 11: Access ACM console
oc get route multicloud-console -n open-cluster-management \
  -o jsonpath='{.spec.host}'
```

#### 5.1.2 Required Pre-Work

| Task | Estimated Time | Complexity |
|------|----------------|------------|
| Fix MAC address field mismatch | 5 minutes | Low |
| Gather hardware details (BMC IPs, MACs) | 30 minutes | Medium |
| Setup DNS records | 30 minutes | Medium |
| Configure DHCP reservations | 20 minutes | Low |
| Install/configure httpd | 15 minutes | Low |
| Customize playbook variables | 45 minutes | Medium |
| Test BMC connectivity | 15 minutes | Low |
| **Total First-Time Deployment** | **3-4 hours** | Medium |

#### 5.1.3 Post-Deployment Manual Steps

| Task | Why Needed | Estimated Time |
|------|------------|----------------|
| Set default StorageClass | LVM playbook doesn't set annotation | 2 minutes |
| Deploy object storage (MinIO/ODF) | ACM observability requires S3 | 1-2 hours |
| Comment out `acm-observability` app | Fails without object storage | 5 minutes |
| Create BMC credentials secret | Required by BareMetalHost CRs | 10 minutes |
| Verify all ArgoCD apps healthy | Ensure GitOps working | 15 minutes |

### 5.2 Can You Deploy Managed Clusters Today?

**Answer: YES, but requires significant manual work**

#### 5.2.1 Deployment Procedure (Per Cluster)

```bash
# Step 1: Create cluster namespace and overlay directory
mkdir -p clusters/hub/overlays/cluster-prod1
cat > clusters/hub/overlays/cluster-prod1/namespace.yaml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: prod1
EOF

# Step 2: Create BareMetalHost YAML for each host (manual copy/paste/edit)
# Must create: prod1-master-0-baremetal-host.yaml, prod1-master-1-..., etc.
# Example: 3 nodes = 3 files × 20 lines = 60 lines of YAML

# Step 3: Create NMStateConfig YAML for each host (if static IPs)
# Must create: prod1-master-0-nmstate-config.yaml, prod1-master-1-..., etc.
# Example: 3 nodes × 80 lines = 240 lines of complex YAML

# Step 4: Create FQDN/DNS YAML for each host (if using external-dns)
# Must create: prod1-master-0-fqdn.yaml, prod1-master-1-..., etc.
# Example: 3 nodes × 15 lines = 45 lines of YAML

# Step 5: Create kustomization.yaml listing all files + helm chart
cat > clusters/hub/overlays/cluster-prod1/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - prod1-master-0-baremetal-host.yaml
  - prod1-master-1-baremetal-host.yaml
  - prod1-master-2-baremetal-host.yaml
  - prod1-master-0-nmstate-config.yaml
  - prod1-master-1-nmstate-config.yaml
  - prod1-master-2-nmstate-config.yaml
  - prod1-master-0-fqdn.yaml
  - prod1-master-1-fqdn.yaml
  - prod1-master-2-fqdn.yaml

helmGlobals:
  chartHome: ../../../../.helm-charts

helmCharts:
  - name: bm-cluster-agent-install
    releaseName: prod1
    namespace: prod1
    valuesInline:
      clusterSet: default
      workerNumber: 0
      imageSet: img4.17.2-x86-64-appsub
      mastersSchedulable: true
      sshKey: ssh-rsa AAAA...
      networking:
        podCIDR: 10.140.0.0/14
        serviceCIDR: 172.33.0.0/16
        ingressVIP: "10.9.60.100"
        apiVIP: "10.9.60.101"

  - name: cluster-registration
    releaseName: prod1
    namespace: prod1
EOF

# Step 6: Register cluster in clusters/hub/values.yaml
# Add to applications: section:
  prod1:
    annotations:
      argocd.argoproj.io/sync-wave: '25'
    destination:
      namespace: prod1
    source:
      path: clusters/hub/overlays/cluster-prod1

# Step 7: Commit and push to Git
git add clusters/hub/overlays/cluster-prod1/
git add clusters/hub/values.yaml
git commit -m "Add prod1 cluster definition"
git push

# Step 8: ArgoCD syncs, cluster deploys (30-60 minutes)
# Monitor in ACM console

# Step 9: Create day-2 configuration directory
mkdir -p clusters/prod1/overlays/openshift-config
# Add storage, networking, virtualization operators...
```

#### 5.2.2 Time Estimates

| Cluster Size | Files to Create | Manual Editing | Total Time (First Cluster) |
|--------------|-----------------|----------------|----------------------------|
| 3 nodes | 9 files | ~350 lines YAML | 6-8 hours |
| 5 nodes | 15 files | ~600 lines YAML | 8-10 hours |
| 10 nodes | 30 files | ~1200 lines YAML | 12-16 hours |

**Subsequent Clusters** (once familiar with pattern): 50% time reduction

### 5.3 Real-World Deployment Scenarios

#### Scenario A: Small Lab (3-node hub, 2 managed 3-node clusters)

**Timeline**:
- Hub deployment: 4 hours (first time)
- Storage setup: 1 hour
- First managed cluster: 8 hours
- Second managed cluster: 4 hours (copy/paste from first)
- **Total: ~17 hours**

**Blockers**:
- MAC address mismatch (1 hour debugging)
- NMState syntax error (2 hours trial-and-error)
- Missing default StorageClass (30 min debugging)

#### Scenario B: Production (3-node hub, 10 managed 5-node clusters)

**Timeline**:
- Hub deployment: 6 hours (with hardening)
- Object storage (ODF): 3 hours
- First managed cluster: 10 hours (includes process documentation)
- Remaining 9 clusters: 5 hours each = 45 hours
- Day-2 config (storage, virtualization): 20 hours
- **Total: ~84 hours** (2 weeks of work)

**Major Gaps**:
- Per-host file explosion (9 files × 10 clusters × 5 nodes = 450 files!)
- No secrets management (compliance audit failure)
- No backup/restore (need OADP configuration)

---

## 6. Recommendations

### 6.1 Immediate (Fix to be Functional)

| Priority | Recommendation | Effort | Impact |
|----------|----------------|--------|--------|
| 🔴 **P0** | **Fix MAC address field name mismatch** | 5 min | Hub deployment works |
| 🔴 **P0** | **Add default StorageClass annotation to lvmcluster playbook** | 10 min | Prevents PVC failures |
| 🔴 **P0** | **Create hardware profile vars files** | 2 hours | Eliminates 8+ edit locations |
| 🔴 **P0** | **Document ISO hosting requirements** | 1 hour | Prevents httpd confusion |

#### 6.1.1 Implementation Details

**Fix 1: MAC Address Field Mismatch**
```yaml
# Option A: Edit ansible/baremetal-acm.yml line 23
bm_hosts:
  - name: hub-master-0
    bmc_address: "10.9.48.100"
    bmc_user: "admin"
    bmc_password: "funkychunkymonkey"
    mac_address: "ff:ff:ff:ff:ff:ff"  # Changed from macAddress

# Option B: Edit ansible/templates/agent-config.yaml.j2 line 11
interfaces:
  - name: eth0
    macAddress: "{{ host.macAddress }}"  # Changed from mac_address
```

**Fix 2: StorageClass Default Annotation**
```yaml
# Add to ansible/lvmcluster-config.yml after line 106
- name: Set LVM StorageClass as default
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: storage.k8s.io/v1
      kind: StorageClass
      metadata:
        name: topolvm-provisioner
        annotations:
          storageclass.kubernetes.io/is-default-class: "true"
```

**Fix 3: Hardware Profiles**
```bash
# Create ansible/hardware_profiles/dell-r640.yml
---
interface_name: eno1
bmc_type: idrac
boot_mode: uefi
# ... other Dell-specific settings

# Create ansible/hardware_profiles/hpe-dl380.yml
---
interface_name: eth0
bmc_type: ilo
boot_mode: legacy
# ... other HPE-specific settings

# Update playbook to load profile:
- include_vars: "hardware_profiles/{{ hardware_type }}.yml"
```

### 6.2 Short-Term (Improve Usability)

| Priority | Recommendation | Effort | Impact |
|----------|----------------|--------|--------|
| 🟡 **P1** | **Create BMH/NMState generator playbook** | 1 week | Eliminates per-host file explosion |
| 🟡 **P1** | **Add pre-flight validation to baremetal-acm.yml** | 3 days | Catch errors early |
| 🟡 **P1** | **Integrate Sealed Secrets** | 1 week | Addresses security gap |
| 🟡 **P1** | **Create NMState examples library** | 1 week | Reduces networking errors |

#### 6.2.1 BMH/NMState Generator Design

```yaml
# ansible/generate-cluster-yamls.yml
---
- name: Generate cluster manifests from inventory
  hosts: localhost
  vars:
    cluster_inventory: inventory/prod1.csv  # CSV with MAC,BMC,IP columns
  tasks:
    - name: Read cluster inventory CSV
      read_csv:
        path: "{{ cluster_inventory }}"
      register: hosts_csv

    - name: Generate BareMetalHost YAML for each node
      template:
        src: templates/baremetal-host.yaml.j2
        dest: "clusters/hub/overlays/cluster-{{ cluster_name }}/{{ item.hostname }}-baremetal-host.yaml"
      loop: "{{ hosts_csv.list }}"

    - name: Generate NMStateConfig YAML for each node
      template:
        src: templates/nmstate-config.yaml.j2
        dest: "clusters/hub/overlays/cluster-{{ cluster_name }}/{{ item.hostname }}-nmstate-config.yaml"
      loop: "{{ hosts_csv.list }}"
      when: item.static_ip is defined

    - name: Generate kustomization.yaml with all resources
      template:
        src: templates/kustomization.yaml.j2
        dest: "clusters/hub/overlays/cluster-{{ cluster_name }}/kustomization.yaml"
```

**Example Inventory CSV**:
```csv
hostname,bmc_address,bmc_user,bmc_password,mac_address,static_ip,gateway,interface
prod1-master-0,10.9.60.10,admin,secret,aa:bb:cc:dd:ee:01,10.9.60.100,10.9.60.1,eno1
prod1-master-1,10.9.60.11,admin,secret,aa:bb:cc:dd:ee:02,10.9.60.101,10.9.60.1,eno1
prod1-master-2,10.9.60.12,admin,secret,aa:bb:cc:dd:ee:03,10.9.60.102,10.9.60.1,eno1
```

**Result**: 3-node cluster = 1 CSV file instead of 9 YAML files

### 6.3 Medium-Term (Production-Ready)

| Priority | Recommendation | Effort | Impact |
|----------|----------------|--------|--------|
| 🟢 **P2** | **Add object storage component (MinIO/ODF)** | 2 weeks | Enables ACM observability |
| 🟢 **P2** | **Create cluster upgrade automation** | 2 weeks | Simplifies version management |
| 🟢 **P2** | **Add OADP backup/restore** | 1 week | Disaster recovery capability |
| 🟢 **P2** | **Add OpenShift Virtualization to managed clusters** | 1 week | Aligns with repo purpose |
| 🟢 **P2** | **Create comprehensive troubleshooting guide** | 1 week | Reduces support burden |

#### 6.3.1 Object Storage Component Design

**Create components/minio-operator/**:
```yaml
# components/minio-operator/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: minio-operator
resources:
  - namespace.yaml
  - subscription.yaml

# components/minio-operator/subscription.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: minio-operator
  namespace: minio-operator
spec:
  channel: stable
  name: minio-operator
  source: certified-operators
  sourceNamespace: openshift-marketplace
```

**Create components/minio-instance/**:
```yaml
# For ACM observability S3 bucket
apiVersion: minio.min.io/v2
kind: Tenant
metadata:
  name: acm-observability
  namespace: open-cluster-management
spec:
  pools:
    - servers: 1
      volumesPerServer: 4
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 10Gi
          storageClassName: topolvm-provisioner
```

**Update clusters/hub/values.yaml**:
```yaml
applications:
  minio-operator:
    annotations:
      argocd.argoproj.io/sync-wave: '5'
    source:
      path: components/minio-operator

  minio-instance:
    annotations:
      argocd.argoproj.io/sync-wave: '15'
    destination:
      namespace: open-cluster-management
    source:
      path: components/minio-instance

  acm-observability:
    annotations:
      argocd.argoproj.io/sync-wave: '25'  # Now works!
    source:
      path: components/acm-observability
```

### 6.4 Long-Term (Enterprise Features)

| Priority | Recommendation | Effort | Impact |
|----------|----------------|--------|--------|
| 🔵 **P3** | **Hub federation/disaster recovery** | 1 month | Multi-datacenter resilience |
| 🔵 **P3** | **Cost tracking/metering** | 2 weeks | Financial visibility |
| 🔵 **P3** | **GPU/accelerator support** | 1 month | AI/ML workload enablement |
| 🔵 **P3** | **Policy compliance automation** | 2 weeks | CIS benchmark enforcement |
| 🔵 **P3** | **Cluster lifecycle API** | 1 month | Self-service cluster provisioning |

### 6.5 Documentation Improvements

| Document | Content | Priority |
|----------|---------|----------|
| **QUICKSTART.md** | Step-by-step hub deployment guide | P0 |
| **HARDWARE_COMPATIBILITY.md** | Tested hardware, BIOS settings, network configs | P1 |
| **TROUBLESHOOTING.md** | Common errors, debug procedures, log locations | P1 |
| **UPGRADE_GUIDE.md** | OCP version upgrade procedures | P2 |
| **DISASTER_RECOVERY.md** | Hub backup/restore runbook | P2 |
| **SECURITY_HARDENING.md** | CIS benchmarks, compliance procedures | P2 |

---

## 7. Appendix: File Reference

### 7.1 Key Repository Files

| Path | Purpose | Lines | Criticality |
|------|---------|-------|-------------|
| **ansible/baremetal-acm.yml** | End-to-end hub provisioning | 152 | Critical |
| **ansible/deploy-acm-mce.yml** | ACM/MCE deployment automation | 364 | High |
| **ansible/lvmcluster-config.yml** | LVM storage automation | 112 | High |
| **ansible/templates/install-config.yaml.j2** | OpenShift install config template | 25 | Critical |
| **ansible/templates/agent-config.yaml.j2** | Agent-based installer host config | 14 | Critical |
| **.bootstrap/argocd.yaml** | ArgoCD instance with plugins | 625 | Critical |
| **.bootstrap/root-application.yaml** | Root app-of-apps | 55 | Critical |
| **clusters/hub/values.yaml** | Hub cluster ArgoCD applications | 142 | Critical |
| **clusters/hub/kustomization.yaml** | Hub cluster kustomize config | 42 | High |
| **components/acm-operator/** | ACM operator deployment | - | High |
| **components/acm-instance/** | MultiClusterHub CR | - | High |
| **components/acm-configuration/** | AgentServiceConfig, Provisioning | - | High |
| **.helm-charts/bm-cluster-agent-install/** | Agent-based cluster creation | - | Critical |
| **.helm-charts/cluster-registration/** | ManagedCluster registration | - | High |

### 7.2 Environment Variables (ArgoCD Plugin)

| Variable | Source | Example | Usage |
|----------|--------|---------|-------|
| `CLUSTER_NAME` | Playbook: `cluster_name` | `hub` | Cluster identification |
| `CLUSTER_BASE_DOMAIN` | Auto-detected from ingress | `hub.ocp.rht-labs.com` | DNS configuration |
| `PLATFORM_BASE_DOMAIN` | Derived from cluster domain | `ocp.rht-labs.com` | Parent DNS zone |
| `HUB_BASE_DOMAIN` | Same as cluster base domain | `hub.ocp.rht-labs.com` | Hub cluster reference |
| `INFRA_GITOPS_REPO` | Playbook: `gitops_repo` | `https://github.com/...` | Source repo URL |

**Substitution Mechanism**:
- Defined in `.bootstrap/argocd.yaml` lines 618-624
- Applied via `setenv-cmp-plugin` ArgoCD sidecar (lines 494-512)
- Used in manifests: `host: "console.apps.${CLUSTER_BASE_DOMAIN}"`

### 7.3 ArgoCD Sync Waves

| Wave | Component Type | Examples |
|------|----------------|----------|
| **5** | Operators | acm-operator, nmstate-operator, metallb-operator, cert-manager-operator |
| **6** | Operator configurations | nmstate-instance, metallb-configuration, cert-manager-configuration |
| **15** | Operator instances | acm-instance, gitops-bootstrap-policy, aap-configuration, mtv-configuration |
| **25** | Complex configurations | acm-configuration, acm-observability, cluster-etl4, cluster-etl6 |

**Rationale**: Ensures operators are ready before deploying instances, instances ready before complex configs

### 7.4 Required External Dependencies

| Dependency | Purpose | Installation |
|------------|---------|--------------|
| **openshift-install** | Generate Agent ISO | [Download from console.redhat.com](https://console.redhat.com/openshift/downloads) |
| **oc** | OpenShift CLI | Included with openshift-install |
| **ansible** | Automation framework | `dnf install ansible-core` |
| **kubernetes.core** | Ansible k8s modules | `ansible-galaxy collection install kubernetes.core` |
| **community.general** | Redfish modules | `ansible-galaxy collection install community.general` |
| **httpd** | ISO hosting | `dnf install httpd && systemctl start httpd` |

### 7.5 Network Requirements

| Purpose | Ports | Protocol | Direction |
|---------|-------|----------|-----------|
| **BMC Management** | 623 (IPMI), 443 (Redfish) | TCP/UDP | Ansible → BMC |
| **ISO Download** | 80/443 | HTTP/HTTPS | Nodes → Ansible |
| **OpenShift API** | 6443 | HTTPS | Admin → API VIP |
| **OpenShift Ingress** | 80, 443 | HTTP/HTTPS | Users → Ingress VIP |
| **DHCP** | 67, 68 | UDP | Nodes ↔ DHCP Server |
| **DNS** | 53 | UDP/TCP | Nodes → DNS Server |

### 7.6 DNS Record Requirements

| Record Type | Name | Value | Purpose |
|-------------|------|-------|---------|
| **A** | `api.<cluster>.<domain>` | API VIP | Kubernetes API access |
| **A** | `*.apps.<cluster>.<domain>` | Ingress VIP | Application routes |
| **A** | `<hostname>.<cluster>.<domain>` | Node IP | Per-node resolution (optional) |
| **PTR** | Reverse for API VIP | `api.<cluster>.<domain>` | Reverse lookup (optional) |

**Example for cluster "hub" in domain "ocp.rht-labs.com"**:
```dns
api.hub.ocp.rht-labs.com.     IN  A     10.9.51.155
*.apps.hub.ocp.rht-labs.com.  IN  A     10.9.51.154
```

---

## Conclusion

This codebase provides a **solid foundation** for GitOps-based multi-cluster OpenShift management with ACM. The recently added Ansible playbooks successfully bridge the gap from baremetal hardware to a fully-managed hub cluster.

**Key Strengths**:
- Well-architected ArgoCD app-of-apps pattern
- Comprehensive operator coverage (ACM, AAP, MTV, cert-manager, etc.)
- Environment variable substitution for cluster-specific configuration
- Agent-based cluster provisioning via ACM (ZTP-style)

**Critical Path to Production**:
1. Fix MAC address field mismatch (5 minutes)
2. Add storage class default annotation (10 minutes)
3. Create hardware profile abstraction (2 hours)
4. Implement BMH/NMState generator (1 week)
5. Integrate secrets management (1 week)
6. Deploy object storage for ACM observability (2 weeks)

**Timeline Estimate**:
- **Proof-of-Concept**: 1 week (with critical fixes)
- **Pilot (3 clusters)**: 1 month (with short-term improvements)
- **Production (10+ clusters)**: 2-3 months (with medium-term features)

**Recommendation**: This codebase is **viable for immediate use** with the critical fixes applied. For production deployment, prioritize the short-term improvements (BMH generator, secrets management) to achieve operational efficiency and security compliance.

---

**Document Version:** 1.0  
**Last Updated:** 2026-04-28  
**Maintained By:** Infrastructure Team
