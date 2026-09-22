# ChicKernel tapas-custom

Custom build of [ChicKernel](https://github.com/ChicKernel/device_xiaomi_gemstones-kernel)
for the **Xiaomi Redmi Note 12 4G (tapas)** only.

Built on top of `android13-5.15-lts` with the **KSU-Next + SuSFS** variant.

---

## What this branch changes

### 1. KernelSU-Next + SuSFS (base variant)

Build config: `build.config.gki.aarch64.chickernel.ksun.susfs.tapas`
(Bazel target: `chickernel_ksun_susfs_tapas`)

Root via KernelSU-Next with SuSFS hiding (`CONFIG_KSU=y`,
`CONFIG_KSU_SUSFS` support compiled in, managed through the KernelSU-Next
submodule on branch `next-susfs`).

### 2. TTL = 65 (tethering detection bypass)

`net/ipv4/ip_output.c` — every egress IPv4 packet (on-device **and**
forwarded/tethered) gets its TTL normalized to **65** at
`__ip_finish_output()`, before GSO segmentation and fragmentation, with the
IP header checksum recomputed. Carrier-side TTL-comparison tethering
detection sees the same hop count for both.

- Change the value: edit `#define TAPAS_FORCE_TTL 65` or use the CI input.
- IPv4 only. IPv6 hop-limit is untouched.
- Side effects: traceroute becomes flat (expected).

### 3. Modem / radio / IMEI / GPS / NFC removal

Reality of this device: the modem stack (MPSS remoteproc, QRTR control
channel, IPA data path) and the NFC driver are **prebuilt vendor modules**
in `vendor_dlkm` — they are not part of this kernel image and cannot be
compiled out. The kernel-side levers are:

| Layer | Change | Effect |
|---|---|---|
| Kernel image | `# CONFIG_NFC is not set` | NFC core removed; vendor `nq-nci` module can no longer resolve its symbols → NFC hardware dead |
| Kernel image | `# CONFIG_GNSS is not set` | GNSS subsystem removed (Qualcomm GPS does not use it, kept for completeness) |
| Cmdline | `module_blacklist=ipa3,ipa,rmnet_*,datactl,dpl,qcom_q6v5_pas,qrtr,qrtr-smd,mhi,mhi_net,nq-nci` | modem/radio modules refuse to load at boot |

Consequences of the **full** blacklist (default):

- No cellular: no 4G/LTE data, no calls, no SMS.
- IMEI: RIL gets no modem response — IMEI is not exposed to the OS.
  (The IMEI itself physically lives in modem NVRAM; this build makes it
  inaccessible, it does not erase it.)
- GPS: the GPS engine on SDM685 is part of the modem subsystem (MPSS) —
  with MPSS dead, satellite location is dead. (Network/WiFi location from
  userspace services is a separate matter.)

> ⚠️ **AUDIO WARNING (SDM685)**: ADSP (audio DSP) is booted by the *same*
> vendor module as the modem (`qcom_q6v5_pas`), and parts of the audio
> stack talk over QRTR. If speaker/microphone/camera die with the full
> tier, rebuild with the **safe** tier (CI input `modem_tier: safe`):
> mobile data + NFC stay dead, calls/SMS/IMEI/GPS keep working.

### 4. Battery stats logging

Already handled by the base kernel: `kernel/printk/printk.c` contains a
`filtered_modules[]` array that drops kmsg spam from `[sm5602]`, `nopmi`,
`[bq25`, `[sc85` vendor modules whenever `CONFIG_POWER_SUPPLY_DEBUG` is
unset — this branch explicitly keeps it unset in `tapas.config`.
(Android's `dumpsys batterystats` is a userspace framework feature; the
per-UID CPU accounting it relies on (`CONFIG_UID_SYS_STATS`) is kept
enabled so the Settings battery screen keeps working.)

### 5. Filesystem support (added)

`CONFIG_SQUASHFS` (+XZ/ZSTD), `CONFIG_BTRFS_FS` (+POSIX ACL),
`CONFIG_NTFS3_FS` (+LZX/XPRESS, +POSIX ACL), `CONFIG_ISO9660_FS`,
`CONFIG_UDF_FS` (+NLS), `CONFIG_HFSPLUS_FS`, `CONFIG_CIFS` — all built-in.

Already present in the base kernel: ext4, F2FS, exFAT, EROFS, vfat/msdos,
9p, overlayfs, fuse/virtiofs, incrementalfs.

Still available in-tree if you want more (not enabled):
`CONFIG_XFS_FS`, `CONFIG_JFS_FS`, `CONFIG_REISERFS_FS`, `CONFIG_NILFS2_FS`,
`CONFIG_MINIX_FS`, `CONFIG_SYSV_FS`, `CONFIG_ADFS_FS`, `CONFIG_AFFS_FS`,
`CONFIG_BFS_FS`, `CONFIG_CRAMFS`, `CONFIG_ROMFS`, `CONFIG_JFFS2`,
`CONFIG_UBIFS_FS`, `CONFIG_QNX4FS`, `CONFIG_QNX6FS`, `CONFIG_UFS_FS`,
`CONFIG_EFS_FS`, `CONFIG_HPFS_FS`, `CONFIG_FREEVXFS_FS`, `CONFIG_BEFS_FS`,
`CONFIG_OCFS2`, `CONFIG_GFS2_FS`, `CONFIG_NFS_FS`, `CONFIG_AFS_FS`,
`CONFIG_CODA_FS`, `CONFIG_ORANGEFS_FS`, `CONFIG_ZONEFS_FS`, `CONFIG_ECRYPT_FS`.

### 6. VPN hide patch

`net.core.hide_tun` sysctl (default `1`): hides `tunX` interfaces from
`/proc/net/dev` and `/proc/net/if_inet6` — the two `/proc` listings apps
commonly scan to detect an active VPN kernel-side.

Toggle at runtime:

```sh
# hide (default)
echo 1 > /proc/sys/net/core/hide_tun
# show again
echo 0 > /proc/sys/net/core/hide_tun
```

Honest limitations — a kernel patch **cannot** hide:
- the Android framework's own VPN state (ConnectivityManager/VpnService);
- `/sys/class/net/tun0`;
- rtnetlink dumps (`getifaddrs()`).

For framework-level hiding, use per-app VPN exclusion in your VPN client.

---

## Building (GitHub Actions)

1. Fork `ChicKernel/device_xiaomi_gemstones-kernel` **with all branches**
   (uncheck "Copy the readme branch only" — actually make sure the fork
   includes `android13-5.15-lts`; if not, push it manually as below).
2. Add this `tapas-custom` branch and the workflow file to your fork:
   ```sh
   git clone https://github.com/<you>/device_xiaomi_gemstones-kernel -b android13-5.15-lts
   cd device_xiaomi_gemstones-kernel
   git fetch https://github.com/<source>/device_xiaomi_gemstones-kernel tapas-custom
   git push origin FETCH_HEAD:refs/heads/tapas-custom
   ```
   (or apply the patch series from `patches/` on top of `android13-5.15-lts`)
3. In your fork: **Actions → Build tapas kernel → Run workflow**.
   Defaults build the `tapas-custom` branch of the repo the workflow
   lives in. Inputs:
   - `modem_tier`: `full` (default) or `safe` (see warning above)
   - `ttl`: `65` (default)
4. Artifacts: `ChicKernel-tapas` — AnyKernel3 zip + `Image`/`Image.gz`/
   `Image.lz4` + `System.map` + `SHA256SUMS`.
   Push a `v*` tag to also get a GitHub Release.

### Manual/local builds

Same procedure as upstream (see the
[upstream build notes](https://github.com/chickendrop89/device_xiaomi_unified-kernel/wiki/Build-notes)),
using `chickernel.xml` from the `readme` branch with your fork URL, then:

```sh
BUILD_CONFIG=common/build.config.gki.aarch64.chickernel.ksun.susfs.tapas build/build.sh
# or
tools/bazel build --config=fast //common:chickernel_ksun_susfs_tapas
```

---

## Flashing (tapas)

**AnyKernel3 zip** — in TWRP/OrangeFox recovery:
Install → select `ChicKernel-tapas-*.zip` → done.

**Raw images** — replace the kernel in the stock boot image:
```sh
# via fastboot (extract boot.img, swap kernel with magiskboot, flash back)
magiskboot unpack boot.img
mv Image.gz kernel     # or Image, matching the original format
magiskboot repack boot.img newboot.img
fastboot flash boot newboot.img
```

First boot after flashing a module-blacklisting kernel may take longer
(vendor module load failures time out). That is expected.

---

## Verifying on device

```sh
# variant + localversion
uname -r          # ...-chickernel-tapas-7-ksun-susfs

# battery logs suppressed, TTL active
dmesg | grep -iE "sm5602|nopmi"        # (empty or near-empty)
# TTL: from a browser, https://ifconfig.me shows your IP;
# TTL check: ping shows ttl=65 on replies to tethered clients

# NFC / GNSS / modem dead
cat /proc/config.gz | gunzip | grep -E "CONFIG_NFC|CONFIG_GNSS"
lsmod | grep -E "ipa|qrtr|mhi|q6v5"    # (empty on full tier)
getprop | grep -i imei                 # null/unavailable on full tier

# tun hiding
cat /proc/net/dev | grep tun           # (empty while VPN is up)
cat /proc/sys/net/core/hide_tun        # 1
```

## KernelSU + SuSFS

Manage root through the KernelSU-Next manager app; SuSFS options are
available in the manager (hide root from specific apps, sus_path etc.).
Upstream docs: [KernelSU-Next](https://github.com/rifux/KernelSU-Next) /
[susfs4ksu](https://gitlab.com/simonpunk/susfs4ksu).

---

## Credits

- [chickendrop89](https://github.com/chickendrop89) — ChicKernel, the
  ack-build-workflow CI this workflow is adapted from, and the vendor
  module kmsg suppression mechanism.
- The Android Common Kernel / GKI teams.
- This branch's tweaks assembled for a tapas WiFi-only privacy build.
