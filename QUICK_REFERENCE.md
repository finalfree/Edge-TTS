# edge-tts 快速参考 / Quick Reference Guide

## 一句话回答 / One-Line Answer

**是的，edge-tts 通过逆向工程微软 Edge 浏览器的 TTS 接口实现了文本转语音功能。**

**Yes, edge-tts implements TTS by reverse-engineering Microsoft Edge browser's TTS interface.**

---

## 核心技术 / Core Technologies

| 技术 / Technology | 说明 / Description |
|-------------------|-------------------|
| **WebSocket** | 与微软服务器实时通信 / Real-time communication with Microsoft servers |
| **SSML** | 语音合成标记语言 / Speech Synthesis Markup Language |
| **DRM Token** | 基于时间的认证令牌 / Time-based authentication token |
| **aiohttp** | 异步 HTTP/WebSocket 客户端 / Async HTTP/WebSocket client |

---

## 依赖项 / Dependencies

```bash
pip install edge-tts
```

仅需 4 个依赖 / Only 4 dependencies:
- `aiohttp` - WebSocket 通信
- `certifi` - SSL 证书
- `tabulate` - 表格显示
- `typing-extensions` - 类型提示

---

## 快速开始 / Quick Start

### Python 代码 / Python Code

```python
import asyncio
import edge_tts

async def text_to_speech():
    communicate = edge_tts.Communicate(
        text="Hello, world!",
        voice="en-US-EmmaMultilingualNeural"
    )
    await communicate.save("output.mp3")

asyncio.run(text_to_speech())
```

### 命令行 / Command Line

```bash
# 基础使用 / Basic usage
edge-tts --text "Hello" --write-media hello.mp3

# 中文语音 / Chinese voice
edge-tts --voice zh-CN-XiaoxiaoNeural --text "你好" --write-media hello_cn.mp3

# 列出语音 / List voices
edge-tts --list-voices

# 调整参数 / Adjust parameters
edge-tts --rate=+50% --pitch=+10Hz --volume=-20% --text "Modified speech" --write-media modified.mp3
```

---

## 关键文件 / Key Files

| 文件 / File | 功能 / Function |
|-------------|-----------------|
| `communicate.py` | 核心 TTS 通信逻辑 / Core TTS communication |
| `drm.py` | DRM 令牌生成和时钟同步 / DRM token generation & clock sync |
| `constants.py` | URL、头部、令牌常量 / URLs, headers, token constants |
| `voices.py` | 语音列表管理 / Voice list management |

---

## 认证机制 / Authentication

```
当前时间 → Windows 文件时间 → 对齐到5分钟 → + TrustedClientToken → SHA256 → Sec-MS-GEC

Current Time → Windows File Time → Round to 5min → + TrustedClientToken → SHA256 → Sec-MS-GEC
```

**TrustedClientToken**: `6A5AA1D4EAFF4E9FB37E23D68491D6F4`

---

## 支持的语音参数 / Supported Voice Parameters

| 参数 / Parameter | 范围 / Range | 示例 / Example |
|------------------|--------------|----------------|
| **rate** (语速) | -100% 到 +200% | `+50%`, `-20%` |
| **volume** (音量) | -100% 到 +100% | `+0%`, `-50%` |
| **pitch** (音调) | -50Hz 到 +50Hz | `+10Hz`, `-5Hz` |

---

## 常用语音 / Popular Voices

### 英语 / English
- `en-US-EmmaMultilingualNeural` (女声, 多语言)
- `en-US-AndrewMultilingualNeural` (男声, 多语言)
- `en-GB-SoniaNeural` (英式女声)

### 中文 / Chinese
- `zh-CN-XiaoxiaoNeural` (女声, 标准普通话)
- `zh-CN-YunxiNeural` (男声, 标准普通话)
- `zh-CN-XiaoyiNeural` (女声, 儿童音)

### 其他 / Others
- `ja-JP-NanamiNeural` (日语女声)
- `ko-KR-SunHiNeural` (韩语女声)
- `fr-FR-DeniseNeural` (法语女声)
- `de-DE-KatjaNeural` (德语女声)
- `es-ES-ElviraNeural` (西班牙语女声)

---

## 输出格式 / Output Formats

| 格式 / Format | 说明 / Description |
|---------------|-------------------|
| **MP3** | 音频文件 (24kHz, 48kbps) / Audio file |
| **VTT/SRT** | 字幕文件 / Subtitle file |
| **JSON** | 元数据 (字边界) / Metadata (word boundaries) |

---

## 错误处理 / Error Handling

| 错误 / Error | 原因 / Cause | 解决方案 / Solution |
|--------------|--------------|---------------------|
| `403 Forbidden` | 时钟偏差 / Clock skew | 自动校正 / Auto-corrected |
| `NoAudioReceived` | 参数错误 / Invalid parameters | 检查语音名称 / Check voice name |
| `WebSocketError` | 网络问题 / Network issue | 检查连接 / Check connection |

---

## API 对比 / API Comparison

### 异步 API / Async API (推荐 / Recommended)

```python
async for chunk in communicate.stream():
    if chunk["type"] == "audio":
        # 处理音频 / Process audio
        audio_data = chunk["data"]
    elif chunk["type"] == "WordBoundary":
        # 处理字边界 / Process word boundary
        offset = chunk["offset"]
```

### 同步 API / Sync API

```python
for chunk in communicate.stream_sync():
    # 同样的处理逻辑 / Same processing logic
    pass
```

---

## 性能提示 / Performance Tips

1. **复用连接器** / Reuse Connector
   ```python
   connector = aiohttp.TCPConnector(limit=10)
   communicate = edge_tts.Communicate(text, voice, connector=connector)
   ```

2. **批量处理** / Batch Processing
   - 长文本自动分块 (4096 字节)
   - Automatically split into chunks (4096 bytes)

3. **异步并发** / Async Concurrency
   - 同时处理多个 TTS 请求
   - Process multiple TTS requests simultaneously

---

## 限制 / Limitations

- ❌ 不支持自定义 SSML / No custom SSML support
- ❌ 音频格式固定为 MP3 24kHz / Fixed MP3 24kHz format
- ❌ 依赖微软服务可用性 / Depends on Microsoft service availability
- ✅ 完全免费 / Completely free
- ✅ 无需 API 密钥 / No API key required

---

## 文档链接 / Documentation Links

- 📖 [完整中文报告 / Full Chinese Report](./TTS实现调查报告.md)
- 📖 [English Summary](./INVESTIGATION_SUMMARY.md)
- 📖 [Architecture Diagrams](./ARCHITECTURE.md)
- 📖 [文档索引 / Documentation Index](./调查文档索引.md)

---

## 联系方式 / Contact

- GitHub: https://github.com/rany2/edge-tts
- Issues: https://github.com/rany2/edge-tts/issues
- PyPI: https://pypi.org/project/edge-tts/

---

**最后更新 / Last Updated**: 2025-12-28
