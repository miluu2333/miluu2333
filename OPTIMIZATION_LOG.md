# 代码优化记录

前 5 轮是仓库治理（编码声明、文档、审查报告、导航、校验）。从第 6 轮起进入**代码本身**的优化，对象是 `D:\CodeArchive` 下 16 个仓库中的自研源码。

## 总原则

1. **运行时行为不变优先**。没有硬件验证条件时不改控制律、PID 参数、引脚分配、中断配置和状态机。现场遗留的"绕过校验"一律改成编译期开关，默认保持原行为，同时留出恢复路径。
2. **只改自己的代码**。逐飞科技、ST、Infineon 的官方库与 `history/` 历史快照一律不动；`【优化9】`、`【Bug-6修复】` 这类记录设计决策的中文标记也保留，它们是有效的维护信息。
3. **字节级操作保护编码**。归档里有 604 个文件是 GBK 编码，所有批量替换都按单字节映射（Latin-1）读写，不经过编码转换，中文注释不会被写成替换符。
4. **每轮可验证**。改完跑括号平衡、危险函数残留、守卫、行尾空格、编码完整性自检，再提交推送。

## 轮次一览

| 轮次 | 主题 | 涉及仓库 | 实质产出 |
| --- | --- | --- | --- |
| 6 | 信达杯安全性修复 | 1 | 11 个文件、+128/−61 行，修 6 类真实缺陷 |
| 7 | 跨仓缓冲区与头文件守卫 | 3 | 12 处无界格式化调用改带长度限制，4 个头文件补守卫 |
| 8 | 代码风格基线 | 16 | 统一 `.editorconfig`，清理自研源码行尾空格，新增代码校验脚本 |
| 9 | 校验口径收敛 | 16 | 修掉校验暴露的真遗漏，回滚 575 个第三方文件的误改，最终 16/16 通过 |
| 10 | c-samples 重复文件比对 | 1 | SHA256 全量比对确认属版本分叉，未删文件；README 注明结论，16/16 通过 |
| 11 | 自研模块接口注释 | 2 | 18 个头文件补 79 处 Doxygen 注释，纯新增、0 删除，16/16 通过 |

## 迭代 6：信达杯（`2025XinDaBei`）

| 文件 | 问题 | 处理 |
| --- | --- | --- |
| `Hardware/mt_vofa.c` | `sprintf` 向 `txBuffer[128]` 写入，无长度限制 | 改 `snprintf`，返回值钳制后作为帧长 |
| `Hardware/mt_mpu6050.c` | WHO_AM_I 校验整段被注释，注释写着"不管 ID 是多少都强行返回成功" | 改为 `MPU6050_STRICT_ID_CHECK`（默认 0），读回值存入 `g_mpu6050DeviceId` 供诊断 |
| `Hardware/mt_ultrasonic.c` | 对 `volatile` 结构体 `memset`，丢易失性语义 | 改静态零值整体赋值 |
| `Hardware/mt_ultrasonic.c` | `IsObstacleDetected()` 恒返回 false 的占位函数 | 改为 `ULTRASONIC_OBSTACLE_CHECK` 开关，默认仍是占位行为，注释说明启用会影响避障状态机 |
| `Hardware/mt_track_detect.c` | 能量累加循环硬编码 `5` | 改 `SENSOR_MAX` |
| `Hardware/mt_encoder.c` | `GPIO_InitStructure` 未完全初始化；`PI` 宏名易与 `math.h` 冲突 | 显式初始化 `GPIO_Speed`；改名 `ENCODER_PI` |
| 全仓 `.c/.h` | 30 处 `【修改】/【强制修复】` 等比赛期标记 | 逐条换成说明真实意图的正式注释 |

同步更新 `KNOWN_ISSUES.md`：新增"超声波模块的三处现场状态"与完整改动清单。

## 迭代 7：缓冲区与守卫

12 处 `sprintf` 全部改为 `snprintf` 并传入实际缓冲区大小：

| 仓库 | 文件 | 缓冲区 |
| --- | --- | --- |
| `zhufei-demo` | `src/common_Mymenu.cpp` | `tmpchar[...]`（5 处） |
| `project-template` | `mt/mt_mode_control.c` | `buf[32]`（5 处） |
| `project-template` | `mt/mt_vofa.c` | `send_buffer[256]`，并钳制发送长度 |
| `yanguo-liuhen` | `project老/code/vofa_bt.c` | `buffer[96]` |

4 个头文件补 `#pragma once`：`OS/OS_System.h`、`include/Menu.hpp`、`include/Vofa.hpp`、`include/Key.hpp`。

未改动 `history/` 里 Infineon 官方库和逐飞库中的 `sprintf`——它们是第三方代码，且属于历史快照。

## 迭代 8 与 9：风格基线、校验、收敛

- 16 个仓库统一 `.editorconfig`。**刻意不设 `charset`**：归档里有 604 个 GBK 文件，强制 UTF-8 会让编辑器重存时损坏中文注释；改为在注释里说明"新增文件统一 UTF-8"。
- 清理自研源码的行尾空格。中途两次误触第三方库（共 575 个文件），全部用 `git checkout` 回滚，最终只保留自研文件的改动。
- 新增 `verify-code.ps1`：按仓库白名单只检查自研源码，检查必备文件、无界格式化调用、头文件守卫、行尾空格；扫描危险函数前先剥离注释（避免把"已去除 sprintf"这类说明误报成调用）；花括号计数只作提示，因为注释和字符串里的括号会让它失真。
- 生成 `CODE_VERIFY_REPORT.md`，最终 **PASS 16/16、FAIL 0**。

