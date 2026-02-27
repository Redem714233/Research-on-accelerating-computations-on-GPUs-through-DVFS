# GPU 能效优化研究：基于 DVFS 的性能保障策略

本项目实现了 **GEEPAFS**（GPU Energy-Efficient and Performance-Assured Frequency Scaling），一种通过动态电压频率调节（DVFS）提升 GPU 能效的应用透明策略。

## 项目概述

GEEPAFS 通过在线建模应用性能与 GPU 内存带宽利用率的关系，动态调整 GPU 频率以最大化能效，同时保证性能损失在可控范围内（如 10%）。

**核心特性：**
- ⚡ **性能保障**：性能损失可控（平均 5.8%，最差 12.5%）
- 🔄 **应用透明**：无需离线分析或修改应用代码
- 💡 **显著节能**：V100 上平均提升 26.7% 能效，A100 上提升 20.2%

## 项目结构

```
.
├── DVFS/
│   ├── geepafs/              # GEEPAFS 核心实现
│   │   ├── dvfs.c            # DVFS 策略主程序（C/NVML）
│   │   ├── runExp.py         # 实验运行脚本
│   │   ├── postprocessing.py # 数据后处理
│   │   ├── compare_dvfs.py   # 策略对比分析
│   │   └── cuda_samples/     # CUDA 基准测试程序
│   ├── paper.md              # 论文内容（Markdown）
│   └── 论文.pdf              # 论文 PDF
├── output/                   # 实验输出结果
└── 2025秋_DVFS_创新实践报告.pdf
```

## 支持的 GPU

- NVIDIA V100 (163W / 300W TDP)
- NVIDIA A100 (400W TDP)
- NVIDIA RTX 4070 Laptop

其他 GPU 需修改 `dvfs.c` 中的频率参数配置。

## 快速开始

### 1. 编译 DVFS 程序

```bash
cd DVFS/geepafs
make
```

> 注意：可能需要修改 `Makefile` 中的 `CUDA_PATH`

### 2. 运行 DVFS 策略

```bash
# 需要 root 权限
sudo ./dvfs mod Assure p90
```

**可用策略：**
- `Assure`：性能保障策略（推荐）
- `MaxFreq`：最大频率（基线）
- `EfficientFix`：固定高效频率
- `UtilizScale`：利用率比例调节
- `NVboost`：NVIDIA Boost 默认策略

### 3. 运行完整实验

```bash
cd DVFS/geepafs
sudo python3 runExp.py
```

实验结果将保存到 `output/` 目录。

## 实验分析

项目提供多个分析脚本：

```bash
# 后处理实验数据
python3 postprocessing.py

# 对比不同 DVFS 策略
python3 compare_dvfs.py

# 敏感性分析
python3 analyze_sensitivity.py

# 生成可视化图表
python3 make_sweep_plots.py
```

## 论文信息

**标题：** Improving GPU Energy Efficiency through an Application-transparent Frequency Scaling Policy with Performance Assurance

**作者：** Yijia Zhang, Qiang Wang, Zhe Lin, Pengxiang Xu, Bingqiang Wang

**会议：** EuroSys '24, April 22–25, 2024, Athens, Greece

**DOI：** [10.1145/3627703.3629584](https://doi.org/10.1145/3627703.3629584)

## 依赖环境

- CUDA Toolkit（含 NVML）
- Python 3.x
- GCC/G++
- Root 权限（用于频率调节）

## 许可证

MIT License - 详见源代码文件头部

## 引用

如果本项目对你的研究有帮助，请引用原论文：

```bibtex
@inproceedings{zhang2024geepafs,
  title={Improving GPU Energy Efficiency through an Application-transparent Frequency Scaling Policy with Performance Assurance},
  author={Zhang, Yijia and Wang, Qiang and Lin, Zhe and Xu, Pengxiang and Wang, Bingqiang},
  booktitle={Proceedings of the Nineteenth European Conference on Computer Systems},
  year={2024},
  pages={},
  doi={10.1145/3627703.3629584}
}
```
