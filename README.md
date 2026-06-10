# ansible-netplan

An [Ansible](https://www.ansible.com) role to manage [Netplan](https://netplan.io) network configuration.

This role supports **all Netplan configuration keys** dynamically via YAML merging — you do not need to wait for role updates to use new Netplan features.

## Ansible Galaxy

```bash
ansible-galaxy install mrlesmithjr.netplan
```

## Supported Platforms

| Platform | Versions |
|----------|----------|
| Ubuntu | 18.04, 20.04, 22.04, 24.04 |
| Debian | 11, 12 |

## Requirements

Run with `become: true`. The `netplan.io` package is installed automatically.

## Quick Start

```yaml
---
- hosts: all
  become: true
  roles:
    - role: mrlesmithjr.netplan
      netplan_enabled: true
      netplan_config_file: /etc/netplan/ansible-config.yaml
      netplan_renderer: networkd
      netplan_configuration:
        network:
          version: 2
          ethernets:
            enp3s0:
              dhcp4: true
```

## Key Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `netplan_enabled` | `true` | Master switch — set to `false` to skip all tasks |
| `netplan_config_file` | `/etc/netplan/ansible-config.yaml` | Path to the managed Netplan config file |
| `netplan_config_file_mode` | `0600` | File permissions for the config file |
| `netplan_renderer` | `""` | Renderer: `networkd` or `NetworkManager` (uses system default if unset) |
| `netplan_configuration` | `{}` | Your Netplan configuration (merged with defaults) |
| `netplan_apply` | `true` | Run `netplan apply` after configuration changes |
| `netplan_remove_existing` | `false` | Remove existing Netplan config files before writing |
| `netplan_backup_existing` | `false` | Back up existing config files before overwriting |
| `netplan_enable_recovery` | `true` | Wrap `netplan apply` in a block/rescue that reverts to `50-cloud-init.yaml` if the apply exits non-zero while SSH stays connected (see note below) |
| `netplan_blacklist_e1000e` | `false` | Blacklist the `e1000e` kernel module via `/etc/modprobe.d/blacklist-e1000e.conf` and trigger `update-initramfs` (see note below) |
| `netplan_enable_nic_watchdog` | `false` | Deploy a systemd timer that checks for NIC hang events every 5 minutes and reboots if the default gateway is unreachable (see note below) |

### Apply-Failure Recovery (`netplan_enable_recovery`)

When `true` (the default), the `netplan apply` step runs inside a block/rescue. If the apply command exits non-zero and the control connection is still alive, the role restores `/etc/netplan/50-cloud-init.yaml` from a pre-apply backup and re-applies it, then fails the play with a clear message.

> **Important:** This recovery covers only apply-time errors that are caught while SSH remains connected. If `netplan apply` severs the SSH session (UNREACHABLE), Ansible never enters the rescue block and the host is left in whatever state the apply reached. True out-of-band rollback on SSH drop requires `netplan try` (interactive, not automatable via Ansible) or the NIC watchdog below.

Set `netplan_enable_recovery: false` to restore the pre-backport behaviour with no rescue wrapper.

### Intel e1000e NIC Blacklist (`netplan_blacklist_e1000e`)

Some Intel NICs driven by the `e1000e` driver exhibit hardware unit hang events under load in certain hypervisor or bare-metal environments, causing the interface to become unresponsive until the driver is reset or the host reboots. When `netplan_blacklist_e1000e: true`, the role writes `/etc/modprobe.d/blacklist-e1000e.conf` and calls `update-initramfs` so the driver is excluded from the initrd on the next boot.

Use this when hang events are frequent enough that the watchdog cannot recover fast enough, or when the NIC is not the primary interface and can be disabled entirely. When the blacklist is enabled the NIC watchdog is not deployed (the driver will be absent after reboot).

### NIC Watchdog (`netplan_enable_nic_watchdog`)

When `netplan_enable_nic_watchdog: true`, the role deploys `/usr/local/bin/watch_nic_hang.sh` and a systemd timer (`nic-hang-watch.timer`) that runs it every 5 minutes starting 5 minutes after boot. The script:

1. Checks `journalctl` for `"Detected Hardware Unit Hang"` in the last 10 minutes.
2. If a hang is found, pings the default gateway (3 probes, 2-second timeout).
3. If the gateway is unreachable, calls `/sbin/reboot`.
4. If the gateway responds, logs a warning and takes no action (transient log noise).

The watchdog is a mitigation for environments where hang events are occasional and a reboot is acceptable. It is not deployed when `netplan_blacklist_e1000e: true`, because the driver is absent after that change takes effect on reboot.

To remove a previously deployed watchdog, set `netplan_enable_nic_watchdog: false` and re-run the role. The role stops and disables the timer and removes the script and unit files.

### Configuration Examples

**Static IP:**

```yaml
netplan_configuration:
  network:
    version: 2
    ethernets:
      enp3s0:
        addresses:
          - 192.168.1.100/24
        routes:
          - to: default
            via: 192.168.1.1
        nameservers:
          addresses: [1.1.1.1, 8.8.8.8]
```

**Bond:**

```yaml
netplan_configuration:
  network:
    version: 2
    bonds:
      bond0:
        dhcp4: true
        interfaces: [enp3s0, enp4s0]
        parameters:
          mode: active-backup
          primary: enp3s0
```

**WireGuard (using vaulted variables):**

```yaml
netplan_configuration:
  network:
    version: 2
    tunnels:
      wg0:
        mode: wireguard
        key: "{{ my_wireguard_private_key }}"

my_wireguard_private_key: !vault |
  ...
```

> Vault-encrypted variables must be defined **outside** `netplan_configuration` to be evaluated correctly.

See [defaults/main.yml](defaults/main.yml) for the full variable reference including VLANs, bridges, and WiFi.

## Testing

```bash
pip install molecule molecule-docker
molecule test
```

## Support This Project

If your organization uses this role in production, consider
[sponsoring its maintenance](https://github.com/sponsors/mrlesmithjr).
Enterprise support tiers are available.

## License

MIT

## Author

Larry Smith Jr. — [everythingshouldbevirtual.com](http://everythingshouldbevirtual.com) · [mrlesmithjr@gmail.com](mailto:mrlesmithjr@gmail.com)
