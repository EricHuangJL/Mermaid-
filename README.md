flowchart TD

%% 颜色主题
classDef startstop fill:#4CAF50,color:#FFFFFF,stroke:#2E7D32,stroke-width:2px
classDef dbus      fill:#00BCD4,color:#FFFFFF,stroke:#00838F,stroke-width:2px
classDef mctp      fill:#FF9800,color:#FFFFFF,stroke:#EF6C00,stroke-width:2px
classDef nic       fill:#3F51B5,color:#FFFFFF,stroke:#283593,stroke-width:2px
classDef event     fill:#9C27B0,color:#FFFFFF,stroke:#6A1B9A,stroke-width:2px
classDef decision  fill:#FFC107,color:#000000,stroke:#FFA000,stroke-width:2px
classDef error     fill:#F44336,color:#FFFFFF,stroke:#C62828,stroke-width:2px
classDef op        fill:#ECEFF1,color:#000000,stroke:#90A4AE,stroke-width:1px

%% 主流程 MAIN
subgraph MAIN_服务启动与初始化
  direction TB
  M_START[程序启动 main 开始]:::startstop --> M_OS[创建 object server 并注册 manager 路径 sensors 与 inventory]:::dbus
  M_OS --> M_NAME[请求总线名 xyz.openbmc_project.NicManager]:::dbus
  M_NAME --> M_CFG[创建 MCTP 配置 使用 NCSI 加 PCIe VDM]:::mctp
  M_CFG --> M_WRAP[创建 MCTPWrapper 实例]:::mctp
  M_WRAP --> M_MATCH[注册 MCTP 信号监听 InterfacesAdded 与 InterfacesRemoved]:::event
  M_MATCH --> M_SCAN[主动扫描 调用 mctpHandle]:::mctp
  M_SCAN --> M_POST[注册主机状态监听 POST 完成属性变化]:::event
  M_POST --> M_RUN[进入事件循环 io run]:::startstop
end

%% mctpHandle 协程
subgraph HANDLE_mctpHandle 主动发现
  direction TB
  H_SPAWN[启动协程]:::op --> H_DET[探测 MCTP 端点 detectMctpEndpoints]:::mctp
  H_DET --> H_OK{探测是否成功}:::decision
  H_OK -->|否| H_ERR[记录错误并返回]:::error
  H_OK -->|是| H_MAP[获取端点映射 getEndpointMap]:::mctp
  H_MAP --> H_EACH{遍历每个 EID}:::decision
  H_EACH -->|下一项| H_MAKE[调用 creatNicDevice 进行创建设备]:::nic
end

%% MCTP 接口添加
subgraph ADDED_InterfacesAdded 回调
  direction TB
  IA_SIG[收到 InterfacesAdded 信号]:::event --> IA_PATH{路径是否以 mctp 前缀开头}:::decision
  IA_PATH -->|否| IA_IGN[忽略本次信号]:::op
  IA_PATH -->|是| IA_MSGT[读取接口 SupportedMessageTypes]:::dbus
  IA_MSGT --> IA_NCSI{是否支持 NCSI 类型}:::decision
  IA_NCSI -->|否| IA_IGN2[忽略本次信号]:::op
  IA_NCSI -->|是| IA_EID[从对象路径解析 EID 数值]:::mctp
  IA_EID --> IA_SPAWN[启动协程 再次探测并创建设备]:::mctp
  IA_SPAWN --> IA_MAKE[调用 creatNicDevice 针对该 EID]:::nic
end

%% MCTP 接口移除
subgraph REMOVED_InterfacesRemoved 回调
  direction TB
  IR_SIG[收到 InterfacesRemoved 信号]:::event --> IR_PATH{路径是否以 mctp 前缀开头}:::decision
  IR_PATH -->|否| IR_IGN[忽略本次信号]:::op
  IR_PATH -->|是| IR_TYPE{移除列表包含 SupportedMessageTypes}:::decision
  IR_TYPE -->|否| IR_IGN2[忽略本次信号]:::op
  IR_TYPE -->|是| IR_EID[从对象路径解析 EID 数值]:::mctp
  IR_EID --> IR_STOP[遍历 Nics 找到匹配 EID 的对象并停止采集 setInit 为 false]:::nic
