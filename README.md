# 浆体管道专家系统分析报告

> F-Rev F 版本完整解读 · 算法模型 · 数据路由 · 重构建议

## 📋 目录

- [系统概述](#系统概述)
- [核心算法模型](#核心算法模型)
- [数据路由架构](#数据路由架构)
- [参数分类](#参数分类)
- [重构建议](#重构建议)

---

## 系统概述

### 技术栈
- **语言**: Delphi 7 (Object Pascal)
- **数据库**: Microsoft Access (.mdb)
- **通信**: OPC DA (sopcdaauto.dll)
- **架构**: 桌面应用，6层数据流架构

### 核心功能
1. **水力模拟**: 基于特征线法(MOC)的管道瞬态流计算
2. **泄漏检测**: 多仪表投票机制 + 负压力波分析
3. **专家控制**: PID控制器 + 规则引擎的阀门/泵速优化
4. **批次跟踪**: 矿浆批次在管道中的位置追踪

### 代码规模
- **源文件**: 84个 .pas 文件
- **核心模块**: 25个目录
- **数据表**: 20+ 张 Access 表
- **设备数量**: 50+ 台泵/阀/仪表

---

## 核心算法模型

### 1. 水力求解器 (Simulator)

**方法**: 隐式特征线法 (MOC)

**核心方程**:
```
HGL计算: Hp = px.HPold + B·(Qnew - px.QPold) + dx·HydGrad
流量求解: QP = NR迭代求解 f = HGL_calc - HGL_target
```

**摩擦因子** (Darcy-Weisbach):
- 层流 (Re < 2500): `f = 64/Re`
- 湍流: Colebrook 显式近似（三级嵌套对数）

**固定常数**:
- Vconst = 353.6777651 (流量→流速换算)
- Gconst = 50983.991 (水力梯度常数)
- Bconst = 2.8325e-5 (压缩因子)

### 2. 泵特性模型

**离心泵** (4阶多项式拟合):
```
tdh = HR·numSeries·numStages·Σ(a[i]·(Q/speed)^i)·speed²
eff = Σ(e[i]·(Q/speed)^i)
NPSHr = Σ(h[i]·(Q/speed)^i)·speed²
```

**隔膜泵**:
```
Displacement = (π/4)·VolEff·Plex·Bore²·Stroke·1e-6
Flow = 60·Displacement·MaxSPM·(Speed-SpeedMin)/(SpeedMax-SpeedMin)
```

### 3. 阀门特性

**阀门阻力**:
```
Cv-开度特性: 查表线性插值 VCharacter 表
kv = numSeries · 13.69213 / Cv²
```

**节流阀**:
```
kc = 6378.35·(1-β⁴)/CD²/IDThroat⁴
CDfactor = a0 + a1·ln(Re) + a2·ln(Re)² + a3·ln(Re)³
```

### 4. 浆体物性

**比重**:
```
SlurrySG = 1 / (conc/SolSG + (1-conc)/LiqSG)
```

**刚度系数**:
```
CoeffRigidity = visCoeff · 10^(visExpon·VR)
VR = (LiqSG/SolSG)·(conc/(1-conc))
```

**声速** (Streeter公式):
```
Y = entAir/(head·ρ/1000·PressFac) + plv/liqBM + psv/solBM + c1/(eoD·walEM)
X = 1/(entAir·ρ_air + plv·ρ_l + psv·ρ_s)
a = √(X/Y)
```

### 5. 边界条件 (BC)

**泵站** (24种运行状态):
- Ramu PS2: 6泵 + 6泄压阀 + 32阀门
- 状态机: conPumpSta 0-23
- 由4条件组合: conPump/conP/conPRV/conbp

**阀站**:
- ValveStationK: 4种流路组合
- TerminalK: 16种流路组合 (2⁴)
- PS2K: 8种流路组合

### 6. 泄漏检测

**LDStatusCheck** (6种场景):
1. VS1 阀门变化
2. VS2 阀门变化
3. 终端阀门变化
4. 泵速变化 (ThreseHoldSpeed + MaxSpeedChange)
5. 系统停机 (5条流路全关)
6. 系统启动 (至少1条流路通)

**逻辑**: 检测操作 → 闭锁LD防误报 → 多仪表投票判断泄漏

### 7. 专家控制 (ES)

**压力检查**:
- CheckLinePressures: 遍历计算点检查管线压力
- CheckStationPressure: 站场压力 vs SetPoint

**阀门选择** (CheckWhichValve):
- 8种阀门配置 × 开/关动作
- 选择策略: 累积行程最少优先（均衡磨损）

**PID控制器**:
```
pos = V0 + direction·(PB/100)·[(1+dT/Ti)·e0 - e1 + (Td/dT)·(e0-2e1+e2)]
```

**泵速优先级** (PumpSpeedControllerIndex):
- 5测点 × 32状态
- 优先级: 吸入口 > 排出口 > VS3 > SF1 > SF4
- 双控制器取低信号 (select lowest)

---

## 数据路由架构

### 6层数据流

```
① 数据源 → ② 数据加载 → ③ 内存模型 → ④ 计算引擎 → ⑤ 专家控制 → ⑥ 输出/界面
```

### 核心数据节点

**设备数组** (Global_Variables_L1):
- cenpump[501..510] 离心泵
- pdpump[301..306] 隔膜泵
- valve[801..854] 阀门
- choke[] 节流阀
- controller[] PID控制器
- instrument[] 仪表
- tank[601..604] 储罐
- prd[] 泄压阀
- surgeTK[] 调压罐

**管道模型** (二维数组 [System, Pipeline]):
- Section: 管段属性 (Idiam/rf/GFw/GFs/deltaX/B)
- IPoint: 计算点 (LineP/QPnew/Conc/SG/HGL)
- Locations: 站点位置 (km/el/usP/dsP)

### 仿真循环 (每 dT 秒)

```
1. ReadInstrData(OPC) → instrument[].trendValue
2. Simulator(SI, PIp)
   ├─ for each Section:
   │   ├─ CallBC(NodeID) → 边界条件
   │   ├─ Hp(sign) → HGL (masl)
   │   ├─ QP(sign) → Flow (m³/h)
   │   └─ 更新 IPoint[]
   ├─ UpdateValve/CenPump/PDPump (设备动态)
   ├─ TankLevel / SurgeTankLevel (液位)
   ├─ LDStatusCheck (泄漏检测)
   └─ CheckPressures (专家控制)
3. OPCCommunications → 写回 SCADA
```

### 关键数据流路径

**路径A - 启动加载**:
```
Access DB → LoadFromDatabase → 全局设备数组/管道模型/流体物性
```

**路径B - 仿真循环**:
```
OPC → ReadInstrData → Simulator → CallBC → NR迭代 → IPoint更新
```

**路径C - 专家控制闭环**:
```
instrument[] → CheckPressures → CheckWhichValve → ExpertControl
→ SetPosition → ValvePosition → kv更新 → 下一周期Simulator
```

**路径D - 泵速控制闭环**:
```
instrument[压力] → PumpSpeedControllerIndex → PumpSpeedSetPoint
→ PID输出 → SpeedSet → PDPumpSpeed → Flow更新
```

---

## 参数分类

### 🔒 固定参数 (代码硬编码)

| 常量 | 值 | 用途 |
|------|-----|------|
| Vconst | 353.6777651 | 流量→流速换算 |
| Gconst | 50983.991 | 水力梯度常数 |
| Bconst | 2.8325e-5 | 压缩因子 |
| ChokeConst | 6378.35 | 节流阀阻力 |
| KFac | 13.69213 | 阀门Cv→k |
| IMax (NR) | 24 | NR最大迭代 |
| maxRampVelocity | 12000.0 | 最大斜坡速度 |
| SecPHour | 3600 | 秒→小时 |

### ⚙️ 项目配置参数

**设备参数** (从DB加载):
- 离心泵: a[0..3], b[0..3], e[0..3], h[0..3], Qmax, SpeedMax/Min
- 隔膜泵: Bore, Stroke, Plex, VolEff, MaxSPM
- 阀门: OpenCv, VChar, StrokeOpen/Close
- 控制器: PB, IntegralTime, DerivativeTime, SetPoint

**管道参数**:
- Idiam (内径), rf (粗糙度)
- GFw/GFs (水/浆体梯度因子)
- SSAllow/TransAllow (允许压力)
- deltaX (管段长度)

**流体物性**:
- 固体: SG, bulkModulus, equalMoist
- 浆体: visCoeff, visExpon, TauCoeff
- 液体: Temperature → SG/viscosity/vapPress

**专家控制参数** (Job-specific):
- ThreseHoldSpeed, MaxSpeedChange
- MinPToleranceS1/S2, MaxPToleranceS1/S2
- ChokeTimerSgt1/2
- TempSummer/Winter, GfactorSummer/Winter

---

## 重构建议

### 1. 架构升级

**当前问题**:
- 全局数组耦合严重
- BC硬编码路由 (NodeID→具体项目)
- Access DB单点

**建议方案**:
```
┌─────────────────────────────────────────┐
│  Spring Boot + MyBatis Plus + React     │
├─────────────────────────────────────────┤
│  配置中心 (Nacos)                        │
│  - 设备参数配置化                        │
│  - 专家控制规则引擎                      │
├─────────────────────────────────────────┤
│  数据层                                  │
│  - MySQL/PostgreSQL (配置+历史)          │
│  - Redis (实时数据缓存)                  │
│  - InfluxDB/TDengine (时序数据)          │
├─────────────────────────────────────────┤
│  通信层                                  │
│  - OPC UA (替代OPC DA)                  │
│  - MQTT (轻量级IoT)                      │
└─────────────────────────────────────────┘
```

### 2. 核心模块重构

**水力求解器**:
- 封装为 `HydraulicSolver` 服务
- NR迭代器抽象为 `NonlinearSolver` 接口
- 支持多种求解策略 (MOC/稳态/动态)

**边界条件**:
- 策略模式: `IBoundaryCondition` 接口
- 配置驱动: NodeID → BC类型映射表
- 插件化: 新项目只需实现新BC

**设备模型**:
- 多态: `Pump`, `Valve`, `Tank` 基类
- 特性曲线: 数据库存储 + 插值算法
- 动态行为: 状态机模式

**专家控制**:
- 规则引擎: Drools/Aviator
- 规则配置化: 阈值/优先级/动作可配置
- PID控制器: 独立服务，支持自整定

### 3. 数据层重构

**设备参数表**:
```sql
CREATE TABLE equipment (
  id INT PRIMARY KEY,
  type ENUM('cenpump', 'pdpump', 'valve', ...),
  tag VARCHAR(20),
  params JSON,  -- 设备特有参数
  config JSON   -- 运行时配置
);
```

**管道模型表**:
```sql
CREATE TABLE pipeline_section (
  id INT PRIMARY KEY,
  pipeline_id INT,
  start_km DOUBLE,
  end_km DOUBLE,
  idiam DOUBLE,
  roughness DOUBLE,
  ...
);

CREATE TABLE calculation_point (
  id INT PRIMARY KEY,
  section_id INT,
  km_post DOUBLE,
  elevation DOUBLE,
  ...
);
```

**时序数据**:
```sql
-- InfluxDB
measurement: instrument_data
tags: instrument_id, type
fields: value_fld, value_calc, value_dlt
time: timestamp
```

### 4. 前端重构

**技术栈**: React + TypeScript + Ant Design

**核心页面**:
- 系统总览: P&ID动态显示 (SVG/Canvas)
- 站场详情: 设备状态 + 趋势图 (ECharts)
- 设备编辑: 表单 + 特性曲线可视化
- 报警中心: 实时报警 + 历史查询
- 配置管理: 设备参数 + 专家规则

### 5. 部署方案

**容器化**:
```yaml
services:
  backend: Spring Boot (API + 计算引擎)
  frontend: React (Nginx)
  database: MySQL/PostgreSQL
  cache: Redis
  tsdb: InfluxDB
  opc-gateway: OPC UA Client → MQTT
```

**高可用**:
- 计算引擎: 主备切换
- 数据库: 主从复制
- 前端: CDN + 负载均衡

---

## 📊 可视化

[数据路由可视化](./data-routing.html)

点击节点可追踪完整的数据流路径，悬停查看详细说明。

---

## 📝 总结

F-Rev F 是一个成熟的工业级矿浆管道 SCADA 专家系统，核心优势:
- ✅ 完整的水力瞬态模拟
- ✅ 多项目适配 (Samarco/Ramu/李家沟/翁福/CMDIC)
- ✅ 专家控制 + 泄漏检测
- ✅ 20+ 年运行验证

重构重点:
- 🎯 解耦全局状态 → 依赖注入
- 🎯 配置驱动 → 硬编码参数
- 🎯 现代技术栈 → Delphi 7
- 🎯 云原生部署 → 单机桌面

---

**生成时间**: 2026-09-21  
**分析工具**: Hermes Agent + Delphi 源码解析  
**可视化**: 交互式 SVG 数据流图
