# BehaviorTree.CPP v3 从入门到工程实战

面向 **ROS2 导航 / 任务编排** 的 BehaviorTree.CPP **v3** 工程教程。  
强调：**树结构用 XML、逻辑用自定义节点、数据走黑板 Ports、可测可观测可热改树**。

> 官方文档（v3.8）：[behaviortree.dev/docs/3.8](https://www.behaviortree.dev/docs/3.8/)  
> 源码分支：[BehaviorTree.CPP `v3.8`](https://github.com/BehaviorTree/BehaviorTree.CPP/tree/v3.8)  
> Nav2 集成：[`nav2_behavior_tree`](https://docs.nav2.org/behavior_trees/index.html)

## 学习路径

| 阶段 | 章节 | 目标 |
|------|------|------|
| 入门 | [[00-导读与环境]] → [[01-核心概念与Tick模型]] | 能装、懂 SUCCESS/FAILURE/RUNNING |
| 必会 | [[02-控制节点与装饰节点]] → [[03-黑板与Ports]] → [[04-XML定义与加载]] | 能读树、能接线、能改 XML |
| 工程 | [[05-自定义节点与插件]] → [[06-ROS2与Nav2集成]] → [[07-工程结构与可测试性]] | 能写插件、能接 Action、能组包 |
| 落地 | [[08-案例-点到点导航与恢复]] → [[09-案例-多点巡检任务编排]] → [[11-案例-AOI工业检测流程]] → [[10-调试观测与FAQ]] | 能开项目、能排障 |

## 章节目录

1. [[00-导读与环境]] — 为何用 BT、v3 vs v4、依赖与最小可运行  
2. [[01-核心概念与Tick模型]] — NodeStatus、tick、同步/异步、与 FSM 对比  
3. [[02-控制节点与装饰节点]] — Sequence / Fallback / Parallel / Decorator 选型表  
4. [[03-黑板与Ports]] — Blackboard、Input/Output Port、类型安全  
5. [[04-XML定义与加载]] — 树语法、`main_tree_to_execute`、子树与 remapping  
6. [[05-自定义节点与插件]] — Sync/Stateful/Async、`BT_REGISTER_NODES`  
7. [[06-ROS2与Nav2集成]] — `BtActionNode`、bt_navigator、常用 Nav2 节点  
8. [[07-工程结构与可测试性]] — 包布局、参数、单测、CI、配置驱动  
9. [[08-案例-点到点导航与恢复]] — 规划→跟踪→清图→重试的工程树  
10. [[09-案例-多点巡检任务编排]] — 任务队列、子树复用、取消与抢占  
11. [[10-调试观测与FAQ]] — Groot、日志、常见崩溃与选型 FAQ  
12. [[11-案例-AOI工业检测流程]] — 配方→到位→采集→门禁→推理→MES、急停并行 

## 本库约定

- 代码 include 用 **`behaviortree_cpp_v3/...`**（不是 v4 的 `behaviortree_cpp`）
- XML 用 v3 风格：`<root main_tree_to_execute="...">`（不要写 `BTCPP_format="4"`）
- 黑板引用写 `{key}`；Nav2 文档里偶见 `${key}`，工程里统一 `{key}`
- 一篇笔记只讲一件事；案例里的 XML / C++ 可直接改包名复用

关联：资源总览 [[README]] · 图示可用 [[PlantUML从入门到精通]]
