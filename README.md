# WebREPL Protocol Specifications

This repository contains protocol specifications for WebREPL, a WebSocket-based protocol for accessing remote Python REPL (Read-Eval-Print Loop) and performing file transfer operations, primarily used with MicroPython devices.

## Documents

### 1. [webrepl_rfc.txt](webrepl_rfc.txt) - Legacy WebREPL Protocol

An RFC-style format specification  of the original WebREPL protocol that describes:

- **Connection & Authentication**: Password-based authentication over WebSocket
- **Terminal Mode**: Interactive REPL access using WebSocket Text frames
- **File Transfer Mode**: Binary frame-based file operations with "WA"/"WB" magic bytes
- **Operations**:
  - `PUT_FILE` - Upload files to device
  - `GET_FILE` - Download files from device
  - `GET_VER` - Get protocol version

**Key Characteristics:**
- Uses WebSocket Text frames for REPL I/O
- Uses WebSocket Binary frames with 82-byte fixed headers for file transfers
- 64-character filename limit
- Magic bytes: "WA" (client→server), "WB" (server→client)

### 2. [webrepl_cb_rfc.md](webrepl_cb_rfc.md) - WebREPL Channelized Binary (WBP) Protocol

A modern, next-generation WebREPL protocol using CBOR encoding and channel multiplexing.

**Subprotocol Identifier:** `webrepl.binary.v1`

**Major Improvements:**
- **Channelized Communication**: 255 independent channels for multiplexing
- **CBOR Encoding**: Compact, self-describing binary messages
- **Binary Bytecode Support**: Native `.mpy` execution
- **TFTP-based File Transfers**: Reliable, resumable block-based transfers
- **Event System**: Structured logs and system notifications
- **Extensible**: Optional trailing fields for future features

**Channel Allocation:**
- Channel 0: Events (auth, logs, notifications)
- Channel 1: Terminal REPL (Human-Machine Interface)
- Channel 2: Machine-to-Machine RPC
- Channel 3: Debug output
- Channel 23: File operations (TFTP-based)
- Channels 4-22: Reserved for future use
- Channels 24-254: Application-defined

**Features:**
- Sub-10 byte message overhead for common operations
- Unlimited filename lengths
- File metadata (timestamps, permissions)
- Progress tracking for file transfers
- 4KB block sizes aligned with ESP32 flash sectors
- Support for files up to 268 MB

## Use Cases

### Web-Based IDEs
Build browser-based development environments for MicroPython devices with:
- Multiple simultaneous terminal sessions
- Background file operations
- Real-time event notifications
- Structured logging

### Automation & Machine-to-Machine
- Automated device configuration
- Remote monitoring and control
- Firmware updates
- Data collection from IoT devices

### Embedded Development
- Interactive debugging
- Binary bytecode deployment
- Filesystem management
- Over-the-air updates

## Protocol Comparison

| Feature | Legacy (WA/WB) | WCB (CBOR) |
|---------|----------------|------------|
| **Binary execution** | ❌ | ✅ |
| **Message overhead** | 82 bytes | 4-15 bytes |
| **Filename limit** | 64 chars | Unlimited |
| **File metadata** | None | Timestamps, permissions |
| **Multiplexing** | Text/Binary split | 255 channels |
| **Extensibility** | Fixed headers | Trailing fields |
| **Self-describing** | No | Yes (CBOR) |
| **Progress tracking** | Manual | Built-in |

## Implementation Status

### Legacy Protocol
- ✅ **Server**: Implemented in MicroPython (ESP32, ESP8266)
- ✅ **Client**: Multiple implementations (webrepl_cli.py, web clients)
- ✅ **Production**: Widely deployed

### WebREPL Binary Protocol
- 🚧 **Server**: Under development (MicroPython/ESP32)
- 🚧 **Client**: Reference implementation in progress
- 📋 **Status**: Draft specification

## Getting Started

### Using Legacy WebREPL

```python
# On MicroPython device
import webrepl
webrepl.start(password='mypassword')
```

```bash
# Command-line client
webrepl_cli.py -p mypassword ws://192.168.4.1:8266/
```

### Implementing WebREPL Binary Protocol

See the detailed examples in [webrepl_cb_rfc.md](webrepl_cb_rfc.md):
- Server implementation (MicroPython)
- Client implementation (JavaScript)
- Message format examples
- Protocol flows

## Security Considerations

⚠️ **Important Security Notes:**

1. **Use WSS (WebSocket Secure)** in production deployments
2. **WS (unencrypted)** should only be used for:
   - Local development
   - Isolated/private networks
   - Initial device configuration on AP mode

3. **Authentication**: Both protocols use password-based authentication
4. **Rate Limiting**: Implement protection against brute-force attacks
5. **Input Validation**: Always validate paths, sizes, and opcodes

## Contributing

This repository contains protocol specifications. Implementations should be contributed to:
- **MicroPython**: For server-side implementation
- **ScriptO Studio**: For web-based client implementation
- **Other projects**: As appropriate

## Related Projects

- [MicroPython](https://github.com/micropython/micropython) - Python for microcontrollers
- [ScriptO Studio](https://github.com/scripto-studio/scripto-studio) - Web-based MicroPython IDE
- [mpDirect](https://github.com/jep/mpDirect) - Direct MicroPython communication layer

## Authors

**Jonathan Peace**  
Scripto Studio  
Email: jep@scripto.studio.com

## License

These specifications are provided for informational purposes to enable interoperable implementations.

---

**Status:**
- `webrepl_rfc.txt`: Informational (describes existing protocol)
- `webrepl_cb_rfc.md`: Draft (v1.0, December 2025)
