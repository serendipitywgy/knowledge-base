# 03 · 黑板与 Ports

← [[02-控制节点与装饰节点]] · [[BehaviorTree.CPP-v3从入门到工程实战|目录]] · 下一章 → [[04-XML定义与加载]]

---

## 1. Blackboard 是什么

黑板 = 树内共享的 **key → 值** 存储。  
节点之间**不互相调用**，只通过端口读写约定好的 key。

```text
ComputePathToPose  --out path-->  {path}  --in path-->  FollowPath
```

## 2. Port：显式数据接口

自定义节点用静态方法声明端口：

```cpp
#include "behaviortree_cpp_v3/action_node.h"

class ComputePath : public BT::SyncActionNode
{
public:
  ComputePath(const std::string& name, const BT::NodeConfiguration& config)
  : BT::SyncActionNode(name, config) {}

  static BT::PortsList providedPorts()
  {
    return {
      BT::InputPort<geometry_msgs::msg::PoseStamped>("goal"),
      BT::OutputPort<nav_msgs::msg::Path>("path"),
      BT::InputPort<std::string>("planner_id", "GridBased", "planner plugin id"),
    };
  }

  BT::NodeStatus tick() override
  {
    geometry_msgs::msg::PoseStamped goal;
    if (!getInput("goal", goal)) {
      // 缺输入：明确失败，不要默默用垃圾值
      return BT::NodeStatus::FAILURE;
    }
    nav_msgs::msg::Path path;
    // ... 调用规划 ...
    setOutput("path", path);
    return BT::NodeStatus::SUCCESS;
  }
};
```

| API | 用途 |
|-----|------|
| `InputPort<T>` | 读 |
| `OutputPort<T>` | 写 |
| `getInput("x", var)` | 取输入，失败返回 false |
| `setOutput("y", val)` | 写输出 |

## 3. XML 里如何接线

花括号表示「指向黑板 entry」：

```xml
<Sequence>
  <GetGoalFromTopic goal="{goal}"/>
  <ComputePathToPose goal="{goal}" path="{path}" planner_id="GridBased"/>
  <FollowPath path="{path}" controller_id="FollowPath"/>
</Sequence>
```

- `goal="{goal}"`：端口 `goal` ↔ 黑板 key `goal`  
- `planner_id="GridBased"`：字面量，不进黑板  

类型不一致会在 `createTree*` 时抛异常——这是好事，尽早暴露。

## 4. 子树与 remapping

子树有独立黑板作用域时，需要 remapping（把父 key 映射到子端口名）：

```xml
<root main_tree_to_execute="MainTree">
  <BehaviorTree ID="NavigateOnce">
    <Sequence>
      <ComputePathToPose goal="{target}" path="{path}"/>
      <FollowPath path="{path}"/>
    </Sequence>
  </BehaviorTree>

  <BehaviorTree ID="MainTree">
    <Sequence>
      <SetBlackboard output_key="goal_a" value="..."/>
      <SubTree ID="NavigateOnce" target="{goal_a}"/>
    </Sequence>
  </BehaviorTree>
</root>
```

工程建议：

- 子树对外只暴露少量端口（像函数参数）  
- 禁止子树偷偷读写未声明的父黑板 key（难测、难复用）

## 5. 工程规范（强烈建议）

1. **端口命名稳定**：`goal` / `path` / `error_code`，与 Nav2 习惯对齐，降低认知成本  
2. **缺输入必须 FAILURE**：并打日志（含节点名、缺哪个 port）  
3. **大对象注意拷贝**：`Path`、`OccupancyGrid` 频繁拷贝会卡；必要时存 `shared_ptr` 或缩短生命周期  
4. **错误码也走端口**：`error_code` / `error_msg` 输出，恢复子树按码分支  
5. **不要当全局变量堆**：只放节点间契约数据；进程级配置用 ROS 参数

## 6. 与 ROS 参数分工

| 数据 | 放哪 |
|------|------|
| 树拓扑、端口连线 | XML |
| 节点默认超时、server 名 | Port 默认值或 ROS 参数 |
| 一次任务的 goal / waypoint 列表 | 黑板（运行时） |
| 地图、TF、传感器流 | ROS topic / TF；节点内订阅，不塞满黑板 |

下一章把树写成可加载的 XML。

下一章 → [[04-XML定义与加载]]
