# OpenWrt 25.12 Firewall Setup Script

`fw-setup.sh` prepares a router running OpenWrt 25.12 as a security platform. It installs VPN and policy-routing packages, moves package storage onto a USB 3.0 drive, and installs ClamAV, Snort 3, mitmproxy and nDPI builds for the router's CPU architecture.

> [!WARNING]
> A fresh installation **erases the selected USB drive**. Resuming an existing USB/extroot installation preserves its layout. The script installs unsigned packages with `apk add --allow-untrusted`. Read this guide before running it.

## Working options for v3.3.5

Use these options together when installing or resuming with the included mitmproxy compatibility bundle:

```sh
-upgrade -with-mitmproxy -mitmproxy-compat -noreboot
```

`-upgrade` enables package upgrades, `-with-mitmproxy` includes mitmproxy, `-mitmproxy-compat` selects its embedded prebuilt dependencies and compatibility launchers, and `-noreboot` keeps the router running after successful stage 2. These settings are saved for the automatic stage 2 run and later resumes. The first reboot is still needed to activate extroot on a fresh setup.

Version 3.3.5 was confirmed working on the GL.iNet GL-MT3000 (Beryl AX). This guide describes that version. Compatibility mode is opt-in: including mitmproxy does **not** automatically enable `-mitmproxy-compat`.

## How it works

OpenWrt only switches package storage to a USB drive (called *extroot*) while booting, so the script runs in two stages with a reboot in between. You start stage 1 yourself; stage 2 starts on its own after the reboot.

| Stage | Steps | What happens |
|---|---|---|
| 1 | 1 – 5 | Updates the package lists, upgrades installed packages by default (skip with `-noupgrade`), installs the base packages, partitions and formats the USB drive, copies the router's current settings and packages to it, and activates extroot. Then the router reboots. |
| 2 | 6 – 10 | Starts about a minute after boot. Installs tcpdump, dnstop and nginx, downloads the custom packages for your router to `/usb/tmp`, installs them, checks that the main tools run and that the selected mitmproxy compatibility mode forwards HTTP/HTTPS, and reboots unless `-noreboot` was selected. |

Extroot is only activated after the copy on the USB drive has been verified. Stage 2 only reboots when every step succeeded; otherwise it stops and leaves a log explaining what failed.

## Supported routers

| `-target` | Devices | SoC | Architecture |
|---|---|---|---|
| `linksyswrt` | Linksys WRT1200AC, WRT1900AC, WRT3200ACM, WRT32X | Marvell Armada 385 (WRT1900AC v1: Armada XP) | `arm_cortex-a9_vfpv3-d16` |
| `glinet` | GL.iNet GL-MT6000 (Flint 2), GL-MT3000 (Beryl AX), GL-MT3600BE (Beryl 7) | MediaTek MT7986AV, MT7981BA, MT7987AV | `aarch64_cortex-a53` |

The script reads the router's architecture and stops if it does not match the `-target` you chose.

## Requirements

You need a router from the table above running OpenWrt 25.12 (any 25.12.x release; the script stops on other versions). For a fresh installation, start with a freshly flashed router. To repair or continue an existing USB/extroot installation, use the resume commands below. The router needs working internet on its WAN port, a root password, and SSH access from your computer.

You also need a USB 3.0 flash drive or SSD, plugged into the router's USB 3.0 port. A fresh setup erases the selected drive. Resuming an existing extroot installation does not repartition or format it. The script warns if the drive is not running at USB 3.0 speed.

The embedded mitmproxy compatibility bundles require musl Python 3.13 on the matching ARM64 or ARMv7 target. The script checks the Python/CPU match before installing them. All dependencies are prebuilt; no GCC, Rust compiler, source builds or automatic PyPI downloads run on the router.

## USB drive layout

| Option | Drive size | Partition 1 | Partition 2 |
|---|---|---|---|
| `-usb 32g` | 32 GB or smaller | swap, 2048 MiB (label `usbswap`) | ext4, from 2049 MiB to the end (label `usb`) |
| `-usb 64g` | 64 GB or larger | swap, 4096 MiB (label `usbswap`) | ext4, from 4097 MiB to the end (label `usb`) |

Both partitions are mounted by block-mount at every boot. The ext4 partition becomes the router's package storage (`/overlay`), and `/usb` points to the root of that partition, so everything under `/usb` is on the drive.

## Installation

### Step 1: Prepare the router

Flash OpenWrt 25.12, set a root password, and connect the WAN port. Use a wired LAN connection for the setup, because stage 1 stops if your SSH session drops. Then log in over SSH and check the internet connection:

```sh
ssh root@192.168.1.1
ping -c 3 downloads.openwrt.org
```

### Step 2: Plug in the USB drive

