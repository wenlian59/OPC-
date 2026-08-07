# SharpSCADA OPC UA 接入方案与验证记录

## 1. 已完成内容

### 1.1 工程结构

新增 OPC UA 驱动工程：

```text
work/SharpSCADA/SCADA/Program/CoreApp/DataService/OpcUaDriver/
  OpcUaDriver.csproj
  OpcUaReader.cs
  OpcUaGroup.cs
```

`OpcUaDriver.csproj` 目标框架为 `net10.0`，引用：

- `OPCFoundation.NetStandard.Opc.Ua.Client`
- `OPCFoundation.NetStandard.Opc.Ua.Configuration`
- 现有 `DataService.csproj`

`DataService.sln` 和 `GateWay.csproj` 已加入该工程，Gateway 输出目录包含 `OpcUaDriver.dll` 和 `Opc.Ua.*.dll`。

### 1.2 驱动实现

`OpcUaReader` 实现 `IPLCDriver`、`IMultiReadWrite`：

- 构造函数匹配 `DAService.AddDriver` 反射签名。
- `GetDeviceAddress` 使用 `NodeId.Parse`，句柄保存在驱动内部，不改动 `DeviceAddress` 结构。
- `Connect` 构建 `ApplicationConfiguration`，执行 Endpoint 发现并创建 UA Session。
- 支持 `ReadValues` / `WriteValues` 批量读写。
- 支持 KeepAlive 与 `SessionReconnectHandler` 重连，重连替换 Session 时会先清理旧订阅再重建。

`OpcUaGroup` 实现完整 `IGroup`：

- `AddItems` 先保存标签元数据，Session 就绪后再创建 `Subscription` 和 `MonitoredItem`。
- `IsActive` 控制 `PublishingEnabled` / `MonitoringMode`。
- `MonitoredItem.Notification` 将 UA `DataValue` 转为 SharpSCADA `Storage`，更新 `ITag` 并触发 `DataChange`。
- 订阅不可用或组不活跃时，`BatchRead` 回退到 `IMultiReadWrite`。
- 字符串通过 `StringTag.String` 单独维护。

### 1.3 配置注册

新增：

```text
outputs/config/opcua_config.sql
```

注册关系：

- `registermodule.DriverID = 11`，`ClassFullName = OpcUaDriver.OpcUaReader`
- `meta_driver.DriverID = 3`，`DriverType = 11`
- `meta_group.GroupID = 20006`，组名 `OpcUaTest`
- `meta_tag.TagID = 3101-3106`

`config/paths.json` 新增 `opcuaDriverDll`；`apply_config.ps1` 的 `ValidateSet` 已加入 `opcua`。

> 原计划的 `GroupID=20005 / TagID=3001+` 已被现有 S7 `PLCBatch` 占用，因此使用 `20006 / 3101+`，避免覆盖现有配置。

### 1.4 验证工具

```text
outputs/probes/opcua_probe/
outputs/probes/opcua_connect_probe/
```

该 probe 可验证：

- Connect / Read / Write
- Subscription 创建
- DataChange 回调
- BatchRead / BatchWrite
- Dispose 后 Session、Subscription、线程状态

`opcua_connect_probe` 用于快速验证任意 OPC UA 端点，例如实机 PLC：

```powershell
dotnet run --project 'D:\OPC_hzy\outputs\probes\opcua_connect_probe\opcua_connect_probe.csproj' -- 'opc.tcp://192.168.1.1:4840' 'ns=3;s=DB1_Int'
```

## 2. 类型与质量映射

| OPC UA 类型 | SharpSCADA 类型 | 说明 |
| --- | --- | --- |
| Boolean | BOOL | 已支持 |
| Byte / SByte | BYTE | 已支持 |
| Int16 / UInt16 | SHORT | 已支持，超出范围转换失败会返回空值 |
| Int32 / UInt32 | INT | 已支持 |
| Float / Double | FLOAT | Double 会转为 Single |
| String | STR | 通过 `StringTag.String` 维护 |
| Int64 / UInt64 / DateTime / ByteString | 不映射 | 当前安全跳过 |
| 数组 / Variant / 复杂结构 | 不映射 | 当前安全跳过 |

质量映射：

| OPC UA | SharpSCADA |
| --- | --- |
| Good | QUALITY_GOOD |
| Uncertain | QUALITY_UNCERTAIN |
| Bad | QUALITY_BAD |

## 3. 手动验证步骤

### 3.1 启动环境

先启动 OPC UA 参考服务器：

```powershell
Start-Process 'D:\OPC_hzy\work\UA-.NETStandard\samples\ConsoleReferenceServer\bin\Debug\net10.0\ConsoleReferenceServer.exe' -WorkingDirectory 'D:\OPC_hzy\work\UA-.NETStandard\samples\ConsoleReferenceServer\bin\Debug\net10.0' -WindowStyle Hidden
```

再启动 MariaDB / Modbus 模拟器 / Gateway：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File 'D:\OPC_hzy\work\restart_all.ps1'
```

确认端口：

```powershell
netstat -ano | Select-String ':3306|:502|:62541|:6543'
```

### 3.2 应用 OPC UA 配置

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File 'D:\OPC_hzy\outputs\config\apply_config.ps1' -Config opcua
```

