# How to Use These Playbooks

## Playbook Selection

### AWS Playbooks

| Playbook | Architecture | Spoke Provisioning |
|---|---|---|
| `full-global-hub-setup.yaml` | Three-tier Global Hub | Tier 2+3 via ACM/Hive |
| `full-global-hub-ipi-setup.yaml` | Three-tier Global Hub | All tiers via IPI |
| `hub-spoke-setup.yaml` | Two-tier Hub+Spoke | Spokes via ACM/Hive |
| `hub-spoke-ipi-setup.yaml` | Two-tier Hub+Spoke | All clusters via IPI |

### Disconnected Playbooks

| Playbook | Architecture | Spoke Provisioning |
|---|---|---|
| `provision-jumphost.yaml` | Jumphost provisioning only | N/A |
| `hub-spoke-disconnected-setup.yaml` | Two-tier Hub+Spoke (Disconnected) | Spokes via ACM/Hive on AWS |

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

## Provision Jumphost (AWS)

If you don't have a stable jumphost with a public IP, use this playbook to provision an EC2 instance that will serve as the mirror registry host and automation control node.

```bash
cd playbooks
ansible-playbook provision-jumphost.yaml -i inventory-disconnected.yaml
```

This will:
1. Create an EC2 key pair from your `~/.ssh/id_ed25519.pub`
2. Create a security group with ports 22 (SSH) and 8443 (mirror registry) open
3. Launch a RHEL 9 `t3.large` instance with a 500 GB gp3 volume
4. Install Ansible, Python dependencies, AWS CLI, and `git`
5. Clone the automation repo to the instance
6. Copy your pull secret and AWS credentials
7. Print the SSH command and next steps

After it completes, set `disconnected.mirror_registry.hostname` in your inventory to the printed public IP, then copy the inventory to the jumphost and run the disconnected playbook from there.

**Customization** (override in inventory `all.vars`):

| Variable | Default | Description |
|---|---|---|
| `jumphost_instance_type` | `t3.large` | EC2 instance type |
| `jumphost_name` | `disconnected-jumphost` | EC2 instance Name tag |
| `jumphost_volume_size_gb` | `500` | Root volume size (oc-mirror workspace can be 50-200 GB) |
| `automation_repo_url` | GitHub SSH URL | Repo to clone onto the jumphost |
| `automation_repo_branch` | `main` | Branch to check out |

---

## Disconnected Hub+Spoke (AWS)

The disconnected playbook deploys the same Hub+Spoke architecture in an environment with limited or no internet access. It uses a local mirror registry on the jumphost to host OCP release images and operator catalogs.

### Workflow

The playbook has five phases, controlled via tags:

1. **`jumphost`** — Installs tools (`oc-mirror`, `mirror-registry`), stands up a local Quay-based mirror registry, and mirrors OCP release images + operator catalogs. Run once.
2. **`hub`** — Provisions the ACM Hub cluster via IPI using mirrored images, applies disconnected cluster configuration (IDMS, CatalogSource, pull secret), installs Day 2 operators via ArgoCD, and configures ACM for disconnected spoke provisioning.
3. **`spokes`** — Provisions managed clusters via ACM/Hive with disconnected settings. Each spoke receives mirror configuration via ManifestWork.
4. **`migrate_mirror`** — After Quay is running on the hub, migrates all mirrored content from the jumphost mirror registry to Quay, updates IDMS on the hub and all spokes, and enables jumphost decommission. Skip with `--skip-tags migrate_mirror` if you intend to keep the jumphost as the permanent mirror registry.
5. **`summary`** — Prints connection info for all clusters.

### Prerequisites

In addition to the [standard prerequisites](../README.md):

- The jumphost must have internet access (even limited) to pull images from `registry.redhat.io`, `quay.io`, and `mirror.openshift.com`
- The jumphost FQDN/IP must be reachable from all cluster nodes (hub and spokes) on the mirror registry port (default: 8443)
- Sufficient disk space on the jumphost for mirrored content (~50-100 GB depending on operators selected)

