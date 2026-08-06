# 批量标签导入与日常操作 README

本文档是完成“TIA 变量表导出 ->
批量导入网关 -> 实时读取 -> 归档验证”的说明

文档中所有D:\Codex\2026-08-03\w都需要替换为本机实际地址

## 1. 系统组成

| 部件 | 位置/地址 | 说明 |
| --- | --- | --- |
| PLC | `192.168.1.1` | S7-1500，S7 走 102，Modbus TCP 走 502 |
| 网关 | 宿主机 `192.168.1.50` | `GateWay.exe`，读取数据库配置并轮询 PLC |
| 数据库 | `127.0.0.1:3306` | MariaDB，库 `scada`，账号 `root/root` |
| 客户端 | 宿主机 | `CoreTest.exe`，查看实时标签 |
| TIA 虚拟机 | `192.168.1.60` | 导出 PLC 变量表、修改测试值 |

## 2. 目录说明

```text
outputs\
  config\   配置 SQL、PLC 变量表（PLCTags.xlsx）
  probes\   导入脚本、查询/探测脚本
  docs\     手册与说明文档
```

## 3. 日常启动

MariaDB、模拟器、网关都在宿主机，重启电脑后需要重新启动：

```powershell
powershell -ExecutionPolicy Bypass -File D:\Codex\2026-08-03\w\work\restart_all.ps1
```

只启动网关：

```powershell
$gw = 'D:\Codex\2026-08-03\w\work\SharpSCADA\SCADA\Program\CoreApp\DataService\GateWay\bin\Debug\net10.0\GateWay.exe'
Get-Process GateWay -ErrorAction SilentlyContinue | Stop-Process -Force
Start-Process -FilePath $gw -WorkingDirectory (Split-Path $gw) -WindowStyle Hidden
```

## 4. 批量导入流程

### 4.1 从 TIA 导出变量表

1. 打开 TIA 项目，展开 PLC 设备。
2. 双击“PLC 变量”或打开对应的 DB 变量表。
3. `Ctrl+A` 全选 -> `Ctrl+C` 复制 -> 粘贴到 Excel。
4. 另存为 xlsx，放到 `D:\Codex\2026-08-03\w\outputs\config\`。
5. 只改数值不需要重新导出；只有新增/删除/改名/改地址才需要。

### 4.2 预检（推荐先跑）

```powershell
python D:\Codex\2026-08-03\w\outputs\probes\plc_tag_import.py --mode dry
```

输出里 `imported` 显示将要导入的标签和 TagID，`errors` 显示无法导入的行。

自定义文件或参数示例：

```powershell
python D:\Codex\2026-08-03\w\outputs\probes\plc_tag_import.py --file D:\Codex\2026-08-03\w\outputs\config\NewTags.xlsx --mode dry --start-tagid 3101
```

### 4.3 导入数据库

```powershell
python D:\Codex\2026-08-03\w\outputs\probes\plc_tag_import.py --mode db
```

脚本行为：

- 自动创建组 `20005 PLCBatch`（S7 驱动，1000ms 轮询）。
- 新标签从 TagID 3001 开始自动分配，默认归档 `Archive=1`、`Cycle=1000ms`。
- 重复执行不会产生重复数据，已有同名变量会跳过。
- 支持地址：`M0.0`、`MW2`、`MD4`、`DB1.DBX0.0`、`DB1.DBW2`、`DB1.DBD8` 等。
- 类型映射：Bool=1、Byte=3、Int=4、Word=5、DInt=7、Real=8。

### 4.4 重启网关并验证

```powershell
$gw = 'D:\Codex\2026-08-03\w\work\SharpSCADA\SCADA\Program\CoreApp\DataService\GateWay\bin\Debug\net10.0\GateWay.exe'
Get-Process GateWay -ErrorAction SilentlyContinue | Stop-Process -Force
Start-Process -FilePath $gw -WorkingDirectory (Split-Path $gw) -WindowStyle Hidden
Start-Sleep -Seconds 3
```

读实时值（第二参数是字节数：Bool 用 1，Int 用 2，Real/DInt 用 4）：

```powershell
python D:\Codex\2026-08-03\w\outputs\probes\query_tag.py 3002 1
python D:\Codex\2026-08-03\w\outputs\probes\query_tag.py 3004 2
```

在 TIA 监视表里改值后，网关 1 秒内应读到新值。

## 5. 归档验证

默认归档周期较长（1 小时批量写一次），快速验证需临时改成 30 秒：

```powershell
$files = @('C:\DataConfig\server.xml', 'D:\Codex\2026-08-03\w\work\SharpSCADA\SCADA\DataConfig\server.xml')
foreach ($f in $files) {
  (Get-Content -LiteralPath $f -Raw) -replace 'WriteCycle="600000" Delay="3600000"', 'WriteCycle="30000" Delay="30000"' | Set-Content -LiteralPath $f -Encoding UTF8
}
Get-Process GateWay -ErrorAction SilentlyContinue | Stop-Process -Force
$gw = 'D:\Codex\2026-08-03\w\work\SharpSCADA\SCADA\Program\CoreApp\DataService\GateWay\bin\Debug\net10.0\GateWay.exe'
Start-Process -FilePath $gw -WorkingDirectory (Split-Path $gw) -WindowStyle Hidden
Start-Sleep -Seconds 40

