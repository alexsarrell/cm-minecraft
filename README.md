# CM Minecraft Ansible

Opinionated Ansible playbook and role to deploy and operate a Minecraft server on a remote host. Supports version-aware, idempotent deploys, automatic EULA acceptance on first deploy of a given version, mods synchronization, systemd + screen daemonization, health checks, and log retrieval on failures.

## Repository layout
```
inventory/
  hosts.ini            # Ansible inventory
playbook.yml           # Entry playbook
roles/
  minecraft/
    defaults/
      main.yml         # Tunables (version, paths, memory, port)
    tasks/
      main.yml         # Orchestrator importing task files
      setup_env.yml    # Facts, env, change detection
      deploy_version.yml
      deploy_mods.yml
      install_loaders.yml
    templates/
      eula.txt.j2
      systemd-minecraft.service.j2
      server.properties.j2
      dotenv.j2
mods/                  # Optional: local mods to sync (default path)
fetched_logs/          # Latest logs fetched on failures
```

## Prerequisites
- Control node: Ansible, Python 3, rsync
- Remote host: Debian/Ubuntu or RHEL/CentOS family; role installs: OpenJDK 21, screen, curl, unzip, rsync
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
- minecraft_version: "1.21.1"
- server_dir: "/opt/minecraft"
- versions_dir: "{{ server_dir }}/versions"
- version_dir: "{{ versions_dir }}/{{ minecraft_version }}"
- current_link: "{{ server_dir }}/current"
- service_name: minecraft
- server_port: 25565
- xms: 1G
- xmx: 8G
- mods_src: "{{ lookup('env','MC_MODS') | default('mods', true) }}"  # Local path synced to remote
- loader: vanilla|fabric|forge|neoforge
- loader_version: e.g. "21.1.208" (Forge/NeoForge) or Fabric installer version
- java_version: 21 (java_bin_path configurable in defaults)
- healthcheck_retries: 180; healthcheck_interval: 1; healthcheck_total_timeout: 30000

Env var fallback (lower priority than playbook/inventory vars):
- MC_VERSION, MC_SERVER_DIR, MC_MODS

## Overridable parameters (playbook.yml or -e)

| Name | Type | Default | Description |
|------|------|---------|-------------|
| minecraft_version | string | 1.21.1 | Minecraft server version to deploy |
| loader | enum | neoforge | Server loader: vanilla, fabric, forge, neoforge |
| loader_version | string | 21.1.208 | Loader version (Forge/NeoForge) or Fabric installer version |
| server_dir | path | /opt/minecraft | Base directory on remote host |
| versions_dir | path | {{ server_dir }}/versions | Per-version directory root |
| current_link | path | {{ server_dir }}/current | Symlink to active version dir |
| service_name | string | minecraft | systemd service name |
| server_port | int | 25565 | Minecraft TCP port |
| xms | string | 1G | JVM initial heap size |
| xmx | string | 8G | JVM max heap size |
| mods_src | path (local) | ./mods | Local mods folder to rsync |
| java_version | int | 21 | Java major version to install |
| java_pkg_deb | string | openjdk-{{ java_version }}-jre-headless | Debian package name |
| java_pkg_rpm | string | java-{{ java_version }}-openjdk | RHEL package name |
| java_bin_path | path | /usr/lib/jvm/java-{{ java_version }}-openjdk/bin/java | JAVA_BIN used by service |
| healthcheck_retries | int | 180 | Max curl attempts before failing |
| healthcheck_interval | int (seconds) | 1 | Delay between curl attempts |
| healthcheck_total_timeout | int (seconds) | 30000 | Fallback to derive retries if not set |
| fabric_installer_url | url | computed | Fabric installer URL override |
| forge_installer_url | url | computed | Forge installer URL override |
| neoforge_installer_url | url | computed | NeoForge installer URL override |
| post_start_grace | int | 0 | Extra pause before checks (usually 0) |
| clean_build | bool | false | Deprecated; ignored |

Env var fallback (lowest precedence): MC_VERSION, MC_SERVER_DIR, MC_MODS

## How it works (idempotency + version awareness)
- Per-version dirs: {{server_dir}}/versions/{{minecraft_version}}
- Symlink: {{server_dir}}/current points to the active version dir
- First deploy of a version:
  - Downloads server (vanilla or via loader installer), writes eula.txt, syncs mods, creates/updates systemd service, points current symlink, starts service, healthchecks
- Subsequent runs with same version:
  - Syncs mods to {{server_dir}}/current/mods; service is restarted on every run (always)
- Version or loader change:
  - Stops service, provisions new version dir, performs full deploy steps, switches symlink, starts and healthchecks
- Healthcheck: controller curls server_port until curl rc==52 (Empty reply); retries/interval configurable; on failure, logs fetched to ./fetched_logs/latest-<timestamp>.log

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
- On failed deploy/restart, role fetches latest log to ./fetched_logs/latest-<timestamp>.log on the control node

Overriding variables at runtime
- `ansible-playbook -i inventory/hosts.ini playbook.yml -e minecraft_version=1.21.1 -e xmx=8G`

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

## TODOs
- Packwiz integration for mods pack management
  - Detect pack.toml in repo and run packwiz refresh/export on control node
  - Sync generated mods to remote before restart
  - Optional: lockfile handling and validation

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
- Override healthcheck: `-e healthcheck_retries=180 -e healthcheck_interval=1`

## Notes
- Java 21 required for many recent Forge/NeoForge builds
- Systemd unit uses Type=simple + screen; EnvironmentFile=.env with JAVA_BIN/XMS/XMX/PORT/etc
- .env is rendered per-version and kept at {{server_dir}}/versions/{{minecraft_version}}/.env