### Deploy the complete disconnected environment:

```bash
cd playbooks
ansible-playbook hub-spoke-disconnected-setup.yaml -i inventory-disconnected.yaml
```

### Prepare jumphost only (mirror content first, deploy later):

```bash
ansible-playbook hub-spoke-disconnected-setup.yaml -i inventory-disconnected.yaml --tags jumphost
```

### Deploy hub after jumphost is ready:

```bash
ansible-playbook hub-spoke-disconnected-setup.yaml -i inventory-disconnected.yaml --tags hub
```

### Add spokes after hub is running:

```bash
ansible-playbook hub-spoke-disconnected-setup.yaml -i inventory-disconnected.yaml --tags spokes
# Or a single spoke:
ansible-playbook hub-spoke-disconnected-setup.yaml -i inventory-disconnected.yaml --tags spokes --limit managed-cluster-3
```

### Migrate mirror registry from jumphost to Quay (decommission jumphost):

Runs automatically as part of the full playbook. To run standalone after the fact:

```bash
ansible-playbook migrate-mirror-to-quay.yaml -i inventory-disconnected.yaml
```

To skip migration and keep the jumphost as the permanent mirror registry:

```bash
ansible-playbook hub-spoke-disconnected-setup.yaml -i inventory-disconnected.yaml --skip-tags migrate_mirror
```

### Disconnected Hub+Spoke Inventory