Plug the drive into the router's USB 3.0 port. Copy off anything you want to keep first; the drive will be erased.

### Step 3: Copy the script to the router

Use the v3.3.5 script. Copy it from your computer and name the router copy `fw-setup.sh`, as used in the examples below. The `-O` option uses the legacy SCP copy mode supported by OpenWrt's SSH server:

```sh
scp -O fw-setup-3-3-5.sh root@192.168.1.1:/root/fw-setup.sh
```

If you keep the versioned filename on the router, substitute that filename in the commands. For example, `sh /tmp/tmp/fw-setup-3-3-5.sh ...` runs the same installer. On resume, run the newly copied file once to update the saved installer at `/etc/fw-setup/fw-setup.sh`.

### Step 4: Check the router and drive

```sh
sh /root/fw-setup.sh -h
```

The help screen ends with the router's model, its architecture and the matching `-target`, plus any USB drives it can see. If no drive is listed, that is normal on a fresh router: the USB storage drivers are installed in step 3 of the script.

### Step 5: Run stage 1

Pick the `-usb` and `-target` values for your hardware, for example:

```sh
sh /root/fw-setup.sh -usb 64g -target glinet -upgrade -with-mitmproxy -mitmproxy-compat -noreboot
```

For a Linksys WRT router with a 32 GB drive:

```sh
sh /root/fw-setup.sh -usb 32g -target linksyswrt -upgrade -with-mitmproxy -mitmproxy-compat -noreboot
```

Choose the actual drive size and matching target. Package upgrades are enabled by default; explicit `-upgrade` also overrides a saved `-noupgrade` setting when resuming. Use `-noupgrade` only when you deliberately want to skip package upgrades.

When the script reaches step 4 it shows the drive it found and asks you to type `YES` before erasing it. After step 5 it reboots the router within 10 seconds and your SSH session closes.

### Step 6: Let stage 2 finish

Wait a minute or two, log in again, and follow the log:

```sh
tail -f /usb/fw-setup.log
```

Stage 2 first waits (up to five minutes) for internet access and a synced clock, then installs everything. The mitmproxy install can take a while because it also installs prebuilt Python modules. Version 3.3.5 allows up to three minutes for the local mitmweb test to start and prints progress every 15 seconds while waiting.

With `-noreboot`, successful setup ends with **Setup complete. Reboot skipped (-noreboot); reboot manually when ready.** Without that option, stage 2 reboots after all required checks pass. A failed check stops completion and prevents the final reboot.

### Resume an existing installation

Use these commands after copying the latest script to `/root/fw-setup.sh`. They keep the existing USB/extroot layout and run stage 2 without formatting the drive.

GL.iNet (including GL-MT3000 Beryl AX):

```sh
sh /root/fw-setup.sh -resume -target glinet -upgrade -with-mitmproxy -mitmproxy-compat -noreboot
```

Linksys WRT:

```sh
sh /root/fw-setup.sh -resume -target linksyswrt -upgrade -with-mitmproxy -mitmproxy-compat -noreboot
```

After that first run updates the saved installer, subsequent retries can use `/etc/fw-setup/fw-setup.sh` with the same options. Explicit options override the corresponding saved settings.

## Command-line options

| Option | Meaning |
|---|---|
| `-usb 32g` or `-usb 64g` | Drive size class; sets the swap size (2 GB or 4 GB). Required for a first run. |
| `-target linksyswrt` or `-target glinet` | Router family; selects which custom packages to download. Required for a fresh setup; resume can use the saved target. |
| `-dev /dev/sdX` | Use this USB drive instead of detecting it automatically (needed if more than one USB drive is plugged in). Only USB drives are accepted. |
| `-force` | Erase the USB drive without asking for `YES`, for unattended runs. |
| `-upgrade` | Enable `apk upgrade` (the default), including when resuming. Overrides an older saved `-noupgrade`. Uses `-y` if supported by the installed apk; otherwise uses its supported non-interactive behavior. Upgrades packages, not the router firmware. |
| `-noupgrade` | Explicitly skip package upgrades. |
| `-with-mitmproxy` | Include mitmproxy; also overrides a previous `-skip-mitmproxy`. Mitmproxy is included by default, but compatibility mode needs its own option. |
| `-mitmproxy-compat` | Use the embedded prebuilt dependencies, reviewed Python replacements and private launchers. Supports regular HTTP/HTTPS proxy mode on the matching ARM64/ARMv7 musl Python 3.13 target. Saved across reboot/resume. |
| `-skip-mitmproxy` | Install ClamAV, Snort 3 and nDPI without mitmproxy. Saved across reboot/resume. |
| `-noreboot` | Skip only the final reboot after successful stage 2. The first reboot for fresh extroot activation still occurs. Saved across reboot/resume. |
| `-manual-reboot` | Pause at the end of stage 1 so you can manually reboot to activate extroot and start stage 2. |
| `-resume` | Run stage 2 by hand, for example after fixing an error. |
| `-h` | Show help, router details and detected USB drives. |

