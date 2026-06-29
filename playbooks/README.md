# How to Use These Playbooks

## Playbook Selection

### AWS Playbooks

| Playbook | Architecture | Spoke Provisioning |
|---|---|---|
| `full-global-hub-setup.yaml` | Three-tier Global Hub | Tier 2+3 via ACM/Hive |
| `full-global-hub-ipi-setup.yaml` | Three-tier Global Hub | All tiers via IPI |
| `hub-spoke-setup.yaml` | Two-tier Hub+Spoke | Spokes via ACM/Hive |
| `hub-spoke-ipi-setup.yaml` | Two-tier Hub+Spoke | All clusters via IPI |

### Nutanix AHV Playbooks

| Playbook | Architecture | Spoke Provisioning |
|---|---|---|
| `hub-spoke-setup-nutanix.yaml` | Two-tier Hub+Spoke | Spokes via ACM/Hive on Nutanix |
| `hub-spoke-ipi-setup-nutanix.yaml` | Two-tier Hub+Spoke | All clusters via IPI on Nutanix |

---

## Three-Tier Global Hub

### Deploy the entire environment (Tiers 1, 2, and 3):

```bash
ansible-playbook playbooks/full-global-hub-setup.yaml -i inventory.yaml
```

### Deploy ONLY a new batch of Managed Clusters (Tier 3):
Let's say a month from now you add `managed-cluster-3` and `managed-cluster-4` to your inventory file. You don't want to touch the Hubs. Just run:

```bash
ansible-playbook playbooks/full-global-hub-setup.yaml -i inventory.yaml --tags "tier3"
```

### Deploy a specific tier, but limit it to a single new cluster:

```bash
ansible-playbook playbooks/full-global-hub-setup.yaml -i inventory.yaml --tags "tier3" --limit "managed-cluster-3"
```

---

## Two-Tier Hub+Spoke (no Global Hub)

The hub+spoke playbooks deploy a single ACM hub with direct managed cluster attachment — no Regional Hubs, no ACM Global Hub. The hub gets ACM, ODF, MCO, and GitOps enforcement policies.

### Deploy hub and all spokes (all IPI):

```bash
ansible-playbook playbooks/hub-spoke-ipi-setup.yaml -i inventory.yaml
```

### Deploy hub and all spokes (spokes via Hive):

```bash
ansible-playbook playbooks/hub-spoke-setup.yaml -i inventory.yaml
```

### Deploy hub only, add spokes later:

```bash
ansible-playbook playbooks/hub-spoke-ipi-setup.yaml -i inventory.yaml --tags "hub"
# ... later, add spokes to inventory ...
ansible-playbook playbooks/hub-spoke-ipi-setup.yaml -i inventory.yaml --tags "spokes"
```

### Add a single new spoke:

```bash
ansible-playbook playbooks/hub-spoke-ipi-setup.yaml -i inventory.yaml --tags "spokes" --limit "managed-cluster-3"
```

---

## Example Inventories

