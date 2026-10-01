# LoongArch SIMD 编译优化

本仓库收录 2026 年全国大学生计算机系统能力大赛编译系统设计赛（编译系统挑战赛）“面向 LoongArch 架构的 SIMD 编译优化”项目的 LLVM 补丁与设计文档。

项目以 `llvmorg-22.1.3` 为基线，面向龙芯 3A6000（LA664）及 LSX/LASX 向量扩展，在 `llvm/lib/Target/LoongArch/` 范围内增强自动向量化、目标代价模型和后端代码生成。

## 仓库内容

- [`patches/0001-LoongArch-SIMD-TTI-idiom-pass-pack-trunc-SATD-SWAR-c.patch`](patches/0001-LoongArch-SIMD-TTI-idiom-pass-pack-trunc-SATD-SWAR-c.patch)：可应用到 LLVM 源码树的 Git 补丁，涉及 11 个文件，新增 8,248 行、删除 36 行。
- [`docs/设计说明文档0819.pdf`](docs/设计说明文档0819.pdf)：项目设计说明、实现细节、性能数据、正确性验证与已知限制。
- [`docs/赛题技术方案.pdf`](docs/赛题技术方案.pdf)：赛事官方技术方案与提交要求。

## 主要优化

- 补全 LoongArch `TargetTransformInfo`（TTI）代价模型，覆盖算术、访存、混洗、类型转换、归约和 intrinsic 等路径。
- 在 Pass Pipeline 中接入目标相关优化，包括向量化前的 `LoongArchFPIndVarPass`、向量化后的 `VectorCombinePass` 与 `LoongArchVectorIdiomPass`。
- 改进 LSX/LASX 浮点向量归约，使用 VBSRL 树减少逐元素抽取和串行依赖。
- 优化整数截断打包、寄存器内扩展、SAD/SATD、SWAR、Hadamard、chroma 与 lowres 等模式。
- 增强交错访存、FMA 融合和向量 lane 0 的低成本提取。
- 针对二维列访问、浮点递推、min/max、手工展开循环等模式进行等价 IR 改写。

补丁中的两个目标专用 Pass 默认在非 `-O0` 优化流水线启用，可用隐藏选项进行消融测试：

```text
--loongarch-enable-vector-idiom=false
--loongarch-enable-fp-indvar=false
```

## 应用补丁

先获取与补丁一致的 LLVM 基线：

```bash
git clone --branch llvmorg-22.1.3 --depth 1 \
  https://github.com/llvm/llvm-project.git
cd llvm-project
git apply --check ../loongarch_SIMD/patches/0001-LoongArch-SIMD-TTI-idiom-pass-pack-trunc-SATD-SWAR-c.patch
git am ../loongarch_SIMD/patches/0001-LoongArch-SIMD-TTI-idiom-pass-pack-trunc-SATD-SWAR-c.patch
```

如果只希望应用工作区改动、不保留补丁提交信息，可将最后一条命令替换为 `git apply <补丁路径>`。

## 构建示例

以下命令给出一个常见的 Release 构建方式；可按本机工具链与并行度调整：

```bash
cmake -S llvm -B build -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_PROJECTS="clang" \
  -DLLVM_TARGETS_TO_BUILD="LoongArch"
cmake --build build
```

在 LoongArch 3A6000 上进行性能测试时，使用 `-O3 -mlasx` 启用 LASX 优化路径；也可使用 `-mlsx` 单独测试 LSX 路径。

## 评测结果

设计文档记录的龙芯 3A6000 实机结果如下：

- TSVC2 共 150 个计分 kernel，未封顶几何平均加速比为 `1.317193x`（约 `+31.72%`）。
- 将 s317 单项加速比封顶为 10 后，几何平均加速比为 `1.253963x`（约 `+25.40%`）。
- 基线总运行时间约 `957.1 s`，优化后约 `738.7 s`。
- TSVC2 checksum mismatch 为 0；赛方功能测试记录为 `546/546 Passed`。

上述数据来自项目设计文档，应在相同 LLVM 版本、硬件、系统和测试配置下复测后再作横向比较。

## 正确性与限制

- 改动限定于 `llvm/lib/Target/LoongArch/`，未修改目标无关的 Analysis、Clang Driver 或 LTO 代码。
- LoongArch 缺少原生掩码向量访存指令，因此补丁在 TTI 中显式拒绝或提高相关路径的代价。
- 交错访存重点覆盖 Factor 2/4；更高 factor 与交错写仍保留通用路径。
- 有符号向量平均尚未纳入惯用式改写；部分模式匹配采取保守策略，正确性优先。
- 补丁包含 AI 辅助生成内容。平台、模型、约束和工作方式已在补丁源码及设计文档中披露；合入生产环境或向上游提交前必须进行人工逐行审查和完整回归测试。

## 文件完整性（SHA-256）

```text
a4702d2205d51581539559fd594ef301260157a008451fe99d56a4edff8c5985  patches/0001-LoongArch-SIMD-TTI-idiom-pass-pack-trunc-SATD-SWAR-c.patch
d2b796a7c37cbdbbbda250040191e283e4ace225c52ab81ce42c2277debde019  docs/设计说明文档0819.pdf
19c04ecb8a7c03829e7437610d995849242ed54e67d410cedc923b1f3b795fa9  docs/赛题技术方案.pdf
```

## 参考资料

- [LLVM 22.1.3](https://github.com/llvm/llvm-project/releases/tag/llvmorg-22.1.3)
- [LoongArch ABI 规范](https://github.com/loongson/la-abi-specs)
- [全国大学生计算机系统能力大赛编译系统设计赛技术平台](https://compiler.xtnl.org.cn/)

## 许可说明

补丁修改的 LLVM 源码遵循上游项目的 `Apache-2.0 WITH LLVM-exception` 许可。仓库中的设计文档和赛事资料版权分别归其作者及相关权利方所有；除适用法律或权利方授权外，请勿将其视为已按 LLVM 许可证重新授权。
