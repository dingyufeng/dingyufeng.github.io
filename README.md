# BCNS 新一代参考实现（Python）

> 用统一技术栈替代原有 10 类老软件的**可运行原型**。
> 设计原则：**配置驱动、沿用现有 Schema、渐进迁移**。
> 2026-09-10

---

## 一、这套原型解决什么

| 老软件 | 问题 | 本原型替代 |
|---|---|---|
| YeatCtl（VC++/MFC） | 源码老、无版本管理、改一处要改所有项目 | `collector.py` 配置驱动的采集服务 |
| BCNSCom / YEATDLL | COM 组件封闭 | `api_server.py` / `api_server_lite.py` REST/JSON |
| webbcns（ASP+Flash） | Flash 停服、无法手机访问 | 前端另行 Vue3 化（本目录为后端支撑） |
| VB6 组态工具 | VB6 已淘汰 | 点表/报警全部 YAML 配置化 |

---

## 二、原两大核心程序的功能覆盖对照

### ① YeatCtl.exe（后台监控服务器）—— 四大功能

| 原功能 | 本原型实现 | 状态 |
|---|---|---|
| ① 定时轮询各网络控制器（≤32 台）获取状态 | `collector.py` 按配置轮询，台数不限 | ✅ |
| ② 状态变化**即时**存库 | `ChangeDetector` 变化检测 + `on_change_only` 可只在变化时写库 | ✅ |
| ③ 经 COM 与前台交换数据、**接收用户命令** | API `POST /control` 入队 → 采集服务执行 | ✅ |
| ④ 向指定控制器**按协议下发命令** | `write_register`（功能码 6）+ 命令队列闭环 | ✅ |
| 配置 YeatCtl.INI（SvrName/DBType/DBPath/UID/CtrlLib/DataPort） | YAML：host/port/framer/parity/table/driver 等 | ✅ 等价 |

**说明**：原程序为 VC++/MFC 编译产物；本原型以 Python **重新实现同等功能**，
不是"保留原内核编译"。原 VC++ 源码仍完整保留在 `01_核心软件_公共底座/A1` 供对照与回退。

### ② BCNSCom / YEATDLL.dll（COM 通讯组件）—— 协议语义保留

`bcnscom_adapter.py` 完整保留原协议语义并封装为标准适配层：

| 原协议要素 | 实现 |
|---|---|
| REQUEST / RESPONSE 各分 A、B 两类 | `BCNSRequest` / `BCNSResponse`，kind='A'\|'B' |
| A 类最多 8 个 VARIANT 参数 | `MAX_VARIANTS = 8`，超限校验 |
| B 类 128 字节 COMBUFFER | `COMBuffer`，128 字节定长、超容量校验 |
| 按 IP 末字节寻址 bCtlNumIP | 所有接口首参 `bCtlNumIP` |
| GET / PUT 收发命令 | `get_response()` / `put_request()` |
| 功能码 0FFH 后台服务请求 | `FUNC.SERVICE_REQ`，返回服务状态 |
| 81H/82H 读控制点、91H—93H 读参数、F2H 特殊控制 | 均有对应分支 |
| 功能码分区（<40H 操作 / 40H—80H 请求 / >80H 块发送） | `FUNC.zone()` 自动分区处理 |

自测：`python bcnscom_adapter.py`（6 项全通过）

### 控制闭环（可审计）

```
前台/数字员工
   │  POST /api/v1/{code}/control  {"equip_id":621002,"addr":5,"value":100}
   ▼
API ──写入──▶ command_queue 表（status=pending，含来源与时间戳）
   ▼
collector 每周期出队 ──▶ write_register（功能码6）──▶ 控制器
   │                                                  │
   └──────────status=done/failed ◀─────────────────────┘
   ▼
GET /api/v1/{code}/commands  查看执行结果（全程可追溯）

CLI 直发：python collector.py --config ... --write 5 100 --equip 621002
```

---

## 二、目录结构

```
10_新一代参考实现_Python/
├── config/
│   └── dffd_东方饭店.yaml      # 试点项目配置（点表/控制器/采集参数/报警判据）
├── collector.py                # 采集服务（Modbus RTU-over-TCP，配置驱动）
├── api_server.py               # 取数 API（FastAPI 版，功能全，需 pip 安装）
├── api_server_lite.py          # 取数 API（零依赖版，标准库即可运行）★现场推荐
├── data/dffd.db                # SQLite 演示库（采集自动创建）
└── README.md
```

---

## 三、快速开始

### 3.1 安装依赖

```bash
pip install pyyaml                      # 必需（已随本环境安装）
pip install pymodbus                    # 真实采集时需要
pip install fastapi uvicorn httpx       # 仅 api_server.py 需要
```

