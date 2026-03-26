# NASty → nasty-csi Integration Roadmap

This document maps what nasty-csi expects from NASty against what currently exists,
and lists work needed on each side to reach full CSI spec compliance.

## Current State

The CSI driver is a thin JSON-RPC client over NASty's WebSocket API. Core volume
lifecycle (create, delete, NFS/SMB/iSCSI/NVMe-oF sharing, basic snapshots) is wired
end-to-end. The following are open gaps.

---

## P0 — Blockers (CSI won't fully work without these)

### 1. Filesystem subvolume quota enforcement

**Problem:** NASty creates bcachefs subvolumes for NFS/SMB volumes but has no way to
enforce a size limit on them. The CSI driver stores `capacity_bytes` as an xattr and
reports it to Kubernetes, but the underlying filesystem is uncapped — a pod can write
more than the requested PVC size.

For block volumes (iSCSI/NVMe-oF) this is not a problem because `vol.img` is a fixed
sparse file and the loop device limits writes naturally.

**What's needed:** bcachefs per-subvolume quota support via `bcachefs quota`.
The bcachefs-tools quota commands (`bcachefs quota set`, `bcachefs quota show`) exist
but their stability in the kernel version NASty targets needs to be verified.

**Work:**
- NASty engine: after `subvolume.create` (filesystem type), call `bcachefs quota set`
  with `volsize` = requested bytes if available
- NASty engine: on `subvolume.resize`, update the quota
- NASty engine: expose quota usage in `Subvolume.used_bytes`
- Verify the bcachefs quota kernel feature is enabled in the NixOS kernel config

**Affects:** NFS, SMB volumes. Block volumes are unaffected.

---

## P1 — High Priority (important features return Unimplemented)

### 2. CreateVolume from snapshot (`snapshot.clone`)

**Problem:** Every protocol's `createVolumeFromSnapshot` handler in nasty-csi returns
`codes.Unimplemented`. The entire "restore from backup" workflow in Kubernetes is broken.

**What exists:** NASty already has `snapshot.clone` which creates a new subvolume from
a snapshot. The CSI driver just has not implemented the call yet.

**Work in nasty-csi:**
- Implement `createFromSnapshot` path for each protocol (NFS, SMB, iSCSI, NVMe-oF)
- Flow: `snapshot.clone` → create share on the new subvolume → set xattr properties
- Return proper `ContentSource` in the CreateVolume response

**Work in NASty:**
- Verify `snapshot.clone` preserves subvolume type (block vs filesystem)
- Verify loop device is attached after cloning a block subvolume (currently
  `restore_block_devices` runs at startup; a clone needs an explicit attach)
- `snapshot.clone` should accept an optional `pool` target so cross-pool clones
  are possible in the future

### 3. Snapshot xattr properties

**Problem:** Snapshots have no xattr storage. The CSI driver encodes snapshot metadata
entirely into the snapshot ID string (`{protocol}:{volume_id}@{snapshot_name}`). This
means:

- Cannot look up a snapshot by its CSI snapshot name without decoding the ID
- Cannot store creation timestamp, source volume, or delete strategy on the snapshot itself
- `ListSnapshots` has to do string parsing rather than property lookups

There is a `TODO` in the driver: *"When NASty supports snapshot properties/xattrs,
use PropertySnapshotID instead."*

**What's needed:** bcachefs exposes xattrs on subvolumes via `xattr(7)` (`user.*`
namespace). Snapshots are also subvolumes in bcachefs — they should support xattrs
too, but this needs to be verified and exposed via the NASty API.

**Work in NASty:**
- Verify xattrs work on read-only bcachefs snapshots
- Add `snapshot.set_properties` and `snapshot.get_properties` API methods
- (Or expose snapshots through the existing `subvolume.set_properties` path if they
  share the same xattr interface)

**Work in nasty-csi (after NASty supports it):**
- Switch snapshot ID storage to xattr-based lookup (same pattern as volumes)
- Remove encoding/decoding of snapshot IDs from strings

---

## P2 — Medium Priority (functionality gaps)

