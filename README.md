# OnePlus 13 (dodge) — PixelOS 内核

**设备专属** fork，只针对**一台设备、一个 ROM**——我手上这台 OnePlus 13，其他一概不管。
所有东西都从 ROM **自带的内核树**构建，所以内核 release string 与 ROM 逐字节一致，
`vendor_dlkm` 模块正常加载。

> 上游 [WildKernels/OnePlus_KernelSU_SUSFS](https://github.com/WildKernels/OnePlus_KernelSU_SUSFS)
> 面向 **OnePlus 官方 (OnePlusOSS)** 内核树。本 ROM 的内核来自 **LineageOS/AOSP 树**，
> 所以构建流水线需要重新指向它。详见 [为什么要 fork](#-为什么-fork工作原理)。

---

## 🎯 目标

| | |
|---|---|
| **设备** | OnePlus 13 — 代号 **`dodge`**，SoC `sun` / **SM8750**（骁龙 8 Elite） |
| **ROM** | `PixelOS_dodge-17.0-20260912-0535`（**Android 17**） |
| **内核** | **`6.6.142-4k-gc568e18c7f62`** — 必须与 ROM 完全一致 |
| **源码** | `LineageOS/android_kernel_oneplus_sm8750` @ `c568e18c7f62` + ROM 自带 `.config`（从其 `boot.img` 提取） |
| **工具链** | ZyC **clang 22**（clang 19 会静默产出无法启动的内核） |
| **Vermagic** | `6.6.142-4k-gc568e18c7f62 SMP preempt mod_unload modversions aarch64` |

⚠️ 本内核**仅**适用于上述组合。刷到其他 ROM/内核版本会导致模块加载失败
（release string 是模块 ABI 的一部分）。

## ✅ 已支持功能

| 功能 | 状态 |
|---|---|
| **KernelSU-Next**（versionCode `33239`） | ✅ |
| 配套 **Manager APK**（`33239`），随 Release 发布 | ✅ |
| **SUSFS** v2.2.0（+ 用户态模块） | ✅ |
| **BBG** — Baseband Guard（保护非用户分区） | ✅ |
| **Unicode 绕过修复**（非打印字符路径穿越） | ✅ |
| **IP_SET + IPv6 NAT**、**TTL 目标** | ✅ |
| **zram**：内置，**LZ4** 默认，ZSTD 可用，多压缩流 | ✅ 真机验证（`[lz4]`，6 GB swap） |
| **CVE-2026-89839 / 80830 / 80842**（f2fs / USB hub / bridge） | ✅ 已打补丁**并编译进镜像**（`=y` 子系统） |
| **CVE-2026-93235**（f2fs 扩缩时 EOF 后残留数据） | ✅ 已移植，**真机复现并修复**（见下文） |
| **CVE-2026-90255 / 93209 / 80762**（内核 BT 核心） | ➖ **本设备不适用** — 见 [蓝牙](#-本-rom-上的蓝牙) |
| Droidspaces + NTSync | ✅ **仅 `OP13-full`** — 真机验证通过（模块集与 lite 一致，`0` CRC 不匹配） |
| **Re:Kernel** v11.7（tombstone / freeze 支持） | ✅ **编译进镜像** — dmesg 中 `Re-Kernel hooked!` |
| **ADIOS** I/O 调度器 | ✅ **内置并设为默认**（所有块设备 `[adios]`） |
| NetHunter + rtw88 / monitor-mode 注入 | ⏳ 计划中（full） |
| KPM (KernelPatch Next) | ⏳ 计划中（full） |
| LZ4KD zram 算法（实验性） | ⏳ 计划中（full） |
| HMBIRD (OnePlus fengchi SCX) | ❌ **不可行** — 该调度器源码不在此树中 |

## 🔀 两个变体 — `OP13-lite` 和 `OP13-full`

构建矩阵由**每一个** `configs/**/*.json` 生成，所以一次 `op_model=android15-6.6`
会同时构建两个独立 job：

| 配置 | 内容 |
|---|---|
| [`configs/a16/OP13-lite.json`](configs/a16/OP13-lite.json) | 上面所有 ✅ 功能。**日常驱动版本**，Release 发布的就是它。 |
| [`configs/a16/OP13-full.json`](configs/a16/OP13-full.json) | lite **+ Droidspaces (`ds`) + NTSync (`ntsync`)**，后续加 NetHunter/rtw88、KPM、LZ4KD — 每轮加一组。 |

⚠️ `full` **不是**刷了就能用：Droidspaces 翻开了五个原版 ROM 关闭的选项
（`CONFIG_SYSVIPC`、`CONFIG_POSIX_MQUEUE`、`CONFIG_USER_NS`、`CONFIG_PID_NS`、`CONFIG_DEVTMPFS`），
其中 `CONFIG_SYSVIPC` 会改变 `struct ipc_namespace` 的布局。由于模块 CRC（genksyms）
对每个导出原型的传递类型闭包做哈希，一个 Kconfig 翻转就可能让 `vendor_dlkm` 的
`.ko` 静默无法加载——`CONFIG_BPF_STREAM_PARSER` 就是这样搞坏 `bluetooth.ko` 的。

所以 `full` 构建必须通过**模块加载对比**才接受：刷入后检查 dmesg 没有
`disagrees about version of symbol`，且已加载模块集没有缩水
（`lite-r13` 基线：589 个 `.ko` 中 548 个加载成功：21 个是我们故意黑名单的调试模块，
`zram`/`zsmalloc` 已内置进镜像，剩下约 20 个——`can_*`、`tls`、`video`、`icnss2`、
`kheaders` 等——在这台设备上根本不会加载）。回滚只需一次刷机——保留 `lite` 的 AK3 包，
或用 ROM 自带 `boot.img` 执行 `fastboot flash boot_a/b`。

**结果（2026-10-07，`OP13-full` 真机）：** dmesg 无 `disagrees about version of symbol`，
已加载模块集与 `lite-r13` **逐字节同名**（548/589）。Wi-Fi（已关联 + 拿到 DHCP）、
蜂窝数据（NR，已验证）、蓝牙（`State: ON`）、NFC（`mState=on`）、相机 HAL（5 个设备）
全部正常；`logcat -b crash` 为空；无 avc 拒绝涉及 `ntsync`/`sysvipc`/命名空间。
`lite` 功能也都在：`[adios]` 仍是默认调度器，`baseband_guard` 在 LSM 链中，
`Re-Kernel hooked!`，zram0 6 GB `[lz4]`。

> 自己验证刷机时有两点值得注意：
> - `/proc/config.gz` 是**伪造的**（"Fake config.gz" 补丁让它匹配原版 ROM），所以
>   要通过运行时接口确认新功能：`/proc/sysvipc/{msg,sem,shm}`、`/dev/ntsync`
>   （ROM 的 `file_contexts` 将其标记为 `gpu_device`）、`/proc/sys/user/max_user_namespaces`。
> - 恢复界面显示"成功"不算数。检查槽位：`sha256sum /dev/block/by-name/boot_a`
>   对比 `boot_b` 对比 ROM 的 `boot.img`，并对比 `cat /proc/version` 的构建时间戳
>   与你从 workflow run 下载的 `Image` 中嵌入的时间戳。

## 📋 TODO

- [x] 从 ROM 自己的内核树构建，使 `uname -r` / vermagic 完全匹配
- [x] 追踪早期启动静默挂起 → 根因是**编译器版本**
- [x] KernelSU-Next `33239` + 精确 versionCode 的 Manager APK（不再报 *"kernel update required"*）
- [x] SUSFS v2.2.0 + BBG
- [x] `unicode`、`ip_set`、`ttl`
- [x] zram 内置，LZ4 默认 + 多压缩流 — **真机验证**
- [x] CVE 回溯 `89839 / 80830 / 80842` — **已打补丁*并*编译进镜像**（`=y` 子系统）
- [x] CVE `90255 / 93209 / 80762` — **本设备不适用**：它们在*内核* BT 核心中，本 ROM 从不用（见下文）
- [x] CVE-2026-93235（f2fs：扩展文件大小时零化 EOF 后数据）— **已修复**。我们的 f2fs 是 AOSP/OnePlus 变体，从来没有上游的 `f2fs_zero_post_eof_page()`，stable 回溯补丁全部不适用（五个分支每个 hunk 都失败）。`patches/cve/CVE-2026-93235.patch` 在上游修复后实现之上移植，基于现有 `fill_zero()`，从相同八个位置调用。先用 zram 支持的 f2fs 设备做 raw-PBA 测试复现，再验证修复：修复前 `truncate-up` 暴露 `[4080,4096)` 中的旧 `0x5a` 字节，修复后该区间读零。见补丁注释中唯一一处有意偏离上游之处（`/data` 挂载为 `fsync_mode=nobarrier`，所以零化不能门控在 strict 模式或缓存页上）。
- [x] Re:Kernel — **编译进内核镜像**（`drivers/android/` 中的 `obj-y`）。`.ko` 路线在此行不通：设置 `O=` 后，kbuild 的 `Makefile.modfinal` 没有最终 `.ko` 的规则（`No rule to make target rekernel.ko, needed by '__modfinal'`）。内置还消除了对 `kallsyms_lookup_name` 导出的运行时依赖。
- [x] ADIOS — `patches/adios/adios-6.6.patch`（6.12→6.6：只有 `elevator_find_get(q, name)` 和 `!blk_queue_nonrot(q)` 不同），内置并设为默认
- [x] **lite** = KSU + SUSFS + BBG + unicode/ip_set/ttl + zram + Re:Kernel + ADIOS — **真机验证**：`[adios]` 默认、zram `[lz4]`、`baseband_guard` 在 LSM 链中、`Re:Kernel v11.7 … Re-Kernel hooked!`
- [x] **full** 第 1 轮 — Droidspaces + NTSync：构建为 `OP13-full`，已刷入，**真机验证**（见[两个变体](#-两个变体--op13-lite-和-op13-full)）
- [x] 将构建矩阵拆分为 `OP13-lite` / `OP13-full` 配置（见[两个变体](#-两个变体--op13-lite-和-op13-full)）
- [x] ~~切换到 BakaSU/SukiSU 作为 root 方案~~ — **放弃**：SUSFS 补丁无法应用到 BakaSU 的树（94/97 个 hunk 失败），且 SUSFS 不可妥协
- [x] ~~HMBIRD~~ — **放弃**：此树中无 fengchi SCX 源码
- [ ] **full** 第 2 轮 — NetHunter + rtw88 / monitor-mode 注入（风险最高，单独一轮）
- [ ] **full** 第 3 轮 — LZ4KD zram 算法（实验性）
- [ ] KPM (KernelPatch Next) — **需要决策**：它是第二个 root 方案，不只是另一个补丁
- [ ] **`opt` 优化补丁逐个测试**：当前 `"opt": false` 全部跳过。WildKernels 上游补丁针对 OnePlusOSS 树，本树是 LineageOS/AOSP 树，全开会卡 logo。应逐个启用测试，安全的留下，卡 logo 的丢弃。优先测试低风险补丁：`reduce_gc_thread_sleep_time`、`silence_irq_cpu_logspam`、`increase_sk_mem_packets`、`reduce_freeze_timeout`
- [ ] **Droidspaces + NTSync 合入 lite**：当前只在 `OP13-full` 里，如日常需要可移到 `OP13-lite`
- [ ] **zram 压缩算法测试 ZSTD**：当前 LZ4 最快但压缩率最低，ZSTD 压缩率高 30-40%，swap 空间更大
- [ ] **源码下载缓存**：当前每次构建都重新下载内核源码，可用 `actions/cache` 按 manifest revision 缓存
- [ ] **考虑 sccache 替代 ccache**：sccache 为 CI 设计，并行性更好，迁移成本低

### 已确认不做 / 不需要

| 方向 | 结论 | 原因 |
|---|---|---|
| `opt: true` 全开 | ❌ | 卡 logo，LineageOS 树与补丁不兼容 |
| BBR 日用 | ❌ | 仅热点/高延迟网络有益，日用无收益 |
| ThinLTO | ❌ | 模块 CRC 风险 > 1-3% 收益 |
| SCX 调度器移植 | ❌ | 6.6 无 SCX 框架，backport 工作量巨大 |
| HMBIRD (fengchi) | ❌ | OnePlus 闭源 OEM 调度器，源码不可得 |
| 编译器额外标志调优 | ❌ | ZyCromerZ clang 22 已足够新，内核非计算密集型 |
| ADIOS I/O 调度器 | ✅ 已启用 | 块 I/O 层面已覆盖，与 CPU 调度器不冲突 |

## 📶 本 ROM 上的蓝牙

蓝牙**能用**，但不走内核：QTI HAL
（`android.hardware.bluetooth@aidl-service-qti`）持有 `/dev/ttyHS0`，在**用户态**说
**H4**（Fluoride/GD 栈在 `com.android.bluetooth` 内实现 L2CAP/RFCOMM）。
`system_dlkm` 的 GKI BT 核心模块（`bluetooth.ko`、`hci_uart.ko` 等）在这里是
**无用的参考模块**——在这个内核上它们甚至加载失败，无害：

```
bluetooth: disagrees about version of symbol sk_filter_trim_cap (err -22)
```

这个 CRC 分歧来自 `CONFIG_BPF_STREAM_PARSER`——上游 *ip_set* 步骤会打开它，
尽管 ROM 出厂是关的（`sk_filter_trim_cap` 的原型带 `struct sock *`，
所以 modversions CRC 覆盖该类型的闭包）。构建现在对 `wild/sm8750` 保持它关闭，
每次构建都会打印 `CONFIG PARITY vs ROM baseline` 差异，让静默配置漂移现形。

**结论：** CVE-2026-90255 / 93209 / 80762 针对 `net/bluetooth/*.c`，
即本设备不执行的代码路径——为了"修"它们而把 BT 核心编进镜像只会加入从不运行的代码。
记录为 N/A 而非打补丁。

## 🧪 验证 f2fs EOF 后修复（CVE-2026-93235）

复现遵循上游自己的 fstests `generic/794` 序列，但用 **zram 支持**的
f2fs 设备，不碰 `/data`：

```
创建 zram 设备 (echo N > /sys/class/zram-control/hot_add)，make_f2fs，挂载
写入 1 MiB 的 0x5a，文件偏移 16 处放唯一标记
扫描设备找文件第一个块
截断文件到 4080，卸载
向该块直接写入 4096 个 0x5a      # CVE 所说的"残留字节"
重新挂载，扩展文件到 8192
读 [4080,4096)  ->  全零 = 已修复，0x5a =  vulnerable
```

复现者注意：loop-over-file 设备在此行不通——内核*不允许*写普通文件
（`u:r:kernel:s0` 对 backing file 得到 EIO），会留下无法卸载的 f2fs 并卡死设备。
用 zram，每个 `umount` 都加 `timeout` 保护，绝不留下损坏的挂载。

## 🔍 在设备上验证构建

`/proc/config.gz` 全局可读，最快的审计方式：

```sh
adb pull /proc/config.gz && zcat config.gz | grep -E 'ZRAM_DEF_COMP|KSU|BBG|LTO_NONE'
adb shell 'cat /sys/kernel/security/lsm'          # 需要 root shell：期望 baseband_guard
adb shell 'cat /proc/swaps; cat /sys/block/zram0/comp_algorithm'   # 期望 [lz4]
adb shell 'uname -r'                              # 期望 6.6.142-4k-gc568e18c7f62
adb shell 'cat /sys/block/sda/queue/scheduler'    # 期望: none mq-deadline kyber [adios] bfq
adb shell 'dmesg | grep -iE "Re-Kernel"'          # 期望: Re:Kernel v11.7 … / Re-Kernel hooked!
```


## 📥 安装

1. 从[最新 Release](../../releases) 下载两个资产：
   - AnyKernel3 内核包
   - 配套 **KernelSU-Next Manager APK**（`…_33239-release.apk`）
2. 备份当前 boot 镜像。
3. 在 OrangeFox recovery（或 KernelFlasher）中刷入内核包。只写**活动槽位的 `boot`
   分区**；A/B 自动处理。
4. 安装**与内核相同 versionCode** 的 Manager APK（更新 = *"kernel update required"*）。
5. 为 SUSFS 隐藏，通过 Manager 安装用户态模块 `sidex15/ksu_module_susfs`（v2.2.0）。

**回滚：** 用 ROM 自带 boot 镜像执行 `fastboot flash boot_a boot.img` 和 `fastboot flash boot_b boot.img`。

## 🔧 为什么 fork / 工作原理

- **源码**：ROM 内核是 `LineageOS/android_kernel_oneplus_sm8750` @ `c568e18c7f62`；
  上游只从 OnePlusOSS 树构建，版本止于 6.6.118。
- **版本字符串**：action 被修补使 `LOCALVERSION` 固定为 `-4k-gc568e18c7f62`，
  LTO 强制为 `LTO_NONE`（与 ROM 一致），并移除 `-mcpu=oryon-1`。
- **启动挂起**：每个用 ZyC **clang 19** 构建的 6.6.142 重建都在第一个 logo 处静默挂起——
  无报错、vermagic 正确、补丁干净。切换到 **clang 22** 后修复。如果你重建，
  不要降低编译器版本。
- **SUSFS** 以 vendor 形式放入仓库（`vendor_susfs4ksu/`），因为 GitHub runner 上
  GitLab archive 下载被屏蔽。
- **补丁**放在 `patches/`（如已验证的 CVE 回溯在 `patches/cve/`）。
- **AK3**：`boot` 版本检查已禁用（`do.check_boot_version=0`），因为它只识别
  `-androidNN` 风格版本字符串，本 ROM 不用。

## 🙏 致谢

[WildKernels](https://github.com/WildKernels)（流水线）、[KernelSU-Next](https://github.com/KernelSU-Next/KernelSU-Next)、
[tiann/KernelSU](https://github.com/tiann/KernelSU)、[simonpunk/susfs4ksu](https://gitlab.com/simonpunk/susfs4ksu)、
[vc-teahouse/Baseband-guard](https://github.com/vc-teahouse/Baseband-guard)、[sidex15](https://github.com/sidex15)、
[ZyCromerZ/Clang](https://github.com/ZyCromerZ/Clang)，以及 LineageOS/PixelOS 维护者。

## ⚠️ 免责声明

刷入本内核会修改你的设备。先备份。风险自负——无保修，不背锅。