$mysql = 'D:\Codex\2026-08-03\w\work\mariadb\mariadb-11.4.2-winx64\bin\mysql.exe'
& $mysql --skip-ssl --host=127.0.0.1 --port=3306 -uroot -proot --database=scada --execute="SELECT ID, TIMESTAMP, VALUE FROM log_hdata WHERE ID=3004 ORDER BY TIMESTAMP DESC LIMIT 10;"
```

验证完必须恢复默认：

```powershell
$files = @('C:\DataConfig\server.xml', 'D:\Codex\2026-08-03\w\work\SharpSCADA\SCADA\DataConfig\server.xml')
foreach ($f in $files) {
  (Get-Content -LiteralPath $f -Raw) -replace 'WriteCycle="30000" Delay="30000"', 'WriteCycle="600000" Delay="3600000"' | Set-Content -LiteralPath $f -Encoding UTF8
}
Get-Process GateWay -ErrorAction SilentlyContinue | Stop-Process -Force
Start-Process -FilePath $gw -WorkingDirectory (Split-Path $gw) -WindowStyle Hidden
```

## 6. S7 / Modbus 切换

两种协议共用同一台 PLC（`192.168.1.1`），改完配置必须重启网关：

```powershell
Get-Process GateWay -ErrorAction SilentlyContinue | Stop-Process -Force
$mysql = 'D:\Codex\2026-08-03\w\work\mariadb\mariadb-11.4.2-winx64\bin\mysql.exe'

# 切到 Modbus TCP
cmd /c "`"$mysql`" --skip-ssl --host=127.0.0.1 --port=3306 -uroot -proot scada < `"D:\Codex\2026-08-03\w\outputs\config\modbus_config.sql`""

# 切回 S7
cmd /c "`"$mysql`" --skip-ssl --host=127.0.0.1 --port=3306 -uroot -proot scada < `"D:\Codex\2026-08-03\w\outputs\config\s7_config.sql`""
```

## 7. 常见问题

| 现象 | 处理 |
| --- | --- |
| 导入报“unsupported data type” | 变量类型不在支持列表，先手工确认类型 |
| 导入报“invalid S7 address” | 地址格式不符合，例如缺少 M/I/Q 或 DB 前缀 |
| 重复导入 | 脚本自动跳过已有同名标签，不会产生重复 |
| 读到的值不变化 | 检查 PLC 是否 RUN、网关是否已重启 |
| 网关连不上 PLC | `netstat -ano | Select-String ':6543|192.168.1.1'` 查看连接 |

详细原理见 `docs\真实PLC实测链路与迁移说明.md`。
