# 01 · 核心概念与 Tick 模型

← [[00-导读与环境]] · [[BehaviorTree.CPP-v3从入门到工程实战|目录]] · 下一章 → [[02-控制节点与装饰节点]]

---

## 1. 节点四类

| 类型 | 角色 | 例子 |
|------|------|------|
| **Action** | 做事 | 发导航目标、抓取、鸣笛 |
| **Condition** | 只读判断 | 电池 > 20%、是否已定位 |
| **Control** | 编排子节点 | Sequence、Fallback、Parallel |
| **Decorator** | 包装单个子节点 | Retry、Timeout、Inverter |

叶子（Action / Condition）承载领域逻辑；中间节点只做控制流。

## 2. NodeStatus 语义

```text
IDLE ──tick──► RUNNING ──► SUCCESS
                 │
                 └──────► FAILURE
```

- **IDLE**：未在执行，或上次已结束被重置  
- **RUNNING**：跨 tick 保持；再次 tick 应**续跑**，不是从头  
- **SUCCESS / FAILURE**：终态；父节点据此推进或切换

工程规则：

- Condition 通常**不同时返回 RUNNING**（瞬时判断）  
- 调用 ROS Action / 等待传感器时，Action 必须能 RUNNING  
- 收到取消 / 抢占时，节点应清理资源并返回 FAILURE（或约定状态）

## 3. Tick 是什么

`tick` =「问节点：现在该干什么、结果如何」。  
整棵树从根向下传播；父节点决定：

- 调哪个子节点  
- 是否继续 / 跳过 / 重置其他子节点  

典型主循环（任务节点内）：

```cpp
BT::NodeStatus status = BT::NodeStatus::IDLE;
rclcpp::Rate rate(10);  // 10 Hz，按项目定
while (rclcpp::ok() &&
         (status == BT::NodeStatus::RUNNING || status == BT::NodeStatus::IDLE)) {
    status = tree.tickRoot();
    // 处理 ROS 回调：executor.spin_some() 等
    rate.sleep();
  }
```

> Nav2 的 `bt_navigator` 已封装 tick + executor；自研编排节点时要明确**谁在 spin、谁在 tick**。

## 4. 同步 vs 异步节点

| 写法 | tick 内行为 | 适用 |
|------|-------------|------|
| `SyncActionNode` | 一次 tick 内结束 | 写黑板、轻量计算 |
| `StatefulActionNode` | `onStart` / `onRunning` / `onHalted` | 多数业务长任务（推荐） |
| `AsyncActionNode` / 协程节点 | 另线程或协程 | 遗留代码、阻塞第三方库 |

**反模式**：在 `SyncActionNode::tick()` 里 `rclcpp::spin_until_future_complete` 死等。  
一棵树被一个阻塞叶子拖死，恢复与取消都会失效。

## 5. 与 FSM 对照（何时用 BT）

| 场景 | 更合适 |
|------|--------|
| 少状态、强时序、少分支 | FSM / 生命周期状态机 |
| 多恢复路径、可插拔策略、要热改结构 | **行为树** |
| 导航默认行为 + 清图/自旋/后退 | Nav2 BT（事实标准） |
| 安全互锁、硬实时 | 独立安全层；BT 只做任务层 |

BT 不替代：底盘驱动、控制器、安全 PLC。它是**任务编排层**。

## 6. 最小心智模型（导航）

```text
Fallback（有一条成功即可）
 └─ Sequence（规划并跟踪）
 │    ├─ ComputePath
 │    └─ FollowPath          ← 常 RUNNING
 └─ Sequence（恢复）
      ├─ ClearCostmap
      └─ Spin / Backup
```

父节点语义决定「失败后干什么」——下一章专门讲控制与装饰节点。

下一章 → [[02-控制节点与装饰节点]]
