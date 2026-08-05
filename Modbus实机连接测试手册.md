# Modbus TCP 实机连接测试手册

> 首次部署先运行：
> - powershell -File <REPO_ROOT>\config\Sync-DataConfig.ps1
> - powershell -File <REPO_ROOT>\config\Update-PathsProps.ps1


配套文档：`S7实机连接测试手册.md`（网络/IP/TIA 基本功）、`modbus_config.sql`（数据库切换）、`modbus_probe.py`（直连探测）。

## 0. 目标

网关从 S7 协议（TCP 102）切到 Modbus TCP 协议（TCP 502），用同一台
`SIMATIC S7-1500 CPU 1515T-2 PN`（IP `192.168.1.1`）读 DB1 里的测试值。
S7 那套配置和标签保留，方便对比和回退。

## 1. 网络

沿用 S7 方案，不需要改：

| 设备 | IP |
| --- | --- |
| PLC | `192.168.1.1` |
| 笔记本网口 | `192.168.1.50` |
| 虚拟机 Windows（TIA V20） | `192.168.1.60` |

S7 走 102 端口，Modbus TCP 走 502 端口，两者共用同一根网线、同一个网口，互不冲突。

## 2. PLC 侧：在 TIA V20 里加 MB_SERVER（关键步骤）

S7-1500 默认不会提供 Modbus TCP 服务，必须在 PLC 程序里用 `MB_SERVER` 指令
把数据块映射成 Modbus 保持寄存器。这一步不做，宿主机怎么改都连不上 502。

### 2.1 添加指令

1. 打开 OB1（主程序）。
2. 在右侧指令列表找：`通信 -> 开放式用户通信 -> Modbus TCP`，把 `MB_SERVER`
   拖进 OB1 的一个网络。
3. 拖入时 TIA 会弹“调用选项”，这时创建的是**背景数据块（实例 DB）**，例如
   `MB_SERVER_DB`。它保存 MB_SERVER 的运行数据；`CONNECT` 是另一个“连接描述
   数据块”，保存网络参数（协议、端口），两者不是一回事。

### 2.1.1 创建 CONNECT 连接（方法 A：手动建 TCON_IP_v4，推荐）

1. 在项目树的 PLC 程序块下双击“添加新块”，选“数据块 (DB)”，起名
   `MB_TCON`，点“确定”。
2. 打开 `MB_TCON`，新建一个变量（例如 `Server_Connect`），在“数据类型”列
   手动输入 `TCON_IP_v4`（TIA 自带的系统数据类型，输入时会有自动补全）。
3. 展开 `Server_Connect`，按下表填：

   | 成员 | 值 | 说明 |
   | --- | --- | --- |
   | `InterfaceId` | `64`（以你项目实际值为准） | PROFINET 接口的硬件标识符 |
   | `ID` | `1` | 连接编号，1-4095 内唯一 |
   | `ConnectionType` | `16#0B` | TCP |
   | `ActiveEstablished` | `FALSE`（0） | 服务器被动监听 |
   | `RemoteAddress` | `0, 0, 0, 0` | 4 个字节的数组，全 0 = 接受任意客户端 |
   | `RemotePort` | `0` | 服务器侧固定为 0 |
   | `LocalPort` | `502` | Modbus TCP 端口 |

   `BlockType` 保持默认 `16#0101`，不要改。
4. 回到 OB1 的 `MB_SERVER` 调用，在 `CONNECT` 引脚的值输入框里直接填
   `"MB_TCON".Server_Connect`，不需要找“...”按钮。

`InterfaceId` 的确认方法：设备组态里选中 CPU 的 PROFINET 接口（X1），在属性
面板找“硬件标识符”，或用“系统常量”表确认。1515T-2 PN 的集成 PN 口通常是
`64`，但以你项目里显示的值为主。

