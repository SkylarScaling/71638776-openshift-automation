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
- **VPC Endpoints** (S3 gateway; EC2, ELB, STS, Route53 interface endpoints) in
  the Cluster VPC, so the AWS cloud-provider, ELB/EBS CSI controllers, and the
  ingress operator's Route53 DNS management work without any internet path.
  Not every service's interface endpoint is offered in every region yet
  (Route53's is a recent, still-rolling-out PrivateLink addition) — the role
  discovers what AWS actually offers in the target region first and skips
  (with a warning, not a failure) anything not yet available there, rather
  than hardcoding a region list that would silently go stale.
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
