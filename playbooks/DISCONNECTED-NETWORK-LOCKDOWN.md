# Disconnected Network Lockdown

By default, `hub-spoke-disconnected-setup.yaml` only controls *where images come
from* (the mirror registry + IDMS/CatalogSource). It does **not** isolate the
cluster's network: the OpenShift installer provisions its own VPC with public
subnets, an Internet Gateway, and NAT Gateways, and cluster nodes reach the
jumphost's mirror registry over its public IP across the open internet. A
cluster built this way can still reach the public internet for anything not
covered by the mirror config (telemetry, Cincinnati, a missed image reference).

This feature adds real network isolation, enabled **by default** for
`hub-spoke-disconnected-setup.yaml` (a disconnected install should mean
network-disconnected, not just image-source-disconnected) without changing
the existing jumphost-mirrors-then-migrate-to-hub workflow. Set
`disconnected.network_lockdown.enabled: false` to opt out.

## Architecture

- **Jumphost VPC** (unchanged, created by `provision-jumphost.yaml` /
  `aws/aws_provision_jumphost`): public subnet, Internet Gateway, public IP.
  This is the only thing in the environment allowed general internet egress.
- **Cluster VPC** (new): private subnets only — no Internet Gateway, no NAT
  Gateway attached, ever. Houses the hub and every spoke.
- **VPC Peering Connection** between the two VPCs, so cluster nodes reach the
  jumphost's mirror registry and LDAP over private AWS networking instead of
  the public internet. The peering route (and the security-group egress that
  allows it) is only needed until `migrate_mirror` runs — at that point it's
  revoked, so the clusters become fully unreachable to/from the jumphost.
- **Private hosted zone / jumphost VPC association**: `publish: Internal`
  clusters get a private Route53 zone associated only with the Cluster VPC —
  but `openshift-install` for the hub runs *on the jumphost*, outside that
  VPC, so without an explicit second association it can't resolve
  `api.<cluster>.<base_domain>` and bootstrap fails. `ocp/aws_provision_disconnected_ocp_ipi`
  runs `aws/associate_private_zone_with_jumphost` concurrently with the async
  install job to add that association as soon as the zone exists. Spokes need
  it too, just for a different reason: Hive's installer runs inside
  hive-controllers on the hub (already in the Cluster VPC, so Hive itself is
  never blocked), but every Day 2 spoke-configuration task in
  `hub-spoke-disconnected-setup.yaml` runs `oc`/ansible against the spoke's
  own kubeconfig from the jumphost, which needs the same resolution.
  `acm/aws_provision_disconnected_cluster_via_acm` runs the same role once the
  spoke's infra ID is known, before any Day 2 task touches the spoke's API.
- **Jumphost ingress to node security groups**: VPC peering and the routes it
  adds only establish a network *path* — they don't open anything. Security
  groups are a separate, stateful allow-list, and openshift-install's
  auto-created master/worker/bootstrap security groups only allow ingress
  from the cluster's own machineNetwork (the Cluster VPC CIDR) by default,
  never from a peered VPC. Without an explicit rule, the jumphost can't SSH
  the bootstrap node for log gathering, reach the API while the installer
  waits for it, or run any Day 2 `oc`/ansible config against the cluster
  afterwards. `aws/allow_jumphost_ingress_to_nodes` authorizes ingress from
  the jumphost VPC CIDR for SSH (22), the API (6443), and the Machine Config
  Server (22623) — run concurrently with the hub's async install job (via its
  `metadata.json`/infra ID, same timing as the zone association) and once,
  right after kubeconfig extraction, for each spoke (via `oc get
  infrastructure` against the spoke's kubeconfig, since spokes have no local
  `metadata.json`). `aws/lockdown_cluster_security_groups` revokes this same
  ingress at the same final-cutover point it revokes egress.
