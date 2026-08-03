# 今日工作总结：模拟器实时数据 → 客户端实时刷新

## 一、已实现功能
- 核心环境：Windows + .NET 10 + MariaDB 11.4 + SharpSCADA CoreApp（原版已由 netcoreapp2.0 迁移到 net10.0）。
- Modbus 模拟器 `modbus_sim.py` 监听 `127.0.0.1:502`，支持 FC1/FC2/FC3/FC4，数据每秒变化。
- 网关每秒轮询模拟器，检测到变化后通过数据变化事件推送给已连接的 CoreTest 客户端。
- CoreTest 客户端 tag2 实时刷新（0/1 交替）

## 二、怎么启动项目

1. 一键启动 MariaDB、模拟器、网关：

   ```powershell
   powershell -ExecutionPolicy Bypass -File D:\Codex\2026-08-03\w\work\restart_all.ps1
   ```

2. 打开客户端：

   ```powershell
   D:\Codex\2026-08-03\w\work\SharpSCADA\SCADA\Example\CoreTest.exe
   ```

3. 验证实时数据：
   - 客户端观察 tag2 每秒变化；
   - 命令行连续查两次，结果应不同：
     ```powershell
     python D:\Codex\2026-08-03\w\outputs\query_tag.py 2 1
     ```

## 三、对原版项目的改动

1. `modbus_sim.py`（新增）：监听 127.0.0.1:502，按每秒 tick 变化输出 FC1/FC2 线圈位和 FC3/FC4 寄存器数值。
2. `ModbusTCPReader.cs`：`ReadBytes` 中 `area < 2` 改为 `area < 3`，使 FC1/FC2/FC3 三个区域都能被网关读取。
3. `PLCGroup.cs`（关键修复）：两处 `deleg.BeginInvoke(...)` 改为 `ThreadPool.QueueUserWorkItem(...)`。原因是项目运行在 .NET 10，而 .NET Core 不支持 `Delegate.BeginInvoke`（会抛 `PlatformNotSupportedException`），导致数据变化事件从不触发、客户端收不到实时推送。
4. `DAService.cs`：连接接受/接收线程从 ThreadPool 改为独立后台线程，避免阻塞线程池；并加入临时调试日志（ACCEPT/DATAEVENT/DATACHANGE/SEND）。
5. `restart_all.ps1`（新增/修复）：一键启动 MariaDB、模拟器、网关；修复了空参数启动网关时 PowerShell 5.1 报 ArgumentList 校验错误的问题。

## 四、其他说明
- `gw_debug.log` / `gw_dataevent.log` 是排查用的临时日志，后续可以清理。