> **现场老服务器若无外网**：只装 `pyyaml`，用 `api_server_lite.py`（零依赖）即可。

### 3.2 先跑通流程（模拟数据，无需设备）

```bash
python collector.py --config config/dffd_东方饭店.yaml --simulate --once
```

输出示例（17 个测点）：

```
采集 2#冷水机组：17 个测点，报警 3 条
    U_rCoWtr_InTmp       = 29.2     冷却水进口温度
    U_rChWtr_OutTmp      = 13.3     冷冻水出口温度
    U_DQFH               = 78.0     机组当前负荷
    U_YXXS               = 12020.0  总运行小时
    [报警] 冷冻水出水温度高 = 13.3 (warn)
```

数据自动写入 `data/dffd.db` 的 `tblUnitData`（**字段与现有 Schema 完全一致**）。

### 3.3 启动 API

```bash
# 零依赖版（推荐现场）
python api_server_lite.py --port 8080

# 或 FastAPI 版
uvicorn api_server:app --host 0.0.0.0 --port 8080
```

测试：

```bash
curl http://127.0.0.1:8080/api/v1/health
curl http://127.0.0.1:8080/api/v1/dffd/realtime
curl http://127.0.0.1:8080/api/v1/dffd/alarms
curl "http://127.0.0.1:8080/api/v1/dffd/history?field=U_rChWtr_OutTmp&equip_id=621002&limit=100"
curl http://127.0.0.1:8080/api/v1/dffd/snapshot
```

### 3.4 接真实设备

编辑 `config/dffd_东方饭店.yaml`：

```yaml
transport:
  host: 192.168.1.241     # 现场串口服务器 IP
  port: 4001
  framer: rtu             # RTU over TCP
  parity: N               # T25 实测无校验（顿汉布什为偶校验，勿混用）
controllers:
  - equip_id: 621002
    active: true          # 现场仅 2 号冷机启用
```

然后：

```bash
python collector.py --config config/dffd_东方饭店.yaml     # 持续采集（周期 60s）
```

---

## 四、接口一览

| 接口 | 说明 |
|---|---|
| `GET /api/v1/health` | 健康检查 |
| `GET /api/v1/projects` | 项目列表 |
| `GET /api/v1/{code}/points` | 点表（寄存器地址、字段名、系数、单位） |
| `GET /api/v1/{code}/devices` | 设备清单（机组、型号、启用状态） |
| `GET /api/v1/{code}/realtime` | 实时值（每台机组最新一条） |
| `GET /api/v1/{code}/history` | 历史序列（`field` + `equip_id` + 时间范围） |
| `GET /api/v1/{code}/alarms` | 报警（按配置判据评估） |
| `GET /api/v1/{code}/snapshot` | 一键快照（数字员工生成报告用） |
| `WS /ws/{code}` | 实时推送（仅 FastAPI 版） |

---

## 五、配置说明（核心：一个文件定义一个项目）

```yaml
project:    { code: dffd, name: 东方饭店, iPrjID: 10060801 }
transport:  { mode: rtu-over-tcp, host, port, framer, parity }
acquisition:{ func: 3, interval_sec: 60, groups: [{start,len}] }
controllers:[ { equip_id, name, model, unit, active } ]
points:     [ { reg, name, field, scale, unit } ]
output:     { table: tblUnitData, driver: sqlite|mssql }
alarms:     [ { field, name, op, threshold, level } ]
```

**新增一个项目 = 复制一份 YAML 改配置**，程序完全通用——这正是"1 个公共底座 + N 份项目配置"的落地。

### 多机型项目：point_profiles

一个项目里常有**不同型号的设备**（冷机 + 电表 + 水泵），点表各不相同。用 `point_profiles` 定义多套点表，设备用 `profile` 引用：

```yaml
point_profiles:
  chiller_york:        # 约克离心机 20 点
    - {reg: 10, name: 冷冻水进口温度, field: U_rChWtr_InTmp, scale: 0.1, unit: '℃'}
  power_meter:         # 电量仪表 10 点
    - {reg: 1,  name: A相电压, field: M_A_voltage, scale: 0.1, unit: 'V'}

controllers:
  - {equip_id: 621001, name: 1#冷水机组, profile: chiller_york, active: true}
  - {equip_id: 730001, name: 1#电表, profile: power_meter,
     groups: [{start: 1, len: 20}], active: true}      # 可覆盖分组
```

### 已内置的机型模板

| profile | 机型 | 点数 | 来源 |
|---|---|---:|---|
| `chiller_t25` | 特灵 T25 | 17 | 东方饭店实测 |
| `chiller_york` | 约克离心机 | 20 | 港中旅实测 |
| `power_meter` | 电量仪表 | 10 | 港中旅实测 |
| `chiller_dunham` | 顿汉布什冷机 | 15 | 磁器口实测 |
| `cold_station_plc` | 冷站 PLC（台达 DVP） | 16 | 磁器口实测 |
| `ac_unit_plc` | 空调机组 PLC | 11 | 磁器口实测 |