### Global Hub Inventory
```yaml
all:
  vars:
    home_dir: /home/sscaling
    tmp_dir: "{{ home_dir }}/tmp"
    openshift_version: "4.20"
    ocp_patch_version: "13"
    odf_git_repo_revision: "main"
    force_update: true
    base_domain: "sandbox123.opentlc.com"
    ssh_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
    pull_secret: "{{ lookup('file', '~/pull-secret.json') | from_json }}"
    aws:
      account_id: <aws_account_id>
      aws_access_key_id: <aws_access_key_id>
      aws_secret_access_key: <aws_secret_access_key>
      aws_region: <aws_region>
    smtp:
      host: "smtp.gmail.com"
      port: "587"
      from_address: "<your_email>@gmail.com"
      require_tls: "true"
      username: "<your_username>@gmail.com"
      app_password: "<16 char app password>"
      default_to: "<default_email>@gmail.com"
      esp_to: "<esp_rule_email>@gmail.com"
  children:
    hub_cluster:
      hosts:
        global-hub:
          ansible_connection: local
          ocp_cluster:
            name: "global-hub"
            spokes_clusterset: regional-hubs
          install_config:    
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.2xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
            infra:
              instance_type: "m6a.4xlarge"
              replicas: "3"
            storage:
              instance_type: "m6a.4xlarge"
              replicas: "3"
    regional_hubs:
      hosts:
        regional-hub-1:
          ansible_connection: local
          ocp_cluster:
            name: "regional-hub-1"
            parent_clusterset: regional-hubs
            spokes_clusterset: managed-clusters
          install_config:    
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.2xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
            infra:
              instance_type: "m6a.4xlarge"
              replicas: "3"
            storage:
              instance_type: "m6a.4xlarge"
              replicas: "3"
        regional-hub-2:
          ansible_connection: local
          ocp_cluster:
            name: "regional-hub-2"
            parent_clusterset: regional-hubs
            spokes_clusterset: managed-clusters
          install_config:    
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.2xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
            infra:
              instance_type: "m6a.4xlarge"
              replicas: "3"
            storage:
              instance_type: "m6a.4xlarge"
              replicas: "3"
    managed_clusters:
      hosts:
        managed-cluster-1:
          ansible_connection: local
          regional_hub: regional-hub-1
          gitops_version: "1.18"
          ocp_cluster:
            name: "managed-cluster-1"
            parent_clusterset: managed-clusters
          install_config:    
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
        managed-cluster-2:
          ansible_connection: local
          regional_hub: regional-hub-2
          gitops_version: "1.19"
          ocp_cluster:
            name: "managed-cluster-2"
            parent_clusterset: managed-clusters
          install_config:    
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
```

### Hub+Spoke Inventory

Simpler than the global hub inventory: no `regional_hubs` group, no `regional_hub` per-spoke variable. Managed clusters reference their clusterset via `parent_clusterset`; the hub targets them via `spokes_clusterset`.

```yaml
all:
  vars:
    home_dir: /home/sscaling
    tmp_dir: "{{ home_dir }}/tmp"
    openshift_version: "4.20"
    ocp_patch_version: "13"
    force_update: true
    base_domain: "sandbox123.opentlc.com"
    ssh_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
    pull_secret: "{{ lookup('file', '~/pull-secret.json') | from_json }}"
    aws:
      account_id: <aws_account_id>
      aws_access_key_id: <aws_access_key_id>
      aws_secret_access_key: <aws_secret_access_key>
      aws_region: <aws_region>
    smtp:
      host: "smtp.gmail.com"
      port: "587"
      from_address: "<your_email>@gmail.com"
      require_tls: "true"
      username: "<your_username>@gmail.com"
      app_password: "<16 char app password>"
      default_to: "<default_email>@gmail.com"
      esp_to: "<esp_rule_email>@gmail.com"
  children:
    hub_cluster:
      hosts:
        acm-hub:
          ansible_connection: local
          ocp_cluster:
            name: "acm-hub"
            spokes_clusterset: managed-clusters
          install_config:
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.2xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
            infra:
              instance_type: "m6a.4xlarge"
              replicas: "3"
            storage:
              instance_type: "m6a.4xlarge"
              replicas: "3"
    managed_clusters:
      hosts:
        managed-cluster-1:
          ansible_connection: local
          gitops_version: "1.18"
          ocp_cluster:
            name: "managed-cluster-1"
            parent_clusterset: managed-clusters
          install_config:
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
        managed-cluster-2:
          ansible_connection: local
          gitops_version: "1.19"
          ocp_cluster:
            name: "managed-cluster-2"
            parent_clusterset: managed-clusters
          install_config:
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
```

---

## Nutanix AHV Deployments

The Nutanix playbooks deploy the same Hub+Spoke architecture but provision clusters on Nutanix AHV using the OpenShift IPI installer's native Nutanix platform support.

