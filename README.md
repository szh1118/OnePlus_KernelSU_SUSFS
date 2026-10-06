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
| **CVE-2026-93235** (f2fs post-EOF stale data on size extension) | ✅ ported, **reproduced and fixed on device** (see below) |
| **CVE-2026-90255 / 93209 / 80762** (kernel BT core) | ➖ **N/A on this device** — see [Bluetooth](#-bluetooth-on-this-rom) |
| Droidspaces + NTSync | ✅ **`OP13-full` only** — device-verified (module set identical to lite, `0` CRC mismatches) |
| **Re:Kernel** v11.7 (tombstone / freeze support) | ✅ **built into the image** — `Re-Kernel hooked!` in dmesg |
| **ADIOS** I/O scheduler | ✅ **built in and set as the default** (`[adios]` on every block device) |
| NetHunter + rtw88 / monitor-mode injection | ⏳ planned (full) |
| KPM (KernelPatch Next) | ❌ **dropped** — it is a competing root solution, not an add-on to KernelSU-Next |
| LZ4KD zram algorithm (experimental) | ⏳ planned (full) |
| HMBIRD (OnePlus fengchi SCX) | ❌ **not possible** — the scheduler source does not exist in this tree |

## 🔀 Two variants — `OP13-lite` and `OP13-full`

The build matrix is generated from **every** `configs/**/*.json`, so one dispatch of
`op_model=android15-6.6` builds both as independent jobs:

| Config | Contents |
|---|---|
| [`configs/a16/OP13-lite.json`](configs/a16/OP13-lite.json) | Everything marked ✅ above, minus Droidspaces/NTSync. Ships as the **Latest** release (`lite-r13`) — the conservative choice. |
| [`configs/a16/OP13-full.json`](configs/a16/OP13-full.json) | lite **+ Droidspaces (`ds`) + NTSync (`ntsync`)** — shipped as a **pre-release** (`full-r1`); later rounds may add NetHunter/rtw88 and LZ4KD, one group at a time. |

⚠️ `full` is **not** automatically safe to flash: Droidspaces flips five options the stock ROM leaves
off (`CONFIG_SYSVIPC`, `CONFIG_POSIX_MQUEUE`, `CONFIG_USER_NS`, `CONFIG_PID_NS`, `CONFIG_DEVTMPFS`), and
`CONFIG_SYSVIPC` changes the layout of `struct ipc_namespace`. Because module CRCs (genksyms) hash the
transitive type closure of each exported prototype, a Kconfig flip can silently make a `vendor_dlkm`
`.ko` unloadable — that is exactly how `CONFIG_BPF_STREAM_PARSER` once broke `bluetooth.ko` here.

So a `full` build is accepted only after the **module-load comparison**: boot it, then check that
dmesg still has no `disagrees about version of symbol` lines and that the set of loaded modules has not
shrunk versus the `lite` baseline (548 of the 589 shipped `.ko` load on `lite-r13`: 21 are the debug
modules we blacklist on purpose, `zram`/`zsmalloc` are built into our image, and the remaining ~20 —
`can_*`, `tls`, `video`, `icnss2`, `kheaders`, … — simply never load on this device). Rollback is one
flash away — keep the `lite` AK3 zip, or `fastboot flash boot_a/b` with the ROM's own `boot.img`.

**Result (2026-10-07, `OP13-full` on device):** no `disagrees about version of symbol` in dmesg, and the
loaded-module set is **byte-for-byte the same names** as on `lite-r13` (548 of 589). Wi-Fi (associated +
got a DHCP lease), cellular data (NR, validated), Bluetooth (`State: ON`), NFC (`mState=on`), camera HAL
(5 devices) all work; `logcat -b crash` is empty; no avc denial involves `ntsync`/`sysvipc`/namespaces.
The `lite` features survived too: `[adios]` still the default scheduler, `baseband_guard` in the LSM
chain, `Re-Kernel hooked!`, zram0 6 GB `[lz4]`.

> Two things worth knowing when you verify a flash yourself:
> - `/proc/config.gz` is **faked** here (the "Fake config.gz" patch keeps it matching the stock ROM), so
>   confirm new features through runtime gates instead: `/proc/sysvipc/{msg,sem,shm}`, `/dev/ntsync`
>   (labelled `gpu_device` by the ROM's own `file_contexts`), `/proc/sys/user/max_user_namespaces`.
> - A recovery UI printing "success" is not evidence. Check the slot: `sha256sum /dev/block/by-name/boot_a`
>   vs `boot_b` vs the ROM's `boot.img`, and compare `cat /proc/version`'s build timestamp with the one
>   embedded in the `Image` you downloaded from the workflow run.

## 📋 TODO

- [x] Build from the ROM's own kernel tree so `uname -r` / vermagic match it exactly
- [x] Track down the early-boot silent hang → root cause was the **compiler version**
- [x] KernelSU-Next `33239` + an exact-versionCode Manager APK (no more *"kernel update required"*)
- [x] SUSFS v2.2.0 + BBG
- [x] `unicode`, `ip_set`, `ttl`
- [x] zram built-in with LZ4 default + multi-comp — **verified on device**
- [x] CVE backports `89839 / 80830 / 80842` — **patched *and* compiled into the image** (`=y` subsystems)
- [x] CVE `90255 / 93209 / 80762` — **not applicable here**: they live in the *kernel* BT core, which this ROM never uses (see below)
- [x] CVE-2026-93235 (f2fs: zero post-EOF data when extending file size) — **fixed here**. Our f2fs is the AOSP/OnePlus variant and never had upstream's `f2fs_zero_post_eof_page()`, so the stable backports do not apply (all five branches fail every hunk). `patches/cve/CVE-2026-93235.patch` ports upstream's post-fix implementation on top of the existing `fill_zero()` and calls it from the same eight sites. Reproduced first and verified after, both with a raw-PBA test on a zram-backed f2fs device: before the fix `truncate-up` exposed the stale `0x5a` bytes in `[4080,4096)`, after it the gap reads zeros. See the patch comment for the one deliberate deviation from upstream (`/data` is mounted `fsync_mode=nobarrier`, so the zeroing must not be gated on strict mode or on cached pages).
- [x] Re:Kernel — **compiled into the kernel image** (`obj-y` in `drivers/android/`). The out-of-tree `.ko` route cannot work here: with `O=` set, kbuild's `Makefile.modfinal` never gets a rule for the final `.ko` (`No rule to make target rekernel.ko, needed by '__modfinal'`). Built-in also removes the runtime dependency on `kallsyms_lookup_name` being exported.
- [x] ADIOS — `patches/adios/adios-6.6.patch` (6.12→6.6: only `elevator_find_get(q, name)` and `!blk_queue_nonrot(q)` differ), built-in and set as default
- [x] **lite** = KSU + SUSFS + BBG + unicode/ip_set/ttl + zram + Re:Kernel + ADIOS — **verified on device**: `[adios]` default, zram `[lz4]`, `baseband_guard` in the LSM chain, `Re:Kernel v11.7 … Re-Kernel hooked!`
- [x] **full** round 1 — Droidspaces + NTSync: built as `OP13-full`, flashed, **device-verified** (see [Two variants](#-two-variants--op13-lite-and-op13-full))
- [ ] **full** round 2 — NetHunter + rtw88 / monitor-mode injection (riskiest group, own round)
- [ ] **full** round 3 — LZ4KD zram algorithm (experimental)
- [ ] ~~KPM (KernelPatch Next)~~ — **dropped by decision (2026-10-07)**: KernelPatch is a second root solution; running it next to KernelSU-Next is a conflict, not a feature.
- [x] Split the build matrix into `OP13-lite` / `OP13-full` configs (see [Two variants](#-two-variants--op13-lite-and-op13-full))
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

## 🧪 Verifying the f2fs post-EOF fix (CVE-2026-93235)

The reproduction follows upstream's own fstests `generic/794` sequence, but on a **zram-backed**
f2fs device so nothing touches `/data`:

```
create a zram device (echo N > /sys/class/zram-control/hot_add), make_f2fs it, mount it
write 1 MiB of 0x5a with a unique marker at file offset 16
find the file's first block by scanning the device for the marker
truncate the file to 4080, umount
write 4096 x 0x5a straight at that block      # the "stale bytes" the CVE is about
mount again, extend the file to 8192
read [4080,4096)  ->  all zeros = fixed, 0x5a = vulnerable
```

Notes for anyone repeating this: a loop-over-file device cannot work here — the kernel is not
allowed to *write* a regular file (`u:r:kernel:s0` gets EIO on the backing file), which leaves an
un-unmountable f2fs and can wedge the device. Use zram, guard every `umount` with `timeout`, and
never leave a broken mount behind.

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

   Pick a variant: **`lite-r13`** is the Latest (recommended); **`full-r1`** is a pre-release that adds
   Droidspaces + NTSync and also ships a ready-made `new_boot_full.img` for the fastboot route in step 5.
2. Back up your current boot image.
3. **Verify the checksum on the device before flashing.** A recovery screen saying "Success" is not proof
   that anything was written — check `sha256sum /dev/block/by-name/boot_a` against what you expect, and
   compare `/proc/version`'s build timestamp with the one stated in the release notes.
4. Flash the kernel zip in OrangeFox recovery (or KernelFlasher). It writes **only the `boot`
   partition** of the active slot; A/B is handled automatically.
5. If the recovery route misbehaves, use the prebuilt image instead (recovery lives in its own partition on
   this device, so a bad `boot` is always recoverable):

   ```
   adb reboot bootloader
   fastboot getvar current-slot
   fastboot flash boot_a <the release's boot image>
   ```

6. Install the Manager APK **of the same versionCode as the kernel** (newer = *"kernel update
   required"*).
7. For SUSFS hiding, install the userspace module `sidex15/ksu_module_susfs` (v2.2.0) through the Manager.

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

## 📋 优化 TODO / 待办

> 2026-07-11 分析结论。按优先级排列。

### P0 — 高收益、低风险

- [ ] **`opt` 优化补丁逐个测试**：当前 `"opt": false`，全部跳过。WildKernels 上游补丁针对 OnePlusOSS 树，本树是 LineageOS/AOSP 树，全开会卡 logo。应逐个启用测试，安全的留下，卡 logo 的丢弃。优先测试低风险补丁：reduce_gc_thread_sleep_time、silence_irq_cpu_logspam、increase_sk_mem_packets、reduce_freeze_timeout。
- [ ] **Droidspaces + NTSync 合入 lite**：当前只在 `OP13-full` 里。如果日常使用需要，考虑移到 `OP13-lite`。
- [ ] **zram 压缩算法测试 ZSTD**：当前 LZ4 最快但压缩率最低。ZSTD 压缩率高 30-40%，swap 空间更大。对手机 RAM 扩展场景，压缩率可能比速度更重要。

### P1 — 中等收益

- [ ] **源码下载缓存**：当前每次构建都重新下载内核源码。可用 actions/cache 按 manifest revision 缓存，revision 不变时跳过下载。注意不要影响上游更新追踪。
- [ ] **考虑 sccache 替代 ccache**：sccache 为 CI 设计，并行性更好，迁移成本低（改环境变量即可）。
- [ ] **BBR/BBR3**：日用场景收益小，主要收益在热点共享/高延迟网络。当前不开是对的，保持现状。

### P2 — 低优先级 / 高工作量

- [ ] **ThinLTO**：`ld-wrapper` 已写好 `--thinlto-jobs`，但会改变符号表影响 vendor 模块 CRC。收益 1-3%，风险高，不建议。
- [ ] **SCX 调度器移植**：内核 6.10+ 才有 SCX 框架，backport 到 6.6 工作量巨大，性价比低。
- [ ] **HMBIRD (fengchi)**：OnePlus 闭源 OEM 调度器，源码不在公开树里，无法使用。
- [ ] **编译器额外标志**（`-falign-jump=32`、`-falign-functions=32` 等）：内核非计算密集型，收益极小，不建议折腾。

### 已确认不需要 / 不做

| 方向 | 结论 | 原因 |
|---|---|---|
| `opt: true` 全开 | ❌ | 卡 logo，LineageOS 树与补丁不兼容 |
| BBR 日用 | ❌ | 仅热点/高延迟网络有益，日用无收益 |
| ThinLTO | ❌ | 模块 CRC 风险 > 1-3% 收益 |
| SCX 移植 | ❌ | 工作量巨大，6.6 无 SCX 框架 |
| HMBIRD | ❌ | 闭源，源码不可得 |
| 编译器标志调优 | ❌ | ZyCromerZ clang 22 已足够新，内核非计算密集型 |
| ADIOS I/O 调度器 | ✅ 已启用 | 块 I/O 层面已覆盖，与 CPU 调度器不冲突 |

---
## 🙏 Credits

[WildKernels](https://github.com/WildKernels) (pipeline), [KernelSU-Next](https://github.com/KernelSU-Next/KernelSU-Next),
[tiann/KernelSU](https://github.com/tiann/KernelSU), [simonpunk/susfs4ksu](https://gitlab.com/simonpunk/susfs4ksu),
[vc-teahouse/Baseband-guard](https://github.com/vc-teahouse/Baseband-guard), [sidex15](https://github.com/sidex15),
[ZyCromerZ/Clang](https://github.com/ZyCromerZ/Clang), and the LineageOS/PixelOS maintainers.

## ⚠️ Disclaimer

Flashing this modifies your device. Back up first. Use at your own risk — no warranty, no blame.
