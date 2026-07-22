# 11 · 案例：AOI 工业检测流程

← [[10-调试观测与FAQ]] · [[BehaviorTree.CPP-v3从入门到工程实战|目录]]

---

## 1. 需求（可写进设计说明）

产线 AOI（自动光学检测）对一块 PCB / 工件做多 ROI 检测：

1. 读任务与配方（光源、曝光、ROI 列表、判定阈值）  
2. 运动到位（平台 / 机械臂 / 传送定位）  
3. 切光 → 触发采集 → 图像质量门禁（清晰度 / 过曝）  
4. 推理判缺陷；NG 可复检一次（换光或微动）  
5. 结果写 MES / 本地报告；急停与门禁可随时打断  
6. 单 ROI 失败策略可配置：跳过 / 整单失败  

**BT 只做任务编排**；视觉算法、相机驱动、运动控制仍在各自节点 / Action Server 内。

与 [[09-案例-多点巡检任务编排]] 的差别：这里没有 Nav2，叶子是 **运动 / 光源 / 相机 / 推理 / MES**，控制流同样是 Sequence + Fallback + Retry + Parallel。

## 2. 系统分层

```text
产线 PLC / 上位机 ──任务──► aoi_mission_bt
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
   move_action_server   camera/light nodes   infer + MES
          │                   │                   │
          └─────────── 硬件 / SDK / 模型 ─────────┘
```

| 层 | 放什么 | 不放什么 |
|----|--------|----------|
| XML 树 | 步骤顺序、重试、超时、跳过策略 | OpenCV / 模型代码 |
| BT 叶子 | Action/Service 调用、端口校验 | 隐式全局状态乱写 |
| 安全 | Parallel 监控急停；或独立安全层 | 用 BT 替代硬安全回路 |

## 3. 包内角色划分

| 组件 | 类型 | 职责 |
|------|------|------|
| `LoadRecipe` | SyncAction | 读配方 JSON/参数 → 黑板 |
| `PopNextRoi` | SyncAction | 从 ROI 队列取下一个 |
| `MoveToPose` | StatefulAction | 调运动 Action，RUNNING 直到到位 |
| `SetLighting` | Sync / Stateful | 切光源通道与亮度 |
| `TriggerCapture` | StatefulAction | 触发相机，输出 `image_id` / `frame` |
| `CheckImageQuality` | Condition / Sync | 清晰度、亮度门禁 |
| `RunInference` | StatefulAction | 调推理服务，输出 `defects` / `ok` |
| `UploadMes` | StatefulAction | 上报结果，超时可失败 |
| `IsEstopActive` | Condition | 急停 / 安全门 |
| `trees/aoi_inspect.xml` | XML | 编排 |

推荐包布局见 [[07-工程结构与可测试性]]，把 `trees/aoi_inspect.xml` 与 `nodes/aoi_*` 放进同一功能包即可。

## 4. 树结构（心智模型）

```text
Parallel（成功策略：主任务成功且监控未失败）
 ├─ Sequence  MainJob
 │    ├─ LoadRecipe
 │    └─ （ROI 循环，见下）
 └─ ReactiveSequence  Safety
      ├─ Inverter → IsEstopActive   // 急停按下 → 监控失败 → halt 主任务
      └─ AlwaysSuccess
```

单 ROI 子树：

```text
Sequence InspectOneRoi
 ├─ MoveToPose
 ├─ RetryUntilSuccessful (2)
 │    └─ Fallback
 │         ├─ Sequence CaptureOk
 │         │    ├─ SetLighting (primary)
 │         │    ├─ TriggerCapture
 │         │    ├─ CheckImageQuality
 │         │    └─ RunInference
 │         └─ Sequence Recapture
 │              ├─ SetLighting (alternate) 或 MicroJog
 │              ├─ TriggerCapture
 │              └─ ForceFailure   // 回到 Retry 再走主采集
 └─ Fallback
      ├─ Sequence PassPath
      │    ├─ IsInferenceOk
      │    └─ AlwaysSuccess
      └─ Sequence NgPath
           ├─ LogNg
           └─ （AlwaysSuccess=跳过 / AlwaysFailure=整单停）
```

## 5. XML（可放入 `trees/aoi_inspect.xml`）

节点名按你 `registerNodeType` 为准；下面是可直接改的工程骨架。

