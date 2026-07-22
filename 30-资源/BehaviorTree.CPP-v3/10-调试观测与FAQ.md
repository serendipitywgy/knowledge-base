# 10 · 调试、观测与 FAQ

← [[09-案例-多点巡检任务编排]] · [[BehaviorTree.CPP-v3从入门到工程实战|目录]] · 下一章 → [[11-案例-AOI工业检测流程]]

---

## 1. 调试工具箱

| 手段 | 用途 |
|------|------|
| **Groot**（v3 一代） | 可视化树、在线看节点状态（配 publisher） |
| `StdCoutLogger` / `FileLogger` | 每次 tick 状态变化落盘 |
| 节点内 ROS 日志 | 带 `name()`、ports、error_code |
| `/diagnostics` 或自定义 status topic | 现场 HMI |
| 仿真 + 替身节点 | 先证伪控制流，再接真 Action |

挂 logger（示意）：

```cpp
#include "behaviortree_cpp_v3/loggers/bt_cout_logger.h"
BT::StdCoutLogger logger(tree);
```

## 2. 常见故障对照

| 现象 | 可能原因 | 处理 |
|------|----------|------|
| `createTree*` 抛异常：Node not registered | 插件未加载 / 注册名与 XML 不一致 | 对一下 `registerNodeType` 字符串 |
| Port 类型 mismatch | 两边 `PortsList` 类型不同 | 统一类型或中间转换节点 |
| 树卡住一直 RUNNING | 叶子阻塞；或忘记返回终态 | 查当前 RUNNING 叶子；加 Timeout |
| 取消不停车 | `onHalted` 空实现 | 取消 Action / 发零速 |
| Reactive 子树反复重初始化 | 重活放在 Reactive 左侧 | 左条件右动作；初始化放 onStart |
| 子树读不到父数据 | remapping 漏配 | SubTree 属性显式映射 |
| Humble 能编 Jazzy 挂 | v3/v4 混用 | 锁发行版，改 include/包名 |
| Groot 连不上 | 未起 ZMQ/publisher 或版本不对 | v3 用 Groot1；查文档端口 |

## 3. 性能与实时性

- tick 频率：导航任务常见 10–100 Hz，按叶子成本调  
- 避免在 tick 路径做重规划、大图拷贝；规划应在 Action server  
- Parallel 子树不要无节制加；共享资源加互斥或单写者约定  

## 4. 选型 FAQ

**Q: 还要不要状态机？**  
A: 要。Lifecycle / 整机模式（手动、自动、急停）常用 FSM；**任务层**用 BT。两层别抢职责。

**Q: 业务全写 XML 脚本行吗？**  
A: 不行。XML 只编排；算法、协议、硬件在 C++ 节点。

**Q: 直接改 Nav2 大树还是上层再包一层？**  
A: 多任务 / 多业务 → **上层任务树**；只调导航恢复策略 → 改 Nav2 XML。

**Q: v3 还是直接上 v4？**  
A: 跟发行版与 Nav2 版本走。已锁 Humble 产线 → v3；新栈且依赖已是 v4 → 跟 v4，本系列概念仍适用，API/XML 需对照迁移说明。

**Q: 如何做 Code Review？**  
A: PR 里必看：XML 控制流变更、新增端口兼容性、`onHalted`、测试树是否更新。

## 5. 开项目一周建议节奏

| 天 | 产出 |
|----|------|
| 1 | 空包 + 最小树 tick 通；参数加载 XML |
| 2 | `NavigateToPose` 叶子 + 单点目标仿真 |
| 3 | 恢复策略（或沿用 Nav2 默认树） |
| 4 | 巡检队列 + 跳过/失败策略 |
| 5 | 取消、电池抢占、日志/HMI 字段 |
| 6–7 | 单测 + 仿真验收清单 + 实车安全用例 |

## 6. 延伸阅读

- BehaviorTree.CPP v3.8 文档：https://www.behaviortree.dev/docs/3.8/  
- Nav2 Behavior Trees：https://docs.nav2.org/behavior_trees/index.html  
- Nav2 BT 插件说明：https://docs.nav2.org/configuration/packages/configuring-bt-navigator.html  
- 本系列目录：[[BehaviorTree.CPP-v3从入门到工程实战]]
- 工业视觉编排案例：[[11-案例-AOI工业检测流程]]

---

下一章 → [[11-案例-AOI工业检测流程]] · 回到目录 → [[BehaviorTree.CPP-v3从入门到工程实战]]