---

## 五之二、已有配置文件

| 文件 | 项目 | 状态 |
|---|---|---|
| `_template_通用项目模板.yaml` | — | **模板**（以 `_` 开头，API 不加载，供复制） |
| `dffd_东方饭店.yaml` | 东方饭店 | ✅ 实测可用（T25×3，17 点） |
| `gzl_港中旅.yaml` | 港中旅维景大酒店 | ✅ 实测可用（约克冷机×4 + 电表×2，30 点） |
| `cqk_地铁磁器口.yaml` | 地铁7号线磁器口站 | ✅ **多协议混合接入**（顿汉布什冷机 + 台达PLC + 空调机组，42 点） |
| `cfzx_财富购物中心.yaml` | 财富购物中心 | ⚠️ **待现场核实** |

### 多协议混合接入（磁器口 cqk）

磁器口是最复杂的样例：**三种设备、两条通道、三张落点表**共存于一个项目。

| 设备 | 通道 | 协议 / 功能码 | 落点表 |
|---|---|---|---|
| 顿汉布什冷机 ×3 | 串口服务器 `192.168.1.241:4001` | Modbus **RTU**（9600, **偶校验**, 8, 1），功能码 3 | `tblUnitData` |
| 冷站 PLC（台达 DVP） | 以太网 `192.168.2.221:502` | Modbus **TCP**，功能码 1（线圈）+ 3（寄存器） | `tblColdData` |
| 空调机组 PLC ×2 | 以太网 `192.168.2.226/227:1502` | Modbus **TCP**，功能码 3 | `tblACData` |

实现要点：
- **控制器级覆盖通道**：`host` / `port` / `framer` 写在 controller 上即覆盖全局 `transport`
- **控制器级覆盖落点表**：`table:` 字段，不同设备写不同表
- **地址空间分离**：线圈（功能码 1/2）与寄存器（3/4）地址可重叠，点表用 `space: coil` 指明取自线圈空间

```yaml
controllers:
  - {equip_id: 621002, profile: chiller_dunham, table: tblUnitData, unit: 2, active: true}
  - {equip_id: 221001, profile: cold_station_plc, table: tblColdData,
     host: 192.168.2.221, port: 502, framer: tcp,          # ★覆盖通道
     groups: [{start: 0, len: 8, func: 1},                 # 线圈：泵/故障状态
              {start: 1, len: 10, func: 3}], active: true} # 寄存器：流量/温度
```

> ⚠️ **校验方式**：顿汉布什原厂默认**偶校验**，特灵 T25 为**无校验**——同一项目内不同机型也可能不同，务必逐台核对。

### 机组—点表对照总表（四个项目）

| 项目 | 机组（实际） | 联网情况 | 点表来源 |
|---|---|---|---|
| **地铁磁器口** cqk | 顿汉布什**螺杆机** | 2#、3# 冷机 + 冷站PLC + 空调PLC | 厂家点表《顿汉布什机组modbus点表.pdf》+ 现役库实测 |
| **港中旅** gzl | **开利** ×4 | **仅 3#、4# 联网** | 现役库 `tblCtrlPntSet` 实测 99 点 |
| **东方饭店** dffd | **麦克维尔螺杆机** ×3 | 3 台 | 现役库实测 51 点 |
| **财富购物中心** cfzx | **约克离心机** ×3 | 待核实 | ⚠️ 备份库无数据，暂用约克模板占位 |

> ⚠️ **重要教训：`cCtlType` 字段不可作为机型判据！**
> 该字段在库里往往是**模板复制后未修改**的值——港中旅库里登记为 `T25`（特灵），
> 但实际机组是**开利**；东方饭店同样是 `T25`，实际却是**麦克维尔螺杆机**。
> 判断机型应依据：①现场铭牌 / 厂家点表；②`tblCtrlPntSet` 的测点特征交叉验证。

### 如何知道一个项目与机组联网的点表？

点表**全部记录在现役数据库的系统表**里，用配套工具一键导出：

| 系统表 | 内容 |
|---|---|
| `tblModBusConfig` | 通道：串口(COM) 还是网口(IP)、波特率/校验、IP 与端口 |
| `tblCtrlConfig` | 控制器 IP、端口、采集周期 |
| `tblModBusPntSet` | **Modbus 采集段**：起始地址 `DBg`、长度 `DLg`、功能码 `cCode`、系数 `Xsk` |
| `tblCtrlPntSet` | **测点明细**：`bSerial` = 寄存器地址、`sName`、`sField`、`sTable` |
| `tblEquList` | 设备清单 |

**一键提取**：