DryRun 检查：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File 'D:\OPC_hzy\outputs\config\apply_config.ps1' -Config opcua -DryRun
```

### 3.3 检查 OPC UA 链路

Gateway 启动后等待数秒，检查：

```powershell
Get-Content 'D:\OPC_hzy\work\gw_debug.log' -Tail 50
```

应看到：

```text
OPCUA OPC UA group OpcUaTest subscription ready, items=6
```

`gw_dataevent.log` 应持续产生：

```text
DATAEVENT count=1 sockets=0
```

### 3.4 查询实时值

```powershell
python D:\OPC_hzy\outputs\probes\query_tag.py 3101 1
python D:\OPC_hzy\outputs\probes\query_tag.py 3104 4
python D:\OPC_hzy\outputs\probes\query_tag.py 3105 4
```

返回的 `value=` 段应随时间变化。

### 3.5 检查归档

`log_hdata` 默认在 `MaxHdaCap=10000` 或 1 小时定时时落库。查询最近 OPC UA 归档：

```sql
SELECT ID, TimeStamp, Value
FROM scada.log_hdata
WHERE ID IN (3101,3102,3103,3104,3105,3106)
ORDER BY TimeStamp DESC
LIMIT 20;
```

### 3.6 回归 probe

```powershell
dotnet run --project 'D:\OPC_hzy\outputs\probes\opcua_probe\opcua_probe.csproj'
```

应看到：

- `Connected=True`
- `subscription=Subscription monitored=3`
- `BatchRead` 全部 `QUALITY_GOOD`
- `BatchWrite=0`
- `DataChange` 持续产生
- Dispose 后 `subscription=null monitored=0`
- `SessionAfterDispose=null`

### 3.7 命令行连接 probe

先只验证端点是否能连接：

```powershell
dotnet run --project 'D:\OPC_hzy\outputs\probes\opcua_connect_probe\opcua_connect_probe.csproj' -- 'opc.tcp://192.168.1.1:4840'
```

`Connected=True` 后再带上 NodeId 验证读取：

```powershell
dotnet run --project 'D:\OPC_hzy\outputs\probes\opcua_connect_probe\opcua_connect_probe.csproj' -- 'opc.tcp://192.168.1.1:4840' 'ns=3;s=DB1_Int'
```

如果 NodeId 本身包含双引号，例如 Siemens 的 `ns=3;s="DB10"."wTest"`，PowerShell 会剥掉双引号，应改用环境变量传递：

```powershell
$env:OPCUA_NODE_ID='ns=3;s="DB10"."wTest"'
dotnet run --project 'D:\OPC_hzy\outputs\probes\opcua_connect_probe\opcua_connect_probe.csproj' -- 'opc.tcp://192.168.1.1:4840'
```

probe 会先打印 `NodeIdArg=...` 和长度，方便确认引号是否完整保留。

## 4. 验证记录

- 阶段 3：probe 验证订阅回调，Int32 / Float / String 均正常。
- 阶段 5：配置入库并重启 Gateway；`query_tag.py` 实时值正常；`log_hdata` 已写入 `3101-3106`。
- 阶段 6：停 ReferenceServer 后记录 `KeepAlive lost: BadSecureChannelClosed`，数据事件停止；重启后 `Reconnect complete: connected`，`subscription ready, items=10`，数据恢复。
- 异常标签：无效 NodeId、数组、Variant、Int64、未配置 GroupID 均不拖垮 Gateway；验证后已清理。
- Dispose：连续创建/销毁驱动实例，线程数稳定，Session 与 Subscription 均为空。

## 5. 连接真实 PLC 的 OPC UA

### 5.1 前置条件

- PLC 或网关必须启用 OPC UA Server。
- 已知 OPC UA 端点 URL，例如：
  - `opc.tcp://192.168.1.10:4840`
  - `opc.tcp://192.168.1.10:4840/UA/Server`
- 已知需要采集的节点 ID，例如：
  - `ns=2;s=DB1_Int`
  - `ns=2;i=1001`
- 当前驱动只支持 `MessageSecurityMode.None + SecurityPolicy.None`。

### 5.2 修改配置

