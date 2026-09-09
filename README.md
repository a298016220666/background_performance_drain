# background performance drain v1.0.50
> 基于天玑9300+ 的假调度省电模块 v1.0.50（覆盖 v1.0.49）
> 作者：偷吃性能
## v1.0.50 修复：power_switch skip 分支误杀守护（2026-09-09 事故）
**事故现场**：05:34:39 日志 `already in mode 'performance', skip` 后 guard_state=stopped、
三个守护（guard/temp_monitor/game_monitor）全部消失，系统卡在旧档位无人守护。
**根因**：power_switch.sh 原版把 `kill_guard/kill_temp_monitor/kill_game_monitor` 放在
case 判断之前无条件执行。同档重复调用（mode 相同）时 kill 已先跑掉，skip 分支却不重建
守护 -> 系统停留在当前档位的峰值/低功耗状态且无任何守护进程兜底。
**修复**：
1. 幂等 skip 判断提前到 kill 之前，且要求 guard.pid 存在且存活才 skip（不杀守护）
2. mode 相同但 guard 已死时不再 skip，落入具体档位分支完整重建（自带自愈，可修复同类事故）

> 注：`dvfsrc_force_vcore_dvfs_opp` 为**写只读节点**（写入用 echo -n，读取返回空/IO error 属正常）。
> 实际生效值请查 `cat /sys/kernel/helio-dvfsrc/dvfsrc_dump`（看 FORCE_OPP_IDX / DDR / Vcore 三行）。

## v1.0.49 更新：game_monitor 游戏守护（仅 performance 档 + 白名单）
- 白名单 games.list 前台命中才激活：Fast 基座（cpu4=2.85G/cpu7=3.4G/GPU 1.1G/DDR auto）
- C 方案 rate_limit 跟手（温控>82°C 自动回 5000 us，否则 1000 us）
- E 方案 taskset 绑核：外挂脚本绑中核 0x0F、游戏线程大核 0xF0 互不抢资源
- 连续 2 次(1s) 非游戏自动回退；切档立即被杀并恢复 performance 原值
- balance/powersave 档零干预

## v1.0.48 更新：sched_* 档位上下文，performance 档外挂优化

**问题**：若把 `sched_child_runs_first`/`sched_rt_runtime_us`/GPU boost 设为全局参数，
切回 balance/powersave 会残留——autogroup 类参数影响省电档节能，RT 不限流有咬死 CPU 风险。

**修复**：sched_* 改为**档位上下文**——

- **进 performance**：写入 `sched_child_runs_first=1`（子线程优先跑，外挂逻辑响应快）、
  `sched_rt_runtime_us=-1`（RT 不限流，关键线程不饿死）、GPU boost 开（按需响应）
- **切回 balance/powersave**：立即恢复内核默认值（child=0、rt=950000、boost 关）

三档互不污染，balance/powersave 调度表现与 v1.0.47 完全一致（零增量）。
guard 仍只守护 performance 生效期间的 CPU/GPU 目标。

> 注：`sched_autogroup_enabled` 本机不存在，写入自动跳过，无影响。

## v1.0.47 更新：powersave 档 DDR 提升至 5.46G 修复崩溃

**问题**：v1.0.46 的 powersave 档 DDR 锁 3094（force=12），SMI 总线带宽不足，
JPEG 解码等突发流量时触发 SMI hang，出现 kernel panic 崩溃/死机。

**修复**：powersave 档 `force_dram 12`（3094）→ `force_dram 6`（5460@630mV），
DDR 带宽翻倍，SMI 带宽窗口大幅缩小。日常省电档也能安全解码图片/视频。

## v1.0.46 核心突破：DDR 硬件锁（DVFSRC 强制 OPP）

**问题**：v1.0.44 用 devfreq 锁 DDR 无效——devfreq 只锁“软件目标”，游戏吃带宽时
DVFSRC 仲裁综合各模块带宽需求，把 DDR 顶回 6370/8528（实测）。

**突破**：找到 `/sys/kernel/helio-dvfsrc/dvfsrc_force_vcore_dvfs_opp` 硬件强制 OPP 档位节点，
写值直接绕过仲裁锁 DDR，实测打内存带宽都纹丝不动。

**本机实测映射**（天玑9300+）：

| force值 | DDR | Vcore | 用途 |
|---|---|---|---|
| 0 | 自动(顶8528) | - | fast档 |
| 4 | 6370 | 640mV | - |
| 5 | 5460 | 730mV | - |
| 6 | 5460 | 630mV | **balance/performance/powersave（省电）** |
| 7 | 4212 | 725mV | - |
| 8 | 4212 | 630mV | - |
| 10 | 3094 | 725mV | - |
| 12 | 3094 | 575mV | - |
| 15 | 2067 | 630mV | - |

**档位更新**：balance/performance 用 force=6（5460@630mV），powersave 用 force=6（5460@630mV），fast 用 force=0（自动）。

## v1.0.44 改动回顾

1. guard 防覆盖守护：3s 轮询、仅偏离时写回，防 scene/系统覆盖 CPU/GPU
2. balance 档 GPU 上限 260M（实测 GPU 只需 200-300M）
3. game_cleanup.sh 一键清后台（QQ/MSF，实测 22%+99% 抢核）

## 用法

游戏前：
```sh
sh /data/adb/modules/background_performance_drain/game_cleanup.sh --proot
sh /data/adb/modules/background_performance_drain/power_switch.sh balance
```

## 档位

| 档位 | CPU | GPU | DDR | 场景 |
|---|---|---|---|---|
| powersave | A720 1.6G + X4 1.2G | 150-300M | force=6 (5460) | 日常省电 |
| balance | A720 1.6G + X4 1.6G | 150-260M | force=6 (5460) | 游戏平衡(推荐) |
| performance | A720 满 + X4 1.7G | 260-650M | force=6 (5460) | 吃GPU高负载 |
| fast | 全核满频 | 400M-1.1G | force=0 (自动) | 竞技极限 |

## 回滚

原 v1.0.43 备份在 background_performance_drain.bak_143。