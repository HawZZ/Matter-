# Matter学习问答

1. 在开发新的matter产品时，是谁来定义cluster？
   答：标准Cluster由CSA统一定义和维护，参考文档[Matter Application Cluster Specification](https://csa-iot.org/wp-content/uploads/2023/10/Matter-1.2-Application-Cluster-Specification.pdf)。
   自定义Cluster可以实现厂商特定功能，但是如果需要实现与其他厂商设备的互操作性就需要申请认证。
   扩展现有Cluster，通过标注`manufacturerCode`可以实现属性和命令中添加扩展字段

2. 部分属性在Scenes Cluster中明确强制的版本是1.2，之前的版本不强制，导致如果简单调用Scenes命令去实现场景控制，部分设备存在不兼容的情况。
   通常可以通过`Cluster Discovery`检测设备是否支持Scenes控制某个属性，用于判断兼容情况。如果不兼容，在调用场景时，可以针对不兼容设备切换到Command指令模式。

3. 场景表存储在哪里？

   | 角色                            | 部署位置                                           | 核心职责                                                     |
   | ------------------------------- | -------------------------------------------------- | ------------------------------------------------------------ |
   | **Scenes Server**（场景服务器） | 每个 “支持场景的目标设备” 上（如智能灯、智能插座） | 1. **存储本地场景表**（记录该设备在不同场景下的属性状态，如自身的 OnOff、亮度）；<br />2. 响应 Client 的命令（如 AddScene 存储场景、RecallScene 恢复场景）；<br />3. 自主执行场景状态恢复（从本地场景表读取数据，更新自身硬件状态）。 |
   | **Scenes Client**（场景客户端） | Controller 上（如手机 APP、智能网关、语音音箱）    | 1. 发起场景管理命令（AddScene/StoreScene/RecallScene 等）；<br />2. 管理 “场景元信息”（如场景名称、场景 ID、关联的设备列表）；<br />3. 协调多个设备同步执行场景（如调用 “睡眠场景” 时，向所有关联灯的 Scenes Server 发送 RecallScene 命令）。 |

4. 为什么 Matter 要设计成 “场景表存在设备端”？

   这种分布式存储设计是 Matter 协议 “去中心化、高可靠” 的核心体现，主要有 3 个优势：

   1. **降低 Controller 依赖**：即使 Controller（如 APP、网关）离线，若设备支持本地触发（如智能灯自带场景按键），仍能通过设备自身的 Scenes Server 调用场景（例如按灯上的 “睡眠场景” 按键，灯直接读取本地场景表关闭），避免 “Controller 离线则场景失效” 的问题。
   2. **减少网络开销**：场景调用时，Controller 仅需发送 “场景 ID”（几个字节），无需发送所有设备的属性状态（若场景关联 10 个设备，每个设备需传 OnOff、亮度等参数，数据量会大幅增加），尤其适合弱网环境。
   3. **提高可靠性**：每个设备只负责自己的场景状态，若某一个设备离线，不影响其他设备的场景执行（例如卧室灯离线，客厅灯仍能正常读取本地场景表关闭），避免 “一个设备故障导致整个场景失效”。

5. 不同的Controller如何共享场景？
   1. **Matter 协议本身仅定义了 “设备与 Controller 的交互规则”，并未直接规定 “Controller 之间的场景共享机制”**。跨 Controller 共享场景的实现，主要依赖于**上层生态系统（如苹果 HomeKit、谷歌 Home、亚马逊 Alexa）的统一管理**，以及 Matter 的 “Fabric（织物）” 和 “Group（群组）” 能力作为底层支撑。
   2. 跨生态共享场景无法实现，一般是手动在不同生态配置同样的场景。
   3. Matter1.3版本支持Scene Sharing功能，可以通过`shareScene`在局域网中给不同的controller同步场景。
