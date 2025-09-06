# CM Minecraft Ansible

Opinionated Ansible playbook and role to deploy and operate a Minecraft server on a remote host. Supports version-aware, idempotent deploys, automatic EULA acceptance on first deploy of a given version, mods synchronization, systemd + screen daemonization, health checks, and log retrieval on failures.

## Repository layout
```
inventory/
  hosts.ini            # Your Ansible inventory
playbook.yml           # Entry playbook
roles/
  minecraft/
    defaults/main.yml  # Tunables (version, paths, memory, port)
    tasks/main.yml     # Version-aware deploy logic
    templates/
      eula.txt.j2
      systemd-minecraft.service.j2
mods/                  # Optional: local mods to sync (default path)
```

## Prerequisites
- Control node: Ansible, Python 3, rsync
- Remote host: Debian/Ubuntu or RHEL/CentOS family; role installs: OpenJDK 17, screen, curl, unzip, rsync
- Network: Ensure TCP port 25565 (or your server_port) open in firewall/security groups

## Inventory
inventory/hosts.ini
```
[minecraft]
178.72.129.136 ansible_user=root
```
Add hosts or set host_vars/group_vars as needed (SSH key auth recommended).

## Quick start
- Put your mods (if any) into ./mods
- Optionally edit playbook.yml vars (version/paths/memory/port)
- Run: `ansible-playbook -i inventory/hosts.ini playbook.yml`

## Variables (defaults) – roles/minecraft/defaults/main.yml
- minecraft_version: "1.20.1"
- server_dir: "/opt/minecraft"
- versions_dir: "{{ server_dir }}/versions"
- version_dir: "{{ versions_dir }}/{{ minecraft_version }}"
- current_link: "{{ server_dir }}/current"
- service_name: minecraft
- server_port: 25565
- xms: 1G
- xmx: 2G
- mods_src: "{{ lookup('env','MC_MODS') | default('mods', true) }}"  # Local path synced to remote

Env var fallback (lower priority than playbook/inventory vars):
- MC_VERSION, MC_SERVER_DIR, MC_MODS

## How it works (idempotency + version awareness)
- Per-version dirs: {{server_dir}}/versions/{{minecraft_version}}
- Symlink: {{server_dir}}/current points to the active version dir
- First deploy of a version:
  - Downloads Mojang server.jar for that version, writes eula.txt, syncs mods, creates/updates systemd service, points current symlink, starts service, healthchecks
- Subsequent runs with same version:
  - Only syncs mods to {{server_dir}}/current/mods and restarts the service if mods changed; EULA/service creation/download are skipped
- Version change:
  - Stops service, provisions new version dir, performs full deploy steps, switches symlink, starts and healthchecks
- Healthcheck: waits for port, then scans latest.log for Exception/ERROR/FATAL; on failure, play fails and fetches logs to ./fetched_logs/latest.log

## Common operations
Change Minecraft version
- Edit minecraft_version in playbook.yml (preferred) or pass `-e minecraft_version=1.20.4` or export `MC_VERSION=1.20.4`
- Run the playbook; role stops existing service, deploys new version, switches symlink

Add/update mods
- Place .jar files in the local mods directory (default: ./mods or override mods_src)
- Run the playbook; mods sync to {{server_dir}}/current/mods and service restarts only if mods changed

Change memory limits
- Edit xms/xmx in playbook.yml or set via `-e xms=2G -e xmx=4G`; rerun playbook (service will restart)

Change install path
- Edit server_dir (and optionally service_name) in playbook.yml; rerun playbook

Change server port
- Edit server_port and ensure firewall allows it; rerun playbook

Start/stop/restart manually on remote
- `systemctl status minecraft`
- `systemctl stop minecraft`
- `systemctl start minecraft`
- `journalctl -u minecraft -e`

Logs
- Server logs: {{server_dir}}/current/logs/latest.log
- On failed deploy/restart, role fetches the latest log to ./fetched_logs/latest.log on the control node

Overriding variables at runtime
- `ansible-playbook -i inventory/hosts.ini playbook.yml -e minecraft_version=1.20.4 -e xmx=4G`

Using env fallback (lowest precedence)
- `export MC_VERSION=1.20.4` `export MC_MODS=/path/to/mods` then run playbook

## Tips
- Keep the control-node mods folder under version control to track mod set changes
- Test version upgrades in a staging server first; some worlds/mods are not forward/backward compatible
- Backup world data (world/ dir under current) before major upgrades
- If you use Forge/Fabric, ensure the correct server.jar loader is used; this role downloads the vanilla Mojang server by default and does not install mod loaders

## Troubleshooting
- Play failed and fetched logs:
  - Inspect ./fetched_logs/latest.log and remote {{server_dir}}/current/logs/latest.log
- Service won’t start:
  - `journalctl -u minecraft -e` for systemd logs; check Java version (17) and memory settings
- Mods didn’t sync:
  - Confirm rsync installed on control node; ensure mods_src path is correct
- Port not open externally:
  - Update firewall/security groups to allow server_port from your clients

## Advanced
Multiple servers
- Use multiple inventory hosts; override service_name, server_port, server_dir per host in host_vars

Separate mods per version
- Point mods_src to a versioned local path when changing versions to keep clean sets per version

## Commands reference
- Run: `ansible-playbook -i inventory/hosts.ini playbook.yml`
- Limit to one host: `-l 178.72.129.136`
- Check mode: `--check` (download/rsync won’t fully apply)
- Verbose: `-vvv`

## Notes
- Java 17 required for modern server versions (1.18+)
- Screen is used under systemd to provide console detachment; service name defaults to "minecraft"
