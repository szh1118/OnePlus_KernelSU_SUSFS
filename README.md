# OnePlus 13 (dodge) — PixelOS kernel

A **device-specific** fork. It targets **exactly one device and one ROM** — my own OnePlus 13 — and
nothing else. Everything here is built from the ROM's **own kernel tree**, so the kernel release
string matches the ROM byte-for-byte and its `vendor_dlkm` modules load.

> Upstream [WildKernels/OnePlus_KernelSU_SUSFS](https://github.com/WildKernels/OnePlus_KernelSU_SUSFS)
> targets **OnePlus stock (OnePlusOSS) trees**. This ROM's kernel comes from the **LineageOS/AOSP
> tree**, so the pipeline had to be re-pointed at it. See [Why a fork](#why-a-fork).

---

## 🎯 Target

| | |
|---|---|
| **Device** | OnePlus 13 — codename **`dodge`**, SoC `sun` / **SM8750** (Snapdragon 8 Elite) |
| **ROM** | `PixelOS_dodge-17.0-20260912-0535` (**Android 17**) |
| **Kernel** | **`6.6.142-4k-gc568e18c7f62`** — must match the ROM exactly |
| **Source** | `LineageOS/android_kernel_oneplus_sm8750` @ `c568e18c7f62` + the ROM's own `.config` (extracted from its `boot.img`) |
| **Toolchain** | ZyC **clang 22** (clang 19 silently produces an unbootable kernel here) |
| **Vermagic** | `6.6.142-4k-gc568e18c7f62 SMP preempt mod_unload modversions aarch64` |

⚠️ This kernel is **only** for the combination above. Flashing it on any other ROM/kernel version
will break module loading (the release string is part of the module ABI).

## ✅ Supported features

| Feature | State |
|---|---|
| **KernelSU-Next** (versionCode `33239`) | ✅ |
| Matching **Manager APK** (`33239`) shipped in Releases | ✅ |
| **SUSFS** v2.2.0 (+ userspace module) | ✅ |
| **BBG** — Baseband Guard (protects non-user partitions) | ✅ |
| **Unicode bypass fix** (non-printable path traversal) | ✅ |
| **IP_SET + IPv6 NAT**, **TTL target** | ✅ |
| **zram**: built-in, **LZ4** default, ZSTD available, multi-comp streams | ✅ verified on device (`[lz4]`, 6 GB swap) |
| **CVE-2026-89839 / 80830 / 80842** (f2fs / USB hub / bridge) | ✅ patched **and compiled in** (`=y` subsystems) |
| **CVE-2026-90255 / 93209 / 80762** (kernel BT core) | ➖ **N/A on this device** — see [Bluetooth](#-bluetooth-on-this-rom) |
| Droidspaces + NTSync | ⏳ planned (full) |
| **Re:Kernel** v11.7 (tombstone / freeze support) | ✅ **built into the image** — `Re-Kernel hooked!` in dmesg |
| **ADIOS** I/O scheduler | ✅ **built in and set as the default** (`[adios]` on every block device) |
| NetHunter + rtw88 / monitor-mode injection | ⏳ planned (full) |
| KPM (KernelPatch Next) | ⏳ planned (full) |
| LZ4KD zram algorithm (experimental) | ⏳ planned (full) |
| HMBIRD (OnePlus fengchi SCX) | ❌ **not possible** — the scheduler source does not exist in this tree |

## 📋 TODO

- [x] Build from the ROM's own kernel tree so `uname -r` / vermagic match it exactly
- [x] Track down the early-boot silent hang → root cause was the **compiler version**
- [x] KernelSU-Next `33239` + an exact-versionCode Manager APK (no more *"kernel update required"*)
- [x] SUSFS v2.2.0 + BBG
- [x] `unicode`, `ip_set`, `ttl`
- [x] zram built-in with LZ4 default + multi-comp — **verified on device**
- [x] CVE backports `89839 / 80830 / 80842` — **patched *and* compiled into the image** (`=y` subsystems)
- [x] CVE `90255 / 93209 / 80762` — **not applicable here**: they live in the *kernel* BT core, which this ROM never uses (see below)
- [ ] CVE-2026-93235 (f2fs post-EOF zeroing) — our 6.6.142 snapshot lacks the prerequisite `f2fs_zero_post_eof_page()` series entirely (fixed upstream in 6.6.157), so this is not a one-patch backport; **deliberately not applied**
- [x] Re:Kernel — **compiled into the kernel image** (`obj-y` in `drivers/android/`). The out-of-tree `.ko` route cannot work here: with `O=` set, kbuild's `Makefile.modfinal` never gets a rule for the final `.ko` (`No rule to make target rekernel.ko, needed by '__modfinal'`). Built-in also removes the runtime dependency on `kallsyms_lookup_name` being exported.
- [x] ADIOS — `patches/adios/adios-6.6.patch` (6.12→6.6: only `elevator_find_get(q, name)` and `!blk_queue_nonrot(q)` differ), built-in and set as default
- [x] **lite** = KSU + SUSFS + BBG + unicode/ip_set/ttl + zram + Re:Kernel + ADIOS — **verified on device**: `[adios]` default, zram `[lz4]`, `baseband_guard` in the LSM chain, `Re:Kernel v11.7 … Re-Kernel hooked!`
- [ ] **full** = lite + Droidspaces/NTSync + NetHunter/rtw88 + KPM + LZ4KD
- [ ] Split the build matrix into `OP13-lite` / `OP13-full` configs
- [x] ~~Switch the root solution to BakaSU/SukiSU~~ — **dropped**: the SUSFS patches do not apply to BakaSU's tree (94 / 97 failed hunks), and SUSFS is not negotiable
- [x] ~~HMBIRD~~ — **dropped**: no fengchi SCX source in this tree

## 📶 Bluetooth on this ROM

Bluetooth **works**, but not through the kernel: the QTI HAL
(`android.hardware.bluetooth@aidl-service-qti`) owns `/dev/ttyHS0` and speaks **H4 in userspace**
(the Fluoride/GD stack implements L2CAP/RFCOMM inside `com.android.bluetooth`). The GKI BT core
modules from `system_dlkm` (`bluetooth.ko`, `hci_uart.ko`, …) are therefore **unused reference
modules** here — on this kernel they even fail to load, and that is harmless:

```
bluetooth: disagrees about version of symbol sk_filter_trim_cap (err -22)
```

That CRC divergence comes from `CONFIG_BPF_STREAM_PARSER`, which upstream's *ip_set* step switches on
even though the ROM ships it off (`sk_filter_trim_cap`'s prototype takes a `struct sock *`, so the
modversions CRC covers that type's closure). The build now keeps it off for `wild/sm8750`, and every
build prints a `CONFIG PARITY vs ROM baseline` diff so silent config drift shows up in the log.

**Consequence:** CVE-2026-90255 / 93209 / 80762 target `net/bluetooth/*.c`, i.e. a code path this
device does not execute — building the BT core into the image just to "fix" them would add code that
never runs. They are documented as N/A rather than patched.

## 🔍 Verifying a build on the device

`/proc/config.gz` is world-readable, so the fastest audit is:

```sh
adb pull /proc/config.gz && zcat config.gz | grep -E 'ZRAM_DEF_COMP|KSU|BBG|LTO_NONE'
adb shell 'cat /sys/kernel/security/lsm'          # needs a root shell: expect baseband_guard
adb shell 'cat /proc/swaps; cat /sys/block/zram0/comp_algorithm'   # expect [lz4]
adb shell 'uname -r'                              # expect 6.6.142-4k-gc568e18c7f62
adb shell 'cat /sys/block/sda/queue/scheduler'    # expect: none mq-deadline kyber [adios] bfq
adb shell 'dmesg | grep -iE "Re-Kernel"'          # expect: Re:Kernel v11.7 … / Re-Kernel hooked!
```


## 📥 Install

1. Grab both assets from the [latest release](../../releases):
   - the AnyKernel3 kernel zip
   - the matching **KernelSU-Next Manager APK** (`…_33239-release.apk`)
2. Back up your current boot image.
3. Flash the kernel zip in OrangeFox recovery (or KernelFlasher). It writes **only the `boot`
   partition** of the active slot; A/B is handled automatically.
4. Install the Manager APK **of the same versionCode as the kernel** (newer = *"kernel update
   required"*).
5. For SUSFS hiding, install the userspace module `sidex15/ksu_module_susfs` (v2.2.0) through the Manager.

**Rollback:** `fastboot flash boot_a boot.img` and `fastboot flash boot_b boot.img` with the ROM's
stock boot image.

## 🔧 Why a fork / how it works

- **Source**: the ROM's kernel is `LineageOS/android_kernel_oneplus_sm8750` @ `c568e18c7f62`; upstream
  builds only from OnePlusOSS trees whose versions stop at 6.6.118.
- **Version string**: the action is patched so `LOCALVERSION` is pinned to `-4k-gc568e18c7f62`, LTO is
  forced to `LTO_NONE` (as the ROM builds it) and `-mcpu=oryon-1` is removed.
- **The boot hang**: every 6.6.142 rebuild built with ZyC **clang 19** hung silently at the first
  logo — no error, correct vermagic, clean patches. Switching to **clang 22** fixed it. If you rebuild,
  do not lower the compiler version.
- **SUSFS** is vendored into the repo (`vendor_susfs4ksu/`) because GitLab archive downloads are
  blocked from GitHub runners.
- **Patches** live in `patches/` (e.g. the verified CVE backports in `patches/cve/`).
- **AK3**: the `boot`-version check is disabled (`do.check_boot_version=0`) because it only recognises
  `-androidNN` style version strings, which this ROM does not use.

## 🙏 Credits

[WildKernels](https://github.com/WildKernels) (pipeline), [KernelSU-Next](https://github.com/KernelSU-Next/KernelSU-Next),
[tiann/KernelSU](https://github.com/tiann/KernelSU), [simonpunk/susfs4ksu](https://gitlab.com/simonpunk/susfs4ksu),
[vc-teahouse/Baseband-guard](https://github.com/vc-teahouse/Baseband-guard), [sidex15](https://github.com/sidex15),
[ZyCromerZ/Clang](https://github.com/ZyCromerZ/Clang), and the LineageOS/PixelOS maintainers.

## ⚠️ Disclaimer

Flashing this modifies your device. Back up first. Use at your own risk — no warranty, no blame.