`RemoteAddress` 是 `ARRAY[1..4] OF BYTE`，要在 DB 里展开成员逐个填 0、0、0、0；
不要填成字符串 `"0.0.0.0"`，否则编译会报类型错误。

### 2.1.2 创建 CONNECT 连接（方法 B：图形化新建，版本不同位置不同）

如果 MB_SERVER 调用块的 `CONNECT` 行右侧确实有“...”或下拉按钮，也可以用
图形化方式：

1. 单击 `CONNECT` 行右侧输入框，点出现的按钮，选“新建连接”。
2. 连接类型/协议选 `Modbus TCP`，不勾选“主动建立连接”，本地端口 `502`，
   本地接口选 PROFINET 接口（X1）。
3. 确定后 `CONNECT` 自动填上连接名，无需手动改。

### 2.2 填参数

| 参数 | 值 | 说明 |
| --- | --- | --- |
| `MB_MODE` | `0` | 响应保持寄存器功能码 FC3/FC6/FC16 |
| `MB_IP_PORT` | `502` | 监听端口 |
| `MB_HOLD_REG` | `P#DB1.DBX0.0 WORD 6` | 保持寄存器数据区指向 DB1 开头 |
| `MB_HOLD_REG_LEN` | `6` | 6 个保持寄存器 = DB1 的 12 字节 |

`MB_HOLD_REG` 是指针参数，输入 `P#DB1.DBX0.0 WORD 6`；部分 TIA 版本也可以
直接选择 `DB1` 并填偏移 `DBX0.0`，长度统一由 `MB_HOLD_REG_LEN=6` 决定。
以后给 DB1 补了 `DBD12`，把这两个长度改成 8，再启用标签 2005。

### 2.3 编译下载运行

1. 保存、编译整个项目，确认无错误。
2. 下载到设备（沿用 S7 手册第 5.6 节的方法），把 PLC 切到 `RUN`。
3. 用 TIA 监视表确认测试值还在：`iTest1=1234`、`rTest=3.14`、`bTest` 可切换。

注意：

- `MB_SERVER` 必须放在循环执行的 OB（例如 OB1）里，每个扫描周期都被调用；
  只放在启动 OB 里连接不会保持。
- `MB_HOLD_REG` 指向的 DB1 必须是非优化访问（S7 测试时已经是）。
- S7 的 PUT/GET 开关不用动，两个协议可以同时用。
- 一个 `MB_SERVER` 实例的 `MB_MODE` 决定响应哪一类功能码。本手册用 `0`
  （保持寄存器）。想同时测线圈（FC1），需要另开一个实例或另用一个端口，
  标签 2006 已预留，见第 8 节。

## 3. 寄存器对照表

`MB_SERVER` 把 DB1 按字映射成保持寄存器，`40001` 是第一个字：

| DB1 偏移 | TIA 变量 | Modbus 地址 | 网关标签 |
| --- | --- | --- | --- |
| `DBW0` | `bTest`（Bool，0/1） | `40001` | 2001 `Modbus_Word0`（地址 `40001.0`） |
| `DBW2` | `iTest1`（Int，1234） | `40002` | 2002 `Modbus_Int2` |
| `DBW4` | `iTest2`（Int） | `40003` | 2003 `Modbus_Int4` |
| `DBD8` | `rTest`（Real，3.14） | `40005`-`40006` | 2004 `Modbus_Real8` |
| `DBD12` | `dTest`（现在没有） | `40007`-`40008` | 2005（停用） |
| `M0.0` | `mTest` | 线圈区（可选） | 2006（停用） |

## 4. 先直连探测 PLC（不用启动网关）

布尔标签注意：保持寄存器里的布尔按“寄存器值非 0 即真”处理，位号不参与判断，
所以 `Modbus_Word0` 的地址写 `40001.0` 即可，与 `DB1.DBX0.0` 对应。

PLC 侧下载完成后，先用探测脚本确认 502 真的能读出数据，避免把问题混到网关里：