The disconnected inventory extends the standard Hub+Spoke inventory with a `disconnected` block in `all.vars`:

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
    aws:
      account_id: <aws_account_id>
      aws_access_key_id: <aws_access_key_id>
      aws_secret_access_key: <aws_secret_access_key>
      aws_region: <aws_region>
    disconnected:
      mirror_registry:
        hostname: "jumphost.example.com"     # FQDN reachable by all cluster nodes
        port: 8443                            # Mirror registry port (default: 8443)
        init_user: "init"                     # Mirror registry admin user
        init_password: "<registry_password>"  # Mirror registry admin password
      mirror_base_path: "ocp4"               # Base path in mirror registry
      operators:                              # Operators to mirror for the disconnected install
        # ACM and its required MCE sub-operator (versions must be paired)
        - name: advanced-cluster-management
          channels:
            - name: "release-2.17"            # verify current channel: oc-mirror --v2 list operators --catalog ... --package advanced-cluster-management
        - name: multicluster-engine
          channels:
            - name: "stable-2.17"             # must match ACM version (ACM 2.17 ships with MCE 2.17)
        # ODF meta-operator + all sub-operators it installs automatically.
        # ODF 4.22 uses a meta-operator pattern: odf-operator reads a ConfigMap and
        # installs each sub-operator from the catalog. All sub-packages must be mirrored.
        - name: odf-operator
          channels:
            - name: "stable-4.22"
        - name: ocs-operator
          channels:
            - name: "stable-4.22"
        - name: mcg-operator
          channels:
            - name: "stable-4.22"
        - name: rook-ceph-operator
          channels:
            - name: "stable-4.22"
        - name: odf-csi-addons-operator
          channels:
            - name: "stable-4.22"
        - name: cephcsi-operator
          channels:
            - name: "stable-4.22"
        - name: odf-dependencies
          channels:
            - name: "stable-4.22"
        - name: odf-prometheus-operator
          channels:
            - name: "stable-4.22"
        - name: recipe
          channels:
            - name: "stable-4.22"
        - name: ocs-client-operator
          channels:
            - name: "stable-4.22"
        - name: ocs-tls-profiles
          channels:
            - name: "stable-4.22"
        - name: odf-external-snapshotter-operator
          channels:
            - name: "stable-4.22"
        # GitOps operator — channel drives install_openshift_gitops and policy_install_openshift_gitops roles
        - name: openshift-gitops-operator
          channels:
            - name: "gitops-1.21"             # must match what is available in your OCP version's catalog
        - name: quay-operator
          channels:
            - name: "stable-3.18"
      # Optional: additional images to mirror (not part of release or operator catalogs)
      # additional_images:
      #   - registry.redhat.io/ubi9/ubi:latest
      gitea:
        admin_password: "<gitea_admin_password>"  # Admin password for the internal Git server on the hub
      quay_mirror:
        namespace: "local-quay"                   # Namespace where Quay is deployed
        registry_name: "quay-registry"            # QuayRegistry CR name
        org: "ocp4"                               # Org to create in Quay (matches mirror_base_path)
        robot_account: "mirror"                   # Robot account name for oc-mirror push/pull
        admin_user: "quayadmin"                   # Quay admin username
        admin_password: "<quay_admin_password>"   # Quay admin password
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
        site-1-cluster:
          ansible_connection: local
          gitops_version: "1.19"
          ocp_cluster:
            name: "site-1-cluster"
            parent_clusterset: managed-clusters
          install_config:
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
        site-2-cluster:
          ansible_connection: local
          gitops_version: "1.19"
          ocp_cluster:
            name: "site-2-cluster"
            parent_clusterset: managed-clusters
          install_config:
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
        site-3-cluster:
          ansible_connection: local
          gitops_version: "1.19"
          ocp_cluster:
            name: "site-3-cluster"
            parent_clusterset: managed-clusters
          install_config:
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
        site-4-cluster:
          ansible_connection: local
          gitops_version: "1.19"
          ocp_cluster:
            name: "site-4-cluster"
            parent_clusterset: managed-clusters
          install_config:
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
        site-5-cluster:
          ansible_connection: local
          gitops_version: "1.19"
          ocp_cluster:
            name: "site-5-cluster"
            parent_clusterset: managed-clusters
          install_config:
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
```

### Inventory variable reference (Disconnected)

**`disconnected` block (top-level `all.vars`):**

| Variable | Required | Default | Description |
|---|---|---|---|
| `disconnected.mirror_registry.hostname` | Yes | — | FQDN or IP of the jumphost, reachable by all cluster nodes |
| `disconnected.mirror_registry.port` | No | `8443` | Mirror registry port |
| `disconnected.mirror_registry.init_user` | No | `init` | Mirror registry admin username |
| `disconnected.mirror_registry.init_password` | Yes | — | Mirror registry admin password |
| `disconnected.mirror_base_path` | No | `ocp4` | Base path prefix in the mirror registry |
| `disconnected.operators` | Yes | — | List of operators to mirror (see example above). ODF requires all 12 sub-operator packages — see example inventory. |
| `disconnected.operators[].name` | Yes | — | Operator package name from the Red Hat catalog |
| `disconnected.operators[].channels` | No | — | List of channels to mirror (omit to mirror default channel). Always specify channels — omitting causes the catalog default to be mirrored, which may pull far more content than intended. |
| `disconnected.additional_images` | No | `[]` | Extra images to mirror beyond release and operators |
| `disconnected.gitea.admin_password` | No | `R3dH4t!gitea` | Admin password for the Gitea Git server deployed on the hub cluster |
| `disconnected.quay_mirror.namespace` | No | `local-quay` | Namespace where the Quay operator is deployed |
| `disconnected.quay_mirror.registry_name` | No | `quay-registry` | Name of the QuayRegistry CR |
| `disconnected.quay_mirror.org` | No | value of `mirror_base_path` | Quay organization to create for mirrored content |
| `disconnected.quay_mirror.robot_account` | No | `mirror` | Robot account name used by oc-mirror for push/pull |
| `disconnected.quay_mirror.admin_user` | No | `quayadmin` | Quay admin username (used to create org and robot account) |
| `disconnected.quay_mirror.admin_password` | Yes (migrate_mirror phase) | — | Quay admin password |

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