# pstar KernelSU-Next + SUSFS（基于 lss4/KernelSU_Action_pstar）

面向 **Motorola Edge S Pro (pstar)**、**LineageOS 23.2**、内核 **4.19.325** 的 GitHub Actions 定制方案。

## 你需要知道的硬限制

1. **pstar 是非 GKI（4.19）**  
   不能用 GKI 的 LKM / 预编译 boot；必须把 KernelSU-Next 驱动和 SUSFS **编进内核**。

2. **KernelSU-Next 上游变动很大**  
   - 默认分支已是 `dev`（面向 GKI）。  
   - 非 GKI 用 `legacy` 分支。  
   - 旧的 `next` 分支、`legacy-susfs` tag **已经不存在**。  
   - 项目一原 `misused.env` 里的 `KERNELSU_SETUP=.../next/kernel/setup.sh` 和 `KERNELSU_TAG=legacy-susfs` 会直接失败。

3. **SUSFS 不是“装个模块就行”**  
   - 内核补丁必须编进内核。  
   - Manager 里的 susfs 用户态模块只负责配置。  
   - **KSU 侧补丁版本** 和 **内核侧 SUSFS 版本** 必须一致。  
   - 项目一原配置是 KSU 侧 2.0.0 + 内核侧 2.1.0，接口对不上，会编不过。本方案统一为 **2.0.0**。

4. **“最新”在非 GKI 上不等于“能编过”**  
   `legacy` 目录结构多次重组后，`10-ksun-susfs-2.0.0-test.patch` 可能打不上。  
   若 Apply KSU-side SUSFS 步骤失败，把 `KERNELSU_TAG` 改成能打上补丁的 **固定 commit SHA**（见下方排错）。

## 文件说明

| 文件 | 作用 |
|------|------|
| `misused.env` | 设备 / KSU / SUSFS 配置（已修正 setup URL、版本配对） |
| `.github/workflows/misused.yml` | 完整构建流水线（基于项目一 misused，修正关键步骤） |
| `shared.env` | 与项目一共用工具链 / 附加补丁开关（可直接用上游的） |

## 使用步骤

1. Fork [lss4/KernelSU_Action_pstar](https://github.com/lss4/KernelSU_Action_pstar)，分支选 `Lineage_pstar`。
2. 用本目录的 `misused.env` **覆盖** 仓库根目录的 `misused.env`。
3. 用本目录的 `.github/workflows/misused.yml` **覆盖** `.github/workflows/misused.yml`。
4. 保留仓库原有的 `shared.env`、`patches/`（若有）、以及 Actions 依赖的其它文件。
5. 打开 Actions → **(pstar) (MiSuSed) ...** → **Run workflow**。
6. 成功后下载 artifact：`AnyKernel3...` zip。
7. 在 **Lineage Recovery** 或 TWRP 中刷入该 zip（**不要**把裸 `Image` 直接 `fastboot flash boot`）。
8. 开机后安装：
   - 与内核内嵌驱动匹配的 **KernelSU-Next Manager**（版本尽量接近驱动，过新可能不认旧驱动）。
   - 用户态 [susfs4ksu-module](https://github.com/sidex15/susfs4ksu-module)（需内核已编入 SUSFS）。

## 刷机注意

- 先备份 `boot_a` / `boot_b`。  
- 测试优先 `fastboot boot AnyKernel 解包后的 boot`（若设备支持），确认能开机再永久写入。  
- `Remove_Kernel_Validation=true` 会放宽模块版本检查，利于自编内核加载 vendor 模块；若仍重启循环，检查 `LOCALVERSION` / vermagic 是否与当前 Lineage 一致。

## 排错：KSU 侧 SUSFS 补丁打不上

日志类似：

```text
patching file ...
Hunk #N FAILED at ...
```

说明当前 `legacy` 的目录结构已变。处理：

1. 在 Actions 日志里找到 `Setup KernelSU` 检出的 commit。  
2. 本地克隆 `KernelSU-Next` 的 `legacy`，用 `git log --oneline` 往回找，直到能成功：

   ```bash
   patch -p1 --dry-run < 10-ksun-susfs-2.0.0-test.patch
   ```

3. 把 `misused.env` 里 `KERNELSU_TAG=` 改成该 commit 的完整 SHA。  
4. 重新跑 workflow。

备选：换用补丁仓库里的 `10-ksun-susfs-2.0.0-maxsteeel-1.patch`（改 `SUSFS_PATCH=`），有时对某段 legacy 更合适。

## 排错：想用 SUSFS 2.1.0

需要 **同时** 具备：

- 内核侧：`50_add_susfs_in_kernel-4.19.325-2.1.0.patch` + `susfs-2.1.0/`
- KSU 侧：针对同一代 KernelSU-Next 源码写好的 2.1.0 对接补丁  

目前 lss4 公开的只有 **2.0.0** 的 KSU 侧补丁。硬改 `SUSFS_VERSION=2.1.0` 而不换 KSU 侧补丁，几乎必编不过。

## 与“最新 Manager / 最新 GKI SUSFS LKM”的关系

- GKI 设备上的 **SUSFS LKM 自动加载**（v3.4.2 一类）**不适用于** 4.19 non-GKI。  
- 你这边只能走 **内置驱动 + 内核 SUSFS 补丁**。  
- Manager 可以尽量新，但若提示驱动过旧，就需要把 `KERNELSU_TAG` 推到更新的 legacy（并重做/移植 SUSFS 对接补丁）。

## 参考

- 上游 Action：https://github.com/lss4/KernelSU_Action_pstar  
- 补丁仓：https://github.com/lss4/kernel_patches  
- KernelSU-Next：https://github.com/KernelSU-Next/KernelSU-Next （分支 `legacy`）  
- susfs 用户态模块：https://github.com/sidex15/susfs4ksu-module  
