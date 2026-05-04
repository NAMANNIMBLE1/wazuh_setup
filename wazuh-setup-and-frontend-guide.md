# Wazuh All-in-One Setup + Frontend Editing Guide
### Ubuntu — Indexer · Server · Dashboard

---

## PART 1 — Before You Start

### Check your machine specs

Run these commands first. If you don't meet the minimums, the indexer will silently fail or crash.

```bash
# RAM — you need at least 8GB free
free -h

# CPU cores — 4+ recommended
nproc

# Disk — you need 50GB+ free
df -h /

# Ubuntu version — must be 22.04 or 24.04 LTS
lsb_release -a
```

### Get your machine's real IP address

This is the most important step. Wazuh's installer breaks if you use `localhost` or `127.0.0.1` — you must use your actual LAN IP.

```bash
# Look for the inet address on your main interface (e.g. eth0, ens33, wlan0)
ip a | grep "inet " | grep -v 127.0.0.1
```

Write down the IP you see — something like `192.168.1.105`. You will use it in **every** node config entry.

### Set JVM memory for OpenSearch (the indexer)

The indexer uses OpenSearch under the hood, which is Java-based and needs this kernel setting or it refuses to start.

```bash
sudo sysctl -w vm.max_map_count=262144

# Make it permanent across reboots
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

---

## PART 2 — Download the Installer

```bash
# Go to your home directory
cd ~

# Download the assisted installer script and the config template
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.14/config.yml

# Confirm both files are there
ls -lh wazuh-install.sh config.yml
```

---

## PART 3 — Configure All Three Nodes

Open config.yml and replace every placeholder IP with your actual IP from Part 1.

```bash
nano config.yml
```

The file should look like this (replace `YOUR_IP` with your real IP on every line):

```yaml
nodes:
  # ── INDEXER (OpenSearch) ──────────────────────
  indexer:
    - name: node-1
      ip: "YOUR_IP"        # e.g. 192.168.1.105

  # ── SERVER (Wazuh Manager + Filebeat) ─────────
  server:
    - name: wazuh-1
      ip: "YOUR_IP"        # same IP as above

  # ── DASHBOARD (Web UI) ────────────────────────
  dashboard:
    - name: dashboard
      ip: "YOUR_IP"        # same IP again
```

Save with Ctrl+O then Enter, exit with Ctrl+X.

---

## PART 4 — Generate Certificates and Passwords

This step creates all TLS certificates and a passwords file. Everything gets bundled into `wazuh-install-files.tar`.

```bash
sudo bash wazuh-install.sh --generate-config-files
```

You should see it create `wazuh-install-files.tar` in your current directory.

```bash
# Verify it was created
ls -lh wazuh-install-files.tar
```

---

## PART 5 — Install the Three Components (in order)

**Order matters.** The indexer must be running before you install the server. The server must be running before you install the dashboard.

### Step 5A — Install the Indexer

```bash
sudo bash wazuh-install.sh --wazuh-indexer node-1
```

This takes a few minutes. When it finishes, start the cluster:

```bash
sudo bash wazuh-install.sh --start-cluster
```

Verify the indexer is healthy:

```bash
# Get the admin password first
sudo tar -axf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt -O | grep -P "'admin'" -A 1

# Then test the indexer is responding (replace PASS and YOUR_IP)
curl -k -u admin:PASS https://YOUR_IP:9200
# You should see a JSON response with cluster_name and status
```

### Step 5B — Install the Server (Wazuh Manager + Filebeat)

```bash
sudo bash wazuh-install.sh --wazuh-server wazuh-1
```

Verify it's running:

```bash
sudo systemctl status wazuh-manager
# Should say: active (running)
```

### Step 5C — Install the Dashboard

```bash
sudo bash wazuh-install.sh --wazuh-dashboard dashboard
```

At the end of this step the installer prints your admin credentials — **copy them somewhere safe**.

---

## PART 6 — Verify Everything is Running

```bash
# Check all three services
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-dashboard

