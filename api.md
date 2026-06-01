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
{"jsonrpc": "2.0", "id": 1, "error": {"code": -32603, "message": "filesystem not found: mypool"}}
```

## Authentication

Send `POST /api/login` with `{"username": "...", "password": "..."}` to receive a session token. Pass it as a cookie (`session=<token>`) or `Authorization: Bearer <token>` header on the WebSocket upgrade.

## Roles

| Role | Description |
|------|-------------|
| `admin` | Full access to all methods |
| `operator` | Create/delete subvolumes and snapshots; read pools. Cannot manage users, destroy pools, or change system settings. |
| `readonly` | Read-only access to all list/get methods |

API tokens can additionally be scoped to a single **filesystem** (restricts visibility) and for operator tokens to a single **owner** (restricts to subvolumes owned by that token).

## Real-time Events

After any successful mutation the server broadcasts an event on the same WebSocket:
```json
{"event": "pool"}
```
Clients should re-fetch the relevant resource when they receive an event. Event types: `filesystem`, `subvolume`, `snapshot`, `share.nfs`, `share.smb`, `share.iscsi`, `share.nvmeof`, `protocol`, `settings`, `alert`.

---

## Contents

- [Authentication](#authentication)
- [System](#system)
- [System Update](#system-update)
- [Settings](#settings)
- [Network](#network)
- [Protocols & Services](#protocols--services)
- [Alert Rules](#alert-rules)
- [Block Devices](#block-devices)
- [Filesystems](#filesystems)
- [Filesystem Devices](#filesystem-devices)
- [Subvolumes](#subvolumes)
- [Snapshots](#snapshots)
- [NFS Shares](#nfs-shares)
- [SMB Shares](#smb-shares)
- [iSCSI Targets](#iscsi-targets)
- [NVMe-oF Subsystems](#nvme-of-subsystems)
- [OIDC](#oidc)
- [WebAuthn](#webauthn)
- [Audit](#audit)
- [System (continued)](#system-continued)
- [System Generations](#system-generations)
- [System Hardware](#system-hardware)
- [System Logs](#system-logs)
- [System Metrics](#system-metrics)
- [System ACME](#system-acme)
- [System TLS](#system-tls)
- [System NUT (UPS)](#system-nut-ups)
- [System Passthrough](#system-passthrough)
- [System Secure Boot](#system-secure-boot)
- [System SSH](#system-ssh)
- [System Tailscale](#system-tailscale)
- [System Tuning](#system-tuning)
- [System Firewall](#system-firewall)
- [System Update Channel](#system-update-channel)
- [Network (continued)](#network-continued)
- [Protocols & Services (continued)](#protocols--services-continued)
- [Telemetry](#telemetry)
- [Notifications](#notifications)
- [Firmware](#firmware)
- [Filesystem Encryption](#filesystem-encryption)
- [Filesystem Reconcile](#filesystem-reconcile)
- [Filesystem Dependents](#filesystem-dependents)
- [bcachefs Tools](#bcachefs-tools)
- [Subvolumes (continued)](#subvolumes-continued)
- [SMB Users](#smb-users)
- [SMB Groups](#smb-groups)
- [Backup](#backup)
- [VMs](#vms)
- [VM Disk Images](#vm-disk-images)
- [Apps](#apps)

## Authentication

### `auth.me`

Return the current session's username and role.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `client_ip` | string | no | Client IP that created this session. Requests from other IPs are rejected. |
| `created_at` | integer | yes | Unix timestamp when this session was created. |
| `filesystem` | string | no | For API tokens: restricts filesystem visibility to a single filesystem. |
| `must_change_password` | boolean | no | When true, the user must change their password before doing anything else. |
| `owner` | string | no | For API tokens: only subvolumes with this owner are visible/manageable. |
| `role` | `Role` | yes |  |
| `token` | string | yes |  |
| `username` | string | yes |  |


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

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `username` | string | yes | Login username of the account to delete. |


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

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `expires_in_secs` | integer | no | Seconds until the token expires. Omit for a non-expiring token. |
| `filesystem` | string | no | If set, the token can only see subvolumes in this filesystem. |
| `name` | string | yes | Human-readable token name. |
| `role` | `Role` | yes |  |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `allowed_ips` | string[] | no | If set, token is only accepted from these IP addresses. Empty = any IP. |
| `created_at` | integer | yes |  |
| `expires_at` | integer | no | Unix timestamp after which the token is rejected. None = never expires. |
| `filesystem` | string | no | If set, token can only see/manage subvolumes in this filesystem. |
| `id` | string | yes |  |
| `name` | string | yes |  |
| `role` | `Role` | yes |  |
| `token` | string | yes | Argon2 hash of the token value. The raw token is returned only once on creation. |


### `auth.token.delete`

Delete an API token by ID.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | Unique token identifier. |


## System

### `system.info`

Return hostname, OS version, uptime, bcachefs-tools version info.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bcachefs_commit` | string | no | Short (12-char) commit SHA of the pinned bcachefs-tools in flake.lock |
| `bcachefs_debug_checks` | boolean | yes | Whether the RUNNING bcachefs module was built with debug checks.
Only true when debug checks are configured AND the system has been rebooted into it. |
| `bcachefs_debug_symbols` | boolean | yes | Whether the loaded bcachefs kernel module contains debug symbols. |
| `bcachefs_is_custom` | boolean | yes | True when the running bcachefs kernel module version doesn't
match the wrapper's currently-pinned `bcachefs-tools` ref —
i.e. an upgrade or pin change has activated a new generation
but the box hasn't been rebooted into it yet. The WebUI uses
this to surface a top-bar "reboot pending" cue. False when
the running version probe fails (unknown) or the wrapper has
no pinned ref to compare against. |
| `bcachefs_pinned_ref` | string | no | The ref currently pinned in `/etc/nixos/flake.lock` for `bcachefs-tools`. |
| `bcachefs_version` | string | yes | Output of `bcachefs version` (first line). |
| `engine_built` | string | no | Build timestamp of the engine binary. |
| `engine_commit` | string | no | Git commit the engine binary was compiled from. |
| `hostname` | string | yes | System hostname. |
| `is_virtual` | boolean | yes | Whether the system is running inside a virtual machine. |
| `kernel` | string | yes | Running Linux kernel version string. |
| `kvm_available` | boolean | yes | Whether KVM hardware virtualization is available (/dev/kvm exists). |
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
| `channel` | `ReleaseChannel` | yes | Active release channel. |
| `current_version` | string | yes | Currently installed version (short commit SHA or `dev`). |
| `error` | string | no | Human-readable explanation when the latest-version lookup failed
(GitHub unreachable, rate-limited, refused token, …). Populated
by `check()`; `version()` leaves it `None`. Surfaced in the UI
as an amber banner so operators don't see a silent dash when
GitHub is misbehaving — previously the failure mode was
indistinguishable from "no check has ever run". |
| `inputs` | `VersionInputInfo`[] | no | Snapshot of each tracked flake input (`nasty`, `nixpkgs`,
`bcachefs-tools`) — name, URL, locked rev. Embedded here so the
Version page can render all three pinned components in the
summary card without making a second RPC. None when the
engine can't read the local flake (parse error, fresh install
pre-bootstrap, etc); the UI falls back to the nasty rev alone
in that case. |
| `last_attempt` | string | no | Result of the last upgrade-unit invocation: `"success"`, `"failed"`,
or `None` if no upgrade has ever been kicked off (or it's still
running). When `"failed"`, the engine forces `update_available =
Some(true)` regardless of the tag comparison so the WebUI keeps
offering Upgrade — a failed rebuild often leaves `flake.lock`
pointing at the target tag, which would otherwise make the check
look like a no-op. |
| `latest_version` | string | no | Latest upstream version, if the check has been performed. |
| `update_available` | boolean | no | Whether a newer version is available. None if the check has not been run yet. |


### `system.update.check`

Check for available updates against the upstream repository.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `channel` | `ReleaseChannel` | yes | Active release channel. |
| `current_version` | string | yes | Currently installed version (short commit SHA or `dev`). |
| `error` | string | no | Human-readable explanation when the latest-version lookup failed
(GitHub unreachable, rate-limited, refused token, …). Populated
by `check()`; `version()` leaves it `None`. Surfaced in the UI
as an amber banner so operators don't see a silent dash when
GitHub is misbehaving — previously the failure mode was
indistinguishable from "no check has ever run". |
| `inputs` | `VersionInputInfo`[] | no | Snapshot of each tracked flake input (`nasty`, `nixpkgs`,
`bcachefs-tools`) — name, URL, locked rev. Embedded here so the
Version page can render all three pinned components in the
summary card without making a second RPC. None when the
engine can't read the local flake (parse error, fresh install
pre-bootstrap, etc); the UI falls back to the nasty rev alone
in that case. |
| `last_attempt` | string | no | Result of the last upgrade-unit invocation: `"success"`, `"failed"`,
or `None` if no upgrade has ever been kicked off (or it's still
running). When `"failed"`, the engine forces `update_available =
Some(true)` regardless of the tag comparison so the WebUI keeps
offering Upgrade — a failed rebuild often leaves `flake.lock`
pointing at the target tag, which would otherwise make the check
look like a no-op. |
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


### `system.update.build_dir.get`

Return the configured Nix build-dir spillover path (if any) plus the live list of mounted bcachefs pools eligible to host the sandbox. Useful on small-rootfs installs where the default `/tmp` (tmpfs) doesn't have room for kernel-module / Rust compile sandboxes.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_pools` | string[] | yes | Mounted bcachefs pool roots under `/fs/` discovered live from
`/proc/mounts`. Used by the WebUI to populate the dropdown of
viable spillover targets. Empty on single-disk (mode-1)
installs that don't have a separate data pool — the feature
can't help those boxes and the UI hides the option. |
| `path` | string | no | Persisted spillover path (`None` = unset, builds use the
default `/tmp` → tmpfs → root path). Returned verbatim from
the on-disk setting, **not** the auto-resolved
`<pool>/.nasty-nix-build` derivation — the WebUI dropdown
stores pool roots, the engine resolves at script-render time. |
| `resolved` | string | no | Resolved sandbox path the engine would actually pass to
`nixos-rebuild` (i.e. `<pool>/.nasty-nix-build`). Surfaced so
the WebUI can show operators where the spillover will land
without re-implementing the derivation rule. |


### `system.update.build_dir.set`

Set or clear the Nix build-dir spillover path. Pass `{"path": "/fs/<pool>"}` to enable (must match one of the mounted bcachefs pools reported by `build_dir.get`) or `{"path": null}` to disable. When set, the engine runs upgrade scripts with `NIX_REMOTE=local` and `--option build-dir <pool>/.nasty-nix-build` so the sandbox spills onto bcachefs instead of tmpfs/root.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `path` | string | yes | Filesystem mount path to use as the Nix sandbox spillover, or null to disable. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_pools` | string[] | yes | Mounted bcachefs pool roots under `/fs/` discovered live from
`/proc/mounts`. Used by the WebUI to populate the dropdown of
viable spillover targets. Empty on single-disk (mode-1)
installs that don't have a separate data pool — the feature
can't help those boxes and the UI hides the option. |
| `path` | string | no | Persisted spillover path (`None` = unset, builds use the
default `/tmp` → tmpfs → root path). Returned verbatim from
the on-disk setting, **not** the auto-resolved
`<pool>/.nasty-nix-build` derivation — the WebUI dropdown
stores pool roots, the engine resolves at script-render time. |
| `resolved` | string | no | Resolved sandbox path the engine would actually pass to
`nixos-rebuild` (i.e. `<pool>/.nasty-nix-build`). Surfaced so
the WebUI can show operators where the spillover will land
without re-implementing the derivation rule. |


### `system.version.get`

Return exact input URLs from `/etc/nixos/flake.nix` and locked revs from `/etc/nixos/flake.lock` for the Version page.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `inputs` | `VersionInputInfo`[] | yes | Inputs shown on the Version page in fixed display order. |


### `system.version.tagged_release_notice`

Return the latest official tagged release and whether the current `nasty.url` already matches its standard `github:nasty-project/nasty/vX.Y.Z` form.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `current_is_latest_standard_url` | boolean | yes | True when `nasty.url` already matches the newest official tagged release. |
| `current_url` | string | yes | Exact current `nasty.url` string from `/etc/nixos/flake.nix`. |
| `latest_tag` | string | yes | Latest official NASty release tag available upstream. |
| `latest_url` | string | yes | Standard shorthand URL for the latest official tagged release. |


### `system.version.upgrade_tagged_release`

Bootstrap a new wrapper `flake.nix` from the latest official tagged release template and start a switch rebuild.

**Role:** `admin`


### `system.version.switch`

Update selected flake inputs on the installed system and rebuild only if `flake.lock` changed.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `inputs` | `VersionSwitchInput`[] | yes | Requested URLs and update flags for the Version page. |


## Settings

### `system.settings.get`

Return current system settings.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `clock_24h` | boolean | no | Whether to display clocks in 24-hour format. |
| `hostname` | string | no | System hostname. |
| `oidc` | `OidcSettings` | no | OpenID Connect single-sign-on configuration. Disabled by default. |
| `telemetry_enabled` | boolean | no | Whether anonymous telemetry is enabled (drive count, storage capacity). |
| `temp_unit` | `TempUnit` | no | Unit for displayed temperatures (CPU, disks, alert thresholds).
Storage and alert evaluation always use Celsius internally — this
only affects rendering in the WebUI. |
| `timezone` | string | no | IANA timezone string applied to the system (e.g. `UTC`, `America/New_York`). |
| `tls_acme_email` | string | no | Email address for Let's Encrypt ACME notifications. |
| `tls_acme_enabled` | boolean | no | Whether Let's Encrypt is enabled. Requires tls_domain and tls_acme_email. |
| `tls_acme_staging` | boolean | no | Use Let's Encrypt staging environment (for testing, avoids rate limits). |
| `tls_challenge_type` | string | no | ACME challenge type. Caddy's built-in ACME issuer handles all three:
  - "tls-alpn"  → TLS-ALPN-01 (port 443)
  - "http"      → HTTP-01 (port 80)
  - "dns"       → DNS-01 via a DNS-provider plugin compiled into Caddy |
| `tls_dns_credentials` | string | no | DNS provider API credentials as KEY=VALUE lines.
Written to a Caddy `EnvironmentFile` and referenced from the
generated `tls` block via `{env.KEY}` placeholders. |
| `tls_dns_propagation_wait` | integer | no | Seconds to wait after creating the TXT record before checking
propagation. Defaults to 30. The recursive resolvers we use to
verify propagation often have a long negative TTL on the
`_acme-challenge.X` name (cached NXDOMAIN from prior lookups);
without a wait, Caddy queries immediately, sees the cached
answer, and the timer-based propagation check times out before
the cache flushes. Bump this when issuance still times out
after several minutes (slow DNS providers, long SOA MINIMUM
TTL on the parent zone). |
| `tls_dns_provider` | string | no | DNS provider code for DNS-01 challenge (e.g. "cloudflare", "route53").
Must match a DNS plugin compiled into the Caddy binary. |
| `tls_dns_resolver` | string | no | External DNS resolvers (comma-separated) to use when verifying
TXT-record propagation during DNS-01. Defaults to "1.1.1.1,8.8.8.8".
Set this when the box's authoritative DNS isn't reachable via
public resolvers (split-horizon zones, air-gapped networks).
Empty / None ⇒ use the default. |
| `tls_domain` | string | no | Domain name for Let's Encrypt TLS (e.g. "nasty.example.com"). Empty = self-signed. |


### `system.settings.update`

Update system settings. Only provided fields are changed.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `clock_24h` | boolean | no | Whether to use 24-hour clock display (optional). |
| `hostname` | string | no | New hostname to set (optional). |
| `telemetry_enabled` | boolean | no | Enable/disable anonymous telemetry. |
| `temp_unit` | `TempUnit` \| null | no | Display unit for temperatures (optional). |
| `timezone` | string | no | New IANA timezone to apply (optional). |
| `tls_acme_email` | string | no | Email address for ACME notifications. |
| `tls_acme_enabled` | boolean | no | Enable/disable Let's Encrypt. |
| `tls_acme_staging` | boolean | no | Use staging environment. |
| `tls_challenge_type` | string | no | Challenge type: "tls-alpn" or "dns". |
| `tls_dns_credentials` | string | no | DNS API credentials (KEY=VALUE per line). |
| `tls_dns_propagation_wait` | integer | no | Propagation wait in seconds. 0 clears (engine treats as default). |
| `tls_dns_provider` | string | no | DNS provider code. |
| `tls_dns_resolver` | string | no | External DNS resolvers (comma-separated). Empty string clears. |
| `tls_domain` | string | no | Domain name for Let's Encrypt TLS (set to empty string to disable). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `clock_24h` | boolean | no | Whether to display clocks in 24-hour format. |
| `hostname` | string | no | System hostname. |
| `oidc` | `OidcSettings` | no | OpenID Connect single-sign-on configuration. Disabled by default. |
| `telemetry_enabled` | boolean | no | Whether anonymous telemetry is enabled (drive count, storage capacity). |
| `temp_unit` | `TempUnit` | no | Unit for displayed temperatures (CPU, disks, alert thresholds).
Storage and alert evaluation always use Celsius internally — this
only affects rendering in the WebUI. |
| `timezone` | string | no | IANA timezone string applied to the system (e.g. `UTC`, `America/New_York`). |
| `tls_acme_email` | string | no | Email address for Let's Encrypt ACME notifications. |
| `tls_acme_enabled` | boolean | no | Whether Let's Encrypt is enabled. Requires tls_domain and tls_acme_email. |
| `tls_acme_staging` | boolean | no | Use Let's Encrypt staging environment (for testing, avoids rate limits). |
| `tls_challenge_type` | string | no | ACME challenge type. Caddy's built-in ACME issuer handles all three:
  - "tls-alpn"  → TLS-ALPN-01 (port 443)
  - "http"      → HTTP-01 (port 80)
  - "dns"       → DNS-01 via a DNS-provider plugin compiled into Caddy |
| `tls_dns_credentials` | string | no | DNS provider API credentials as KEY=VALUE lines.
Written to a Caddy `EnvironmentFile` and referenced from the
generated `tls` block via `{env.KEY}` placeholders. |
| `tls_dns_propagation_wait` | integer | no | Seconds to wait after creating the TXT record before checking
propagation. Defaults to 30. The recursive resolvers we use to
verify propagation often have a long negative TTL on the
`_acme-challenge.X` name (cached NXDOMAIN from prior lookups);
without a wait, Caddy queries immediately, sees the cached
answer, and the timer-based propagation check times out before
the cache flushes. Bump this when issuance still times out
after several minutes (slow DNS providers, long SOA MINIMUM
TTL on the parent zone). |
| `tls_dns_provider` | string | no | DNS provider code for DNS-01 challenge (e.g. "cloudflare", "route53").
Must match a DNS plugin compiled into the Caddy binary. |
| `tls_dns_resolver` | string | no | External DNS resolvers (comma-separated) to use when verifying
TXT-record propagation during DNS-01. Defaults to "1.1.1.1,8.8.8.8".
Set this when the box's authoritative DNS isn't reachable via
public resolvers (split-horizon zones, air-gapped networks).
Empty / None ⇒ use the default. |
| `tls_domain` | string | no | Domain name for Let's Encrypt TLS (e.g. "nasty.example.com"). Empty = self-signed. |


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
| `bonds` | `BondConfig`[] | no |  |
| `bridges` | `BridgeConfig`[] | no |  |
| `dns` | string[] | no |  |
| `interfaces` | `InterfaceConfig`[] | no |  |
| `vlans` | `VlanConfig`[] | no |  |


### `system.network.update`

Update network configuration (DHCP or static). Applied immediately without rebooting.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bonds` | `BondConfig`[] | no |  |
| `bridges` | `BridgeConfig`[] | no |  |
| `dns` | string[] | no |  |
| `interfaces` | `InterfaceConfig`[] | no |  |
| `vlans` | `VlanConfig`[] | no |  |


## Protocols & Services

### `service.protocol.list`

List all protocols and their current status.

**Role:** `any`

**Returns:**

``ProtocolStatus`[]`


### `service.protocol.enable`

Enable a protocol service. Available names: `nfs`, `smb`, `iscsi`, `nvmeof`, `ssh`, `avahi`, `smart`.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Protocol name (nfs, smb, iscsi, nvmeof, ssh, avahi, smart). |

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

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Protocol name (nfs, smb, iscsi, nvmeof, ssh, avahi, smart). |

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

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | Unique rule identifier. |


## Block Devices

### `device.list`

List all block devices and partitions visible to the system.

**Role:** `any`

**Returns:**

``BlockDevice`[]`


### `device.wipe`

Erase all filesystem signatures from a device (wipefs). The device must not be in use.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `path` | string | yes | Block device path (e.g. /dev/sdb). |


## Filesystems

### `fs.list`

List all filesystems. Filesystem-scoped tokens see only their assigned filesystem.

**Role:** `any`

**Returns:**

``Filesystem`[]`


### `fs.get`

