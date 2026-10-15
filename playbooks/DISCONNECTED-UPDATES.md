# Disconnected Day 2 Update Process

This document describes the roles and playbook used to update an already-deployed
disconnected Hub+Spoke environment (`hub-spoke-disconnected-setup.yaml`): OpenShift
minor/patch version updates, and operator/catalog Day 2 updates. It only applies
after `migrate_mirror` has run — i.e. Quay on the hub is the live mirror registry
and the jumphost mirror has (or could be) decommissioned.

It does **not** apply to the connected (`hub-spoke-setup.yaml` / `hub-spoke-ipi-setup*.yaml`)
or Nutanix playbooks — those clusters update via the normal OperatorHub/Cincinnati
graph and are out of scope here.

## Design goals

- **One inventory, one mechanism.** No new per-version variables. A version bump is
  just editing the same `openshift_version` / `ocp_patch_version` / `disconnected.operators[].channels`
  values already used for initial install, in the same inventory file, then
  re-running a playbook. Nothing in the roles hardcodes a version.
- **Mirror → Catalog → Operators → Platform**, in that order, matching the order
  components actually depend on each other. Each phase is its own tag, re-runnable
  independently, mirroring the existing `jumphost` / `hub` / `spokes` / `migrate_mirror`
  / `verify` tag structure in `hub-spoke-disconnected-setup.yaml`.
- **Hub and spokes use the same split as initial provisioning**: the hub is
  self-managed (direct `oc adm upgrade`), spokes are Hive/ACM-managed (`ClusterImageSet`
  + `ClusterCurator`, the same primitives Hive/ACM already use to provision them).
- **Reuse proven, bug-fixed logic** rather than re-implementing it: the new mirror
  role shares the `ImageSetConfiguration` template with `mirror_ocp_content` and
  the Quay repo-creation race workaround (`remirror_to_quay/tasks/converge_unauthorized.yml`,
  see [[project_disconnected_bugs]] Bug 7) rather than duplicating it.
- **MCO-safe waits everywhere.** Every MachineConfigPool wait uses the two-step
  "wait for Updating=True, then wait for Updated=True" pattern. A bare
  `Updated=True` wait can pass against a stale condition left over from a prior
  roll (see [[project_disconnected_bugs]] Bug 6) — this bit the initial-install
  playbook once already and the update playbook performs the same class of
  MachineConfig-triggering operations (new IDMS, new CatalogSource digest).

## New roles

| Role | Target | Purpose |
|---|---|---|
| `ocp/mirror_update_content` | hub (jumphost/local) | Re-mirrors the OCP release + operator catalogs described by the *current* inventory (`openshift_version`, `ocp_patch_version`, `disconnected.operators`) directly into the existing Quay instance on the hub. Shares `mirror_ocp_content`'s `ImageSetConfiguration` template and `remirror_to_quay`'s Quay race-convergence logic. |
| `ocp/update_catalog_source` | hub + spokes | Applies the new IDMS and patches the `redhat-operators` CatalogSource (hub directly, spokes via `ManifestWork`) to the freshly mirrored catalog digest. Reuses the same `ManifestWork` object names `patch_cluster_idms` created, so there is always exactly one steady-state object per cluster, not one per update. |
| `ocp/approve_install_plans` | hub + spokes | Generic: finds OLM `Subscription`s in `UpgradePending` and approves their `InstallPlan` if `installPlanApproval: Manual`, then waits for the corresponding CSV to reach `Succeeded`. Not tied to any specific operator or version. |
| `ocp/verify_cluster_update` | hub + spokes | Pre-flight (no ClusterOperator degraded, MCPs settled) and post-flight (ClusterVersion at target, ClusterOperators healthy, MCPs `Updated`) health gate. Mode selected by `verify_cluster_update_mode: preflight|postflight`. |
| `ocp/update_ocp_hub` | hub only | Self-managed upgrade: `oc adm upgrade --to-image=<mirrored release tag> --allow-explicit-upgrade`, then the MCO-safe wait pattern until `ClusterVersion` reports the target version and is no longer `Progressing`. |
| `acm/update_ocp_spoke` | spokes (via hub) | Hive/ACM-managed upgrade: creates/reuses a `ClusterImageSet` pointing at the Quay-mirrored release image for the target version, then applies a `ClusterCurator` (`desiredCuration: upgrade`) and waits for it to report success and for the spoke's reported version to match. |

## New playbook: `disconnected-update.yaml`

```bash
cd playbooks
ansible-playbook disconnected-update.yaml -i inventory-disconnected.yaml
```

Phases (tags), run in order by default:

| Tag | Hosts | What it does |
|---|---|---|
| `preflight` | hub, spokes | Health gate — refuses to proceed if any ClusterOperator is degraded or an MCP is mid-roll. |
| `mirror` | hub | `configure_quay_mirror` (re-derive live Quay route + robot credentials — these are not persisted facts) then `mirror_update_content`. |
| `catalog` | hub, spokes | `update_catalog_source`, then wait for CatalogSources `READY` (reuses `verify_mirror_health`). |
| `operators` | hub, spokes | `approve_install_plans`. |
| `ocp_hub` | hub | `update_ocp_hub`. |
| `ocp_spokes` | spokes | `acm/update_ocp_spoke` (one cluster at a time by default — see below). |
| `verify` | hub, spokes | `verify_cluster_update` (postflight). |

### Usage

1. Edit the **same inventory file** used for initial deployment:
   - Patch update: bump `ocp_patch_version`.
   - Minor update: bump `openshift_version` and `ocp_patch_version` together.
     OpenShift does not support skipping a minor version — if jumping more than
     one minor, run the whole playbook once per minor, landing on that minor's
     latest supported z-stream each time, same as a normal OCP upgrade path.
   - Operator update: edit `disconnected.operators[].channels` (bump
     `min_version`/`max_version`, or add a new channel) for the operator(s) being
     updated.
2. Run the full playbook, or scope it:
   ```bash
   ansible-playbook disconnected-update.yaml -i inventory-disconnected.yaml --tags mirror,catalog,operators
   # ... verify operators landed, then:
   ansible-playbook disconnected-update.yaml -i inventory-disconnected.yaml --tags ocp_hub
   ansible-playbook disconnected-update.yaml -i inventory-disconnected.yaml --tags ocp_spokes --limit managed-cluster-3
   ```
3. Always upgrade the hub before spokes (the default play order already does
   this) — ACM/MCE/Hive on the hub must be at or above the version it's managing.

### Notes

- This playbook assumes `migrate_mirror` has already run against the target
  environment (Quay on hub is the live mirror). It does not touch the jumphost.
- `mirror_update_content` mirrors into the **same** Quay org/registry — it does
  not create new catalog versions side-by-side. Re-running `--tags mirror` with
  the same inventory values is a safe no-op (same idempotency pattern as
  `mirror_ocp_content`/`remirror_to_quay`: a version+image sentinel check).
- Operator upgrades beyond a single channel step, or cross-channel moves, follow
  whatever the operator's own OLM channel/replaces graph allows — this playbook
  only mirrors content and approves pending `InstallPlan`s, it does not change
  Subscription channel itself (that's defined in the external `gitops-install-*`
  repos per [[CLAUDE.md]]'s ArgoCD-first Day 2 pattern).
