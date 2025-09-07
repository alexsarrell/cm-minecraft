# Minecraft server
- Host: 178.72.129.136
- Root user: root

# About
I want to make ansible inventory from that repo for deploying minecraft server to remote server. Core concepts: i want to that ansible pipeline could: 1. Download minecraft
server with version specified in ansible playbook (with environment variables or in YML config of playbook, with priopity to the second option). Then accept eula.txt user
agreement, deploy mods from specified in playbook mod folder, and then run minecraft server as a daemon (in a screen mode, for example). If startup fails - you shoud
implement the best healthcheck for that, playbook should fail with error result. Perfectly if information about error from minecraft logs automatically exports from server
to mine local computer.

Note, minecraft server is only deploying for the first time running that playbook (if clean_build is not set). Also it touches EULA.txt, service creation and etc. It should run only once. I
want such loging as checking current minecraft version (in configs on the server for example) and redeploing that only if version was changed. If version hasnt changed i
want only deploy new mods and restart current minecraft server version. If minecraft version was changed - i want to stop current version of minecraft, create a new
directory for new minecraft version and run full playbook - with EULA.txt, downloading a new version and etc.

# Workflow
- Use ssh while authenticating to minecraft server.
- Minecraft server version to download should be configurable.
- Don't forget about eula.txt rights consumption for first deploy minecraft server.
- I want to have configurable folder path to minecraft mods, that i want to deploy to the minecraft server. Perfectly, i want to have version of control (some public/local git repo to control mod set).

# How should work
- Healtcheck should be performed while curl receiving curl: (7) Failed to connect to localhost port 25565 after 0 ms: Couldn't connect to server. When getting curl: (52) Empty reply from server - finish with success.

Scope implemented
- Inventory/playbook/role scaffolded. Inventory at inventory/hosts.ini; main playbook at playbook.yml; role at roles/minecraft
- Version-aware, idempotent deploy:
    - Per-version dir: {{server_dir}}/versions/{{minecraft_version}}
    - Active symlink: {{server_dir}}/current
    - Full deploy runs when version changes or clean_build=true; otherwise mods-only path
- Mods deploy:
    - Rsync from control node to {{current_link}}/mods (or new {{version_dir}}/mods)
    - Skips if local mods folder missing; uses rsync --itemize-changes with debug output; restarts service only if mods changed
- Java management:
    - Parametrized by package/bin names (not numbers):
        - java_pkg_deb (e.g., openjdk-21-jre-headless)
        - java_pkg_rpm (e.g., java-21-openjdk)
        - java_bin_path (e.g., /usr/lib/jvm/java-21-openjdk/bin/java)
    - Installs with apt/yum and sets default via alternatives/update-alternatives
- Loader support:
    - loader: vanilla|fabric|forge|neoforge
    - loader_version used to construct installer URLs in defaults
    - Vanilla: java -jar server.jar nogui
    - Fabric: java -jar fabric-server-launch.jar nogui (installer downloads Minecraft and launcher)
    - Forge/NeoForge: run via run.sh (screen starts /bin/bash run.sh nogui)
- Systemd service (templated):
    - Type=forking, WorkingDirectory={{current_link}}, runs inside screen
    - NeoForge/Forge use run.sh; Fabric uses fabric-server-launch.jar; Vanilla uses server.jar
    - Wants/After network-online.target; Restart=on-failure; StartSec tuned
- Service control in tasks:
    - Full-deploy path: create/update unit, update symlink, daemon-reload, ensure enabled, then non-blocking restart
    - Stop path tries graceful stop, waits for world/session.lock, force-kills lingering screen/java if needed
- Healthchecks:
    - wait_for port {{server_port}}, wait for logs/latest.log to appear, pause for post_start_grace seconds, then grep for Exception|ERROR|FATAL
    - On failure, fetches logs to ./fetched_logs/latest.log (see TODO to timestamp)
- EULA acceptance: eula.txt templated to version dir

Key variables (with examples)
- minecraft_version: "1.21.1"
- loader: "vanilla"|"fabric"|"forge"|"neoforge"
- loader_version: e.g. "21.1.208" (Forge/NeoForge); Fabric installer version
- server_dir: "/opt/minecraft"; versions_dir/current/version_dir derived
- mods_src: "./mods" (local path on control node)
- server_port: 25565
- xms/xmx: e.g. 1G / 8G
- clean_build: false|true (force full deploy)
- post_start_grace: seconds to wait before scanning logs
- Java:
    - java_pkg_deb: openjdk-21-jre-headless
    - java_pkg_rpm: java-21-openjdk
    - java_bin_path: /usr/lib/jvm/java-21-openjdk/bin/java

Important notes
- NeoForge and many Forge builds require Java 21; defaults now set to Java 21
- Fabric/Forge/NeoForge installers are supported; server.jar is symlinked/selected appropriately
- Mods sync requires rsync on control node; enable SSH key auth for remote

Remaining TODOs
- Fetch logs with timestamped name latest-<timestamp>.log in both startup and restart rescue blocks (currently static latest.log)
- Optionally re-template systemd unit outside full-deploy and restart if unit changed (to pick up xms/xmx/port changes without clean_build)
- Implement realistic healthcheck, that no checking logs for ERRORS and something like that, thats weird. Healthcheck should just send curl to the server from hosts.ini on port from vars. While server is restarting - port is unreachable and returns "curl: (7) Failed to connect to 178.72.129.136 port 25565 after 8 ms: Couldn't connect to server". When ready - 200, and "curl: (52) Empty reply from server". If Health check timeout is exeeeded and great response isn't received - you should export logs to fetched_logs. In that refactoring, also, remove crutch-timeouts like waiting 60s before end of playbook. This functionality should be responsible and flexible - and health-check should be finished right after successful response is received.

How to run
- Set vars in playbook.yml or via -e (loader, loader_version, minecraft_version, java_* vars, xms/xmx, post_start_grace)
- Run: ansible-playbook -i inventory/hosts.ini playbook.yml
- To force full redeploy: -e clean_build=true
