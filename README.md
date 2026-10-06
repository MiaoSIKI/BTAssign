# BTAssign-R：Open3DBench K=2 复现

本仓库在 [BTAssign](https://github.com/Lusica1031/BTAssign) 公开会议版代码基础上，补充了面向 Open3DBench 的混合键合端子（HBT）分配与 OpenROAD 验证流程。

复现目标是期刊版本中的多终端设置：每条 3D net 最多分配两个 HBT（`K=2`），并比较：

- **BTAssign-WL**：以分层 MST 线长为目标；
- **BTAssign-R**：在 WL 解上加入 P-RUDY 路由需求优化，并保证代理线长和 P-RUDY 不退化。

> 期刊版 BTAssign-R 源码没有公开。本仓库中的 R 模式是根据论文公式与实验参数实现的 Open3DBench 防退化适配，不是作者期刊代码的逐行复制。

## 复现状态

测试用例：Open3DBench AES / cadb1021，1,492 条 3D nets，HBT pitch 为 2 um。

参数与论文保持一致的部分：

- 最大终端数 `K=2`；
- quadtree 停止尺寸 50 um；
- 两轮候选采样因子 8 和 4；
- `alpha=1`；
- 自适应 `beta = min(WL) / mean(P-RUDY)`；
- 第二终端使用公开源码中的 0.5 增量线长门限。

### 分配阶段代理指标

| 分配 | HBT 总数 | 第二 HBT | Layered MST | 相对初始解 | Mean P-RUDY |
| --- | ---: | ---: | ---: | ---: | ---: |
| Open3DBench 初始解 | 1,492 | 0 | 59,917.71 um | - | 0.034130 |
| BTAssign-WL | 1,912 | 420 | 55,153.00 um | -7.952% | 0.030697 |
| BTAssign-R | 1,912 | 420 | 54,783.58 um | -8.569% | 0.029977 |

R 相比 WL：MST 进一步降低 0.670%，P-RUDY 进一步降低 2.344%。1,912 个 HBT 全部唯一、网格对齐，间距违规为 0。

### FastRoute 全局布线结果

OpenROAD：`26Q2-1164-g08f67ee5ec`。

| 指标 | BTAssign-WL | BTAssign-R | R 相对 WL |
| --- | ---: | ---: | ---: |
| Routed wirelength | 288,921 um | 287,972 um | -0.3285% |
| Routing demand | 64,215 | 63,953 | -0.4080% |
| Number of vias | 137,753 | 137,959 | +0.1495% |
| Routing usage | 23.01% | 22.92% | -0.09 percentage point |
| Total overflow | 0 | 0 | 相同 |

线长、routing demand 和零 overflow 的趋势与论文一致；全局布线 via 数小幅增加。论文只报告 vias 在多数用例改善，并未保证所有用例均改善。

### 详细布线补充结果

| 指标 | BTAssign-WL | BTAssign-R | R 相对 WL |
| --- | ---: | ---: | ---: |
| Number of vias | 138,487 | 138,250 | -0.1711% |
| Final violations | 23,701 | 23,767 | +0.2785% |
| Detailed-route runtime | 9,605 s | 9,680 s | +0.7808% |

论文 Table 4 使用 FastRoute 指标；详细布线结果仅作为 Open3DBench 全流程补充。

## 仓库结构

```text
.
├── grt/                         # 上游 BTAssign-WL / FastRoute C++ 实现
├── parser/                      # 上游网表转换工具
├── data/                        # 上游数据入口
└── Open3DBench-reproduction/
    ├── code/
    │   └── third_party/Open3DBench/
    │       ├── Place-LoL/btassign.py
    │       └── OpenROAD-3D/flow/
    │           ├── run_btassign_aes.sh
    │           └── designs/.../config_btassign_{wl,r}_k2.mk
    ├── patches/
    │   └── Open3DBench-flow-Makefile.patch
    └── verification/
        ├── report.json
        └── server-results.md
```

更详细的修改清单见 [Open3DBench-reproduction/README.md](Open3DBench-reproduction/README.md)。

## 使用方法

### 1. 准备 Open3DBench

本仓库不重复提交完整 Open3DBench、OpenROAD、第三方依赖和生成结果。准备一个完整的 Open3DBench 工作目录后，将适配文件复制进去：

```bash
cp -a Open3DBench-reproduction/code/third_party/Open3DBench/. /path/to/Open3DBench/
```

在包含 `Open3DBench/` 的父目录应用 flow 补丁：

```bash
cd /path/to
patch -p0 < BTA-R/Open3DBench-reproduction/patches/Open3DBench-flow-Makefile.patch
```

Python 分配器需要 NumPy 和 SciPy。

### 2. 生成 K=2 分配

```bash
cd /path/to/Open3DBench
python3 Place-LoL/btassign.py --input Place-LoL/binaries/converted_input/default/aes.input --placement Place-LoL/binaries/iccad2022/cadb1021/output/default/aes.txt --def Place-LoL/binaries/converted_output/default/cadb1021/aes.def --output-dir Place-LoL/results/btassign/aes_cadb1021_k2 --budget 2
```

该命令同时生成 WL/R 的 assignments JSON、K=2 DEF 和 `report.json`。

### 3. 运行 OpenROAD

只运行论文口径的 FastRoute：

```bash
cd OpenROAD-3D/flow
./run_btassign_aes.sh wl grt
./run_btassign_aes.sh r grt
```

运行完整详细布线：

```bash
./run_btassign_aes.sh wl full
./run_btassign_aes.sh r full
```

WL 与 R 使用独立的 `FLOW_VARIANT` 目录，不会互相覆盖。

## 与论文实验的差异

本仓库复现了核心方法和定性趋势，但不是论文 Table 4 的同数据数值复刻：

- 当前 AES 只有 1,492 条 3D nets；论文 `tinyaes + cadb1021` 有 19,979 条；
- 当前 HBT pitch 为 2 um；论文为 1 um；
- 当前使用 Open3DBench metal1-metal20 流程；论文使用异构 Nangate15/Nangate45、13 层加 BT 层；
- BTAssign-R 为论文驱动的适配实现，作者期刊版源码未公开。

## 上游来源与引用

本仓库 fork 自 [Lusica1031/BTAssign](https://github.com/Lusica1031/BTAssign)。原始公开实现对应：

```bibtex
@inproceedings{liu2024routing,
  title={Routing-aware Legal Hybrid Bonding Terminal Assignment for 3D Face-to-Face Stacked ICs},
  author={Liu, Siting and Jiang, Jiaxi and He, Zhuolun and Wang, Ziyi and Lin, Yibo and Yu, Bei and Wong, Martin},
  booktitle={Proceedings of the 2024 International Symposium on Physical Design},
  pages={75--82},
  year={2024}
}
```

原项目与本 fork 均保留 [CU-SD License](LICENSE)。
