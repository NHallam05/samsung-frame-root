# Samsung Frame TV Root

Root access for the **Samsung "The Frame" QN55LS03AAFXZA** on Tizen 6.0.

Uses [SVE-2025-50109](https://security.samsungmobile.com/) — an SDB package-name injection vulnerability — combined with a signed-archive privilege escalation chain to achieve `uid=0` root from the local network.

## Tested On

| Model | Firmware | Tizen | Kernel | Status |
|-------|----------|-------|--------|--------|
| QN55LS03AAFXZA | T-NKM2AKUC-REL-202607211843 | 6.0.0 | 4.1.10 armv7l | Working |

Other Frame models on Tizen 6.x with the same SDB daemon should work. If you confirm another model, open an issue or PR.

## How It Works

```
PC (Python) ──TCP 26101──▶ SDB daemon ──injection──▶ uid=901 shell
                                                        │
                          dotnet (CAP_SYS_ADMIN+SETGID) ◀┘
                                  │
                     private mount namespace
                     bind-mount RSA key + passwd
                                  │
                          sdbd-tarlauncher ──▶ uid=0 root
                                  │
                          SdbJob.dll ──▶ run command, serve result
                                  │
PC ◀──SDB tcp:PORT──loopback──────┘  (HMAC-authenticated)
```

**Step by step:**

1. **SDB injection** — Developer Mode exposes SDB on TCP 26101. The `appinstall tpk <name>` handler interpolates the package name into a shell command without quoting. Backtick injection gives code execution as `uid=901(sdk)`.

2. **Mount namespace bootstrap** — The injected shell runs Samsung's signed `/usr/bin/dotnet`, which has `CAP_SYS_ADMIN` and `CAP_SETGID`. A .NET 3.1 bootstrap (`SamsungTvArchiveRoot.dll`) creates a private mount namespace and bind-mounts a per-run RSA public key over `/usr/share/sdbd/public.pem` and a crafted `/etc/passwd` that maps `owner` to UID 0.

3. **Signed tar launcher** — Samsung's signed `/usr/sbin/sdbd-tarlauncher` verifies the archive against the (now-replaced) public key and runs `launch.sh` as UID 0.

4. **Root agent** — `launch.sh` starts `SdbJob.dll` as root. It executes the target command, captures stdout/stderr, and serves the result over a loopback TCP socket. The PC connects through SDB's `tcp:PORT` forwarding, authenticates with a per-job 256-bit HMAC key, and reads the result.

Root is **volatile** — lost on every reboot. The exploit must be re-run each time.

## Prerequisites

- **Python 3.10+** with `cryptography` (`pip install cryptography`)
- **Developer Mode enabled on the TV:**
  1. Smart Hub → Apps → gear icon (App Settings)
  2. Enter `12345` on the remote
  3. Toggle Developer Mode **ON**
  4. Enter your PC's LAN IP address
  5. Reboot the TV
- **Samsung SDB** — install [Tizen Studio](https://developer.tizen.org/development/tizen-studio/download) or just grab `sdb.exe` from the tools package. Only needed to kill its background server before running the exploit.
- **Network access** — your PC must reach the TV on port 26101.

## Quick Start

```bash
# Clone
git clone https://github.com/YOUR_USERNAME/samsung-frame-root.git
cd samsung-frame-root

# Install Python dependency
pip install cryptography

# Kill any sdb background server (it holds an exclusive connection)
sdb kill-server
# Or on Windows: C:\tizen-studio\tools\sdb.exe kill-server

# Set your TV's IP
export TV_HOST=192.168.1.xxx
# Or on Windows: set TV_HOST=192.168.1.xxx

# Probe — verifies the full chain works and you have root
python exploit.py --probe

# Run any command as root
python exploit.py "id"
python exploit.py "cat /etc/shadow | head -3"
python exploit.py "uname -a"

# Longer commands (default timeout is 45s)
python exploit.py --timeout 120 "find /opt -name '*.db' -ls"
```

### Expected probe output

```
SDB handshake + root chain verified on 192.168.1.xxx:26101
Device: uid 0, Tizen "6.0.0 (Tizen6/TV)", kernel 4.1.10, armv7l, whoami=root
Firmware build: T-NKM2AKUC-REL-202607211843
  present: /usr/bin/dotnet
  present: /usr/sbin/sdbd-tarlauncher
  present: /usr/share/sdbd/public.pem
Assessment: the root chain WORKS on this device (uid=0 verified).
```

## TV Internals

### Hardware (QN55LS03AAFXZA)

| Component | Detail |
|-----------|--------|
| SoC | ARM Cortex-A53 (ARMv7), 4 cores @ 1.7 GHz |
| RAM | ~1.8 GB |
| Storage | 15 GB eMMC (mmcblk0), 25 partitions |
| WiFi | 802.11ac (wlan0) |
| Ethernet | eth0 (optional) |

### Key Partitions

| Partition | Size | Mount | Notes |
|-----------|------|-------|-------|
| mmcblk0p18 | 1.7 GB | `/` | Root FS (vdfs, **read-only**) |
| mmcblk0p19 | 1.7 GB | — | Backup root (A/B scheme) |
| mmcblk0p21 | 11 GB | `/opt` | User data (read-write) |
| mmcblk0p17 | 12 MB | `/mnt/drm` | DRM keys |
| mmcblk0p20 | 20 MB | `/mnt/systemrw` | System writable |
| mmcblk0p22 | 12 MB | `/factory_peq` | Factory calibration (squashfs) |
| zram0 | 926 MB | swap | Compressed RAM |

### Key Binaries

| Path | Capabilities | Role |
|------|-------------|------|
| `/usr/bin/dotnet` | `cap_setgid,cap_sys_admin+ei` | Bootstrap (mount namespace) |
| `/usr/sbin/sdbd-tarlauncher` | `cap_setgid,cap_setuid+eip` | Root escalation |
| `/usr/share/sdbd/public.pem` | — | Archive verification key (bind-mounted) |

### .NET Runtimes Available

```
/usr/share/dotnet/shared/Microsoft.NETCore.App/
  2.0.0, 2.1.0, 2.1.1, 2.1.4, 3.0.0, 3.1.3
```

Payloads **must** target `netcoreapp3.1`. The TV has no .NET 6.0+ runtime — .NET 6 DLLs fail with `FileLoadException: Could not load 'System.Runtime, Version=6.0.0.0'`.

### Security

- **SMACK** (Simplified Mandatory Access Control) — SDB user runs as `System` label, root as `User`
- **TrustZone** — active, handles DRM and secure storage
- **Root FS** — read-only vdfs, cannot be persistently modified
- **No UEP** (`/proc/uep` absent) — no kernel-level exploit protection

## File Layout

```
samsung-frame-root/
├── README.md                          # This file
├── exploit.py                         # Main exploit script
├── payloads/
│   ├── SamsungTvArchiveRoot.dll       # Pre-built .NET 3.1 bootstrap
│   ├── SamsungTvArchiveRoot.runtimeconfig.json
│   ├── SdbJob.dll                     # Pre-built .NET 3.1 root agent
│   └── src/
│       ├── SamsungTvArchiveRoot.cs    # Bootstrap source
│       └── SdbJob.cs                  # Root agent source
├── build/                             # Build projects (for rebuilding)
│   ├── ArchiveRoot/
│   │   └── SamsungTvArchiveRoot.csproj
│   └── SdbJob/
│       └── SdbJob.csproj
├── TROUBLESHOOTING.md
├── LICENSE
└── .gitignore
```

## Rebuilding Payloads

If you need to rebuild the .NET DLLs (requires .NET SDK 3.1+ on your PC):

```bash
dotnet build build/ArchiveRoot/SamsungTvArchiveRoot.csproj -c Release
dotnet build build/SdbJob/SdbJob.csproj -c Release

cp build/ArchiveRoot/bin/Release/netcoreapp3.1/SamsungTvArchiveRoot.dll payloads/
cp build/SdbJob/bin/Release/netcoreapp3.1/SdbJob.dll payloads/
```

## After Reboot

Root is volatile — it lives in tmpfs/memory and is lost when the TV powers off or reboots.

To re-acquire root:
1. Wait for the TV to fully boot and reconnect to your network
2. `sdb kill-server` (if sdb is installed)
3. `python exploit.py --probe`
4. Root commands work immediately

## Credits

- Exploit chain based on research from [samsung-tv-root](https://github.com/chris-ritsen/samsung-tv-root) and [s90c-tools](https://github.com/nickel-iaea/s90c-tools)
- SVE-2025-50109 — Samsung SDB `appinstall` command injection
- Archive-root technique — mount namespace + signed tar launcher privilege escalation

## Disclaimer

This tool is for **personal use on your own TV**. Rooting may void your warranty. The root filesystem is read-only, so normal use of this tool cannot brick the TV — but modifying kernel or boot partitions can. Don't touch partitions you don't understand.

## License

MIT
