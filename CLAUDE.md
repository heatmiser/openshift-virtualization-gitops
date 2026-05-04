# Project: OpenShift ACM Baremetal Hub & Managed Cluster Deployment

## Project Goals

This repository provides **automated, templateable deployment** of:

1. **ACM Hub Cluster**: 3-node compact OpenShift cluster on baremetal with Advanced Cluster Management (ACM)
2. **Managed Child Clusters**: Additional OpenShift clusters provisioned and managed via ACM Hub using GitOps

### Primary Use Cases
- Baremetal OpenShift installations using Agent-Based Installer (ABI)
- GitOps-based cluster lifecycle management via ArgoCD
- Multi-cluster management with Red Hat ACM
- Templateable configurations for varying hardware/network environments

### Out of Scope
- Cloud provider deployments (AWS, Azure, GCP)
- Virtualized environments (unless using baremetal KVM hosts)
- Features unrelated to ACM hub or managed cluster provisioning

**Note**: If existing code doesn't align with these goals, rewrite/removal is acceptable.

---

## Architecture Overview

### Phase 1: Hub Cluster Deployment
```
Baremetal Servers (powered off)
  ↓ (Ansible playbooks)
OpenShift Agent-Based Installer ISO
  ↓ (Redfish virtual media boot)
Base OpenShift Cluster (3 compact nodes)
  ↓ (GitOps bootstrap)
ArgoCD Deployment
  ↓ (ArgoCD app-of-apps)
ACM Hub Cluster (fully configured)
```

### Phase 2: Managed Cluster Deployment
```
ACM Hub Cluster
  ↓ (GitOps manifests in clusters/hub/overlays/)
BareMetalHost + NMStateConfig + InfraEnv
  ↓ (ACM assisted-service)
Discovery ISO per cluster
  ↓ (Agent-based installation)
Managed Cluster (registered to ACM)
  ↓ (GitOps day-2 config in clusters/<name>/)
Fully Configured Managed Cluster
```

### Repository Structure
```
.
├── ansible/                    # Phase 1: Hub cluster provisioning
│   ├── baremetal-acm.yml       # End-to-end hub deployment
│   ├── deploy-acm-mce.yml      # ACM/MCE operator deployment
│   ├── lvmcluster-config.yml   # LVM storage automation
│   └── templates/              # Jinja2 templates for OpenShift configs
├── .bootstrap/                 # ArgoCD initial deployment manifests
├── components/                 # Reusable Kustomize components (operators, configs)
├── groups/                     # Shared configurations for cluster groups
│   └── all/                    # Universal day-2 configs (all clusters)
├── clusters/                   # Cluster-specific configurations
│   ├── hub/                    # Hub cluster ArgoCD applications
│   │   ├── values.yaml         # Hub-specific apps (ACM, AAP, MTV)
│   │   └── overlays/           # Managed cluster definitions (etl4, etl6)
│   ├── <managed-cluster-name>/ # Day-2 configs for managed clusters
│   │   ├── values.yaml         # Cluster-specific apps
│   │   └── overlays/           # Cluster-specific overlays
└── .helm-charts/               # Helm charts for cluster provisioning
    ├── bm-cluster-agent-install/   # ACM agent-based cluster creation
    └── cluster-registration/       # ManagedCluster registration
```

---

## Coding Standards

### Ansible Best Practices

