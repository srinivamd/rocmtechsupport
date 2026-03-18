# Self-Hosted Runner Setup for MI350 Docker Monitor

The `mi350-docker-monitor` workflow runs `docker ps` **directly on each MI350
server** using self-hosted runners installed on the hosts.  No SSH keys or
VPN tunnels are required.

## Runner Naming Convention

Runners must be registered with a name that includes the server hostname so
the workflow matrix can target each one individually.  Expected names:

| Server hostname                          | Runner name                                  |
|------------------------------------------|----------------------------------------------|
| cv350-1e707-c03-2.mkm.dcgpu             | `aiswhud-mi350-cv350-1e707-c03-2`           |
| smci350-zts-gtu-b14-05.zts-gtu.dcgpu   | `aiswhud-mi350-smci350-zts-gtu-b14-05`      |
| cv350-zts-gtu-h41-08.zts-gtu.dcgpu     | `aiswhud-mi350-cv350-zts-gtu-h41-08`        |
| gbt350-odcdh1-b10-1.png-odc.dcgpu      | `aiswhud-mi350-gbt350-odcdh1-b10-1`         |

If your actual runner names differ, update the `runner:` fields in
`.github/workflows/mi350-docker-monitor.yml` → `jobs.probe.strategy.matrix`.

---

## Install the Runner on Each MI350 Server

Run the following on **each** MI350 server (substitute the correct token and
runner name for that host).

```bash
# 1. Create a dedicated runner user (optional but recommended)
sudo useradd -m -s /bin/bash github-runner
sudo usermod -aG docker github-runner   # allow docker ps without sudo
sudo su - github-runner

# 2. Download the Actions runner (check https://github.com/actions/runner/releases for latest)
mkdir ~/actions-runner && cd ~/actions-runner
curl -O -L https://github.com/actions/runner/releases/download/v2.322.0/actions-runner-linux-x64-2.322.0.tar.gz
tar xzf actions-runner-linux-x64-2.322.0.tar.gz

# 3. Register — get TOKEN from:
#    GitHub → repo → Settings → Actions → Runners → New self-hosted runner
#    Replace <TOKEN> and <RUNNER-NAME> for each server:
./config.sh \
  --url https://github.com/amddcgpuce/rocmtechsupport \
  --token <TOKEN> \
  --name aiswhud-mi350-cv350-1e707-c03-2 \
  --labels self-hosted,mi350,aiswhud-mi350-cv350-1e707-c03-2 \
  --unattended

# 4. Install and start as a systemd service
sudo ./svc.sh install github-runner
sudo ./svc.sh start
sudo ./svc.sh status
```

Repeat for each server, changing `--name` and `--labels` to match the table above.

---

## Verify Runner Registration

After installing on all servers, confirm all four appear online:

```
GitHub → repo → Settings → Actions → Runners
```

Each runner should show status **Idle**.

---

## No Secrets Required

Because each job runs **on** the MI350 server (not SSH-ing into it), the
workflow needs no `MI350_SSH_KEY` or `MI350_KNOWN_HOSTS` secrets.
The runner process executes `docker ps` with the permissions of the
`github-runner` user, which must be in the `docker` group (see step 1 above).

---

## Trigger a Manual Test Run

```
GitHub → Actions → MI350 Docker Process Monitor → Run workflow
```

- Leave `extra_cmd` blank for a plain `docker ps`.
- Or enter e.g. `docker images` to also list pulled images on every server.

## View Results

| Where | What |
|-------|------|
| Actions → run → Summary | Consolidated table + per-server `docker ps` blocks |
| Actions → run → Artifacts | Per-server `.txt` files (retained 7 days) |
| Actions → MI350 Docker Process Monitor | Full hourly history |