Running the script again on a router that already uses the USB drive for package storage skips steps 4 and 5 and does not touch the drive.

## What gets installed

**Step 3, on the router's internal flash:** `openvpn-openssl`, `luci-app-openvpn`, `wireguard-tools`, `kmod-wireguard`, `luci-proto-wireguard`, `pbr`, `luci-app-pbr`, `dnsmasq-full` (replaces `dnsmasq`), `block-mount`, `swap-utils`, `usbutils`, `kmod-usb-core`, `kmod-usb2`, `kmod-usb3`, `kmod-usb-storage`, `kmod-usb-storage-uas`, `kmod-fs-ext4`, `kmod-fs-vfat`, `fdisk`, `gdisk`, `parted`, `e2fsprogs`, `curl` and `coreutils`. If any of these fail, the script stops before touching the USB drive.

**Step 6, on the USB drive:** `tcpdump`, `dnstop`, `nginx-full`, `nginx-mod-naxsi`, `nginx-mod-luci` and `luci-nginx`. LuCI stays on uhttpd at ports 80/443. nginx runs alongside it on ports 8088/8443, leaving ports 8080/8081 free for mitmproxy.

**Steps 8 and 9, custom packages on the USB drive:**

| Package set | Files | Source |
|---|---|---|
| ClamAV 1.5.4 | `clamav`, `freshclam`, `libclamav`, `luci-app-clamav` | [clamav-1.5.4-r1_For-OpenWRT](https://github.com/infinity-avenir/clamav-1.5.4-r1_For-OpenWRT) |
| Snort 3.12.2.0 | `snort3`, `libdaq3`, `libdnet`, `libhwloc`, `liblzma`, `libnetfilter-queue1`, `libpciaccess`, `libtirpc`, `luajit2`, `luci-app-snort3` | [snort3-3.12.2.0-r1-for-OpenWRT](https://github.com/infinity-avenir/snort3-3.12.2.0-r1-for-OpenWRT) |
| mitmproxy 12.2.3 | `mitmproxy`, `luci-app-mitmproxy` | [mitmproxy_12.2.3-r1_For_OpenWRT](https://github.com/infinity-avenir/mitmproxy_12.2.3-r1_For_OpenWRT) |
| nDPI 6.0 | `libndpi`, `ndpi-tools`, `luci-app-ndpi` | [ndpi-tools-6.0-r1_For_OpenWRT](https://github.com/infinity-avenir/ndpi-tools-6.0-r1_For_OpenWRT) |

Each router family gets the builds for its own architecture. The files are saved in `/usb/tmp`. The LuCI apps are installed separately from the tools, so a problem with a LuCI package cannot block the tool itself. If the router already has one of the general-purpose libraries bundled with Snort (`liblzma`, `libtirpc`, `luajit2`, `libpciaccess`, `libhwloc`, `libnetfilter-queue1`), the installed version is kept instead of being replaced by the bundled one.

After installing, the script runs `clamscan --version`, `freshclam --version`, `snort -V`, `mitmdump --version` and `mitmweb --version` to confirm the tools actually start. The ClamAV LuCI package is r15 and the Snort LuCI package is r2.

With `-mitmproxy-compat`, it also tests HTTP forwarding, HTTPS interception with certificate verification, and the mitmweb interface using temporary loopback ports. On Linksys, it additionally tests compressed responses and Argon2 authentication. These tests do not enable a permanent proxy service or configure traffic interception.

Compatibility mode supports **regular HTTP/HTTPS proxying only**. HTTP/3, UDP proxying, WireGuard, TUN and local redirect modes within mitmproxy are unavailable. This restriction does not remove the separately installed WireGuard VPN tools. The embedded `mitmproxy_rs` Python replacement is a restricted compatibility implementation, not the native Rust extension.

## Checking the result

```sh
df -h /overlay      # should show the USB drive's second partition, e.g. /dev/sda2
cat /proc/swaps     # should list the USB drive's swap partition
ls -l /usb          # should show: /usb -> /overlay
snort -V
clamscan --version
mitmdump --version
```

## Logs and files

| Path | Contents |
|---|---|
| `/usb/fw-setup.log` | Full log of both stages |
| `/tmp/fw-setup.log` | Stage 1 log while it runs (cleared at reboot) |
| `logread -e fw-setup` | Stage 2 messages in the system log |
| `/etc/fw-setup/` | Copy of the script and its saved settings, used by stage 2 |
| `/etc/init.d/fw-setup` | Temporary boot hook that starts stage 2; removed when stage 2 ends |
| `/usb/tmp/` | Downloaded custom packages |
| `/etc/fw-setup/mitmproxy-compat-3-3/runtime-check.log` | Preserved mitmweb test process log on GL.iNet |
| `/etc/fw-setup/mitmproxy-compat-armv7-3-3-3/runtime-check.log` | Preserved mitmweb test process log on Linksys |

## Troubleshooting

**Stage 2 stopped with errors.** The end of `/usb/fw-setup.log` lists every error. The router does not reboot, and the boot hook is removed so stage 2 does not repeat at every boot. Fix the cause (for example a missing or broken package in one of the repositories), then run:

```sh
sh /etc/fw-setup/fw-setup.sh -resume -upgrade -with-mitmproxy -mitmproxy-compat -noreboot
```

This uses the saved router target. Add `-target glinet` or `-target linksyswrt` if you need to specify it explicitly.

**Stage 2 never started, or `df -h /overlay` does not show the USB drive.** The drive was not mounted as extroot, so the router booted from internal flash. Stage 2 refuses to run in that case, so nothing is installed onto the small internal flash. Check `block info` and `logread | grep -i -e block -e extroot`, reseat the drive, and reboot; stage 2 tries again at each boot until it succeeds.

**No internet during stage 2.** Stage 2 waits up to five minutes for internet access and a synced clock, then stops. Fix the WAN connection and reboot, or run the `-resume` command above.

**LuCI is not reachable after setup.** Version 3.3.5 keeps LuCI on uhttpd at ports 80/443. Check `logread -e uhttpd`. To stop nginx and restart uhttpd by hand:

```sh
/etc/init.d/nginx stop; /etc/init.d/nginx disable
/etc/init.d/uhttpd enable; /etc/init.d/uhttpd start
```

**A LuCI app fails to install.** Check its APK metadata: the architecture should be `noarch` and the version should use apk's format, such as `1.0.0-r1`. The tool itself still installs.

**`ModuleNotFoundError: No module named 'mitmproxy_rs'`.** Confirm that compatibility mode was selected:

```sh
grep '^MITMPROXY_COMPAT=' /etc/fw-setup/state | tail -1
```

`MITMPROXY_COMPAT='0'` means the embedded bundle was not selected. Resume with `-with-mitmproxy -mitmproxy-compat`, as shown above. `-with-mitmproxy` alone includes the application but does not select its embedded compatibility dependencies. A value of `1` is expected for compatibility mode; if the error persists, inspect step 9 for an installation or launcher failure.

**mitmproxy versions pass but the local test times out.** Version 3.3.5 waits up to 180 seconds for mitmweb to open its local test ports and preserves its process log. A process that exits early or never becomes ready still fails. Inspect the log for your target:

```sh
# GL.iNet
cat /etc/fw-setup/mitmproxy-compat-3-3/runtime-check.log

# Linksys WRT
cat /etc/fw-setup/mitmproxy-compat-armv7-3-3-3/runtime-check.log
```

The complete forwarding test has a 420-second outer timeout where the `timeout` utility is available. Certificate verification remains enabled, and the script does not report success or perform the final reboot when a required runtime check fails.

## Going back to internal flash

Power off the router, remove the USB drive, and power it on. The router boots from its internal flash with the settings and packages it had at the end of stage 1. Then remove the USB entries and the stage 2 files:

```sh
uci -q delete fstab.usb
uci -q delete fstab.usbswap
uci -q delete fstab.rwm
uci commit fstab
rm -rf /etc/fw-setup /etc/init.d/fw-setup /etc/rc.d/S99fw-setup
```

## Important notes

**Unsigned packages.** The custom packages are installed with `--allow-untrusted`, which skips signature checks. Only install them from repositories you control.

**Firmware upgrades.** A sysupgrade resets the extroot setup. Run the script again after upgrading the firmware; this erases and rebuilds the USB drive.

**ClamAV virus database.** By default ClamAV keeps its database under `/var`, which is in RAM. Point its database directory to a folder under `/usb` (for example `/usb/clamav`) before running `freshclam`.

**Package upgrades.** Version 3.3.5 enables `apk upgrade` by default. The examples include `-upgrade` explicitly so a saved `-noupgrade` does not suppress it. Use `-noupgrade` to opt out. This upgrades installed packages; it does not perform a firmware sysupgrade. The script does not use `apk upgrade --available`.

**Configuration is not included.** The script installs the tools but does not configure them. VPN tunnels and policy-based routing rules, Snort rules and traffic handling, the mitmproxy setup, NAXSI rules and the ClamAV database location all need to be set up separately.
