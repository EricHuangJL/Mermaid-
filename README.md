```mermaid
flowchart TD
  A[程序启动 main()] --> B[创建 object_server 并 add_manager<br/>/xyz/openbmc_project/sensors<br/>/xyz/openbmc_project/inventory]
  B --> C[conn->request_name<br/>xyz.openbmc_project.NicManager]
  C --> D[构造 MCTPConfiguration<br/>(MessageType=NCSI, Binding=MCTP over PCIe VDM)]
  D --> E[创建 mctpWrapper]
  E --> F[注册 MCTP D-Bus 匹配器<br/>监听 /xyz/openbmc_project/mctp<br/>InterfacesAdded/Removed]
  F --> G[主动扫描 mctpHandle(conn, mctpWrapper)]
  G --> H[注册 hostEventMatch<br/>监听 POST 完成状态变化]
  H --> I[io.run() 事件循环常驻]

  subgraph S1[函数 mctpHandle(conn, mctpWrapper)]
    S1a[spawn 协程] --> S1b[mctpWrapper->detectMctpEndpoints(yield)]
    S1b -->|成功| S1c[getEndpointMap()]
    S1c --> S1d{遍历每个 EID}
    S1d --> S1e[creatNicDevice(conn, mctpWrapper, eid)]
    S1b -->|失败| S1f[lg2::error 并返回]
  end

  subgraph S2[ifacesAddedCallback(msg)]
    X1[收到 InterfacesAdded 信号] --> X2{路径以<br/>/xyz/openbmc_project/mctp/ 开头?}
    X2 -->|否| Xend1[忽略]
    X2 -->|是| X3[取接口<br/>xyz.openbmc_project.MCTP.SupportedMessageTypes]
    X3 --> X4{属性 NCSI 为 true?}
    X4 -->|否| Xend2[忽略]
    X4 -->|是| X5[解析 EID（路径最后一段）]
    X5 --> X6[spawn 协程：<br/>detectMctpEndpoints → creatNicDevice(eid)]
  end

  subgraph S3[ifacesRemovedCallback(msg)]
    Y1[收到 InterfacesRemoved 信号] --> Y2{路径以<br/>/xyz/openbmc_project/mctp/ 开头?}
    Y2 -->|否| Yend1[忽略]
    Y2 -->|是| Y3{移除接口列表包含<br/>SupportedMessageTypes?}
    Y3 -->|否| Yend2[忽略]
    Y3 -->|是| Y4[解析 EID]
    Y4 --> Y5[遍历 Nics，匹配 EID<br/>对命中对象 setInit(false)]
  end

  subgraph S4[hostEventMatch (POST 完成状态)]
    Z1[收到 PropertiesChanged] --> Z2{property == 'Functional'?}
    Z2 -->|否| Zend[忽略]
    Z2 -->|是| Z3{Functional == true?}
    Z3 -->|是| Z4[调用 mctpHandle 再次扫描]
    Z3 -->|否| Z5[stopHandle(): 遍历 Nics setInit(false)]
  end

  subgraph S5[creatNicDevice(conn, mctpWrapper, eid)]
    C1[eidpath = mctp_pcie::mctpManagerPath + eid] --> C2[Properties.GetAll<br/>接口: mctp_pcie::pdInterface<br/>读取 Bus/Device/Function]
    C2 --> C3{ec?}
    C3 -->|错误| Cend1[警告日志并返回]
    C3 -->|成功| C4[填充 mctpconf(bus, device, function)<br/>并转为十六进制字符串]
    C4 --> C5{BDF 合法?（代码现检查 bus/device/device）}
    C5 -->|不合法| Cend2[错误日志并返回]
    C5 -->|合法| C6[bdfPath = '00_' + bus + '_' + device<br/>(注意当前未含 function)]
    C6 --> C7{Nics.contains(bdfPath)?}
    C7 -->|是| C8[重初始化：<br/>setInit(false) → updateEid → baseReInit → setInit(true) → setupRead → 返回]
    C7 -->|否| C9[ObjectMapper.GetSubTree<br/>根:/.../inventory/.../pcie/<br/>接口: PCIeDevice]
    C9 --> C10{ec2 或 subtree 为空?}
    C10 -->|是| Cend3[警告并返回]
    C10 -->|否| C11{遍历 subtree<br/>是否有 objName == bdfPath?}
    C11 -->|无| Cend4[未匹配到，返回]
    C11 -->|有| C12[owner = 第一个服务名]
    C12 --> C13[Properties.Get 'SlotNumber'<br/>接口: PCIeDevice]
    C13 --> C14{ec3?}
    C14 -->|错误| Cend5[警告并返回]
    C14 -->|成功| C15{SlotNumber 字符串有效?}
    C15 -->|否| Cend6[返回]
    C15 -->|是| C16[去 0x 前缀 → std::stoi(hex)<br/>→ 小写十六进制 → 'P'+值]
    C16 --> C17[nicSlotPath = nicBasePath + slotNumber]
    C17 --> C18[调用 createNic(conn, mctpWrapper, nicSlotPath, mctpconf)]
    C18 --> C19[nic->createSensors(); nic->setupRead()]
    C19 --> C20[Nics.emplace(bdfPath, nic)]
  end

  subgraph S6[createNic(...)]
    K1[getDeviceInfo(mctpWrapper, eid)] --> K2{返回了 Key?}
    K2 -->|否| K6[返回 new GeneralNic]
    K2 -->|是| K3{nicMap.contains(Key)?}
    K3 -->|是| K4[调用映射工厂构造<br/>Mellanox/Broadcom 等子类]
    K3 -->|否| K5[返回 new GeneralNic]
  end

  subgraph S7[getDeviceInfo(mctpWrapper, eid)]
    G1[构造 ncsi::NCSIDev(eid)] --> G2[按优先顺序 selectPackage]
    G2 --> G3[getVersionId(packageID, 0x00, versionID)]
    G3 --> G4{ec?}
    G4 -->|错误| Gend1[记录错误并返回空/默认]
    G4 -->|成功| G5[Key = htons(DID, VID, SSID, SVID)]
    G5 --> Gend2[返回 Key]
  end
  ```
