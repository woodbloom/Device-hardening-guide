<div align="center">

<img width="100%" src="https://capsule-render.vercel.app/api?type=waving&color=gradient&customColorList=2,3,6&height=140&section=header&text=Device+Hardening+Guide&fontSize=40&fontColor=ffffff&animation=fadeIn&fontAlignY=45&desc=Reduce+tracking%2C+linkability+%26+digital+exposure+on+desktop&descAlignY=65&descSize=16&descColor=00b894"/>

[![Typing SVG](https://readme-typing-svg.demolab.com?font=JetBrains+Mono&weight=600&size=16&duration=2500&pause=1000&color=00B894&center=true&vCenter=true&width=700&lines=VPN+%2B+Kill+Switch+%2B+WireGuard;DNS-over-HTTPS+%2B+Encrypted+Resolvers;Browser+Hardening+%2B+Anti-Fingerprinting;Identity+Separation+%2B+Compartmentalization;Disk+Encryption+%2B+Firewall+%2B+Minimal+Install)](https://github.com/cordinsanity/Device-hardening-guide)

</div>

> **Threat model:** Reduce passive tracking, advertising profiling, behavioral linkability, and ISP surveillance.
> This guide is **not** designed for nation-state adversaries or physical seizure scenarios — see [Qubes OS](https://www.qubes-os.org/) for that level.

---

## Table of Contents

- [Recommended Operating Systems](#-recommended-operating-systems)
- [VPN](#-vpn)
- [DNS](#-dns)
- [Firewall](#-firewall)
- [Disk Encryption](#-disk-encryption)
- [Browser Hardening](#-browser-hardening)
- [Extensions](#-extensions)
- [Email Privacy](#-email-privacy)
- [Password Management](#-password-management)
- [Two-Factor Authentication](#-two-factor-authentication)
- [Identity & Profile Separation](#-identity--profile-separation)
- [Virtualization & Sandboxing](#-virtualization--sandboxing)
- [Metadata Hygiene](#-metadata-hygiene)
- [Secure Communication](#-secure-communication)
- [System Hardening Checklist](#-system-hardening-checklist)
- [Audit Tools](#-audit-tools)
- [Common Failure Points](#-common-failure-points)
- [Quick Reference](#-quick-reference)

---

## 🖥️ Recommended Operating Systems

| OS | Security Level | Notes |
|----|---------------|-------|
| **Fedora** | ⭐⭐⭐⭐ | SELinux enforcing by default, fast updates, great balance |
| **Debian Stable** | ⭐⭐⭐⭐ | Rock-solid stability, minimal attack surface |
| **Linux Mint** | ⭐⭐⭐ | Beginner-friendly Debian/Ubuntu base |
| **Arch Linux** | ⭐⭐⭐⭐ | Minimal by default, you control everything |
| **Qubes OS** | ⭐⭐⭐⭐⭐ | Maximum isolation via VMs — for advanced users |
| **Tails** | ⭐⭐⭐⭐⭐ | Amnesic live USB, routes everything through Tor |
| **Windows 11** | ⭐⭐ | Heavy telemetry — harden aggressively or avoid |
| **macOS** | ⭐⭐⭐ | Better than Windows but closed-source, iCloud risks |

### Windows — Minimum Hardening (if you must use it)

```powershell
# Disable telemetry via registry
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v AllowTelemetry /t REG_DWORD /d 0 /f

# Disable Cortana
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Search" /v AllowCortana /t REG_DWORD /d 0 /f

# Disable advertising ID
reg add "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\AdvertisingInfo" /v Enabled /t REG_DWORD /d 0 /f
```

Tools: [O&O ShutUp10++](https://www.oo-software.com/en/shutup10) · [Sophia Script](https://github.com/farag2/Sophia-Script-for-Windows) · [WindowsSpyBlocker](https://github.com/crazy-max/WindowsSpyBlocker)

---

## 🔒 VPN

A VPN hides your IP from websites and encrypts traffic from your ISP. It does **not** make you anonymous.

### Provider Selection Criteria

| Criterion | What to look for |
|-----------|-----------------|
| **No-log policy** | Independently audited — not just claimed |
| **Jurisdiction** | Outside 5/9/14 Eyes if possible |
| **Protocol** | WireGuard preferred (faster, smaller attack surface) |
| **Payment** | Cash, Monero, or crypto accepted |
| **Kill switch** | Mandatory — must block all traffic on VPN drop |
| **IPv6** | Must be fully tunneled or disabled |

### Recommended Providers

| Provider | Jurisdiction | Payment | Notes |
|----------|-------------|---------|-------|
| **Mullvad** | Sweden | Cash, Monero, crypto | No account email required |
| **ProtonVPN** | Switzerland | Card, crypto | Free tier available |
| **IVPN** | Gibraltar | Cash, Monero | Minimal logging architecture |

### WireGuard Setup (Linux)

```bash
# Install WireGuard
sudo apt install wireguard   # Debian/Ubuntu
sudo dnf install wireguard-tools  # Fedora

# Generate keys
wg genkey | tee private.key | wg pubkey > public.key

# /etc/wireguard/wg0.conf
[Interface]
Address = 10.2.0.2/32
PrivateKey = <your_private_key>
DNS = 10.64.0.1
PostUp = iptables -I OUTPUT ! -o wg0 -m mark ! --mark $(wg show wg0 fwmark) -j REJECT
PreDown = iptables -D OUTPUT ! -o wg0 -m mark ! --mark $(wg show wg0 fwmark) -j REJECT

[Peer]
PublicKey = <server_public_key>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = vpn.example.com:51820
PersistentKeepalive = 25
```

```bash
# Enable and start
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0

# Verify — your IP should be the VPN exit IP
curl ifconfig.me
```

### Kill Switch (iptables)

```bash
# Block all non-VPN traffic
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
iptables -A INPUT -i wg0 -j ACCEPT
iptables -A OUTPUT -o wg0 -j ACCEPT
# Allow VPN handshake only
iptables -A OUTPUT -p udp --dport 51820 -j ACCEPT
```

### What a VPN Does and Doesn't Protect Against

| Threat | Protected? |
|--------|-----------|
| ISP seeing your traffic | ✅ Yes |
| IP-based geolocation | ✅ Yes |
| DNS leaks (if DoH/DoT configured) | ✅ Yes |
| Browser fingerprinting | ❌ No |
| Logged-in account tracking (Google, Meta) | ❌ No |
| Behavioral / cross-site tracking | ❌ No |
| Malware | ❌ No |
| VPN provider logging your traffic | ❌ No (trust-based) |

### Check for Leaks

```bash
# DNS leak test
curl https://dnsleaktest.com/results.html

# IPv6 leak check
curl -6 ifconfig.me  # Should fail or show VPN IPv6

# WebRTC leak — use browser: https://browserleaks.com/webrtc
```

---

## 🌐 DNS

Default DNS is unencrypted and sends every domain you visit to your ISP. Fix this.

### Options

| Protocol | Port | Encryption | Notes |
|----------|------|-----------|-------|
| **DoH** (DNS-over-HTTPS) | 443 | ✅ TLS | Looks like regular web traffic |
| **DoT** (DNS-over-TLS) | 853 | ✅ TLS | Dedicated port, easier to block |
| **DNSCrypt** | Variable | ✅ DNSCrypt | Authenticated resolver identity |

### Recommended Resolvers

| Resolver | DoH URL | Notes |
|----------|---------|-------|
| **Mullvad** | `https://dns.mullvad.net/dns-query` | No logging, blocks ads/trackers |
| **NextDNS** | `https://dns.nextdns.io/YOUR_ID` | Custom blocklists, per-device |
| **Cloudflare** | `https://cloudflare-dns.com/dns-query` | Fast, US-based, some logging |
| **AdGuard** | `https://dns.adguard.com/dns-query` | Ad/tracker blocking |

### Linux — systemd-resolved

```bash
# /etc/systemd/resolved.conf
[Resolve]
DNS=194.242.2.2#dns.mullvad.net
FallbackDNS=1.1.1.1#cloudflare-dns.com
DNSOverTLS=yes
DNSSEC=yes
MulticastDNS=no
LLMNR=no
```

```bash
sudo systemctl restart systemd-resolved
resolvectl status  # Verify protocol shows "yes" for DNS over TLS
```

### Firefox / LibreWolf DoH

```
about:preferences → General → Network Settings → Enable DNS over HTTPS
Provider: Custom → https://dns.mullvad.net/dns-query
```

### Verify DNS Is Encrypted

```bash
# Check which resolver is actually being used
resolvectl query github.com

# Confirm no plain DNS on port 53 leaking
sudo tcpdump -i any port 53
# Should show zero traffic while browsing
```

---

## 🔥 Firewall

### Linux — ufw (simple)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
sudo ufw status verbose
```

### Linux — firewalld (Fedora/RHEL)

```bash
# Set default zone to drop
sudo firewall-cmd --set-default-zone=drop

# Allow only specific services
sudo firewall-cmd --zone=drop --add-service=dhcpv6-client --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

### Check Open Ports

```bash
ss -tulnp           # All listening sockets with process names
sudo netstat -tlnp  # Alternative
sudo lsof -i -P -n  # All open connections
```

### Block Outbound Telemetry (hosts file)

```bash
# /etc/hosts — add these to block common telemetry
0.0.0.0 telemetry.microsoft.com
0.0.0.0 vortex.data.microsoft.com
0.0.0.0 data.microsoft.com
0.0.0.0 browser.events.data.microsoft.com
0.0.0.0 stats.adobe.com
0.0.0.0 metrics.apple.com
```

---

## 💾 Disk Encryption

Full disk encryption protects your data if the device is stolen or seized.

### Linux — LUKS (recommended)

Most Linux installers offer LUKS encryption during installation. **Enable it there** — it's much easier than after-the-fact.

```bash
# Check if your disk is encrypted
sudo dmsetup status
sudo cryptsetup luksDump /dev/sda

# Create an encrypted partition manually
sudo cryptsetup luksFormat --type luks2 --cipher aes-xts-plain64 --key-size 512 --hash sha512 /dev/sdX

# Open / mount
sudo cryptsetup open /dev/sdX cryptdisk
sudo mount /dev/mapper/cryptdisk /mnt
```

### Strong Passphrase Rules

- Minimum 6 random words (diceware) or 20+ character random string
- Store a recovery key offline on paper in a physically secure location
- Do **not** use the same passphrase as your user login

### Swap Encryption

Unencrypted swap can leak sensitive data.

```bash
# /etc/crypttab — encrypt swap
swap  /dev/sdX  /dev/urandom  swap,cipher=aes-xts-plain64,size=256

# Verify
sudo swapon --show
```

### macOS

```
System Settings → Privacy & Security → FileVault → Turn On
```
Store the recovery key locally or with iCloud — **not** iCloud if you don't trust Apple.

### Windows — BitLocker / VeraCrypt

- **BitLocker** (Pro/Enterprise only): stores key in TPM by default — not ideal without PIN
- **VeraCrypt**: open-source, cross-platform, recommended for sensitive containers

```
VeraCrypt → Create Volume → Encrypt system partition
Algorithm: AES · Hash: SHA-512 · PIM: set a value higher than default
```

---

## 🌍 Browser Hardening

### Recommended Browsers (Ranked)

| Browser | Anti-Fingerprinting | Notes |
|---------|-------------------|-------|
| **Mullvad Browser** | ✅ Excellent | Tor Browser network without Tor, all users share one fingerprint |
| **Tor Browser** | ✅ Excellent | Full anonymity, slow — use for sensitive lookups |
| **LibreWolf** | ✅ Good | Hardened Firefox fork, no telemetry, pre-configured |
| **Firefox ESR** | ⚠️ Needs config | Good base, requires manual hardening (see below) |
| **Brave** | ✅ Good | Built-in shields, Chromium base — avoid sync features |
| **Chrome/Edge** | ❌ Avoid | Heavy telemetry, fingerprint-friendly by design |

### Firefox / LibreWolf `about:config` Hardening

Open `about:config` and set the following:

```
// Anti-fingerprinting
privacy.resistFingerprinting = true
privacy.resistFingerprinting.letterboxing = true
privacy.fingerprintingProtection = true

// Tracking protection
privacy.trackingprotection.enabled = true
privacy.trackingprotection.socialtracking.enabled = true
privacy.trackingprotection.cryptomining.enabled = true
privacy.trackingprotection.fingerprinting.enabled = true

// First-party isolation
privacy.firstparty.isolate = true

// WebRTC (prevent IP leak)
media.peerconnection.enabled = false
media.peerconnection.ice.default_address_only = true
media.peerconnection.ice.no_host = true

// Geolocation
geo.enabled = false
geo.provider.use_corelocation = false

// Sensors
dom.battery.enabled = false
device.sensors.enabled = false
dom.gamepad.enabled = false
dom.vr.enabled = false

// Media / microphone / camera
media.navigator.enabled = false
media.autoplay.default = 5
permissions.default.camera = 2
permissions.default.microphone = 2

// Telemetry (Firefox only — already off in LibreWolf)
toolkit.telemetry.enabled = false
toolkit.telemetry.unified = false
toolkit.telemetry.server = ""
datareporting.healthreport.uploadEnabled = false
datareporting.policy.dataSubmissionEnabled = false
app.shield.optoutstudies.enabled = false
browser.discovery.enabled = false

// Prefetch / cache
network.prefetch-next = false
network.dns.disablePrefetch = true
network.http.speculative-parallel-limit = 0

// Referrer
network.http.referer.XOriginPolicy = 2
network.http.referer.XOriginTrimmingPolicy = 2
network.http.referer.trimmingPolicy = 2

// Cookies
network.cookie.cookieBehavior = 1
network.cookie.thirdparty.sessionOnly = true
network.cookie.thirdparty.nonsecureSessionOnly = true

// HTTPS
dom.security.https_only_mode = true
dom.security.https_only_mode_ever_enabled = true

// Search
browser.search.suggest.enabled = false
browser.urlbar.suggest.searches = false
browser.urlbar.speculativeConnect.enabled = false

// Safe browsing (sends URLs to Google)
browser.safebrowsing.malware.enabled = false
browser.safebrowsing.phishing.enabled = false
browser.safebrowsing.downloads.enabled = false
```

### Test Your Browser

| Tool | What it checks |
|------|---------------|
| [coveryourtracks.eff.org](https://coveryourtracks.eff.org) | Fingerprint uniqueness |
| [browserleaks.com](https://browserleaks.com) | Full leak suite |
| [dnsleaktest.com](https://dnsleaktest.com) | DNS resolver |
| [ipleak.net](https://ipleak.net) | IP, DNS, WebRTC |
| [amiunique.org](https://amiunique.org) | Fingerprint database check |

---

## 🧩 Extensions

> **Rule: fewer extensions = smaller fingerprint surface.** Every extension makes your browser more unique.

| Extension | Purpose | Priority |
|-----------|---------|---------|
| **uBlock Origin** | Ad/tracker blocking (use medium or hard mode) | 🔴 Essential |
| **ClearURLs** | Strip tracking parameters from URLs | 🟡 Recommended |
| **Cookie AutoDelete** | Auto-delete cookies when tab closes | 🟡 Recommended |
| **Temporary Containers** (Firefox) | Each site in an isolated container | 🟡 Recommended |
| **LocalCDN** | Serve common JS libraries locally (prevents CDN tracking) | 🟢 Optional |
| **Firefox Multi-Account Containers** | Manual identity separation | 🟢 Optional |

### uBlock Origin — Medium Mode Setup

```
Settings → Filter lists → Enable:
- uBlock filters (all)
- EasyList
- EasyPrivacy
- Peter Lowe's Ad and tracking server list
- AdGuard Annoyances
- Fanboy Annoyances

Dashboard → My filters → Add:
||googletagmanager.com^
||google-analytics.com^
||doubleclick.net^
||facebook.net^$third-party
||connect.facebook.net^
```

### Do NOT Install

- Password managers as browser extensions — use standalone apps
- VPN browser extensions — use system-level VPN
- Multiple overlapping ad blockers
- Any extension requesting "access to all websites" unless you understand exactly what it does

---

## 📧 Email Privacy

Email is inherently insecure for sensitive content. For regular use, choose a privacy-respecting provider.

### Recommended Providers

| Provider | Jurisdiction | E2EE | Notes |
|----------|-------------|------|-------|
| **ProtonMail** | Switzerland | ✅ PGP | Free tier, strong reputation |
| **Tutanota** | Germany | ✅ Proprietary | Clean UI, free tier |
| **Disroot** | Netherlands | ⚠️ Optional | Community-run, donate |
| **SimpleLogin** | France/Switzerland | ✅ Aliases | Email alias service, pairs with any provider |
| **AnonAddy** | UK | ✅ Aliases | Open-source alias service |

### PGP Email Encryption

```bash
# Generate key
gpg --full-gen-key
# Choose: RSA 4096-bit, no expiry (or set one)

# Export public key to share with contacts
gpg --armor --export your@email.com > public.asc

# Encrypt a file/message for someone
gpg --encrypt --armor --recipient their@email.com message.txt

# Decrypt
gpg --decrypt message.asc
```

### Rules

- Use **email aliases** (SimpleLogin / AnonAddy) when signing up for services — never your real address
- Use a separate email address per major service category (shopping, social, finance, etc.)
- Never open attachments from unknown senders
- Disable remote image loading (these can track when you open an email)

---

## 🔑 Password Management

| Tool | Type | Notes |
|------|------|-------|
| **Bitwarden** | Cloud-synced, open-source | Free, self-hostable, audited |
| **KeePassXC** | Local file-based | Fully offline, great for advanced users |
| **Vaultwarden** | Self-hosted Bitwarden | Full control of your vault |

### KeePassXC Hardening

```
Settings → Security:
- Lock database after 5 minutes of inactivity
- Clear clipboard after 30 seconds
- Require password + YubiKey for unlock
- Enable database encryption: AES256 + ChaCha20

Database → Encryption Settings:
- Algorithm: ChaCha20
- KDF: Argon2id
- Memory: 64 MB · Iterations: 12 · Parallelism: 2
```

### Password Rules

- **Unique password for every account** — no exceptions
- Minimum 20 characters for important accounts, 16 for others
- Use the password manager's built-in generator
- Enable breach monitoring (Bitwarden alerts when a password appears in a data breach)

---

## 🔐 Two-Factor Authentication

| Type | Security | Notes |
|------|---------|-------|
| **Hardware key** (YubiKey, Nitrokey) | ✅✅✅ Best | Phishing-resistant, physical |
| **TOTP app** (Aegis, Raivo, KeePassXC) | ✅✅ Good | Avoid Google/Microsoft Authenticator |
| **SMS** | ⚠️ Weak | SIM swap attacks, intercept possible |
| **Email code** | ⚠️ Weak | As secure as your email |

### Setup Priority

1. Email account → hardware key or TOTP
2. Password manager → hardware key or TOTP
3. GitHub, financial, VPN accounts → hardware key or TOTP
4. Everything else → TOTP minimum

### YubiKey Usage

```bash
# Install tools
sudo apt install yubikey-manager

# List connected YubiKeys
ykman list

# Set PIN for FIDO2
ykman fido access change-pin

# Generate a resident passkey
ykman fido credentials list
```

---

## 🪪 Identity & Profile Separation

### Rule: One Identity Per Context

| Context | Browser Profile | VPN | Notes |
|---------|----------------|-----|-------|
| Personal | Profile A | Exit node A | Real name accounts |
| Work | Profile B | Exit node B | Corporate apps |
| Finance | Profile C | Exit node A | Banking, crypto |
| Research / Anonymous | Tor Browser or VM | Tor | No login, fresh state |

### Firefox Multi-Account Containers

```
Install extension → Right-click any link → Open in Container Tab
Containers: Personal / Work / Finance / Shopping / Social
Auto-assign domains: facebook.com → Social container always
```

### Browser Profile Isolation

```bash
# Launch Firefox with a specific profile
firefox -P "Work" --no-remote
firefox -P "Personal" --no-remote
```

### Never Mix

- Logged-in personal accounts in anonymous sessions
- Real name in any research/anonymous context
- Same username across unlinked services
- Personal email for throwaway signups

---

## 🖥️ Virtualization & Sandboxing

### Virtual Machines

```bash
# Install VirtualBox
sudo apt install virtualbox virtualbox-ext-pack

# Or KVM/QEMU (better performance on Linux)
sudo apt install qemu-kvm libvirt-daemon-system virt-manager

# Create an isolated VM for risky tasks
virt-manager  # GUI tool
```

Use cases: opening untrusted files, testing software, anonymous browsing sessions, isolating corporate tools.

### Qubes OS — Maximum Isolation

Qubes OS runs every task in a separate VM:

- **Personal VM** — personal browsing/files
- **Work VM** — work files, never mixed with personal
- **Untrusted VM** — opening suspicious files/links
- **Vault VM** — password manager, offline, no network
- **Whonix VM** — anonymous internet via Tor

### Firejail — Application Sandboxing (Linux)

```bash
sudo apt install firejail

# Run a browser sandboxed
firejail --private firefox
firejail --private --net=none libreoffice suspicious.docx

# Sandbox specific app permanently
sudo firejail --profile=/etc/firejail/firefox.profile firefox
```

---

## 🗂️ Metadata Hygiene

### Images — Strip EXIF

```bash
# Check what metadata exists
exiftool photo.jpg
exiftool photo.jpg | grep -i "gps\|location\|device"

# Strip all metadata
exiftool -all= photo.jpg
exiftool -all= -r ./photos/   # Recursive

# Or use mat2
mat2 photo.jpg
mat2 --inplace photo.jpg
```

### Documents

```bash
# PDF metadata
mat2 document.pdf
exiftool -all= document.pdf

# LibreOffice
File → Properties → Remove Personal Information button

# DOCX (zip internally)
mat2 document.docx
```

### Audio / Video

```bash
# Strip all tags from audio
ffmpeg -i input.mp3 -map_metadata -1 -c:a copy output.mp3

# Strip from video
ffmpeg -i input.mp4 -map_metadata -1 -c copy output.mp4
```

### Rules

- Never share photos from sensitive locations without stripping GPS data first
- Re-encoding images can reintroduce metadata — always check after
- Screenshots contain no EXIF but can contain information in pixels (location visible in image, timestamps in UI)

---

## 💬 Secure Communication

| Tool | Use Case | Metadata | Notes |
|------|----------|---------|-------|
| **Signal** | Trusted contacts, voice, video | Minimal | Enable disappearing messages |
| **Session** | Anonymous contacts | None | No phone number, no metadata |
| **SimpleX** | Maximum anonymity | None | No user IDs at all |
| **Matrix/Element** | Groups, self-hosted | Variable | Trust depends on server |
| **Briar** | Offline/Tor mesh | Minimal | Bluetooth/WiFi capable |

### Signal Hardening

```
Settings → Privacy:
- Screen lock: ON
- Screen security (hide content in app switcher): ON
- Incognito keyboard: ON
- Read receipts: OFF
- Typing indicators: OFF
- Link previews: OFF
- Note to self: Delete after 1 week

Settings → Chats:
- Default timer: 1 week or less for sensitive contacts
```

### Rules

- Use **disappearing messages** by default — always
- Do not sync your phonebook to any app (use manual contact entry)
- Never discuss sensitive topics in SMS or standard phone calls
- Keep messaging apps separate per identity if possible

---

## ✅ System Hardening Checklist

### Linux

```bash
# Automatic security updates
sudo apt install unattended-upgrades  # Debian
sudo dnf install dnf-automatic        # Fedora

# Check for rootkits
sudo apt install rkhunter chkrootkit
sudo rkhunter --update && sudo rkhunter --check
sudo chkrootkit

# AppArmor (Ubuntu/Debian) or SELinux (Fedora)
sudo aa-status              # AppArmor
getenforce                  # SELinux — should be "Enforcing"

# Audit SSH
cat /etc/ssh/sshd_config | grep -E "PermitRootLogin|PasswordAuthentication|X11Forwarding"
# Should be: PermitRootLogin no | PasswordAuthentication no | X11Forwarding no

# Disable unused services
sudo systemctl list-units --type=service --state=running
sudo systemctl disable bluetooth  # If not needed
sudo systemctl disable cups       # Printing daemon if unused
sudo systemctl disable avahi-daemon  # mDNS — disable if not using local network discovery
```

- [x] Full disk encryption enabled (LUKS / BitLocker / FileVault)
- [x] Strong BIOS/UEFI password set
- [x] Secure Boot enabled
- [x] Firewall active with default deny incoming
- [x] Automatic security updates enabled
- [x] SSH: root login disabled, password auth off, key-only
- [x] Screen lock after ≤ 5 minutes
- [x] Minimal software installed
- [x] No unnecessary services running
- [x] AppArmor / SELinux enforcing
- [x] WireGuard VPN with kill switch
- [x] DNS-over-HTTPS configured at system level
- [x] Browser hardened (resistFingerprinting, no WebRTC leaks)
- [x] Password manager in use, unique passwords everywhere
- [x] Hardware 2FA on critical accounts
- [x] Identity separation per context

---

## 🔍 Audit Tools

| Tool | What it does |
|------|-------------|
| `ss -tulnp` | Show all open ports and listening services |
| `rkhunter` | Scan for rootkits and suspicious files |
| `chkrootkit` | Second rootkit scanner |
| `lynis` | System security audit |
| `nmap -sV localhost` | Port scan your own machine |
| `auditd` | Linux system call auditing |
| `fail2ban` | Auto-ban IPs after failed login attempts |
| `Wireshark / tcpdump` | Capture and inspect network traffic |
| `netstat -tulnp` | Legacy port listing |

```bash
# Full system audit with lynis
sudo apt install lynis
sudo lynis audit system

# Monitor outbound connections in real-time
sudo tcpdump -i eth0 -n

# Check for unexpected cron jobs
crontab -l
ls /etc/cron.*
```

---

## ⚠️ Common Failure Points

Most privacy failures are **behavioral**, not technical:

| Failure | Fix |
|---------|-----|
| Reusing usernames across services | Use unique names per context, password manager |
| Mixing identities in one browser session | Separate profiles or containers |
| Logging into personal accounts while "anonymous" | Hard separation — different browser/VM |
| Cloud backup of sensitive files | Local or encrypted backup only (Cryptomator) |
| Browser extensions with broad permissions | Minimize — each extension increases fingerprint |
| Skipping security updates | Enable automatic updates |
| Using SMS for 2FA | Switch to TOTP app or hardware key |
| Sharing files without stripping metadata | Always run `mat2` or `exiftool -all=` first |
| Using the same email for all signups | Use aliases (SimpleLogin / AnonAddy) |
| Trusting VPN without leak testing | Test on every new config — dnsleaktest.com |
| Saving passwords in the browser | Use a dedicated password manager |

---

## ⚡ Quick Reference

```
VPN         →  WireGuard · Kill switch · Mullvad / ProtonVPN / IVPN · Leak test
DNS         →  DoH/DoT · Mullvad or NextDNS · system-level, not just browser
Firewall    →  ufw default deny · check open ports weekly
Disk        →  LUKS (Linux) · BitLocker+PIN (Windows) · FileVault (Mac)
Browser     →  Mullvad Browser or LibreWolf · resistFingerprinting · no WebRTC
Extensions  →  uBlock Origin · ClearURLs · Temporary Containers · that's it
Email       →  ProtonMail / Tutanota · SimpleLogin aliases per service
Passwords   →  KeePassXC or Bitwarden · unique everywhere · 20+ chars
2FA         →  YubiKey > TOTP app > anything else · never SMS
Identity    →  One context per profile · never mix · Tor for anonymous
Metadata    →  exiftool / mat2 before sharing any file
Comms       →  Signal · Session · SimpleX
```

---

<div align="center">

> **Privacy is not hiding.**
> It's reducing the surface area of what can be linked, tracked, and sold.

<br/>

<img width="100%" src="https://capsule-render.vercel.app/api?type=waving&color=gradient&customColorList=2,3,6&height=100&section=footer"/>

**[↑ Back to top](#-recommended-operating-systems)** · [Phone Hardening Guide](https://github.com/cordinsanity/Phone-hardening-guide) · [Revenge Plugins](https://github.com/cordinsanity/revenge-plugins)

</div>