### Prerequisites

In addition to the [standard prerequisites](../README.md), Nutanix deployments require:

- Nutanix Prism Central (PC) 2022.6 or later
- Nutanix AHV 20220304.342 or later
- A Prism Central admin account for the installer
- AHV IP Address Management (IPAM) enabled on the target subnet
- Two available static IPs on the subnet for the API VIP and Ingress VIP
- The Prism Central certificate must be signed by a publicly trusted CA (or provide the CA via `additional_trust_bundle`)
- The Prism Central Image Service must reach `rhcos.mirror.openshift.com` (or provide `cluster_os_image` for disconnected installs)

### Deploy hub and all spokes (all IPI):

```bash
ansible-playbook hub-spoke-ipi-setup-nutanix.yaml -i inventory-nutanix.yaml
```

### Deploy hub and all spokes (spokes via ACM/Hive):

```bash
ansible-playbook hub-spoke-setup-nutanix.yaml -i inventory-nutanix.yaml
```

### Deploy hub only, add spokes later:

```bash
ansible-playbook hub-spoke-ipi-setup-nutanix.yaml -i inventory-nutanix.yaml --tags hub
# ... later, add spokes to inventory ...
ansible-playbook hub-spoke-ipi-setup-nutanix.yaml -i inventory-nutanix.yaml --tags spokes
```

### Add a single new spoke:

```bash
ansible-playbook hub-spoke-ipi-setup-nutanix.yaml -i inventory-nutanix.yaml --tags spokes --limit managed-cluster-3
```

### Nutanix Hub+Spoke Inventory

```yaml
all:
  vars:
    home_dir: /home/sscaling
    tmp_dir: "{{ home_dir }}/tmp"
    openshift_version: "4.20"
    ocp_patch_version: "13"
    force_update: true
    base_domain: "example.com"
    ssh_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
    pull_secret: "{{ lookup('file', '~/pull-secret.json') | from_json }}"
    smtp:
      host: "smtp.gmail.com"
      port: "587"
      from_address: "<your_email>@gmail.com"
      require_tls: "true"
      username: "<your_username>@gmail.com"
      app_password: "<16 char app password>"
      default_to: "<default_email>@gmail.com"
      esp_to: "<esp_rule_email>@gmail.com"
    nutanix:
      # Prism Central credentials (used by installer and ACM/Hive)
      prism_central:
        address: prism-central.example.com
        port: 9440
        username: admin
        password: <prism_central_password>
      # Prism Element (AHV cluster) — only one cluster per OCP install is supported
      prism_element:
        address: prism-element.example.com
        port: 9440
        uuid: <prism_element_uuid>          # From Prism Element > Settings > Cluster Details
        subnet_uuid: <subnet_uuid>           # From Prism Central > Network > Subnets
      # Static VIPs reserved on the target subnet
      api_vip: 10.0.0.100
      ingress_vip: 10.0.0.101
      # Optional: provide RHCOS image URL for disconnected/air-gapped installs
      # cluster_os_image: http://internal-mirror.example.com/rhcos-4.20-nutanix.qcow2
      # Optional: custom CA bundle if Prism Central uses a non-public certificate
      # additional_trust_bundle: |
      #   -----BEGIN CERTIFICATE-----
      #   <your_CA_certificate>
      #   -----END CERTIFICATE-----
  children:
    hub_cluster:
      hosts:
        acm-hub:
          ansible_connection: local
          ocp_cluster:
            name: "acm-hub"
            spokes_clusterset: managed-clusters
          install_config:
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              vcpu_sockets: 4
              cores_per_socket: 2
              memory_mib: 16384
              disk_gib: 120
            workers:
              vcpu_sockets: 2
              cores_per_socket: 2
              memory_mib: 8192
              disk_gib: 120
              replicas: 3
            infra:                  # Optional: creates dedicated infra node pool
              vcpu_sockets: 4
              cores_per_socket: 1
              memory_mib: 16384
              disk_gib: 120
              replicas: 3
            storage:                # Optional: creates dedicated ODF storage node pool
              vcpu_sockets: 4
              cores_per_socket: 1
              memory_mib: 16384
              disk_gib: 120
              replicas: 3
    managed_clusters:
      hosts:
        managed-cluster-1:
          ansible_connection: local
          gitops_version: "1.18"
          ocp_cluster:
            name: "managed-cluster-1"
            parent_clusterset: managed-clusters
          install_config:
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              vcpu_sockets: 4
              cores_per_socket: 2
              memory_mib: 16384
              disk_gib: 120
            workers:
              vcpu_sockets: 2
              cores_per_socket: 2
              memory_mib: 8192
              disk_gib: 120
              replicas: 3
        managed-cluster-2:
          ansible_connection: local
          gitops_version: "1.19"
          ocp_cluster:
            name: "managed-cluster-2"
            parent_clusterset: managed-clusters
          install_config:
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              vcpu_sockets: 4
              cores_per_socket: 2
              memory_mib: 16384
              disk_gib: 120
            workers:
              vcpu_sockets: 2
              cores_per_socket: 2
              memory_mib: 8192
              disk_gib: 120
              replicas: 3
```

