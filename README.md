# background performance drain

> 基于天玑9300+ 的**假调度**省电模块 v1.0.43 · 作者：偷吃性能

一个设计初衷**只为了蹭 scene 调度功能**所写的假调度模块，几乎保留了所有官调内容，又可以蹭 scene 的调度功能（核心分配、旁路充电）。虽是假调度，但包含调度基本的频率切换、cpuset、情景模式、GPU/DDR/MIGT 控制功能。

---

## 功能介绍

### 核心理念
> **高频低负载保帧率、低频高负载才耗电**

- **日常**：压 A720 能效核心频率，降低功耗
- **游戏**：压 GPU 频率上限降温降功耗，但保证稳态帧率不崩
- **目标**：日常省电稳定、游戏降温、帧率不崩

### 档位说明

| 档位 | CPU | GPU | DDR | 适用场景 |
|---|---|---|---|---|
| **powersave** | A720 压 1.6G，X4 1.2G 兜底 | 150~300M | 20.67G 低功耗 | 日常省电（v1.0.43 起日常可用） |
| **balance** | A720 压 1.6G 主力，X4 1.6G 兜底 | 150~300M | 20.67G 低功耗 | 日常 + 普通游戏（默认档） |
| **performance** | A720 满 + X4 1.7G | 260~650M | 锁 54.6G | 吃 GPU 的高负载游戏 |
| **fast** | 全核满频 | 400M~1.1G | 锁 68G | 竞技类极限性能 |

### 内置温度监控
模块内置独立温度守护进程，无需外部调度器参与：

| 状态 | 条件 | 动作 |
|---|---|---|
| 触发降频 | 温度 > **82°C** | 大核(policy7)降到 1.6G，中核(policy4)降到 1.8G（保留游戏可玩性） |
| 恢复 | 温度 ≤ **76°C** | 恢复到当前档位配置（带回差，防止频繁抖动） |

监控间隔 5 秒；正常时只读温度，不覆写 sysfs，避免额外唤醒。

---

## 使用方法

### 环境要求
- **必须**由 **scene** 或 **perapp-rs** 等外部调度管理器接管，否则无法生效
- 支持机型：天玑9300+（MT6989）；其他 SoC 需自行适配配置
- 需要 Magisk / KernelSU / APatch 类 Root 环境

### 安装与切换
1. 在 scene 中安装/导入本模块的 `powercfg`（模块开机时会自动生成 `/data/powercfg.json` 与 `/data/powercfg.sh`）
2. 通过 scene 切换档位，或在终端手动调用：

```sh
# 手动切换档位（balance / powersave / performance / fast / init）
sh /data/adb/modules/background_performance_drain/power_switch.sh balance
```

3. 配合 scene 的 **per-app 规则**：给游戏 App 绑定 performance/fast 档，日常 App 用 balance/powersave 档

### 日志
运行日志输出至：
```
/data/local/tmp/background_performance_drain.log
```
包含档位切换、节点写入失败、温度监控等关键事件，排查问题先看这里。

### 卸载
- 直接卸载模块即可，`uninstall.sh` 会后台清理 `/data/powercfg.json`、`/data/powercfg.sh` 及日志文件
- 若想保留 scene 里其他调度器，卸载后请在 scene 中重新指定其 powercfg

---

## 下载

- Release 页面：https://github.com/a298016220666/background_performance_drain/releases
- v1.0.43 直链：https://github.com/a298016220666/background_performance_drain/releases/download/v1.0.43/background_performance_drain_v1.0.43.zip

---
*本模块为假调度，核心能力依赖 scene 等外部管理器；仅供学习与个人使用。*