Get a single filesystem by name.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `FilesystemDevice`[] | yes | Member devices of the filesystem. |
| `mount_point` | string | no | Absolute path where the filesystem is mounted (e.g. `/fs/tank`). |
| `mounted` | boolean | yes | Whether the filesystem is currently mounted. |
| `name` | string | yes | Human-readable filesystem name, derived from the mount point directory. |
| `options` | `FilesystemOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `fs.create`

Format and mount a new bcachefs filesystem.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `background_target` | string | no | Target label for background migration. |
| `bind_to_tpm` | boolean | no | Whether to seal the stored key with the host's TPM2 immediately
after creation (PCR-7 bound, same shape as `fs.tpm.bind`). Saves
the operator the WebUI "Bind to TPM" round-trip and avoids the
brief window between FS creation and binding when the plaintext
`.key` exists alone on disk. Requires `encryption == true`,
`store_key != false`, and a usable TPM2 on the host — request
is rejected upfront when any are missing. |
| `bucket_size` | string | no | Bucket size in bytes (e.g. `"512k"`, `"1M"`). Affects allocation granularity. |
| `compression` | string | no | Inline compression algorithm (e.g. `lz4`, `zstd`, `none`). |
| `data_checksum` | string | no | Data checksum algorithm (e.g. `crc32c`, `crc64`, `xxhash`, `none`). |
| `devices` | `DeviceSpec`[] | yes | Devices to include in the filesystem. |
| `encoded_extent_max` | string | no | Maximum encoded extent size (e.g. `"64k"`, `"128k"`). |
| `encryption` | boolean | no | Whether to enable encryption at format time. |
| `erasure_code` | boolean | no | Whether to enable erasure coding. |
| `foreground_target` | string | no | Tiering targets set at format time. |
| `io_scheduler` | string | no | I/O scheduler for member block devices (`none`, `mq-deadline`, `kyber`).
`none` is recommended for SSDs. |
| `journal_flush_delay` | integer | no | Journal flush delay in microseconds (default: 1000). Higher values batch
more journal writes, improving throughput under sync-heavy workloads. |
| `label` | string | no | Filesystem-wide label (used as default when no per-device labels set). |
| `metadata_checksum` | string | no | Metadata checksum algorithm. |
| `metadata_target` | string | no | Target label for metadata placement. |
| `name` | string | yes | Name for the new filesystem; becomes the mount point directory under `/fs/`. |
| `passphrase` | string | no | Passphrase for encryption (required when encryption is true). |
| `promote_target` | string | no | Target label for data promotion (cache tier). |
| `replicas` | integer | no | Number of data replicas (default 1). |
| `store_key` | boolean | no | Whether to store the key for auto-unlock on boot (default true).
When false, user must enter passphrase via WebUI after every reboot. |
| `version_upgrade` | string | no | Version upgrade behavior at mount time: `compatible`, `incompatible`, or `none`. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `FilesystemDevice`[] | yes | Member devices of the filesystem. |
| `mount_point` | string | no | Absolute path where the filesystem is mounted (e.g. `/fs/tank`). |
| `mounted` | boolean | yes | Whether the filesystem is currently mounted. |
| `name` | string | yes | Human-readable filesystem name, derived from the mount point directory. |
| `options` | `FilesystemOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `fs.destroy`

Unmount and unregister a filesystem. Does not wipe the devices.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `confirm_name` | string | yes | Must match `name` exactly — guards against accidental destruction. |
| `name` | string | yes | Name of the filesystem to destroy. |


### `fs.mount`

Mount a known filesystem.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `FilesystemDevice`[] | yes | Member devices of the filesystem. |
| `mount_point` | string | no | Absolute path where the filesystem is mounted (e.g. `/fs/tank`). |
| `mounted` | boolean | yes | Whether the filesystem is currently mounted. |
| `name` | string | yes | Human-readable filesystem name, derived from the mount point directory. |
| `options` | `FilesystemOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `fs.unmount`

Unmount a filesystem.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |


### `fs.options.update`

Update runtime-mutable bcachefs filesystem options (written to sysfs).

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `background_compression` | string | no | Background recompression algorithm. |
| `background_target` | string | no | Target label for background migration. |
| `compression` | string | no | Inline compression algorithm (e.g. `lz4`, `zstd`, `none`). |
| `data_checksum` | string | no | Data checksum algorithm (`none`, `crc32c`, `crc64`, `xxhash`). |
| `data_replicas` | integer | no | Number of data replicas. |
| `degraded` | boolean | no | Mount in degraded mode (allow mounting with missing devices). |
| `erasure_code` | boolean | no | Whether to enable erasure coding. |
| `error_action` | string | no | Action on unrecoverable read errors (`continue`, `ro`, `panic`). |
| `foreground_target` | string | no | Target label for foreground (new) writes. |
| `fsck` | boolean | no | Run fsck at mount time. |
| `io_scheduler` | string | no | I/O scheduler for member block devices (`none`, `mq-deadline`, `kyber`). |
| `journal_flush_delay` | integer | no | Journal flush delay in microseconds. Higher values batch more journal writes. |
| `journal_flush_disabled` | boolean | no | Disable journal flushing (unsafe, for benchmarking). |
| `metadata_checksum` | string | no | Metadata checksum algorithm (`none`, `crc32c`, `crc64`, `xxhash`). |
| `metadata_replicas` | integer | no | Number of metadata replicas. |
| `metadata_target` | string | no | Target label for metadata placement. |
| `move_bytes_in_flight` | string | no | Maximum bytes in flight for background mover (e.g. `"8.0M"`). |
| `move_ios_in_flight` | integer | no | Maximum concurrent background mover IOs. |
| `name` | string | yes | Name of the filesystem to update. |
| `promote_target` | string | no | Target label for data promotion (cache tier). |
| `verbose` | boolean | no | Enable verbose mount logging. |
| `version_upgrade` | string | no | Version upgrade behavior at mount time: `compatible`, `incompatible`, or `none`.
Changing mount options requires a remount. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `FilesystemDevice`[] | yes | Member devices of the filesystem. |
| `mount_point` | string | no | Absolute path where the filesystem is mounted (e.g. `/fs/tank`). |
| `mounted` | boolean | yes | Whether the filesystem is currently mounted. |
| `name` | string | yes | Human-readable filesystem name, derived from the mount point directory. |
| `options` | `FilesystemOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `fs.usage`

Return detailed bcachefs `fs usage` breakdown.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `data_bytes` | integer | yes | Total data stored (before replication). |
| `devices` | `DeviceUsage`[] | yes | Per-device usage breakdown. |
| `metadata_bytes` | integer | yes | Total metadata stored. |
| `raw` | string | yes | Raw output from `bcachefs fs usage`, structured where possible. |
| `reserved_bytes` | integer | yes | Reserved bytes. |


### `fs.scrub.start`

Start a scrub on a mounted filesystem.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |


### `fs.scrub.status`

Return current scrub status.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `raw` | string | yes | Raw text output from the bcachefs scrub status command. |
| `running` | boolean | yes | Whether a scrub is currently in progress. |


### `fs.reconcile.status`

Return bcachefs background work (reconcile) status.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `enabled` | boolean | yes | Whether reconcile is currently enabled on this filesystem. |
| `raw` | string | yes | Raw text output from the bcachefs reconcile status command. |


### `bcachefs.usage`

Return raw `bcachefs fs usage` output for a filesystem.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `data_bytes` | integer | yes | Total data stored (before replication). |
| `devices` | `DeviceUsage`[] | yes | Per-device usage breakdown. |
| `metadata_bytes` | integer | yes | Total metadata stored. |
| `raw` | string | yes | Raw output from `bcachefs fs usage`, structured where possible. |
| `reserved_bytes` | integer | yes | Reserved bytes. |


### `fs.tpm.status`

Report TPM2 host capability and per-filesystem bind state. `tpm_available` reflects whether `/dev/tpmrm0` is present; `bound` reflects whether a sealed-key blob exists for this filesystem.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bound` | boolean | yes | A `<KEYS_DIR>/<name>.tpm` sealed blob exists for this filesystem. |
| `tpm_available` | boolean | yes | Host has a usable TPM 2.0 resource manager (`/dev/tpmrm0`). |


### `fs.tpm.bind`

Seal the filesystem's stored encryption key with the host TPM2 (PCR-7 bound). Writes the sealed blob next to the plaintext `.key`; the plaintext is retained as a recovery path until `fs.key.delete` is invoked. Errors when the host has no usable TPM2 or no stored `.key` exists.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bound` | boolean | yes | A `<KEYS_DIR>/<name>.tpm` sealed blob exists for this filesystem. |
| `tpm_available` | boolean | yes | Host has a usable TPM 2.0 resource manager (`/dev/tpmrm0`). |


### `fs.tpm.unbind`

Remove the TPM2-sealed copy of the encryption key. The plaintext `.key` is unaffected. No-op success when no sealed blob exists.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bound` | boolean | yes | A `<KEYS_DIR>/<name>.tpm` sealed blob exists for this filesystem. |
| `tpm_available` | boolean | yes | Host has a usable TPM 2.0 resource manager (`/dev/tpmrm0`). |


## Filesystem Devices

### `fs.device.add`

Add a device to an existing mounted filesystem.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device` | `DeviceSpec` | yes | Device to add, with optional label and durability settings. |
| `filesystem` | string | yes | Name of the filesystem to add the device to. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `FilesystemDevice`[] | yes | Member devices of the filesystem. |
| `mount_point` | string | no | Absolute path where the filesystem is mounted (e.g. `/fs/tank`). |
| `mounted` | boolean | yes | Whether the filesystem is currently mounted. |
| `name` | string | yes | Human-readable filesystem name, derived from the mount point directory. |
| `options` | `FilesystemOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `fs.device.remove`

Remove a device from a filesystem. The device should be fully evacuated first.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device` | string | yes | Absolute path of the block device (e.g. `/dev/sdb`). |
| `filesystem` | string | yes | Name of the filesystem containing the device. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `FilesystemDevice`[] | yes | Member devices of the filesystem. |
| `mount_point` | string | no | Absolute path where the filesystem is mounted (e.g. `/fs/tank`). |
| `mounted` | boolean | yes | Whether the filesystem is currently mounted. |
| `name` | string | yes | Human-readable filesystem name, derived from the mount point directory. |
| `options` | `FilesystemOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `fs.device.evacuate`

Evacuate all data from a device to the remaining filesystem members. Long-running — returns `{"status": "started"}` immediately. Filesystem events are broadcast every 3s during evacuation.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device` | string | yes | Absolute path of the block device (e.g. `/dev/sdb`). |
| `filesystem` | string | yes | Name of the filesystem containing the device. |


### `fs.device.set_state`

Set persistent device state (`rw`, `ro`, `failed`, `spare`).

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device` | string | yes | Absolute path of the block device (e.g. `/dev/sdb`). |
| `filesystem` | string | yes | Name of the filesystem containing the device. |
| `state` | string | yes | One of: rw, ro, failed, spare |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `FilesystemDevice`[] | yes | Member devices of the filesystem. |
| `mount_point` | string | no | Absolute path where the filesystem is mounted (e.g. `/fs/tank`). |
| `mounted` | boolean | yes | Whether the filesystem is currently mounted. |
| `name` | string | yes | Human-readable filesystem name, derived from the mount point directory. |
| `options` | `FilesystemOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `fs.device.set_label`

Set or update the hierarchical label on a device in a mounted filesystem. Written live via sysfs.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device` | string | yes | Absolute path of the block device (e.g. `/dev/sdb`). |
| `filesystem` | string | yes | Name of the filesystem containing the device. |
| `label` | string | yes | New hierarchical label (e.g. `ssd.fast`, `hdd.archive`). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `FilesystemDevice`[] | yes | Member devices of the filesystem. |
| `mount_point` | string | no | Absolute path where the filesystem is mounted (e.g. `/fs/tank`). |
| `mounted` | boolean | yes | Whether the filesystem is currently mounted. |
| `name` | string | yes | Human-readable filesystem name, derived from the mount point directory. |
| `options` | `FilesystemOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `fs.device.online`

Bring a device back online.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device` | string | yes | Absolute path of the block device (e.g. `/dev/sdb`). |
| `filesystem` | string | yes | Name of the filesystem containing the device. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `FilesystemDevice`[] | yes | Member devices of the filesystem. |
| `mount_point` | string | no | Absolute path where the filesystem is mounted (e.g. `/fs/tank`). |
| `mounted` | boolean | yes | Whether the filesystem is currently mounted. |
| `name` | string | yes | Human-readable filesystem name, derived from the mount point directory. |
| `options` | `FilesystemOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `fs.device.offline`

Take a device offline.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device` | string | yes | Absolute path of the block device (e.g. `/dev/sdb`). |
| `filesystem` | string | yes | Name of the filesystem containing the device. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `FilesystemDevice`[] | yes | Member devices of the filesystem. |
| `mount_point` | string | no | Absolute path where the filesystem is mounted (e.g. `/fs/tank`). |
| `mounted` | boolean | yes | Whether the filesystem is currently mounted. |
| `name` | string | yes | Human-readable filesystem name, derived from the mount point directory. |
| `options` | `FilesystemOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


## Subvolumes

### `subvolume.list`

List subvolumes in a filesystem.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filesystem` | string | yes | Filesystem name. |

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

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filesystem` | string | yes | Filesystem name. |
| `name` | string | yes | Subvolume name. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bcachefs_options` | object | no | Effective bcachefs options set on this subvolume (from bcachefs_effective.* xattrs).
Only includes options that differ from the filesystem default. |
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `direct_io` | boolean | no | Whether O_DIRECT is enabled on the loop device (block subvolumes only). |
| `filesystem` | string | yes | Name of the filesystem that contains this subvolume. |
| `name` | string | yes | Subvolume name (unique within the filesystem). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `parent` | string | no | Parent subvolume name if this is a clone (from bcachefs snapshot_parent). |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `quota_bytes` | integer | no | Hard quota limit in bytes for filesystem subvolumes. `None`
means no limit set (the subvolume can grow to fill the
filesystem). Always `None` for block subvolumes — their
ceiling is `volsize_bytes`, not a quota. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes. For filesystem subvolumes, comes from the
per-project quota (set on every create, so tracking is always
on); `None` only on legacy subvolumes created before the
always-track change. For block subvolumes, comes from the
backing image's allocated size. |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |


### `subvolume.create`

Create a new bcachefs subvolume (filesystem or block-backed).

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `background_target` | string | no | Device or label for background moves/recompression (overrides filesystem default). |
| `comments` | string | no | Optional description for the subvolume. |
| `compression` | string | no | Compression algorithm to set on the subvolume (e.g. `lz4`, `zstd`). |
| `data_replicas` | integer | no | Number of data replicas for this subvolume (overrides filesystem default). |
| `direct_io` | boolean | no | Enable O_DIRECT on the loop device (bypasses host page cache for the backing file). |
| `filesystem` | string | yes | Name of the filesystem to create the subvolume in. |
| `foreground_target` | string | no | Device or label for foreground writes (overrides filesystem default). |
| `metadata_target` | string | no | Device or label for metadata/btree writes (overrides filesystem default). |
| `name` | string | yes | Name for the new subvolume. |
| `promote_target` | string | no | Device or label to promote data to on read (cache tier, overrides filesystem default). |
| `subvolume_type` | `SubvolumeType` | no | Whether to create a filesystem or block-backed subvolume (default: filesystem). |
| `volsize_bytes` | integer | no | Size of the block backing image in bytes (required for block subvolumes). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bcachefs_options` | object | no | Effective bcachefs options set on this subvolume (from bcachefs_effective.* xattrs).
Only includes options that differ from the filesystem default. |
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `direct_io` | boolean | no | Whether O_DIRECT is enabled on the loop device (block subvolumes only). |
| `filesystem` | string | yes | Name of the filesystem that contains this subvolume. |
| `name` | string | yes | Subvolume name (unique within the filesystem). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `parent` | string | no | Parent subvolume name if this is a clone (from bcachefs snapshot_parent). |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `quota_bytes` | integer | no | Hard quota limit in bytes for filesystem subvolumes. `None`
means no limit set (the subvolume can grow to fill the
filesystem). Always `None` for block subvolumes — their
ceiling is `volsize_bytes`, not a quota. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes. For filesystem subvolumes, comes from the
per-project quota (set on every create, so tracking is always
on); `None` only on legacy subvolumes created before the
always-track change. For block subvolumes, comes from the
backing image's allocated size. |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |


### `subvolume.delete`

Delete a subvolume and all its snapshots.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filesystem` | string | yes | Name of the filesystem containing the subvolume. |
| `name` | string | yes | Name of the subvolume to delete. |


### `subvolume.attach`

Attach the loop device for a block subvolume (mounts `vol.img` via losetup).

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filesystem` | string | yes | Filesystem name. |
| `name` | string | yes | Subvolume name. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bcachefs_options` | object | no | Effective bcachefs options set on this subvolume (from bcachefs_effective.* xattrs).
Only includes options that differ from the filesystem default. |
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `direct_io` | boolean | no | Whether O_DIRECT is enabled on the loop device (block subvolumes only). |
| `filesystem` | string | yes | Name of the filesystem that contains this subvolume. |
| `name` | string | yes | Subvolume name (unique within the filesystem). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `parent` | string | no | Parent subvolume name if this is a clone (from bcachefs snapshot_parent). |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `quota_bytes` | integer | no | Hard quota limit in bytes for filesystem subvolumes. `None`
means no limit set (the subvolume can grow to fill the
filesystem). Always `None` for block subvolumes — their
ceiling is `volsize_bytes`, not a quota. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes. For filesystem subvolumes, comes from the
per-project quota (set on every create, so tracking is always
on); `None` only on legacy subvolumes created before the
always-track change. For block subvolumes, comes from the
backing image's allocated size. |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |


### `subvolume.detach`

Detach the loop device for a block subvolume.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filesystem` | string | yes | Filesystem name. |
| `name` | string | yes | Subvolume name. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bcachefs_options` | object | no | Effective bcachefs options set on this subvolume (from bcachefs_effective.* xattrs).
Only includes options that differ from the filesystem default. |
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `direct_io` | boolean | no | Whether O_DIRECT is enabled on the loop device (block subvolumes only). |
| `filesystem` | string | yes | Name of the filesystem that contains this subvolume. |
| `name` | string | yes | Subvolume name (unique within the filesystem). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `parent` | string | no | Parent subvolume name if this is a clone (from bcachefs snapshot_parent). |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `quota_bytes` | integer | no | Hard quota limit in bytes for filesystem subvolumes. `None`
means no limit set (the subvolume can grow to fill the
filesystem). Always `None` for block subvolumes — their
ceiling is `volsize_bytes`, not a quota. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes. For filesystem subvolumes, comes from the
per-project quota (set on every create, so tracking is always
on); `None` only on legacy subvolumes created before the
always-track change. For block subvolumes, comes from the
backing image's allocated size. |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |


### `subvolume.resize`

Resize a block subvolume's backing image.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filesystem` | string | yes | Name of the filesystem containing the subvolume. |
| `name` | string | yes | Name of the subvolume to resize. |
| `volsize_bytes` | integer | yes | New size in bytes. For block subvolumes: sparse image size. For filesystem subvolumes: bcachefs quota limit. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bcachefs_options` | object | no | Effective bcachefs options set on this subvolume (from bcachefs_effective.* xattrs).
Only includes options that differ from the filesystem default. |
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `direct_io` | boolean | no | Whether O_DIRECT is enabled on the loop device (block subvolumes only). |
| `filesystem` | string | yes | Name of the filesystem that contains this subvolume. |
| `name` | string | yes | Subvolume name (unique within the filesystem). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `parent` | string | no | Parent subvolume name if this is a clone (from bcachefs snapshot_parent). |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `quota_bytes` | integer | no | Hard quota limit in bytes for filesystem subvolumes. `None`
means no limit set (the subvolume can grow to fill the
filesystem). Always `None` for block subvolumes — their
ceiling is `volsize_bytes`, not a quota. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes. For filesystem subvolumes, comes from the
per-project quota (set on every create, so tracking is always
on); `None` only on legacy subvolumes created before the
always-track change. For block subvolumes, comes from the
backing image's allocated size. |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |


### `subvolume.set_properties`

Set arbitrary key-value metadata on a subvolume (stored as POSIX xattrs in the `user.*` namespace). Used by the CSI driver.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filesystem` | string | yes | Name of the filesystem containing the subvolume. |
| `name` | string | yes | Name of the subvolume to update. |
| `properties` | object | yes | Key-value pairs to set (merged with existing properties). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bcachefs_options` | object | no | Effective bcachefs options set on this subvolume (from bcachefs_effective.* xattrs).
Only includes options that differ from the filesystem default. |
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `direct_io` | boolean | no | Whether O_DIRECT is enabled on the loop device (block subvolumes only). |
| `filesystem` | string | yes | Name of the filesystem that contains this subvolume. |
| `name` | string | yes | Subvolume name (unique within the filesystem). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `parent` | string | no | Parent subvolume name if this is a clone (from bcachefs snapshot_parent). |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `quota_bytes` | integer | no | Hard quota limit in bytes for filesystem subvolumes. `None`
means no limit set (the subvolume can grow to fill the
filesystem). Always `None` for block subvolumes — their
ceiling is `volsize_bytes`, not a quota. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes. For filesystem subvolumes, comes from the
per-project quota (set on every create, so tracking is always
on); `None` only on legacy subvolumes created before the
always-track change. For block subvolumes, comes from the
backing image's allocated size. |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |


### `subvolume.remove_properties`

Remove specific metadata keys from a subvolume.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filesystem` | string | yes | Name of the filesystem containing the subvolume. |
| `keys` | string[] | yes | Property keys to remove. |
| `name` | string | yes | Name of the subvolume to update. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bcachefs_options` | object | no | Effective bcachefs options set on this subvolume (from bcachefs_effective.* xattrs).
Only includes options that differ from the filesystem default. |
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `direct_io` | boolean | no | Whether O_DIRECT is enabled on the loop device (block subvolumes only). |
| `filesystem` | string | yes | Name of the filesystem that contains this subvolume. |
| `name` | string | yes | Subvolume name (unique within the filesystem). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `parent` | string | no | Parent subvolume name if this is a clone (from bcachefs snapshot_parent). |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `quota_bytes` | integer | no | Hard quota limit in bytes for filesystem subvolumes. `None`
means no limit set (the subvolume can grow to fill the
filesystem). Always `None` for block subvolumes — their
ceiling is `volsize_bytes`, not a quota. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes. For filesystem subvolumes, comes from the
per-project quota (set on every create, so tracking is always
on); `None` only on legacy subvolumes created before the
always-track change. For block subvolumes, comes from the
backing image's allocated size. |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |


### `subvolume.find_by_property`

Find subvolumes matching a specific metadata key-value pair.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filesystem` | string | no | Optional filesystem to restrict the search to. |
| `key` | string | yes | xattr property key to match against. |
| `value` | string | yes | Value that the property key must equal. |

**Returns:**

``Subvolume`[]`


## Snapshots

### `snapshot.list`

List snapshots for all subvolumes in a filesystem.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filesystem` | string | yes | Filesystem name. |

**Returns:**

``Snapshot`[]`


### `snapshot.create`

Create a snapshot of a subvolume.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filesystem` | string | yes | Name of the filesystem containing the subvolume. |
| `name` | string | yes | Name for the new snapshot. |
| `read_only` | boolean | no | Whether to create a read-only snapshot (default: true). |
| `subvolume` | string | yes | Name of the subvolume to snapshot. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `block_device` | string | no | Loop device path if this snapshot's vol.img is currently attached (block snapshots only). |
| `filesystem` | string | yes | Name of the filesystem that contains this snapshot. |
| `name` | string | yes | Snapshot name (unique within the parent subvolume). |
| `parent` | string | no | Parent subvolume path as tracked by bcachefs (from snapshot_parent). |
| `path` | string | yes | Absolute filesystem path to the snapshot directory. |
| `read_only` | boolean | yes | Whether this snapshot is read-only. |
| `subvolume` | string | yes | Name of the parent subvolume. |


