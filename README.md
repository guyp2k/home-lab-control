# Home Lab Control

A single dashboard for running a home lab: what's up, what broke overnight, how the network is laid out, where every container lives, and how to get back in when something goes wrong.

It runs on the lab's own Unraid server, stays on the LAN behind basic auth, and has no cloud dependency.

![Services](screenshots/services.png)

> **Source code:** not public yet. This page describes what the dashboard does.
> All screenshots use made-up example data. No real hosts, addresses or credentials appear in them.

## What It Does

### Services
Every service in the lab gets a tile, grouped by role (Core, Media, Smart Home, AI, Git & CI, Handoff). Each tile shows whether the service is up, refreshes every 30 seconds, and opens the service's web UI on click. The header keeps a running count ("21 up / 23 total") so a problem is visible from any tab.

Each group folds into a section, and a closed section's header still shows a live count, such as "2 down · 6 up". A section opens by itself while something in it is down and closes again once it recovers. Anything you open or close stays that way in your browser, and Expand all / Collapse all handle every section at once.

Some tiles show live stats as well as up or down. The Ollama tile shows which model is loaded and how much of it sits on the GPU. The GPU tile shows load, VRAM, temperature and power.

### Health
Every morning each Unraid server runs a read-only Daily Health Check and posts its report to the dashboard and to Telegram. A second check reads the router's and Home Assistant's logs from the central syslog collector and reports on those too.

![Health](screenshots/health.png)

Each host gets a card that is green, amber or red. Problems, warnings and notes are listed in plain language, findings that weren't there yesterday are marked **NEW**, and every card lists what was checked.

A card also turns red if its host hasn't reported for over a day. A host that is down can't report that it's down, so a missing report counts as a problem in itself.

### Logs
Syslog from every server, the router and Home Assistant, collected in one place and kept for 30 days. You can filter by host and app, or search with a regex. Logs are stored in Loki and viewed in Grafana inside the dashboard, behind the same login.

### Network
A topology diagram of the whole network (router, switches, access points, servers) and the VLAN plan as an editable table.

The diagram is live:
- **Status on every box.** A dot shows whether each device answers: green up, red down, grey not checked yet. A cold-standby box is marked as standby, not as an outage.
- **Click for details.** Clicking a box (or tabbing to it and pressing Enter) opens a side panel with its live status, its address and ports, and a link to its web UI.
- **Wires from UniFi.** Each cable is coloured from its switch port on the UniFi controller: green 10 GbE, teal 2.5 GbE, cyan 1 GbE, amber slower, red link down. Hovering a wire shows both ends, the link speed, current traffic, PoE draw and errors. The panel adds what UniFi knows about the device: clients on an access point, ports up on a switch, and the mesh parent of a wireless extender.
- **Honest when data is missing.** If the controller can't be read, the wires stay as drawn and a note under the diagram says why. If a reading goes stale, the last good one stays on screen, marked stale.

![Network](screenshots/network.png)

### Reference
The lab's inventory, all of it editable in place:
- Core devices
- UniFi switches and access points
- Port-by-port switch maps
- Each server's network interfaces
- DNS and blocklist versions
- Docker containers and compose stacks

Docker containers are grouped by server, then by VLAN, so you can see at a glance what runs where and on which address. Stopped containers are listed too.

![Docker Containers](screenshots/docker-containers.png)

Compose stacks get their own table: which containers each stack owns, whether Unraid's Compose Manager or the command line runs it, whether it starts on boot, and where its files live.

![Docker Compose](screenshots/docker-compose.png)

### Documents
A read-only browser for the lab's document share: network drawings, runbooks, PDFs. Click a folder to open it, a file to download it.

![Documents](screenshots/documents.png)

### Logins
A link to the self-hosted password vault (Vaultwarden), an index of which accounts exist and where each password is kept, and a step-by-step break-glass procedure for getting in if the owner isn't available.

**No passwords are stored in the dashboard.** The index only says where each one lives in the vault.

![Logins](screenshots/logins.png)

### Open Items and Lessons
Open Items is a prioritized follow-up list.

Lessons is a searchable record of things that went wrong and how they were fixed, so the same mistake isn't made twice.

![Open Items](screenshots/open-items.png)

![Lessons](screenshots/lessons.png)

### AI Explain
Every problem, warning and note on the Health tab gets an **Explain** button. Clicking it asks a language model running locally on the lab's own GPU (through Ollama) for the likely cause and the next steps to check. One switch in the Health header turns the feature on or off for the whole page.

![AI Explain](screenshots/ai-explain.png)

How it works:
- **Local only.** The model runs on the lab's GPU. Logs never leave the LAN.
- **Advice, never action.** The model has no tools and can't run commands. Every answer is labelled "AI suggestion · verify before acting" and ends with "Advice only. Nothing was run on any host."
- **Shows its evidence.** When a finding quotes a log line, the matching lines from the last two days are pulled from the log store, given to the model and shown under the answer, so you can check its reasoning against the source.
- **Uses the lab's own history.** Lessons that match the finding are given to the model too, and the answer names them.
- **Plays nicely with other GPU users.** It uses whichever model is already loaded rather than swapping in one of its own. It waits its turn if the GPU is busy, and only one explanation runs at a time.
- **Saves answers.** A finding that comes back the next day, with only its count or timestamp changed, reuses the earlier answer. **Ask again** gets a fresh one.
- **Treats logs as untrusted.** Credentials are scrubbed from every line before the model sees it, and log text is passed as data, never as instructions.
- **Skips stale cards.** Findings on a host that hasn't reported for over a day get no button.

## How It's Built

- **Web:** a static page served by nginx, with no build step and no framework. Editable tables save straight back to the server.
- **API:** a small Python service using only the standard library. It checks each service's status, serves and stores the editable tables, and receives the daily health reports. The tables can only be read or changed through the dashboard itself.
- **Link data:** the same service reads the UniFi controller every 30 seconds for each switch port's link state, speed and traffic, and each device's uplink.
- **GPU stats:** a separate small service reads the GPU through the NVIDIA driver, so the main API needs no GPU access.
- **Logs:** Grafana Alloy ships syslog to Loki, and Grafana shows it inside the Logs tab.
- **AI:** Ollama on a local GPU answers AI Explain requests. The API reads evidence from Loki and sends the model only scrubbed text.
- **Health checks:** read-only scripts on each Unraid server, run daily by Unraid's User Scripts plugin.
- **Deploy:** Docker Compose on Unraid.
  - Merged changes to the page and the compose file deploy automatically.
  - Every change goes through CI and a mandatory review before it merges.

## Security

- LAN only, behind basic auth. Nothing is exposed to the internet.
- No secrets live in the dashboard or its repository. Passwords stay in the vault.
- The editable tables, AI Explain and the link data accept requests only from the dashboard's web server.
- UniFi is read with a View Only local account. Its login lives in a file on the server, never in the repository.
- Health reports are accepted only from the lab's own servers, and each server can update only its own card.