**Primary Reference**: [Red Hat CoP Automation Good Practices](https://github.com/redhat-cop/automation-good-practices/)

#### Module Naming
- **Always use FQCN** (Fully Qualified Collection Names)
- **Prefer `ansible.builtin` prefix** over short names

✅ **Correct**:
```yaml
- name: Create directory
  ansible.builtin.file:
    path: /opt/data
    state: directory
    mode: '0755'

- name: Apply Kubernetes manifest
  kubernetes.core.k8s:
    state: present
    definition: "{{ lookup('ansible.builtin.file', 'manifest.yaml') | from_yaml }}"
```

❌ **Incorrect**:
```yaml
- name: Create directory
  file:  # Missing ansible.builtin prefix
    path: /opt/data
    state: directory

- name: Apply manifest
  k8s:  # Missing kubernetes.core prefix
    state: present
    definition: "{{ lookup('file', 'manifest.yaml') | from_yaml }}"
```

#### Variable Management
- **Prefer `vars_files` over inline `vars`**
- Use `group_vars/` and `host_vars/` for inventory-based variables
- Use `defaults/main.yml` in roles for default values

✅ **Correct**:
```yaml
# playbook.yml
---
- name: Deploy OpenShift cluster
  hosts: localhost
  vars_files:
    - vars/cluster_config.yml
    - vars/hardware_profiles/{{ hardware_type }}.yml
  tasks:
    - name: Generate install-config
      ansible.builtin.template:
        src: install-config.yaml.j2
        dest: "{{ install_dir }}/install-config.yaml"
```

```yaml
# vars/cluster_config.yml
---
cluster_name: hub
openshift_version: "4.17.2"
base_domain: ocp.example.com
```

❌ **Incorrect**:
```yaml
---
- name: Deploy OpenShift cluster
  hosts: localhost
  vars:  # Should use vars_files
    cluster_name: hub
    openshift_version: "4.17.2"
    base_domain: ocp.example.com
  tasks:
    - name: Generate install-config
      ansible.builtin.template:
        src: install-config.yaml.j2
        dest: "{{ install_dir }}/install-config.yaml"
```

#### Comments in YAML
- **Comments are encouraged** for complex logic, non-obvious decisions, or hardware-specific configurations
- Use inline comments for parameter explanations
- Use block comments for task/section descriptions

```yaml
---
# This playbook discovers non-OS disks on OpenShift worker nodes and configures
# LVM Storage Operator. It uses 'oc debug' to inspect each node's block devices
# and automatically generates the LVMCluster CR.

- name: Configure LVM Storage Operator on OpenShift Nodes
  hosts: localhost
  gather_facts: false
  vars_files:
    - vars/storage_config.yml
  
  tasks:
    # Use oc debug to run commands directly on the node's host filesystem
    # This is necessary because we need to inspect /dev/disk/by-id paths
    # which aren't visible from regular pods
    - name: Discover non-OS disks on each node via oc debug
      ansible.builtin.shell:
        cmd: >
          oc debug node/{{ item }} -- chroot /host /bin/bash -c
          'BOOT_DEV=$(findmnt -vno SOURCE /boot | head -n 1);
          OS_DISK=$(lsblk -no pkname $BOOT_DEV | head -n 1);
          for dev in $(lsblk -nd -o NAME,TYPE | awk "\$2==\"disk\" {print \$1}"); do
            if [ "$dev" != "$OS_DISK" ]; then
              find /dev/disk/by-id -type l -lname "*/$dev" -print -quit;
            fi;
          done' 2>/dev/null | grep '^/dev/disk/by-id' || true
      loop: "{{ target_nodes }}"
      register: node_disks
      changed_when: false
```

#### Linting and Validation
- **Always run `ansible-lint` before completing work**
- Address all errors; warnings should be evaluated (may suppress with inline comments if justified)
- Install separately: `pip install ansible-lint`

**Check before committing**:
```bash
# Lint all playbooks
ansible-lint ansible/*.yml

# Lint specific playbook
ansible-lint ansible/baremetal-acm.yml

# Check syntax only (faster)
ansible-playbook --syntax-check ansible/baremetal-acm.yml
```

**Common ansible-lint rules to follow**:
- `name[casing]`: Task names should be capitalized sentences
- `yaml[line-length]`: Keep lines under 160 characters (use `>` for long strings)
- `risky-shell-pipe`: Use `set -o pipefail` with shell pipes
- `no-changed-when`: Set `changed_when: false` for read-only commands
- `package-latest`: Avoid `state: latest`, prefer specific versions

### OpenShift GitOps Best Practices

**Reference**: [OpenShift GitOps Recommended Practices](https://developers.redhat.com/blog/2025/03/05/openshift-gitops-recommended-practices#)

#### Sync Waves
Use ArgoCD sync waves to control deployment order:

| Wave | Component Type | Examples |
|------|----------------|----------|
| 5 | Operators | `acm-operator`, `nmstate-operator`, `metallb-operator` |
| 6 | Operator configuration | `nmstate-instance`, `metallb-configuration` |
| 15 | Operator instances | `acm-instance`, `mtv-configuration` |
| 25 | Complex configurations | `acm-configuration`, cluster definitions |

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: acm-operator
  namespace: openshift-gitops
  annotations:
    argocd.argoproj.io/sync-wave: '5'  # Deploy operator first
spec:
  # ... application spec
```

#### Health Checks
Define custom health checks for operators that don't follow standard patterns:

```yaml
# In ArgoCD CR spec.resourceHealthChecks
- group: operator.open-cluster-management.io
  kind: MultiClusterHub
  check: |
    hs = {}
    if obj.status ~= nil and obj.status.phase ~= nil then
      if obj.status.phase == "Running" then
        hs.status = "Healthy"
        hs.message = "MultiClusterHub is running"
        return hs
      end
    end
    hs.status = "Progressing"
    hs.message = "Waiting for MultiClusterHub"
    return hs
```

#### Kustomize Best Practices
- Use `components` for reusable, composable configuration pieces
- Avoid deep overlay nesting (max 2-3 levels)
- Use `replacements` for variable substitution over patches when possible
- Keep `kustomization.yaml` files simple and readable

✅ **Correct**:
```yaml
# clusters/hub/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../groups/all  # Import base configurations

helmCharts:
  - name: argocd-app-of-app
    valuesFile: values.yaml
    namespace: openshift-gitops
```

#### Environment Variable Substitution
This repo uses a custom ArgoCD plugin for envsubst-style variable replacement:

**Available variables** (defined in `.bootstrap/argocd.yaml`):
- `${CLUSTER_NAME}`: Current cluster name (e.g., `hub`)
- `${CLUSTER_BASE_DOMAIN}`: Cluster base domain (e.g., `hub.ocp.example.com`)
- `${PLATFORM_BASE_DOMAIN}`: Platform domain (e.g., `ocp.example.com`)
- `${INFRA_GITOPS_REPO}`: GitOps repository URL

**Usage in manifests**:
```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-app
spec:
  host: "myapp.apps.${CLUSTER_BASE_DOMAIN}"  # Substituted at apply time
  to:
    kind: Service
    name: my-app
```

---

## Development Workflow

### Working on Ansible Playbooks

1. **Create/edit playbook** in `ansible/` directory
2. **Use vars_files** for configuration (create in `ansible/vars/`)
3. **Add inline comments** for complex logic
4. **Use FQCN** for all modules
5. **Test syntax**: `ansible-playbook --syntax-check <playbook>.yml`
6. **Run ansible-lint**: `ansible-lint <playbook>.yml`
7. **Fix all errors**, evaluate warnings
8. **Test execution** in lab environment (if available)
9. **Commit changes**

### Working on GitOps Configurations

1. **Identify component** (operator, instance, configuration)
2. **Choose location**:
   - New reusable component → `components/<name>/`
   - Cluster-specific → `clusters/<cluster-name>/overlays/<name>/`
   - Group-wide → `groups/<group-name>/overlays/<name>/`
3. **Create kustomization.yaml** with resources
4. **Test build**: `kustomize build --enable-helm <directory>`
5. **Add to appropriate values.yaml** (hub or managed cluster)
6. **Set sync wave** annotation appropriately
7. **Commit changes**
8. **Monitor ArgoCD** for sync status

### Creating New Managed Cluster Definitions

Current approach requires manual file creation (see `BAREMETAL_ACM_ASSESSMENT.md` section 3.2.1 for gaps).

**Planned improvement**: Generator playbook to create from CSV inventory.

**For now**, use existing clusters as templates:
```bash
# Copy existing cluster definition
cp -r clusters/hub/overlays/cluster-etl4 clusters/hub/overlays/cluster-prod1

# Edit all files to update:
# - Cluster name (etl4 → prod1)
# - BMC addresses
# - MAC addresses
# - IP addresses (static IPs, VIPs)
# - Network configuration (NMStateConfig)

# Add to clusters/hub/values.yaml:
applications:
  prod1:
    annotations:
      argocd.argoproj.io/sync-wave: '25'
    destination:
      namespace: prod1
    source:
      path: clusters/hub/overlays/cluster-prod1
```

---

## Known Issues and Gaps

Reference: `BAREMETAL_ACM_ASSESSMENT.md` for comprehensive analysis.

### Critical Issues (must fix before use)
1. **MAC address field mismatch**: `ansible/baremetal-acm.yml` uses `macAddress`, but `ansible/templates/agent-config.yaml.j2` expects `mac_address`
2. **No default StorageClass**: LVM playbook doesn't set default annotation
3. **Per-host file explosion**: 3 YAML files per baremetal node (not scalable)

### High Priority Gaps
- No secrets management (BMC credentials in plaintext)
- No object storage for ACM observability (MinIO or ODF needed)
- No pre-flight validation in playbooks
- Network templates incomplete (bonding, VLANs examples needed)

### Planned Improvements
- BMH/NMState generator playbook (from CSV inventory)
- Hardware profile abstraction (Dell, HPE, etc.)
- Sealed Secrets or External Secrets Operator integration
- Object storage component (MinIO or ODF)

**When working on code**: Check assessment document for context on design decisions and known limitations.

---

## Testing and Validation

### Ansible Playbook Testing

**Syntax check** (fast, catches YAML/Jinja2 errors):
```bash
ansible-playbook --syntax-check ansible/baremetal-acm.yml
```

**Lint check** (enforces best practices):
```bash
ansible-lint ansible/baremetal-acm.yml
```

**Dry-run** (check mode, doesn't make changes):
```bash
ansible-playbook --check ansible/baremetal-acm.yml
```

**Diff mode** (shows what would change):
```bash
ansible-playbook --check --diff ansible/baremetal-acm.yml
```

### Kustomize Testing

**Build test** (renders manifests locally):
```bash
kustomize build --enable-helm clusters/hub/
```

**Validate YAML** (checks for syntax errors):
```bash
kustomize build --enable-helm clusters/hub/ | yamllint -
```

**Check for substitution variables**:
```bash
# Should NOT contain literal ${CLUSTER_NAME} etc. after envsubst
kustomize build --enable-helm clusters/hub/ | grep '\${' || echo "No unsubstituted vars"
```

### GitOps Testing

**Preview changes** (in ArgoCD UI):
- Click application → App Diff → Show Full Diff

**Sync with --dry-run** (via CLI):
```bash
argocd app sync <app-name> --dry-run
```

**Check sync waves** (ensure order is correct):
```bash
argocd app get <app-name> -o yaml | grep sync-wave
```

---

## Pre-Commit Checklist

Before committing Ansible playbooks:
- [ ] All modules use FQCN (e.g., `ansible.builtin.file`)
- [ ] Variables moved to `vars_files` (not inline `vars`)
- [ ] Comments added for complex logic
- [ ] `ansible-playbook --syntax-check` passes
- [ ] `ansible-lint` errors addressed (warnings evaluated)
- [ ] Sensitive data removed (no hardcoded passwords, API keys)

Before committing GitOps manifests:
- [ ] `kustomize build` renders successfully
- [ ] Sync wave annotation set appropriately
- [ ] Health check defined for custom CRs (if needed)
- [ ] Environment variables use correct syntax `${VAR_NAME}`
- [ ] Application registered in appropriate `values.yaml`

---

## User Profile and Preferences

**Role**: Senior Principal Chief Architect  
**Experience**: 30 years in infrastructure/platform engineering  
**Relevant Skills**:
- Ansible: 7 years
- Linux/UNIX: 30 years (since SunOS/early 1990s)
- Python: 20 years
- KVM: 15 years
- OpenShift: 3 years (2019-2022), returning after focus on RHEL/Satellite/IdM

**Communication Preferences**:
- **Concise explanations** (high experience level, can refine if needed)
- **Direct feedback** on code quality and architectural decisions
- **Proactive suggestions** for improvements when applicable
- **Examples over theory** (practical, working code preferred)

**Interaction Style**:
- Will provide ongoing feedback and refinements
- Prefers iterative development (start with basics, improve over time)
- Open to complete rewrites if existing code doesn't meet goals
- Values automation best practices and standards compliance

---

## Assistant Behavior

### Code Quality Standards
- Always use Ansible FQCN and `ansible.builtin` prefix
- Run `ansible-lint` checks before marking work complete
- Suggest `vars_files` refactoring when inline vars are excessive
- Reference Red Hat CoP automation practices when relevant

### Architecture Decisions
- Prioritize **simplicity and maintainability** over feature completeness
- **Suggest rewrites** if existing code strays from project goals
- **Flag security issues** (plaintext secrets, missing RBAC, etc.)
- Consider **scalability** (works for 3 nodes AND 30 nodes)

### Documentation
- Keep explanations **concise** given user's experience level
- Provide **working examples** over theoretical descriptions
- Reference external docs (Red Hat CoP, OpenShift GitOps practices)
- Update `BAREMETAL_ACM_ASSESSMENT.md` if gaps/recommendations change

### Workflow
- **Read assessment document** before suggesting major changes
- **Check existing patterns** in the repo before introducing new approaches
- **Validate syntax** (ansible-lint, kustomize build) before completion
- **Ask clarifying questions** if requirements are ambiguous

### Proactive Suggestions
When reviewing or creating code, proactively suggest:
- Refactoring inline vars to `vars_files`
- Adding comments for complex NMState/Kustomize logic
- Health checks for custom CRs in ArgoCD
- Pre-flight validations in Ansible playbooks
- Security improvements (secrets management, RBAC)

---

## Quick Reference

### Ansible Collection Prefixes
```yaml
ansible.builtin      # Core modules (file, copy, template, shell, etc.)
kubernetes.core      # k8s, k8s_info, helm, etc.
community.general    # redfish_command, redfish_info, etc.
ansible.posix        # mount, authorized_key, etc.
```

### Common ArgoCD Annotations
```yaml
argocd.argoproj.io/sync-wave: '15'                    # Sync order
argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
argocd.argoproj.io/compare-options: IgnoreExtraneous  # For operators with dynamic resources
```

### Kustomize Generators
```yaml
helmCharts:           # Helm chart inflation
configMapGenerator:   # Generate ConfigMaps
secretGenerator:      # Generate Secrets
replacements:         # Variable substitution
```

### Environment Variables (ArgoCD Plugin)
```bash
CLUSTER_NAME           # e.g., "hub"
CLUSTER_BASE_DOMAIN    # e.g., "hub.ocp.example.com"
PLATFORM_BASE_DOMAIN   # e.g., "ocp.example.com"
HUB_BASE_DOMAIN        # Hub cluster domain
INFRA_GITOPS_REPO      # This repository URL
```

---

## Project Contacts

**Maintainer**: Senior Principal Chief Architect  
**Repository**: `openshift-virtualization-gitops`  
**Target Platform**: Red Hat OpenShift 4.16+ with Advanced Cluster Management  
**Deployment Model**: Baremetal Agent-Based Installer + GitOps

---

**Last Updated**: 2026-04-28  
**Version**: 1.0
