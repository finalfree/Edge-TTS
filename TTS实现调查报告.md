# edge-tts TTS实现调查报告

## 概述

`edge-tts` 是一个Python模块，允许用户在Python代码中使用微软Edge的在线文本转语音（TTS）服务，而无需安装Windows系统或Edge浏览器。

## 一、TTS实现原理

### 1.1 核心机制

该项目**确实是逆向工程了微软的接口**。具体实现方式如下：

1. **WebSocket连接**：通过WebSocket协议连接到微软Bing的语音合成服务
   - 服务端点：`wss://speech.platform.bing.com/consumer/speech/synthesize/readaloud/edge/v1`

2. **SSML格式**：使用SSML（Speech Synthesis Markup Language）XML格式发送文本
   - 支持音调（pitch）、语速（rate）、音量（volume）调整
   - 通过`<voice>`和`<prosody>`标签控制语音参数

3. **认证机制**：模拟Microsoft Edge浏览器的请求
   - 使用固定的`TrustedClientToken`：`6A5AA1D4EAFF4E9FB37E23D68491D6F4`
   - 实现DRM保护机制，生成`Sec-MS-GEC`令牌
   - 使用时间戳和哈希算法验证请求合法性

### 1.2 关键技术细节

**DRM令牌生成**（`src/edge_tts/drm.py`）：
- 获取当前时间戳并转换为Windows文件时间格式（从1601-01-01开始）
- 将时间四舍五入到最近的5分钟
- 与TrustedClientToken组合后进行SHA256哈希
- 支持时钟偏差校正，确保与服务器时间同步

**浏览器模拟**（`src/edge_tts/constants.py`）：
- User-Agent模拟Chromium/Edge浏览器（当前版本：143.0.3650.75）
- 设置Origin为Chrome扩展：`chrome-extension://jdiccldimpdaibmpdkjnbmckianbfold`
- 包含完整的HTTP头信息以通过服务器验证

**数据流处理**：
- 支持异步（async/await）和同步两种API
- 自动将长文本分割为4096字节的块
- 处理UTF-8编码和XML实体，避免字符分割问题
- 实时流式传输音频和元数据（字边界/句边界）

## 二、项目依赖

### 2.1 核心依赖（来自`setup.py`）

```python
install_requires=[
    "aiohttp>=3.8.0,<4.0.0",      # 异步HTTP客户端，用于WebSocket连接
    "certifi>=2023.11.17",         # SSL证书验证
    "tabulate>=0.4.4,<1.0.0",     # 表格格式化（用于列出语音列表）
    "typing-extensions>=4.1.0,<5.0.0",  # 类型提示扩展
]
```

### 2.2 依赖说明

1. **aiohttp**：
   - 提供异步HTTP/WebSocket客户端功能
   - 用于建立与微软服务器的WebSocket连接
   - 处理音频数据的实时流式传输

2. **certifi**：
   - 提供Mozilla的CA证书包
   - 确保HTTPS/WSS连接的安全性

3. **tabulate**：
   - 用于美化命令行输出
   - 格式化显示可用语音列表

4. **typing-extensions**：
   - 为旧版本Python提供现代类型提示支持
   - 使用`Literal`等高级类型

### 2.3 开发依赖

```python
dev = [
    "black",        # 代码格式化
    "isort",        # import排序
    "mypy",         # 静态类型检查
    "pylint",       # 代码质量检查
    "types-tabulate"  # tabulate的类型存根
]
```

## 三、核心文件结构

```
src/edge_tts/
├── __init__.py           # 包入口，导出主要API
├── __main__.py           # 命令行入口
├── communicate.py        # 核心通信类，实现TTS逻辑
├── drm.py               # DRM令牌生成和时钟同步
├── constants.py         # 常量定义（URLs, Headers, Tokens）
├── voices.py            # 语音列表管理
├── data_classes.py      # 数据类定义
├── exceptions.py        # 自定义异常
├── submaker.py          # 字幕生成器
├── srt_composer.py      # SRT字幕组合器
├── typing.py            # 类型定义
├── util.py              # 实用工具函数
└── version.py           # 版本信息
```

## 四、工作流程

### 4.1 完整的TTS请求流程

1. **初始化**：创建`Communicate`对象，传入文本和语音参数
2. **文本处理**：
   - 移除不兼容字符（如垂直制表符）
   - XML转义特殊字符
   - 按4096字节分割长文本
3. **建立连接**：
   - 创建SSL上下文
   - 生成连接ID（UUID）
   - 生成Sec-MS-GEC令牌
   - 建立WebSocket连接
4. **发送配置**：发送`speech.config`消息配置音频格式（24kHz 48kbps MP3）
5. **发送SSML**：发送包含文本的SSML请求
6. **接收数据**：
   - 接收二进制音频数据（audio/mpeg）
   - 接收元数据（字边界/句边界信息）
   - 接收会话结束信号（turn.end）
7. **循环处理**：对每个文本块重复步骤5-6
8. **输出结果**：保存音频文件和字幕文件

### 4.2 示例代码

```python
import asyncio
import edge_tts

async def main():
    text = "你好，世界！"
    voice = "zh-CN-XiaoxiaoNeural"
    
    communicate = edge_tts.Communicate(text, voice)
    await communicate.save("output.mp3", "output.vtt")

asyncio.run(main())
```

## 五、逆向工程细节

### 5.1 发现的关键信息

1. **TrustedClientToken**：`6A5AA1D4EAFF4E9FB37E23D68491D6F4`
   - 硬编码在Edge浏览器中的固定令牌
   - 用于标识合法的客户端

2. **Sec-MS-GEC机制**：
   - 基于时间的动态令牌生成
   - 每5分钟更新一次
   - 通过SHA256哈希确保安全性

3. **Origin伪装**：
   - 模拟来自Edge浏览器的"Read Aloud"扩展
   - Chrome扩展ID：`jdiccldimpdaibmpdkjnbmckianbfold`

4. **时钟同步**：
   - 通过服务器返回的`Date`头自动校正时钟偏差
   - 解决系统时间不准确导致的认证失败

### 5.2 限制和约束

1. **SSML限制**：
   - 微软服务器只接受Edge浏览器能生成的SSML格式
   - 只允许单个`<voice>`标签包含单个`<prosody>`标签
   - 不支持自定义SSML（已移除该功能）

2. **分块限制**：
   - 每个SSML请求最大4096字节
   - 需要智能分割避免截断UTF-8字符和XML实体

## 六、技术亮点

1. **异步设计**：充分利用Python的async/await特性，支持高并发
2. **容错机制**：
   - 时钟偏差自动校正
   - 403错误自动重试
   - 完整的异常处理
3. **字幕支持**：提供字边界和句边界时间戳，可生成SRT字幕
4. **跨平台**：纯Python实现，支持Windows、Linux、macOS
5. **命令行工具**：提供`edge-tts`和`edge-playback`命令

## 七、结论

**edge-tts项目是通过逆向工程微软Edge浏览器的TTS功能实现的**：

- ✅ 成功复制了Edge浏览器的认证机制
- ✅ 模拟了完整的浏览器请求头和行为
- ✅ 实现了DRM令牌生成算法
- ✅ 支持所有Edge TTS提供的语音和参数调整
- ✅ 依赖项少且轻量（仅4个核心依赖）
- ✅ 代码质量高，有完整的类型提示和文档

该项目是一个优秀的逆向工程案例，展示了如何通过分析网络流量和浏览器行为来复制云服务的功能。