### `snapshot.delete`

Delete a snapshot.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filesystem` | string | yes | Name of the filesystem containing the snapshot. |
| `name` | string | yes | Name of the snapshot to delete. |
| `subvolume` | string | yes | Name of the parent subvolume. |


### `snapshot.clone`

Clone a snapshot into a new independent subvolume.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filesystem` | string | yes | Name of the filesystem containing the snapshot. |
| `new_name` | string | yes | Name for the new writable subvolume created from the snapshot. |
| `snapshot` | string | yes | Name of the snapshot to clone. |
| `subvolume` | string | yes | Name of the parent subvolume. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bcachefs_options` | object | no | Effective bcachefs options set on this subvolume (from bcachefs_effective.* xattrs).
Only includes options that differ from the filesystem default. |
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `direct_io` | boolean | no | Whether O_DIRECT is enabled on the loop device (block subvolumes only). |
| `filesystem` | string | yes | Name of the filesystem that contains this subvolume. |
| `name` | string | yes | Subvolume name (unique within the filesystem). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `parent` | string | no | Parent subvolume name if this is a clone (from bcachefs snapshot_parent). |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `quota_bytes` | integer | no | Hard quota limit in bytes for filesystem subvolumes. `None`
means no limit set (the subvolume can grow to fill the
filesystem). Always `None` for block subvolumes — their
ceiling is `volsize_bytes`, not a quota. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes. For filesystem subvolumes, comes from the
per-project quota (set on every create, so tracking is always
on); `None` only on legacy subvolumes created before the
always-track change. For block subvolumes, comes from the
backing image's allocated size. |
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

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | Unique share identifier. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `clients` | `NfsClient`[] | yes | List of allowed clients and their export options. |
| `comment` | string | no | Optional description of the share. |
| `enabled` | boolean | yes | Whether the share is currently active in `/etc/exports.d/nasty.exports`. |
| `id` | string | yes | Unique share identifier (UUID). |
| `path` | string | yes | Absolute filesystem path being exported (must be under `/fs/`). |


### `share.nfs.create`

Create an NFS share.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `clients` | `NfsClient`[] | yes | Allowed clients and their export options. |
| `comment` | string | no | Optional description. |
| `enabled` | boolean | no | Whether to enable the share immediately (default: true). |
| `path` | string | yes | Absolute path to export (must exist and be under `/fs/`). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `clients` | `NfsClient`[] | yes | List of allowed clients and their export options. |
| `comment` | string | no | Optional description of the share. |
| `enabled` | boolean | yes | Whether the share is currently active in `/etc/exports.d/nasty.exports`. |
| `id` | string | yes | Unique share identifier (UUID). |
| `path` | string | yes | Absolute filesystem path being exported (must be under `/fs/`). |


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
| `path` | string | yes | Absolute filesystem path being exported (must be under `/fs/`). |


### `share.nfs.delete`

Delete an NFS share.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes |  |


## SMB Shares

### `share.smb.list`

List all SMB shares.

**Role:** `any`

**Returns:**

``SmbShare`[]`


### `share.smb.get`

Get an SMB share by ID.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | Unique share identifier. |

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
| `path` | string | yes | Absolute filesystem path being shared (must be under `/fs/`). |
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
| `path` | string | yes | Absolute path to share (must exist and be under `/fs/`). |
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
| `path` | string | yes | Absolute filesystem path being shared (must be under `/fs/`). |
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
| `path` | string | yes | Absolute filesystem path being shared (must be under `/fs/`). |
| `read_only` | boolean | yes | Whether the share is read-only. |
| `valid_users` | string[] | yes | Usernames allowed to connect (empty means no restriction beyond authentication). |


### `share.smb.delete`

Delete an SMB share.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes |  |


## iSCSI Targets

### `share.iscsi.list`

List all iSCSI targets.

**Role:** `any`

**Returns:**

``IscsiTarget`[]`


### `share.iscsi.get`

Get an iSCSI target by ID.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | Unique share identifier. |

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
| `acls` | `AclEntry`[] | no | Initiator ACLs to set up. When provided, `generate_node_acls` is
disabled and only these initiators are allowed. |
| `alias` | string | no | Optional human-readable alias for the target. |
| `device_path` | string | no | Block device path (e.g. /dev/loop0). When provided, a LUN is
automatically created and the target is ready for connections. |
| `name` | string | yes | Short name used to generate the IQN: iqn.2137-01.com.nasty:<name> |
| `portals` | `Portal`[] | no | Defaults to 0.0.0.0:3260 |

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

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | ID of the iSCSI target to delete. |


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

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `lun_id` | integer | yes | LUN ID to remove. |
| `target_id` | string | yes | ID of the target from which to remove the LUN. |

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

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `initiator_iqn` | string | yes | Initiator IQN to disallow. |
| `target_id` | string | yes | ID of the target from which to remove the ACL. |

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

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | Unique share identifier. |

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
| `addr` | string | no | Listen address (default 0.0.0.0). Only used when `device_path` is set. |
| `allow_any_host` | boolean | no | Whether any host NQN is permitted to connect (default: true). |
| `allowed_hosts` | string[] | no | Host NQNs to allow. When provided, `allow_any_host` is set to false
and only these hosts are permitted. |
| `device_path` | string | no | Block device path (e.g. /dev/loop0). When provided, a namespace is
automatically created. |
| `name` | string | yes | Short name appended to NQN prefix |
| `port` | integer | no | Port number (default 4420). Only used when `device_path` is set. |

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

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | ID of the NVMe-oF subsystem to delete. |


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

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `nsid` | integer | yes | Namespace ID to remove. |
| `subsystem_id` | string | yes | ID of the subsystem from which to remove the namespace. |

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

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `port_id` | integer | yes | Port ID to remove. |
| `subsystem_id` | string | yes | ID of the subsystem from which to remove the port. |

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

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `host_nqn` | string | yes | NQN of the host to disallow. |
| `subsystem_id` | string | yes | ID of the subsystem from which to revoke access. |

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


## OIDC

### `auth.oidc.config_status`

Return the current OIDC settings with `client_secret` redacted to `<set>`/`<unset>` so the secret value never leaves the engine.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `auto_provision` | boolean | no | When true, unknown OIDC subjects are auto-provisioned as local users on first login. |
| `client_id` | string | no | OAuth client_id registered with the IdP. |
| `client_secret` | string | no | OAuth client_secret. Returned as a placeholder over RPC; only the engine sees the real value. |
| `default_role` | string | no | Role applied when no group mapping matches. None = deny login. |
| `enabled` | boolean | no | Master switch — when false, OIDC endpoints return 404 and no IdP traffic occurs. |
| `groups_claim` | string | no | Name of the ID-token claim that carries the user's groups. |
| `issuer_url` | string | no | IdP issuer URL (used for OIDC discovery, e.g. "https://auth.example.com"). |
| `redirect_uri` | string | no | Absolute redirect URI registered with the IdP (e.g. "https://nasty.local/api/auth/oidc/callback"). |
| `role_mappings` | `OidcRoleMapping`[] | no | Group → role mappings. Evaluated in order; first match wins. |
| `scopes` | string[] | no | OAuth scopes to request. Defaults to ["openid","profile","email","groups"]. |


### `auth.oidc.test`

Dry run that maps a sample claims object through the current OIDC role-mapping policy without contacting the IdP.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `claims` | object | yes | Sample IdP claims to feed through the role-mapping policy. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `role` | string | yes | Matched role name, or null when no mapping fires. |


### `auth.oidc.update_config`

Update the OIDC settings (preserves the stored client_secret if the caller sends the `<unchanged>` placeholder), rebuild the live OIDC client, and audit-log the change.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `auto_provision` | boolean | no | When true, unknown OIDC subjects are auto-provisioned as local users on first login. |
| `client_id` | string | no | OAuth client_id registered with the IdP. |
| `client_secret` | string | no | OAuth client_secret. Returned as a placeholder over RPC; only the engine sees the real value. |
| `default_role` | string | no | Role applied when no group mapping matches. None = deny login. |
| `enabled` | boolean | no | Master switch — when false, OIDC endpoints return 404 and no IdP traffic occurs. |
| `groups_claim` | string | no | Name of the ID-token claim that carries the user's groups. |
| `issuer_url` | string | no | IdP issuer URL (used for OIDC discovery, e.g. "https://auth.example.com"). |
| `redirect_uri` | string | no | Absolute redirect URI registered with the IdP (e.g. "https://nasty.local/api/auth/oidc/callback"). |
| `role_mappings` | `OidcRoleMapping`[] | no | Group → role mappings. Evaluated in order; first match wins. |
| `scopes` | string[] | no | OAuth scopes to request. Defaults to ["openid","profile","email","groups"]. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `auto_provision` | boolean | no | When true, unknown OIDC subjects are auto-provisioned as local users on first login. |
| `client_id` | string | no | OAuth client_id registered with the IdP. |
| `client_secret` | string | no | OAuth client_secret. Returned as a placeholder over RPC; only the engine sees the real value. |
| `default_role` | string | no | Role applied when no group mapping matches. None = deny login. |
| `enabled` | boolean | no | Master switch — when false, OIDC endpoints return 404 and no IdP traffic occurs. |
| `groups_claim` | string | no | Name of the ID-token claim that carries the user's groups. |
| `issuer_url` | string | no | IdP issuer URL (used for OIDC discovery, e.g. "https://auth.example.com"). |
| `redirect_uri` | string | no | Absolute redirect URI registered with the IdP (e.g. "https://nasty.local/api/auth/oidc/callback"). |
| `role_mappings` | `OidcRoleMapping`[] | no | Group → role mappings. Evaluated in order; first match wins. |
| `scopes` | string[] | no | OAuth scopes to request. Defaults to ["openid","profile","email","groups"]. |


## WebAuthn

### `auth.webauthn.config`

Return the engine-pinned WebAuthn RP ID so the WebUI can pre-check the operator's current origin before triggering credential creation.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `rp_id` | string | yes | RP ID baked into this engine instance. |


### `auth.webauthn.list`

List the calling user's registered WebAuthn credentials (label, created_at, base64url credential_id) — no public-key material.

**Role:** `any`

**Returns:**

`object[]`


### `auth.webauthn.register.start`

Begin WebAuthn registration: issue a server-side challenge and return creation_options for `navigator.credentials.create` plus a registration_id to round-trip to `register.finish`; rejects when the user has no non-WebAuthn fallback factor.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `label` | string | yes | Operator-facing label for the new credential ("YubiKey", "Phone", …). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `creation_options` | object | yes | Browser-facing `PublicKeyCredentialCreationOptions`; pass straight to `navigator.credentials.create`. |
| `registration_id` | string | yes | Opaque token to pass back to register.finish. |


### `auth.webauthn.register.finish`

Complete WebAuthn registration: validate the browser's `navigator.credentials.create` response against the pending challenge and persist the new passkey under the caller's user record.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `registration_id` | string | yes | Opaque token from register.start. |
| `response` | object | yes | Browser's `RegisterPublicKeyCredential` response object. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `created_at` | integer | yes |  |
| `credential_id` | string | yes |  |
| `label` | string | yes |  |


### `auth.webauthn.delete`

Delete one of the calling user's own registered WebAuthn credentials, identified by its base64url credential_id.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `credential_id` | string | yes | Base64url credential ID to remove. |


### `auth.webauthn.reset_for_user`

Admin recovery: clear every WebAuthn credential registered to the target user (used when they've lost all authenticators); audit-logged.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `username` | string | yes | Login username whose credentials should be cleared. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `removed` | integer | yes | Number of credentials cleared (0 is a successful no-op). |


## Audit

### `audit.list`

Return the most recent audit-log entries (default 200, capped by `limit`), parsed line-by-line in reverse chronological order. Entry shape depends on the action being audited.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `limit` | integer | no | Maximum number of entries to return. |

**Returns:**

`object[]`


## System (continued)

### `system.reboot_required`

Return true if the booted kernel/modules differ from the current system (a reboot is needed).

**Role:** `any`

**Returns:**

`object`


## System Generations

### `system.generations.list`

List all NixOS generations with metadata (date, kernel, NASty version, current/booted flags, user-assigned label).

**Role:** `any`

**Returns:**

``Generation`[]`


### `system.generations.label`

Set or clear a user-assigned label (e.g. "known good") on a NixOS generation.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `generation` | integer | yes | Generation number to label. |
| `label` | string | no | New label, or null to clear. |


### `system.generations.delete`

Delete a specific NixOS generation from the boot menu.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `generation` | integer | yes | Generation number to delete. |


### `system.generations.switch`

Switch the active system to a specific NixOS generation (rebuild-switch into it). Returns immediately while the switch runs in the background.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `generation` | integer | yes | Generation number to switch to. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `status` | string | yes |  |


## System Hardware

### `system.hardware.iommu`

Return IOMMU groups with their PCI device members (for passthrough planning).

**Role:** `any`

**Returns:**

``IommuGroup`[]`


### `system.hardware.summary`

Return a host hardware summary (DMI system/BIOS, CPU, memory DIMMs, USB devices, TPM, Secure Boot state).

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bios` | `DmiBios` \| null | no |  |
| `cpu` | `CpuSummary` \| null | no |  |
| `memory` | `MemorySummary` | yes |  |
| `secure_boot` | `SecureBootStatus` | yes | Secure Boot state as reported by `sbctl status --json`. Always
present — failure modes (BIOS boot, sbctl missing, sbctl
errored) collapse into a struct with `enabled = None` and a
human-readable `note` rather than an absent field. The WebUI
renders one of: enabled / disabled / unknown. |
| `system` | `DmiSystem` \| null | no |  |
| `tpm` | `TpmInfo` \| null | no | TPM chip presence + usability for the encryption-key sealing
flows that #102 is building toward. `None` when no TPM device
is enumerated by the kernel at all (no chip, disabled in
firmware, missing driver). |
| `usb` | `UsbDevice`[] | yes |  |


## System Logs

### `system.logs`

Return the tail of a systemd unit's journal.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `grep` | string | no | Optional substring filter. |
| `lines` | integer | no | Number of lines from the tail. |
| `unit` | string | no | Systemd unit to read. |

**Returns:**

`object`


### `system.logs.units`

Return the list of well-known systemd units that exist on this host (for the log-viewer unit picker).

**Role:** `any`

**Returns:**

`string[]`


### `system.log.level`

Return the engine's currently-active tracing/log filter string.

**Role:** `any`

**Returns:**

`object`


### `system.log.set_level`

Hot-reload the engine's tracing log filter without restart.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filter` | string | yes | New `EnvFilter` directive (e.g. `nasty_engine=trace,nasty_storage=debug`). |


## System Metrics

### `system.metrics.history`

Proxy historical metrics samples (CPU/net/disk/etc.) from the nasty-metrics sidecar.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `kind` | string | no | Resource kind (cpu, net, disk, ...). |
| `name` | string | no | Resource name (interface, device, ...). |
| `offset` | integer | no | Seconds offset from now (negative = past). |
| `range` | string | no | Lookback window (e.g. `1h`, `24h`). |

**Returns:**

`object[]`


### `system.metrics.prometheus`

Proxy the raw Prometheus-formatted scrape from the nasty-metrics sidecar.

**Role:** `any`

**Returns:**

`object`


## System ACME

### `system.acme.status`

Return the current ACME certificate issuance status (state, message, domain, issuer, expiry).

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `domain` | string | no | Domain the cert is for |
| `expires` | string | no | When the cert expires, if known |
| `issued` | string | no | When the cert was issued, if known |
| `issuer` | string | no | Certificate issuer (e.g. "Let's Encrypt") |
| `last_attempt` | string | no | When the last attempt was made |
| `message` | string | yes | Human-readable message (error details, progress info) |
| `state` | string | yes | "idle", "running", "success", "error" |


### `system.acme.reset`

Reset the in-memory ACME certificate issuance status back to "idle".

**Role:** `admin`


### `system.acme.retry`

Re-apply Caddy's TLS automation policy from disk to retry stalled ACME issuance.

**Role:** `admin`


## System TLS

### `system.tls.host_statuses`

Return per-host TLS automation status for every hostname Caddy is managing (active/issuing/failed/pending with last log message).

**Role:** `any`

**Returns:**

``HostTlsStatus`[]`


### `system.tls.local_ca_root`

Return Caddy's internal-CA root certificate (PEM) so operators can import it into their trust store. Errors with a "try again" message when Caddy hasn't bootstrapped yet.

**Role:** `any`

**Returns:**

`object`


## System NUT (UPS)

### `system.nut.config.get`

Return the persisted NUT (UPS) configuration.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `description` | string | no | Human-readable description.  Local mode only. |
| `driver` | string | no | NUT driver name (e.g. `usbhid-ups`, `blazer_usb`, `snmp-ups`).
Local mode only. |
| `mode` | `NutMode` | no | Whether the UPS is attached locally or monitored over the network. |
| `port` | string | no | Device port. `auto` for USB auto-detection, or a path like `/dev/ttyS0`.
Local mode only. |
| `remote_host` | string | no | Hostname or IP of the remote NUT server.  Remote mode only. |
| `remote_password` | string | no | Password configured in the remote upsd.users.  Remote mode only. |
| `remote_port` | integer | no | Port the remote NUT server listens on (default 3493).  Remote mode only. |
| `remote_username` | string | no | Username configured in the remote upsd.users.  Remote mode only. |
| `shutdown_command` | string | no | Command to execute for system shutdown. |
| `shutdown_on_battery_percent` | integer | no | Initiate shutdown when battery drops below this percentage. |
| `shutdown_on_battery_seconds` | integer | no | Initiate shutdown after this many seconds on battery power. |
| `ups_name` | string | no | UPS identifier used by upsc/upsd (e.g. `ups`). |


### `system.nut.config.update`

Apply a partial update to the NUT configuration and persist it.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `description` | string | no |  |
| `driver` | string | no |  |
| `mode` | `NutMode` \| null | no |  |
| `port` | string | no |  |
| `remote_host` | string | no |  |
| `remote_password` | string | no |  |
| `remote_port` | integer | no |  |
| `remote_username` | string | no |  |
| `shutdown_command` | string | no |  |
| `shutdown_on_battery_percent` | integer | no |  |
| `shutdown_on_battery_seconds` | integer | no |  |
| `ups_name` | string | no |  |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `description` | string | no | Human-readable description.  Local mode only. |
| `driver` | string | no | NUT driver name (e.g. `usbhid-ups`, `blazer_usb`, `snmp-ups`).
Local mode only. |
| `mode` | `NutMode` | no | Whether the UPS is attached locally or monitored over the network. |
| `port` | string | no | Device port. `auto` for USB auto-detection, or a path like `/dev/ttyS0`.
Local mode only. |
| `remote_host` | string | no | Hostname or IP of the remote NUT server.  Remote mode only. |
| `remote_password` | string | no | Password configured in the remote upsd.users.  Remote mode only. |
| `remote_port` | integer | no | Port the remote NUT server listens on (default 3493).  Remote mode only. |
| `remote_username` | string | no | Username configured in the remote upsd.users.  Remote mode only. |
| `shutdown_command` | string | no | Command to execute for system shutdown. |
| `shutdown_on_battery_percent` | integer | no | Initiate shutdown when battery drops below this percentage. |
| `shutdown_on_battery_seconds` | integer | no | Initiate shutdown after this many seconds on battery power. |
| `ups_name` | string | no | UPS identifier used by upsc/upsd (e.g. `ups`). |


### `system.nut.status`

