# CameraBridge agent

Connect your cameras to StreamGuard from behind NAT or CGNAT, with no public IP, port forward or
VPN. The agent runs on a machine on the same local network as the cameras, pulls each camera's RTSP
stream locally, and sends it over one outbound TLS connection on port 443.

The source repository is private. This repository holds only the audited, secret-free,
cosign-signed agent binaries and installers published by its CI.

**You need:**

- A machine that stays on, on the same network as the cameras (not the NVR itself). Linux with
  systemd, or Windows 10/11 or Server. A laptop works if it never sleeps on mains power.
- Outbound internet on port 443, and the clock set automatically.
- Each camera's RTSP address, username and password.
- Someone who can approve the enrollment: a person holding the **site enroller** role for your
  organisation. It can be you. They approve from their own phone or computer.

Setting up a site takes three steps: install, enroll, check.

---

## 1. Install

Both installers refuse to install unless the release signature verifies, so install cosign first.

**Linux** (amd64 or arm64):

```sh
curl -fsSL https://github.com/Digital-Dynamics-Corporation/CameraBridge-dist/releases/latest/download/install.sh | sudo sh
```

Get cosign from your package manager or https://docs.sigstore.dev/cosign/system_config/installation/.

**Windows** (amd64), in PowerShell **run as administrator**:

```powershell
winget install sigstore.cosign
```

Close that window, open a new PowerShell as administrator, then:

```powershell
irm https://github.com/Digital-Dynamics-Corporation/CameraBridge-dist/releases/latest/download/install.ps1 | iex
```

The installer prints `Verified OK` and the version. It installs the `camerabridge-agent` service but
does not start it; enrolling does that.

---

## 2. Enroll

Run one command on the machine. List each camera as `name=rtsp://username@address/path`. The name is
a short lowercase label you choose (`front-door`, `gate`). **Leave the password out**: the agent asks
for each one and keeps it only on this machine.

**Linux:**

```sh
sudo camerabridge-agent enroll --relay camerabridge-relay.vivekmalipatel.com \
  --site-name "Warehouse 3" --location warehouse-3 \
  --camera front-door=rtsp://admin@192.168.1.20:554/stream1 \
  --camera gate=rtsp://admin@192.168.1.21:554/stream1
```

**Windows** (PowerShell as administrator):

```powershell
& "C:\Program Files\CameraBridge\camerabridge-agent.exe" enroll --relay camerabridge-relay.vivekmalipatel.com `
  --site-name "Warehouse 3" --location warehouse-3 `
  --camera front-door=rtsp://admin@192.168.1.20:554/stream1 `
  --camera gate=rtsp://admin@192.168.1.21:554/stream1
```

`--location` is a short lowercase label for this place (letters, digits and hyphens), one per site.
`--site-name` is how the site is shown.

Then:

1. The agent asks for each camera's password. Nothing is shown as you type.
2. It prints a link, a code, and a summary of this machine:

   ```
   To approve this enrollment, open https://sso.vivekmalipatel.com/device and enter  ABCD-EFGH
   Approve it only if you are enrolling this box yourself, right now:
     host   warehouse-pc
     key    N9SNi6FZ...
     site   "Warehouse 3" at warehouse-3
   ```

3. The approver opens the link, enters the code, signs in and approves. **Only approve a code you
   can see on this machine's screen right now.** Never approve a code someone sends you.
   The code expires after 5 minutes; if it does, run the command again.
4. The agent finishes by itself: it registers this machine as its own identity, writes its
   settings, and starts the service. It prints `enrolled site ...`.

The approver's sign-in is used once and never stored on this machine. The machine's private key is
created here and never leaves it.

### Replacing the machine at an existing site

Enroll the new machine with `--rebind <site-id>` instead of `--site-name` and `--location`. The
approver needs the **site rebinder** role. The old machine stops working the moment the new one is
enrolled.

---

## 3. Check

**Linux:**

```sh
systemctl status camerabridge-agent
journalctl -u camerabridge-agent -f
```

**Windows:**

```powershell
Get-Service camerabridge-agent
Get-WinEvent -ProviderName camerabridge-agent -MaxEvents 30 | Format-List TimeCreated,Message
```

A healthy agent logs `login to server success` and, for each camera, `camera=<name> state=streaming`. If a
camera shows `auth-failed`, its password is wrong: the agent then waits 5 minutes between tries so
the camera does not lock the account.

If the site is revoked, or its settings are wrong, the service stops and stays stopped (exit 2)
instead of retrying. Other failures restart it after 10 seconds, at most five times in ten minutes.

---

## Common problems

| You see | Do this |
|---|---|
| `run enroll as root` / `run enroll from an elevated PowerShell` | Use `sudo` on Linux, or PowerShell opened with **Run as administrator** on Windows. |
| `... not installed; run install.sh first` (or `install.ps1`) | Run step 1, then enroll again. |
| `never put a camera password on the command line` | Remove `:password` from the `--camera` address; the agent asks for it. |
| The approver sees "the user is required to have at least one grant" | They do not hold a CameraBridge role yet. Ask your CameraBridge administrator. |
| `does not hold the site-enroller role` | Same: the approver needs the site enroller role. |
| `the approval is too old` or the code expired | Run enroll again and approve within 5 minutes. |
| `429` / `wait a minute` | Too many attempts from your network. Wait a minute, then run enroll again. |
| `the approval is dated in the future; check this box's clock` | Turn on automatic time on this machine. |
| `location ... already has a site` | That location is enrolled. To replace its machine, use `--rebind <site-id>`. |
| Windows SmartScreen warns about the exe | It is not Authenticode-signed yet; the installer verified the signature and checksum before clearing the warning. |

To see every option: `camerabridge-agent enroll -h`.

---

## Update

Re-run the installer. It replaces the binary and restarts the service only if it was running. Pin
a version with `sh -s -- --version vX.Y.Z` on Linux, or `$env:CAMERABRIDGE_VERSION = "vX.Y.Z"` before
the `irm ... | iex` line on Windows.

```sh
camerabridge-agent version
```

---

## Verify (supply chain)

Every release ships `SHA256SUMS`, `SHA256SUMS.sig` and `SHA256SUMS.pem` (Sigstore keyless) and a
CycloneDX SBOM. The installers check these for you:

```sh
cosign verify-blob SHA256SUMS --signature SHA256SUMS.sig --certificate SHA256SUMS.pem \
  --certificate-identity-regexp '^https://github\.com/Digital-Dynamics-Corporation/StreamGuard/\.github/workflows/camerabridge-release\.yml@refs/tags/camerabridge-agent-v' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
sha256sum -c SHA256SUMS --ignore-missing
```
