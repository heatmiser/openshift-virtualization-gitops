# Bare-metal ACM cluster installation

ACM is not a standalone operating system; it is an operator that runs on top of an existing OpenShift cluster. Deploying ACM requires a two-phase approach:

Phase 1: Build the Foundation. First utilize the standalone OpenShift Agent-Based Installer (ABI) to provision a base OpenShift cluster on bare-metal hardware.

Phase 2: Bootstrap GitOps & ACM. Once the base cluster is running, apply the .bootstrap configurations from the repository. By targeting the clusters/hub configuration, ArgoCD will take over and automatically install the acm-operator and acm-instance.

Within the ansible folder, a comprehensive, end-to-end Ansible playbook that starts from powered-off bare-metal servers to a fully functional ACM Hub cluster managed by GitOps.

# Prerequisites
The openshift-install and oc binaries installed on your Ansible control node.

The kubernetes.core and community.general Ansible collections installed.

Your install-config.yaml.j2 and agent-config.yaml.j2 templates ready.

The End-to-End Hub Provisioning Playbook