Return the live UPS status (charge, runtime, voltage, load, model) as reported by `upsc`.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available` | boolean | yes | Whether the UPS service is running and reachable. |
| `battery_charge` | number | no | Battery charge percentage. |
| `battery_runtime` | integer | no | Estimated battery runtime in seconds. |
| `input_voltage` | number | no | Input voltage (from mains). |
| `output_voltage` | number | no | Output voltage (to load). |
| `raw` | object | yes | All raw key-value pairs from upsc. |
| `status` | string | yes | UPS status string (e.g. `OL` = online, `OB` = on battery, `LB` = low battery). |
| `ups_load` | number | no | UPS load percentage. |
| `ups_model` | string | no | UPS model/product name. |
| `ups_serial` | string | no | UPS serial number. |


## System Passthrough

### `system.passthrough.get`

Return the persisted PCI vfio-pci passthrough configuration (vendor:device pairs claimed at boot).

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `ids` | `DeviceId`[] | yes | (vendor, device) pairs to bind to vfio-pci at boot. Order is
not significant — we sort+dedupe on save and write. |


### `system.passthrough.update`

Validate, persist, and regenerate the passthrough Nix snippet from a new vendor:device list (reboot required to apply).

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `ids` | `DeviceId`[] | yes |  |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `ids` | `DeviceId`[] | yes | (vendor, device) pairs to bind to vfio-pci at boot. Order is
not significant — we sort+dedupe on save and write. |


## System Secure Boot

### `system.secure_boot.readiness`

Compute the Secure Boot readiness checklist (UEFI/TPM/ESP space/lanzaboote-in-flake/sbctl keys) for the Hardware page.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `blocker` | string | no | Human-readable one-line summary of why `ready == false`. The
WebUI surfaces this next to a disabled "Enable Secure Boot"
button. |
| `esp_free_bytes` | integer | no | Free bytes on `/boot` as reported by `statvfs`. `None` when
`/boot` isn't a separate mount or `statvfs` fails (rare). |
| `esp_required_bytes` | integer | yes | Threshold the WebUI compares `esp_free_bytes` against. Echoed
in the response so the UI doesn't have to hard-code the same
number on its side. |
| `ready` | boolean | yes | Top-level answer: can the operator flip
`services.nasty.secureBoot.enable = true` right now and
have it succeed end-to-end? `false` while any blocker
remains. |
| `sb_currently_off` | boolean | no | `Some(true)` when SB is currently off — i.e. ready to enable.
`Some(false)` when SB is already on (this readiness probe
doesn't apply to that path). `None` on BIOS boots / when
bootctl can't read the state. |
| `sb_supported_by_firmware` | boolean | no | `Some(true)` when bootctl reports SB support. `Some(false)`
when bootctl reports `disabled (unsupported)` — the firmware
itself lacks SB. `None` when we couldn't determine (BIOS
boot, bootctl missing). |
| `sbctl_keys_already_generated` | boolean | yes | True iff `/var/lib/sbctl/keys/db/db.key` exists. Purely
informational — when lanzaboote turns on with
`autoGenerateKeys.enable = true`, the keys get generated on
the first SB-enabled boot, so `false` here is the expected
state pre-enrollment. The WebUI surfaces this so an operator
who manually pre-generated keys sees that NASty noticed. |
| `tpm2_available` | boolean | yes | True iff `/dev/tpmrm0` is present — TPM2 sealing of bcachefs
keys requires it. Without TPM the SB opt-in still works, but
the bcachefs-binding part of NASty's TPM story has nothing
to seal to, so we treat this as a hard requirement for the
ceremony. |
| `uefi_boot` | boolean | yes | True iff `/sys/firmware/efi` exists — i.e. the kernel sees a
UEFI boot environment. BIOS / legacy boots return false and
every downstream check collapses to `None`. |
| `wrapper_has_lanzaboote_input` | boolean | no | `Some(true)` when `/etc/nixos/flake.nix` declares
`lanzaboote.url = ...` at top level. `Some(false)` on pre-
this-PR wrappers (operator needs to upgrade once so the
engine re-renders the template). `None` when we couldn't
read /etc/nixos/flake.nix (operator running an unusual
install or read failed). |


### `system.secure_boot.enrollment.status`

Return the combined persistent enrollment state plus the live `nasty-rebuild` unit snapshot.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `initiated_by` | string | no | Username that initiated the most recent ceremony attempt.
`None` only on a freshly-installed box that's never started
enrollment. |
| `phase` | `EnrollmentPhase` | yes |  |
| `rebuild` | `RebuildSnapshot` | yes |  |
| `rebuild_triggered_at` | integer | no | Unix seconds when the most recent wizard-driven
`nasty-rebuild` was triggered. `None` until the operator
clicks Rebuild from the wizard. Cleared back to `None` on
every `begin()` / `abort()` so each fresh ceremony starts
without a stale marker. The Abort copy reads this to
decide whether "the overlay was never applied" or "you
need to rebuild once more to revert" is accurate. |


### `system.secure_boot.enrollment.begin`

Start the Secure Boot enrollment ceremony by writing the lanzaboote Nix overlay and locking inputs.

**Role:** `admin`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `initiated_by` | string | no | Username that initiated the most recent ceremony attempt.
`None` only on a freshly-installed box that's never started
enrollment. |
| `phase` | `EnrollmentPhase` | yes |  |
| `rebuild_triggered_at` | integer | no | Unix seconds when the most recent wizard-driven
`nasty-rebuild` was triggered. `None` until the operator
clicks Rebuild from the wizard. Cleared back to `None` on
every `begin()` / `abort()` so each fresh ceremony starts
without a stale marker. The Abort copy reads this to
decide whether "the overlay was never applied" or "you
need to rebuild once more to revert" is accurate. |


### `system.secure_boot.enrollment.rebuild`

Trigger `nasty-rebuild` via systemd-run to apply the enrollment overlay the wizard wrote.

**Role:** `admin`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `triggered` | boolean | yes |  |


### `system.secure_boot.enrollment.complete`

Mark the Secure Boot enrollment ceremony done (only valid from PostEnrollment phase).

**Role:** `admin`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `initiated_by` | string | no | Username that initiated the most recent ceremony attempt.
`None` only on a freshly-installed box that's never started
enrollment. |
| `phase` | `EnrollmentPhase` | yes |  |
| `rebuild_triggered_at` | integer | no | Unix seconds when the most recent wizard-driven
`nasty-rebuild` was triggered. `None` until the operator
clicks Rebuild from the wizard. Cleared back to `None` on
every `begin()` / `abort()` so each fresh ceremony starts
without a stale marker. The Abort copy reads this to
decide whether "the overlay was never applied" or "you
need to rebuild once more to revert" is accurate. |


### `system.secure_boot.enrollment.abort`

Abort an in-progress Secure Boot enrollment by removing the lanzaboote overlay and lock entries.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `reason` | string | no | Optional operator-supplied reason recorded in audit log. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `initiated_by` | string | no | Username that initiated the most recent ceremony attempt.
`None` only on a freshly-installed box that's never started
enrollment. |
| `phase` | `EnrollmentPhase` | yes |  |
| `rebuild_triggered_at` | integer | no | Unix seconds when the most recent wizard-driven
`nasty-rebuild` was triggered. `None` until the operator
clicks Rebuild from the wizard. Cleared back to `None` on
every `begin()` / `abort()` so each fresh ceremony starts
without a stale marker. The Abort copy reads this to
decide whether "the overlay was never applied" or "you
need to rebuild once more to revert" is accurate. |


## System SSH

### `system.ssh.status`

Return whether sshd password auth is enabled and the list of authorized SSH keys.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `keys` | string[] | yes |  |
| `password_auth` | boolean | yes |  |


### `system.ssh.add_key`

Append an SSH public key to `/root/.ssh/authorized_keys`.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `key` | string | yes | Full public key line (must start with `ssh-` or `ecdsa-`). |


### `system.ssh.remove_key`

Remove a matching SSH public key line from `/root/.ssh/authorized_keys`.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `key` | string | yes | Full public key line to remove. |


### `system.ssh.set_password_auth`

Toggle sshd `PasswordAuthentication` via the engine-managed override file and reload sshd. Refuses to disable when no SSH keys are present.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `enabled` | boolean | no | True to enable password auth, false to disable. |


## System Tailscale

### `system.tailscale.get`

Return the persisted Tailscale config plus the live daemon/connection state (IP, hostname, version).

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `connected` | boolean | yes | Whether Tailscale is connected to the network. |
| `daemon_running` | boolean | yes | Whether the tailscaled daemon is running. |
| `enabled` | boolean | yes | Persisted configuration. |
| `has_auth_key` | boolean | yes | Whether an auth key is configured. |
| `hostname` | string | no | Tailscale hostname. |
| `ip` | string | no | Tailscale IPv4 address (100.x.y.z). |
| `version` | string | no | Tailscale client version. |


### `system.tailscale.connect`

Start the Tailscale daemon and authenticate with the supplied auth key (falling back to the stored key when empty); also re-sync NVMe-oF ports for the new Tailscale IP.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `auth_key` | string | yes |  |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `connected` | boolean | yes | Whether Tailscale is connected to the network. |
| `daemon_running` | boolean | yes | Whether the tailscaled daemon is running. |
| `enabled` | boolean | yes | Persisted configuration. |
| `has_auth_key` | boolean | yes | Whether an auth key is configured. |
| `hostname` | string | no | Tailscale hostname. |
| `ip` | string | no | Tailscale IPv4 address (100.x.y.z). |
| `version` | string | no | Tailscale client version. |


### `system.tailscale.disconnect`

Stop the Tailscale daemon, persist `enabled=false`, and clean up NVMe-oF ports on the 100.x Tailscale IP.

**Role:** `admin`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `connected` | boolean | yes | Whether Tailscale is connected to the network. |
| `daemon_running` | boolean | yes | Whether the tailscaled daemon is running. |
| `enabled` | boolean | yes | Persisted configuration. |
| `has_auth_key` | boolean | yes | Whether an auth key is configured. |
| `hostname` | string | no | Tailscale hostname. |
| `ip` | string | no | Tailscale IPv4 address (100.x.y.z). |
| `version` | string | no | Tailscale client version. |


## System Tuning

### `system.tuning.get`

Return the persisted system-wide NAS performance tuning configuration (NFS/SMB/iSCSI/VM-writeback knobs).

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `iscsi_default_cmdsn_depth` | integer | no | Default command queue depth per iSCSI session. |
| `iscsi_login_timeout` | integer | no | iSCSI login timeout in seconds. |
| `nfs_grace_time` | integer | no | NFSv4 grace period in seconds after server restart. Clients can reclaim locks. |
| `nfs_lease_time` | integer | no | NFSv4 lease time in seconds. Clients must renew state within this window. |
| `nfs_threads` | integer | no | Number of NFS server (nfsd) kernel threads. |
| `smb_deadtime` | integer | no | Minutes before idle SMB clients are disconnected (0 = never). |
| `smb_max_connections` | integer | no | Maximum simultaneous SMB connections (0 = unlimited). |
| `smb_socket_options` | string | no | Samba socket options for TCP tuning (e.g. `SO_RCVBUF=131072 SO_SNDBUF=131072`). |
| `vm_dirty_background_ratio` | integer | no | Dirty page percentage at which background writeback starts. |
| `vm_dirty_expire_centisecs` | integer | no | Centiseconds before dirty pages are old enough to be written out. |
| `vm_dirty_ratio` | integer | no | Maximum percentage of memory that can be dirty before synchronous writeback kicks in. |
| `vm_dirty_writeback_centisecs` | integer | no | Centiseconds between writeback daemon wakeups. |


### `system.tuning.update`

Apply a partial update to the tuning configuration, persist it, and reapply the affected sysctl/Samba/etc. settings.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `iscsi_default_cmdsn_depth` | integer | no |  |
| `iscsi_login_timeout` | integer | no |  |
| `nfs_grace_time` | integer | no |  |
| `nfs_lease_time` | integer | no |  |
| `nfs_threads` | integer | no |  |
| `smb_deadtime` | integer | no |  |
| `smb_max_connections` | integer | no |  |
| `smb_socket_options` | string | no |  |
| `vm_dirty_background_ratio` | integer | no |  |
| `vm_dirty_expire_centisecs` | integer | no |  |
| `vm_dirty_ratio` | integer | no |  |
| `vm_dirty_writeback_centisecs` | integer | no |  |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `iscsi_default_cmdsn_depth` | integer | no | Default command queue depth per iSCSI session. |
| `iscsi_login_timeout` | integer | no | iSCSI login timeout in seconds. |
| `nfs_grace_time` | integer | no | NFSv4 grace period in seconds after server restart. Clients can reclaim locks. |
| `nfs_lease_time` | integer | no | NFSv4 lease time in seconds. Clients must renew state within this window. |
| `nfs_threads` | integer | no | Number of NFS server (nfsd) kernel threads. |
| `smb_deadtime` | integer | no | Minutes before idle SMB clients are disconnected (0 = never). |
| `smb_max_connections` | integer | no | Maximum simultaneous SMB connections (0 = unlimited). |
| `smb_socket_options` | string | no | Samba socket options for TCP tuning (e.g. `SO_RCVBUF=131072 SO_SNDBUF=131072`). |
| `vm_dirty_background_ratio` | integer | no | Dirty page percentage at which background writeback starts. |
| `vm_dirty_expire_centisecs` | integer | no | Centiseconds before dirty pages are old enough to be written out. |
| `vm_dirty_ratio` | integer | no | Maximum percentage of memory that can be dirty before synchronous writeback kicks in. |
| `vm_dirty_writeback_centisecs` | integer | no | Centiseconds between writeback daemon wakeups. |


## System Firewall

### `system.firewall.status`

Return the current firewall rules and per-service source/interface restrictions.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `active` | boolean | yes |  |
| `interface_restrictions` | object | yes | Per-service interface restrictions. |
| `restrictions` | object | yes | Per-service source IP restrictions. |
| `rules` | `FirewallRule`[] | yes |  |


### `system.firewall.restrict`

Set per-service source-IP/interface restrictions and rebuild the nftables rules.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `interfaces` | string[] | no | Allowed interfaces. Omit to clear. |
| `service` | string | yes | Service name (nfs, smb, iscsi, …). |
| `sources` | string[] | no | Allowed source CIDRs. Omit to clear. |


## System Update Channel

### `system.update.channel.get`

Return the currently-selected release channel (Mild/Spicy/Nasty).

**Role:** `any`

**Returns:**

`object`


### `system.update.channel.set`

Persist the selected release channel.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `channel` | string | yes | Channel name (`mild`, `spicy`, or `nasty`). |

**Returns:**

`object`


## Network (continued)

### `system.network.pending`

Return the list of network-update transactions still awaiting confirm-or-rollback. (Admin-only by current role-gate even though it's a read.)

**Role:** `admin`

**Returns:**

``NetworkPendingTxn`[]`


### `system.network.confirm`

Confirm a pending network-change rollback transaction so the new config sticks.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `txn_id` | string | yes |  |


### `system.network.nm_preview`

Compute the diff between desired NetworkManager profiles and NM's current state without applying.

**Role:** `admin`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `summary` | `NmDiffSummary` | yes | Counts for at-a-glance display in the WebUI. |
| `to_add` | string[] | yes | Connection IDs present in the desired set but not currently in
NM — `apply_profiles` calls `Settings.AddConnection` for these. |
| `to_delete` | string[] | yes | NASty-managed IDs in NM but not in the desired set — user must
have removed the link.  `apply_profiles` calls `Connection.Delete`. |
| `to_update` | `NmDiffUpdate`[] | yes | IDs present in both, but with different settings — `apply_profiles`
calls `Connection.Update`. |


### `system.network.nm_apply`

Apply the desired NetworkManager profile set (add/update/delete + activate) to the live system.

**Role:** `admin`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `activated` | string[] | yes | Connection IDs successfully activated this apply. Subset of
`added ∪ updated ∪ unchanged` — we only activate enabled
connections that have a matching NM-managed device. |
| `added` | string[] | yes |  |
| `deleted` | string[] | yes |  |
| `errors` | object | yes | Per-id error map. Empty on full success. The apply is best-
effort — one failed connection doesn't abort the rest. |
| `unchanged` | string[] | yes |  |
| `updated` | string[] | yes |  |


## Protocols & Services (continued)

### `service.base_names.get`

Return the configured iSCSI IQN prefix and NVMe-oF NQN prefix used as the base for newly created targets/subsystems (built-in defaults if no override file exists).

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `iqn_prefix` | string | yes |  |
| `nqn_prefix` | string | yes |  |


### `service.base_names.update`

Persist user-supplied iSCSI IQN and/or NVMe-oF NQN base name prefixes so future target/subsystem creations use them.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `iqn_prefix` | string | no |  |
| `nqn_prefix` | string | no |  |


### `service.rest_server.config`

Return the configured filesystem path used by the embedded `nasty-rest-server` (restic REST API) backup endpoint.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `path` | string | yes |  |


### `service.rest_server.configure`

Set the rest-server storage path, auto-creating a subvolume under `/fs/<name>/...` if needed, persisting the path, and restarting `nasty-rest-server` to pick up the change.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `path` | string | yes | Absolute filesystem path for restic REST API storage. |


## Telemetry

### `telemetry.send`

Trigger an immediate anonymous telemetry report (drive/VM/app counts, version, arch) to the NASty telemetry endpoint. No-op when telemetry is disabled in settings.

**Role:** `admin`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `sent` | boolean | yes | True if the report was transmitted; false when telemetry is disabled. |


## Notifications

### `notifications.config.get`

Return the persisted notification-channels configuration (SMTP / Telegram / Webhook / ntfy / Signal).

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `channels` | `ChannelConfig`[] | yes |  |


### `notifications.config.update`

Replace the on-disk notifications config with the supplied one. File is chmod 0600 because it carries SMTP passwords and bot tokens.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `channels` | `ChannelConfig`[] | yes |  |


### `notifications.test`

Send a one-shot test message ("NASty Test") through the supplied channel configuration without persisting it.

**Role:** `admin`

**Params:**

`object`

**Returns:**

`object`


## Firmware

### `firmware.available`

Report whether firmware management is usable on this host (false inside VMs as detected by `systemd-detect-virt`, true on bare metal).

**Role:** `any`

**Returns:**

`object`


### `firmware.devices`

List every device known to `fwupdmgr` with its name, vendor, device ID, and currently installed firmware version (no update check).

**Role:** `any`

**Returns:**

``FirmwareDevice`[]`


### `firmware.check`

Refresh LVFS metadata via `fwupdmgr refresh` then return the device list with `update_available`/`update_version`/`update_description` populated for devices with pending updates.

**Role:** `any`

**Returns:**

``FirmwareDevice`[]`


### `firmware.constraints`

Return a snapshot of system-level blockers on applying firmware updates (today: whether Secure Boot enforcement is preventing fwupd's capsule shim per lanzaboote#591).

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `sb_blocks_apply` | boolean | yes | True when the engine has detected Secure Boot is enforcing.
The Apply UI surfaces a banner + per-row disable when set;
`firmware.update` itself rejects the call as well. |
| `sb_blocks_apply_reason` | string | yes | Operator-facing reason string, suitable for surfacing
verbatim in a tooltip or banner. Empty when
`sb_blocks_apply` is false (the WebUI hides the banner). |


### `firmware.update`

Apply the available firmware update for the named device via fwupd. Refuses the call if Secure Boot constraints block the capsule-apply path.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device_id` | string | yes | fwupd device identifier (from firmware.devices). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device_name` | string | yes |  |
| `message` | string | yes |  |
| `reboot_required` | boolean | yes | Whether a reboot is required to apply the update. |
| `success` | boolean | yes |  |


## Filesystem Encryption

### `fs.lock`

Lock an encrypted filesystem by unmounting it (with cascading dependent teardown) and unlinking its key from the session keyring.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `FilesystemDevice`[] | yes | Member devices of the filesystem. |
| `mount_point` | string | no | Absolute path where the filesystem is mounted (e.g. `/fs/tank`). |
| `mounted` | boolean | yes | Whether the filesystem is currently mounted. |
| `name` | string | yes | Human-readable filesystem name, derived from the mount point directory. |
| `options` | `FilesystemOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `fs.unlock`

Unlock an encrypted filesystem by passing the supplied passphrase to `bcachefs unlock` against its first device, loading the key into the session keyring.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |
| `passphrase` | string | yes | User-supplied unlock passphrase. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `FilesystemDevice`[] | yes | Member devices of the filesystem. |
| `mount_point` | string | no | Absolute path where the filesystem is mounted (e.g. `/fs/tank`). |
| `mounted` | boolean | yes | Whether the filesystem is currently mounted. |
| `name` | string | yes | Human-readable filesystem name, derived from the mount point directory. |
| `options` | `FilesystemOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |


### `fs.key.export`

Read and return the stored encryption key file contents for the named encrypted filesystem.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |

**Returns:**

`object`


### `fs.key.delete`

Delete the on-disk stored encryption key for an encrypted filesystem, switching it to passphrase-only mode.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |


## Filesystem Reconcile

### `fs.reconcile.enable`

Turn on bcachefs background reconcile work on a mounted filesystem by writing `1` to its sysfs `reconcile_enabled` knob.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |


### `fs.reconcile.disable`

Turn off bcachefs background reconcile work on a mounted filesystem by writing `0` to its sysfs `reconcile_enabled` knob.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |


## Filesystem Dependents

### `fs.dependents`

Return all downstream entities (subvolumes, apps, VMs, backup jobs, NFS/SMB/iSCSI/NVMe-oF shares) that reference a given filesystem, used to preview impact before destructive operations like lock.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `apps` | string[] | yes |  |
| `backup_jobs` | string[] | yes |  |
| `filesystem` | string | yes |  |
| `iscsi_targets` | string[] | yes |  |
| `mounted` | boolean | yes |  |
| `nfs_shares` | string[] | yes |  |
| `nvmeof_subsystems` | string[] | yes |  |
| `smb_shares` | string[] | yes |  |
| `subvolumes` | string[] | yes |  |
| `vms` | string[] | yes |  |


### `fs.locked_dependents`

Return the reverse-index of currently locked encrypted filesystems mapped to their app/VM dependents (for the WebUI's "locked on FS" badges).

**Role:** `any`

**Returns:**

``FsDependents`[]`


## bcachefs Tools

### `bcachefs.top`

Capture ~2 seconds of `bcachefs fs top` output for the named filesystem via a PTY, strip ANSI/header noise, and return the last complete frame as plain text.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |

**Returns:**

`object`


### `bcachefs.timestats`

Run `bcachefs fs timestats --json --once` against the named filesystem's mount point and return the parsed JSON (latency/duration histograms for bcachefs operations).

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Filesystem name. |

**Returns:**

`object`


## Subvolumes (continued)

### `subvolume.children`

List nested child subvolume names found beneath the named parent subvolume on the given filesystem.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filesystem` | string | yes | Filesystem name. |
| `name` | string | yes | Parent subvolume name. |

**Returns:**

`string[]`


### `subvolume.clone`

Create a writable COW clone of a subvolume by taking a non-read-only bcachefs snapshot under a new name (O(1), shares data blocks with the source).

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filesystem` | string | yes | Name of the filesystem containing the source subvolume. |
| `name` | string | yes | Name of the subvolume to clone. |
| `new_name` | string | yes | Name for the new writable subvolume. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bcachefs_options` | object | no | Effective bcachefs options set on this subvolume (from bcachefs_effective.* xattrs).
Only includes options that differ from the filesystem default. |
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `direct_io` | boolean | no | Whether O_DIRECT is enabled on the loop device (block subvolumes only). |
| `filesystem` | string | yes | Name of the filesystem that contains this subvolume. |
| `name` | string | yes | Subvolume name (unique within the filesystem). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `parent` | string | no | Parent subvolume name if this is a clone (from bcachefs snapshot_parent). |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `quota_bytes` | integer | no | Hard quota limit in bytes for filesystem subvolumes. `None`
means no limit set (the subvolume can grow to fill the
filesystem). Always `None` for block subvolumes — their
ceiling is `volsize_bytes`, not a quota. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes. For filesystem subvolumes, comes from the
per-project quota (set on every create, so tracking is always
on); `None` only on legacy subvolumes created before the
always-track change. For block subvolumes, comes from the
backing image's allocated size. |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |


### `subvolume.update`