```powershell
python <REPO_ROOT>\outputs\probes\modbus_probe.py 192.168.1.1 502 3 0 6
```

期望结果：

```text
reg 00001 = 0x0001 = 1        <- bTest 为 1 时
reg 00002 = 0x04d2 = 1234     <- iTest1
reg 00003 = 0x0000 = 0        <- iTest2
reg 00005 = 0x4048 = 16456
reg 00006 = 0xf5c3 = 62915
pair 00005-00006 int=1078523331 float=3.14
```

如果 TCP 连不上或超时：先确认 PLC 是 `RUN`、程序已下载、`MB_SERVER` 在 OB1 里。
如果返回 `exception code 02`：`MB_HOLD_REG` / `MB_HOLD_REG_LEN` 没覆盖到要读的地址。

## 5. 宿主机：切数据库配置并重启网关

1. 停掉正在运行的网关：

   ```powershell
   Get-Process GateWay -ErrorAction SilentlyContinue | Stop-Process -Force
   ```

2. 执行配置脚本（可重复执行）：

   ```powershell
   $mysql = '<REPO_ROOT>\work\mariadb\mariadb-11.4.2-winx64\bin\mysql.exe'
   powershell -File <REPO_ROOT>\outputs\config\apply_config.ps1 -Config modbus
   ```

   脚本做的事：
   - `registermodule` 9 指向 `ModbusDriver.dll / ModbusTCPReader`；
   - `meta_driver` 1 的 `Server` 改为 `192.168.1.1`（`Spare1=1` 是 Modbus 单元号）；
   - 停用旧的模拟器演示组 20001/20002；
   - 新建组 20004 `ModbusTest` 和标签 2001-2004（1000ms 轮询，已开归档）；
   - 不动 S7 驱动 2 和标签 1001-1006。

3. 启动网关（或继续用 `restart_all.ps1` 一键启动；它最后校验的 tag 2 是模拟器专用的，这次忽略即可）：

   ```powershell
   $gw = '<REPO_ROOT>\work\SharpSCADA\SCADA\Program\CoreApp\DataService\GateWay\bin\Debug\net10.0\GateWay.exe'
   Start-Process -FilePath $gw -WorkingDirectory (Split-Path $gw) -WindowStyle Hidden
   ```

## 6. 验证

1. 确认网关已经连上 PLC 的 502：

   ```powershell
   netstat -ano | Select-String ':502|192.168.1.1'
   ```

   应能看到类似 `192.168.1.50:xxxxx 192.168.1.1:502 ESTABLISHED`。

2. 通过网关读实时值：

   ```powershell
   python <REPO_ROOT>\outputs\probes\query_tag.py 2001 1
   python <REPO_ROOT>\outputs\probes\query_tag.py 2002 2
   python <REPO_ROOT>\outputs\probes\query_tag.py 2004 4
   ```

   期望：`2001` 是 `01`，`2002` 原始字节是 `d204`（1234），`2004` 原始字节是
   `c3f54840`（3.14）。注意探测脚本打印的是网络字节序（寄存器 40005 是
   `0x4048`），query_tag 打印的是网关缓存里的小端字节 `c3f54840`，两个都是 3.14。

3. 打开客户端：

   ```powershell
   <REPO_ROOT>\work\SharpSCADA\SCADA\Example\CoreTest.exe
   ```

   在实时标签列表里找 `Modbus_*`。回 TIA 改 `iTest1`/`rTest`/`bTest`，
   客户端 1 秒内应跟着变（S7 标签 1001-1006 同时还在，可对比）。