end

%% creatNicDevice 逻辑
subgraph CREATE_creatNicDevice 针对单个 EID 创建设备
  direction TB
  C_EIDP[拼装 eidpath 形如 mctp 路径加 EID]:::op --> C_GETALL[读取 MCTP PCIe 端点属性 GetAll 包含 Bus Device Function]:::dbus
  C_GETALL --> C_EC{调用是否出错}:::decision
  C_EC -->|是| C_RET1[记录告警并返回]:::error
  C_EC -->|否| C_PARSE[解析属性并填充 mctpconf 与十六进制字符串]:::op
  C_PARSE --> C_BDFOK{是否成功获取 B D F 属性}:::decision
  C_BDFOK -->|否| C_RET2[记录错误并返回]:::error
  C_BDFOK -->|是| C_BDF[组装 bdfPath 例如 00_bus_device 注意当前未含 function]:::op
  C_BDF --> C_HAS{Nics 是否已包含该 bdfPath}:::decision
  C_HAS -->|是| C_REINIT[执行重初始化 停止 更新 EID 基础重建 重新启动读取]:::nic
  C_HAS -->|否| C_MAP[通过 ObjectMapper GetSubTree 查找 PCIeDevice 节点]:::dbus
  C_MAP --> C_EC2{是否出错或结果为空}:::decision
  C_EC2 -->|是| C_RET3[记录告警并返回]:::error
  C_EC2 -->|否| C_FIND{是否找到对象名等于 bdfPath 的节点}:::decision
  C_FIND -->|否| C_RET4[未匹配到合适节点 返回]:::op
  C_FIND -->|是| C_GETSLOT[读取 PCIeDevice 属性 SlotNumber]:::dbus
  C_GETSLOT --> C_EC3{读取是否出错}:::decision
  C_EC3 -->|是| C_RET5[记录告警并返回]:::error
  C_EC3 -->|否| C_SLOTS[解析十六进制 SlotNumber 并转为形如 P 加十六进制]:::op
  C_SLOTS --> C_PATH[组合库存路径 nicBasePath 加 slotNumber]:::op
  C_PATH --> C_NEW[调用 createNic 工厂函数创建具体 NIC 实例]:::nic
  C_NEW --> C_SENS[nic 创建传感器 并启动读取]:::nic
  C_SENS --> C_SAVE[加入全局映射 Nics 使用 bdfPath 作为键]:::nic
end

%% createNic 工厂
subgraph FACTORY_createNic 选择具体 NIC 类
  direction TB
  CN_GET[getDeviceInfo 读取设备四元组 DID VID SSID SVID]:::mctp --> CN_HAS{是否获取到有效设备信息}:::decision
  CN_HAS -->|否| CN_DEF[返回 GeneralNic 通用实现]:::nic
  CN_HAS -->|是| CN_MAP{在 nicMap 中是否命中型号映射}:::decision
  CN_MAP -->|是| CN_SPEC[返回具体子类 例如 MellanoxNic 或 BroadcomNic]:::nic
  CN_MAP -->|否| CN_DEF2[返回 GeneralNic 通用实现]:::nic
end

%% getDeviceInfo 读取设备信息
subgraph INFO_getDeviceInfo 基于 NCSI 识别设备
  direction TB
  GD_DEV[构造 NCSIDev 使用传入 EID]:::mctp --> GD_SEL[按优先顺序尝试 selectPackage]:::mctp
  GD_SEL --> GD_VER[调用 getVersionId 读取版本与 PCI 标识]:::mctp
  GD_VER --> GD_OK{调用是否出错}:::decision
  GD_OK -->|是| GD_FAIL[记录错误并返回空结果]:::error
  GD_OK -->|否| GD_KEY[组合 Key 使用 htons 转换字节序]:::op
  GD_KEY --> GD_RET[返回设备标识 Key 用于型号映射]:::op
end

%% 连接各分区的关键跳转
M_SCAN --> H_SPAWN
IA_MAKE --> C_EIDP
IR_STOP --> M_RUN
H_MAKE --> C_EIDP
C_NEW --> CN_GET
CN_GET --> GD_DEV