### Inventory variable reference (Nutanix)

**`nutanix` block (top-level `all.vars`):**

| Variable | Required | Description |
|---|---|---|
| `nutanix.prism_central.address` | Yes | Hostname or IP of Prism Central |
| `nutanix.prism_central.port` | No | PC API port (default: `9440`) |
| `nutanix.prism_central.username` | Yes | PC admin username |
| `nutanix.prism_central.password` | Yes | PC admin password |
| `nutanix.prism_element.address` | Yes | Hostname or IP of the AHV cluster (Prism Element) |
| `nutanix.prism_element.port` | No | PE API port (default: `9440`) |
| `nutanix.prism_element.uuid` | Yes | UUID of the Prism Element cluster |
| `nutanix.prism_element.subnet_uuid` | Yes | UUID of the IPAM-enabled subnet for cluster VMs |
| `nutanix.api_vip` | Yes | Static IP reserved for the OpenShift API endpoint |
| `nutanix.ingress_vip` | Yes | Static IP reserved for the OpenShift Ingress router |
| `nutanix.cluster_os_image` | No | URL to RHCOS `.qcow2` image (disconnected installs) |
| `nutanix.additional_trust_bundle` | No | PEM CA certificate for non-public Prism Central TLS |

**`install_config` per-host (Nutanix):**

| Variable | Required | Default | Description |
|---|---|---|---|
| `control_plane.vcpu_sockets` | No | `4` | vCPU sockets per control plane node |
| `control_plane.cores_per_socket` | No | `2` | Cores per vCPU socket |
| `control_plane.memory_mib` | No | `16384` | RAM in MiB |
| `control_plane.disk_gib` | No | `120` | OS disk size in GiB |
| `workers.vcpu_sockets` | No | `2` | vCPU sockets per worker node |
| `workers.cores_per_socket` | No | `2` | Cores per vCPU socket |
| `workers.memory_mib` | No | `8192` | RAM in MiB |
| `workers.disk_gib` | No | `120` | OS disk size in GiB |
| `workers.replicas` | Yes | — | Number of worker nodes |
| `infra.vcpu_sockets` | No | `4` | vCPU sockets per infra node |
| `infra.cores_per_socket` | No | `1` | Cores per vCPU socket |
| `infra.memory_mib` | No | `16384` | RAM in MiB |
| `infra.disk_gib` | No | `120` | OS disk size in GiB |
| `infra.replicas` | Yes* | — | *Required when `infra:` block is present |
| `storage.*` | No | same as infra | Same fields as `infra`, for ODF storage nodes |