```xml
<root main_tree_to_execute="AoiMission">

  <BehaviorTree ID="CaptureAndInfer">
    <RetryUntilSuccessful num_attempts="2" name="capture_retry">
      <Fallback name="primary_or_recapture">
        <Sequence name="primary">
          <SetLighting profile="{light_primary}"/>
          <Timeout msec="3000">
            <TriggerCapture image="{image}" meta="{capture_meta}"/>
          </Timeout>
          <CheckImageQuality image="{image}" min_sharpness="80.0"/>
          <Timeout msec="5000">
            <RunInference image="{image}" recipe="{recipe}"
                          ok="{infer_ok}" defects="{defects}"/>
          </Timeout>
        </Sequence>

        <Sequence name="recapture">
          <SetLighting profile="{light_alternate}"/>
          <MicroJog axis="z" delta_mm="0.05"/>
          <Timeout msec="3000">
            <TriggerCapture image="{image}" meta="{capture_meta}"/>
          </Timeout>
          <ForceFailure name="retry_primary"/>
        </Sequence>
      </Fallback>
    </RetryUntilSuccessful>
  </BehaviorTree>

  <BehaviorTree ID="InspectOneRoi">
    <Sequence name="one_roi">
      <MoveToPose target="{roi_pose}" timeout_ms="10000"/>
      <SubTree ID="CaptureAndInfer"/>
      <Fallback name="judge">
        <Sequence name="pass">
          <IsInferenceOk ok="{infer_ok}"/>
          <AppendResult station_id="{roi_id}" status="PASS" defects="{defects}"
                        report="{report}"/>
        </Sequence>
        <Sequence name="ng_skip">
          <AppendResult station_id="{roi_id}" status="NG" defects="{defects}"
                        report="{report}"/>
          <!-- 要「NG 即整单失败」时改成 AlwaysFailure -->
          <AlwaysSuccess/>
        </Sequence>
      </Fallback>
    </Sequence>
  </BehaviorTree>

  <BehaviorTree ID="AoiMission">
    <Parallel name="job_with_safety" success_threshold="1" failure_threshold="1">
      <Sequence name="main_job">
        <LoadRecipe order_id="{order_id}" recipe="{recipe}"
                    roi_queue="{roi_queue}"
                    light_primary="{light_primary}"
                    light_alternate="{light_alternate}"/>
        <!-- 真实项目用 WhileQueueNotEmpty Decorator；此处示意单步 + 外层节点循环 -->
        <PopNextRoi queue="{roi_queue}" roi_id="{roi_id}" roi_pose="{roi_pose}"/>
        <SubTree ID="InspectOneRoi"/>
        <Timeout msec="8000">
          <UploadMes order_id="{order_id}" report="{report}"/>
        </Timeout>
      </Sequence>

      <ReactiveSequence name="safety">
        <Inverter>
          <IsEstopActive/>
        </Inverter>
        <KeepRunningUntilFailure>
          <AlwaysSuccess/>
        </KeepRunningUntilFailure>
      </ReactiveSequence>
    </Parallel>
  </BehaviorTree>

  <TreeNodesModel>
    <Action ID="LoadRecipe">
      <input_port name="order_id"/>
      <output_port name="recipe"/>
      <output_port name="roi_queue"/>
    </Action>
    <Action ID="TriggerCapture">
      <output_port name="image"/>
      <output_port name="meta"/>
    </Action>
    <Action ID="RunInference">
      <input_port name="image"/>
      <input_port name="recipe"/>
      <output_port name="ok"/>
      <output_port name="defects"/>
    </Action>
  </TreeNodesModel>
</root>
```

说明：

- `Timeout` 包采集 / 推理 / MES，防止 SDK 卡死拖死整树  
- 复检末尾 `ForceFailure`：让外层 Retry 重新走主采集（与 [[08-案例-点到点导航与恢复]] 同一套路）  
- `Parallel` + 急停：监控失败会 halt 兄弟分支，**`MoveToPose` / `TriggerCapture` 的 `onHalted` 必须停轴、关触发**  
- ROI 循环：v3 无通用 While 时，用自定义 Decorator，或在 `aoi_mission_bt` 里「队列非空则反复 tick 子树」

## 6. 关键自定义节点接口

### 6.1 LoadRecipe

