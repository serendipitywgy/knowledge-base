# 06 · ROS2 与 Nav2 集成

← [[05-自定义节点与插件]] · [[BehaviorTree.CPP-v3从入门到工程实战|目录]] · 下一章 → [[07-工程结构与可测试性]]

---

## 1. 在栈里的位置

```text
任务层 / 业务节点
    │  自定义 BT（巡检、对接、上下料）
    ▼
nav2_bt_navigator（导航 BT）
    │  ComputePath / FollowPath / Spin / Backup ...
    ▼
planner / controller / recoveries / costmap
```

两种集成深度：

| 模式 | 做法 | 适用 |
|------|------|------|
| **用 Nav2 默认导航树** | 只改 `default_nav_to_pose_bt.xml` 等 | 纯导航产品 |
| **上层任务 BT + 下层 Nav2** | 任务树里调 `NavigateToPose` Action | 巡检 / 多任务（推荐） |

多数工程项目：上层自研任务树，导航整段当作一个 Action 叶子。

## 2. 把 ROS2 Action 封成 BT 节点

Nav2 提供模板思路（`nav2_behavior_tree::BtActionNode`）：节点在 `onStart` 发 goal，`onRunning` 查结果，`onHalted` 取消。

自研最小形态：

```cpp
// 伪代码骨架 —— 实际可复用 nav2 的 BtActionNode 模板
BT::NodeStatus onStart() override {
  auto goal = buildGoalFromPorts();
  future_ = action_client_->async_send_goal(goal);
  return BT::NodeStatus::RUNNING;
}

BT::NodeStatus onRunning() override {
  // 检查 goal handle / result；SUCCESS / FAILURE / RUNNING
}

void onHalted() override {
  action_client_->async_cancel_goal(goal_handle_);
}
```

要点：

- `server_name`、`server_timeout` 做成 InputPort，便于 XML / 参数覆盖  
- 错误码输出到黑板，供 Fallback 分支  

## 3. Nav2 常用叶子（认知地图）

| 节点 | 作用 |
|------|------|
| `ComputePathToPose` / `ComputePathThroughPoses` | 规划 |
| `FollowPath` | 跟踪 |
| `Spin` / `BackUp` / `Wait` | 恢复行为 |
| `ClearEntireCostmap` | 清局部/全局代价地图 |
| `IsStuck` / 各类 Condition | 状态检测 |
| `NavigateToPose`（作 Action） | 上层任务树常用入口 |

黑板典型 key：`goal`、`path`、`controller_id`、`planner_id`、`error_code`。

## 4. 最小「上层任务树」XML

不改 Nav2 内部树，只把导航当黑盒：

```xml
<root main_tree_to_execute="Patrol">
  <BehaviorTree ID="Patrol">
    <Sequence name="one_round">
      <NavigateToPose goal="{wp1}" error_code="{nav_error}"/>
      <Wait msec="2000"/>
      <NavigateToPose goal="{wp2}" error_code="{nav_error}"/>
      <Wait msec="2000"/>
      <NavigateToPose goal="{wp3}" error_code="{nav_error}"/>
    </Sequence>
  </BehaviorTree>
</root>
```

`NavigateToPose` 为你注册的 `BtActionNode<nav2_msgs::action::NavigateToPose>`。

## 5. 生命周期与 Executor

BT tick 循环和 ROS 回调必须共存：

```text
MultiThreadedExecutor / 专用 callback group
  ├─ Action client 回调
  ├─ 订阅（电池、任务话题）
  └─ 定时器里 tree.tickRoot()
```

建议：

- Action client 使用**可重入 / 独立 callback group**，避免和 tick 死锁  
- Lifecycle Node：`on_activate` 加载树，`on_deactivate` halt 并释放  

## 6. 参数示例（概念）

```yaml
mission_bt:
  ros__parameters:
    tree_xml_file: "trees/patrol.xml"
    plugin_lib_names:
      - my_mission_bt_nodes
    tick_rate_hz: 10.0
    navigate_action: "navigate_to_pose"
```

与 Nav2 的 `bt_navigator` 参数风格对齐，运维只改 YAML/XML。

## 7. 发行版注意

- **Humble / Iron**：`behaviortree_cpp_v3` + Nav2 BT 插件  
- **Jazzy+**：确认是否已迁 v4；API / XML / Groot 都可能变  

锁定发行版后再固化本仓库教程中的 include 与 XML 方言。

下一章 → [[07-工程结构与可测试性]]
