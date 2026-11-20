# panvk-JM-kirin

专为麒麟（Kirin）机型维护的 PanVK **JM 后端**，代码基于 [JimVulkan/mali-panvk](https://github.com/JimVulkan/mali-panvk)（Mesa 27.0.0-devel，保留 JimVulkan 版权头），只保留 job manager 路径（Mali v6 / v7 / v9），已删除全部 CSF（arch ≥ 10）代码。

本仓库只放源码，**不提供任何预编译包**（zip / so 均不在仓库内）。

English: PanVK job-manager backend for Kirin devices, based on [JimVulkan/mali-panvk](https://github.com/JimVulkan/mali-panvk) (Mali-G78 and other pre-CSF Valhall/Bifrost GPUs on Arm kbase). Source only, no prebuilt packages.

## 适用机型

| 项目 | 要求 |
| --- | --- |
| GPU | Arm Mali Bifrost / Valhall 的 JM 型号，arch 6 / 7 / 9 |
| 内核驱动 | 原生 Arm kbase（`/dev/mali0`），不是 panthor |
| kbase UAPI | 原则上任意版本；实测 11.38。更老的 UAPI 会自动回退到 64 / 48 字节原子布局 |
| 系统 | Android 12+（API 31），arm64 |
| 包装层 | Winlator / AdrenoTools 等直接加载 `libvulkan_panfrost.so` 的环境 |

实测机型：**华为 / 荣耀麒麟 Mali-G78 MC24**（Kirin 9000 系列，arch 9，csf 0，kbase UAPI 11.38）。在该机上游戏已出画面。

**不适用**：arch ≥ 10 的 CSF GPU（Mali-G610 / G710 / G715 / G720、Immortalis 系列等）。这类 GPU 请使用 CSF 后端，本仓库不含相关代码，也无法在本仓库上得到支持。

## 相对通用 kbase 后端的改动

这台麒麟（UAPI 11.38）暴露出三处与通用实现不一致的地方，均已修复：

1. **MEM_IMPORT 请求尺寸为 32 字节**
   华为内核的 `kbase_ioctl_mem_import` 参数结构是 32 字节；按通用的 24 / 40 字节发请求只会得到 `ENOTTY`(25) / `EINVAL`(22)，表现为 dma-buf 全部导入失败。驱动启动时依次探测 24 / 40 / 32 / 48 / 56，记住可用尺寸并在后续导入中复用。

2. **原子是 56 字节的 `base_jd_atom_v2`**
   该内核只接受 56 字节 stride；64 字节的两种布局（seq_nr-first、v2+frame_nr）都会被以 `EINVAL` 拒绝，提交静默失败、画面全黑。驱动现在：
   - 先按内核 UAPI 版本排序候选布局（UAPI ≥ 11 先试 56 字节）；
   - 用一个只做依赖、不进 GPU 的原子探测，并用完成事件里原样回传的 tag 校验字段偏移；
   - 探测全部被拒时，首批真实提交会再试其余布局，内核接受哪个就永久切换（EINVAL 发生在入队前，不会重复执行工作）。

3. **ANB 多 fd 回退**
   设备 gralloc 布局未知时（`ARM gralloc: unknown handle layout`），`u_gralloc` 选不出正确的 dma-buf fd。导入现在会遍历 native handle 内所有有效 fd，逐个尝试直到内核接受。

## WSI

使用标准 `wsi_common` 路径（`src/panfrost/vulkan/panvk_wsi.c` 只是 common 的薄封装）。

## 构建

Linux / WSL，需要 Android NDK（API 31 arm64）、meson、ninja、glslangValidator，以及带 clang / libclc / SPIR-V 翻译器的 LLVM：

```sh
./build.sh /path/to/android-ndk
```

产物为 `dist/mali-panvk-27.0.0-devel.zip`，仅本地使用；本仓库不发布任何包。

## 日志

驱动日志需要在启动环境里设置：

```sh
MESA_LOG=file,android
MESA_LOG_FILE=/sdcard/mesa.log
PANVK_DEBUG=startup
```

在麒麟这台机上，正常启动应看到：

```text
kbase: kernel UAPI 11.38
kbase: atom layout: 56-byte atoms, tag back in udata[0]
kbase: MEM_IMPORT uses a 32-byte request
```

如果 `atom layout` 之后还有 `atom layout: 64-byte atoms refused`，说明内核 UAPI 与实测机不同，属预期探测，不影响后续布局选择。

## 来源

本仓库的驱动代码基于 [JimVulkan/mali-panvk](https://github.com/JimVulkan/mali-panvk)。在其代码之上删除了 CSF 后端、只保留 JM（v6 / v7 / v9），并加入上面列出的麒麟适配；版权与许可仍归原作者与 Mesa 项目所有。

## 许可

Mesa 标准 MIT 许可，见 `LICENSE` 与 `docs/license.rst`。第三方组件见 `THIRD_PARTY_NOTICES.md`。