```cpp
static BT::PortsList providedPorts() {
  return {
    BT::InputPort<std::string>("order_id"),
    BT::OutputPort<Recipe>("recipe"),
    BT::OutputPort<std::vector<Roi>>("roi_queue"),
    BT::OutputPort<LightProfile>("light_primary"),
    BT::OutputPort<LightProfile>("light_alternate"),
  };
}
// 缺文件 / JSON 坏 → FAILURE + 明确 error_code
```

### 6.2 TriggerCapture / RunInference

用 `StatefulActionNode`（见 [[05-自定义节点与插件]]）：

- `onStart`：发 Action / 投递请求  
- `onRunning`：查结果；成功写 `image` / `defects`  
- `onHalted`：取消触发、释放相机锁  

### 6.3 CheckImageQuality

```cpp
// SyncAction 或 Condition：读 image，算 sharpness；不达标 FAILURE
// 不要在这里做推理——门禁与算法分离，便于单测与调参
```

### 6.4 IsEstopActive

`ConditionNode`：读安全 IO / `/safety/estop` 缓存；**按下 = SUCCESS**（再经 `Inverter` 让监控失败）。

## 7. 黑板契约

| Key | 含义 | 谁写 | 谁读 |
|-----|------|------|------|
| `order_id` | 工单号 | 任务入口 | LoadRecipe / MES |
| `recipe` | 配方 | LoadRecipe | 推理 / 光源 |
| `roi_queue` | 待检 ROI | LoadRecipe / Pop | Pop / 循环条件 |
| `roi_pose` / `roi_id` | 当前 ROI | PopNextRoi | Move / 报告 |
| `image` | 帧或句柄 | Capture | Quality / Infer |
| `infer_ok` / `defects` | 判定 | Infer | 分支 / MES |
| `report` | 累计结果 | AppendResult | UploadMes |
| `light_*` | 光源配置 | LoadRecipe | SetLighting |

大图建议黑板只存 **id / shared_ptr**，避免 Path 式大对象拷贝卡 tick（见 [[03-黑板与Ports]]）。

## 8. 任务入口与取消

```cpp
void onInspectOrder(const OrderMsg& msg) {
  tree.haltTree();  // 取消当前：停轴、停采集
  auto bb = tree.rootBlackboard();
  bb->set("order_id", msg.order_id);
  // recipe 路径也可只设 order_id，由 LoadRecipe 去拉
}
```

换型 / 急停解除后重新派单：先确认安全 Condition，再 `haltTree` + 写黑板。

## 9. 验收清单（产线前）

| 用例 | 期望 |
|------|------|
| 单 ROI 良品 | 一次采集成功，MES PASS |
| 轻微虚焦 | Quality FAILURE → 复检微动 → PASS 或最终 NG |
| 推理超时 | Timeout → Retry/整单失败策略符合配置 |
| 检测中急停 | Parallel 监控失败，轴停、相机不继续触发 |
| 取消工单 | `haltTree` 后硬件空闲，可接新单 |
| 配方缺失 | LoadRecipe FAILURE，不发运动 |
| NG 跳过 vs 停线 | 只改 XML 末尾 AlwaysSuccess/Failure，无需改 C++ |

单测：用 `AlwaysSuccess` / FakeCapture / FakeInfer 替身先证伪控制流，再接真机（[[07-工程结构与可测试性]]）。

## 10. 落地改法（开 AOI 项目时）

1. 先打通 **Move → Capture → 假推理 → MES 日志** 一条 Sequence  
2. 再加 Quality 门禁与 Retry 复检  
3. 再加 Parallel 急停（实机必须先在仿真 / 干跑验证 `onHalted`）  
4. 多 ROI：自定义 `WhileQueueNotEmpty` 或节点外循环  
5. 配方与阈值全部配置化，XML 只保留控制流  

## 11. 和前面案例的关系

```text
08 导航恢复     → Fallback + Retry + ForceFailure 套路（本案例复检同构）
09 多点巡检     → 队列 + 子树 + 可跳过（本案例 ROI 队列同构）
11 AOI 检测     → 把叶子换成工业视觉/运动/MES；加质量门禁与安全并行
```

控制流可复用；换领域时 **少改 XML 骨架，多换插件节点**。

---

回到目录 → [[BehaviorTree.CPP-v3从入门到工程实战]]
