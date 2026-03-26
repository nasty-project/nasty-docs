# NASty JSON-RPC API

NASty exposes a **JSON-RPC 2.0** API over **WebSocket** at `/ws`.

## Transport

Connect to `ws://<host>/ws` with a valid session cookie or `Authorization: Bearer <token>` header.

**Request:**
```json
{"jsonrpc": "2.0", "id": 1, "method": "pool.list", "params": {}}
```

**Response:**
```json
{"jsonrpc": "2.0", "id": 1, "result": [...]}
```

**Error:**
```json
{"jsonrpc": "2.0", "id": 1, "error": {"code": -32603, "message": "pool not found: mypool"}}
```

## Authentication

Send `POST /api/login` with `{"username": "...", "password": "..."}` to receive a session token. Pass it as a cookie (`session=<token>`) or `Authorization: Bearer <token>` header on the WebSocket upgrade.

## Roles

| Role | Description |
|------|-------------|
| `admin` | Full access to all methods |
| `operator` | Create/delete subvolumes and snapshots; read pools. Cannot manage users, destroy pools, or change system settings. |
| `readonly` | Read-only access to all list/get methods |

API tokens can additionally be scoped to a single **pool** (restricts visibility) and for operator tokens to a single **owner** (restricts to subvolumes owned by that token).

## Real-time Events

After any successful mutation the server broadcasts an event on the same WebSocket:
```json
{"event": "pool"}
```
Clients should re-fetch the relevant resource when they receive an event. Event types: `pool`, `subvolume`, `snapshot`, `share.nfs`, `share.smb`, `share.iscsi`, `share.nvmeof`, `protocol`, `settings`, `alert`.

---

## Contents