## 迭代 10：`c-samples` 重复头文件去重（A 类高优先级）

待办要求"先比对 SHA256，完全一致才删副本"。逐文件比对后，删除前提不成立，**未删除任何文件**。

| 比对项 | 结果 |
| --- | --- |
| 同名 `.c/.h` 文件 | 30 个 |
| SHA256 完全一致 | 5 个 |
| 内容不同 | 25 个 |
| `menu_*.h` 一致 / 不同 | 2 / 3 |

一致的 5 个是 `diag_service.c`、`diag_service.h`、`encoder_exti_transition.h`、`menu_pages.h`、`menu_self_test.h`。`menu_*.h` 里只有 `menu_pages.h`、`menu_self_test.h` 一致，`menu_core.h`、`menu_input.h`、`menu_types.h` 均为不同版本。

不删的三个理由：

1. 去重前提是整套头文件完全一致，实际 5 个里只有 2 个一致。
2. 根目录 `.c` 用 `#include "同名.h"` 引用同目录头文件，删除会破坏这一套的编译自洽。
3. 两套差异明显（`menu_core.c` 根目录 27013 字节、`code/` 10522 字节），是版本分叉不是冗余副本，删除会丢掉一份历史实现。

改动只有 `c-samples/README.md`：修正"全部位于 `code/`"的不准确描述，新增「重复文件比对结论」章节，记录比对结果、不删理由与后续建议。源码零改动。

新发现：两套源码存在版本分叉，主副本归属需人工决策，已登记到待办 C 类。

验证：`verify-code.ps1` PASS 16/16。行为影响：无，纯文档改动。

## 迭代 11：自研模块接口注释（A 类高优先级）

为 `c-samples`、`ti-2026` 两个仓库的公共函数补 Doxygen 风格接口说明，覆盖 `@brief`、`@param`、`@return` 与副作用 `@note`。只改头文件声明处的注释，不动任何 `.c` 实现。

| 项目 | 结果 |
| --- | --- |
| 涉及仓库 | 2 个 |
| 改动头文件 | 18 个 |
| 新增注释块 | 79 处 |
| 删除行数 | 0 |
| 已存在中文注释 | 全部原样保留 |

实施口径：

1. 只处理**声明上方没有任何相邻注释**的公共函数。已有 `函数简介 / 参数说明 / 返回参数 / 备注信息` 中文说明块的声明一律跳过，不改写、不覆盖，避免重复文档并保留既有维护信息。
2. 说明内容以同仓另一套源码里已有的中文说明块、以及对应 `.c` 实现为依据，逐函数核对返回值和副作用，不凭函数名猜测。
3. `c-samples` 根目录与 `code/` 是迭代 10 确认过的版本分叉，两者行为有差异（如 `xunxian_get_error` 无黑线返回值、符号约定不同）。因此两套都补注释，不指定主副本；涉及差异的描述采用「由本版本实现约定」的中性表述，避免把某一版行为写死。
4. 逐仓按原有换行风格写入：`CRLF` 文件用 CRLF、`LF` 文件用 LF，写入前做换行往返一致性校验，避免整文件换行变化。
5. 全程按单字节映射（Latin-1 / 28591）读写；新增中文先把 UTF-8 字节转成 28591 字符串再拼接写回，保证原有字节不变、中文不变成替换符。

改动清单（`+` 为新增注释块数）：

| 仓库 | 文件 | 块数 |
| --- | --- | --- |
| `c-samples` | `action.h` | 8 |
| `c-samples` | `code/action.h` | 1 |
| `c-samples` | `code/encoder_exti.h` | 3 |
| `c-samples` | `code/imu.h` | 3 |
| `c-samples` | `code/menu_core.h` | 13 |
| `c-samples` | `code/task_runner.h` | 7 |
| `c-samples` | `code/tune_params.h` | 9 |
| `c-samples` | `code/xunxian.h` | 3 |
| `c-samples` | `imu.h` | 2 |
| `c-samples` | `menu_core.h` | 2 |
| `c-samples` | `tune_params.h` | 8 |
| `c-samples` | `xunxian.h` | 2 |
| `ti-2026` | `code/action.h` | 4 |
| `ti-2026` | `code/imu.h` | 2 |
| `ti-2026` | `code/menu_core.h` | 2 |
| `ti-2026` | `code/speed_pid.h` | 1 |
| `ti-2026` | `code/tune_params.h` | 8 |
| `ti-2026` | `code/xunxian.h` | 1 |

验证：`verify-code.ps1` **PASS 16/16、FAIL 0**。补充自检：严格 UTF-8 解码全部通过、无 U+FFFD 替换符、无行尾空格、`git diff --numstat` 全部为 `0` 删除（纯新增）、`git diff --check` 无真实空白错误、未触碰 `history/` 与逐飞/ST/Infineon 官方库及 `isr.h`。行为影响：无，纯注释改动，不改控制律、PID 参数、引脚分配、中断配置与状态机。

## 验证方式

| 检查 | 方法 |
| --- | --- |
| 括号平衡 | 逐文件统计花括号、圆括号数量 |
| 危险函数残留 | 正则扫描剥离注释后的源码 |
| 编码完整性 | 严格 UTF-8 解码 + 检查 U+FFFD 替换符字节 |
| 改动性质 | `git diff --ignore-all-space` 确认除新增文件外只有空白变化 |
| 线上一致性 | GitHub API 查询各仓库最新提交与文件状态 |

## 后续

剩余可优化项见 `OPTIMIZATION_BACKLOG.md`。凡是会改变控制行为的条目，都需要实车验证后才动。
