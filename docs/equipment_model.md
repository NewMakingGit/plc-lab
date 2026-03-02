# 虚拟工厂设备模型设计文档

**项目：** NewMaking Lab — Virtual Factory  
**版本：** v0.1  
**日期：** 2026-03-02  
**状态：** 草稿

---

## 一、设计目标

本模型旨在构建一套可独立运行的虚拟工厂设备体系，核心特征：

- 设备可被"领用"实例化，独立运行，不依赖真实物理设备
- 支持手动操作、半自动、全自动三种运行模式
- 可通过 MQTT 接收指令、发布状态，与数据底座对接
- 模型结构参考工业标准，保留向标准扩展的路径
- 面向未来工业元宇宙，每台虚拟设备可作为虚拟车间节点

---

## 二、参考标准

本模型设计参考以下工业标准，采用简化实现，保留向标准扩展的接口：

| 标准 | 说明 | 参考内容 |
|------|------|---------|
| IEC 62264 / ISA-95 | 工厂信息模型 | 设备层级、设备与工单关系 |
| AAS（Asset Administration Shell）| 工业4.0资产管理壳 | 资产身份、子模型结构 |
| IEC 61360 | 设备属性描述规范 | 属性数据类型、单位定义 |
| OPC UA 信息模型 | 设备通信与描述标准 | 设备节点结构、数据类型 |
| ECLASS | 工业产品分类标准 | 设备分类编码 |

---

## 三、两个核心概念

### 3.1 模板（Equipment Template）

设备的通用定义，存放在设备库中，只有一份。  
描述这类设备"应该是什么样"，包含默认参数和行为规则。

### 3.2 实例（Equipment Instance）

从模板中"领用"出来的具体设备，有自己唯一的编号、位置和运行历史，可以存在多台。  
一个模板可以领用出无数台实例，互不干涉。

> 对应关系：模板 = 类（Class），实例 = 对象（Object）

---

## 四、基础设备模型（BaseEquipment）

所有设备共有的属性，任何具体设备都继承自此模型。

### 4.1 身份信息

| 字段 | 类型 | 说明 | 对应标准 |
|------|------|------|---------|
| equipment_id | UUID | 设备唯一编号 | AAS Asset ID |
| name | String | 设备名称 | - |
| category | String | 设备分类（成型/加工/检测/装配） | ECLASS |
| brand | String | 品牌 | - |
| model | String | 型号 | - |
| serial_number | String | 出厂编号 | - |
| template_id | UUID | 来源模板编号 | - |
| install_date | DateTime | 投入使用日期 | - |
| location | String | 位置（车间/产线/工位） | ISA-95 |

### 4.2 运行状态

| 字段 | 类型 | 说明 |
|------|------|------|
| status | Enum | 见状态定义 |
| status_changed_at | DateTime | 状态最后变更时间 |
| run_mode | Enum | 手动 / 半自动 / 全自动 |
| total_runtime_hours | Float | 累计运行小时数 |
| total_alarm_count | Integer | 累计报警次数 |

**状态定义（Equipment Status）：**

```
OFFLINE     关机/离线
IDLE        待机（上电但未运行）
RUNNING     运行中
ALARM       报警
MAINTENANCE 维护中
```

### 4.3 生产模式

设备分为两种基本生产模式：

| 模式 | 说明 | 适用设备 |
|------|------|---------|
| CYCLE | 节拍式，按次计数 | 注塑机、冲床、装配工位 |
| CONTINUOUS | 连续式，按速率计量 | 挤出机、激光切割、传送带 |

### 4.4 节拍式专用字段

| 字段 | 类型 | 说明 |
|------|------|------|
| cycle_time_target | Float | 目标节拍时间（秒） |
| cycle_time_actual | Float | 实际节拍时间（秒，实时） |
| shot_count_total | Integer | 累计循环次数 |
| shot_count_shift | Integer | 本班次循环次数 |
| good_count | Integer | 良品数量 |
| reject_count | Integer | 不良品数量 |

### 4.5 连续式专用字段

| 字段 | 类型 | 说明 |
|------|------|------|
| process_speed | Float | 当前加工速度（单位因机器而异） |
| speed_target | Float | 目标速度 |
| output_rate | Float | 产出速率（件/小时 或 kg/小时） |
| output_total | Float | 累计产出 |

---

## 五、注塑机扩展模型（InjectionMoldingMachine）

继承 BaseEquipment，生产模式为 CYCLE。

### 5.1 设备参数（静态，存 PostgreSQL）