- **Mirror registry address**: the Cluster VPC has no Internet Gateway/NAT at
  all, and VPC peering only routes to the peer VPC's private CIDR — it can
  never route to the jumphost's public IP/DNS. `disconnected.mirror_registry.hostname`
  (documented as the jumphost's public IP/FQDN) is therefore unreachable from
  any cluster node under lockdown. `mirror_registry_hostname`/`mirror_registry_url`
  (in `ocp/setup_mirror_registry`, `ocp/remirror_to_quay`, and the main
  playbook's own derivations) default to `jumphost_private_ip` instead
  whenever `network_lockdown.enabled` is true, falling back to
  `disconnected.mirror_registry.hostname` only when lockdown is disabled.
- **Manual cloud credentials**: there is no IAM VPC endpoint in commercial AWS
  regions, so the Cloud Credential Operator's mint-mode `GetUser` validation
  against `iam.amazonaws.com` can never succeed from the no-IGW/no-NAT Cluster
  VPC — it fails with an i/o timeout, zero CredentialsRequests get
  provisioned, and no worker machines ever launch. The hub install-config sets
  `credentialsMode: Manual` under lockdown, and `ocp/aws_provision_disconnected_ocp_ipi`
  runs `openshift-install create manifests` then `ocp/create_manual_cloud_credentials`
  (extracts CredentialsRequests from the mirrored release image, writes a
  long-lived-key Secret per request into `manifests/`) before `create cluster`.
  Hive-provisioned spokes (`roles/acm/aws_provision_disconnected_cluster_via_acm`)
  land in the same isolated VPC and hit the identical failure; `acm/create_manual_cloud_credentials_configmap`
  does the same extraction and writes the Secrets into a ConfigMap referenced
  from the `ClusterDeployment`'s `spec.provisioning.manifestsConfigMapRef`
  (Hive's equivalent of dropping files into `manifests/` before install).
- **AWS API proxy for Hive spoke installs**: Hive's spoke install pod runs on
  a hub node — inside the same isolated Cluster VPC as the cluster being
  installed, not on the jumphost. That VPC has no route to IAM or Route53 at
  all (see above), so beyond CredentialsRequests, the install pod also can't
  create the spoke's master/worker instance profiles or private hosted zone
  (confirmed live: 3h, zero AWS resources created, installer hung
  indefinitely on a tagging API call). `aws/setup_aws_api_proxy_on_jumphost`
  runs a squid forward proxy on the jumphost, reachable from the Cluster VPC
  over the existing peering, restricted to `*.amazonaws.com` only — not
  general internet access. The spoke's `ClusterDeployment` points its
  install pod at this proxy via `spec.provisioning.installerEnv`
  (HTTP_PROXY/HTTPS_PROXY/NO_PROXY), so the one-shot install process can
  reach AWS's own APIs directly, matching the real target topology (ACM Hub
  provisions spokes directly, even across sites) without ever wiring this
  proxy into the resulting cluster itself — once installed, spoke nodes stay
  exactly as locked down as before. The hub doesn't need this: its own
  `openshift-install` runs directly on the jumphost, which already has full
  internet access.
- **VPC Endpoints** (S3 gateway; EC2, ELB, STS, Service Quotas, Resource
  Group Tagging API interface endpoints) in the
  Cluster VPC, so the AWS cloud-provider and ELB/EBS CSI controllers work
  without any internet path. There is no Route53 interface endpoint —
  `com.amazonaws.<region>.route53` does not exist in any commercial AWS
  region (confirmed via `describe-vpc-endpoint-services`; it is not a
  still-rolling-out gap, it will never appear) — so the ingress operator can
  never reach Route53 to manage DNS itself from inside the Cluster VPC. See
  "Apps wildcard DNS" below for how that's handled instead. Not every other
  service's interface endpoint is offered in every region either — the role
  discovers what AWS actually offers in the target region first and skips
  (with a warning, not a failure) anything not available there, rather than
  hardcoding a region list that would silently go stale.
- **Apps wildcard DNS**: because the ingress operator can't reach Route53 (see
  above), the default IngressController's `endpointPublishingStrategy.loadBalancer.dnsManagementPolicy`
  is set to `Unmanaged` via a manifest `ocp/aws_provision_disconnected_ocp_ipi`
  injects at `create manifests` time — otherwise it fails forever with
  `NoZones: The record isn't present in any zones`. `aws/create_apps_wildcard_dns_record`
  creates the `*.apps` CNAME in the installer-managed private hosted zone
  itself instead, from the jumphost (which has real AWS access, unlike the
  cluster), once the router's load balancer exists. Runs concurrently with
  the hub's async install job, same pattern as the private-zone-association
  and jumphost-ingress roles. Hive-provisioned spokes hit the identical
  `NoZones` failure and get the same fix: the IngressController manifest
  rides along in the spoke's existing manifestsConfigMap (Hive's
  `manifestsConfigMapRef` only accepts one), and the record is created
  mid-install using the infra ID from the `ClusterDeployment`'s own
  `spec.clusterMetadata.infraID` (read from the Hub) rather than `oc get
  infrastructure` against the spoke — the jumphost has no route to a spoke's API yet at
  this point in the install.
