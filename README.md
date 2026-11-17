# VeilRoute Beta 1.2

**A TLS-protected TCP relay with a local SOCKS5 proxy for Windows clients.**

VeilRoute is a lightweight proxy relay for applications that support SOCKS5. The Windows client connects to a VeilRoute server over TLS/TCP and exposes a local SOCKS5 endpoint at `127.0.0.1:1080`.

> **Project status: Beta**
>
> VeilRoute Beta 1.2 is a TCP-based proxy relay, not a full system-wide VPN. UDP traffic, system-wide DNS routing, and all-device tunneling are not provided by the operational relay in this release. Wintun/TUN components are experimental and are not connected to the active relay path. This release has not undergone an independent security audit and should not be used for sensitive or production-critical workloads without further review.

## Contents

- [How it works](#how-it-works)
- [Features and limitations](#features-and-limitations)
- [Requirements](#requirements)
- [Repository layout](#repository-layout)
- [1. Prepare the project](#1-prepare-the-project)
- [2. Copy the server files to Ubuntu](#2-copy-the-server-files-to-ubuntu)
- [3. Install the server](#3-install-the-server)
- [4. Configure the firewall](#4-configure-the-firewall)
- [5. Verify the server](#5-verify-the-server)
- [6. Get the TLS certificate fingerprint](#6-get-the-tls-certificate-fingerprint)
- [7. Retrieve the one-time pairing code](#7-retrieve-the-one-time-pairing-code)
- [8. Install the Windows client](#8-install-the-windows-client)
- [9. Pair the client with the server](#9-pair-the-client-with-the-server)
- [10. Start the local SOCKS5 proxy](#10-start-the-local-socks5-proxy)
- [11. Test the connection](#11-test-the-connection)
- [Server administration](#server-administration)
- [Troubleshooting](#troubleshooting)
- [Security recommendations](#security-recommendations)
- [Default ports](#default-ports)

## How it works

```text
Windows application
       |
       | SOCKS5
       v
127.0.0.1:1080
       |
       | VeilRoute client
       | TLS over TCP
       v
Ubuntu VPS:8443
       |
       v
Remote network
```

The client must remain running while the local proxy is in use. Configure individual applications to use this SOCKS5 endpoint:

- **Proxy type:** SOCKS5
- **Host:** `127.0.0.1`
- **Port:** `1080`

## Features and limitations

### Included in Beta 1.2

- TLS-protected TCP connection between the client and server.
- One-time device pairing.
- Local SOCKS5 proxy on Windows.
- Local storage of paired client credentials.
- Ubuntu server installation through the provided installation script.
- A `systemd` service for server process management.

### Not provided by the active relay in this release

- Full-device or system-wide VPN tunneling.
- UDP relay.
- System-wide DNS routing.
- Operational Wintun/TUN integration.
- A guarantee of anonymity, censorship resistance, or uninterrupted availability.

## Requirements

### Server

- An Ubuntu VPS.
- SSH access with a user that can run `sudo`.
- A public IPv4 address or a domain pointing to the VPS.
- Permission to configure the VPS firewall or cloud security group.
- An available TCP port. This guide uses `8443`.

### Windows client

- Windows 10 or Windows 11.
- Python 3.10 or newer.
- The `VeilRoute-Beta-1.2` project archive.
- The SSH private key used for server administration, if key-based SSH authentication is enabled.

## Repository layout

The project archive should contain the client package, server installer, documentation, tests, and any included SDK files. The expected top-level layout is similar to:

```text
VeilRoute-Beta-1.2/
├── README.md
├── pyproject.toml
├── veilroute/
├── server/
│   ├── pyproject.toml
│   └── install.sh
├── tests/
├── docs/
└── wintun-sdk/
```

The exact contents may vary by packaged build. The server installation commands below assume that `server/install.sh` exists.

## 1. Prepare the project

1. Download and extract the project archive.
2. Locate the extracted directory. This guide uses:

```text
C:\VeilRoute-Beta-1.2
```

3. Open PowerShell and change to the project directory:

```powershell
Set-Location C:\VeilRoute-Beta-1.2
```

## 2. Copy the server files to Ubuntu

Create the destination directory on the VPS:

```powershell
ssh -i "$env:USERPROFILE\Downloads\YOUR-KEY.pem" ubuntu@YOUR_SERVER_IP "mkdir -p /home/ubuntu/VeilRoute-Beta-1.2"
```

Copy the local `server` directory to the VPS:

```powershell
scp -i "$env:USERPROFILE\Downloads\YOUR-KEY.pem" -r .\server ubuntu@YOUR_SERVER_IP:/home/ubuntu/VeilRoute-Beta-1.2/
```

Replace the placeholders:

| Placeholder | Meaning |
|---|---|
| `YOUR-KEY.pem` | Your SSH private-key filename |
| `YOUR_SERVER_IP` | The public IP address of your VPS |
| `ubuntu` | The SSH username; the default Ubuntu image on AWS commonly uses `ubuntu` |

Connect to the VPS:

```powershell
ssh -i "$env:USERPROFILE\Downloads\YOUR-KEY.pem" ubuntu@YOUR_SERVER_IP
```

If your VPS provider uses a different SSH username or key format, use the values supplied by that provider.

## 3. Install the server

After connecting to Ubuntu, run:

```bash
sudo apt-get update
sudo apt-get install -y openssl
```

Ensure the project directory exists and is owned by the SSH user:

```bash
sudo mkdir -p /home/ubuntu/VeilRoute-Beta-1.2
sudo chown -R ubuntu:ubuntu /home/ubuntu/VeilRoute-Beta-1.2
```

Run the installer with the selected server port:

```bash
sudo VEILROUTE_PORT=8443 bash /home/ubuntu/VeilRoute-Beta-1.2/server/install.sh
```

According to the supplied installation workflow, the installer:

- Installs the server under `/opt/veilroute/`.
- Creates the TLS certificate and private key.
- Creates and enables a `systemd` service.
- Attempts to configure UFW for the selected port when UFW is active.

Review the installer before running it, especially on a production server. Run installation scripts only when you trust and understand their contents.

## 4. Configure the firewall

Allow inbound TCP traffic on port `8443` in every firewall layer that applies to your VPS.

### Cloud provider firewall

For an AWS EC2 instance:

1. Open **EC2 → Security Groups**.
2. Select the security group attached to your instance.
3. Open **Inbound rules** and add a rule:

| Setting | Value |
|---|---|
| Type | Custom TCP |
| Protocol | TCP |
| Port range | `8443` |
| Source | Your trusted client IP address, where practical |

Avoid exposing the service to the entire internet unless your use case requires it and you understand the risk.

### UFW on Ubuntu

Check the current status:

```bash
sudo ufw status
```

If UFW is active and the installer has not added the rule, allow the port:

```bash
sudo ufw allow 8443/tcp
```

**Do not remove or block SSH access on TCP port 22 while connected remotely.** Confirm that SSH access remains permitted before changing firewall rules.

## 5. Verify the server

Check the service:

```bash
sudo systemctl status veilroute --no-pager
```

Check whether the service is listening on TCP port `8443`:

```bash
sudo ss -lntp | grep ':8443'
```

View recent logs:

```bash
sudo journalctl -u veilroute -n 30 --no-pager
```

The service should report `Active: active (running)`. The listening-port check should show a process bound to the configured port.

## 6. Get the TLS certificate fingerprint

Run this command on the VPS:

```bash
sudo openssl x509 -in /etc/veilroute/server.crt -noout -fingerprint -sha256
```

The output should resemble:

```text
sha256 Fingerprint=AA:BB:CC:DD:...
```

Copy the complete fingerprint value after `=` and enter it in the Windows client when pairing.

The fingerprint is specific to the certificate on that server. Retrieve it from the same VPS to which the client will connect. **Do not disable certificate-fingerprint verification to work around a mismatch.**

## 7. Retrieve the one-time pairing code

On the VPS, run:

```bash
sudo journalctl -u veilroute -n 100 --no-pager | grep 'CODE PAIRING ONE-TIME'
```

Use the pairing code only for the intended initial device pairing.

- Treat the code as a secret.
- Do not publish it in screenshots, issue reports, chat groups, or GitHub.
- The code is intended for one-time use.
- Restarting the service does **not** necessarily generate a new pairing code.
- If the code is consumed or exposed, use the project's supported pairing-reset procedure. Do not assume that restarting the service resets pairing state.

## 8. Install the Windows client

Open PowerShell and change to the project directory:

```powershell
Set-Location C:\VeilRoute-Beta-1.2
```

Create a Python virtual environment:

```powershell
py -3 -m venv .venv
```

Activate it:

```powershell
.\.venv\Scripts\Activate.ps1
```

Install the project:

```powershell
python -m pip install --upgrade pip
python -m pip install -e .
```

If PowerShell blocks virtual-environment activation, allow it for the current PowerShell process only:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

Then activate the environment again:

```powershell
.\.venv\Scripts\Activate.ps1
```

Start the client:

```powershell
python -m veilroute
```

You can also run it without activating the environment, after installation has completed:

```powershell
.\.venv\Scripts\python.exe -m veilroute
```

## 9. Pair the client with the server

In the VeilRoute client menu, select:

```text
2 - Pair this device with a one-time code
```

Enter the server details when prompted:

| Prompt | Value |
|---|---|
| Server | `YOUR_SERVER_IP` or your server domain |
| Port | `8443` |
| SHA-256 fingerprint | The fingerprint obtained in Step 6 |
| Pairing code | The one-time code obtained in Step 7 |

After successful pairing, the client should confirm that the device credential has been stored locally.

The client configuration is stored at:

```text
%USERPROFILE%\.veilroute\client.json
```

This file contains sensitive device credentials. Keep it private and never commit it to Git.

## 10. Start the local SOCKS5 proxy

After pairing, select the client menu option:

```text
1 - Start local SOCKS5 proxy
```

The default local endpoint is:

| Setting | Value |
|---|---|
| Proxy type | SOCKS5 |
| Host | `127.0.0.1` |
| Port | `1080` |

Configure each supported application to use this SOCKS5 endpoint. Keep the VeilRoute client running while you use the proxy.

Do not configure the local proxy to listen on a public network interface unless you have deliberately designed and secured that setup. The default loopback address `127.0.0.1` limits access to the local machine.

## 11. Test the connection

### Test TCP reachability from Windows

In a separate PowerShell window, run:

```powershell
Test-NetConnection YOUR_SERVER_IP -Port 8443
```

A result containing:

```text
TcpTestSucceeded : True
```

means that a TCP connection to the specified host and port could be established.

**This test does not verify the TLS handshake, VeilRoute pairing, or end-to-end SOCKS5 traffic.**

### Test traffic through the relay

In the VeilRoute client menu, select option `8` to run the built-in HTTP traffic test. If the test succeeds, it can report the observed outbound IP address.

For a complete check, verify that:

1. The server service is running.
2. The client has paired successfully.
3. The local SOCKS5 listener is active on `127.0.0.1:1080`.
4. A SOCKS5-capable application can reach a test destination through the proxy.

## Server administration

Run these commands on the Ubuntu VPS.

| Task | Command |
|---|---|
| Check status | `sudo systemctl status veilroute --no-pager` |
| Start | `sudo systemctl start veilroute` |
| Stop | `sudo systemctl stop veilroute` |
| Restart | `sudo systemctl restart veilroute` |
| Follow logs | `sudo journalctl -u veilroute -f` |
| View recent logs | `sudo journalctl -u veilroute -n 50 --no-pager` |
| Check listening port | `sudo ss -lntp \| grep ':8443'` |

## Troubleshooting

### Pairing timeout

Check the service first:

```bash
sudo systemctl status veilroute --no-pager
sudo ss -lntp | grep ':8443'
```

Then test TCP reachability from Windows:

```powershell
Test-NetConnection YOUR_SERVER_IP -Port 8443
```

Verify all of the following:

- The IP address or domain is correct.
- TCP port `8443` is allowed by the cloud firewall/security group.
- TCP port `8443` is allowed by UFW, if UFW is active.
- The server service is running and listening on the expected port.
- The client and server versions are compatible.
- The pairing code is valid and has not already been used.
- The TLS fingerprint belongs to the server being contacted.

Review the server logs:

```bash
sudo journalctl -u veilroute -n 50 --no-pager
```

### TLS fingerprint mismatch

Retrieve the fingerprint again from the VPS:

```bash
sudo openssl x509 -in /etc/veilroute/server.crt -noout -fingerprint -sha256
```

Compare the full value with the fingerprint entered in the client. Make sure there are no missing characters or extra spaces. Do not turn off fingerprint verification.

### TCP port is reachable, but pairing fails

`Test-NetConnection` checks TCP reachability only. It does not prove that the TLS handshake or the VeilRoute pairing protocol is working.

Check:

```bash
sudo systemctl status veilroute --no-pager
sudo journalctl -u veilroute -n 50 --no-pager
```

Also confirm that the client and server builds are compatible and that the pairing code has not been consumed.

### SOCKS5 proxy is not working

- Complete device pairing before starting the proxy.
- Select **Start local SOCKS5 proxy** in the client.
- Keep the VeilRoute client process running.
- Set the application proxy type to **SOCKS5**, not HTTP.
- Use host `127.0.0.1` and port `1080`.
- Confirm that another process is not already using port `1080`.

## Security recommendations

- Deploy VeilRoute only on servers you own or are explicitly authorized to use.
- Keep SSH private keys, TLS private keys, pairing codes, and device credentials secret.
- Never commit `client.json`, private keys, or server secrets to GitHub.
- Keep the local SOCKS5 listener bound to `127.0.0.1`.
- Restrict inbound server access to trusted client IP addresses where practical.
- Keep SSH access available before changing firewall rules.
- Use the TLS fingerprint from the intended server and never disable verification to bypass a mismatch.
- Review installation scripts before running them with `sudo`.
- Keep the operating system and project dependencies updated.
- Back up configuration and recovery information securely.

## Default ports

| Component | Protocol | Default port |
|---|---|---:|
| VeilRoute server | TCP over TLS | `8443` |
| Local SOCKS5 proxy | SOCKS5 over TCP | `1080` |
| SSH administration | TCP | `22` |

## License

Add the license that applies to this repository before publishing it. Do not claim an open-source license unless you have selected one and have the right to distribute all included components under it.

---

**VeilRoute Beta 1.2**  
Transport: TLS over TCP  
Local proxy: SOCKS5  
Default server port: `8443`  
Default local proxy: `127.0.0.1:1080`
