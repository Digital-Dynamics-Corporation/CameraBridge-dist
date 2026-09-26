# CameraBridge agent

Connect a site's cameras to StreamGuard from behind NAT or CGNAT, with no public IP, port forward
or VPN. The agent runs on a machine on the same local network as the cameras, pulls each camera's
RTSP stream locally, and forwards it over one outbound TLS connection on port 443.

The source repository is private. This repository holds only the audited, secret-free,
cosign-signed agent binaries and installers published by its CI.

The machine must not be the NVR itself, needs outbound TCP 443, a synchronised clock, and the
cameras' RTSP reachable on its LAN.

---

## Install

Both installers refuse to install unless the release signature verifies, so install cosign first.

Linux (amd64, arm64; systemd):

```sh
curl -fsSL https://github.com/Digital-Dynamics-Corporation/CameraBridge-dist/releases/latest/download/install.sh | sudo sh
```

Windows (amd64), from an elevated PowerShell after `winget install sigstore.cosign`:

```powershell
irm https://github.com/Digital-Dynamics-Corporation/CameraBridge-dist/releases/latest/download/install.ps1 | iex
```

Linux installs `/usr/local/bin/camerabridge-agent`, a `camerabridge` system user, and the
`camerabridge-agent` systemd unit. Windows installs `C:\Program Files\CameraBridge`, a data
directory `C:\ProgramData\CameraBridge` readable only by SYSTEM, Administrators and the service
account, and the `camerabridge-agent` Windows service running as `NT SERVICE\camerabridge-agent`.
Neither starts the service.

```sh
camerabridge-agent version
```

---

## Enroll a site

Your CameraBridge operator enrolls each site. Generate the site's key on the machine and send the
operator **only the public key**; the private key never leaves the machine.

Linux:

```sh
sudo -u camerabridge camerabridge-agent keygen -key /var/lib/camerabridge/site.key > site.pub
```

Windows:

```powershell
& "C:\Program Files\CameraBridge\camerabridge-agent.exe" keygen -key C:\ProgramData\CameraBridge\site.key | Out-File -Encoding ascii site.pub
```

The operator replies with the configuration for `/etc/camerabridge/agent.env` (Linux, root, mode
0600) or `C:\ProgramData\CameraBridge\agent.env` (Windows). It includes your camera URLs, which
carry camera passwords, so keep that file where the installer put it. The agent refuses a key or a
configuration file that anyone else can read.

## Start

```sh
sudo systemctl enable --now camerabridge-agent
journalctl -u camerabridge-agent -f
```

```powershell
Start-Service camerabridge-agent
Get-WinEvent -ProviderName camerabridge-agent -MaxEvents 30 | Format-List TimeCreated,Message
```

A configuration or identity error (exit 2) stops the service and it stays stopped until fixed;
other failures restart it after 10 seconds, at most five times in ten minutes.

---

## Update

Re-run the installer. It replaces the binary and restarts the service only if it was running.
Pin a version with `sh -s -- --version vX.Y.Z` (Linux) or `$env:CAMERABRIDGE_VERSION = "vX.Y.Z"`
before `irm ... | iex` (Windows).

---

## Verify (supply chain)

Every release ships `SHA256SUMS`, `SHA256SUMS.sig` and `SHA256SUMS.pem` (Sigstore keyless) and a
CycloneDX SBOM. The installers do this for you:

```sh
cosign verify-blob SHA256SUMS --signature SHA256SUMS.sig --certificate SHA256SUMS.pem \
  --certificate-identity-regexp '^https://github\.com/Digital-Dynamics-Corporation/StreamGuard/\.github/workflows/camerabridge-release\.yml@refs/tags/camerabridge-agent-v' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
sha256sum -c SHA256SUMS --ignore-missing
```

## Troubleshooting

- **Windows SmartScreen warns about the exe:** it is not Authenticode-signed yet; the installer
  clears Mark-of-the-Web after the signature and checksum verify.
- **`accessTokenType must be JWT` or `Errors.User.NotActive`:** the site's identity was created
  wrongly or revoked; contact your operator.
- **`auth-failed` for a camera:** wrong camera password. The agent waits 5 minutes between attempts
  so the camera does not lock the account.
- **Laptops:** keep the machine awake on mains power, or the cameras drop when it sleeps.