# Check all ports are listening
sudo ss -tlnp | grep -E ':(443|9200|55000|1514|1515)'
```

Expected ports:
- `443`   → Dashboard (HTTPS web UI)
- `9200`  → Indexer (OpenSearch API)
- `55000` → Wazuh Manager REST API
- `1514`  → Agent communication
- `1515`  → Agent enrollment

### Open the Dashboard in your browser

```
https://YOUR_IP
```

Accept the self-signed certificate warning. Log in with username `admin` and the password the installer printed.

### Lock the repositories so apt doesn't accidentally upgrade Wazuh

```bash
sudo sed -i "s/^deb/#deb/" /etc/apt/sources.list.d/wazuh.list
sudo apt update
```

---

## PART 7 — Understanding the Frontend File Structure

Now that everything is running, here is a complete map of every directory that matters for frontend editing.

### The key insight: two layers of files

Wazuh Dashboard is built on top of **OpenSearch Dashboards**, which is itself a fork of Kibana. The Wazuh-specific UI lives as a plugin inside it. There are two layers:

1. **The base platform** — OpenSearch Dashboards core (don't touch this)
2. **The Wazuh plugin** — everything Wazuh-specific (this is what you edit)

### Complete directory map

```
/usr/share/wazuh-dashboard/          ← ROOT of the whole dashboard app
│
├── bin/                             ← Start/stop scripts
├── config/                          ← Symlink to /etc/wazuh-dashboard/
├── node_modules/                    ← Node.js dependencies (don't touch)
├── src/                             ← OpenSearch Dashboards core source
│
└── plugins/
    └── wazuh/                       ← THE WAZUH PLUGIN — this is your target
        │
        ├── public/                  ← All frontend files served to the browser
        │   ├── assets/              ← Images, logos, icons
        │   │   └── custom/
        │   │       └── images/      ← Put custom logos here
        │   │
        │   ├── *.chunk.js           ← Built/bundled JS (minified production files)
        │   ├── *.chunk.css          ← Built/bundled CSS
        │   └── index.js             ← Entry point
        │
        ├── server/                  ← Node.js backend of the plugin (API routes)
        └── common/                  ← Shared types/constants between front and back
```

### Configuration files (separate from the plugin)

```
/etc/wazuh-dashboard/
├── opensearch_dashboards.yml        ← Main dashboard config (host, ports, certs)
└── certs/                           ← TLS certificates

/usr/share/wazuh-dashboard/data/wazuh/config/
└── wazuh.yml                        ← Wazuh plugin config (API connection, branding)
                                      (this file is auto-generated on first login)
```

### Explore the structure yourself

```bash
# See the full wazuh plugin directory
sudo ls -la /usr/share/wazuh-dashboard/plugins/wazuh/

# See what's inside public/ — this is where the built frontend files are
sudo ls -la /usr/share/wazuh-dashboard/plugins/wazuh/public/

# See the assets folder (logos, images)
sudo ls -la /usr/share/wazuh-dashboard/plugins/wazuh/public/assets/

# See the config file (after you log in once)
sudo cat /usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml
```

---

## PART 8 — Why the Files Look Like Chunks (and How to Work With Them)

### What you're seeing and why

When you open `/usr/share/wazuh-dashboard/plugins/wazuh/public/` you will see files named like:

```
chunk.3f8a2b1c.js
chunk.a7d9e2f1.js
main.bundle.css
```

These are **production build artifacts** — webpack took hundreds of organized React component files and merged + minified them into a few large files. The hash in the name (the random letters) changes every time the code is rebuilt, which forces browsers to fetch the latest version instead of using a cached old one.

This is exactly what your company server has too. It's not a different version — it's the same code, just compiled.

### The organized source lives on GitHub

The readable, organized source code that produces those chunks is at:
```
https://github.com/wazuh/wazuh-dashboard-plugins
```

Inside that repo, the main plugin source looks like:

```
plugins/main/public/
├── components/           ← Every React component, organized by feature
│   ├── common/           ← Shared UI elements (buttons, tables, modals)
│   ├── overview/         ← The main Overview/Home page
│   ├── management/       ← Management section
│   ├── agents/           ← Agent management views
│   └── security/         ← Security module views
│
├── controllers/          ← AngularJS controllers (legacy parts)
├── services/             ← API service layer
├── utils/                ← Helper functions
└── styles/               ← SCSS stylesheets
```

---

## PART 9 — Two Ways to Edit the Frontend

### Method A — Direct edit of built files (quick hacks, CSS changes)

This works for CSS tweaks and small JS changes where you know exactly what to look for. It's fragile — a restart or reinstall wipes your changes — but it's the fastest feedback loop.

```bash
# Give yourself write permission to the plugin directory
sudo chown -R $USER:$USER /usr/share/wazuh-dashboard/plugins/wazuh/

# For CSS changes — find and edit the main CSS bundle
ls /usr/share/wazuh-dashboard/plugins/wazuh/public/*.css

# Open it in nano (it will be minified/unreadable but searchable)
nano /usr/share/wazuh-dashboard/plugins/wazuh/public/main.bundle.css

# After saving, restart the dashboard to pick up changes
sudo systemctl restart wazuh-dashboard
```

For logos and images, this is actually straightforward:

```bash
# Replace the app logo (check the config for exact filename expected)
sudo cp your-logo.png /usr/share/wazuh-dashboard/plugins/wazuh/public/assets/custom/images/

# Update wazuh.yml to point to your logo
sudo nano /usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml
```

### Method B — Clone the source, edit, build, deploy (proper development workflow)

This is how you make real, sustainable changes — the same way your company built their version.

**Step 1 — Find your installed Wazuh version:**

```bash
sudo dpkg -l | grep wazuh-dashboard
# Note the version number e.g. 4.14.0
```

**Step 2 — Install build dependencies:**

```bash
# Install Node.js 18 (required by Wazuh dashboard build)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install yarn
sudo npm install -g yarn

# Verify
node --version    # should be v18.x
yarn --version
```

**Step 3 — Clone the source and check out the matching version tag:**

```bash
cd ~
git clone https://github.com/wazuh/wazuh-dashboard-plugins.git
cd wazuh-dashboard-plugins

# List tags to find your version
git tag | grep "v4.14"

# Check out the exact version you installed (replace with your version)
git checkout v4.14.0
```

**Step 4 — Install dependencies and explore:**

```bash
cd plugins/main
yarn install

# Now browse the organized source
ls public/components/
ls public/components/overview/
ls public/styles/
```

**Step 5 — Make your change. For example, to edit the main Overview page:**

```bash
# Find the component you want to change
find public/components -name "*.tsx" | head -20

# Edit it
nano public/components/overview/index.tsx
```

**Step 6 — Build the plugin:**

```bash
# From inside plugins/main/
yarn build

# The output goes to the build/ folder
ls build/wazuh/public/
# Now you'll see the chunk.*.js files — this is what was on your server
```

**Step 7 — Deploy your build to the running dashboard:**

```bash
# Back up the original first (always do this)
sudo cp -r /usr/share/wazuh-dashboard/plugins/wazuh \
           /usr/share/wazuh-dashboard/plugins/wazuh.backup

# Copy your built files in
sudo cp -r ~/wazuh-dashboard-plugins/plugins/main/build/wazuh/* \
           /usr/share/wazuh-dashboard/plugins/wazuh/

# Restart to load the new files
sudo systemctl restart wazuh-dashboard

# Watch the logs as it starts up
sudo journalctl -fu wazuh-dashboard
```

**Step 8 — Set up a watch/rebuild loop for faster iteration:**

```bash
# In one terminal — auto-rebuild on file save
cd ~/wazuh-dashboard-plugins/plugins/main
yarn watch    # rebuilds on every file change

# In another terminal — re-deploy and restart after each build
# (you can automate this with a shell script)
while inotifywait -e close_write ~/wazuh-dashboard-plugins/plugins/main/build; do
  sudo cp -r ~/wazuh-dashboard-plugins/plugins/main/build/wazuh/* \
             /usr/share/wazuh-dashboard/plugins/wazuh/
  sudo systemctl restart wazuh-dashboard
  echo "Deployed and restarted at $(date)"
done
```

---

## PART 10 — Quick Reference: Most Useful Commands

```bash
# ── SERVICE MANAGEMENT ──────────────────────────────────
sudo systemctl start|stop|restart|status wazuh-indexer
sudo systemctl start|stop|restart|status wazuh-manager
sudo systemctl start|stop|restart|status wazuh-dashboard

# ── LOGS (watch in real time) ───────────────────────────
sudo journalctl -fu wazuh-dashboard        # dashboard logs
sudo tail -f /var/ossec/logs/ossec.log     # manager/server logs
sudo tail -f /var/log/wazuh-indexer/*.log  # indexer logs

# ── KEY PATHS ───────────────────────────────────────────
# Plugin frontend files (built chunks)
/usr/share/wazuh-dashboard/plugins/wazuh/public/

# Plugin assets (logos, images)
/usr/share/wazuh-dashboard/plugins/wazuh/public/assets/

# Wazuh plugin config (API connection, branding)
/usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml

# Dashboard main config (host, ports, certs, OpenSearch connection)
/etc/wazuh-dashboard/opensearch_dashboards.yml

# TLS certificates
/etc/wazuh-dashboard/certs/

# Manager rules and decoders
/var/ossec/etc/rules/
/var/ossec/etc/decoders/

# ── PASSWORDS ───────────────────────────────────────────
# View all generated passwords
sudo tar -axf ~/wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt -O

# ── RESTORE BACKUP IF SOMETHING BREAKS ─────────────────
sudo cp -r /usr/share/wazuh-dashboard/plugins/wazuh.backup \
           /usr/share/wazuh-dashboard/plugins/wazuh
sudo systemctl restart wazuh-dashboard
```

---

## Troubleshooting Common Issues

**Dashboard says "server is not ready yet"** — the dashboard can't reach the indexer. Check `/etc/wazuh-dashboard/opensearch_dashboards.yml` and confirm `opensearch.hosts` points to your correct IP, not localhost.

**wazuh.yml not found** — this file is auto-generated on first login. Log in to the dashboard once first, then it will appear.

**Indexer fails to start** — almost always the `vm.max_map_count` setting. Run `sudo sysctl -w vm.max_map_count=262144` and try again.

**Certificate errors in browser** — expected on first visit. Click Advanced → Accept the risk and proceed. Or import `/etc/wazuh-dashboard/certs/root-ca.pem` into your browser.

**Port 443 in use** — if something else (Apache, Nginx) is running on port 443, stop it first with `sudo systemctl stop apache2` or `sudo systemctl stop nginx`.