```bash
python extract_pointmap.py "D:\...\05_项目配置库_模块D\02_港中旅" --name gzl
# 或直指 mdb：python extract_pointmap.py "D:\...\Data\Controls.mdb" --name cqk
```

产出（`pointmap/<项目>/`）：
- `<项目>_点表台账.md`——通道 / 控制器 / 采集段 / **测点→寄存器映射**（按设备分组）/ 设备清单
- `<项目>_配置片段.yaml`——可直接粘贴进 collector 的配置骨架

已生成的台账（`pointmap/<项目>/`，每个含 `*_点表台账.md` + `*_配置片段.yaml`）：

| 项目 | 台账 | 通道 | 控制器 | 采集段 | 测点 | 说明 |
|---|---|---|---:|---:|---:|---|
| `cqk` 磁器口 | 69K | 7 | 7 | 46 | **831** | 顿汉布什螺杆机 + 台达PLC + 空调PLC |
| `gzl` 港中旅 | 8.6K | 2 | 5 | 2 | 99 | 开利 ×4（仅3#4#联网）+ 电表 ×2 |
| `dffd` 东方饭店 | 6.0K | 2 | 5 | 15 | 51 | 麦克维尔螺杆机 ×3 |
| `cfzx` 财富中心 | 5.0K | — | — | — | — | ⚠️ **待现场核实**（备份库无源库，仅骨架） |

每个台账首部均标注**实际机组**，并警示 `cCtlType` 为模板值。

生成命令（可带实际机型）：

```bash
python extract_pointmap.py "...\05_项目配置库_模块D\01_东方饭店" --name dffd --model "麦克维尔螺杆机 ×3"
python extract_pointmap.py "...\02_港中旅"   --name gzl  --model "开利机组 ×4（仅 3#、4# 联网）+ 电量仪表 ×2"
python extract_pointmap.py "...\03_地铁瓷器口" --name cqk --model "顿汉布什螺杆机（2#、3# 冷机 + PLC）"
# 财富中心：现场取得 mdb 后同样执行，替换占位台账
python extract_pointmap.py "<现场Controls.mdb>" --name cfzx --model "约克离心机 ×3"
```

### 财富购物中心说明

该项目为**云端项目**（`http://www.chinaborry.com/cfzx`，博瑞AI智控），
**历史备份库中没有它的工程数据**，故配置中 IP、点表等以 `★待填` 标注。现场核实 5 步（约 30 分钟）：

1. 打开 `CONTROLS_cfzx` 库，读 `tblCtrlConfig` → 填控制器与通道
2. 读 `tblModBusConfig` → 确认 `framer`（COM→rtu，IP→tcp）与串口参数
3. 读 `tblCtrlPntSet` → 替换点表（`bSerial` 即寄存器地址）
4. 确认 `cCtlType`，从《品牌机型适配档案》套用点表模板
5. 把 `active` 改为 `true` 启用

### 新增项目三步法

```bash
cp config/_template_通用项目模板.yaml config/xxxx_某某项目.yaml
# 编辑：项目信息 → 通道 → 控制器 → 点表（可直接引用 profile）
python collector.py --config config/xxxx_某某项目.yaml --simulate --once   # 先跑通
python collector.py --config config/xxxx_某某项目.yaml                      # 接真机
```

---

## 六、与现有系统的兼容

- **表结构沿用** `tblUnitData`（`iPrjID / lEquID / dTime / U_xxx`），存量系统可直接读
- **点表来源**为现役库实测（`tblCtrlPntSet.bSerial` = 寄存器地址，已验证）
- 支持与 YeatCtl **并行双写**：新采集写同一张表或影子表，比对一致后再切换
- 驱动可切 `mssql`，直接连现场 SQL Server

---

## 七、部署到现场

| 环境 | 方式 |
|---|---|
| 新服务器（Win2019+） | Docker Compose（一并部署采集 + API + 前端） |
| 老服务器（Win2008R） | NSSM 把 `collector.py` 与 `api_server_lite.py` 注册为 Windows 服务 |
| 极老/无 Python 环境 | 采集端改用 Go 单文件版（见方案文档） |

注册服务示例：

```
nssm install BCNSCollector "C:\Python38\python.exe" "C:\bcns\collector.py --config C:\bcns\config\dffd.yaml"
nssm install BCNSApi "C:\Python38\python.exe" "C:\bcns\api_server_lite.py --port 8080"
```

---

## 八、下一步

- [ ] 磁器口配置（`config/cqk_地铁磁器口.yaml`）——验证顿汉布什 + 台达 PLC + S7-200 混合接入
- [ ] 与 YeatCtl 双写比对，出具一致性报告
- [ ] Vue3 前端（去 Flash、支持手机）
- [ ] Web 组态工具（替代 VB6）