Update mutable subvolume attributes (compression, comments, foreground/background/promote/metadata targets, data replicas) via bcachefs `set-file-option` and xattrs.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `background_target` | string | no | Device or label for background moves/recompression. Use `-` to remove. |
| `comments` | string | no | New description for the subvolume. Empty string clears the comment. |
| `compression` | string | no | New compression algorithm (e.g. `lz4`, `zstd`, `none`). `none` clears compression. |
| `data_replicas` | integer | no | Number of data replicas. Use `0` to reset to filesystem default. |
| `filesystem` | string | yes | Name of the filesystem containing the subvolume. |
| `foreground_target` | string | no | Device or label for foreground writes. Use `-` to remove. |
| `metadata_target` | string | no | Device or label for metadata/btree writes. Use `-` to remove. |
| `name` | string | yes | Name of the subvolume to update. |
| `promote_target` | string | no | Device or label to promote data to on read. Use `-` to remove. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bcachefs_options` | object | no | Effective bcachefs options set on this subvolume (from bcachefs_effective.* xattrs).
Only includes options that differ from the filesystem default. |
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `direct_io` | boolean | no | Whether O_DIRECT is enabled on the loop device (block subvolumes only). |
| `filesystem` | string | yes | Name of the filesystem that contains this subvolume. |
| `name` | string | yes | Subvolume name (unique within the filesystem). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `parent` | string | no | Parent subvolume name if this is a clone (from bcachefs snapshot_parent). |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `quota_bytes` | integer | no | Hard quota limit in bytes for filesystem subvolumes. `None`
means no limit set (the subvolume can grow to fill the
filesystem). Always `None` for block subvolumes — their
ceiling is `volsize_bytes`, not a quota. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes. For filesystem subvolumes, comes from the
per-project quota (set on every create, so tracking is always
on); `None` only on legacy subvolumes created before the
always-track change. For block subvolumes, comes from the
backing image's allocated size. |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |


### `subvolume.list_dependents`

Batched read returning the set of downstream entities (apps, VMs, backup jobs, shares of every protocol) attributed to each subvolume on the system, optionally filtered to the session's scoped filesystem.

**Role:** `any`

**Returns:**

``SubvolumeDependents`[]`


## SMB Users

### `smb.user.list`

List SMB users by parsing `pdbedit -L` output and filtering to UIDs ≥ 3000.

**Role:** `any`

**Returns:**

``SmbUser`[]`


### `smb.user.create`

Create a Linux system user (no shell, no home, UID auto-assigned from 3000+) and set their Samba password. Requires the SMB protocol to be enabled.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `password` | string | yes | Password for SMB authentication. |
| `username` | string | yes | Username (alphanumeric + hyphens, 1-32 chars). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `uid` | integer | yes | Unix UID. |
| `username` | string | yes | Linux username. |


### `smb.user.delete`

Remove the user's Samba password entry and delete the Linux system account. Requires the SMB protocol to be enabled.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `username` | string | yes | SMB username to delete. |


### `smb.user.set_password`

Change an existing SMB user's Samba password. Requires the SMB protocol to be enabled.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `password` | string | yes | New password. |
| `username` | string | yes | Existing SMB username. |


## SMB Groups

### `smb.group.list`

List SMB-managed groups (GIDs in the 3000-3999 range) read from `/etc/group`, including members.

**Role:** `any`

**Returns:**

``SmbGroup`[]`


### `smb.group.create`

Create a Linux system group (GID auto-assigned from the SMB range, 3000+) used for SMB access control.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Group name to create. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `gid` | integer | yes | Unix GID. |
| `members` | string[] | yes | Group members (usernames). |
| `name` | string | yes | Linux group name. |


### `smb.group.delete`

Delete the SMB-managed Linux group via `groupdel`.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Group name to delete. |


### `smb.group.add_member`

Add an existing user to an existing SMB group via `usermod -aG`.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `group` | string | yes | Group name. |
| `user` | string | yes | Username to add. |


### `smb.group.remove_member`

Remove a user from an SMB group via `gpasswd -d`.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `group` | string | yes | Group name. |
| `user` | string | yes | Username to remove. |


## Backup

### `backup.status`

Report whether any backup is currently running and which profile id it belongs to.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `profile_id` | string | no |  |
| `progress` | string | no |  |
| `running` | boolean | yes |  |


### `backup.profile.list`

Return all configured backup profiles.

**Role:** `any`

**Returns:**

``BackupProfile`[]`


### `backup.profile.get`

Return a single backup profile by id.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | Backup profile identifier. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `enabled` | boolean | yes |  |
| `id` | string | yes |  |
| `last_run` | `BackupRunResult` \| null | no |  |
| `name` | string | yes |  |
| `password` | string | yes |  |
| `repo_initialized` | boolean | no |  |
| `retention` | `RetentionPolicy` | no |  |
| `schedule` | string | no |  |
| `snapshot_before` | boolean | no |  |
| `sources` | string[] | yes |  |
| `target` | `BackupTarget` | yes |  |


### `backup.profile.create`

Create a new backup profile (name, sources, target, password, retention) and persist it to `/var/lib/nasty/backups.json`. Returns the stored profile with an auto-assigned 8-char UUID id when the caller leaves `id` empty.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `enabled` | boolean | yes |  |
| `id` | string | yes |  |
| `last_run` | `BackupRunResult` \| null | no |  |
| `name` | string | yes |  |
| `password` | string | yes |  |
| `repo_initialized` | boolean | no |  |
| `retention` | `RetentionPolicy` | no |  |
| `schedule` | string | no |  |
| `snapshot_before` | boolean | no |  |
| `sources` | string[] | yes |  |
| `target` | `BackupTarget` | yes |  |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `enabled` | boolean | yes |  |
| `id` | string | yes |  |
| `last_run` | `BackupRunResult` \| null | no |  |
| `name` | string | yes |  |
| `password` | string | yes |  |
| `repo_initialized` | boolean | no |  |
| `retention` | `RetentionPolicy` | no |  |
| `schedule` | string | no |  |
| `snapshot_before` | boolean | no |  |
| `sources` | string[] | yes |  |
| `target` | `BackupTarget` | yes |  |


### `backup.profile.update`

Replace the backup profile identified by `id` with the supplied profile body. The handler reads `id` from params *and* deserializes the same params object into BackupProfile — send the full BackupProfile shape with `id` populated.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `enabled` | boolean | yes |  |
| `id` | string | yes |  |
| `last_run` | `BackupRunResult` \| null | no |  |
| `name` | string | yes |  |
| `password` | string | yes |  |
| `repo_initialized` | boolean | no |  |
| `retention` | `RetentionPolicy` | no |  |
| `schedule` | string | no |  |
| `snapshot_before` | boolean | no |  |
| `sources` | string[] | yes |  |
| `target` | `BackupTarget` | yes |  |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `enabled` | boolean | yes |  |
| `id` | string | yes |  |
| `last_run` | `BackupRunResult` \| null | no |  |
| `name` | string | yes |  |
| `password` | string | yes |  |
| `repo_initialized` | boolean | no |  |
| `retention` | `RetentionPolicy` | no |  |
| `schedule` | string | no |  |
| `snapshot_before` | boolean | no |  |
| `sources` | string[] | yes |  |
| `target` | `BackupTarget` | yes |  |


### `backup.profile.delete`

Remove the backup profile with the given id from persisted state.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | Backup profile identifier. |


### `backup.run`

Spawn a background task that runs the profile's backup (auto-initializing the repo if needed, then pruning per the retention policy); the RPC returns immediately after accepting the request.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | Backup profile identifier. |

**Returns:**

`object`


### `backup.snapshots`

List all snapshots stored in the profile's repository (id, time, hostname, paths, tags).

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | Backup profile identifier. |

**Returns:**

``BackupSnapshot`[]`


### `backup.repo.init`

Initialize a fresh rustic repository at the profile's target using its password, then mark the profile as `repo_initialized`.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | Backup profile identifier. |

**Returns:**

`object`


### `backup.repo.check`

Run a rustic repository integrity check (`repo.check`) on the profile's target repo and return a status message.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | Backup profile identifier. |

**Returns:**

`object`


## VMs

### `vm.capabilities`

Report host VM capabilities (KVM availability, UEFI firmware availability, CPU arch, and PCI devices available for passthrough).

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `arch` | string | yes | CPU architecture (e.g. "x86_64", "aarch64"). |
| `kvm_available` | boolean | yes | Whether KVM hardware acceleration is available. |
| `passthrough_devices` | `PciDevice2`[] | yes | Available PCI devices for passthrough. |
| `uefi_available` | boolean | yes | Whether OVMF UEFI firmware is available. |


### `vm.list`

List all VMs with their current status (config plus running/pid/vnc_port).

**Role:** `any`

**Returns:**

``VmStatus`[]`


### `vm.get`

Return full status (config plus running/pid/vnc_port) for a single VM by id.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | VM identifier. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `autostart` | boolean | no | Whether the VM should auto-start on NASty boot. |
| `boot_iso` | string | no | Legacy single-ISO field, kept for cross-version state-file
compatibility. On load we migrate this into `cdroms` if
`cdroms` is empty; on save we mirror `cdroms.first()` back
into here so a hypothetical rollback to a pre-`cdroms` engine
still sees the boot ISO. New code reads `cdroms` exclusively. |
| `boot_order` | string | no | Boot order: "disk", "cdrom", or "network". |
| `cdroms` | string[] | no | ISO files to attach as CD-ROM devices. The first entry is the
one QEMU treats as the boot CD when `boot_order = "cdrom"`;
additional entries show up as extra read-only CDs inside the
guest (typical use: Windows 11 install needs the Win11 ISO
alongside the virtio-win driver ISO so the installer can see
the virtio storage controller — issue #285). |
| `cpu_model` | string | no | CPU model: "host" (default), "max", "qemu64", etc. |
| `cpus` | integer | yes | Number of virtual CPU cores. |
| `description` | string | no | Optional description. |
| `disks` | `VmDisk`[] | yes | Boot disk configuration. |
| `extra_args` | string[] | no | Extra raw QEMU arguments for advanced users. |
| `id` | string | yes | Unique VM identifier (UUID). |
| `machine_type` | string | no | Machine type: "q35" (default for x86), "i440fx". |
| `memory_mib` | integer | yes | RAM in MiB. |
| `name` | string | yes | Human-readable name. |
| `networks` | `VmNetwork`[] | yes | Network interfaces. |
| `passthrough_devices` | `PassthroughDevice`[] | yes | PCI devices to pass through via VFIO. |
| `pid` | integer | no | QEMU process PID (if running). |
| `running` | boolean | yes | Whether the VM is currently running. |
| `uefi` | boolean | no | Whether to use UEFI boot (default: true). |
| `usb_devices` | `UsbPassthrough`[] | no | USB devices to pass through. Identified by vendor/product ID
rather than bus/addr because USB enumeration order shuffles
across reboots; pinning to IDs is the stable choice. Caveat:
all devices matching a (vendor, product) pair attach, so
plugging in two identical keyboards passes both through. |
| `vga` | string | no | VGA device type: "virtio" (default), "qxl", "std", "none". |
| `vnc_port` | integer | no | VNC display port (if running, for console access). |


### `vm.create`

Create a new VM config from the supplied spec (name, CPUs, memory, disks, networks, passthrough devices, boot options, etc.).

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `autostart` | boolean | no | Auto-start on NASty boot (default: false). |
| `boot_iso` | string | no | Legacy single-ISO field. When set and `cdroms` is unset, the
engine treats it as `cdroms = vec![boot_iso]`. Kept for
clients that haven't been updated to send the new field. |
| `boot_order` | string | no | Boot order: "disk", "cdrom", or "network". |
| `cdroms` | string[] | no | ISO files to attach as CD-ROM devices. First entry boots when
`boot_order = "cdrom"`. See #285 for the Windows-install
motivating case (Win11 ISO + virtio-win driver ISO). |
| `cpus` | integer | no | Number of virtual CPU cores (default: 1). |
| `description` | string | no | Description. |
| `disks` | `VmDisk`[] | no | Block device paths for VM disks. |
| `memory_mib` | integer | no | RAM in MiB (default: 1024). |
| `name` | string | yes | Human-readable name. |
| `networks` | `VmNetwork`[] | no | Network configuration. |
| `passthrough_devices` | `PassthroughDevice`[] | no | PCI devices to pass through. |
| `uefi` | boolean | no | Use UEFI boot (default: true). |
| `usb_devices` | `UsbPassthrough`[] | no | USB devices to pass through (vendor:product pairs). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `autostart` | boolean | no | Whether the VM should auto-start on NASty boot. |
| `boot_iso` | string | no | Legacy single-ISO field, kept for cross-version state-file
compatibility. On load we migrate this into `cdroms` if
`cdroms` is empty; on save we mirror `cdroms.first()` back
into here so a hypothetical rollback to a pre-`cdroms` engine
still sees the boot ISO. New code reads `cdroms` exclusively. |
| `boot_order` | string | no | Boot order: "disk", "cdrom", or "network". |
| `cdroms` | string[] | no | ISO files to attach as CD-ROM devices. The first entry is the
one QEMU treats as the boot CD when `boot_order = "cdrom"`;
additional entries show up as extra read-only CDs inside the
guest (typical use: Windows 11 install needs the Win11 ISO
alongside the virtio-win driver ISO so the installer can see
the virtio storage controller — issue #285). |
| `cpu_model` | string | no | CPU model: "host" (default), "max", "qemu64", etc. |
| `cpus` | integer | yes | Number of virtual CPU cores. |
| `description` | string | no | Optional description. |
| `disks` | `VmDisk`[] | yes | Boot disk configuration. |
| `extra_args` | string[] | no | Extra raw QEMU arguments for advanced users. |
| `id` | string | yes | Unique VM identifier (UUID). |
| `machine_type` | string | no | Machine type: "q35" (default for x86), "i440fx". |
| `memory_mib` | integer | yes | RAM in MiB. |
| `name` | string | yes | Human-readable name. |
| `networks` | `VmNetwork`[] | yes | Network interfaces. |
| `passthrough_devices` | `PassthroughDevice`[] | yes | PCI devices to pass through via VFIO. |
| `uefi` | boolean | no | Whether to use UEFI boot (default: true). |
| `usb_devices` | `UsbPassthrough`[] | no | USB devices to pass through. Identified by vendor/product ID
rather than bus/addr because USB enumeration order shuffles
across reboots; pinning to IDs is the stable choice. Caveat:
all devices matching a (vendor, product) pair attach, so
plugging in two identical keyboards passes both through. |
| `vga` | string | no | VGA device type: "virtio" (default), "qxl", "std", "none". |


### `vm.update`

Apply partial edits to an existing VM's config (name, CPUs, memory, disks, networks, passthrough, CD-ROMs, boot order, UEFI, autostart, etc.). Hardware changes require the VM to be stopped.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `autostart` | boolean | no | Auto-start. |
| `boot_iso` | string | no | Legacy single-ISO setter. When set and `cdroms` is absent,
the engine treats an empty string as "clear all CD-ROMs" and
a non-empty string as "set CD-ROM list to a single entry."
Use `cdroms` for new code. |
| `boot_order` | string | no | Boot order. |
| `cdroms` | string[] | no | Replace the CD-ROM list. Empty vec clears all CD-ROMs; absent
(`None`) leaves the existing list untouched. |
| `cpu_model` | string | no | CPU model. |
| `cpus` | integer | no | New CPU count. |
| `description` | string | no | Description. |
| `disks` | `VmDisk`[] | no | Replace disk list. |
| `extra_args` | string[] | no | Extra raw QEMU arguments. |
| `id` | string | yes | VM ID. |
| `machine_type` | string | no | Machine type. |
| `memory_mib` | integer | no | New RAM in MiB. |
| `name` | string | no | New name. |
| `networks` | `VmNetwork`[] | no | Replace network list. |
| `passthrough_devices` | `PassthroughDevice`[] | no | Replace passthrough devices. |
| `uefi` | boolean | no | UEFI setting. |
| `usb_devices` | `UsbPassthrough`[] | no | Replace USB passthrough devices. |
| `vga` | string | no | VGA device type. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `autostart` | boolean | no | Whether the VM should auto-start on NASty boot. |
| `boot_iso` | string | no | Legacy single-ISO field, kept for cross-version state-file
compatibility. On load we migrate this into `cdroms` if
`cdroms` is empty; on save we mirror `cdroms.first()` back
into here so a hypothetical rollback to a pre-`cdroms` engine
still sees the boot ISO. New code reads `cdroms` exclusively. |
| `boot_order` | string | no | Boot order: "disk", "cdrom", or "network". |
| `cdroms` | string[] | no | ISO files to attach as CD-ROM devices. The first entry is the
one QEMU treats as the boot CD when `boot_order = "cdrom"`;
additional entries show up as extra read-only CDs inside the
guest (typical use: Windows 11 install needs the Win11 ISO
alongside the virtio-win driver ISO so the installer can see
the virtio storage controller — issue #285). |
| `cpu_model` | string | no | CPU model: "host" (default), "max", "qemu64", etc. |
| `cpus` | integer | yes | Number of virtual CPU cores. |
| `description` | string | no | Optional description. |
| `disks` | `VmDisk`[] | yes | Boot disk configuration. |
| `extra_args` | string[] | no | Extra raw QEMU arguments for advanced users. |
| `id` | string | yes | Unique VM identifier (UUID). |
| `machine_type` | string | no | Machine type: "q35" (default for x86), "i440fx". |
| `memory_mib` | integer | yes | RAM in MiB. |
| `name` | string | yes | Human-readable name. |
| `networks` | `VmNetwork`[] | yes | Network interfaces. |
| `passthrough_devices` | `PassthroughDevice`[] | yes | PCI devices to pass through via VFIO. |
| `uefi` | boolean | no | Whether to use UEFI boot (default: true). |
| `usb_devices` | `UsbPassthrough`[] | no | USB devices to pass through. Identified by vendor/product ID
rather than bus/addr because USB enumeration order shuffles
across reboots; pinning to IDs is the stable choice. Caveat:
all devices matching a (vendor, product) pair attach, so
plugging in two identical keyboards passes both through. |
| `vga` | string | no | VGA device type: "virtio" (default), "qxl", "std", "none". |


### `vm.delete`

Delete the VM config identified by `id` (does not remove backing disk subvolumes).

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | VM identifier. |


### `vm.start`

Launch QEMU for the VM identified by `id` and return its updated status.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | VM identifier. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `autostart` | boolean | no | Whether the VM should auto-start on NASty boot. |
| `boot_iso` | string | no | Legacy single-ISO field, kept for cross-version state-file
compatibility. On load we migrate this into `cdroms` if
`cdroms` is empty; on save we mirror `cdroms.first()` back
into here so a hypothetical rollback to a pre-`cdroms` engine
still sees the boot ISO. New code reads `cdroms` exclusively. |
| `boot_order` | string | no | Boot order: "disk", "cdrom", or "network". |
| `cdroms` | string[] | no | ISO files to attach as CD-ROM devices. The first entry is the
one QEMU treats as the boot CD when `boot_order = "cdrom"`;
additional entries show up as extra read-only CDs inside the
guest (typical use: Windows 11 install needs the Win11 ISO
alongside the virtio-win driver ISO so the installer can see
the virtio storage controller — issue #285). |
| `cpu_model` | string | no | CPU model: "host" (default), "max", "qemu64", etc. |
| `cpus` | integer | yes | Number of virtual CPU cores. |
| `description` | string | no | Optional description. |
| `disks` | `VmDisk`[] | yes | Boot disk configuration. |
| `extra_args` | string[] | no | Extra raw QEMU arguments for advanced users. |
| `id` | string | yes | Unique VM identifier (UUID). |
| `machine_type` | string | no | Machine type: "q35" (default for x86), "i440fx". |
| `memory_mib` | integer | yes | RAM in MiB. |
| `name` | string | yes | Human-readable name. |
| `networks` | `VmNetwork`[] | yes | Network interfaces. |
| `passthrough_devices` | `PassthroughDevice`[] | yes | PCI devices to pass through via VFIO. |
| `pid` | integer | no | QEMU process PID (if running). |
| `running` | boolean | yes | Whether the VM is currently running. |
| `uefi` | boolean | no | Whether to use UEFI boot (default: true). |
| `usb_devices` | `UsbPassthrough`[] | no | USB devices to pass through. Identified by vendor/product ID
rather than bus/addr because USB enumeration order shuffles
across reboots; pinning to IDs is the stable choice. Caveat:
all devices matching a (vendor, product) pair attach, so
plugging in two identical keyboards passes both through. |
| `vga` | string | no | VGA device type: "virtio" (default), "qxl", "std", "none". |
| `vnc_port` | integer | no | VNC display port (if running, for console access). |


### `vm.stop`

Gracefully stop the running VM identified by `id` (ACPI shutdown via QMP).

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | VM identifier. |


### `vm.kill`

Force-terminate the QEMU process for a running VM by `id` (ungraceful — used when `vm.stop` won't return).

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | VM identifier. |


### `vm.snapshot`

Snapshot every block subvolume backing a VM under a shared name, freezing the guest filesystem via QMP `guest-fsfreeze-freeze` first if the VM is running.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | VM ID. |
| `name` | string | yes | Snapshot name (applied to all disk subvolumes). |

**Returns:**

``VmDiskSubvolume`[]`


### `vm.clone`

Clone a stopped VM by COW-cloning each of its disk subvolumes and creating a new VM config that points at those clones.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `id` | string | yes | Source VM ID. |
| `new_name` | string | yes | Name for the cloned VM. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `autostart` | boolean | no | Whether the VM should auto-start on NASty boot. |
| `boot_iso` | string | no | Legacy single-ISO field, kept for cross-version state-file
compatibility. On load we migrate this into `cdroms` if
`cdroms` is empty; on save we mirror `cdroms.first()` back
into here so a hypothetical rollback to a pre-`cdroms` engine
still sees the boot ISO. New code reads `cdroms` exclusively. |
| `boot_order` | string | no | Boot order: "disk", "cdrom", or "network". |
| `cdroms` | string[] | no | ISO files to attach as CD-ROM devices. The first entry is the
one QEMU treats as the boot CD when `boot_order = "cdrom"`;
additional entries show up as extra read-only CDs inside the
guest (typical use: Windows 11 install needs the Win11 ISO
alongside the virtio-win driver ISO so the installer can see
the virtio storage controller — issue #285). |
| `cpu_model` | string | no | CPU model: "host" (default), "max", "qemu64", etc. |
| `cpus` | integer | yes | Number of virtual CPU cores. |
| `description` | string | no | Optional description. |
| `disks` | `VmDisk`[] | yes | Boot disk configuration. |
| `extra_args` | string[] | no | Extra raw QEMU arguments for advanced users. |
| `id` | string | yes | Unique VM identifier (UUID). |
| `machine_type` | string | no | Machine type: "q35" (default for x86), "i440fx". |
| `memory_mib` | integer | yes | RAM in MiB. |
| `name` | string | yes | Human-readable name. |
| `networks` | `VmNetwork`[] | yes | Network interfaces. |
| `passthrough_devices` | `PassthroughDevice`[] | yes | PCI devices to pass through via VFIO. |
| `uefi` | boolean | no | Whether to use UEFI boot (default: true). |
| `usb_devices` | `UsbPassthrough`[] | no | USB devices to pass through. Identified by vendor/product ID
rather than bus/addr because USB enumeration order shuffles
across reboots; pinning to IDs is the stable choice. Caveat:
all devices matching a (vendor, product) pair attach, so
plugging in two identical keyboards passes both through. |
| `vga` | string | no | VGA device type: "virtio" (default), "qxl", "std", "none". |


## VM Disk Images

### `vm.images.list`

List VM image files found under `vms/images` across all mounted filesystems, with per-image name/path/filesystem/size/format/compression, plus a flag indicating whether any such directory exists.

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `images` | object[] | yes |  |
| `subvolume_exists` | boolean | yes | True if at least one `vms/images` directory was found. |


### `vm.images.ensure`

Ensure the `vms/images` directory exists on the named filesystem (creating it, and migrating from legacy `.nasty/images` if present); return the absolute images directory path.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filesystem` | string | yes | Filesystem name. |

