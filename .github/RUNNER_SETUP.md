# Self-Hosted Runner Setup for MI350 Monitor

The `mi350-docker-monitor` workflow requires a GitHub Actions self-hosted runner
with network access to the AMD internal `.dcgpu` domain.

## 1. Register the Runner

On any Linux host that can reach the `.dcgpu` servers (e.g., an AMD VPN-connected
jump host, or a server already in the AMD datacenter):

```bash
# Download the runner from:
# GitHub → repo → Settings → Actions → Runners → New self-hosted runner

mkdir ~/actions-runner && cd ~/actions-runner
curl -O -L https://github.com/actions/runner/releases/download/v2.322.0/actions-runner-linux-x64-2.322.0.tar.gz
tar xzf actions-runner-linux-x64-2.322.0.tar.gz

# Configure — add label 'amd-internal' so the workflow targets it
./config.sh \
  --url https://github.com/amddcgpuce/rocmtechsupport \
  --token <TOKEN_FROM_GITHUB_UI> \
  --labels amd-internal \
  --name mi350-monitor-runner

# Install and start as a service
sudo ./svc.sh install
sudo ./svc.sh start
```

## 2. Add GitHub Secrets

In the repo: **Settings → Secrets and variables → Actions**

### `MI350_SSH_KEY`
The private SSH key used to log into the MI350 servers (must match the public
key already installed in `~/.ssh/authorized_keys` on each server via Conductor).

```bash
# Copy the private key content
cat ~/.ssh/id_rsa   # or id_ed25519, whichever key Conductor uses
```

Paste the entire output (including `-----BEGIN ... KEY-----` / `-----END ... KEY-----`
lines) as the secret value.

### `MI350_KNOWN_HOSTS`
Pre-approved host fingerprints to avoid interactive host-key prompts.

```bash
# Run this from the jump host / runner host:
ssh-keyscan \
  cv350-1e707-c03-2.mkm.dcgpu \
  smci350-zts-gtu-b14-05.zts-gtu.dcgpu \
  cv350-zts-gtu-h41-08.zts-gtu.dcgpu \
  gbt350-odcdh1-b10-1.png-odc.dcgpu \
  2>/dev/null
```

Paste the full output as the `MI350_KNOWN_HOSTS` secret value.

## 3. Verify Connectivity

Before the first scheduled run, confirm the runner host can reach all servers:

```bash
for h in \
  cv350-1e707-c03-2.mkm.dcgpu \
  smci350-zts-gtu-b14-05.zts-gtu.dcgpu \
  cv350-zts-gtu-h41-08.zts-gtu.dcgpu \
  gbt350-odcdh1-b10-1.png-odc.dcgpu; do
  ssh -o ConnectTimeout=5 -i ~/.ssh/id_mi350 ssubrama1@${h} hostname && echo "OK: ${h}" || echo "FAIL: ${h}"
done
```

## 4. Trigger Manually

After setup, trigger a test run:

```
GitHub → Actions → MI350 Docker Process Monitor → Run workflow
```

Optionally pass an extra command (e.g. `docker images`) in the `extra_cmd` input.

## 5. View Results

- **Per-run summary:** Actions tab → select the run → Summary tab
- **Per-server artifacts:** each run uploads `docker-ps-<label>.txt` files
  retained for 7 days
- **History:** all runs listed under the `MI350 Docker Process Monitor` workflow
