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
| **zram**: built-in, **LZ4** default, ZSTD available, multi-comp streams | ✅ |
| **CVE fixes**: 2026-89839 / 93209 / 90255 / 80830 / 80842 | ✅ |
| Droidspaces + NTSync | ⏳ planned (full) |
| Re:Kernel (tombstone support) | ⏳ planned (lite) |
| ADIOS I/O scheduler | ⏳ planned (lite, 6.12→6.6 port) |
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
- [x] zram built-in with LZ4 default + multi-comp *(pending on-device boot check of r2)*
- [x] Five verified CVE fixes backported *(pending on-device boot check of r2)*
- [ ] CVE-2026-93235 / CVE-2026-80762 — need manual porting (OnePlus' f2fs changes / context drift)
- [ ] Re:Kernel — build `re-kernel.ko` against this kernel, ship as a Magisk module
- [ ] ADIOS — port `block/elevator.c` from 6.12 to 6.6, add the scheduler
- [ ] **lite** = everything above (KSU+SUSFS+BBG+unicode/ip_set/ttl+zram+Re:Kernel+ADIOS)
- [ ] **full** = lite + Droidspaces/NTSync + NetHunter/rtw88 + KPM + LZ4KD
- [ ] Split the build matrix into `OP13-lite` / `OP13-full` configs
- [x] ~~Switch the root solution to BakaSU/SukiSU~~ — **dropped**: the SUSFS patches do not apply to BakaSU's tree (94 / 97 failed hunks), and SUSFS is not negotiable
- [x] ~~HMBIRD~~ — **dropped**: no fengchi SCX source in this tree

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