**Returns:**

`object`


### `vm.images.import_info`

Pre-flight inspection for the disk-import WebSocket — return format, virtual size, actual size, and (if applicable) compression for a VM image file under `vms/images` on the named filesystem.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filesystem` | string | yes | Filesystem name. |
| `name` | string | yes | Image filename. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `actual_size` | integer | yes |  |
| `compression` | string | no | Compression algorithm if any. |
| `format` | string | yes | Image format (qcow2, raw, …). |
| `virtual_size` | integer | yes |  |


## Apps

### `apps.status`

Return runtime status of the apps subsystem (enabled flag, Docker running, app count, memory usage, storage path/health, Docker version, total disk usage).

**Role:** `any`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `app_count` | integer | yes | Number of managed apps (running or stopped). |
| `disk_usage_bytes` | integer | no | Docker disk usage: images + containers + volumes in bytes. |
| `docker_version` | string | no | Docker server version. |
| `enabled` | boolean | yes | Whether the apps runtime is enabled. |
| `memory_bytes` | integer | no | Total memory usage of managed containers in bytes. |
| `running` | boolean | yes | Whether Docker is currently running and responsive. |
| `storage_ok` | boolean | yes | Whether the storage directory exists on disk. |
| `storage_path` | string | no | Path to the apps storage directory on bcachefs. |


### `apps.list`

List every NASty-managed app (both simple and compose), returning each one's high-level App record.

**Role:** `any`

**Returns:**

``App`[]`


### `apps.get`

Return the high-level App record (name, image, status, kind, containers, ports, unsafe_mode, proxy_disabled_reason) for a single named app.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | App name. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `containers` | `AppContainer`[] | no | Individual containers (for compose apps with multiple services). |
| `created` | string | yes | ISO 8601 timestamp of when the container was created. |
| `image` | string | yes | Container image (primary image for compose apps). |
| `kind` | string | yes | App kind: "simple" or "compose". |
| `name` | string | yes | App name (container name for simple, project name for compose). |
| `ports` | `MappedPort`[] | no | Host ports mapped by this app. |
| `proxy_disabled_reason` | string | no | Human-readable reason the reverse-proxy ingress was disabled for
this app — set when the post-install probe detects that the
upstream emits absolute root-path assets that path-prefix proxying
can't route (haze-class apps). The WebUI hides the "Open" button
when this is set and surfaces the text as a tooltip explaining
why only the direct host-port link is offered. |
| `status` | string | yes | Current status: "running", "stopped", "restarting", "created", "exited". |
| `unsafe_mode` | boolean | no | True if the app was deployed with allow_unsafe — i.e. it has elevated
privileges (caps, host devices, host namespaces, or bind mounts
outside the standard sandbox). Surfaced as a badge in the WebUI. |


### `apps.config`

Return the deployed configuration of a named simple app (image, ports, env, volumes, resource limits, allow_unsafe), with env entries tagged where they match the image's own defaults so the WebUI Edit form can grey them out.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | App name. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `allow_unsafe` | boolean | no | Whether the app was deployed with allow_unsafe (read from container label). |
| `cpu_limit` | string | no |  |
| `env` | `AppEnv`[] | yes |  |
| `image` | string | yes |  |
| `memory_limit` | string | no |  |
| `name` | string | yes |  |
| `ports` | `AppPort`[] | yes |  |
| `volumes` | `AppVolume`[] | yes |  |


### `apps.stats`

Return live CPU / memory / network / block-IO stats for every NASty-managed app, summed across containers for compose apps.

**Role:** `any`

**Returns:**

``AppStats`[]`


### `apps.logs`

Return Docker logs for a named simple app's container (the `nasty-<name>` container), defaulting to the last 100 lines unless `tail` overrides it.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | App name. |
| `tail` | integer | no | Number of log lines from the tail. |

**Returns:**

`object`


### `apps.container.logs`

Return Docker logs for an arbitrary container by ID or name (no `nasty-` prefix assumed), defaulting to the last 100 lines unless `tail` overrides it.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `container_id` | string | yes | Container ID or name. |
| `tail` | integer | no |  |

**Returns:**

`object`


### `apps.inspect`

Return the raw Docker `inspect` JSON for a named simple app's container as an untyped object.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | App name. |

**Returns:**

`object`


### `apps.inspect_image`

Inspect a container image (registry/local) and return its declared ports, VOLUME bind paths, runtime user, and any known sub-path recipe — used by the install wizard to prefill the form.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `image` | string | yes | Image reference (`repo:tag`). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `ports` | `AppPort`[] | yes |  |
| `subpath_recipe` | `SubPathRecipe` \| null | no | Known recipe for configuring this image to serve under
`/apps/<name>/` behind NASty's path-prefix reverse proxy. When
present, the WebUI offers an "Apply" button that appends the
recipe's env entries to the form (the user can still edit them).
Catches apps that *could* run behind a sub-path but only with
specific env vars set — e.g. Grafana needs `GF_SERVER_ROOT_URL`
plus `GF_SERVER_SERVE_FROM_SUB_PATH=true`; without those, our
post-install probe would (correctly) disable ingress and the
user would only see the direct-port link, even though a one-line
env change would have made the proxy work. |
| `user` | string | no | Image's runtime user as declared in `Config.User`. May be numeric
(`1000` / `1000:1000`) or named (`nonroot:nonroot`). The WebUI
surfaces this so the user knows the host volume dirs will be
chowned to that identity by the install pipeline. `None` = root. |
| `volumes` | `AppVolume`[] | no | Bind-mount paths the image declares via `VOLUME` in its Dockerfile.
The WebUI installer prefills these as Volume rows so the user
doesn't have to know that e.g. ghcr.io/consi/haze needs
`/var/lib/haze` to be persistent for SQLite to work. |


### `apps.caddy.routes`

Return every route Caddy is currently serving (engine-owned and static), enriched with on-disk TLS cert info for host-match rows — powers the Ingress overview page.

**Role:** `any`

**Returns:**

``CaddyRouteSummary`[]`


### `apps.check_ports`

Check a list of host ports for conflicts against other managed apps and system listeners, returning each conflicting port and what is using it.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `exclude_app` | string | no | App name to exclude from conflict check (for updates). |
| `ports` | integer[] | yes | Ports to check for conflicts. |

**Returns:**

``PortConflict`[]`


### `apps.check_devices`

Stat a list of host device paths and report which are missing, including whether the parent directory exists to distinguish "device absent" from "kernel module not loaded".

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `paths` | string[] | yes | Host device paths to check, e.g. `/dev/dri/renderD128`. Anything
after the first colon (the in-container path or cgroup perms) is
the caller's job to strip — this RPC only stat()s host paths. |

**Returns:**

``DeviceMissing`[]`


### `apps.check_volumes`

Parse a docker-compose YAML and report bind-mount source paths whose host owner does not match the container's runtime user (or whose source path is missing).

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `compose` | string | yes | Full docker-compose YAML text. Server parses it and stat()s each
bind-mount source. Sent in full (rather than per-volume) so the
server can correlate sources with their owning service's `user:`
field — that's the comparison we make. |

**Returns:**

``VolumeMismatch`[]`


### `apps.enable`

Enable the apps runtime on this box (optionally pinning the storage filesystem) and start Docker.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `filesystem` | string | no | Filesystem to store app data on. |


### `apps.disable`

Disable the apps runtime on this box (stop Docker and clear the persisted enabled flag in AppsConfig).

**Role:** `operator`


### `apps.install`

Deploy a new simple (single-container) app: validate bind mounts, pull the image, create volume dirs with the image's runtime uid/gid, start the container, and optionally wire up subdomain or path-prefix ingress.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `allow_unsafe` | boolean | no | Opt out of the strict bind-mount allowlist. Admin-only / audited /
surfaced as a badge in the UI. Engine state and the host root are
still rejected even with this set. |
| `cpu_limit` | string | no | CPU limit (e.g. "0.5" for half a core, "2" for 2 cores). |
| `env` | `AppEnv`[] | no | Environment variables. |
| `image` | string | yes | Container image (e.g. "lscr.io/linuxserver/plex:latest"). |
| `memory_limit` | string | no | Memory limit (e.g. "256m", "1g"). |
| `name` | string | yes | App name. Must be DNS-safe. |
| `ports` | `AppPort`[] | no | Ports to expose. |
| `subdomain` | string | no | Optional FQDN to serve the app at via subdomain mode (e.g.
`jellyfin.example.com`). When set, the install pipeline emits a
host-matching Caddy route instead of the default path-prefix
route, and skips the post-install probe (subdomain mode roots
the app at `/`, so the absolute-asset-path failure mode the
probe catches can't happen). Empty/omitted = path-prefix
behaviour, the historical default.

Conflict detection happens at the engine-binary layer before
install runs (see deploy_simple in app_deploy.rs) so the
operator doesn't pay for an image pull just to discover the
hostname is taken. |
| `volumes` | `AppVolume`[] | no | Bind-mount volumes. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `containers` | `AppContainer`[] | no | Individual containers (for compose apps with multiple services). |
| `created` | string | yes | ISO 8601 timestamp of when the container was created. |
| `image` | string | yes | Container image (primary image for compose apps). |
| `kind` | string | yes | App kind: "simple" or "compose". |
| `name` | string | yes | App name (container name for simple, project name for compose). |
| `ports` | `MappedPort`[] | no | Host ports mapped by this app. |
| `proxy_disabled_reason` | string | no | Human-readable reason the reverse-proxy ingress was disabled for
this app — set when the post-install probe detects that the
upstream emits absolute root-path assets that path-prefix proxying
can't route (haze-class apps). The WebUI hides the "Open" button
when this is set and surfaces the text as a tooltip explaining
why only the direct host-port link is offered. |
| `status` | string | yes | Current status: "running", "stopped", "restarting", "created", "exited". |
| `unsafe_mode` | boolean | no | True if the app was deployed with allow_unsafe — i.e. it has elevated
privileges (caps, host devices, host namespaces, or bind mounts
outside the standard sandbox). Surfaced as a badge in the WebUI. |


### `apps.update`

Update a previously installed simple app in place by re-running the install pipeline against the new InstallAppRequest (same shape as install).

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `allow_unsafe` | boolean | no | Opt out of the strict bind-mount allowlist. Admin-only / audited /
surfaced as a badge in the UI. Engine state and the host root are
still rejected even with this set. |
| `cpu_limit` | string | no | CPU limit (e.g. "0.5" for half a core, "2" for 2 cores). |
| `env` | `AppEnv`[] | no | Environment variables. |
| `image` | string | yes | Container image (e.g. "lscr.io/linuxserver/plex:latest"). |
| `memory_limit` | string | no | Memory limit (e.g. "256m", "1g"). |
| `name` | string | yes | App name. Must be DNS-safe. |
| `ports` | `AppPort`[] | no | Ports to expose. |
| `subdomain` | string | no | Optional FQDN to serve the app at via subdomain mode (e.g.
`jellyfin.example.com`). When set, the install pipeline emits a
host-matching Caddy route instead of the default path-prefix
route, and skips the post-install probe (subdomain mode roots
the app at `/`, so the absolute-asset-path failure mode the
probe catches can't happen). Empty/omitted = path-prefix
behaviour, the historical default.

Conflict detection happens at the engine-binary layer before
install runs (see deploy_simple in app_deploy.rs) so the
operator doesn't pay for an image pull just to discover the
hostname is taken. |
| `volumes` | `AppVolume`[] | no | Bind-mount volumes. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `containers` | `AppContainer`[] | no | Individual containers (for compose apps with multiple services). |
| `created` | string | yes | ISO 8601 timestamp of when the container was created. |
| `image` | string | yes | Container image (primary image for compose apps). |
| `kind` | string | yes | App kind: "simple" or "compose". |
| `name` | string | yes | App name (container name for simple, project name for compose). |
| `ports` | `MappedPort`[] | no | Host ports mapped by this app. |
| `proxy_disabled_reason` | string | no | Human-readable reason the reverse-proxy ingress was disabled for
this app — set when the post-install probe detects that the
upstream emits absolute root-path assets that path-prefix proxying
can't route (haze-class apps). The WebUI hides the "Open" button
when this is set and surfaces the text as a tooltip explaining
why only the direct host-port link is offered. |
| `status` | string | yes | Current status: "running", "stopped", "restarting", "created", "exited". |
| `unsafe_mode` | boolean | no | True if the app was deployed with allow_unsafe — i.e. it has elevated
privileges (caps, host devices, host namespaces, or bind mounts
outside the standard sandbox). Surfaced as a badge in the WebUI. |


### `apps.remove`

Remove a named simple app (stop + delete the container, clean up volume dirs, remove the Caddy ingress) and asynchronously reapply TLS so Caddy stops renewing any orphaned subdomain cert.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | App name. |


### `apps.start`

Start a previously stopped named app (simple container or compose project).

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | App name. |


### `apps.stop`

Stop a named app (simple container or compose project) without removing it.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | App name. |


### `apps.restart`

Restart a named app (simple container or compose project).

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | App name. |


### `apps.pull`

Pull the latest image(s) for a named app and recreate the container(s) — for simple apps it stops/removes/reinstalls preserving config and subdomain mode; for compose apps it runs `docker compose pull` then `up -d`.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | App name. |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `containers` | `AppContainer`[] | no | Individual containers (for compose apps with multiple services). |
| `created` | string | yes | ISO 8601 timestamp of when the container was created. |
| `image` | string | yes | Container image (primary image for compose apps). |
| `kind` | string | yes | App kind: "simple" or "compose". |
| `name` | string | yes | App name (container name for simple, project name for compose). |
| `ports` | `MappedPort`[] | no | Host ports mapped by this app. |
| `proxy_disabled_reason` | string | no | Human-readable reason the reverse-proxy ingress was disabled for
this app — set when the post-install probe detects that the
upstream emits absolute root-path assets that path-prefix proxying
can't route (haze-class apps). The WebUI hides the "Open" button
when this is set and surfaces the text as a tooltip explaining
why only the direct host-port link is offered. |
| `status` | string | yes | Current status: "running", "stopped", "restarting", "created", "exited". |
| `unsafe_mode` | boolean | no | True if the app was deployed with allow_unsafe — i.e. it has elevated
privileges (caps, host devices, host namespaces, or bind mounts
outside the standard sandbox). Surfaced as a badge in the WebUI. |


### `apps.prune`

Prune unused Docker images and volumes, returning the number of images removed and the bytes reclaimed.

**Role:** `operator`

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `images_removed` | integer | yes |  |
| `space_reclaimed_bytes` | integer | yes |  |


### `apps.exec_command`

Return the `docker exec -it <container> <shell>` command string for opening an interactive shell into a named app, probing /bin/bash, /bin/sh, /bin/ash to find an available shell.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | App name. |

**Returns:**

`object`


### `apps.fix_volume_perms`

Chown a host bind-mount source path to the given uid/gid (optionally recursively), enforcing the same forbidden-bind validation as compose deploys.

**Role:** `admin`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `gid` | integer | yes |  |
| `host_path` | string | yes | Host bind-mount source to chown. Validated against the same
forbidden-bind rules as compose deploys (no `..`, no `/`, no
engine state). |
| `recursive` | boolean | no | When true, recurse into the directory tree. Off by default
because recursive chown on a path like `/fs/tank/media` rewrites
ownership on every existing file under it — almost never what
the user wants if the path was pre-populated. |
| `uid` | integer | yes |  |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `ok` | boolean | yes |  |


### `apps.compose.get`

Return the raw docker-compose.yml file contents for a named compose-based app.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Compose app name. |

**Returns:**

`object`


### `apps.compose.logs`

Return aggregated `docker compose logs` output for a named compose app, defaulting to the last 100 lines unless `tail` overrides it.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes |  |
| `tail` | integer | no |  |

**Returns:**

`object`


### `apps.compose.install`

Deploy a new compose-based app by writing its docker-compose.yml to disk, pre-creating bind-mount dirs with the right ownership, running `docker compose up -d`, and auto-creating an ingress for the first exposed TCP port.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `compose_file` | string | yes | Contents of docker-compose.yml. |
| `name` | string | yes | App name (used as compose project name). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `containers` | `AppContainer`[] | no | Individual containers (for compose apps with multiple services). |
| `created` | string | yes | ISO 8601 timestamp of when the container was created. |
| `image` | string | yes | Container image (primary image for compose apps). |
| `kind` | string | yes | App kind: "simple" or "compose". |
| `name` | string | yes | App name (container name for simple, project name for compose). |
| `ports` | `MappedPort`[] | no | Host ports mapped by this app. |
| `proxy_disabled_reason` | string | no | Human-readable reason the reverse-proxy ingress was disabled for
this app — set when the post-install probe detects that the
upstream emits absolute root-path assets that path-prefix proxying
can't route (haze-class apps). The WebUI hides the "Open" button
when this is set and surfaces the text as a tooltip explaining
why only the direct host-port link is offered. |
| `status` | string | yes | Current status: "running", "stopped", "restarting", "created", "exited". |
| `unsafe_mode` | boolean | no | True if the app was deployed with allow_unsafe — i.e. it has elevated
privileges (caps, host devices, host namespaces, or bind mounts
outside the standard sandbox). Surfaced as a badge in the WebUI. |


### `apps.compose.update`

Overwrite a compose app's docker-compose.yml, pre-create any newly added bind-mount sources, and run `docker compose up -d --no-build --pull missing --remove-orphans` to apply the new config.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `compose_file` | string | yes | Contents of docker-compose.yml. |
| `name` | string | yes | App name (used as compose project name). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `containers` | `AppContainer`[] | no | Individual containers (for compose apps with multiple services). |
| `created` | string | yes | ISO 8601 timestamp of when the container was created. |
| `image` | string | yes | Container image (primary image for compose apps). |
| `kind` | string | yes | App kind: "simple" or "compose". |
| `name` | string | yes | App name (container name for simple, project name for compose). |
| `ports` | `MappedPort`[] | no | Host ports mapped by this app. |
| `proxy_disabled_reason` | string | no | Human-readable reason the reverse-proxy ingress was disabled for
this app — set when the post-install probe detects that the
upstream emits absolute root-path assets that path-prefix proxying
can't route (haze-class apps). The WebUI hides the "Open" button
when this is set and surfaces the text as a tooltip explaining
why only the direct host-port link is offered. |
| `status` | string | yes | Current status: "running", "stopped", "restarting", "created", "exited". |
| `unsafe_mode` | boolean | no | True if the app was deployed with allow_unsafe — i.e. it has elevated
privileges (caps, host devices, host namespaces, or bind mounts
outside the standard sandbox). Surfaced as a badge in the WebUI. |


### `apps.compose.remove`

Tear down a compose app via `docker compose down -v --remove-orphans`, delete its project directory, and remove its Caddy ingress.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Compose app name. |


### `apps.ingress.list`

List all per-app reverse-proxy ingresses currently registered in Caddy (name, host_port, path, optional subdomain).

**Role:** `any`

**Returns:**

``AppIngress`[]`


### `apps.ingress.set`

Set (or replace) an app's Caddy reverse-proxy ingress, gated on a subdomain-conflict check, persisting subdomain mode and asynchronously reapplying TLS automation so a new subdomain gets a cert immediately.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `host_port` | integer | yes | Host port to proxy to. |
| `name` | string | yes | App name. |
| `subdomain` | string | no | Opt into subdomain mode by providing a fully-qualified hostname
(e.g. `jellyfin.example.com`). When set, the engine emits a
host-matching Caddy route instead of the default
`/apps/<name>/` path-prefix route, and persists the choice in
the app manifest so engine restarts preserve it. Set to `null`
or omit to use path-prefix mode (the historical default). |

**Returns:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `host_port` | integer | yes | Host port to proxy to. |
| `name` | string | yes | App name. |
| `path` | string | yes | URL path prefix (e.g. "/apps/plex/"). Always set; in subdomain
mode it's purely informational (the WebUI prefers the
`subdomain`-derived URL for the Open button) since the app
answers at root under the configured hostname. |
| `subdomain` | string | no | Fully-qualified hostname the app is served under when subdomain
mode is on (e.g. `jellyfin.example.com`). When set, Caddy
matches the route by `host` rather than path-prefix, and the
app sees itself rooted at `/` — sidestepping the absolute-asset
failure mode that #219's probe disables path-prefix ingress for. |


### `apps.ingress.remove`

Remove an app's Caddy ingress route, clear the persisted subdomain choice, and asynchronously reapply TLS automation so Caddy stops trying to renew the orphaned cert.

**Role:** `operator`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | App name. |


### `apps.ingress.check_conflict`

Best-effort read-only lookup that returns a human-readable "in use by X" reason if the proposed subdomain conflicts with another app or the WebUI hostname, or an empty string when the choice is clear.

**Role:** `any`

**Params:**

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | App name (for self-exclusion). |
| `subdomain` | string | yes | Proposed subdomain. |

**Returns:**

`object`


---

## Object Definitions

### `Acl`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `initiator_iqn` | string | yes | Initiator IQN allowed to connect |
| `password` | string | no | CHAP password for this initiator (optional).
Stored on disk for configfs restore but redacted in API responses. |
| `userid` | string | no | CHAP username for this initiator (optional). |

### `AclEntry`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `initiator_iqn` | string | yes | Initiator IQN to allow. |
| `password` | string | no | Optional CHAP password. |
| `userid` | string | no | Optional CHAP username. |

### `ActiveAlert`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `current_value` | number | yes | Current metric value at the time the alert was evaluated. |
| `message` | string | yes | Human-readable description of the alert condition. |
| `metric` | `AlertMetric` | yes | Metric that triggered the alert. |
| `rule_id` | string | yes | ID of the rule that triggered this alert. |
| `rule_name` | string | yes | Name of the rule that triggered this alert. |
| `severity` | `AlertSeverity` | yes | Severity level of the alert. |
| `source` | string | yes | Identifier of the specific resource that triggered the alert (e.g. filesystem name, device path). |
| `threshold` | number | yes | Threshold value configured in the rule. |

### `AlertCondition`

Enum: `above`, `below`, `equals`

### `AlertMetric`

Enum: `fs_usage_percent`

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
| `allowed_ips` | string[] | yes |  |
| `created_at` | integer | yes |  |
| `expires_at` | integer | no |  |
| `filesystem` | string | no |  |
| `id` | string | yes |  |
| `name` | string | yes |  |
| `role` | `Role` | yes |  |

### `App`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `containers` | `AppContainer`[] | no | Individual containers (for compose apps with multiple services). |
| `created` | string | yes | ISO 8601 timestamp of when the container was created. |
| `image` | string | yes | Container image (primary image for compose apps). |
| `kind` | string | yes | App kind: "simple" or "compose". |
| `name` | string | yes | App name (container name for simple, project name for compose). |
| `ports` | `MappedPort`[] | no | Host ports mapped by this app. |
| `proxy_disabled_reason` | string | no | Human-readable reason the reverse-proxy ingress was disabled for
this app — set when the post-install probe detects that the
upstream emits absolute root-path assets that path-prefix proxying
can't route (haze-class apps). The WebUI hides the "Open" button
when this is set and surfaces the text as a tooltip explaining
why only the direct host-port link is offered. |
| `status` | string | yes | Current status: "running", "stopped", "restarting", "created", "exited". |
| `unsafe_mode` | boolean | no | True if the app was deployed with allow_unsafe — i.e. it has elevated
privileges (caps, host devices, host namespaces, or bind mounts
outside the standard sandbox). Surfaced as a badge in the WebUI. |

### `AppContainer`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `container_id` | string | yes | Docker container ID (short). |
| `image` | string | yes | Container image. |
| `name` | string | yes | Service name (compose service or container name). |
| `status` | string | yes | Container status. |

### `AppEnv`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `is_image_default` | boolean | no | Set by `apps.config` when this entry's value matches the image's
own `Config.Env` default for the same key — i.e. the user didn't
set it explicitly, it just came along with the image. The WebUI
greys these rows out in Edit and shows an "Override" button so
the user sees what the image provides without being misled into
thinking they own it. Always `false` when the WebUI submits env
back to the engine (install/update) — the engine doesn't read
this field for create_container; it's purely an Edit-side hint. |
| `name` | string | yes |  |
| `value` | string | yes |  |

### `AppIngress`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `host_port` | integer | yes | Host port to proxy to. |
| `name` | string | yes | App name. |
| `path` | string | yes | URL path prefix (e.g. "/apps/plex/"). Always set; in subdomain
mode it's purely informational (the WebUI prefers the
`subdomain`-derived URL for the Open button) since the app
answers at root under the configured hostname. |
| `subdomain` | string | no | Fully-qualified hostname the app is served under when subdomain
mode is on (e.g. `jellyfin.example.com`). When set, Caddy
matches the route by `host` rather than path-prefix, and the
app sees itself rooted at `/` — sidestepping the absolute-asset
failure mode that #219's probe disables path-prefix ingress for. |

### `AppPort`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `container_port` | integer | yes | Container port number. |
| `host_port` | integer | no | Host port to map to (optional, auto-assigned if omitted). |
| `name` | string | yes | Port name (e.g. "http", "webui"). |
| `protocol` | string | no | Protocol: "TCP" or "UDP" (default: TCP). |

### `AppStats`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `block_read_bytes` | integer | yes | Cumulative block-device bytes read (cgroup v2 io_service_bytes). |
| `block_write_bytes` | integer | yes | Cumulative block-device bytes written. |
| `cpu_percent` | number | yes | CPU percentage averaged over the Docker stats sample window
(Docker stats CLI semantics — capped only by num-CPUs * 100). |
| `memory_bytes` | integer | yes | Memory in use, with page cache / inactive-file subtracted to
match `docker stats`. Sum across compose containers. |
| `memory_limit_bytes` | integer | yes | Memory limit reported by cgroup. Equals total host memory when
no explicit limit is set; the WebUI decides what to render when
the value matches host memory. |
| `name` | string | yes | App name. Matches the `name` of an entry in `apps.list`. |
| `net_rx_bytes` | integer | yes | Total bytes received across all container interfaces. |
| `net_tx_bytes` | integer | yes | Total bytes transmitted across all container interfaces. |

### `AppVolume`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `host_path` | string | no | Host path (auto-generated under apps storage if empty). |
| `mount_path` | string | yes | Mount path inside the container. |
| `name` | string | yes | Volume name (e.g. "config", "data"). |

### `BackupProfile`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `enabled` | boolean | yes |  |
| `id` | string | yes |  |
| `last_run` | `BackupRunResult` \| null | no |  |
| `name` | string | yes |  |
| `password` | string | yes |  |
| `repo_initialized` | boolean | no |  |
| `retention` | `RetentionPolicy` | no |  |
| `schedule` | string | no |  |
| `snapshot_before` | boolean | no |  |
| `sources` | string[] | yes |  |
| `target` | `BackupTarget` | yes |  |

### `BackupRunResult`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bytes_added` | integer | no |  |
| `duration_secs` | integer | yes |  |
| `files_changed` | integer | no |  |
| `files_new` | integer | no |  |
| `message` | string | yes |  |
| `success` | boolean | yes |  |
| `timestamp` | string | yes |  |

