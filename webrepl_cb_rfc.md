# WebREPL Binary Protocol (WBP) Specification

**Protocol Name:** WebREPL Binary Protocol (WBP)  
**Subprotocol Identifier:** `WebREPL.binary.v1`  
**Status:** Draft  
**Version:** 1.0  
**Date:** December 2025  
**Author:** Jonathan E. Peace

---

## Abstract

This document specifies a multiplexing protocol for WebREPL communication over WebSockets, designed for web-based IDEs and machine-to-machine control of MicroPython devices. The protocol uses CBOR (Concise Binary Object Representation) with positional arrays for maximum compactness while supporting binary bytecode execution, structured file operations, and event streaming over a single WebSocket connection.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Design Goals](#2-design-goals)
3. [Protocol Overview](#3-protocol-overview)
4. [Event Messages (Channel 0)](#4-event-messages-channel-0)
5. [Execution Channels (Channels 1-22)](#5-execution-channels-channels-1-22)
6. [File Operations (Channel 23)](#6-file-operations-channel-23)
7. [Connection Negotiation](#7-connection-negotiation)
8. [Protocol Flows](#8-protocol-flows)
9. [Wire Format Examples](#9-wire-format-examples)
10. [Implementation Notes](#10-implementation-notes)
11. [Security Considerations](#11-security-considerations)
12. [Migration from Legacy Protocol](#12-migration-from-legacy-protocol)

---

## 1. Introduction

### 1.1 Background

The legacy WebREPL protocol multiplexes different message types using:
- WebSocket Text frames for terminal I/O
- Binary frames with "WA"/"WB" signatures for file transfers

WebREPL is typically used with web browser clients, which can support a more advanced user interface than simple terminal emulation.
The standard WebREPL protocol provides reliable REPL access and file transfer through text-based streams. However, browser-based clients often need additional capabilities — such as real-time progress information, separate status updates, or structured data exchange — that are difficult to implement efficiently within a single text stream.
The WebREPL Binary Protocol (WBP) introduces a lightweight out-of-band channel for these cases. It allows the client and server to exchange binary or structured data alongside the main REPL stream, without interfering with normal operation. Examples include reporting file transfer progress, sending custom notifications, or using MicroPython functions as a structured API.


WBP adds several features to address these legacy protocol issues:
1. **Channelized**: Messages are multiplexed across 255 independent channels
2. **Binary**: Uses WebSocket Binary mode (0x02 see RFC6455) with CBOR encoding instead of text frames and magic bytes
3. **TFTP-based File Transfers**: File operations use proven TFTP semantics (RFC 1350, 2347, 2348, 2349) with block-based transfers, ACKs, and error handling for reliable, resumable file operations
4. **Compatible**: Uses WebSocket subprotocol negotiation to enable fallback to legacy protocol


### 1.2 Protocol Name Options

This document proposes **WebREPL Binary Protocol** or **WBP**.

For this specification, we use **WBP** and the subprotocol identifier `WebREPL.binary.v1`.

---

## 2. Design Goals

1. **Compactness**: Minimize overhead for high-frequency messages using positional arrays
2. **Binary Support**: Native support for `.mpy` bytecode and binary file data
3. **Extensibility**: Allow optional trailing fields for future features
4. **Self-Describing**: Use CBOR's type system for clear message structure
5. **Unified Protocol**: Single message format for channels, files, and events
6. **Streaming**: Real-time output during code execution
7. **Simplicity**: No schema compilation; standard CBOR libraries work out-of-the-box

---

## 3. Protocol Overview

### 3.1 Transport

All communication uses **WebSocket Binary Frames** (RFC 6455 opcode 0x2). 
By definition, this means frames are delivered reliably, in-order, and without duplicates.
Text frames are never used, which allows the protocol to coexist with text based websocket protocols (such as DAP) on the same port 


### 3.2 Encoding

All messages are encoded as **CBOR arrays** (RFC 8949).

```
WebSocket Binary Frame:
┌─────────────────────┐
│   CBOR Array        │
│   [channel, ...]    │
└─────────────────────┘
```
No magic bytes or additional framing needed. 

### 3.2 Message Format

All messages are CBOR arrays with a channel number at position 0:

```
[channel, field1, field2, ..., ?optional_fields]
```

### 3.4 Message Types

| Type | Channel | Usage |
|------|-------|---------|
| **Event** | `0` | [System events and structured logs](#4-event-messages-channel-0)|
| **Execution** | `1-22` | [Command<->Response](#5-execution-channels-channels-1-22) |
| **File Operations** | `23` | [File Upload, Download](#6-file-operations-channel-23) |

Required fields occupy fixed positions. Optional fields are appended at the end and may be omitted.


### 3.5 CBOR Data Types

| Type | Usage | Example |
|------|-------|---------|
| **Integer** | Channel IDs, opcodes, enums | `0`, `1`, `255` |
| **Text String** | Source code, paths, errors | `"print('hello')\n"` |
| **Byte String** | Binary bytecode, file data | `h'4D0306001F'` |
| **Array** | Completions, file listings | `["sys.path", "sys.platform"]` |
| **Map** | Structured metadata | `{size: 4096, mtime: 1733279222}` |
| **Null** | Optional field omitted | `null` |


### 3.6 Message Discrimination

Messages are discriminated by the **first array element** (channel number):

**Channel Allocation Details:**

| Channel | Name | Purpose | Description |
|---------|------|---------|-------------|
| `0` | EVENT | System events and structured logs | Authentication, notifications, system broadcasts, structured logs |
| `1` | TRM | Terminal REPL | Terminal I/O for HMI |
| `2` | M2M | Machine-to-Machine RPC | M2M requests|
| `3` | DBG | Debug output | Debugger channel |
| `4-22` | Reserved | Future standard channels |
| `23` | FILE | File operations (TFTP-based) | File upload, download (TFTP-based) |
| `24-254` | Custom | Application-Defined channels |

**Why channel 23 for files?** 23 the largest number number that can be encoded as a single byte using CBOR.

---

## 4. Event Messages (Channel 0)

### 4.1 Overview

Event messages handle authentication, notifications, unsolicited broadcasts, and structured logging.

**Format:** `[0, event, ...fields]`

Channel 0 is reserved exclusively for system events and logs.

### 4.2 Event Opcodes

| Event | Name | Direction | Description |
|-------|------|-----------|-------------|
| `0` | AUTH | C→S | Authentication request |
| `1` | AUTH_OK | S→C | Authentication success |
| `2` | AUTH_FAIL | S→C | Authentication failure |
| `3` | INFO | S→C | Informational message (CBOR map) |
| `4` | LOG | S→C | Structured log message |

**Note:** Event opcodes 0-23 encode as single bytes in CBOR (`00`-`17`).

### 4.3 Event Message Schemas

#### 4.3.1 AUTH (Authentication)

```Client→Server
[0, 0, password, ?username]
```

| Position | Field | Type | Description | Required |
|----------|-------|------|-------------|----------|
| 2 | Password | str | Authentication password | ✅ |
| 3 | Username | str | Username (future multi-user) | ❌ |

#### 4.3.2 AUTH_OK (Auth Success)

```Server→Client
[0, 1, ?token, ?expires]
```

| Position | Field | Type | Description | Required |
|----------|-------|------|-------------|----------|
| 2 | Token | str | Session token | ❌ |
| 3 | Expires | int | Token expiration (seconds) | ❌ |

#### 4.3.3 AUTH_FAIL (Auth Failure)

```Server→Client
[0, 2, error]
```

#### 4.3.4 INFO (Informational Message)

```Server→Client
[0, 3, payload]
```

| Position | Field | Type | Description | Required |
|----------|-------|------|-------------|----------|
| 2 | Payload | map | CBOR map with application-specific fields | ✅ |

**Note:** INFO messages carry application-specific data as CBOR maps. Common use cases include welcome messages, device status updates, and other informational broadcasts. The map structure is application-defined.

**Examples:**

```
// Welcome message
[0, 3, {"welcome": "Welcome to MicroPython WebREPL!"}]

// Device status (application-specific)
[0, 3, {"heap": 123456, "uptime": 3600, "rssi": -65}]
```

#### 4.3.5 LOG (Structured Log)

```Server→Client
[0, 5, level, message, ?timestamp, ?source]
```

| Position | Field | Type | Description | Required |
|----------|-------|------|-------------|----------|
| 2 | Level | int | Log level (0=debug, 1=info, 2=warn, 3=error) | ✅ |
| 3 | Message | str | Log message | ✅ |
| 4 | Timestamp | int | Unix timestamp | ❌ |
| 5 | Source | str | Source module/file | ❌ |

**Example:**

```
[0, 5, 2, "Network connection lost", 1733279222, "network.py"]
```

---

## 5. Execution Channels (Channels 1-22)

### 5.1 Overview

Execution channels multiplex different execution contexts (terminal, M2M, debug) over a single WebSocket.

**Format:** `[channel, opcode, ...fields]`

### 5.2 Standard Channels

| Channel | Name | Purpose |
|---------|------|---------|
| `1` | TRM | Terminal REPL |
| `2` | M2M | Machine-to-Machine RPC |
| `3` | DBG | Debug output |
| `4-22` | Reserved | Future standard channels |

### 5.3 Execution Opcodes

#### Client → Server

| Opcode | Name | Description |
|--------|------|-------------|
| `0` | EXE | Execute code or command |
| `1` | INT | Interrupt (Ctrl-C) |
| `2` | RST | Reset device |

#### Server → Client

| Opcode | Name | Description |
|--------|------|-------------|
| `0` | RES | Result data (stdout, return value) |
| `1` | CON | Continuation prompt (need more input) |
| `2` | PRO | Progress/Status (ready, error) |
| `3` | COM | Completions (tab completion results) |

**Note:** Opcodes 0-23 encode as single bytes in CBOR (`00`-`17`), providing room for future expansion while maintaining compactness.

### 5.4 Message Schemas

#### 5.4.1 EXE (Execute)

```Client→Server
[channel, 0, data, ?format, ?id]
```

| Position | Field | Type | Description | Required |
|----------|-------|------|-------------|----------|
| 0 | Channel | int | Channel ID (1-22) | ✅ |
| 1 | Opcode | int | Always `0` (EXE) | ✅ |
| 2 | Data | str/bytes | Code to execute | ✅ |
| 3 | Format | int | `0`=Python source, `1`=.mpy bytecode | ❌ |
| 4 | ID | str | Message tracking ID | ❌ |

**Examples:**

```
// Execute Python source on terminal
[1, 0, "print('hello')\n"]

// Execute .mpy bytecode
[1, 0, h'4D0306001F...', 1]

// Execute with message ID on M2M channel
[2, 0, "get_info()\n", 0, "req-123"]

// Tab completion (ends with \t)
[1, 0, "sys.p\t"]
```

#### 5.4.2 RES (Result)

```Server→Client
[channel, 0, data, ?id]
```

| Position | Field | Type | Description | Required |
|----------|-------|------|-------------|----------|
| 0 | Channel | int | Channel ID | ✅ |
| 1 | Opcode | int | Always `0` (RES) | ✅ |
| 2 | Data | str/bytes | Output data | ✅ |
| 3 | ID | str | Message tracking ID | ❌ |

**Examples:**

```
// Terminal output
[1, 0, "hello\n"]

// M2M response (JSON data)
[2, 0, "{\"heap\":123456,\"uptime\":3600}"]

// With message ID
[2, 0, "{\"result\":\"ok\"}", "req-123"]
```

#### 5.4.3 CON (Continuation)

```Server→Client
[channel, 1]
```

| Position | Field | Type | Description | Required |
|----------|-------|------|-------------|----------|
| 0 | Channel | int | Channel ID | ✅ |
| 1 | Opcode | int | Always `1` (CON) | ✅ |

Sent when the REPL needs more input to complete a statement (e.g., after typing `if True:`).

**Examples:**

```
// Need more input (continuation prompt "...")
[1, 1]
```

#### 5.4.4 COM (Completions)

```Server→Client
[channel, 3, completions]
```

| Position | Field | Type | Description | Required |
|----------|-------|------|-------------|----------|
| 0 | Channel | int | Channel ID | ✅ |
| 1 | Opcode | int | Always `3` (COM) | ✅ |
| 2 | Completions | array | Tab completion suggestions | ✅ |

**Examples:**

```
// Tab completions for "sys.p"
[1, 3, ["sys.path", "sys.platform", "sys.print_exception"]]

// No completions found
[1, 3, []]
```

#### 5.4.5 PRO (Progress)

```Server→Client
[channel, 2, status, ?error, ?id]
```

| Position | Field | Type | Description | Required |
|----------|-------|------|-------------|----------|
| 0 | Channel | int | Channel ID | ✅ |
| 1 | Opcode | int | Always `2` (PRO) | ✅ |
| 2 | Status | int | `0`=success/ready, `1`=error | ✅ |
| 3 | Error | str | Error message if status=1 | ❌ |
| 4 | ID | str | Message tracking ID | ❌ |

**Examples:**

```
// Success (ready for next command)
[1, 2, 0]

// Error
[1, 2, 1, "KeyboardInterrupt"]

// With message ID
[2, 2, 0, null, "req-123"]
```

#### 5.4.6 INT (Interrupt)

```Client→Server
[channel, 1]

// Example (Channel 1, Terminal):
[1, 1]
// CBOR: 82 01 01 (3 bytes)
```

Interrupts code execution on the specified channel (equivalent to Ctrl-C).

#### 5.4.7 RST (Reset)

```Client→Server
[channel, 2, mode]
```

| Position | Field | Type | Description | Required |
|----------|-------|------|-------------|----------|
| 2 | Mode | int | `0`=soft reset, `1`=hard reset | ✅ |

---

## 6. File Operations (Channel 23)

### 6.1 Overview

File operations use **TFTP-based semantics** (RFC 1350, 2347, 2348, 2349) adapted for CBOR encoding over WebSocket. This provides a proven, reliable block-based transfer protocol with acknowledgments and error handling.

**Format:** `[23, opcode, ...fields]`

Channel 23 is reserved exclusively for file operations.

**Key Features:**
- Block-based transfers with ACK confirmation
- Configurable block size (default: 4096 bytes for ESP32 flash alignment)
- Transfer size negotiation for progress tracking
- Standard TFTP error codes
- Supports files up to 268 MB (65535 blocks × 4KB)

**Note:** All file transfers are binary (TFTP "octet" mode). Text mode conversion is not supported.

**Note:** Directory listing and file info operations should be performed via **channel 2 (M2M)** using Python code execution.

### 6.2 File Opcodes

| Opcode | Name | Direction | Description |
|--------|------|-----------|-------------|
| `1` | RRQ | C→S | Read Request (download) |
| `2` | WRQ | C→S | Write Request (upload) |
| `3` | DATA | Both | Data block |
| `4` | ACK | Both | Acknowledgment |
| `5` | ERROR | S→C | Error response |

### 6.3 Constants

```python
DEFAULT_BLKSIZE = 4096    # 4KB - ESP32 flash sector size
DEFAULT_TIMEOUT = 5000    # 5 seconds
MAX_BLOCK_NUMBER = 65535  # 16-bit block number
```

### 6.4 Message Schemas

#### 6.4.1 WRQ (Write Request) - Upload


```Client→Server
[23, 2, filename, tsize, ?blksize, ?timeout, ?mtime]
```

| Position | Field | Type | Description | Required |
|----------|-------|------|-------------|----------|
| 0 | Channel | int | Always `23` (file operations) | ✅ |
| 1 | Opcode | int | `2` (WRQ) | ✅ |
| 2 | Filename | str | File path | ✅ |
| 3 | Tsize | int | Total file size in bytes | ✅ |
| 4 | Blksize | int | Block size (default: 4096) | ❌ |
| 5 | Timeout | int | Timeout in milliseconds (default: 5000) | ❌ |
| 6 | Mtime | int | Modification timestamp (Unix) | ❌ |

**Server → Client (ACK 0 with Confirmation):**

```
[23, 4, 0, tsize, ?blksize]
```

| Position | Field | Type | Description | Required |
|----------|-------|------|-------------|----------|
| 2 | Block# | int | Always `0` (ready to receive) | ✅ |
| 3 | Tsize | int | Echo back file size (confirms space available) | ✅ |
| 4 | Blksize | int | Echo back block size (confirms or adjusts) | ❌ |

If server rejects the request (insufficient space, file too large, etc.), sends ERROR instead.

**Client → Server (DATA Blocks):**

```
[23, 3, block#, data]
```

| Position | Field | Type | Description |
|----------|-------|------|-------------|
| 2 | Block# | int | Block number (1-65535) |
| 3 | Data | bytes | Block data (≤ blksize bytes) |

**Last block**: When `len(data) < blksize`, indicates end of transfer.

**Server → Client (ACK Each Block):**

```
[23, 4, block#]
```

**Examples:**

```
// Client initiates upload: 102400 bytes, 4KB blocks
[23, 2, "/main.py", 102400, 4096, 5000, 1733279222]

// Server confirms and ready to receive
[23, 4, 0, 102400, 4096]

// Client sends block 1 (4096 bytes)
[23, 3, 1, h'<4096 bytes>']

// Server acknowledges block 1
[23, 4, 1]

// Client sends block 2
[23, 3, 2, h'<4096 bytes>']

// Server acknowledges block 2
[23, 4, 2]

// ... blocks 3-24 ...

// Client sends block 25 (last block, 2048 bytes < 4096)
[23, 3, 25, h'<2048 bytes>']

// Server acknowledges block 25 (transfer complete)
[23, 4, 25]
```

**Error Response:**

```
// Server error (disk full)
[23, 5, 3, "Disk full or allocation failed"]

// Server error (file too large)
[23, 5, 0, "File size exceeds limit"]
```

#### 6.4.2 RRQ (Read Request) - Download


```Client→Server
[23, 1, filename, ?blksize, ?timeout]
```

| Position | Field | Type | Description | Required |
|----------|-------|------|-------------|----------|
| 0 | Channel | int | Always `23` (file operations) | ✅ |
| 1 | Opcode | int | `1` (RRQ) | ✅ |
| 2 | Filename | str | File path | ✅ |
| 3 | Blksize | int | Block size (default: 4096) | ❌ |
| 4 | Timeout | int | Timeout in milliseconds (default: 5000) | ❌ |

**Note:** Client does not send `tsize` in request. Server will provide total size in first DATA packet metadata (optional extension) or client deduces end-of-file when receiving block with `len(data) < blksize`.

**Server → Client (ACK 0 with Metadata):**

```
[23, 4, 0, tsize, ?mtime, ?mode]
```

| Position | Field | Type | Description | Required |
|----------|-------|------|-------------|----------|
| 2 | Block# | int | Always `0` (ready to send) | ✅ |
| 3 | Tsize | int | Total file size in bytes | ✅ |
| 4 | Mtime | int | Modification timestamp (Unix) | ❌ |
| 5 | Mode | int | File permissions | ❌ |

**Client → Server (ACK 0 - Ready to Receive):**

```
[23, 4, 0]
```

**Server → Client (DATA Blocks):**

```
[23, 3, block#, data]
```

| Position | Field | Type | Description | Required |
|----------|-------|------|-------------|----------|
| 2 | Block# | int | Block number (1-65535) | ✅ |
| 3 | Data | bytes | Block data (≤ blksize bytes) | ✅ |

**Last block**: When `len(data) < blksize`, indicates end of transfer.

**Client → Server (ACK Each Block):**

```
[23, 4, block#]
```

**Examples:**

```
// Client requests download with 4KB blocks and 5000ms timeout
[23, 1, "/firmware.bin", 4096, 5000]

// Server responds with ACK 0 + metadata (file is 100KB)
[23, 4, 0, 102400, 1733279222, 420]

// Client acknowledges (ready to receive)
[23, 4, 0]

// Server sends block 1
[23, 3, 1, h'<4096 bytes>']

// Client acknowledges block 1
[23, 4, 1]

// Server sends block 2
[23, 3, 2, h'<4096 bytes>']

// Client acknowledges block 2
[23, 4, 2]

// ... blocks 3-24 ...

// Server sends block 25 (last block, 2048 bytes < 4096)
[23, 3, 25, h'<2048 bytes>']

// Client acknowledges block 25 (transfer complete)
[23, 4, 25]
```

**Error Response:**

```
// File not found
[23, 5, 1, "File not found"]

// Permission denied
[23, 5, 2, "Access violation"]
```

#### 6.4.3 DATA Packet

**Format:**

```
[23, 3, block#, data]
```

| Position | Field | Type | Description | Required |
|----------|-------|------|-------------|----------|
| 0 | Channel | int | Always `23` (file operations) | ✅ |
| 1 | Opcode | int | `3` (DATA) | ✅ |
| 2 | Block# | int | Block number (1-65535) | ✅ |
| 3 | Data | bytes | Block data (≤ blksize bytes) | ✅ |

**Usage:**
- Sent by client during upload (WRQ)
- Sent by server during download (RRQ)
- Block numbers start at 1 and increment sequentially
- **Last block**: When `len(data) < blksize`, indicates end of file transfer

**Examples:**

```
// DATA block 1 with full 4096 bytes
[23, 3, 1, h'<4096 bytes>']

// DATA block 25 (middle block, full)
[23, 3, 25, h'<4096 bytes>']

// DATA block 26 (last block, partial - indicates EOF)
[23, 3, 26, h'<2048 bytes>']
```

#### 6.4.4 ACK (Acknowledgment) Packet

**Format:**

```
[23, 4, block#, ?tsize, ?mtime, ?mode]
```

| Position | Field | Type | Description | Required |
|----------|-------|------|-------------|----------|
| 0 | Channel | int | Always `23` (file operations) | ✅ |
| 1 | Opcode | int | `4` (ACK) | ✅ |
| 2 | Block# | int | Block number being acknowledged (0-65535) | ✅ |
| 3 | Tsize | int | Total file size (ACK 0 for RRQ only) | ❌ |
| 4 | Mtime | int | Modification timestamp (ACK 0 for RRQ only) | ❌ |
| 5 | Mode | int | File permissions (ACK 0 for RRQ only) | ❌ |

**Usage:**

**ACK 0 - Special meanings:**
- **WRQ response**: Server sends `[23, 4, 0, tsize, ?blksize]` to confirm options and indicate ready to receive
- **RRQ response**: Server sends `[23, 4, 0, tsize, ?mtime, ?mode]` with file metadata
- **RRQ confirmation**: Client sends `[23, 4, 0]` to confirm metadata and request first block

**ACK N (N > 0):**
- Acknowledges receipt of DATA block N
- Sent by receiver after successfully receiving and processing the block

**Examples:**

```
// WRQ: Server confirms options and ready to receive
[23, 4, 0, 102400, 4096]

// RRQ: Server sends file metadata
[23, 4, 0, 102400, 1733279222, 420]

// RRQ: Client confirms and requests data
[23, 4, 0]

// Acknowledge DATA block 5
[23, 4, 5]

// Acknowledge last block (25)
[23, 4, 25]
```

#### 6.4.5 ERROR Packet

```Server→Client
[23, 5, error_code, error_msg]
```

| Position | Field | Type | Description | Required |
|----------|-------|------|-------------|----------|
| 2 | Error Code | int | TFTP error code (0-8) | ✅ |
| 3 | Error Msg | str | Human-readable error message | ✅ |

**TFTP Error Codes (RFC 1350, 2347):**

| Code | Name | Description |
|------|------|-------------|
| `0` | Not defined | Generic error |
| `1` | File not found | File does not exist |
| `2` | Access violation | Permission denied |
| `3` | Disk full | No space left on device |
| `4` | Illegal TFTP operation | Invalid opcode or malformed request |
| `5` | Unknown transfer ID | Invalid block number or sequence error |
| `6` | File already exists | File exists (if exclusive create requested) |
| `7` | No such user | Authentication failure |
| `8` | Option negotiation failed | Invalid options in RRQ/WRQ |

### 6.5 File Management via M2M

File deletion, directory listing, file info, and folder creation should be performed using **channel 2 (M2M)** with Python code execution:

**Delete File:**
```
[2, 0, "import os; os.remove('/old_config.json')"]
// Response: [2, 2, 0]  // Progress: success
```

**List Directory:**
```
[2, 0, "import os, json; print(json.dumps(os.listdir('/lib')))"]
// Response: [2, 0, '["mqtt.py", "drivers", "helpers.py"]']
// Note: Production clients (like ScriptO Studio) often use optimized scripts with os.ilistdir() for detailed stats.
```

**Get File Info:**
```
[2, 0, "import os, json; print(json.dumps(os.stat('/main.py')))"]
// Response: [2, 0, '[0, 0, 0, 0, 0, 0, 1024, 1733279222, 1733279222, 1733279222]']
```

**Create Directory:**
```
[2, 0, "import os; os.mkdir('/data')"]
// Response: [2, 2, 0]  // Progress: success
```

This approach leverages the existing M2M channel for structured responses without adding protocol complexity.

---

## 7. Connection Negotiation

### 7.1 WebSocket Handshake

Clients negotiate the protocol using the WebSocket subprotocol mechanism:

**Client Request:**

```http
GET /WebREPL HTTP/1.1
Host: 192.168.4.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Protocol: WebREPL.binary.v1, WebREPL.text.v1
Sec-WebSocket-Version: 13
```

**Server Response:**

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: WebREPL.binary.v1
```

### 7.2 Authentication Flow

1. **Client connects** (WebSocket handshake completes)
2. **Client sends AUTH** event with password
3. **Server responds** with AUTH_OK or AUTH_FAIL
4. **If successful**, client can now send channel/file messages

**Sequence:**

```
Client                          Server
  |                               |
  |  WebSocket Handshake          |
  |<----------------------------->|
  |                               |
  |  [0, 0, "secret"]             |
  |------------------------------>| (AUTH on channel 0)
  |                               |
  |  [0, 1]                       |
  |<------------------------------| (AUTH_OK on channel 0)
  |                               |
  |  (Authenticated)              |
```

---

## 8. Protocol Flows

### 8.1 Terminal Execution

**Client executes:** `for i in range(3): print(i)`

```
Client                          Server
  |  [1, 0, "for i in..."]       |
  |------------------------------>| (TRM: EXE)
  |                               | VM executes
  |  [1, 0, "0\n"]                |
  |<------------------------------| (TRM: RES)
  |  [1, 0, "1\n"]                |
  |<------------------------------| (TRM: RES)
  |  [1, 0, "2\n"]                |
  |<------------------------------| (TRM: RES)
  |  [1, 2, 0]                    |
  |<------------------------------| (TRM: PRO ready)
```

### 8.2 M2M Request

**Client calls:** `get_info()`

```
Client                          Server
  |  [2, 0, "get_info()\n"       |
  |  , 0, "r1"]                   |
  |------------------------------>| (M2M: EXE)
  |                               | Execute function
  |  [2, 0, "{\"heap\":...}",     |
  |   "r1"]                       |
  |<------------------------------| (M2M: RES)
  |  [2, 2, 0, null, "r1"]        |
  |<------------------------------| (M2M: PRO)
```

### 8.3 Binary Bytecode Execution

**Client uploads and executes `.mpy` (single block, 2KB):**

```
// WRQ: Upload 2KB module
[23, 2, "/lib/mymodule.mpy", 2048, 4096]

// Server ready
[23, 4, 0]

// DATA block 1 (last block, < 4096)
[23, 3, 1, h'<2048 bytes of .mpy>']

// Server acknowledges
[23, 4, 1]

// Execute bytecode directly
[1, 0, h'4D03...', 1]

// Or import and use
[1, 0, "import mymodule\nmymodule.main()\n"]
```

### 8.4 File Upload (TFTP-style)

**Client uploads 12KB file (3 blocks × 4KB):**

```
Client                          Server
  |  [23, 2, "/main.py",          |
  |   12288, 4096]                 |
  |------------------------------>| WRQ: 12KB file, 4KB blocks
  |                               | Check space, create file
  |                               |
  |  [23, 4, 0, 12288, 4096]      |
  |<------------------------------| ACK 0 (confirmed, ready)
  |                               |
  |  [23, 3, 1, h'<4096 bytes>']  |
  |------------------------------>| DATA block 1
  |                               | Write to flash sector
  |                               |
  |  [23, 4, 1]                   |
  |<------------------------------| ACK block 1
  |                               |
  |  [23, 3, 2, h'<4096 bytes>']  |
  |------------------------------>| DATA block 2
  |                               |
  |  [23, 4, 2]                   |
  |<------------------------------| ACK block 2
  |                               |
  |  [23, 3, 3, h'<4096 bytes>']  |
  |------------------------------>| DATA block 3 (last)
  |                               | Close file, set mtime
  |                               |
  |  [23, 4, 3]                   |
  |<------------------------------| ACK block 3 (complete)
```

**Progress tracking:** Client calculates `(block# × 4096) / 12288 × 100%`

### 8.5 File Download (TFTP-style)

**Client downloads 10KB file:**

```
Client                          Server
  |  [23, 1, "/config.json",     |
  |   4096]                        |
  |------------------------------>| RRQ: 4KB blocks
  |                               | Open file, get size
  |                               |
  |  [23, 4, 0, 10240, 1733279222]|
  |<------------------------------| ACK 0 with metadata (10KB file)
  |                               |
  |  [23, 4, 0]                   |
  |------------------------------>| ACK 0 (ready to receive)
  |                               |
  |  [23, 3, 1, h'<4096 bytes>']  |
  |<------------------------------| DATA block 1
  |                               |
  |  [23, 4, 1]                   |
  |------------------------------>| ACK block 1
  |                               |
  |  [23, 3, 2, h'<4096 bytes>']  |
  |<------------------------------| DATA block 2
  |                               |
  |  [23, 4, 2]                   |
  |------------------------------>| ACK block 2
  |                               |
  |  [23, 3, 3, h'<2048 bytes>']  |
  |<------------------------------| DATA block 3 (last, < 4096)
  |                               |
  |  [23, 4, 3]                   |
  |------------------------------>| ACK block 3 (complete)
```

**Progress tracking:** Client knows total size from ACK 0: `(received_bytes / 10240) × 100%`

### 8.6 Tab Completion

**Client requests completion for `sys.p`:**

```
Client                          Server
  |  [1, 0, "sys.p\t"]           |
  |------------------------------>| (EXE with \t)
  |                               | Compute completions
  |  [1, 3, ["sys.path",          |
  |          "sys.platform",      |
  |          "sys.print_exception"]]
  |<------------------------------| (COM)
```

---

## 9. Wire Format Examples

### 9.1 Execute Python Code

**Message:**

```
[1, 0, "print('hello')\n"]
```

**CBOR Hex:**

```
83                      // Array(3)                 - 1 byte
  01                    // 1 (channel: TRM)         - 1 byte
  00                    // 0 (opcode: EXE)          - 1 byte
  6F                    // Text(15)                 - 1 byte
    7072696E74282768656C6C6F27290A  // "print('hello')\n" - 15 bytes
```

### 9.2 Execute .mpy Bytecode

**Message:**

```
[1, 0, h'4D0306001F', 1]
```

**CBOR Hex:**

```
84                      // Array(4)
  01                    // 1 (channel: TRM)
  00                    // 0 (opcode: EXE)
  45                    // Bytes(5)
    4D0306001F          // .mpy header
  01                    // 1 (format: mpy)
```

### 9.3 File Upload - WRQ (Write Request)

**Message:**

```
[23, 2, "/main.py", 12288, 4096]
```

**CBOR Hex:**

```
85                      // Array(5)
  17                    // 23 (channel: file ops)
  02                    // 2 (opcode: WRQ)
  68                    // Text(8)
    2F6D61696E2E7079    // "/main.py"
  19 3000               // 12288 (tsize)
  19 1000               // 4096 (blksize)
```


### 9.4 File Upload - DATA Block

**Message:**

```
[23, 3, 1, h'<4096 bytes>']
```

**CBOR Hex:**

```
84                      // Array(4)
  17                    // 23 (channel: file ops)
  03                    // 3 (opcode: DATA)
  01                    // 1 (block number)
  59 1000               // Bytes(4096)
    <4096 bytes of data>
```

### 9.5 File Upload - ACK

**Message:**

```
[23, 4, 1]
```

**CBOR Hex:**

```
83                      // Array(3)
  17                    // 23 (channel: file ops)
  04                    // 4 (opcode: ACK)
  01                    // 1 (block number)
```

### 9.6 Authentication

**Message:**

```
[0, 0, "secret"]
```

**CBOR Hex:**

```
83                      // Array(3)
  00                    // 0 (channel: events)
  00                    // 0 (event: AUTH)
  66                    // Text(6)
    736563726574        // "secret"
```

---

## 10. Implementation Notes

### 10.1 Server Implementation (MicroPython)

```python
import cbor2
import json
import websocket

# Constants
CH_EVENT, CH_TRM, CH_M2M, CH_DBG, CH_LOG, CH_FILE = 0, 1, 2, 3, 4, 23
EXE, INT, RST = 0, 1, 2  # Client → Server opcodes
RES, CON, PRO, COM = 0, 1, 2, 3  # Server → Client opcodes

# TFTP-style file operations (RFC 1350)
RRQ, WRQ, DATA, ACK, ERROR = 1, 2, 3, 4, 5
DEFAULT_BLKSIZE = 4096  # 4KB blocks (ESP32 flash sector size)
DEFAULT_TIMEOUT = 5000  # 5 seconds
# Note: All transfers are binary (no mode field needed)

# Events
AUTH, AUTH_OK, AUTH_FAIL, INFO, LOG = 0, 1, 2, 3, 4

current_channel = None
authenticated = False

def handle_message(data):
    msg = cbor2.loads(data)
    channel = msg[0]
    
    if channel == CH_EVENT:
        handle_event(msg)
    elif channel == CH_FILE:
        handle_file(msg)
    else:
        # Regular execution channels (1-22)
        handle_channel(msg)

def handle_channel(msg):
    global current_channel
    ch, op, *rest = msg
    
    if not authenticated:
        send_error(ch, "Not authenticated")
        return
    
    if op == EXE:
        data = rest[0]
        format = rest[1] if len(rest) > 1 else 0
        id = rest[2] if len(rest) > 2 else None
        
        current_channel = ch
        
        if format == 1:
            # Execute .mpy bytecode
            exec(data)
        else:
            # Execute Python source
            if data.endswith('\t'):
                # Tab completion
                completions = get_completions(data[:-1])
                send([ch, COM, completions])
            else:
                exec(data)
                send([ch, PRO, 0, None, id])
        
        current_channel = None
    
    elif op == INT:
        interrupt_channel(ch)
    
    elif op == RST:
        mode = rest[0]
        reset(mode)

def drain_output(data):
    """Route print() output to current channel"""
    if current_channel is not None:
        send([current_channel, RES, data])
    else:
        send([CH_TRM, RES, data])  # Default to TRM

file_state = None  # Tracks active file transfer: {'op': WRQ/RRQ, 'path': str, 'blksize': int, 'f': file, 'next_block': int}

def handle_file(msg):
    global file_state
    _, opcode, *rest = msg
    
    if opcode == WRQ:
        # Write Request (Upload)
        filename, tsize, blksize, *opts = rest
        blksize = blksize if blksize else DEFAULT_BLKSIZE
        
        # Check if file is too large
        if tsize > 1048576:  # 1MB limit
            send([CH_FILE, ERROR, 0, "File too large"])
            return
        
        # Create file handle
        try:
            f = open(filename, 'wb')
            file_state = {'op': WRQ, 'path': filename, 'blksize': blksize, 'f': f, 'next_block': 1, 'tsize': tsize}
            send([CH_FILE, ACK, 0])  # ACK block 0 (ready to receive)
        except OSError as e:
            send([CH_FILE, ERROR, 3, str(e)])  # Disk full or error
    
    elif opcode == RRQ:
        # Read Request (Download)
        filename, blksize, *opts = rest
        blksize = blksize if blksize else DEFAULT_BLKSIZE
        
        try:
            import os
            stat = os.stat(filename)
            tsize = stat[6]
            mtime = stat[8]
            mode = stat[0]
            
            # Send ACK 0 with metadata
            send([CH_FILE, ACK, 0, tsize, mtime, mode])
            
            # Open file and prepare for transfer (wait for client ACK 0)
            f = open(filename, 'rb')
            file_state = {'op': RRQ, 'path': filename, 'blksize': blksize, 'f': f, 'next_block': 1, 'tsize': tsize}
        except OSError as e:
            send([CH_FILE, ERROR, 1, str(e)])  # File not found
    
    elif opcode == DATA:
        # Receiving DATA block (upload)
        block_num, data = rest[0], rest[1]
        
        if file_state and file_state['op'] == WRQ:
            if block_num == file_state['next_block']:
                file_state['f'].write(data)
                send([CH_FILE, ACK, block_num])
                file_state['next_block'] += 1
                
                # Check if last block (< blksize)
                if len(data) < file_state['blksize']:
                    file_state['f'].close()
                    file_state = None
            else:
                send([CH_FILE, ERROR, 5, f"Expected block {file_state['next_block']}, got {block_num}"])
    
    elif opcode == ACK:
        # Receiving ACK (download or upload)
        block_num = rest[0]
        
        if file_state and file_state['op'] == RRQ:
            if block_num == 0:
                # Client acknowledged metadata, start sending data
                data = file_state['f'].read(file_state['blksize'])
                send([CH_FILE, DATA, 1, data])
                file_state['next_block'] = 2
                
                # Check if last block
                if len(data) < file_state['blksize']:
                    file_state['f'].close()
                    file_state = None
            elif block_num == file_state['next_block'] - 1:
                # Send next block
                data = file_state['f'].read(file_state['blksize'])
                if data:
                    send([CH_FILE, DATA, file_state['next_block'], data])
                    file_state['next_block'] += 1
                    
                    # Check if last block
                    if len(data) < file_state['blksize']:
                        file_state['f'].close()
                        file_state = None
                else:
                    # No more data
                    file_state['f'].close()
                    file_state = None
    

def handle_event(msg):
    global authenticated
    _, event, *rest = msg
    
    if event == AUTH:
        password = rest[0]
        if password == "correct_password":
            authenticated = True
            send([CH_EVENT, AUTH_OK])
        else:
            send([CH_EVENT, AUTH_FAIL, "Invalid password"])

def send(msg):
    data = cbor2.dumps(msg)
    websocket.send_binary(data)
```

### 10.2 Client Implementation (JavaScript)

```javascript
import CBOR from 'cbor-js';

const CH = {EVENT: 0, TRM: 1, M2M: 2, DBG: 3, LOG: 4, FILE: 23};
const OP = {
  EXE: 0, INT: 1, RST: 2,  // Client → Server
  RES: 0, CON: 1, PRO: 2, COM: 3  // Server → Client
};

// TFTP-style file operations (RFC 1350)
const FILE_OP = {
  RRQ: 1, WRQ: 2, DATA: 3, ACK: 4, ERROR: 5
};
const DEFAULT_BLKSIZE = 4096;  // 4KB blocks

const EVENT = {AUTH: 0, AUTH_OK: 1, AUTH_FAIL: 2, INFO: 3, LOG: 4};

class WebREPL {
  constructor(url) {
    this.ws = new WebSocket(url, ['WebREPL.binary.v1']);
    this.ws.binaryType = 'arraybuffer';
    this.ws.onmessage = this.onMessage.bind(this);
    this.handlers = {channel: new Map(), event: new Map()};
    this.pendingFileOps = new Map();
    this.fileProgressCallbacks = new Map();
    this.fileState = null;  // Tracks active file transfer
  }
  
  onMessage(event) {
    const msg = CBOR.decode(event.data);
    const channel = msg[0];
    
    if (channel === CH.EVENT) {
      this.handleEvent(msg);
    } else if (channel === CH.FILE) {
      this.handleFile(msg);
    } else {
      // Regular execution channels (1-22)
      this.handleChannel(msg);
    }
  }
  
  handleChannel(msg) {
    const [ch, op, ...rest] = msg;
    const handler = this.handlers.channel.get(ch);
    
    if (op === OP.RES) {
      const data = rest[0];
      if (handler) handler.onResult(data);
    } else if (op === OP.PRO) {
      const status = rest[0];
      const error = rest[1];
      if (handler) handler.onProgress(status, error);
    } else if (op === OP.CON) {
      if (handler) handler.onContinuation();
    } else if (op === OP.COM) {
      const completions = rest[0];
      if (handler) handler.onCompletions(completions);
    }
  }
  
  handleFile(msg) {
    const [_, opcode, ...rest] = msg;
    
    if (opcode === FILE_OP.ACK) {
      // Receiving ACK
      const blockNum = rest[0];
      
      const state = this.fileState;
      
      // Check if this is ACK 0 with metadata (download response)
      if (blockNum === 0 && rest.length > 1 && state && state.op === FILE_OP.RRQ) {
        // Server sent metadata
        state.tsize = rest[1];
        state.mtime = rest[2];
        state.mode = rest[3];
        
        // Send ACK 0 to start receiving data
        this.send([CH.FILE, FILE_OP.ACK, 0]);
        return;
      }
      
      // ACK for upload
      if (state && state.op === FILE_OP.WRQ) {
        if (blockNum === 0) {
          // Server confirmed options - check if accepted
          const confirmedTsize = rest[1];
          const confirmedBlksize = rest[2];
          
          // Optionally verify confirmed options match request
          if (confirmedTsize && confirmedTsize !== state.data.length) {
            const reject = state.reject;
            this.fileState = null;
            reject(new Error(`Server confirmed different size: ${confirmedTsize} vs ${state.data.length}`));
            return;
          }
          
          // Update blksize if server adjusted it
          if (confirmedBlksize) {
            state.blksize = confirmedBlksize;
          }
          
          // Start sending blocks
          state.nextBlock = 1;
          this.sendNextBlock();
        } else if (blockNum === state.nextBlock - 1) {
          // ACK for previous block - send next
          this.sendNextBlock();
        }
      }
    } else if (opcode === FILE_OP.DATA) {
      // Receiving DATA block (download)
      const blockNum = rest[0];
      const data = rest[1];
      
      const state = this.fileState;
      if (state && state.op === FILE_OP.RRQ) {
        if (blockNum === state.nextBlock) {
          state.chunks.push(data);
          state.received += data.length;
          
          // Update progress
          if (state.tsize && state.onProgress) {
            const percent = Math.floor(state.received * 100 / state.tsize);
            state.onProgress(percent);
          }
          
          // Send ACK
          this.send([CH.FILE, FILE_OP.ACK, blockNum]);
          state.nextBlock++;
          
          // Check if last block (< blksize)
          if (data.length < state.blksize) {
            // Transfer complete - concatenate chunks
            const completeData = new Uint8Array(state.received);
            let offset = 0;
            for (const chunk of state.chunks) {
              completeData.set(chunk, offset);
              offset += chunk.length;
            }
            
            const resolve = state.resolve;
            this.fileState = null;
            resolve(completeData);
          }
        }
      }
    } else if (opcode === FILE_OP.ERROR) {
      // Error response
      const errorCode = rest[0];
      const errorMsg = rest[1];
      
      if (this.fileState) {
        const reject = this.fileState.reject;
        this.fileState = null;
        reject(new Error(`File error ${errorCode}: ${errorMsg}`));
      }
    }
  }
  
  sendNextBlock() {
    const state = this.fileState;
    if (!state || state.op !== FILE_OP.WRQ) return;
    
    const offset = (state.nextBlock - 1) * state.blksize;
    if (offset >= state.data.length) {
      // Transfer complete
      const resolve = state.resolve;
      this.fileState = null;
      resolve();
      return;
    }
    
    const chunk = state.data.slice(offset, offset + state.blksize);
    this.send([CH.FILE, FILE_OP.DATA, state.nextBlock, chunk]);
    
    // Update progress
    if (state.onProgress) {
      const percent = Math.floor((offset + chunk.length) * 100 / state.data.length);
      state.onProgress(percent);
    }
    
    state.nextBlock++;
  }
  
  handleEvent(msg) {
    const [_, event, ...rest] = msg;
    const handler = this.handlers.event.get(event);
    if (handler) handler(...rest);
  }
  
  // API methods
  
  async authenticate(password) {
    return new Promise((resolve, reject) => {
      this.handlers.event.set(EVENT.AUTH_OK, () => resolve());
      this.handlers.event.set(EVENT.AUTH_FAIL, (err) => reject(err));
      this.send([CH.EVENT, EVENT.AUTH, password]);
    });
  }
  
  execute(channel, code, format = 0) {
    this.send([channel, OP.EXE, code, format]);
  }
  
  executeBytecode(channel, mpy) {
    this.send([channel, OP.EXE, mpy, 1]);
  }
  
  async uploadFile(path, data, onProgress, blksize = DEFAULT_BLKSIZE) {
    return new Promise((resolve, reject) => {
      // Set up file state
      this.fileState = {
        op: FILE_OP.WRQ,
        path: path,
        data: data,
        blksize: blksize,
        nextBlock: 1,
        onProgress: onProgress,
        resolve: resolve,
        reject: reject
      };
      
      // Send WRQ
      this.send([CH.FILE, FILE_OP.WRQ, path, data.length, blksize]);
    });
  }
  
  async downloadFile(path, onProgress, blksize = DEFAULT_BLKSIZE) {
    return new Promise((resolve, reject) => {
      // Set up file state
      this.fileState = {
        op: FILE_OP.RRQ,
        path: path,
        blksize: blksize,
        chunks: [],
        received: 0,
        nextBlock: 1,
        tsize: null,
        mtime: null,
        onProgress: onProgress,
        resolve: resolve,
        reject: reject
      };
      
      // Send RRQ
      this.send([CH.FILE, FILE_OP.RRQ, path, blksize]);
    });
  }
  
  async deleteFile(path) {
    // Use M2M channel for file deletion
    return this.executeM2M(`import os; os.remove('${path.replace(/'/g, "\\'")}')`);
  }
  
  async listDirectory(path) {
    // Use M2M channel for directory operations
    return this.executeM2M(`import os, json; print(json.dumps(os.listdir('${path}')))`);
  }
  
  send(msg) {
    const data = CBOR.encode(msg);
    this.ws.send(data);
  }
  
  onChannel(channel, handler) {
    this.handlers.channel.set(channel, handler);
  }
}

// Usage
const repl = new WebREPL('ws://192.168.4.1/WebREPL');

await repl.authenticate('secret');

repl.onChannel(CH.TRM, {
  onResult: (data) => terminal.write(data),
  onProgress: (status, error) => {
    if (status === 0) terminal.setReady();
    else terminal.showError(error);
  },
  onContinuation: () => terminal.showPrompt('...'),
  onCompletions: (items) => terminal.showCompletions(items)
});

repl.execute(CH.TRM, "print('hello')\n");

// Upload and execute bytecode
const mpy = await fetch('module.mpy').then(r => r.arrayBuffer());
await repl.uploadFile('/lib/module.mpy', new Uint8Array(mpy));
repl.execute(CH.TRM, "import module\nmodule.main()\n");
```

### 10.3 Channel-Aware Output Routing

The device tracks which channel initiated execution and routes all `print()` output to that channel automatically:

```python
current_channel = None

def handle_exe(ch, data, format):
    global current_channel
    current_channel = ch
    
    # All print() calls during exec will go to current_channel
    if format == 0:
        exec(data)
    else:
        exec(data)  # .mpy bytecode
    
    send([ch, PRO, 0])
    current_channel = None

# Hook into MicroPython's stdout
import sys
class ChannelWriter:
    def write(self, data):
        drain_output(data)

sys.stdout = ChannelWriter()
```

This means M2M helpers can simply use `print(json.dumps(...))` and the output automatically goes to the M2M channel.

---

## 11. Security Considerations

### 11.1 Transport Security

- **Use wss://** (WebSocket Secure) in production deployments
- **ws:// only** for local development or isolated networks
- Implement TLS 1.3 or higher with strong cipher suites

### 11.2 Authentication

- **Require authentication** before allowing any channel/file operations
- Implement **rate limiting**: Maximum 5 auth attempts per minute
- Support **exponential backoff** on failed attempts
- Consider adding:
  - Token-based authentication with expiration
  - Multi-user support with usernames
  - OAuth2 integration for enterprise deployments

### 11.3 Input Validation

- **Validate channel IDs**: Must be 0-254
- **Validate paths**: Prevent directory traversal (`../`, absolute paths outside allowed dirs)
- **Limit file sizes**: Maximum 1MB per file (configurable)
- **Limit message sizes**: Maximum 64KB per WebSocket frame
- **Validate array lengths**: Minimum required fields must be present

### 11.4 Resource Protection

- **Implement quotas**:
  - Maximum concurrent executions per channel
  - Maximum file storage per session
  - Maximum upload rate (bytes/second)
- **Connection timeouts**: Close idle connections after 5 minutes
- **Watchdog timers**: Kill executions exceeding time limit

### 11.5 Code Execution Sandbox

- Consider restricting available modules in `exec()` context
- Disable dangerous operations (e.g., `os.system`, `eval` of user input)
- Run MicroPython with memory limits

---

## 12. Migration from Legacy Protocol

### 13.1 Detection Strategy

Clients can detect protocol support via subprotocol negotiation:

```javascript
const ws = new WebSocket('ws://device/WebREPL', [
  'WebREPL.binary.v1',      // Preferred
  'WebREPL.text.v1'     // Legacy
]);

ws.onopen = () => {
  if (ws.protocol === 'WebREPL.binary.v1') {
    useWBP();
  } else {
    useLegacy();
  }
};
```

### 13.2 Feature Comparison

| Feature | Legacy (WA/WB/WC) | WBP (CBOR) |
|---------|-------------------|------------|
| **Binary execution** | ❌ | ✅ |
| **Message overhead** | 4-82 bytes | 4-15 bytes |
| **Filename limit** | 64 chars | Unlimited |
| **File metadata** | None | Timestamps, permissions |
| **Directory listing** | Not in protocol | Native |
| **Extensibility** | Fixed headers | Trailing fields |
| **Self-describing** | No | Yes |

### 13.3 Transition Period

During migration, servers MAY support both protocols concurrently:

- Detect protocol via subprotocol header
- Maintain separate handlers for legacy and CBOR
- Deprecate legacy protocol after adoption period

---

## Appendix A: CBOR Encoding Reference

### Common Encodings

| Value | CBOR Hex | Description |
|-------|----------|-------------|
| `0` | `00` | Integer 0 |
| `1` | `01` | Integer 1 |
| `255` | `18 FF` | Integer 255 |
| `true` | `F5` | Boolean true |
| `false` | `F4` | Boolean false |
| `null` | `F6` | Null |
| `""` | `60` | Empty string |
| `"a"` | `61 61` | Text(1) "a" |
| `h''` | `40` | Empty bytes |
| `h'FF'` | `41 FF` | Bytes(1) |
| `[]` | `80` | Empty array |
| `[1]` | `81 01` | Array(1) |
| `{}` | `A0` | Empty map |
| `{0: 1}` | `A1 00 01` | Map(1) |

### Size Optimizations

- Integers 0-23: 1 byte
- Integers 24-255: 2 bytes
- Strings up to 23 chars: 1 byte header + data
- Arrays up to 23 items: 1 byte header + items

---

## Appendix B: Error Codes

### Channel Execution Errors

Return via PRO message with status=1:

| Error | Description |
|-------|-------------|
| `"SyntaxError"` | Invalid Python syntax |
| `"NameError"` | Undefined name |
| `"KeyboardInterrupt"` | User interrupted (Ctrl-C) |
| `"MemoryError"` | Out of memory |
| `"TimeoutError"` | Execution exceeded time limit |

### File Operation Errors

Return via status=1:

| Error | Description |
|-------|-------------|
| `"File not found"` | Path does not exist |
| `"Permission denied"` | Insufficient permissions |
| `"Disk full"` | No space left on device |
| `"Invalid path"` | Path contains invalid characters |
| `"Is a directory"` | Tried to read/write directory as file |

---

## Appendix C: Protocol Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | Dec 2025 | Initial WebREPL Binary Protocol specification |

---

## References

- [RFC 6455](https://www.rfc-editor.org/rfc/rfc6455) - The WebSocket Protocol
- [RFC 8949](https://www.rfc-editor.org/rfc/rfc8949) - Concise Binary Object Representation (CBOR)
- [MicroPython Documentation](https://docs.micropython.org/)

---

Jonathan Peace  
Email: jep@alphabetiq.com
GitHub: [@sjetpax](https://github.com/jetpax)