1. 用 UA Expert 或其他 OPC UA 客户端 Browse 出实际节点 ID。
2. 打开 `outputs/config/opcua_config.sql`，把 `meta_driver.Server` 改为 PLC 端点。
3. 把 `meta_tag.Address` 改为真实节点 ID，保留 `DataType` / `DataSize`。
4. 需要用户名密码时，把 `meta_driver.Spare2` 写成 `用户名:密码`。
5. 需要自定义证书目录时，把 `meta_driver.Spare1` 写成 PKI 根目录。
6. 重新执行：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File 'D:\OPC_hzy\outputs\config\apply_config.ps1' -Config opcua
```

7. 重启 Gateway 并检查 `gw_debug.log` 与 `gw_dataevent.log`。

### 5.3 安全认证

当前阶段为安全模式 `None`。若 PLC 强制要求签名或加密：

- 需要把驱动升级到支持 `Basic256Sha256` 和证书信任链。
- 把 PLC 证书加入客户端 trusted store。
- 把客户端证书加入 PLC 的信任列表。

这部分属于方案中的 2.0 升级路径。

### 5.4 当前限制

- 数组、复杂结构、Int64、DateTime、ByteString 等类型当前不会映射。
- 字符串可参与实时链路，但 `log_hdata.Value` 是 float 字段，字符串归档值不会写入该列。
- `meta_tag.Address` 必须能通过 `NodeId.Parse` 解析。
- 建议先用 UA Expert 验证 PLC 节点可读可订阅，再接入 SharpSCADA。

### 5.5 SIMATIC S7-1500 实机接入

目标设备：

- SIMATIC S7-1500 CPU 1515T-2 PN
- 订货号：`6ES7 515-2TN03-0AB0`

S7-1500 自带 OPC UA Server，不需要额外软件网关。默认端口为 `4840`。

#### TIA Portal 侧配置

1. 打开 TIA Portal 项目，选择 CPU。
2. 进入设备属性：
   - `OPC UA > Server > 激活 OPC UA Server`
   - 确认端口为 `4840`
3. 安全策略：
   - 当前 SharpSCADA 驱动只支持 `None`。
   - 测试阶段在 TIA 中允许 `None` 安全策略。
   - 若 PLC 固件/项目强制要求签名或加密，需要先完成 2.0 安全升级。
4. 接口设置：
   - 选择 PLC 使用的 PROFINET 接口和 IP。
   - 例如 `192.168.1.1`，则端点通常为：

```text
opc.tcp://192.168.1.1:4840
```

5. 用户与认证：
   - 测试阶段可启用匿名访问，或创建 OPC UA 用户。
   - 若使用用户名密码，写入 `meta_driver.Spare2`：

```text
用户名:密码
```

6. DB 变量可见性：
   - 需要采集的 DB 变量必须在 DB 属性中允许 OPC UA 访问。
   - 优化 DB 也可以被 OPC UA Server 暴露，但要以 UA Expert 实际 Browse 结果为准。

#### OPC UA 许可证

S7-1500 的 OPC UA Server 是收费选项。在博图中激活 OPC UA Server 但选择“无许可证”时，编译/下载会报“许可证不足”，这是正常行为，不是工程配置错误。

处理方式：

1. 购买 S7-1500 OPC UA 正式授权。
   - 常见订货号：`6ES7822-0AA00-0YA0`
   - 最终以西门子官方报价和当地销售确认为准。
2. 通过 Automation License Manager 安装 License Key。
3. 回到博图：
   - 设备树 → `Runtime licenses` → `OPC UA`
   - 选择已安装的授权，而不是“无许可证”。
4. 部分版本支持 60 天试用授权：
   - 可在 `Runtime licenses` 中尝试激活试用授权。
   - 试用授权通常只能激活一次，且过期后需要正式授权。
5. 如果当前只是想验证 SharpSCADA OPC UA 驱动链路，可以继续使用本地 ReferenceServer，不需要 PLC 授权。

#### UA Expert 确认节点

1. 用 UA Expert 连接：

```text
opc.tcp://192.168.1.1:4840
```

2. 浏览：

```text
Objects > DeviceSet > PLC_1 > ...
```

3. 找到需要采集的标量变量，复制其 NodeId，例如：

```text
ns=3;s=...
ns=3;i=...
```

4. 先做 Read、Write、Subscribe 测试，确认节点可读且数值随时间变化。

#### 修改 SharpSCADA 配置

编辑 `outputs/config/opcua_config.sql`：

脚本中已内置一个 PLC 示例驱动实例：

- `meta_driver.DriverID = 4`
- `meta_group.GroupID = 20007`
- `meta_tag.TagID = 3201`，地址为 `ns=3;s="DB10"."wTest"`

本地 ReferenceServer 仍使用 `DriverID=3 / GroupID=20006`，不会被 PLC 配置覆盖。若 PLC IP 或 NodeId 不同，直接修改脚本中 `DriverID=4` 对应的 `Server` 和 `Address`。

重新应用并重启 Gateway：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File 'D:\OPC_hzy\outputs\config\apply_config.ps1' -Config opcua
```

然后检查 `gw_debug.log` 和 `gw_dataevent.log`。

#### S7-1500 常见坑

- PLC 端不允许 `None` 时，驱动会拒绝连接，日志出现安全模式相关错误。
- 结构化 DB 或数组节点当前不会映射，先选一个标量变量验证。
- NodeId 的 namespace index 可能不是 `2`，必须用 UA Expert 里的实际值。
- 首次连接如果涉及证书，需要把客户端证书加入 PLC 信任列表；纯 `None` 模式通常不涉及。

### 5.6 2.0 升级路径

后续可将 `OpcUaReader` 迁移到 `ManagedSession` / `ISessionFactory`，以获得：

- 原生安全策略与证书管理。
- 更完整的会话生命周期和故障转移。
- 复杂类型解码。
- 按 MonitoredItem 返回错误诊断，替代当前“整组失败即回退”的策略。
- 更细的订阅恢复与发布补偿控制。
