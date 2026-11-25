# panvk-JM-kirin

专为麒麟（Kirin）机型维护的 PanVK **JM 后端**，代码基于 [JimVulkan/mali-panvk](https://github.com/JimVulkan/mali-panvk)（Mesa 27.0.0-devel，保留 JimVulkan 版权头），只保留 job manager 路径（Mali v6 / v7 / v9），不包含 CSF（arch ≥ 10）后端。

本仓库以源码为主；正式预编译包发布在 [Releases](https://github.com/MineGraph-bilibili/panvk-JM-kirin/releases)。

English: PanVK job-manager backend for Kirin devices, based on [JimVulkan/mali-panvk](https://github.com/JimVulkan/mali-panvk) (pre-CSF Valhall/Bifrost GPUs on Arm kbase). Source-first; prebuilt packages are in Releases.

## 正式发布包

| 项目 | 内容 |
| --- | --- |
| 版本 | `1.0.0`（Mesa 27.0.0-devel，JM v6/v7/v9） |
| 文件 | `mali-jm-27.0.0-devel.zip` |
| 包版本 | `packageVersion = 3` |
| SHA256 | `a301caa77e78f39b38ebbf86d47e1f833e44c358ab2e331f4998f186bd3fcccf` |
| 安装 | 在 Winlator 的驱动设置中导入 zip，选中 `mali-jm (Mesa 27.0.0-devel)`，重启容器 |

## 支持的机型（仅 JM 后端 GPU）

本包只构建 JM 后端（Mali Bifrost / Valhall，arch 6 / 7 / 9），不会加载 CSF 后端代码。

| 后端 | GPU | 麒麟平台示例 |
| --- | --- | --- |
| v9 | Mali-G78 | Kirin 9000 / 9000E |
| v7 | Mali-G76 / G77 / G57 | Kirin 990 / 990 5G / 985 / 980 / 820 |
| v6 | Mali-G72 / G52 / G51 / G31 | Kirin 970 / 810 / 710 / 710F |

其他要求：

| 项目 | 要求 |
| --- | --- |
| 内核驱动 | 原生 Arm kbase（`/dev/mali0`），不是 panthor |
| kbase UAPI | 11.x；实测 11.38。更老的 UAPI 会自动回退到 64 / 48 字节原子布局 |
| 系统 | Android 12+（API 31），arm64 |
| 包装层 | Winlator / AdrenoTools 等直接加载 `libvulkan_panfrost.so` 的环境 |

实测机型：**华为 / 荣耀麒麟 Mali-G78 MC24**（Kirin 9000 系列，arch 9，csf 0，kbase UAPI 11.38）。

**不支持**：CSF 后端的 GPU（arch ≥ 10）不在本包范围；非 Mali 的 GPU（例如 Kirin 9000S / 9010 的 Maleoon）也不支持。

## 优化与改动

1. **干净的 JM 提交路径**
   基于 `JimVulkan/mali-panvk` 的上游 JM 代码，移除了本地 fault-hardening / draw / 诊断补丁，提交与故障处理行为回到上游。

2. **MEM_IMPORT 尺寸探测**
   华为内核的 `kbase_ioctl_mem_import` 参数结构是 32 字节；通用的 24 / 40 字节请求只会得到 `ENOTTY`(25) / `EINVAL`(22)，表现为 dma-buf 全部导入失败。驱动按 24 / 40 / 32 / 48 / 56 依次探测，记住内核接受的尺寸并在后续导入中复用。

3. **56 字节 `base_jd_atom_v2`**
   该内核只接受 56 字节 stride；64 字节的两种布局（seq_nr-first、v2+frame_nr）都会被 `EINVAL` 拒绝，提交静默失败、画面全黑。驱动现在：
   - 按内核 UAPI 版本排序候选布局（UAPI ≥ 11 先试 56 字节）；
   - 用一个只做依赖、不进 GPU 的原子探测，并用完成事件原样回传的 tag 校验字段偏移；
   - 探测全部被拒时，首批真实提交再试其余布局，内核接受哪个就永久切换（`EINVAL` 发生在入队前，不会重复执行工作）。

4. **ANB 多 fd 回退**
   设备 gralloc 布局未知时（`ARM gralloc: unknown handle layout`），`u_gralloc` 选不出正确的 dma-buf fd。导入会遍历 native handle 内所有有效 fd，逐个尝试直到内核接受。

5. **X11 WSI dlopen 垫片**
   X11 WSI 通过运行时 `dlopen("libxcb.so.1")` 访问 xcb，不再把 xcb 链进驱动。生成的 `libvulkan_panfrost.so` 没有未定义的 `xcb_*` 符号，避免 Android 加载失败后回退到系统驱动。

6. **构建优化**
   `-Os`、关闭 `panfrost-disassemble`，只编译 JM v6 / v7 / v9；正式包体积比全量构建明显减小。

## 构建

Linux / WSL，需要 Android NDK（API 31 arm64）、meson、ninja、glslangValidator，以及带 clang / libclc / SPIR-V 翻译器的 LLVM：

```sh
./build.sh /path/to/android-ndk
```

产物为本地的 `dist/mali-panvk-27.0.0-devel.zip`；正式包见 Releases。

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
