# 04 · XML 定义与加载

← [[03-黑板与Ports]] · [[BehaviorTree.CPP-v3从入门到工程实战|目录]] · 下一章 → [[05-自定义节点与插件]]

---

## 1. 最小骨架（v3）

```xml
<root main_tree_to_execute="MainTree">
  <BehaviorTree ID="MainTree">
    <Sequence name="root">
      <AlwaysSuccess/>
    </Sequence>
  </BehaviorTree>
</root>
```

| 要素 | 说明 |
|------|------|
| `main_tree_to_execute` | 启动时执行哪棵树 |
| `BehaviorTree ID` | 树名；可多棵，供 SubTree 引用 |
| 子元素标签名 | 必须等于 `factory.register*` 时的 ID |

不要写 `BTCPP_format="4"`（那是 v4）。

## 2. 加载方式

```cpp
BT::BehaviorTreeFactory factory;
// 先注册全部自定义节点 / 插件
registerMyNodes(factory);

// 文件
auto tree = factory.createTreeFromFile(path_to_xml);

// 或字符串（单测友好）
auto tree2 = factory.createTreeFromText(xml_text);

// 可选：传入已有黑板
auto bb = BT::Blackboard::create();
bb->set("goal", goal_msg);
auto tree3 = factory.createTreeFromFile(path_to_xml, bb);
```

加载失败常见原因：节点未注册、端口类型冲突、XML 拼写错误、路径不对。

## 3. 多树与 SubTree

```xml
<root main_tree_to_execute="Mission">
  <BehaviorTree ID="GoToPose">
    <Sequence>
      <ComputePathToPose goal="{goal}" path="{path}"/>
      <FollowPath path="{path}"/>
    </Sequence>
  </BehaviorTree>

  <BehaviorTree ID="Mission">
    <Sequence name="patrol">
      <SubTree ID="GoToPose" goal="{wp1}"/>
      <SubTree ID="GoToPose" goal="{wp2}"/>
      <SubTree ID="GoToPose" goal="{wp3}"/>
    </Sequence>
  </BehaviorTree>
</root>
```

原则：**可复用片段做成 SubTree**，主树只编排任务。

## 4. include 与文件组织

大项目拆文件：

```text
trees/
  mission_patrol.xml      # main
  subtree_navigate.xml    # 可被 include 或单独加载后注册
  subtree_recover.xml
```

若发行版支持 XML include，用官方方式拼装；否则在启动时用 `factory.registerBehaviorTreeFromFile`（v3 API 以你安装的头文件为准）把多文件树注册进 factory，再 `createTree("Mission")`。

工程建议：

- **一任务一主 XML**  
- 恢复、对接、充电等做成子树文件  
- XML 进 Git；用 PR 审「策略变更」

## 5. TreeNodesModel（给 Groot）

可选段，描述端口供 Groot 可视化编辑：

```xml
<root main_tree_to_execute="MainTree">
  <BehaviorTree ID="MainTree">...</BehaviorTree>

  <TreeNodesModel>
    <Action ID="ComputePathToPose">
      <input_port name="goal"/>
      <output_port name="path"/>
    </Action>
    <Action ID="FollowPath">
      <input_port name="path"/>
    </Action>
  </TreeNodesModel>
</root>
```

运行时执行不强制依赖 Model；但缺少它时 Groot 拖拽体验差。自定义节点上线时同步更新 Model。

## 6. 配置驱动工作流

```text
产品/现场调参 ──改 XML──► 重载树（或重启 bt 节点）
开发新能力   ──写 C++ 节点 + 注册──► 再在 XML 引用
```

Checklist：

- [ ] 所有标签已 `registerNodeType` / 插件导出  
- [ ] 端口名与 `providedPorts()` 一致  
- [ ] `main_tree_to_execute` 指向存在的 ID  
- [ ] 黑板 key 在写入方之前不会被强依赖读（或有默认）  

下一章写自定义节点与 `.so` 插件。

下一章 → [[05-自定义节点与插件]]
