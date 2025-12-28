# edge-tts Architecture Diagram

## Overall System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Application                         │
│  (Python script using edge_tts.Communicate)                     │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      edge-tts Library                            │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Communicate  │  │    DRM       │  │   Voices     │         │
│  │   (Main)     │  │ (Auth/Token) │  │  (Manager)   │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
│         │                 │                  │                  │
│         └────────┬────────┴──────────────────┘                  │
│                  │                                               │
│                  ▼                                               │
│         ┌────────────────┐                                      │
│         │  WebSocket     │                                      │
│         │  Connection    │                                      │
│         └────────┬───────┘                                      │
└──────────────────┼──────────────────────────────────────────────┘
                   │
                   │ WSS (WebSocket Secure)
                   │ + Sec-MS-GEC Token
                   │ + TrustedClientToken
                   │ + Browser Headers
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│              Microsoft Edge TTS Service                          │
│   wss://speech.platform.bing.com/consumer/speech/               │
│              synthesize/readaloud/edge/v1                        │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Auth/Verify  │→│ SSML Parser  │→│   TTS Engine │         │
│  └──────────────┘  └──────────────┘  └──────┬───────┘         │
└─────────────────────────────────────────────┼──────────────────┘
                                               │
                    ┌──────────────────────────┴─────────┐
                    │                                    │
                    ▼                                    ▼
            ┌───────────────┐                  ┌────────────────┐
            │ Audio Stream  │                  │   Metadata     │
            │  (MP3 24kHz)  │                  │ (Word/Sentence │
            │               │                  │   Boundaries)  │
            └───────────────┘                  └────────────────┘
```

## Request Flow

```
Client (edge-tts)                    Server (Microsoft)
      │                                      │
      │  1. WebSocket Handshake             │
      │     + Sec-MS-GEC Token              │
      │     + Browser Headers               │
      ├─────────────────────────────────────>│
      │                                      │
      │  2. Connection Established          │
      │<─────────────────────────────────────┤
      │                                      │
      │  3. Send speech.config              │
      │     (Audio format settings)         │
      ├─────────────────────────────────────>│
      │                                      │
      │  4. Send SSML request               │
      │     (Text + Voice params)           │
      ├─────────────────────────────────────>│
      │                                      │
      │  5. Receive audio.metadata          │
      │     (Word boundaries)               │
      │<─────────────────────────────────────┤
      │                                      │
      │  6. Receive audio chunks            │
      │     (Binary MP3 data)               │
      │<─────────────────────────────────────┤
      │<─────────────────────────────────────┤
      │<─────────────────────────────────────┤
      │                                      │
      │  7. Receive turn.end                │
      │<─────────────────────────────────────┤
      │                                      │
      │  (Repeat 4-7 for next text chunk)   │
      │                                      │
```

## Authentication Mechanism

```
┌─────────────────────────────────────────────────────────────┐
│                    Sec-MS-GEC Token Generation              │
└─────────────────────────────────────────────────────────────┘

Current Time (UTC)
      │
      ▼
Convert to Windows File Time
(seconds since 1601-01-01)
      │
      ▼
Round down to nearest 5 minutes
(aligned to 300-second intervals)
      │
      ▼
Convert to 100-nanosecond units
      │
      ▼
Concatenate with TrustedClientToken
(6A5AA1D4EAFF4E9FB37E23D68491D6F4)
      │
      ▼
SHA256 Hash
      │
      ▼
Uppercase Hex Digest
      │
      ▼
Sec-MS-GEC Token
(Valid for ~5 minutes)


Example:
Time: 2025-12-28 04:50:00 UTC
→ Windows ticks: 13393746000000000000
→ String: "13393746000000000006A5AA1D4EAFF4E9FB37E23D68491D6F4"
→ SHA256: "E8F2A3B4C5D6E7F8..." (example)
→ Token: "E8F2A3B4C5D6E7F8..."
```

## Text Processing Pipeline

```
Input Text
      │
      ▼
Remove incompatible characters
(Control chars 0-8, 11-12, 14-31)
      │
      ▼
XML Escape
(&, <, >, ", ')
      │
      ▼
Split by byte length (4096 bytes)
• Prefer newline boundaries
• Preserve UTF-8 integrity
• Avoid splitting XML entities
      │
      ▼
Wrap in SSML
<speak>
  <voice name="...">
    <prosody pitch="..." rate="..." volume="...">
      Text chunk
    </prosody>
  </voice>
</speak>
      │
      ▼
Send to TTS service
```

## Component Details

### Communicate Class
- **Purpose**: Main interface for TTS operations
- **Key Methods**:
  - `stream()`: Async generator for audio/metadata
  - `save()`: Save audio and subtitles to files
  - `stream_sync()`: Synchronous wrapper

### DRM Class
- **Purpose**: Handle authentication and clock sync
- **Key Methods**:
  - `generate_sec_ms_gec()`: Generate auth token
  - `handle_client_response_error()`: Clock skew correction
  - `headers_with_muid()`: Add random MUID cookie

### VoicesManager Class
- **Purpose**: Query and filter available voices
- **Key Methods**:
  - `create()`: Initialize with voice list from API
  - `find()`: Filter voices by attributes

## Data Flow

```
Text Input
    ↓
[Text Splitter] → Chunks (≤4096 bytes)
    ↓
[SSML Builder] → SSML Documents
    ↓
[WebSocket] → Microsoft Server
    ↓
[Audio Decoder] ← MP3 Audio Stream
    ↓
[Metadata Parser] ← Word/Sentence Boundaries
    ↓
Output (Audio + Subtitles)
```