4. 归档落库：2001-2004 已配 `Archive=1 / Cycle=1000`。快速验证时把
   `C:\DataConfig\server.xml` 和 `<REPO_ROOT>\work\SharpSCADA\SCADA\DataConfig\server.xml`
   的 `WriteCycle`、`Delay` 临时改成 `30000`，重启网关约 30 秒后查：

   ```powershell
   & '<REPO_ROOT>\work\mariadb\mariadb-11.4.2-winx64\bin\mysql.exe' --skip-ssl --host=127.0.0.1 --port=3306 -uroot -proot --database=scada --execute="SELECT ID, TIMESTAMP, VALUE FROM log_hdata WHERE ID IN (2001,2002,2003,2004) ORDER BY TIMESTAMP DESC LIMIT 20;"
   ```

   验证完把两个值调回 `600000` / `3600000` 并重启网关。

## 7. 回退

回 S7：

```powershell
powershell -File <REPO_ROOT>\outputs\config\apply_config.ps1 -Config s7
& $mysql --skip-ssl --host=127.0.0.1 --port=3306 -uroot -proot --database=scada --execute="UPDATE meta_group SET IsActive=b'0' WHERE GroupID=20004;"
```

回模拟器：

```powershell
& $mysql --skip-ssl --host=127.0.0.1 --port=3306 -uroot -proot --database=scada --execute="UPDATE meta_driver SET Server='127.0.0.1' WHERE DriverID=1; UPDATE meta_group SET IsActive=b'0' WHERE GroupID=20004; UPDATE meta_group SET IsActive=b'1' WHERE GroupID=20001;"
```

每次改完数据库必须重启网关，网关只在启动时读取配置。

## 8. 可选：测线圈（FC1，对应原 M0.0）

1. 在 TIA 新建非优化数据块 DB3，至少 1 字节。
2. 在 OB1 里把 `M0.0` 赋值给 `DB3.DBX0.0`。
3. 给 `MB_SERVER` 配置 `MB_MODE=2`、`MB_COILS=P#DB3.DBX0.0 BOOL 8`、
   `MB_COILS_LEN=8`，下载运行。注意改成模式 2 后 FC3 保持寄存器不再响应，
   需要再测寄存器时改回 0；两个模式同时生效需要第二个 `MB_SERVER` 实例和独立端口。
4. 宿主机启用标签 2006（地址 `00000`，协议偏移 0 对应第一个线圈）并重启网关。
   这个驱动的线圈地址是 0 起：第一个线圈用 `00000`，不要用 `00001`。

## 9. 常见问题对照表

| 现象 | 检查 |
| --- | --- |
| ping 通但 502 连不上 | MB_SERVER 是否已下载、PLC 是否 RUN、端口是否 502、MB_SERVER 是否在 OB1 每周期调用 |
| FC3 返回 exception 02 | `MB_HOLD_REG` / `MB_HOLD_REG_LEN` 是否覆盖到要读的寄存器（DB1 用 6） |
| 值全是 0 / 不变 | DB1 是否非优化、TIA 监视表是否有测试值、`MB_HOLD_REG` 是否 `DB1.DBX0.0` |
| 网关还连 127.0.0.1:502 | `meta_driver.Server` 是否 `192.168.1.1`，改完是否重启网关 |
| 客户端没有 `Modbus_*` | 组 20004 和标签 2001-2004 是否 `IsActive=1`，重启网关 |
| S7 与 Modbus 同时用 | 不冲突：S7 走 102，Modbus 走 502，PUT/GET 不用关 |
| 连接正常但读不到值 | 单元号不对时把 `meta_driver.Spare1` 改成 `255` 再重启网关试试 |
| 编译报 CONNECT 类型/语法错 | 确认 `MB_TCON` 的变量类型是 `TCON_IP_v4`；`RemoteAddress` 是 4 字节数组（0,0,0,0），不要填字符串 `"0.0.0.0"` |

## 10. 最小概念

- MBAP：Modbus TCP 的报文头，固定走 TCP 502。
- FC3：读保持寄存器（4xxxx）。
- MB_SERVER：PLC 程序里的 Modbus 服务器指令，不是 PLC 的默认功能。
- 保持寄存器：40001 起，1 个寄存器 = 1 个字（2 字节）；REAL 占 2 个寄存器。