- **Security Group lockdown**: the installer always creates cluster node
  security groups with a broad `0.0.0.0/0` / all-protocols egress rule. A new
  role revokes that and replaces it with VPC-CIDR-only (plus jumphost-VPC-CIDR
  until `migrate_mirror`).
- **Telemetry/Insights**: disabled on every cluster regardless of version (no
  OCP capability-list hardcoding — see [[project_disconnected_bugs]] for why
  this codebase avoids version-pinned assumptions) by removing
  `cloud.openshift.com` from the pull secret and scaling down the Insights
  Operator.

## New roles

| Role | Runs | Purpose |
|---|---|---|
| `aws/provision_cluster_vpc` | jumphost phase, once | Isolated Cluster VPC, one shared private route table, peering connection to the jumphost VPC (discovered by tag, same pattern as `aws_provision_jumphost`). |
| `aws/create_private_subnets` | hub phase (hub) / spokes phase (each spoke) | Per-cluster private subnets in the Cluster VPC — one per zone in `install_config.control_plane`/`workers.zones` (or one auto-discovered AZ if zones aren't set). CIDRs are allocated automatically: the Cluster VPC's `/16` is carved into `/24`s, each zone getting the lowest-numbered `/24` not already claimed anywhere in the VPC (re-checked against live AWS state before every allocation, so a spoke run after the hub naturally skips what the hub already took — no cross-host coordination needed). |
| `aws/create_vpc_endpoints` | hub phase, once (after hub's subnets exist) | S3 gateway + EC2/ELB/STS/Route53 interface endpoints. |
| `aws/associate_private_zone_with_jumphost` | hub phase, during the hub's IPI install, concurrently with the async install job | Associates the hub's `publish: Internal` private hosted zone with the jumphost VPC once it exists, so `openshift-install` (running on the jumphost) can resolve `api.<cluster>.<base_domain>` during bootstrap. |
| `aws/allow_jumphost_ingress_to_nodes` | hub phase, during the hub's IPI install, concurrently with the async install job; spokes phase, right after kubeconfig extraction | Authorizes ingress from the jumphost VPC CIDR to the cluster's node security groups for SSH (22), API (6443), and MCS (22623) — the installer's default node SGs only allow ingress from the cluster's own machineNetwork, not a peered VPC. |
| `aws/lockdown_cluster_security_groups` | hub/spokes phase (initial) and migrate_mirror phase (final) | Tightens node security group egress. `network_lockdown_allow_jumphost_egress` controls whether jumphost-VPC-CIDR egress is still allowed. |
| `aws/verify_network_isolation` | verify phase | Asserts no `0.0.0.0/0` route/egress remains; live `oc debug node` curl-to-public-IP check expecting failure. |
| `ocp/disable_telemetry` | hub/spokes phase | Removes `cloud.openshift.com` from the pull secret, scales down Insights Operator. |

## Inventory

Nothing is required — `enabled` defaults to `true` for this playbook, `vpc_name`
and `vpc_cidr` both have defaults, and per-cluster subnets/CIDRs are allocated
automatically from the zones already configured in `install_config`. To
override or opt out:

```yaml
disconnected:
  network_lockdown:
    enabled: false                     # opt out entirely; defaults to true
    vpc_name: "ocp-cluster-vpc"        # optional, default shown
    vpc_cidr: "10.90.0.0/16"           # Cluster VPC CIDR — must be a /16 (auto /24 allocation assumes it) and must not overlap the jumphost VPC CIDR (10.0.0.0/16 by default)
```

The install-config `machineNetwork` must match whatever VPC the subnets
actually belong to, or the installer rejects the config
(`subnet's CIDR range ... is outside of the specified machine networks`). When
enabled, both install-config templates (IPI and Hive-based) default
`machineNetwork` to `vpc_cidr` instead of the old `10.0.0.0/16` jumphost-VPC
default — set `install_config.machine_network.cidr` explicitly per cluster
only if you need something other than the whole Cluster VPC CIDR.

## What this does not do

- It does not touch the jumphost itself — it keeps its own small VPC and
  public subnet exactly as `aws_provision_jumphost` creates it today.
- It does not add NACLs — the isolation is enforced by having no default
  route to an Internet Gateway/NAT Gateway at all (routing-level isolation),
  reinforced by the security-group lockdown. This is simpler to reason about
  and verify than a NACL ruleset, and avoids NACLs' stateless-rule foot-guns.
- It does not exclude the OCP `Insights` capability at install time — that
  would require hardcoding a capability list that changes across OCP
  versions. `ocp/disable_telemetry` achieves the same outcome (no telemetry
  egress) on any version, pre- or post-install.
