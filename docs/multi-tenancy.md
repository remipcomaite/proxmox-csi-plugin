# Multi-tenant deployments

Run one Kubernetes cluster per tenant on a shared Proxmox VE cluster, with each
tenant isolated from the others at the Proxmox level.

> Proxmox VE has no native storage multi-tenancy (no per-tenant isolation or quota at
> the storage backend). The model below is assembled by the operator from existing
> building blocks (pools, RBAC, RBD namespaces). A native feature request is open but
> unanswered: [bugzilla #7625](https://bugzilla.proxmox.com/show_bug.cgi?id=7625).

The controller does not enforce tenant ownership: a tenant that administers its own
cluster can create a static `PersistentVolume` with any `volumeHandle`, or a `Node` /
`CSINode` carrying any VMID or `providerID`. The controller then calls the Proxmox API
with whatever token it holds. **The security boundary is therefore the Proxmox ACL of
that token, not the CSI driver.** The default single-cluster setup in
[install.md](install.md) grants the role on `/`, which lets any tenant reach every
disk. For multiple tenants, scope each token.

## Model

One controller instance and one Proxmox token per tenant. Each tenant gets its own
Proxmox pool, its own storage entry backed by its own RBD namespace, and a distinct
`controllerVmID`.

| Per tenant | Example for `tenant-a` |
| --- | --- |
| Proxmox pool (holds the tenant's worker VMs and storage) | `tenant-a` |
| RBD namespace (isolates the images) | `pool/tenant-a` |
| Storage entry pointing to that namespace | `tenant-a` |
| User + token, ACL on `/pool/tenant-a` only | `csi-tenant-a@pve!csi` |
| `controllerVmID` (distinct per tenant) | `9001` |

## Proxmox setup

```shell
# One RBD namespace per tenant, inside the shared Ceph pool.
rbd namespace create <pool>/tenant-a

# One storage entry per tenant, bound to that namespace.
pvesm add rbd tenant-a --pool <pool> --namespace tenant-a --content images --krbd 0

# One pool per tenant, holding its worker VMs and its storage entry.
pveum pool add tenant-a
pveum pool modify tenant-a --storage tenant-a   # add the worker VMIDs with --vms

# Minimal role. No VM.Allocate: the tenant cannot create or destroy VMs.
pveum role add CSITENANT -privs "VM.Audit VM.Config.Disk Datastore.Allocate Datastore.AllocateSpace Datastore.Audit"

# User + token, ACL scoped to the tenant's pool only.
pveum user add csi-tenant-a@pve
pveum user token add csi-tenant-a@pve csi --privsep 1
pveum acl modify /pool/tenant-a -token 'csi-tenant-a@pve!csi' -role CSITENANT
```

Deploy one controller per tenant, each with that tenant's token and
`controllerVmID` in its cloud config:

```yaml
config:
  features:
    controllerVmID: 9001
  clusters:
    - url: https://<proxmox>:8006/api2/json
      token_id: "csi-tenant-a@pve!csi"
      token_secret: "<secret>"
      region: <cluster>
```

## What the ACL protects

With a token scoped to `/pool/tenant-a`, the Proxmox API refuses every cross-tenant
action a tenant could attempt through the CSI or directly:

| Attempt by tenant A | Result |
| --- | --- |
| List / allocate / delete in another tenant's storage | `403` (`Datastore.*`) |
| Attach a disk to another tenant's VM | `403` (`VM.Config.Disk`) |
| Enumerate another tenant's VMs or storages | not returned by `/cluster/resources` |

One shape to know: a `DeleteVolume` for an out-of-scope volume returns success without
deleting anything (the volume is invisible to the token), rather than an error.

## Verify the isolation

Run these from any host that can reach the Proxmox API, using **tenant A's token**
(`H="Authorization: PVEAPIToken=csi-tenant-a@pve!csi=<secret>"`, `API=https://<proxmox>:8006/api2/json`).
Each cross-tenant call must return `403`; the own-pool call must succeed. Use a
throwaway volume name you created for the test, never a real disk.

```shell
# Enumeration: tenant A sees only its own VMs and storages.
curl -sk -H "$H" "$API/cluster/resources?type=storage" # only tenant-a entries

# Read another tenant's storage -> 403.
curl -sk -o /dev/null -w '%{http_code}\n' -H "$H" "$API/nodes/<node>/storage/tenant-b/status"

# Allocate / delete in another tenant's storage -> 403.
curl -sk -o /dev/null -w '%{http_code}\n' -H "$H" -X POST \
  "$API/nodes/<node>/storage/tenant-b/content" --data-urlencode filename=vm-9001-pvc-x --data-urlencode size=1G --data-urlencode vmid=9001

# Attach a disk to another tenant's VM -> 403.
curl -sk -o /dev/null -w '%{http_code}\n' -H "$H" -X POST \
  "$API/nodes/<node>/qemu/<tenant-b-vmid>/config" --data-urlencode scsi9=tenant-a:<a-test-disk>

# Create or destroy a VM -> 403 (the role has no VM.Allocate).
curl -sk -o /dev/null -w '%{http_code}\n' -H "$H" -X POST "$API/nodes/<node>/qemu" --data-urlencode vmid=<free-id> --data-urlencode memory=512

# Positive control: same action inside tenant A's own pool -> 200.
curl -sk -o /dev/null -w '%{http_code}\n' -H "$H" "$API/nodes/<node>/storage/tenant-a/status"
```

The positive control matters: without it, a `403` could be a plain "not found" from
ACL-filtered enumeration rather than a real authorization denial.

## Pitfalls

- **One storage entry per RBD namespace.** Two storage entries pointing to the same
  pool and namespace expose the same images: a tenant with rights on its own entry
  could then list, attach and delete another tenant's volumes through it. Give each
  storage entry its own namespace (or pool).
- **Do not grant the role on `/`.** That is the single-cluster default and gives every
  tenant access to every disk. Scope each token to its pool.
- **Distinct `controllerVmID` per tenant**, so volumes (`vm-<id>-pvc-*`) are
  attributable and an operation on one VMID never spans tenants.

## Limits

- No volume snapshots: the Proxmox content copy endpoint is root-only, so keep
  `snapshotter.enabled: false`.
- No PVC cloning on shared storage: Proxmox cannot copy a volume within the same
  shared storage.
- No per-namespace quota in Ceph (quotas exist per pool only). A tenant can fill the
  shared pool; monitor usage per namespace and alert on pool fullness.

## Scale (untested)

This model was validated only with a small number of tenants. It has **not** been
tested at larger scale, and two Proxmox-side limits are unverified there:

- `storage.cfg` lives in pmxcfs, whose per-file size cap is 1 MiB. One RBD entry is a
  handful of lines, so this should allow thousands of tenants — but the exact ceiling
  was not measured.
- `pvestatd` polls every active storage periodically; many entries on the same Ceph
  pool mean many repeated `rados` calls. The load as the entry count grows was not
  measured.

Treat large deployments as unproven: measure `storage.cfg` size and `pvestatd` load on
your cluster before scaling the tenant count.
