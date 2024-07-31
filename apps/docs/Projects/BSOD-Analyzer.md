---
sidebar_label: BSOD Analyzer WebApp
sidebar_class_name: Projects
---

## 项目背景

为方便PA针对BSOD现象快速定位问题根源，节省PA/SA Team对接的工作量，同时统一归纳测试期间产生的BSOD Dump文件，BSOD Analyzer WebApp可以自动化执行上传、存储、分析、AI总结、E-Mail提醒等功能。可以大幅缩短PA/SA Team的Debug所用时间。

## 功能

- Dump上传

填写上传人ITCode和简单的描述，再选择要上传的Dump文件，点击Submit即可上传

- 存储

Dump上传后自动按ITCode和上传时间进行分类归档存储在Server端。可以指定存储路径

- 分析

完成Dump的传输过程后，后台自动进行分析。并将对Dump的分析结果生成txt文件，保存在Dump同路径的文件夹中

- AI提取

后端程序会将上一步分析的txt结果提交AI进行分析，并生成一份html格式的报告（目前采用的是ChatGPT，后续随时可更换本地模型）

- E-Mail提醒

后端程序会将上一步产生的html格式的分析报告作为邮件正文，发送到提交人ITCode的邮箱中。

在完成所有步骤后，可以在主页面的提交记录详情中查看分析过程中产生的所有文件

## 技术栈

- 前端：React
- 后端：Nodejs
- 数据库：MongoDB

## 安装

```bash
# 拉取项目文件
git clone https://github.com/ltycn/LNVPE-BSOD.git

# 启动后端程序
cd server
node server.js

# 启动前端
npm start
```

## RESTAPI使用

### 概述

该 API 用于向服务器提交一个包含提交者信息、描述和文件的表单。

- **API URL:** `http://localhost:3000/api/submissions`
- **方法:** POST
- **请求格式:** `multipart/form-data`

### 请求参数

| 参数名      | 类型           | 描述                       | 是否必需 |
|-------------|----------------|----------------------------|---------|
| submitter   | string         | ITCode                    | 是      |
| description | string         | 提交内容的描述             | 是      |
| file        | file (binary)  | 需要上传的文件             | 是      |

### 请求示例

使用 Postman 或类似工具发送请求时，可以按照以下方式构建请求：

- **URL:** `http://localhost:3000/api/submissions`
- **方法:** `POST`
- **请求头:** `Content-Type: multipart/form-data`
- **请求体:**

```json
{
  "submitter": "liuty24",
  "description": "testpostman",
  "file": "<file>"
}
```

```bash
curl -X POST http://localhost:3000/api/submissions \
  -F "submitter=liuty24" \
  -F "description=testpostman" \
  -F "file=@/path/to/your/file"
```

### 响应格式

#### 成功响应

成功的请求将返回一个 JSON 响应，包含提交的详细信息。

```json
{
  "submitter": "liuty24",
  "description": "testpostman",
  "file": "uploads\\liuty24\\240731165850\\MEMORY.DMP",
  "timestamp": "2024-07-31T08:58:50.841Z",
  "_id": "66a9fccabbf27a95144f1730",
  "__v": 0
}
```
#### 错误响应

如果请求失败，服务器将返回一个 JSON 错误响应。

```json
{
  "error": "Submission failed"
}
````


## 截图

![](.\BSOD-Analyzer\1.png)

![](.\BSOD-Analyzer\2.png)

![](.\BSOD-Analyzer\3.png)

![](.\BSOD-Analyzer\4.png)


