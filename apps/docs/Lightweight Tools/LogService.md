---
sidebar_label: Platform Log Service
sidebar_class_name: lightweight tools
---


# Platform Log Service

## 1. 项目概述

Platform Log Service 是一个基于 C# 编写的日志采集与上传服务，适用于 Windows 平台。它通过 **Named Pipe（命名管道）** 接收外部应用日志，并将其上传至 **阿里云日志服务（Aliyun Log Service）** 进行存储和分析。

## 2. 功能特性

- **多管道日志采集**：支持多个 Named Pipe 并行收集日志数据。
- **日志解析与存储**：将日志数据转换为 JSON 并存入队列。
- **异步日志上传**：利用阿里云日志服务 SDK，批量上传日志数据。
- **高并发支持**：使用 `ConcurrentQueue` 处理日志数据，确保高效并发处理。

## 3. 运行环境

- Windows 10/11 或 Windows Server 2016 及以上
- .NET 6.0 及以上
- Visual Studio 2022 及以上
- 阿里云日志服务（Aliyun Log Service）

## 4. 安装与配置

### **4.1 配置阿里云 Access Key**

程序使用 **AES 加密** 存储 `AccessKeyId` 和 `AccessKeySecret`，请在 `LoadCredentials()` 方法中配置加密后的密钥。

若需修改密钥，请使用 `EncryptString()` 方法加密并替换 `encryptedData` 变量的值。

### **4.2 配置 Named Pipe**

修改 `PipeToLogStore` 变量，添加或删除需要监听的 Named Pipe。

```csharp
private static readonly Dictionary<string, string> PipeToLogStore = new()
{
    { "DispatcherLogPipe", "dispatcherlog" },
    { "DispatcherCSVPipe", "dispatchercsv" },
    { "MLScenarioLogPipe", "mlscenariolog" }
};
```

## 5. 启动服务

1. **使用 Visual Studio 运行**：
   - 打开项目，在 `Program.cs` 文件中设置 `Main()` 方法为启动项。
   - 运行 `dotnet run` 启动服务。

2. **编译可执行文件**：
   - 执行 `dotnet publish -c Release -r win-x64 --self-contained false` 生成可执行文件。
   - 运行 `PlatformEnablementLogService.exe` 启动服务。

## 6. Named Pipe 数据格式规范

### **6.1 数据格式**

Named Pipe 传输的数据采用 **键值对格式**，各字段使用 `,` 分隔，每个数据包以 **换行符 `\n` 结尾**。

```plaintext
Key1=Value1,Key2=Value2,Key3=Value3,...,Timestamp=UnixTimeStamp\n
```

### **6.2 传输要求**

1. **数据完整性**：
   - 每条日志数据 **必须** 以 `\n` 结尾。
   - 发送端应 **逐行发送数据**，避免粘包。

2. **字段格式**：
   - 每条数据包 **必须** 包含 `Timestamp` 字段。
   - `Key` 和 `Value` 之间使用 `=` 连接。
   - 使用 `,` 作为字段分隔符。

3. **编码格式**：
   - 统一采用 `UTF-8` 编码。

### **6.3 示例数据**

```plaintext
CpuUsage=25.58,AvailableMemoryMB=0.003,DiskReadWriteBytes=0.000,ThreadCount=9347,Timestamp=1738826595\n
LogLevel=INFO,Message=Dispatcher started successfully,Timestamp=1738826600\n
Model=MLModelX,Accuracy=0.95,Loss=0.02,Timestamp=1738826610\n
```

### **6.4 错误示例**

#### **1. 粘包问题**

```plaintext
Key1=Value1,Key2=Value2,Timestamp=1738826594Key1=Value3,Key2=Value4,Timestamp=1738826595\n
```

*错误原因：缺少换行符，导致多条数据粘连。*

#### **2. 字段缺失**

```plaintext
Key1=Value1,Key2=Value2\n
```

*错误原因：缺少 `Timestamp`，无法排序数据。*

### **6.5 解析建议**

1. **按 `\n` 分割**：保证每条数据单独解析。
2. **按 `,` 分割键值对**。
3. **校验 `Timestamp` 是否存在**，如无则丢弃。
4. **异常数据存储为 `Message` 字段**，避免丢失。

### **6.6 支持的 Named Pipe 列表**

- `DispatcherLogPipe`
- `DispatcherCSVPipe`
- `MLScenarioLogPipe`

## 7. 日志上传机制

1. **日志加入队列**：
   - 解析 Named Pipe 日志后，存入 `ConcurrentQueue`。
2. **批量处理日志**：
   - `ProcessLogQueue` 线程轮询队列，批量上传日志。
3. **上传到阿里云**：
   - 使用 `PostLogStoreLogsAsync()` 上传日志至阿里云。