### `BackupSnapshot`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `hostname` | string | yes |  |
| `id` | string | yes |  |
| `paths` | string[] | yes |  |
| `tags` | string[] | yes |  |
| `time` | string | yes |  |

### `BackupTarget`

*(see schema)*

### `BlockDevice`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `dev_type` | string | yes | lsblk device type: `disk` or `part`. |
| `device_class` | string | yes | Device speed class: "nvme", "ssd", or "hdd". |
| `fs_type` | string | no | Filesystem type detected on the device (e.g. `bcachefs`, `ext4`). |
| `in_use` | boolean | yes | Whether the device is currently in use (mounted, in a filesystem, or has partitions in use). |
| `model` | string | no | Drive model from lsblk (e.g. "Samsung SSD 970 EVO Plus 1TB"). None
for partitions and for virtual disks that don't expose a model. |
| `mount_point` | string | no | Current mount point, if mounted. |
| `path` | string | yes | Absolute path of the block device (e.g. `/dev/sda`). |
| `rotational` | boolean | yes | Whether the underlying disk spins (false for NVMe/SSD, true for HDD). |
| `serial` | string | no | Drive serial from lsblk. None for partitions and virtual disks. |
| `size_bytes` | integer | yes | Total capacity in bytes. |
| `transport` | string | no | Transport bus from lsblk (e.g. "sata", "nvme", "usb"). |
| `vendor` | string | no | Drive vendor from lsblk (e.g. "ATA", "NVMe"). |

### `BondConfig`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `inherit_member_mac` | boolean | no | When true, the bond's MAC address is taken from the primary
member's live MAC instead of letting NM/the kernel generate
a random one. Keeps DHCP servers handing out the same lease
across the enslave step — important when one of the members
is the management interface, since otherwise the user's
session lands on a new IP.

Defaults to `true` because the surprise-IP-change on the
random-MAC default is the much louder failure mode. Users
who want a different identity for the bond can opt out via
the "Don't inherit member MAC" checkbox in the WebUI. |
| `ipv4` | `IpConfig` | no |  |
| `ipv6` | `IpConfig` | no |  |
| `members` | string[] | yes |  |
| `mode` | `BondMode` | yes |  |
| `mtu` | integer | no |  |
| `name` | string | yes |  |

### `BondMode`

Enum: `lacp`, `active_backup`, `balance_rr`, `balance_xor`

### `BridgeConfig`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `forward_delay_s` | integer | no | Bridge forward delay in seconds. `None` leaves the kernel default
(15s with STP on, irrelevant with STP off). Set to 0 to skip the
15-second blackhole when STP is off but forward-delay still applies. |
| `inherit_member_mac` | boolean | no | When true, the bridge's MAC address is taken from the primary
member's live MAC instead of letting NM/the kernel generate
a random one. See `BondConfig::inherit_member_mac` for the
rationale; the rule is identical (DHCP-stable identity for
the master across the enslave step). |
| `ipv4` | `IpConfig` | no |  |
| `ipv6` | `IpConfig` | no |  |
| `members` | string[] | no |  |
| `mtu` | integer | no |  |
| `name` | string | yes |  |
| `stp` | boolean | no | Enable Spanning Tree Protocol on the bridge. |

### `CaddyRouteSummary`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `app_name` | string | no | App name when `source` is `engine-app`; `None` otherwise.
Lets the WebUI link the row back to the Apps page. |
| `cert` | `HostCert` \| null | no | On-disk certificate Caddy currently serves for this route's host.
Populated by the engine binary after `list_all_route_summaries`
returns — nasty-apps doesn't have access to the cert directory
or PEM parser. `None` for non-host routes (`path` / `catch_all`)
and for host routes whose cert isn't on disk yet — the engine
pushes automation policies eagerly via `set_tls_automation` so
issuance starts at policy-push time, not on first request, but
DNS-01 + propagation_delay can take 30-90s before the cert
lands. Use `system.tls.host_statuses` for the live state of
each managed host (issuing / failed / active) with error
details when issuance is stuck. |
| `handler_kind` | string | yes | Dominant handler kind, in display order: `reverse_proxy`,
`file_server`, `static_response`, `rewrite`, or `other`. The
WebUI uses this to render a meaningful "handler" column for
rows whose upstream is None. |
| `match_kind` | string | yes | "host", "path", or "catch_all". The WebUI groups by this so the
operator sees host-match (subdomain) routes separately from
path-prefix routes. |
| `match_value` | string | yes | Human-readable match value:
- host_match: the hostname (`jellyfin.example.com`)
- path_match: the first path glob (`/apps/haze/*`)
- catch_all: `(any)` so the WebUI doesn't have to special-case
  the empty string |
| `server` | string | yes | Caddy server name (`srv0`, `srv1`, ...) so the WebUI can group
by listener — the HTTP-to-HTTPS redirect lives on a different
server and shouldn't be lumped in with the HTTPS routes. |
| `source` | string | yes | `engine-app` when the route's `@id` carries the `nasty-app-`
prefix (owned by AppsService::ingress_set); `static` for
anything else (the Caddyfile-baked WebUI / API / WS routes). |
| `upstream` | string | no | Reverse-proxy upstream (e.g. `127.0.0.1:4420`) when the route
ends in a `reverse_proxy` handler. `None` for `file_server`,
`static_response`, etc. — `handler_kind` carries that detail. |

### `ChannelConfig`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `enabled` | boolean | yes |  |
| `id` | string | yes |  |
| `name` | string | yes |  |

### `CpuStats`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `count` | integer | yes | Number of logical CPU cores. |
| `freq_mhz` | integer | no | Average current CPU frequency across all cores in MHz. |
| `governor` | string | no | CPU frequency scaling governor (e.g. "powersave", "performance"). |
| `load_1` | number | yes | 1-minute load average. |
| `load_15` | number | yes | 15-minute load average. |
| `load_5` | number | yes | 5-minute load average. |
| `temp_c` | integer | no | CPU package temperature in degrees Celsius (from hwmon). |

### `CpuSummary`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `logical_cores` | integer | yes |  |
| `max_mhz` | integer | no | Max advertised speed in MHz from `cpu MHz` (often 0 on idle
systems; better signal than `lscpu --max`). |
| `model` | string | no |  |
| `physical_cores` | integer | yes |  |
| `vendor` | string | no |  |

### `DeviceId`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device` | string | yes | 4-hex-digit device ID, e.g. `2204` (RTX 3080). Lowercase. |
| `vendor` | string | yes | 4-hex-digit vendor ID, e.g. `10de` (NVIDIA). Lowercase. |

### `DeviceMissing`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `parent_exists` | boolean | yes | True when the path's parent directory exists on the host. |
| `path` | string | yes | The device path the caller asked about, echoed back. |

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

### `DimmInfo`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `locator` | string | yes | Slot identifier from DMI Type 17 `Locator`, e.g. `DIMM_A1`. |
| `manufacturer` | string | no |  |
| `mem_type` | string | no | `DDR4`, `DDR5`, `LPDDR4`, etc. Empty/None when slot is empty. |
| `part_number` | string | no |  |
| `size_bytes` | integer | yes | Bytes; 0 means slot is empty. |
| `speed_mts` | integer | no | MT/s rated speed. |

### `DiskHealth`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `ata_port` | string | no | ATA/SATA port identifier (e.g. `ata5`), if available. |
| `attributes` | `SmartAttribute`[] | yes | ATA SMART attribute table (empty for NVMe and SAS drives). |
| `capacity_bytes` | integer | yes | Total drive capacity in bytes. |
| `controller_name` | string | no | Human-readable controller name (e.g. `ASMedia ASM1166`). |
| `controller_pci` | string | no | PCI address of the SATA/NVMe controller (e.g. `03:00.0`). |
| `device` | string | yes | Block device path (e.g. `/dev/sda`). |
| `firmware` | string | yes | Drive firmware version string. |
| `health_passed` | boolean | yes | Whether the SMART overall-health self-assessment test passed. |
| `model` | string | yes | Drive model name reported by SMART. |
| `nvme` | `NvmeHealth` \| null | no | NVMe SMART health information log (`Some` only on NVMe drives). |
| `power_on_hours` | integer | no | Accumulated powered-on time in hours. |
| `serial` | string | yes | Drive serial number. |
| `smart_status` | string | yes | Human-readable SMART health status (`PASSED` or `FAILED`). |
| `temperature_c` | integer | no | Current drive temperature in degrees Celsius. |
| `transport` | string | no | smartctl transport flag used to reach this drive, e.g.
`megaraid,0`, `sat+megaraid,2`, `areca,3`. `None` for drives
reachable via smartctl's default transport (direct-attach
SATA/NVMe). The pair `(device, transport)` uniquely identifies a
physical drive — multiple drives behind a RAID controller share
the same block device path but have distinct transport flags. |

### `DiskIoStats`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `io_in_progress` | integer | yes | Number of I/O operations currently in progress. |
| `name` | string | yes | Kernel device name (e.g. `sda`, `nvme0n1`). |
| `read_bytes` | integer | yes | Cumulative bytes read since boot (from `/proc/diskstats`). |
| `read_ios` | integer | yes | Cumulative read I/O operations completed since boot. |
| `write_bytes` | integer | yes | Cumulative bytes written since boot. |
| `write_ios` | integer | yes | Cumulative write I/O operations completed since boot. |

### `DmiBios`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `release_date` | string | no |  |
| `vendor` | string | no |  |
| `version` | string | no |  |

### `DmiSystem`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `manufacturer` | string | no |  |
| `product` | string | no |  |
| `version` | string | no |  |

### `EnrollmentPhase`

*(see schema)*

### `Filesystem`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | Bytes available for writing. |
| `devices` | `FilesystemDevice`[] | yes | Member devices of the filesystem. |
| `mount_point` | string | no | Absolute path where the filesystem is mounted (e.g. `/fs/tank`). |
| `mounted` | boolean | yes | Whether the filesystem is currently mounted. |
| `name` | string | yes | Human-readable filesystem name, derived from the mount point directory. |
| `options` | `FilesystemOptions` | yes | Filesystem-level options read from sysfs or show-super. |
| `total_bytes` | integer | yes | Total usable capacity in bytes. |
| `used_bytes` | integer | yes | Bytes currently in use. |
| `uuid` | string | yes | bcachefs filesystem UUID. |

### `FilesystemDevice`

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

### `FilesystemOptions`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `background_compression` | string | no | Background recompression algorithm applied by the background worker. |
| `background_target` | string | no | Target label for background migration writes. |
| `compression` | string | no | Foreground (inline) compression algorithm (e.g. `lz4`, `zstd`, `none`). |
| `data_checksum` | string | no | Checksum algorithm for data (e.g. `crc32c`, `xxhash`). |
| `data_replicas` | integer | no | Number of replicas for data extents. |
| `degraded` | boolean | no | Whether mounted in degraded mode (missing devices). |
| `encrypted` | boolean | no | Whether the filesystem is encrypted at rest. |
| `erasure_code` | boolean | no | Whether erasure coding (EC) is enabled on the filesystem. |
| `error_action` | string | no | Action on unrecoverable read errors (`continue`, `ro`, `panic`). |
| `foreground_target` | string | no | Target label for foreground (new) writes. |
| `fsck` | boolean | no | Whether fsck runs at mount time. |
| `io_scheduler` | string | no | I/O scheduler for member block devices (e.g. `none`, `mq-deadline`, `kyber`).
`none` is recommended for SSDs; `mq-deadline` is the kernel default. |
| `journal_flush_delay` | integer | no | Journal flush delay in microseconds. Higher values batch more journal writes,
improving throughput under sync-heavy workloads (e.g. NFS commits). |
| `journal_flush_disabled` | boolean | no | Whether journal flushing is disabled. |
| `key_stored` | boolean | no | Whether a stored key exists for auto-unlock on boot. |
| `locked` | boolean | no | Whether the encrypted filesystem is currently locked (needs unlock before mount). |
| `metadata_checksum` | string | no | Checksum algorithm for metadata. |
| `metadata_replicas` | integer | no | Number of replicas for metadata (btree) extents. |
| `metadata_target` | string | no | Target label for metadata placement. |
| `move_bytes_in_flight` | string | no | Maximum bytes in flight for background mover (e.g. `"8.0M"`). |
| `move_ios_in_flight` | integer | no | Maximum concurrent background mover IOs. |
| `promote_target` | string | no | Target label for data promotion (cache tier). |
| `verbose` | boolean | no | Whether verbose mount logging is enabled. |
| `version_upgrade` | string | no | Version upgrade behavior at mount: `compatible`, `incompatible`, or `none`. |

### `FirewallRule`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `active` | boolean | yes |  |
| `ports` | `PortSpec`[] | yes |  |
| `service` | string | yes | Protocol/service name (e.g. "nfs", "ssh", "webui"). |

### `FirmwareDevice`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device_id` | string | yes | Device ID (fwupd GUID). |
| `name` | string | yes | Device name (e.g. "UEFI dbx", "WD Black SN850X"). |
| `update_available` | boolean | yes | Whether an update is available. |
| `update_description` | string | no | Update description/summary. |
| `update_version` | string | no | Available update version (if any). |
| `vendor` | string | yes | Vendor name. |
| `version` | string | yes | Currently installed firmware version. |

### `FsDependents`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `apps` | string[] | yes |  |
| `backup_jobs` | string[] | yes |  |
| `filesystem` | string | yes |  |
| `iscsi_targets` | string[] | yes |  |
| `mounted` | boolean | yes |  |
| `nfs_shares` | string[] | yes |  |
| `nvmeof_subsystems` | string[] | yes |  |
| `smb_shares` | string[] | yes |  |
| `subvolumes` | string[] | yes |  |
| `vms` | string[] | yes |  |

### `Generation`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `booted` | boolean | yes | Whether this is the generation the system booted into. |
| `current` | boolean | yes | Whether this is the currently activated generation. |
| `date` | string | yes | Build date (e.g. "2026-03-21 11:15:37"). |
| `generation` | integer | yes | NixOS generation number. |
| `kernel_version` | string | yes | Kernel version string. |
| `label` | string | no | User-assigned label (e.g. "known good", "stable"). |
| `nasty_version` | string | no | NASty version baked into this generation (from /etc/nasty-version). |
| `nixos_version` | string | yes | NixOS version string (e.g. "26.05.20260318.b40629e"). |

### `HostCert`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `expires` | string | no |  |
| `expires_in_days` | integer | no | Days until expiry from now; negative = expired. Lets the WebUI
colour the badge (red ≤ 7, amber ≤ 30, green otherwise) without
parsing the RFC-2822 expires string client-side. |
| `issued` | string | no |  |
| `issuer` | string | no |  |
| `path` | string | yes | On-disk path; surfaced as a tooltip in the WebUI for debugging
"which cert is this actually serving" questions. |

### `HostTlsStatus`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `expires` | string | no |  |
| `expires_in_days` | integer | no |  |
| `host` | string | yes |  |
| `issued` | string | no |  |
| `issuer` | string | no |  |
| `message` | string | no | `active` ⇒ on-disk cert path. `failed` / `issuing` ⇒ last log
line, verbatim. `pending` ⇒ None. |
| `state` | string | yes |  |

### `InterfaceConfig`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `enabled` | boolean | no |  |
| `ipv4` | `IpConfig` | no |  |
| `ipv6` | `IpConfig` | no |  |
| `mtu` | integer | no |  |
| `name` | string | yes |  |

### `IommuGroup`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `devices` | `PciDevice`[] | yes |  |
| `id` | integer | yes |  |

### `IpConfig`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `addresses` | string[] | no | Addresses in CIDR notation, e.g. "192.168.1.100/24" or "fd00::1/64". |
| `gateway` | string | no |  |
| `method` | `IpMethod` | yes |  |

### `IpMethod`

Enum: `dhcp`

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

### `MappedPort`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `container_port` | integer | yes | Container port. |
| `host_port` | integer | yes | Host port. |
| `protocol` | string | yes | Protocol (tcp/udp). |

