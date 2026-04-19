# Maintenance

## Clean Old Generations

NASty is built on NixOS, which keeps every system configuration as a "generation." Over time these accumulate and consume disk space. To clean up:

**Keep only the last 2 generations and free disk space:**

```bash
nix-env --profile /nix/var/nix/profiles/system \
  --delete-generations +2

nix-collect-garbage
```

The `+2` means "keep the 2 most recent, delete the rest." You can change this number.

**List all generations before cleaning:**

```bash
nixos-rebuild list-generations
```

The currently booted and active generations cannot be deleted.

**Check how much space was reclaimed:**

```bash
df -h /nix/store
```

## Force Update / Rebuild

The normal update flow (WebUI Update page) runs `nix flake update` and then `nixos-rebuild switch`. If it gets stuck (e.g. shows "Update available" but finishes instantly without rebuilding), the flake lock may have been updated without the rebuild completing.

**Force a full update and rebuild:**

```bash
nix flake update nasty --flake /etc/nixos

nixos-rebuild switch --flake /etc/nixos#nasty
```

If `nixos-rebuild` fails with "Unit nixos-rebuild-switch-to-configuration.service was already loaded", reset the stale unit first:

```bash
systemctl reset-failed \
  nixos-rebuild-switch-to-configuration.service

nixos-rebuild switch --flake /etc/nixos#nasty
```

**Update the version file after a manual rebuild** (so the WebUI shows the correct version):

```bash
jq -r '.nodes.nasty.locked.rev[:7]' /etc/nixos/flake.lock \
  > /var/lib/nasty/version
```

**Full cleanup + update in one go:**

```bash
nix-env --profile /nix/var/nix/profiles/system \
  --delete-generations +2 && \
nix-collect-garbage && \
nix flake update nasty --flake /etc/nixos && \
systemctl reset-failed \
  nixos-rebuild-switch-to-configuration.service 2>/dev/null; \
nixos-rebuild switch --flake /etc/nixos#nasty && \
jq -r '.nodes.nasty.locked.rev[:7]' /etc/nixos/flake.lock \
  > /var/lib/nasty/version
```