### 4. `snapshot.list_all` — pool-agnostic snapshot listing

**Problem:** `snapshot.list` requires a `pool` parameter. The CSI `ListSnapshots`
handler has to enumerate pools first and then list per-pool — there is a `TODO`
comment noting this limitation. If no driver-wide default pool is configured, the
listing is incomplete.

**Work in NASty:**
- Add `snapshot.list_all` (mirrors `subvolume.list_all`) — lists all snapshots across
  all mounted pools

### 5. Volume expansion for block volumes at NASty level

**Problem:** `ControllerExpandVolume` for block protocols (iSCSI, NVMe-oF) only
updates the `capacity_bytes` xattr. Actual resizing of the `vol.img` sparse file and
the loop device happens at the node side via `truncate` + `losetup --set-capacity`.
NASty's own state (`volsize_bytes`) is never updated, so the WebUI shows the old size.

**Work in NASty:**
- `subvolume.resize` should: update the sparse file size (`truncate`), update the loop
  device (`losetup --set-capacity`), and update stored `volsize_bytes`

**Work in nasty-csi:**
- Call `subvolume.resize` from `expandISCSIVolume` / `expandNVMeOFVolume` instead of
  only updating the xattr

### 6. NVMe-oF host ACL in create ✅

**Resolved:** The unified `share.nvmeof.create` method now accepts `allowed_hosts`.
When provided, `allow_any_host` is set to false and each host NQN is added.

---

## P3 — Low Priority / Future

### 7. ListSnapshots CSI endpoint

The CSI `ListSnapshots` is implemented in nasty-csi but depends on `snapshot.list_all`
(item 4 above) for correct operation. Once that is available, this can be tested and
enabled in the driver's capability advertisement.

### 8. Detached snapshot clones (COW volumes)

The driver has stubs for `controller_snapshot_detached.go` with a TODO:
*"Implement detached snapshots using NASty's native bcachefs clone/copy API."*
This enables zero-copy volume cloning within a pool — useful for dev/test environments
that clone production data. Blocked on bcachefs kernel support for writable clones
of read-only snapshots.

### 9. `ControllerModifyVolume`

Returns `Unimplemented`. This is a newer CSI spec feature for changing volume
parameters post-creation (e.g. changing compression). Not used by current Kubernetes
versions in standard workflows.

### 10. Minimum volume size enforcement in NASty

The CSI driver enforces a 1 GiB minimum and errors out before calling NASty.
NASty itself does not validate this. Low risk since the driver guards it, but NASty
should return a clear error if `volsize_bytes < 1 GiB` to protect against direct API
calls.

---

## Summary Table

| # | Item | NASty work | CSI work | Priority |
|---|------|-----------|----------|----------|
| 1 | Filesystem subvolume quotas | `bcachefs quota` in create/resize | None | P0 |
| 2 | CreateVolume from snapshot | Verify clone + block attach | Implement all 4 protocol handlers | P1 |
| 3 | Snapshot xattr properties | `snapshot.set/get_properties` | Switch to xattr lookup | P1 |
| 4 | `snapshot.list_all` | Add method | Use in ListSnapshots | P2 |
| 5 | Block volume resize in NASty | Fix `subvolume.resize` | Call it in expand handlers | P2 |
| 6 | NVMe-oF host ACL in quick-create | Add `hosts` field | Already passing it | P2 |
| 7 | ListSnapshots CSI | (needs #4) | Enable capability | P3 |
| 8 | Detached snapshot clones | bcachefs writable clones | Implement handler | P3 |
| 9 | ControllerModifyVolume | Optional | Implement if needed | P3 |
| 10 | Minimum size validation in NASty | Add guard | None | P3 |

## Notes

- All protocol-level metadata (share IDs, NQNs, IQNs) is stored as POSIX xattrs
  on the bcachefs subvolume. This is the source of truth — no sidecar JSON files.
- Volume IDs use `pool/subvolume_name` format. Legacy plain-name IDs still work via
  O(n) xattr scan.
- Operator API tokens scoped to a pool are the intended isolation mechanism for
  multi-tenant or multi-cluster deployments.