| 字段 | 类型 | 单位 | 说明 |
|------|------|------|------|
| clamping_force | Float | kN | 锁模力 |
| injection_volume | Float | cm³ | 最大注射量 |
| screw_diameter | Float | mm | 螺杆直径 |
| max_injection_pressure | Float | MPa | 最大注射压力 |
| mold_id | String | - | 当前安装模具编号 |
| material_type | String | - | 当前使用原料类型 |

### 5.2 工艺参数（动态，存 TimescaleDB）

| 字段 | 类型 | 单位 | 说明 |
|------|------|------|------|
| barrel_temp_zone1 | Float | °C | 料筒温度区1 |
| barrel_temp_zone2 | Float | °C | 料筒温度区2 |
| barrel_temp_zone3 | Float | °C | 料筒温度区3 |
| nozzle_temp | Float | °C | 喷嘴温度 |
| mold_temp | Float | °C | 模具温度 |
| injection_pressure_actual | Float | MPa | 实际注射压力 |
| holding_pressure | Float | MPa | 保压压力 |
| screw_position | Float | mm | 螺杆位置 |
| cycle_phase | Enum | - | 当前工艺阶段 |

### 5.3 注塑机工艺阶段

一个完整注塑周期由以下阶段顺序组成：

```
MOLD_CLOSE    合模
INJECTION     注射
HOLDING       保压
COOLING       冷却
MOLD_OPEN     开模
EJECTION      顶出
──────────────────→ 循环回合模
```

每个阶段有独立的目标时间和实际时间，差异可用于质量分析。

---

## 六、其他设备扩展模型（概要）

### 6.1 冲床（PunchPress）

继承 BaseEquipment，生产模式为 CYCLE。

额外字段：额定压力（kN）、冲次（次/分钟）、模具编号、滑块位置（mm）

### 6.2 激光切割机（LaserCuttingMachine）

继承 BaseEquipment，生产模式为 CONTINUOUS。

额外字段：激光功率（W）、切割速度（mm/s）、辅助气体类型、焦距（mm）

### 6.3 CNC加工中心（CNCMachiningCenter）

继承 BaseEquipment，生产模式为 CYCLE。

额外字段：主轴转速（RPM）、进给速度（mm/min）、当前程序号、刀具编号

### 6.4 机械手（RoboticArm）

继承 BaseEquipment，生产模式为 CYCLE。

额外字段：自由度、最大负载（kg）、重复定位精度（mm）、当前程序号、各轴角度

---

## 七、数据存储分层

| 数据类型 | 存储位置 | 更新频率 | 说明 |
|---------|---------|---------|------|
| 设备模板 | PostgreSQL | 极少变更 | 设备定义，只读 |
| 设备实例（身份+参数） | PostgreSQL | 偶尔变更 | 安装模具、换料等 |
| 设备实时状态 | PostgreSQL（最新值） | 秒级 | 只保留最新一条 |
| 时序工艺数据 | TimescaleDB | 秒级/每周期 | 完整历史数据 |
| 报警记录 | PostgreSQL | 事件触发 | 报警产生/恢复记录 |

---

## 八、通信接口规范

所有设备实例通过 MQTT 与系统交互，主题格式遵循项目 MQTT 规范：

**状态发布（设备 → 系统）：**
```
factory/{site}/machine/{equipment_id}/telemetry
factory/{site}/machine/{equipment_id}/status
factory/{site}/machine/{equipment_id}/alarm
```

**指令接收（系统 → 设备）：**
```
factory/{site}/machine/{equipment_id}/command
```

**指令格式示例：**
```json
{
  "timestamp": "2026-03-02T10:00:00Z",
  "command": "START",
  "mode": "AUTO",
  "operator_id": "OP001"
}
```

---

## 九、扩展路径

当前为简化实现，未来可按需扩展：

- **接入真实设备**：模拟器替换为 OPC UA Client，连接真实 PLC
- **AAS 完整实现**：为每台设备生成标准 AAS 描述文件
- **数字孪生模式**：虚拟设备与真实设备状态同步
- **多设备联动**：注塑机 + 机械手 + 传送带组成完整产线
- **MES 集成**：设备实例与工单系统对接，实现自动报工

---

## 十、文件结构

```
plc-lab/
├── docs/
│   └── equipment_model.md        ← 本文档
├── src/
│   └── simulation/
│       ├── base_equipment.py     ← 基础设备类
│       ├── injection_molding.py  ← 注塑机模拟器
│       └── mqtt_client.py        ← MQTT 通信封装
└── README.md
```

---

*文档版本：v0.1 | 状态：草稿 | 下一步：注塑机模拟器实现*
