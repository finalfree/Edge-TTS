# edge-tts Investigation Summary

## Quick Answer

**Yes, this repository reverse-engineered Microsoft's Edge TTS interface.**

## How It Works

1. **WebSocket Connection**: Connects to Microsoft's Bing speech synthesis service via WebSocket
   - Endpoint: `wss://speech.platform.bing.com/consumer/speech/synthesize/readaloud/edge/v1`

2. **Authentication**: Mimics Microsoft Edge browser requests
   - Uses hardcoded `TrustedClientToken`: `6A5AA1D4EAFF4E9FB37E23D68491D6F4`
   - Generates time-based `Sec-MS-GEC` tokens using SHA256 hashing
   - Impersonates Chrome extension origin: `chrome-extension://jdiccldimpdaibmpdkjnbmckianbfold`

3. **Text Processing**: Uses SSML (Speech Synthesis Markup Language) format
   - Supports pitch, rate, and volume adjustments
   - Automatically splits long text into 4096-byte chunks

## Dependencies

Only 4 core dependencies (very lightweight):

```python
"aiohttp>=3.8.0,<4.0.0",           # Async HTTP/WebSocket client
"certifi>=2023.11.17",              # SSL certificate verification
"tabulate>=0.4.4,<1.0.0",          # Table formatting for voice lists
"typing-extensions>=4.1.0,<5.0.0", # Type hints for older Python
```

## Key Technical Details

- **DRM Token Generation** (`src/edge_tts/drm.py`): Time-based authentication with 5-minute granularity
- **Clock Skew Correction**: Automatically syncs with server time to handle authentication failures
- **Browser Simulation**: Complete HTTP headers mimicking Edge 143.0.3650.75
- **Async/Sync APIs**: Both async/await and synchronous interfaces
- **Subtitle Support**: Generates word/sentence boundary timestamps for SRT subtitles

## File Structure

```
src/edge_tts/
├── communicate.py    # Core TTS communication logic
├── drm.py           # DRM token generation and clock sync
├── constants.py     # URLs, headers, trusted tokens
├── voices.py        # Voice list management
└── ...
```

## Example Usage

```python
import asyncio
import edge_tts

async def main():
    communicate = edge_tts.Communicate("Hello, world!", "en-US-EmmaMultilingualNeural")
    await communicate.save("output.mp3", "output.vtt")

asyncio.run(main())
```

## Conclusion

This is an excellent reverse engineering example that successfully:
- ✅ Replicated Edge browser's authentication mechanism
- ✅ Implemented DRM token generation algorithm
- ✅ Supports all Edge TTS voices and parameters
- ✅ Minimal dependencies with high code quality
- ✅ Cross-platform pure Python implementation

For detailed analysis in Chinese, see [TTS实现调查报告.md](./TTS实现调查报告.md).