- [Authentication](#authentication)
- [System](#system)
- [System Update](#system-update)
- [bcachefs-tools](#bcachefs-tools)
- [Settings](#settings)
- [Network](#network)
- [Protocols & Services](#protocols--services)
- [Alert Rules](#alert-rules)
- [Block Devices](#block-devices)
- [Pools](#pools)
- [Pool Devices](#pool-devices)
- [Subvolumes](#subvolumes)
- [Snapshots](#snapshots)
- [NFS Shares](#nfs-shares)
- [SMB Shares](#smb-shares)
- [iSCSI Targets](#iscsi-targets)
- [NVMe-oF Subsystems](#nvme-of-subsystems)

## Authentication

### `auth.me`

Return the current session's username and role.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `pool` | string | no | If set, token can only see subvolumes in this pool. |
| `role` | `Role` | yes | Role assigned to this session. |
| `token` | string | yes | Session or API token value. |
| `username` | string | yes | Username of the authenticated user. |


### `auth.logout`

Invalidate the current session token.

**Role:** `any`


### `auth.change_password`

Change a user's password. Admins can change any user; users can change their own.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `new_password` | string | yes | New password to set. |
| `username` | string | yes | Username of the account to update. |


### `auth.create_user`

Create a new local user account.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `password` | string | yes | Initial password for the new user. |
| `role` | `Role` | yes | Role to assign to the new user. |
| `username` | string | yes | Login username for the new user. |


### `auth.delete_user`

Delete a user. Cannot delete your own account.

**Role:** `admin`

**Params:** `{"username": string}`


### `auth.list_users`

List all users (no password hashes).

**Role:** `any`

**Returns:**

``UserInfo`[]`


### `auth.token.list`

List all API tokens (without token values).

**Role:** `admin`

**Returns:**

``ApiTokenInfo`[]`


### `auth.token.create`

Create a long-lived API token. Returns the token value — shown only once.

**Role:** `admin`

**Params:** `{"name": string, "role": Role, "pool": string?, "expires_in_secs": integer?}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `created_at` | integer | yes | Unix timestamp (seconds) when the token was created. |
| `expires_at` | integer | no | Unix timestamp after which the token is rejected. None = never expires. |
| `id` | string | yes | Unique token identifier. |
| `name` | string | yes | Human-readable token name. |
| `pool` | string | no | Pool this token is scoped to, if any. |
| `role` | `Role` | yes | Role assigned to this token. |
| `token` | string | yes | The actual token value — shown only once on creation. |


### `auth.token.delete`

Delete an API token by ID.

**Role:** `admin`

**Params:** `{"id": string}`


## System

### `system.info`

Return hostname, OS version, uptime, bcachefs-tools version info.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bcachefs_commit` | string | no | Short (12-char) commit SHA of the pinned bcachefs-tools in flake.lock |
| `bcachefs_is_custom` | boolean | yes | True when the user has overridden the default bcachefs-tools version |
| `bcachefs_pinned_ref` | string | no | The ref stored in the state file: tag name (e.g. "v1.37.1") or short SHA |
| `bcachefs_version` | string | yes | Output of `bcachefs version` (first line). |
| `hostname` | string | yes | System hostname. |
| `kernel` | string | yes | Running Linux kernel version string. |
| `ntp_synced` | boolean | yes | Whether the system clock is NTP-synchronized. |
| `timezone` | string | yes | IANA timezone string (e.g. `America/New_York`). |
| `uptime_seconds` | integer | yes | System uptime in seconds. |
| `version` | string | yes | NASty engine version string. |


### `system.health`

Return health status of all systemd services.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `services` | `ServiceStatus`[] | yes | Status of individual systemd services. |
| `status` | string | yes | Overall health status string (e.g. `ok`, `degraded`). |


### `system.stats`

Return current CPU, memory, network interface, and disk I/O statistics.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `cpu` | `CpuStats` | yes | CPU core count and load averages. |
| `disk_io` | `DiskIoStats`[] | yes | Per-disk I/O statistics. |
| `memory` | `MemoryStats` | yes | Memory and swap usage. |
| `network` | `NetIfStats`[] | yes | Per-interface network statistics. |


### `system.disks`

Return S.M.A.R.T. health data for all drives. Requires SMART protocol to be enabled.

**Role:** `any`

**Returns:**

``DiskHealth`[]`


### `system.alerts`

Evaluate alert rules against current system state and return any active alerts.

**Role:** `any`

**Returns:**

``ActiveAlert`[]`


### `system.reboot`

Reboot the system.

**Role:** `admin`


### `system.shutdown`

Shut down the system.

**Role:** `admin`


## System Update

### `system.update.version`

Return current version and latest available version.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `current_version` | string | yes | Currently installed version (short commit SHA or `dev`). |
| `latest_version` | string | no | Latest upstream version, if the check has been performed. |
| `update_available` | boolean | no | Whether a newer version is available. None if the check has not been run yet. |


### `system.update.check`

Check for available updates against the upstream repository.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `current_version` | string | yes | Currently installed version (short commit SHA or `dev`). |
| `latest_version` | string | no | Latest upstream version, if the check has been performed. |
| `update_available` | boolean | no | Whether a newer version is available. None if the check has not been run yet. |


### `system.update.apply`

Fetch and apply the latest NixOS generation. Runs `nixos-rebuild switch` in the background.

**Role:** `admin`


### `system.update.rollback`

Roll back to the previous NixOS generation.

**Role:** `admin`


### `system.update.status`

Return the current update operation status and log.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `log` | string | yes |  |
| `reboot_required` | boolean | yes | True when the activated system has a different kernel than the booted one |
| `state` | string | yes | "idle", "running", "success", "failed" |
| `webui_changed` | boolean | yes | True when the webui store path changed during this update (browser reload needed) |


## bcachefs-tools

### `bcachefs.tools.info`

Return bcachefs-tools version info (pinned ref, running version, custom/default flag).

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `default_ref` | string | yes | The default ref from flake.nix (e.g. "v1.37.0") |
| `is_custom` | boolean | yes | True when the user has overridden the default bcachefs-tools version |
| `kernel_rust` | boolean | no | Whether the running kernel was built with Rust support (CONFIG_RUST=y) |
| `pinned_ref` | string | no | The ref in flake.lock original (e.g. "v1.37.0", "master", commit sha) |
| `pinned_rev` | string | no | The resolved full commit sha from flake.lock locked |
| `running_version` | string | yes | Output of `bcachefs version` |


### `bcachefs.tools.switch`

Switch bcachefs-tools to a specific git ref (tag, branch, or commit SHA). Runs `nixos-rebuild switch` in background.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `git_ref` | string | yes | A git ref: tag (v1.37.0), branch (master), or commit hash |


### `bcachefs.tools.status`

Return the current bcachefs-tools switch operation status.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `log` | string | yes |  |
| `reboot_required` | boolean | yes | True when the activated system has a different kernel than the booted one |
| `state` | string | yes | "idle", "running", "success", "failed" |
| `webui_changed` | boolean | yes | True when the webui store path changed during this update (browser reload needed) |


## Settings

### `system.settings.get`

Return current system settings.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `clock_24h` | boolean | no | Whether to display clocks in 24-hour format. |
| `hostname` | string | no | System hostname. |
| `timezone` | string | no | IANA timezone string applied to the system (e.g. `UTC`, `America/New_York`). |


### `system.settings.update`

Update system settings. Only provided fields are changed.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `clock_24h` | boolean | no | Whether to use 24-hour clock display (optional). |
| `hostname` | string | no | New hostname to set (optional). |
| `timezone` | string | no | New IANA timezone to apply (optional). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `clock_24h` | boolean | no | Whether to display clocks in 24-hour format. |
| `hostname` | string | no | System hostname. |
| `timezone` | string | no | IANA timezone string applied to the system (e.g. `UTC`, `America/New_York`). |


### `system.settings.timezones`

Return list of valid IANA timezone strings.

**Role:** `any`

**Returns:**

`string[]`


## Network

### `system.network.get`

Return current network configuration including live interface state.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `address` | string | no | Static IPv4 address (required when `dhcp` is false). |
| `dhcp` | boolean | yes | Whether DHCP is enabled; if false, static address/gateway are used. |
| `gateway` | string | no | Default gateway IPv4 address (required when `dhcp` is false). |
| `interface` | string | no | Network interface name to configure (e.g. `eth0`). Auto-detected if empty. |
| `live_addresses` | string[] | no | Currently assigned addresses on the interface in CIDR notation (read-only). |
| `live_gateway` | string | no | Currently active default gateway (read-only). |
| `nameservers` | string[] | no | DNS nameserver addresses written to `/etc/resolv.conf`. |
| `prefix_length` | integer | no | Subnet prefix length, e.g. `24` for a /24 (required when `dhcp` is false). |


### `system.network.update`

Update network configuration (DHCP or static). Applied immediately without rebooting.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `address` | string | no | Static IPv4 address (required when `dhcp` is false). |
| `dhcp` | boolean | yes | Whether DHCP is enabled; if false, static address/gateway are used. |
| `gateway` | string | no | Default gateway IPv4 address (required when `dhcp` is false). |
| `interface` | string | no | Network interface name to configure (e.g. `eth0`). Auto-detected if empty. |
| `live_addresses` | string[] | no | Currently assigned addresses on the interface in CIDR notation (read-only). |
| `live_gateway` | string | no | Currently active default gateway (read-only). |
| `nameservers` | string[] | no | DNS nameserver addresses written to `/etc/resolv.conf`. |
| `prefix_length` | integer | no | Subnet prefix length, e.g. `24` for a /24 (required when `dhcp` is false). |


## Protocols & Services

### `service.protocol.list`

List all protocols and their current status.

**Role:** `any`

**Returns:**

``ProtocolStatus`[]`


### `service.protocol.enable`

Enable a protocol service. Available names: `nfs`, `smb`, `iscsi`, `nvmeof`, `ssh`, `avahi`, `smart`.

**Role:** `admin`

**Params:** `{"name": string}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `display_name` | string | yes | Human-readable display name (e.g. `NFS`, `SMB`, `iSCSI`). |
| `enabled` | boolean | yes | Whether the protocol is enabled in persistent state. |
| `name` | string | yes | Machine-readable protocol identifier (e.g. `nfs`, `smb`, `iscsi`). |
| `running` | boolean | yes | Whether the protocol's systemd service is currently active. |
| `system_service` | boolean | yes | Whether this is a system-level service (SSH, Avahi, SMART) rather than a storage protocol. |


### `service.protocol.disable`

Disable a protocol service.

**Role:** `admin`

**Params:** `{"name": string}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `display_name` | string | yes | Human-readable display name (e.g. `NFS`, `SMB`, `iSCSI`). |
| `enabled` | boolean | yes | Whether the protocol is enabled in persistent state. |
| `name` | string | yes | Machine-readable protocol identifier (e.g. `nfs`, `smb`, `iscsi`). |
| `running` | boolean | yes | Whether the protocol's systemd service is currently active. |
| `system_service` | boolean | yes | Whether this is a system-level service (SSH, Avahi, SMART) rather than a storage protocol. |


## Alert Rules

### `alert.rules.list`

List all alert rules.

**Role:** `any`

**Returns:**

``AlertRule`[]`


### `alert.rules.create`

Create a new alert rule.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `condition` | `AlertCondition` | yes | Comparison operator applied between the metric value and the threshold. |
| `enabled` | boolean | yes | Whether the rule is active and evaluated. |
| `id` | string | yes | Unique rule identifier. |
| `metric` | `AlertMetric` | yes | The system metric this rule monitors. |
| `name` | string | yes | Human-readable rule name. |
| `severity` | `AlertSeverity` | yes | Severity level when the rule fires. |
| `threshold` | number | yes | Threshold value the metric is compared against. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `condition` | `AlertCondition` | yes | Comparison operator applied between the metric value and the threshold. |
| `enabled` | boolean | yes | Whether the rule is active and evaluated. |
| `id` | string | yes | Unique rule identifier. |
| `metric` | `AlertMetric` | yes | The system metric this rule monitors. |
| `name` | string | yes | Human-readable rule name. |
| `severity` | `AlertSeverity` | yes | Severity level when the rule fires. |
| `threshold` | number | yes | Threshold value the metric is compared against. |


### `alert.rules.update`

Update an existing alert rule. Only provided fields are changed.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `enabled` | boolean | no | Enable or disable the rule (optional). |
| `id` | string | yes | ID of the rule to update. |
| `name` | string | no | New name for the rule (optional). |
| `severity` | `AlertSeverity` \| null | no | New severity level (optional). |
| `threshold` | number | no | New threshold value (optional). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `condition` | `AlertCondition` | yes | Comparison operator applied between the metric value and the threshold. |
| `enabled` | boolean | yes | Whether the rule is active and evaluated. |
| `id` | string | yes | Unique rule identifier. |
| `metric` | `AlertMetric` | yes | The system metric this rule monitors. |
| `name` | string | yes | Human-readable rule name. |
| `severity` | `AlertSeverity` | yes | Severity level when the rule fires. |
| `threshold` | number | yes | Threshold value the metric is compared against. |


### `alert.rules.delete`

Delete an alert rule by ID.

**Role:** `admin`

**Params:** `{"id": string}`


## Block Devices

### `device.list`

List all block devices and partitions visible to the system.

**Role:** `any`

**Returns:**

``BlockDevice`[]`


### `device.wipe`

Erase all filesystem signatures from a device (wipefs). The device must not be in use.

**Role:** `admin`

**Params:** `{"path": string}`


## Pools

### `pool.list`

List all storage pools. Pool-scoped tokens see only their assigned pool.

**Role:** `any`

**Returns:**

``Pool`[]`


### `pool.get`

Get a single pool by name.

**Role:** `any`

**Params:** `{"name": string}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `PoolDevice`[] | yes | Member devices of the pool. |
| `mount_point` | string | no | Absolute path where the pool is mounted (e.g. `/storage/tank`). |
| `mounted` | boolean | yes | Whether the pool is currently mounted. |
| `name` | string | yes | Human-readable pool name, derived from the mount point directory. |
| `options` | `PoolOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `pool.create`

Format and mount a new bcachefs pool.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `background_target` | string | no | Target label for background migration. |
| `compression` | string | no | Inline compression algorithm (e.g. `lz4`, `zstd`, `none`). |
| `devices` | `DeviceSpec`[] | yes | Devices to include in the pool. |
| `encryption` | boolean | no | Whether to enable encryption at format time. |
| `erasure_code` | boolean | no | Whether to enable erasure coding. |
| `foreground_target` | string | no | Tiering targets set at format time. |
| `label` | string | no | Filesystem-wide label (used as default when no per-device labels set). |
| `metadata_target` | string | no | Target label for metadata placement. |
| `name` | string | yes | Name for the new pool; becomes the mount point directory under `/storage/`. |
| `promote_target` | string | no | Target label for data promotion (cache tier). |
| `replicas` | integer | no | Number of data replicas (default 1). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `PoolDevice`[] | yes | Member devices of the pool. |
| `mount_point` | string | no | Absolute path where the pool is mounted (e.g. `/storage/tank`). |
| `mounted` | boolean | yes | Whether the pool is currently mounted. |
| `name` | string | yes | Human-readable pool name, derived from the mount point directory. |
| `options` | `PoolOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `pool.destroy`

Unmount and unregister a pool. Does not wipe the devices.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `force` | boolean | no | If true, wipe bcachefs superblocks from all member devices after unmounting. |
| `name` | string | yes | Name of the pool to destroy. |


### `pool.mount`

Mount a known pool.

**Role:** `admin`

**Params:** `{"name": string}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `PoolDevice`[] | yes | Member devices of the pool. |
| `mount_point` | string | no | Absolute path where the pool is mounted (e.g. `/storage/tank`). |
| `mounted` | boolean | yes | Whether the pool is currently mounted. |
| `name` | string | yes | Human-readable pool name, derived from the mount point directory. |
| `options` | `PoolOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `pool.unmount`

Unmount a pool.

**Role:** `admin`

**Params:** `{"name": string}`


### `pool.options.update`

Update runtime-mutable bcachefs filesystem options (written to sysfs).

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `background_compression` | string | no | Background recompression algorithm. |
| `background_target` | string | no | Target label for background migration. |
| `compression` | string | no | Inline compression algorithm (e.g. `lz4`, `zstd`, `none`). |
| `erasure_code` | boolean | no | Whether to enable erasure coding. |
| `error_action` | string | no | Action on unrecoverable read errors (`continue`, `ro`, `panic`). |
| `foreground_target` | string | no | Target label for foreground (new) writes. |
| `metadata_target` | string | no | Target label for metadata placement. |
| `name` | string | yes | Name of the pool to update. |
| `promote_target` | string | no | Target label for data promotion (cache tier). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `PoolDevice`[] | yes | Member devices of the pool. |
| `mount_point` | string | no | Absolute path where the pool is mounted (e.g. `/storage/tank`). |
| `mounted` | boolean | yes | Whether the pool is currently mounted. |
| `name` | string | yes | Human-readable pool name, derived from the mount point directory. |
| `options` | `PoolOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `pool.usage`

Return detailed bcachefs `fs usage` breakdown.

**Role:** `any`

**Params:** `{"name": string}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `data_bytes` | integer | yes | Total data stored (before replication). |
| `devices` | `DeviceUsage`[] | yes | Per-device usage breakdown. |
| `metadata_bytes` | integer | yes | Total metadata stored. |
| `raw` | string | yes | Raw output from `bcachefs fs usage`, structured where possible. |
| `reserved_bytes` | integer | yes | Reserved bytes. |


### `pool.scrub.start`

Start a scrub on a mounted pool.

**Role:** `admin`

**Params:** `{"name": string}`


### `pool.scrub.status`

Return current scrub status.

**Role:** `any`

**Params:** `{"name": string}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `raw` | string | yes | Raw text output from the bcachefs scrub status command. |
| `running` | boolean | yes | Whether a scrub is currently in progress. |


### `pool.reconcile.status`

Return bcachefs background work (reconcile) status.

**Role:** `any`

**Params:** `{"name": string}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `raw` | string | yes | Raw text output from the bcachefs reconcile status command. |


### `bcachefs.usage`

Return raw `bcachefs fs usage` output for a pool.

**Role:** `any`

**Params:** `{"name": string}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `data_bytes` | integer | yes | Total data stored (before replication). |
| `devices` | `DeviceUsage`[] | yes | Per-device usage breakdown. |
| `metadata_bytes` | integer | yes | Total metadata stored. |
| `raw` | string | yes | Raw output from `bcachefs fs usage`, structured where possible. |
| `reserved_bytes` | integer | yes | Reserved bytes. |


## Pool Devices

### `pool.device.add`

Add a device to an existing mounted pool.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device` | `DeviceSpec` | yes | Device to add, with optional label and durability settings. |
| `pool` | string | yes | Name of the pool to add the device to. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `PoolDevice`[] | yes | Member devices of the pool. |
| `mount_point` | string | no | Absolute path where the pool is mounted (e.g. `/storage/tank`). |
| `mounted` | boolean | yes | Whether the pool is currently mounted. |
| `name` | string | yes | Human-readable pool name, derived from the mount point directory. |
| `options` | `PoolOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `pool.device.remove`

Remove a device from a pool. The device should be fully evacuated first.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device` | string | yes | Absolute path of the block device (e.g. `/dev/sdb`). |
| `pool` | string | yes | Name of the pool containing the device. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `PoolDevice`[] | yes | Member devices of the pool. |
| `mount_point` | string | no | Absolute path where the pool is mounted (e.g. `/storage/tank`). |
| `mounted` | boolean | yes | Whether the pool is currently mounted. |
| `name` | string | yes | Human-readable pool name, derived from the mount point directory. |
| `options` | `PoolOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `pool.device.evacuate`

Evacuate all data from a device to the remaining pool members. Long-running — returns `{"status": "started"}` immediately. Pool events are broadcast every 3s during evacuation.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device` | string | yes | Absolute path of the block device (e.g. `/dev/sdb`). |
| `pool` | string | yes | Name of the pool containing the device. |


### `pool.device.set_state`

Set persistent device state (`rw`, `ro`, `failed`, `spare`).

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device` | string | yes | Absolute path of the block device (e.g. `/dev/sdb`). |
| `pool` | string | yes | Name of the pool containing the device. |
| `state` | string | yes | One of: rw, ro, failed, spare |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `PoolDevice`[] | yes | Member devices of the pool. |
| `mount_point` | string | no | Absolute path where the pool is mounted (e.g. `/storage/tank`). |
| `mounted` | boolean | yes | Whether the pool is currently mounted. |
| `name` | string | yes | Human-readable pool name, derived from the mount point directory. |
| `options` | `PoolOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `pool.device.set_label`

Set or update the hierarchical label on a device in a mounted pool. Written live via sysfs.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device` | string | yes | Absolute path of the block device (e.g. `/dev/sdb`). |
| `label` | string | yes | New hierarchical label (e.g. `ssd.fast`, `hdd.archive`). |
| `pool` | string | yes | Name of the pool containing the device. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `PoolDevice`[] | yes | Member devices of the pool. |
| `mount_point` | string | no | Absolute path where the pool is mounted (e.g. `/storage/tank`). |
| `mounted` | boolean | yes | Whether the pool is currently mounted. |
| `name` | string | yes | Human-readable pool name, derived from the mount point directory. |
| `options` | `PoolOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `pool.device.online`

Bring a device back online.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device` | string | yes | Absolute path of the block device (e.g. `/dev/sdb`). |
| `pool` | string | yes | Name of the pool containing the device. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `PoolDevice`[] | yes | Member devices of the pool. |
| `mount_point` | string | no | Absolute path where the pool is mounted (e.g. `/storage/tank`). |
| `mounted` | boolean | yes | Whether the pool is currently mounted. |
| `name` | string | yes | Human-readable pool name, derived from the mount point directory. |
| `options` | `PoolOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `pool.device.offline`

Take a device offline.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device` | string | yes | Absolute path of the block device (e.g. `/dev/sdb`). |
| `pool` | string | yes | Name of the pool containing the device. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `PoolDevice`[] | yes | Member devices of the pool. |
| `mount_point` | string | no | Absolute path where the pool is mounted (e.g. `/storage/tank`). |
| `mounted` | boolean | yes | Whether the pool is currently mounted. |
| `name` | string | yes | Human-readable pool name, derived from the mount point directory. |
| `options` | `PoolOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


## Subvolumes

### `subvolume.list`

List subvolumes in a pool.

**Role:** `any`

**Params:** `{"pool": string}`

**Returns:**

``Subvolume`[]`


### `subvolume.list_all`

List all subvolumes across all pools.

**Role:** `any`

**Returns:**

``Subvolume`[]`


### `subvolume.get`

Get a single subvolume.

**Role:** `any`

**Params:** `{"pool": string, "name": string}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `name` | string | yes | Subvolume name (unique within the pool). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `pool` | string | yes | Name of the pool that contains this subvolume. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes (filesystem subvolumes only, from `du`). |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |


### `subvolume.create`

Create a new bcachefs subvolume (filesystem or block-backed).

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `comments` | string | no | Optional description for the subvolume. |
| `compression` | string | no | Compression algorithm to set on the subvolume (e.g. `lz4`, `zstd`). |
| `name` | string | yes | Name for the new subvolume. |
| `pool` | string | yes | Name of the pool to create the subvolume in. |
| `subvolume_type` | `SubvolumeType` | no | Whether to create a filesystem or block-backed subvolume (default: filesystem). |
| `volsize_bytes` | integer | no | Size of the block backing image in bytes (required for block subvolumes). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `name` | string | yes | Subvolume name (unique within the pool). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `pool` | string | yes | Name of the pool that contains this subvolume. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes (filesystem subvolumes only, from `du`). |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |


### `subvolume.delete`

Delete a subvolume and all its snapshots.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Name of the subvolume to delete. |
| `pool` | string | yes | Name of the pool containing the subvolume. |


### `subvolume.attach`

Attach the loop device for a block subvolume (mounts `vol.img` via losetup).

**Role:** `operator`

**Params:** `{"pool": string, "name": string}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `name` | string | yes | Subvolume name (unique within the pool). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `pool` | string | yes | Name of the pool that contains this subvolume. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes (filesystem subvolumes only, from `du`). |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |


### `subvolume.detach`

Detach the loop device for a block subvolume.

**Role:** `operator`

**Params:** `{"pool": string, "name": string}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `name` | string | yes | Subvolume name (unique within the pool). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `pool` | string | yes | Name of the pool that contains this subvolume. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes (filesystem subvolumes only, from `du`). |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |


### `subvolume.resize`

Resize a block subvolume's backing image.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Name of the block subvolume to resize. |
| `pool` | string | yes | Name of the pool containing the subvolume. |
| `volsize_bytes` | integer | yes | New size of the backing sparse image in bytes. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `name` | string | yes | Subvolume name (unique within the pool). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `pool` | string | yes | Name of the pool that contains this subvolume. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes (filesystem subvolumes only, from `du`). |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |


### `subvolume.set_properties`

Set arbitrary key-value metadata on a subvolume (stored as POSIX xattrs in the `user.*` namespace). Used by the CSI driver.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Name of the subvolume to update. |
| `pool` | string | yes | Name of the pool containing the subvolume. |
| `properties` | object | yes | Key-value pairs to set (merged with existing properties). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `name` | string | yes | Subvolume name (unique within the pool). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `pool` | string | yes | Name of the pool that contains this subvolume. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes (filesystem subvolumes only, from `du`). |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |


### `subvolume.remove_properties`

Remove specific metadata keys from a subvolume.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `keys` | string[] | yes | Property keys to remove. |
| `name` | string | yes | Name of the subvolume to update. |
| `pool` | string | yes | Name of the pool containing the subvolume. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `name` | string | yes | Subvolume name (unique within the pool). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `pool` | string | yes | Name of the pool that contains this subvolume. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes (filesystem subvolumes only, from `du`). |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |


### `subvolume.find_by_property`

Find subvolumes matching a specific metadata key-value pair.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `key` | string | yes | xattr property key to match against. |
| `pool` | string | no | Optional pool to restrict the search to. |
| `value` | string | yes | Value that the property key must equal. |

**Returns:**

``Subvolume`[]`


## Snapshots

### `snapshot.list`

List snapshots for all subvolumes in a pool.

**Role:** `any`

**Params:** `{"pool": string}`

**Returns:**

``Snapshot`[]`


### `snapshot.create`

Create a snapshot of a subvolume.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Name for the new snapshot. |
| `pool` | string | yes | Name of the pool containing the subvolume. |
| `read_only` | boolean | no | Whether to create a read-only snapshot (default: true). |
| `subvolume` | string | yes | Name of the subvolume to snapshot. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `block_device` | string | no | Loop device path if this snapshot's vol.img is currently attached (block snapshots only). |
| `name` | string | yes | Snapshot name (unique within the parent subvolume). |
| `path` | string | yes | Absolute filesystem path to the snapshot directory. |
| `pool` | string | yes | Name of the pool that contains this snapshot. |
| `read_only` | boolean | yes | Whether this snapshot is read-only. |
| `subvolume` | string | yes | Name of the parent subvolume. |


### `snapshot.delete`

Delete a snapshot.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Name of the snapshot to delete. |
| `pool` | string | yes | Name of the pool containing the snapshot. |
| `subvolume` | string | yes | Name of the parent subvolume. |


### `snapshot.clone`

Clone a snapshot into a new independent subvolume.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `new_name` | string | yes | Name for the new writable subvolume created from the snapshot. |
| `pool` | string | yes | Name of the pool containing the snapshot. |
| `snapshot` | string | yes | Name of the snapshot to clone. |
| `subvolume` | string | yes | Name of the parent subvolume. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `name` | string | yes | Subvolume name (unique within the pool). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `pool` | string | yes | Name of the pool that contains this subvolume. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes (filesystem subvolumes only, from `du`). |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |


## NFS Shares

### `share.nfs.list`

List all NFS shares.

**Role:** `any`

**Returns:**

``NfsShare`[]`


### `share.nfs.get`

Get an NFS share by ID.

**Role:** `any`

**Params:** `{"id": string}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `clients` | `NfsClient`[] | yes | List of allowed clients and their export options. |
| `comment` | string | no | Optional description of the share. |
| `enabled` | boolean | yes | Whether the share is currently active in `/etc/exports.d/nasty.exports`. |
| `id` | string | yes | Unique share identifier (UUID). |
| `path` | string | yes | Absolute filesystem path being exported (must be under `/storage/`). |


### `share.nfs.create`

Create an NFS share.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `clients` | `NfsClient`[] | yes | Allowed clients and their export options. |
| `comment` | string | no | Optional description. |
| `enabled` | boolean | no | Whether to enable the share immediately (default: true). |
| `path` | string | yes | Absolute path to export (must exist and be under `/storage/`). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `clients` | `NfsClient`[] | yes | List of allowed clients and their export options. |
| `comment` | string | no | Optional description of the share. |
| `enabled` | boolean | yes | Whether the share is currently active in `/etc/exports.d/nasty.exports`. |
| `id` | string | yes | Unique share identifier (UUID). |
| `path` | string | yes | Absolute filesystem path being exported (must be under `/storage/`). |


### `share.nfs.update`

Update an NFS share.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `clients` | `NfsClient`[] | no | Replacement client list (optional; replaces entire list when provided). |
| `comment` | string | no | New description (optional). |
| `enabled` | boolean | no | Enable or disable the share (optional). |
| `id` | string | yes | ID of the share to update. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `clients` | `NfsClient`[] | yes | List of allowed clients and their export options. |
| `comment` | string | no | Optional description of the share. |
| `enabled` | boolean | yes | Whether the share is currently active in `/etc/exports.d/nasty.exports`. |
| `id` | string | yes | Unique share identifier (UUID). |
| `path` | string | yes | Absolute filesystem path being exported (must be under `/storage/`). |


### `share.nfs.delete`

Delete an NFS share.

**Role:** `admin`

**Params:** `{"id": string}`


## SMB Shares

### `share.smb.list`

List all SMB shares.

**Role:** `any`

**Returns:**

``SmbShare`[]`


### `share.smb.get`

Get an SMB share by ID.

**Role:** `any`

**Params:** `{"id": string}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `browseable` | boolean | yes | Whether the share is visible in network browse lists. |
| `comment` | string | no | Optional description shown in share listings. |
| `enabled` | boolean | yes | Whether the share is active in `smb.nasty.conf`. |
| `extra_params` | object | yes | Additional raw Samba parameters written to the share section. |
| `guest_ok` | boolean | yes | Whether unauthenticated guest access is allowed. |
| `id` | string | yes | Unique share identifier (UUID). |
| `name` | string | yes | Samba share name used in `\\server\name` UNC paths. |
| `path` | string | yes | Absolute filesystem path being shared (must be under `/storage/`). |
| `read_only` | boolean | yes | Whether the share is read-only. |
| `valid_users` | string[] | yes | Usernames allowed to connect (empty means no restriction beyond authentication). |


### `share.smb.create`

Create an SMB share.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `browseable` | boolean | no | Whether the share appears in browse lists (default: true). |
| `comment` | string | no | Optional description. |
| `enabled` | boolean | no | Whether to enable the share immediately (default: true). |
| `extra_params` | object | no | Additional raw Samba parameters for this share section. |
| `guest_ok` | boolean | no | Whether guest access is allowed (default: false). |
| `name` | string | yes | Samba share name (1–80 characters, no special characters). |
| `path` | string | yes | Absolute path to share (must exist and be under `/storage/`). |
| `read_only` | boolean | no | Whether the share is read-only (default: false). |
| `valid_users` | string[] | no | Allowed usernames; empty means no per-user restriction. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `browseable` | boolean | yes | Whether the share is visible in network browse lists. |
| `comment` | string | no | Optional description shown in share listings. |
| `enabled` | boolean | yes | Whether the share is active in `smb.nasty.conf`. |
| `extra_params` | object | yes | Additional raw Samba parameters written to the share section. |
| `guest_ok` | boolean | yes | Whether unauthenticated guest access is allowed. |
| `id` | string | yes | Unique share identifier (UUID). |
| `name` | string | yes | Samba share name used in `\\server\name` UNC paths. |
| `path` | string | yes | Absolute filesystem path being shared (must be under `/storage/`). |
| `read_only` | boolean | yes | Whether the share is read-only. |
| `valid_users` | string[] | yes | Usernames allowed to connect (empty means no restriction beyond authentication). |


### `share.smb.update`

Update an SMB share.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `browseable` | boolean | no | Update browseable flag (optional). |
| `comment` | string | no | New description (optional). |
| `enabled` | boolean | no | Enable or disable the share (optional). |
| `extra_params` | object | no | Replacement extra Samba parameters (optional). |
| `guest_ok` | boolean | no | Update guest access flag (optional). |
| `id` | string | yes | ID of the share to update. |
| `name` | string | no | New share name (optional; must be unique). |
| `read_only` | boolean | no | Update read-only flag (optional). |
| `valid_users` | string[] | no | Replacement allowed-users list (optional). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `browseable` | boolean | yes | Whether the share is visible in network browse lists. |
| `comment` | string | no | Optional description shown in share listings. |
| `enabled` | boolean | yes | Whether the share is active in `smb.nasty.conf`. |
| `extra_params` | object | yes | Additional raw Samba parameters written to the share section. |
| `guest_ok` | boolean | yes | Whether unauthenticated guest access is allowed. |
| `id` | string | yes | Unique share identifier (UUID). |
| `name` | string | yes | Samba share name used in `\\server\name` UNC paths. |
| `path` | string | yes | Absolute filesystem path being shared (must be under `/storage/`). |
| `read_only` | boolean | yes | Whether the share is read-only. |
| `valid_users` | string[] | yes | Usernames allowed to connect (empty means no restriction beyond authentication). |


### `share.smb.delete`

Delete an SMB share.

**Role:** `admin`

**Params:** `{"id": string}`


## iSCSI Targets

### `share.iscsi.list`

List all iSCSI targets.

**Role:** `any`

**Returns:**

``IscsiTarget`[]`


### `share.iscsi.get`

Get an iSCSI target by ID.

**Role:** `any`

**Params:** `{"id": string}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `acls` | `Acl`[] | yes | Initiator ACL entries controlling which hosts may connect. |
| `alias` | string | no | Optional human-readable alias for the target. |
| `enabled` | boolean | yes | Whether the target is currently active in LIO. |
| `id` | string | yes | Unique target identifier (UUID). |
| `iqn` | string | yes | iSCSI Qualified Name (e.g. `iqn.2137-04.storage.nasty:tank-vol`). |
| `luns` | `Lun`[] | yes | Logical units exposed by this target. |
| `portals` | `Portal`[] | yes | Network portals (IP:port) the target listens on. |


### `share.iscsi.create`

Create an iSCSI target. Optionally attach a LUN and ACLs in one call.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Short name used to generate the IQN: iqn.2137-01.com.nasty:<name> |
| `alias` | string | no | Optional human-readable alias for the target. |
| `portals` | `Portal`[] | no | Defaults to 0.0.0.0:3260 |
| `device_path` | string | no | Block device path (e.g. /dev/loop0). When provided, a LUN is created automatically. |
| `acls` | `AclEntry`[] | no | Initiator ACLs. When provided, only these initiators may connect. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `acls` | `Acl`[] | yes | Initiator ACL entries controlling which hosts may connect. |
| `alias` | string | no | Optional human-readable alias for the target. |
| `enabled` | boolean | yes | Whether the target is currently active in LIO. |
| `id` | string | yes | Unique target identifier (UUID). |
| `iqn` | string | yes | iSCSI Qualified Name (e.g. `iqn.2137-04.storage.nasty:tank-vol`). |
| `luns` | `Lun`[] | yes | Logical units exposed by this target. |
| `portals` | `Portal`[] | yes | Network portals (IP:port) the target listens on. |


### `share.iscsi.delete`

Delete an iSCSI target.

**Role:** `admin`

**Params:** `{"id": string}`


### `share.iscsi.add_lun`

Add a LUN to an iSCSI target.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `backstore_path` | string | yes | Block device path (/dev/sdb) or file path (/mnt/nasty/pool/disk.img) |
| `backstore_type` | string | no | "block" or "fileio" — auto-detected if omitted |
| `size_bytes` | integer | no | Required for fileio if file doesn't exist yet |
| `target_id` | string | yes |  |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `acls` | `Acl`[] | yes | Initiator ACL entries controlling which hosts may connect. |
| `alias` | string | no | Optional human-readable alias for the target. |
| `enabled` | boolean | yes | Whether the target is currently active in LIO. |
| `id` | string | yes | Unique target identifier (UUID). |
| `iqn` | string | yes | iSCSI Qualified Name (e.g. `iqn.2137-04.storage.nasty:tank-vol`). |
| `luns` | `Lun`[] | yes | Logical units exposed by this target. |
| `portals` | `Portal`[] | yes | Network portals (IP:port) the target listens on. |


### `share.iscsi.remove_lun`

Remove a LUN from an iSCSI target.

**Role:** `admin`

**Params:** `{"target_id": string, "lun_id": integer}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `acls` | `Acl`[] | yes | Initiator ACL entries controlling which hosts may connect. |
| `alias` | string | no | Optional human-readable alias for the target. |
| `enabled` | boolean | yes | Whether the target is currently active in LIO. |
| `id` | string | yes | Unique target identifier (UUID). |
| `iqn` | string | yes | iSCSI Qualified Name (e.g. `iqn.2137-04.storage.nasty:tank-vol`). |
| `luns` | `Lun`[] | yes | Logical units exposed by this target. |
| `portals` | `Portal`[] | yes | Network portals (IP:port) the target listens on. |


### `share.iscsi.add_acl`

Allow an iSCSI initiator IQN to connect.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `initiator_iqn` | string | yes | Initiator IQN to allow. |
| `password` | string | no | Optional CHAP password for this initiator. |
| `target_id` | string | yes | ID of the target to add the ACL to. |
| `userid` | string | no | Optional CHAP username for this initiator. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `acls` | `Acl`[] | yes | Initiator ACL entries controlling which hosts may connect. |
| `alias` | string | no | Optional human-readable alias for the target. |
| `enabled` | boolean | yes | Whether the target is currently active in LIO. |
| `id` | string | yes | Unique target identifier (UUID). |
| `iqn` | string | yes | iSCSI Qualified Name (e.g. `iqn.2137-04.storage.nasty:tank-vol`). |
| `luns` | `Lun`[] | yes | Logical units exposed by this target. |
| `portals` | `Portal`[] | yes | Network portals (IP:port) the target listens on. |


### `share.iscsi.remove_acl`

Remove an iSCSI initiator ACL.

**Role:** `admin`

**Params:** `{"target_id": string, "initiator_iqn": string}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `acls` | `Acl`[] | yes | Initiator ACL entries controlling which hosts may connect. |
| `alias` | string | no | Optional human-readable alias for the target. |
| `enabled` | boolean | yes | Whether the target is currently active in LIO. |
| `id` | string | yes | Unique target identifier (UUID). |
| `iqn` | string | yes | iSCSI Qualified Name (e.g. `iqn.2137-04.storage.nasty:tank-vol`). |
| `luns` | `Lun`[] | yes | Logical units exposed by this target. |
| `portals` | `Portal`[] | yes | Network portals (IP:port) the target listens on. |


## NVMe-oF Subsystems

### `share.nvmeof.list`

List all NVMe-oF subsystems.

**Role:** `any`

**Returns:**

``NvmeofSubsystem`[]`


### `share.nvmeof.get`

Get an NVMe-oF subsystem by ID.

**Role:** `any`

**Params:** `{"id": string}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `allow_any_host` | boolean | yes | Whether any host NQN is permitted to connect without explicit ACL entries. |
| `allowed_hosts` | string[] | yes | NQNs of hosts explicitly allowed to connect (used when `allow_any_host` is false). |
| `enabled` | boolean | yes | Whether this subsystem is active in nvmet configfs. |
| `id` | string | yes | Unique subsystem identifier (UUID). |
| `namespaces` | `Namespace`[] | yes | Block device namespaces exposed by this subsystem. |
| `nqn` | string | yes | NVMe Qualified Name (e.g. `nqn.2137-04.storage.nasty:tank-vol`). |
| `ports` | `Port`[] | yes | Transport ports this subsystem is reachable on. |


### `share.nvmeof.create`

Create an NVMe-oF subsystem. Optionally attach a namespace, port, and host ACLs in one call.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Short name appended to NQN prefix |
| `allow_any_host` | boolean | no | Whether any host NQN is permitted to connect (default: true). |
| `device_path` | string | no | Block device path (e.g. /dev/loop0). When provided, a namespace and port are created automatically. |
| `addr` | string | no | Listen address (default 0.0.0.0). Only used when `device_path` is set. |
| `port` | integer | no | Port number (default 4420). Only used when `device_path` is set. |
| `allowed_hosts` | string[] | no | Host NQNs to allow. When provided, `allow_any_host` is set to false. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `allow_any_host` | boolean | yes | Whether any host NQN is permitted to connect without explicit ACL entries. |
| `allowed_hosts` | string[] | yes | NQNs of hosts explicitly allowed to connect (used when `allow_any_host` is false). |
| `enabled` | boolean | yes | Whether this subsystem is active in nvmet configfs. |
| `id` | string | yes | Unique subsystem identifier (UUID). |
| `namespaces` | `Namespace`[] | yes | Block device namespaces exposed by this subsystem. |
| `nqn` | string | yes | NVMe Qualified Name (e.g. `nqn.2137-04.storage.nasty:tank-vol`). |
| `ports` | `Port`[] | yes | Transport ports this subsystem is reachable on. |


### `share.nvmeof.delete`

Delete an NVMe-oF subsystem.

**Role:** `admin`

**Params:** `{"id": string}`


### `share.nvmeof.add_namespace`

Add a namespace (block device) to a subsystem.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device_path` | string | yes | Block device path (e.g. /dev/sdc) |
| `subsystem_id` | string | yes | ID of the subsystem to add the namespace to. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `allow_any_host` | boolean | yes | Whether any host NQN is permitted to connect without explicit ACL entries. |
| `allowed_hosts` | string[] | yes | NQNs of hosts explicitly allowed to connect (used when `allow_any_host` is false). |
| `enabled` | boolean | yes | Whether this subsystem is active in nvmet configfs. |
| `id` | string | yes | Unique subsystem identifier (UUID). |
| `namespaces` | `Namespace`[] | yes | Block device namespaces exposed by this subsystem. |
| `nqn` | string | yes | NVMe Qualified Name (e.g. `nqn.2137-04.storage.nasty:tank-vol`). |
| `ports` | `Port`[] | yes | Transport ports this subsystem is reachable on. |


### `share.nvmeof.remove_namespace`

Remove a namespace from a subsystem.

**Role:** `admin`

**Params:** `{"subsystem_id": string, "nsid": integer}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `allow_any_host` | boolean | yes | Whether any host NQN is permitted to connect without explicit ACL entries. |
| `allowed_hosts` | string[] | yes | NQNs of hosts explicitly allowed to connect (used when `allow_any_host` is false). |
| `enabled` | boolean | yes | Whether this subsystem is active in nvmet configfs. |
| `id` | string | yes | Unique subsystem identifier (UUID). |
| `namespaces` | `Namespace`[] | yes | Block device namespaces exposed by this subsystem. |
| `nqn` | string | yes | NVMe Qualified Name (e.g. `nqn.2137-04.storage.nasty:tank-vol`). |
| `ports` | `Port`[] | yes | Transport ports this subsystem is reachable on. |


### `share.nvmeof.add_port`

Add a transport port to a subsystem.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `addr` | string | no | Listening IP address (default `0.0.0.0`). |
| `addr_family` | string | no | Address family (`ipv4` or `ipv6`; default `ipv4`). |
| `service_id` | integer | no | Port number (default 4420) |
| `subsystem_id` | string | yes | ID of the subsystem to add the port to. |
| `transport` | string | no | "tcp" or "rdma" |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `allow_any_host` | boolean | yes | Whether any host NQN is permitted to connect without explicit ACL entries. |
| `allowed_hosts` | string[] | yes | NQNs of hosts explicitly allowed to connect (used when `allow_any_host` is false). |
| `enabled` | boolean | yes | Whether this subsystem is active in nvmet configfs. |
| `id` | string | yes | Unique subsystem identifier (UUID). |
| `namespaces` | `Namespace`[] | yes | Block device namespaces exposed by this subsystem. |
| `nqn` | string | yes | NVMe Qualified Name (e.g. `nqn.2137-04.storage.nasty:tank-vol`). |
| `ports` | `Port`[] | yes | Transport ports this subsystem is reachable on. |


### `share.nvmeof.remove_port`

Remove a transport port from a subsystem.

**Role:** `admin`

**Params:** `{"subsystem_id": string, "port_id": integer}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `allow_any_host` | boolean | yes | Whether any host NQN is permitted to connect without explicit ACL entries. |
| `allowed_hosts` | string[] | yes | NQNs of hosts explicitly allowed to connect (used when `allow_any_host` is false). |
| `enabled` | boolean | yes | Whether this subsystem is active in nvmet configfs. |
| `id` | string | yes | Unique subsystem identifier (UUID). |
| `namespaces` | `Namespace`[] | yes | Block device namespaces exposed by this subsystem. |
| `nqn` | string | yes | NVMe Qualified Name (e.g. `nqn.2137-04.storage.nasty:tank-vol`). |
| `ports` | `Port`[] | yes | Transport ports this subsystem is reachable on. |


### `share.nvmeof.add_host`

Allow a host NQN to connect to a subsystem.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `host_nqn` | string | yes | NQN of the host to allow. |
| `subsystem_id` | string | yes | ID of the subsystem to which to grant access. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `allow_any_host` | boolean | yes | Whether any host NQN is permitted to connect without explicit ACL entries. |
| `allowed_hosts` | string[] | yes | NQNs of hosts explicitly allowed to connect (used when `allow_any_host` is false). |
| `enabled` | boolean | yes | Whether this subsystem is active in nvmet configfs. |
| `id` | string | yes | Unique subsystem identifier (UUID). |
| `namespaces` | `Namespace`[] | yes | Block device namespaces exposed by this subsystem. |
| `nqn` | string | yes | NVMe Qualified Name (e.g. `nqn.2137-04.storage.nasty:tank-vol`). |
| `ports` | `Port`[] | yes | Transport ports this subsystem is reachable on. |


### `share.nvmeof.remove_host`

Disallow a host NQN from a subsystem.

**Role:** `admin`

**Params:** `{"subsystem_id": string, "nqn": string}`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `allow_any_host` | boolean | yes | Whether any host NQN is permitted to connect without explicit ACL entries. |
| `allowed_hosts` | string[] | yes | NQNs of hosts explicitly allowed to connect (used when `allow_any_host` is false). |
| `enabled` | boolean | yes | Whether this subsystem is active in nvmet configfs. |
| `id` | string | yes | Unique subsystem identifier (UUID). |
| `namespaces` | `Namespace`[] | yes | Block device namespaces exposed by this subsystem. |
| `nqn` | string | yes | NVMe Qualified Name (e.g. `nqn.2137-04.storage.nasty:tank-vol`). |
| `ports` | `Port`[] | yes | Transport ports this subsystem is reachable on. |


---

## Object Definitions

### `Acl`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `initiator_iqn` | string | yes | Initiator IQN allowed to connect |
| `password` | string | no | CHAP password for this initiator (optional). |
| `userid` | string | no | CHAP username for this initiator (optional). |

### `ActiveAlert`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `current_value` | number | yes | Current metric value at the time the alert was evaluated. |
| `message` | string | yes | Human-readable description of the alert condition. |
| `metric` | `AlertMetric` | yes | Metric that triggered the alert. |
| `rule_id` | string | yes | ID of the rule that triggered this alert. |
| `rule_name` | string | yes | Name of the rule that triggered this alert. |
| `severity` | `AlertSeverity` | yes | Severity level of the alert. |
| `source` | string | yes | Identifier of the specific resource that triggered the alert (e.g. pool name, device path). |
| `threshold` | number | yes | Threshold value configured in the rule. |

### `AlertCondition`

Enum: `above`, `below`, `equals`

### `AlertMetric`

Enum: `pool_usage_percent`, `cpu_load_percent`, `memory_usage_percent`, `disk_temperature`, `smart_health`, `swap_usage_percent`

### `AlertRule`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `condition` | `AlertCondition` | yes | Comparison operator applied between the metric value and the threshold. |
| `enabled` | boolean | yes | Whether the rule is active and evaluated. |
| `id` | string | yes | Unique rule identifier. |
| `metric` | `AlertMetric` | yes | The system metric this rule monitors. |
| `name` | string | yes | Human-readable rule name. |
| `severity` | `AlertSeverity` | yes | Severity level when the rule fires. |
| `threshold` | number | yes | Threshold value the metric is compared against. |

### `AlertSeverity`

Enum: `warning`, `critical`

### `ApiTokenInfo`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `created_at` | integer | yes | Unix timestamp (seconds) when the token was created. |
| `expires_at` | integer | no | Unix timestamp after which the token is rejected. None = never expires. |
| `id` | string | yes | Unique token identifier. |
| `name` | string | yes | Human-readable token name. |
| `pool` | string | no | Pool this token is scoped to, if any. |
| `role` | `Role` | yes | Role assigned to this token. |

### `BlockDevice`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `dev_type` | string | yes | lsblk device type: `disk` or `part`. |
| `device_class` | string | yes | Device speed class: "nvme", "ssd", or "hdd". |
| `fs_type` | string | no | Filesystem type detected on the device (e.g. `bcachefs`, `ext4`). |
| `in_use` | boolean | yes | Whether the device is currently in use (mounted, in a pool, or has partitions in use). |
| `mount_point` | string | no | Current mount point, if mounted. |
| `path` | string | yes | Absolute path of the block device (e.g. `/dev/sda`). |
| `rotational` | boolean | yes | Whether the underlying disk spins (false for NVMe/SSD, true for HDD). |
| `size_bytes` | integer | yes | Total capacity in bytes. |

### `CpuStats`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `count` | integer | yes | Number of logical CPU cores. |
| `load_1` | number | yes | 1-minute load average. |
| `load_15` | number | yes | 15-minute load average. |
| `load_5` | number | yes | 5-minute load average. |

### `DeviceSpec`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `durability` | integer | no | Durability: 0 = cache, 1 = normal, 2 = hardware RAID. |
| `label` | string | no | Hierarchical label (e.g. "ssd.fast", "hdd.archive"). |
| `path` | string | yes | Absolute block device path (e.g. `/dev/sda`). |

### `DeviceUsage`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `free_bytes` | integer | yes | Bytes available on this device. |
| `path` | string | yes | Block device path. |
| `total_bytes` | integer | yes | Total capacity of this device in bytes. |
| `used_bytes` | integer | yes | Bytes currently used on this device. |

### `DiskHealth`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `attributes` | `SmartAttribute`[] | yes | ATA SMART attribute table (may be empty for NVMe drives). |
| `capacity_bytes` | integer | yes | Total drive capacity in bytes. |
| `device` | string | yes | Block device path (e.g. `/dev/sda`). |
| `firmware` | string | yes | Drive firmware version string. |
| `health_passed` | boolean | yes | Whether the SMART overall-health self-assessment test passed. |
| `model` | string | yes | Drive model name reported by SMART. |
| `power_on_hours` | integer | no | Accumulated powered-on time in hours. |
| `serial` | string | yes | Drive serial number. |
| `smart_status` | string | yes | Human-readable SMART health status (`PASSED` or `FAILED`). |
| `temperature_c` | integer | no | Current drive temperature in degrees Celsius. |

### `DiskIoStats`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `io_in_progress` | integer | yes | Number of I/O operations currently in progress. |
| `name` | string | yes | Kernel device name (e.g. `sda`, `nvme0n1`). |
| `read_bytes` | integer | yes | Cumulative bytes read since boot (from `/proc/diskstats`). |
| `read_ios` | integer | yes | Cumulative read I/O operations completed since boot. |
| `write_bytes` | integer | yes | Cumulative bytes written since boot. |
| `write_ios` | integer | yes | Cumulative write I/O operations completed since boot. |

### `IscsiTarget`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `acls` | `Acl`[] | yes | Initiator ACL entries controlling which hosts may connect. |
| `alias` | string | no | Optional human-readable alias for the target. |
| `enabled` | boolean | yes | Whether the target is currently active in LIO. |
| `id` | string | yes | Unique target identifier (UUID). |
| `iqn` | string | yes | iSCSI Qualified Name (e.g. `iqn.2137-04.storage.nasty:tank-vol`). |
| `luns` | `Lun`[] | yes | Logical units exposed by this target. |
| `portals` | `Portal`[] | yes | Network portals (IP:port) the target listens on. |

### `Lun`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `backstore_name` | string | yes | LIO backstore name (auto-generated) |
| `backstore_path` | string | yes | Path to block device or file used as backstore |
| `backstore_type` | string | yes | "block" or "fileio" |
| `lun_id` | integer | yes |  |
| `size_bytes` | integer | no |  |

### `MemoryStats`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | RAM available for allocation without swapping. |
| `swap_total_bytes` | integer | yes | Total swap space in bytes. |
| `swap_used_bytes` | integer | yes | Swap space currently in use. |
| `total_bytes` | integer | yes | Total installed RAM in bytes. |
| `used_bytes` | integer | yes | RAM currently in use (total minus available). |

### `Namespace`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device_path` | string | yes | Block device path backing this namespace (e.g. `/dev/loop0`). |
| `enabled` | boolean | yes | Whether the namespace is enabled in configfs. |
| `nsid` | integer | yes | Namespace ID (1-based, auto-assigned). |

### `NetIfStats`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `addresses` | string[] | yes | IPv4 and IPv6 addresses in CIDR notation (e.g. `192.168.1.10/24`). |
| `name` | string | yes | Network interface name (e.g. `eth0`, `ens3`). |
| `rx_bytes` | integer | yes | Cumulative bytes received since boot. |
| `rx_packets` | integer | yes | Cumulative packets received since boot. |
| `speed_mbps` | integer | no | Link speed in Mbit/s (None if unavailable, e.g. virtual interfaces). |
| `tx_bytes` | integer | yes | Cumulative bytes transmitted since boot. |
| `tx_packets` | integer | yes | Cumulative packets transmitted since boot. |
| `up` | boolean | yes | Whether the interface's operstate is `up`. |

### `NfsClient`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `host` | string | yes | Network or host: "192.168.1.0/24", "10.0.0.5", "*" |
| `options` | string | yes | NFS export options: "rw,sync,no_subtree_check,no_root_squash" |

### `NfsShare`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `clients` | `NfsClient`[] | yes | List of allowed clients and their export options. |
| `comment` | string | no | Optional description of the share. |
| `enabled` | boolean | yes | Whether the share is currently active in `/etc/exports.d/nasty.exports`. |
| `id` | string | yes | Unique share identifier (UUID). |
| `path` | string | yes | Absolute filesystem path being exported (must be under `/storage/`). |

### `NvmeofSubsystem`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `allow_any_host` | boolean | yes | Whether any host NQN is permitted to connect without explicit ACL entries. |
| `allowed_hosts` | string[] | yes | NQNs of hosts explicitly allowed to connect (used when `allow_any_host` is false). |
| `enabled` | boolean | yes | Whether this subsystem is active in nvmet configfs. |
| `id` | string | yes | Unique subsystem identifier (UUID). |
| `namespaces` | `Namespace`[] | yes | Block device namespaces exposed by this subsystem. |
| `nqn` | string | yes | NVMe Qualified Name (e.g. `nqn.2137-04.storage.nasty:tank-vol`). |
| `ports` | `Port`[] | yes | Transport ports this subsystem is reachable on. |

### `Pool`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `PoolDevice`[] | yes | Member devices of the pool. |
| `mount_point` | string | no | Absolute path where the pool is mounted (e.g. `/storage/tank`). |
| `mounted` | boolean | yes | Whether the pool is currently mounted. |
| `name` | string | yes | Human-readable pool name, derived from the mount point directory. |
| `options` | `PoolOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |

### `PoolDevice`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `data_allowed` | string | no | Which data types are allowed on this device (e.g. "journal,btree,user"). |
| `discard` | boolean | no | Whether TRIM/discard is enabled on this device. |
| `durability` | integer | no | How many replicas a copy on this device counts for.
0 = cache only, 1 = normal (default), 2 = hardware RAID. |
| `has_data` | string | no | Which data types are currently stored on this device (e.g. "btree,user"). |
| `label` | string | no | Hierarchical label (e.g. "ssd.fast", "hdd.archive").
Used for target-based tiering. |
| `path` | string | yes |  |
| `state` | string | no | Persistent device state: rw, ro, evacuating, spare. |

### `PoolOptions`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `background_compression` | string | no | Background recompression algorithm applied by the background worker. |
| `background_target` | string | no | Target label for background migration writes. |
| `compression` | string | no | Foreground (inline) compression algorithm (e.g. `lz4`, `zstd`, `none`). |
| `data_checksum` | string | no | Checksum algorithm for data (e.g. `crc32c`, `xxhash`). |
| `data_replicas` | integer | no | Number of replicas for data extents. |
| `encrypted` | boolean | no | Whether the filesystem is encrypted at rest. |
| `erasure_code` | boolean | no | Whether erasure coding (EC) is enabled on the filesystem. |
| `error_action` | string | no | Action on unrecoverable read errors (`continue`, `ro`, `panic`). |
| `foreground_target` | string | no | Target label for foreground (new) writes. |
| `metadata_checksum` | string | no | Checksum algorithm for metadata. |
| `metadata_replicas` | integer | no | Number of replicas for metadata (btree) extents. |
| `metadata_target` | string | no | Target label for metadata placement. |
| `promote_target` | string | no | Target label for data promotion (cache tier). |

### `Port`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `addr` | string | yes | Listening IP address (e.g. `0.0.0.0` for all interfaces). |
| `addr_family` | string | yes | Address family (`ipv4` or `ipv6`). |
| `port_id` | integer | yes | nvmet configfs port ID (unique across all subsystems on this host). |
| `service_id` | string | yes | TCP/RDMA port number as a string (default NVMe-oF port is `4420`). |
| `transport` | string | yes | Transport type (e.g. `tcp`, `rdma`). |

### `Portal`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `ip` | string | yes | IP address the portal listens on (use `0.0.0.0` for all interfaces). |
| `port` | integer | yes | TCP port number (default iSCSI port is 3260). |

### `ProtocolStatus`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `display_name` | string | yes | Human-readable display name (e.g. `NFS`, `SMB`, `iSCSI`). |
| `enabled` | boolean | yes | Whether the protocol is enabled in persistent state. |
| `name` | string | yes | Machine-readable protocol identifier (e.g. `nfs`, `smb`, `iscsi`). |
| `running` | boolean | yes | Whether the protocol's systemd service is currently active. |
| `system_service` | boolean | yes | Whether this is a system-level service (SSH, Avahi, SMART) rather than a storage protocol. |

### `Role`

Enum: `admin`, `readonly`, `operator`

### `ServiceStatus`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | systemd service name. |
| `running` | boolean | yes | Whether the service is currently active/running. |

### `SmartAttribute`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `failing` | boolean | yes | Whether this attribute is currently at or below its failure threshold. |
| `id` | integer | yes | ATA attribute ID (1–255). |
| `name` | string | yes | Attribute name (e.g. `Raw_Read_Error_Rate`). |
| `raw_value` | integer | yes | Raw (vendor-specific) attribute value. |
| `threshold` | integer | yes | Failure threshold; attribute is failing when value drops below this. |
| `value` | integer | yes | Normalized current value (higher is better for most attributes). |
| `worst` | integer | yes | Worst normalized value ever recorded. |

### `SmbShare`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `browseable` | boolean | yes | Whether the share is visible in network browse lists. |
| `comment` | string | no | Optional description shown in share listings. |
| `enabled` | boolean | yes | Whether the share is active in `smb.nasty.conf`. |
| `extra_params` | object | yes | Additional raw Samba parameters written to the share section. |
| `guest_ok` | boolean | yes | Whether unauthenticated guest access is allowed. |
| `id` | string | yes | Unique share identifier (UUID). |
| `name` | string | yes | Samba share name used in `\\server\name` UNC paths. |
| `path` | string | yes | Absolute filesystem path being shared (must be under `/storage/`). |
| `read_only` | boolean | yes | Whether the share is read-only. |
| `valid_users` | string[] | yes | Usernames allowed to connect (empty means no restriction beyond authentication). |

### `Snapshot`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `block_device` | string | no | Loop device path if this snapshot's vol.img is currently attached (block snapshots only). |
| `name` | string | yes | Snapshot name (unique within the parent subvolume). |
| `path` | string | yes | Absolute filesystem path to the snapshot directory. |
| `pool` | string | yes | Name of the pool that contains this snapshot. |
| `read_only` | boolean | yes | Whether this snapshot is read-only. |
| `subvolume` | string | yes | Name of the parent subvolume. |

### `Subvolume`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `name` | string | yes | Subvolume name (unique within the pool). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `pool` | string | yes | Name of the pool that contains this subvolume. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes (filesystem subvolumes only, from `du`). |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |

### `SubvolumeType`

Enum: `filesystem`, `block`

### `UserInfo`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `role` | `Role` | yes | Role assigned to this user. |
| `username` | string | yes | Login username. |

