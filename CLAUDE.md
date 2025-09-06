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