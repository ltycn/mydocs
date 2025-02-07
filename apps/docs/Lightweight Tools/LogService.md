---
sidebar_label: Platform Log Service
sidebar_class_name: lightweight tools
---

# Platform Log Service

## 1. 项目概述

Platform Log Service 是一个基于 C# 编写的日志采集与上传服务，适用于 Windows 平台。它通过 **Named Pipe（命名管道）** 接收外部应用日志，并将其上传至 **阿里云日志服务（Aliyun Log Service）** 进行存储和分析。该服务支持管道数据流和文件上传两种模式，用户可以根据需求选择不同的上传方式。

## 2. 功能特性

- **多管道日志采集**：支持多个 Named Pipe 并行收集日志数据。
- **日志解析与存储**：将日志数据转换为 JSON 并存入队列。
- **异步日志上传**：利用阿里云日志服务 SDK，批量上传日志数据。
- **高并发支持**：使用 `ConcurrentQueue` 处理日志数据，确保高效并发处理。
- **文件上传模式**：支持上传指定日志文件到阿里云日志服务，可以是 CSV 文件或普通文本文件。
- **实时日志上传模式**：支持实时流式上传日志数据到阿里云，适用于高频、实时日志处理场景。
- **日志上传模式**：支持实时流式上传（`/stream`）和文件上传（`/upload`）。

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

### **4.3 配置日志上传模式**

您可以选择实时日志流上传或文件上传模式。在程序启动时，根据需要传入相应的命令行参数：

- 使用 `/stream` 启动流模式。
- 使用 `/upload /logstore <filepath>` 启动文件上传模式。

```plaintext
LogService.exe /stream          // 启动流模式
LogService.exe /upload /logstore <filepath>  // 文件上传日志文件
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

### **7.1 实时日志上传（Stream Mode）**

#### **概述**

实时日志上传（Stream Mode）是将 Named Pipe 接收到的日志数据即时传输并上传至阿里云日志服务的模式。它适用于需要持续收集并即时上传日志数据的场景，比如应用程序的实时监控、运行时日志分析等。

#### **工作原理**

1. **实时日志采集**：
   - 程序会持续监听指定的 Named Pipe，当有新的日志数据传输到管道时，立即读取并处理。
2. **数据转换与存储**：
   - 接收到的数据会在内存中暂存，通常存入队列中，以确保高效且线程安全的处理。
   - 日志数据会被转化为 JSON 格式，确保能够适应阿里云日志服务的存储需求。
3. **实时上传**：
   - 程序会在收到数据后立即将其上传至阿里云日志服务。上传操作是异步执行的，以防止阻塞数据采集过程。
   - 为了提高性能，程序会批量上传多个日志条目，而不是每次只上传一条数据。这个过程是自动进行的，确保实时性与上传效率的平衡。

#### **使用方式**

实时日志上传模式通过命令行参数启动：

```plaintext
LogService.exe /stream
```

- **`/stream`**：启用实时日志流上传模式。此模式会在程序启动时自动开始从指定的 Named Pipe 中收集日志数据，并立即上传至阿里云日志服务。

#### **优势与适用场景**

- **优势**：
   - **实时性**：一旦日志数据通过 Named Pipe 传输，它就会立刻被上传，适用于需要快速响应的场景。
   - **低延迟**：实时上传模式确保日志数据可以在尽可能短的时间内送达到阿里云日志服务进行分析。
   - **自动化**：无需手动干预，日志数据被持续收集并上传。

- **适用场景**：
   - **实时日志监控**：适用于需要监控应用程序运行时日志的情况，如实时监控和故障检测。
   - **动态数据处理**：实时日志上传可用于快速捕捉并分析应用程序的运行状态。

#### **上传流程**

1. **监听管道**：程序开始时，会监听配置的 Named Pipe 并实时接收日志数据。
2. **数据处理**：每当接收到一条新日志数据，程序会立即对其进行解析并存储。
3. **上传至阿里云**：数据会在内存中处理后，迅速批量上传到阿里云日志服务。
4. **持续运行**：程序在后台持续运行，保持对日志流的监听并不断上传新数据。

### **7.2 文件上传模式（Upload Mode）**

#### **概述**

文件上传模式是指将一个指定的日志文件（例如 CSV 或文本文件）上传至阿里云日志服务。这一模式适用于批量上传日志文件，常见于需要离线日志文件上传或迁移历史日志数据的情况。

#### **工作原理**

1. **指定 LogStore**：用户通过命令行参数指定上传的 LogStore。例如，`LogService.exe /upload /dispatcherlog <filepath>`。
2. **日志文件解析**：
   - 根据文件格式（如 CSV 或文本格式）解析文件内容。
   - CSV 格式日志会按照列进行处理，确保字段被正确提取，并转换为适合阿里云上传的格式。
   - 文本格式日志会逐行处理，每行日志以 `Timestamp` 字段为关键字段进行解析和上传。
3. **日志上传**：解析后的日志会被批量上传至阿里云日志服务对应的 LogStore。上传过程中，程序会异步处理文件，确保大文件的高效上传。

#### **使用方式**

文件上传模式通过命令行参数启动：

```plaintext
LogService.exe /upload /logstore <filepath>
```

- **`/upload`**：启动文件上传模式。
- **`/logstore`**：指定日志数据将上传至的目标 LogStore。用户需要确保该 LogStore 在阿里云日志服务中已经存在。
- **`<filepath>`**：指定需要上传的日志文件路径。支持的文件格式包括 CSV 格式和普通文本格式。

例如，上传一个 CSV 格式的日志文件至 `dispatcherlog` LogStore：

```plaintext
LogService.exe /upload /dispatcherlog C:\logs\dispatcher.csv
```