### `MemoryStats`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_bytes` | integer | yes | RAM available for allocation without swapping. |
| `swap_total_bytes` | integer | yes | Total swap space in bytes. |
| `swap_used_bytes` | integer | yes | Swap space currently in use. |
| `total_bytes` | integer | yes | Total installed RAM in bytes. |
| `used_bytes` | integer | yes | RAM currently in use (total minus available). |

### `MemorySummary`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `dimms` | `DimmInfo`[] | yes |  |
| `ecc` | boolean | yes | Whether the memory array supports ECC (single bit, multi-bit, or chipkill). |
| `slots_total` | integer | yes | Total DIMM slots on the system (populated + empty). |
| `slots_used` | integer | yes | Slots with a DIMM in them. |
| `total_bytes` | integer | yes | Sum of all populated DIMM sizes in bytes. |

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

### `NetworkPendingTxn`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `revert_at_unix` | integer | yes |  |
| `risk_reason` | string | yes |  |
| `txn_id` | string | yes |  |

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
| `path` | string | yes | Absolute filesystem path being exported (must be under `/fs/`). |

### `NmDiffSummary`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `add` | integer | yes |  |
| `delete` | integer | yes |  |
| `unchanged` | integer | yes |  |
| `update` | integer | yes |  |

### `NmDiffUpdate`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `changed_sections` | string[] | yes | One line per differing top-level section (e.g. `"ipv4"`,
`"bridge"`).  Cheap signal for the UI; the WebUI can render a
richer diff if it wants by re-fetching settings. |
| `id` | string | yes |  |

### `NutMode`

Enum: `local`, `remote`

### `NvmeHealth`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `available_spare_percent` | integer | yes | Remaining spare blocks as a percentage of the initial reserve.
Decreases as the drive remaps failed NAND cells. |
| `available_spare_threshold_percent` | integer | yes | Vendor-set threshold (typically 10%, sometimes higher) below which
`available_spare_percent` triggers the spare-low critical warning. |
| `controller_busy_minutes` | integer | yes | Controller busy time in minutes. |
| `critical_comp_minutes` | integer | yes | Cumulative minutes the controller spent above the critical
temperature threshold. |
| `critical_warning` | integer | yes | Critical-warning bit field. `0` is healthy; non-zero bits flag
spare-below-threshold (0x1), temperature (0x2), reliability (0x4),
read-only (0x8), volatile-backup-failed (0x10), persistent-memory-
region-RO (0x20). |
| `data_units_read` | integer | yes | Read volume reported in NVMe "data units" (1 unit = 1000 × 512-byte
LBAs = 512,000 bytes per spec). UI multiplies for human-readable
totals. |
| `data_units_written` | integer | yes | Write volume in NVMe data units (see `data_units_read`). |
| `host_reads` | integer | yes | Total host read commands issued to the controller. |
| `host_writes` | integer | yes | Total host write commands issued to the controller. |
| `media_errors` | integer | yes | Media and data integrity errors detected by the controller. |
| `most_recent_error` | string | no | Human-readable status string of the most recent entry in the
error information log table, when smartctl returned one. The
table itself is only emitted by smartctl 7.4+; older versions
give just the count above with no way to see what the errors
actually were. `None` when the log is empty or unavailable. |
| `num_err_log_entries` | integer | yes | Number of entries in the controller error information log. |
| `percentage_used` | integer | yes | Endurance estimate: 0 = new, 100 = nominal end of life. May exceed
100 on drives operated past their rated DWPD. Not a hard limit. |
| `power_cycles` | integer | yes | Number of power cycles. |
| `temperature_sensors_c` | integer[] | yes | Per-zone temperatures in degrees Celsius. Some drives only wire up
a subset of sensors and report `null` for the rest (e.g. Kingston
SNV3S reports `[null, 43]`). |
| `unsafe_shutdowns` | integer | yes | Number of unclean shutdowns (drive lost power without a graceful
shutdown notify). |
| `warning_temp_minutes` | integer | yes | Cumulative minutes the controller spent above the warning
temperature threshold. |

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

### `OidcRoleMapping`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `group` | string | yes | Group name (matched verbatim against entries in the configured groups claim). |
| `role` | string | yes | NASty role to assign: "admin", "operator", or "readonly". |

### `OidcSettings`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `auto_provision` | boolean | no | When true, unknown OIDC subjects are auto-provisioned as local users on first login. |
| `client_id` | string | no | OAuth client_id registered with the IdP. |
| `client_secret` | string | no | OAuth client_secret. Returned as a placeholder over RPC; only the engine sees the real value. |
| `default_role` | string | no | Role applied when no group mapping matches. None = deny login. |
| `enabled` | boolean | no | Master switch — when false, OIDC endpoints return 404 and no IdP traffic occurs. |
| `groups_claim` | string | no | Name of the ID-token claim that carries the user's groups. |
| `issuer_url` | string | no | IdP issuer URL (used for OIDC discovery, e.g. "https://auth.example.com"). |
| `redirect_uri` | string | no | Absolute redirect URI registered with the IdP (e.g. "https://nasty.local/api/auth/oidc/callback"). |
| `role_mappings` | `OidcRoleMapping`[] | no | Group → role mappings. Evaluated in order; first match wins. |
| `scopes` | string[] | no | OAuth scopes to request. Defaults to ["openid","profile","email","groups"]. |

### `PassthroughDevice`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `address` | string | yes | PCI address (e.g. "0000:03:00.0"). |
| `label` | string | no | Human-readable label (e.g. "NVIDIA RTX 3080"). |

### `PciDevice`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bdf` | string | yes | Bus:Device.Function in canonical sysfs form, e.g. `0000:01:00.0`. |
| `class_id` | string | yes | 4-hex-digit class code, e.g. `0300` (VGA controller). |
| `class_name` | string | no | Human-readable class name (from pci.ids), if available. |
| `device_id` | string | yes | 4-hex-digit device ID, e.g. `2204` (RTX 3080). |
| `device_name` | string | no | Human-readable device name (from pci.ids), if available. |
| `driver` | string | no | Currently bound kernel driver, e.g. `vfio-pci`, `nvidia`,
`e1000e`. `None` if no driver is bound (rare — usually means
the device is reserved for explicit binding). |
| `vendor_id` | string | yes | 4-hex-digit vendor ID, e.g. `10de` (NVIDIA). |
| `vendor_name` | string | no | Human-readable vendor name (from pci.ids), if available. |

### `PciDevice2`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `address` | string | yes | PCI address (e.g. "0000:03:00.0"). |
| `bound_to_vfio` | boolean | yes | Whether the device is currently bound to vfio-pci. |
| `description` | string | yes | Human-readable description from lspci. |
| `iommu_group` | integer | yes | IOMMU group number. |
| `vendor_device` | string | yes | Vendor:device ID (e.g. "10de:2206"). |

### `Port`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `addr` | string | yes | Listening IP address (e.g. `0.0.0.0` for all interfaces). |
| `addr_family` | string | yes | Address family (`ipv4` or `ipv6`). |
| `port_id` | integer | yes | nvmet configfs port ID (unique across all subsystems on this host). |
| `service_id` | string | yes | TCP/RDMA port number as a string (default NVMe-oF port is `4420`). |
| `transport` | string | yes | Transport type (e.g. `tcp`, `rdma`). |

### `PortConflict`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `port` | integer | yes | The port that has a conflict. |
| `used_by` | string | yes | What is using this port (e.g. "caddy", "app:plex"). |

### `PortSpec`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `iface` | string | no | Optional interface restriction (e.g. "tailscale0"). |
| `port` | integer | yes |  |
| `source` | string | no | Optional source IP/CIDR restriction (e.g. "192.168.1.0/24"). |
| `transport` | `Transport` | yes |  |

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

### `RebuildSnapshot`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `exit_code` | integer | no | Exit code from the last finished run, if any. Useful for
the failed-rebuild error toast. |
| `journal_tail` | string[] | yes | Tail of the unit's journal (last ~20 lines), surfaced
verbatim in the wizard's "rebuild output" expandable.
Empty when the unit was never started, or when journalctl
failed (we log + skip rather than abort the status call). |
| `status` | `RebuildStatus` | yes | `running` / `succeeded` / `failed` / `not_run`. Last
transition is also visible on the wizard's polled UI. |

### `RebuildStatus`

*(see schema)*

### `ReleaseChannel`

*(see schema)*

### `RetentionPolicy`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `keep_daily` | integer | no |  |
| `keep_last` | integer | no |  |
| `keep_monthly` | integer | no |  |
| `keep_weekly` | integer | no |  |
| `keep_yearly` | integer | no |  |

### `Role`

Enum: `admin`

### `SecureBootStatus`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `enabled` | boolean | no | `Some(true)` iff `bootctl status` reports `Secure Boot: enabled`.
`Some(false)` for `disabled` (any parenthetical). `None` when we
couldn't determine state — see `note`. |
| `measured_uki` | boolean | no | `Some(true)` when bootctl reports `Measured UKI: yes` — kernel
and initrd are loaded as a measured Unified Kernel Image.
Useful signal for the future lanzaboote integration where SB
and measured boot together strengthen the PCR-7 seal. |
| `note` | string | no | Free-form one-line reason when we couldn't determine the
state ("bootctl unavailable: …", "bootctl status returned no
System: block", etc.). Surfaced under the Hardware card's
status pill. |
| `setup_mode` | boolean | no | UEFI Setup Mode — when true the firmware accepts arbitrary key
enrollment without PK signing. Sourced from the `(setup)`
parenthetical on the `Secure Boot:` line. |
| `unsupported` | boolean | no | `Some(true)` when bootctl reports `disabled (unsupported)` —
the firmware lacks SB support entirely (e.g. OVMF without the
SB build option, common on default QEMU). Distinct from a
firmware that supports SB but has it switched off, so the
WebUI can show "Unsupported" instead of nudging the operator
to enable a feature they can't. |

### `ServiceStatus`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `cpu_seconds` | number | no | CPU time in seconds (user + system). |
| `memory_bytes` | integer | no | Resident memory usage in bytes. |
| `name` | string | yes | Display name (e.g. "Engine", "Metrics"). |
| `pid` | integer | no | Process ID. |
| `running` | boolean | yes | Whether the service is currently active/running. |
| `uptime_seconds` | integer | no | Process uptime in seconds. |

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

### `SmbGroup`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `gid` | integer | yes | Unix GID. |
| `members` | string[] | yes | Group members (usernames). |
| `name` | string | yes | Linux group name. |

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
| `path` | string | yes | Absolute filesystem path being shared (must be under `/fs/`). |
| `read_only` | boolean | yes | Whether the share is read-only. |
| `valid_users` | string[] | yes | Usernames allowed to connect (empty means no restriction beyond authentication). |

### `SmbUser`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `uid` | integer | yes | Unix UID. |
| `username` | string | yes | Linux username. |

### `Snapshot`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `block_device` | string | no | Loop device path if this snapshot's vol.img is currently attached (block snapshots only). |
| `filesystem` | string | yes | Name of the filesystem that contains this snapshot. |
| `name` | string | yes | Snapshot name (unique within the parent subvolume). |
| `parent` | string | no | Parent subvolume path as tracked by bcachefs (from snapshot_parent). |
| `path` | string | yes | Absolute filesystem path to the snapshot directory. |
| `read_only` | boolean | yes | Whether this snapshot is read-only. |
| `subvolume` | string | yes | Name of the parent subvolume. |

### `SubPathRecipe`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `display_name` | string | yes | Short label shown next to the "Apply" button (e.g. "Grafana
sub-path mode"). Not the env var key — purely human-readable. |
| `env` | `AppEnv`[] | yes | Env vars to add to the install form when the user applies this
recipe. Values may contain `{name}`, `{host}`, `{scheme}` —
see template note above. |

### `Subvolume`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bcachefs_options` | object | no | Effective bcachefs options set on this subvolume (from bcachefs_effective.* xattrs).
Only includes options that differ from the filesystem default. |
| `block_device` | string | no | Loop device path currently attached to the backing image (block subvolumes only). |
| `comments` | string | no | Free-text description or notes for this subvolume. |
| `compression` | string | no | Compression algorithm applied to this subvolume (e.g. `lz4`, `zstd`). |
| `direct_io` | boolean | no | Whether O_DIRECT is enabled on the loop device (block subvolumes only). |
| `filesystem` | string | yes | Name of the filesystem that contains this subvolume. |
| `name` | string | yes | Subvolume name (unique within the filesystem). |
| `owner` | string | no | Token name that created this subvolume; None for subvolumes created by human users. |
| `parent` | string | no | Parent subvolume name if this is a clone (from bcachefs snapshot_parent). |
| `path` | string | yes | Absolute filesystem path to the subvolume directory. |
| `properties` | object | no | Arbitrary key-value metadata stored as POSIX xattrs (user.* namespace).
Used by nasty-csi to track CSI volume metadata without sidecar files. |
| `quota_bytes` | integer | no | Hard quota limit in bytes for filesystem subvolumes. `None`
means no limit set (the subvolume can grow to fill the
filesystem). Always `None` for block subvolumes — their
ceiling is `volsize_bytes`, not a quota. |
| `snapshots` | string[] | yes | Names of snapshots belonging to this subvolume. |
| `subvolume_type` | `SubvolumeType` | yes | Whether this is a filesystem or block-backed subvolume. |
| `used_bytes` | integer | no | Disk usage in bytes. For filesystem subvolumes, comes from the
per-project quota (set on every create, so tracking is always
on); `None` only on legacy subvolumes created before the
always-track change. For block subvolumes, comes from the
backing image's allocated size. |
| `volsize_bytes` | integer | no | Size of the backing sparse image in bytes (block subvolumes only). |

### `SubvolumeDependents`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `apps` | string[] | yes |  |
| `backup_jobs` | string[] | yes |  |
| `filesystem` | string | yes |  |
| `iscsi_targets` | string[] | yes |  |
| `name` | string | yes |  |
| `nfs_shares` | string[] | yes |  |
| `nvmeof_subsystems` | string[] | yes |  |
| `path` | string | yes |  |
| `smb_shares` | string[] | yes |  |
| `vms` | string[] | yes |  |

### `SubvolumeType`

Enum: `filesystem`, `block`

### `TempUnit`

Enum: `celsius`, `fahrenheit`

### `TpmInfo`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `manufacturer` | string | no | 4-character ASCII manufacturer code reported by the chip via
`TPM2_PT_MANUFACTURER` — `IFX` (Infineon), `STM`
(STMicroelectronics), `NTC` (Nuvoton), `IBM ` (swtpm), `AMD`
(fTPM), etc. Queried via `tpm2_getcap` since sysfs doesn't
expose it on most kernel drivers (an empirically-discovered
gap — swtpm and many real TPM drivers publish only
`tpm_version_major`). |
| `rm_available` | boolean | yes | `/dev/tpmrm0` is the resource-manager interface tpm2-tools and
any sealing code actually talks to. Present on TPM 2.0 systems
with the in-kernel resource manager enabled (default on every
modern kernel). When false, the chip exists but the sealing
path will fail at the device-open step. |
| `vendor_string` | string | no | Vendor's marketing model string, also from `tpm2_getcap` —
assembled from `TPM2_PT_VENDOR_STRING_1..4`. E.g. `"SLB9665"`
for an Infineon chip, `"SW   TPM"` for swtpm. Empty / None
when the chip doesn't publish a vendor string or the query
fails. |
| `version_major` | integer | no | `1` for TPM 1.2 (incompatible with the planned sealing flow),
`2` for TPM 2.0. Read from `tpm_version_major`. |

### `Transport`

Enum: `tcp`, `udp`

### `UsbDevice`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bus` | integer | yes | Bus number from `lsusb` (decimal). |
| `description` | string | yes | Single-line "Vendor Name Product Name" rendered by lsusb. The
embedded `pci.ids`/`usb.ids` lookup is lsusb's job, not ours. |
| `device` | integer | yes | Device address on the bus. |
| `product_id` | string | yes | 4-hex-digit product ID. |
| `vendor_id` | string | yes | 4-hex-digit vendor ID. |

### `UsbPassthrough`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `label` | string | no | Human-readable label preserved for the UI (e.g. "Realtek
Bluetooth dongle"). The kernel can't tell us this — it comes
from the original `lsusb` listing the user picked from. |
| `product_id` | string | yes | 4-hex-digit USB product ID. |
| `vendor_id` | string | yes | 4-hex-digit USB vendor ID (e.g. "0bda"). Stored lowercase
without the `0x` prefix to match `lsusb` formatting. |

### `UserInfo`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `role` | `Role` | yes |  |
| `username` | string | yes |  |
| `webauthn_credential_count` | integer | no | How many WebAuthn credentials are registered to this user.
Drives the admin "Reset security keys" affordance on the
/users page — admins only see the button on rows where
there are credentials to reset, instead of a no-op button
on every row. |

### `VersionInputInfo`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Flake input name (e.g. `bcachefs-tools`, `nasty`). |
| `rev` | string | no | Locked commit SHA from `/etc/nixos/flake.lock` (shortened to 12 chars). |
| `tag` | string | no | Human-meaningful ref string from `flake.lock`'s
`nodes[<name>].original.ref` — typically a tag like `v1.38.3`
or a branch name like `main`. When present, prefer this for
display over `rev` (which is just a 12-char SHA prefix).
`None` when the lock node has no `original.ref` set (e.g.
inputs referenced by raw commit hash, or inputs the lock
doesn't carry an `original` block for). |
| `url` | string | yes | Exact `input.url` string from `/etc/nixos/flake.nix`. |

### `VersionSwitchInput`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `name` | string | yes | Flake input name. |
| `update` | boolean | no | Whether this input should be refreshed in `flake.lock`. |
| `url` | string | yes | Replacement URL to write to `/etc/nixos/flake.nix`. |

### `VlanConfig`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `ipv4` | `IpConfig` | no |  |
| `ipv6` | `IpConfig` | no |  |
| `mtu` | integer | no |  |
| `parent` | string | yes |  |
| `vlan_id` | integer | yes |  |

### `VmDisk`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `aio` | string | no | I/O mode: "threads" (default), "native" (requires cache=none). |
| `cache` | string | no | Cache mode: "writeback" (default), "writethrough", "none", "unsafe". |
| `discard` | string | no | Discard/TRIM support: "unmap" or "ignore" (default). |
| `interface` | string | no | Disk interface: "virtio" (default), "scsi", "ide". |
| `iops_rd` | integer | no | I/O throttling: max read IOPS (0 = unlimited). |
| `iops_wr` | integer | no | I/O throttling: max write IOPS (0 = unlimited). |
| `path` | string | yes | Disk path — block device (/dev/loopX) or image file. |
| `readonly` | boolean | no | Whether this is a read-only disk. |

### `VmDiskSubvolume`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `device` | string | yes | Block device path. |
| `filesystem` | string | yes | Filesystem name. |
| `subvolume` | string | yes | Subvolume name. |

### `VmNetwork`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `bridge` | string | no | Bridge name (for bridge mode, e.g. "br0"). |
| `mac` | string | no | MAC address (auto-generated if omitted). |
| `mode` | string | no | Network mode: "bridge" or "user" (NAT). |

### `VmStatus`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `autostart` | boolean | no | Whether the VM should auto-start on NASty boot. |
| `boot_iso` | string | no | Legacy single-ISO field, kept for cross-version state-file
compatibility. On load we migrate this into `cdroms` if
`cdroms` is empty; on save we mirror `cdroms.first()` back
into here so a hypothetical rollback to a pre-`cdroms` engine
still sees the boot ISO. New code reads `cdroms` exclusively. |
| `boot_order` | string | no | Boot order: "disk", "cdrom", or "network". |
| `cdroms` | string[] | no | ISO files to attach as CD-ROM devices. The first entry is the
one QEMU treats as the boot CD when `boot_order = "cdrom"`;
additional entries show up as extra read-only CDs inside the
guest (typical use: Windows 11 install needs the Win11 ISO
alongside the virtio-win driver ISO so the installer can see
the virtio storage controller — issue #285). |
| `cpu_model` | string | no | CPU model: "host" (default), "max", "qemu64", etc. |
| `cpus` | integer | yes | Number of virtual CPU cores. |
| `description` | string | no | Optional description. |
| `disks` | `VmDisk`[] | yes | Boot disk configuration. |
| `extra_args` | string[] | no | Extra raw QEMU arguments for advanced users. |
| `id` | string | yes | Unique VM identifier (UUID). |
| `machine_type` | string | no | Machine type: "q35" (default for x86), "i440fx". |
| `memory_mib` | integer | yes | RAM in MiB. |
| `name` | string | yes | Human-readable name. |
| `networks` | `VmNetwork`[] | yes | Network interfaces. |
| `passthrough_devices` | `PassthroughDevice`[] | yes | PCI devices to pass through via VFIO. |
| `pid` | integer | no | QEMU process PID (if running). |
| `running` | boolean | yes | Whether the VM is currently running. |
| `uefi` | boolean | no | Whether to use UEFI boot (default: true). |
| `usb_devices` | `UsbPassthrough`[] | no | USB devices to pass through. Identified by vendor/product ID
rather than bus/addr because USB enumeration order shuffles
across reboots; pinning to IDs is the stable choice. Caveat:
all devices matching a (vendor, product) pair attach, so
plugging in two identical keyboards passes both through. |
| `vga` | string | no | VGA device type: "virtio" (default), "qxl", "std", "none". |
| `vnc_port` | integer | no | VNC display port (if running, for console access). |

### `VolumeMismatch`

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `current_gid` | integer | no |  |
| `current_uid` | integer | no | Owner UID on the host. None when the path doesn't exist yet. |
| `exists` | boolean | yes | True when the source path exists on the host. False = the
directory will be created by the deploy pipeline; we'll chown
it to expected at create time, so it's informational rather
than an error. |
| `expected_gid` | integer | no |  |
| `expected_uid` | integer | yes |  |
| `filesystem_missing` | boolean | no | True when the path is `/fs/<X>/…` and `<X>` is not a mounted
filesystem. Distinct from `!exists` because pre-create would
`mkdir -p` it on rootfs — a hard error the user must fix in
their compose, not a "we'll create it for you" hint. |
| `host_path` | string | yes |  |
| `line` | integer | no | 1-based line number of the volume entry in the compose file
(for editor underlining). Best-effort: we substring-match the
host path against the source; ambiguous duplicates resolve to
the first occurrence. |
| `mount_path` | string | yes |  |
| `service` | string | yes |  |

