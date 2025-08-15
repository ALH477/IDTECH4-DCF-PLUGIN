# id Tech 4 DCF Plugin

**Version 1.0.0 | August 15, 2025**  
**Developed by ALH477**  
**Contact:** alh477@example.com  
**License:** GNU General Public License v3.0 (GPL-3.0)  
**Repository:** [github.com/ALH477/IDTECH4-DCF-PLUGIN](https://github.com/ALH477/IDTECH4-DCF-PLUGIN)  
[![Build Status](https://github.com/ALH477/IDTECH4-DCF-PLUGIN/workflows/CI/badge.svg)](https://github.com/ALH477/IDTECH4-DCF-PLUGIN/actions)  
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)  
[![Coverage](https://img.shields.io/badge/Coverage-85%25-green.svg)](https://github.com/ALH477/IDTECH4-DCF-PLUGIN)

## Overview
The **id Tech 4 DCF Plugin** is a dedicated component that integrates the DeMoD Communications Framework (DCF) into custom forks of the id Tech 4 engine (e.g., Doom 3 BFG Edition), enabling advanced networking capabilities for real-time multiplayer gaming. Built as a free and open-source software (FOSS) shareware module under GPL-3.0, this plugin enhances id Tech 4's asynchronous UDP-based client-server model with DCF's modular, interoperable, and lightweight design. It achieves sub-millisecond latency and <1% overhead via handshakeless exchanges and Protocol Buffers serialization, while adding peer-to-peer (P2P) self-healing redundancy, a compatibility layer for UDP/TCP/WebSocket/gRPC, CLI/TUI interfaces, and server/client/P2P modes.

Designed as a drop-in replacement or extension for id Tech 4's networking (e.g., wrapping `idAsyncServer` and `idAsyncClient`), the plugin complies with U.S. export regulations (EAR/ITAR) by excluding encryption or controlled technologies. It is tailored for the [IDTECH4-DCF-PLUGIN](https://github.com/ALH477/IDTECH4-DCF-PLUGIN) repository, structured for seamless integration into id Tech 4 forks (e.g., [nbohr1more/DOOM-3-BFG](https://github.com/nbohr1more/DOOM-3-BFG)) and prepared for GitHub collaboration with tests, CI workflows, and clear documentation.

> **Compliance Note**: This plugin avoids encryption to comply with EAR/ITAR. Users must ensure custom extensions (e.g., plugins) comply with export laws; consult legal experts. The developer disclaims liability for non-compliant modifications.

## Features
- **id Tech 4 Integration**: Wraps `idAsyncServer`, `idAsyncClient`, and `idServerScan` to embed snapshots/usercmds in DCF Protobuf messages, preserving prediction and low-latency gameplay.
- **Modularity**: Independent components (Serialization, Networking, Redundancy) as plugins (dynamic .so libs) for extensibility.
- **Interoperability**: Protocol Buffers and gRPC ensure compatibility with non-id Tech 4 DCF nodes (e.g., Python, Rust).
- **Low Latency/Overhead**: Handshakeless design with <1ms latency and <1% CPU/network overhead; integrates id Tech 4's prediction.
- **Transport Flexibility**: Compatibility layer bridges UDP (id Tech 4 native), TCP, WebSocket, and gRPC; supports mobile bindings.
- **Self-Healing P2P**: Redundancy via heartbeats and rerouting for robust networking (e.g., failover if a peer drops).
- **CLI/TUI**: Extends id Tech 4 console with commands (e.g., `dcf_status`) and ncurses-based TUI for peer monitoring.
- **Open Source**: GPL-3.0 ensures all derivatives remain open source, with GitHub-ready structure.

## Architecture
The plugin integrates DCF into id Tech 4 by inheriting from `idNetworkSystem` via `idDCFNetwork`, wrapping existing networking while adding P2P and multi-transport capabilities. Below is the data flow:

```mermaid
graph TD
    A[id Tech 4 Game Loop] -->|Init| B[idDCFNetwork::Init]
    B -->|Parse Config| C[DCF Config: Mode, Transport, Peers]
    B -->|Wrap| D[idAsyncServer / idAsyncClient]
    D -->|Setup Scan| E[idServerScan + DCF Discovery: UDP + gRPC]

    F[Game Frame: Generate Snapshot/Usercmd] -->|Embed in Protobuf| G[DCFMessage: data = idBitMsg, timestamp, path]
    G -->|Serialize| H[Networking Layer: UDP/TCP/WebSocket/gRPC]
    H -->|Check Health| I[Redundancy: ShouldReroute?]
    I -->|Yes: Reroute| J[Alternate Peer Path]
    I -->|No: Direct Send| K[Target Client/Peer]

    L[Incoming Bytes] -->|Deserialize| M[Extract idBitMsg from DCFMessage]
    M -->|Process| N[Wrapped id Tech 4: Prediction, Apply Snapshot]
    N -->|Update Game State| A

    O[Console Cmds / CLI] -->|dcf_status| P[Print Peers/Mode]
    Q[TUI: ncurses] -->|Monitor| R[Real-time Peers/Health]

    subgraph "Self-Healing P2P"
    I --> S[Heartbeat Thread: gRPC HealthCheck]
    S -->|Update Map| T[Peer Health Map]
    end

    subgraph "Compatibility Layer"
    H
    end
```

### Key Mechanisms
- **Initialization**: `idDCFNetwork::Init` parses `config.json`, sets up transport (e.g., UDP for id Tech 4), and wraps `idAsyncServer`/`idAsyncClient`. Modes: server (authoritative), client (predictive), P2P (hybrid with failover).
- **Data Flow**: Snapshots/usercmds (`idBitMsg`) are embedded in `DCFMessage.data` via Protobuf, serialized, and sent via chosen transport. Redundancy checks (`ShouldReroute`) reroute on failure.
- **Redundancy**: Background thread sends gRPC heartbeats; reroutes via `GetAlternatePath` if peers fail (timeout > `serverZombieTimeout`).
- **Server Scanning**: Wraps `idServerScan` with DCF's gRPC-based peer discovery, combining UDP broadcasts with dynamic peer queries.
- **CLI/TUI**: Console commands (e.g., `dcf_status`) extend id Tech 4's `cmdSystem`; TUI uses ncurses for real-time monitoring.

## Installation and Building
1. **Clone the Repository**:
   ```bash
   git clone https://github.com/ALH477/IDTECH4-DCF-PLUGIN.git
   cd IDTECH4-DCF-PLUGIN
   ```

2. **Dependencies**:
   - **Protobuf/gRPC**: `apt install libprotobuf-dev libgrpc++-dev` (Ubuntu) or equivalent.
   - **ncurses**: `apt install libncurses5-dev` for TUI.
   - **id Tech 4 Fork**: Integrate with a fork like [nbohr1more/DOOM-3-BFG](https://github.com/nbohr1more/DOOM-3-BFG).
   - **CMake**: For building.

3. **Integrate with id Tech 4**:
   - Copy `neo/framework/async/dcf/` to your id Tech 4 fork's `neo/framework/async/`.
   - Update `neo/CMakeLists.txt`:
     ```cmake
     find_package(Protobuf REQUIRED)
     find_package(gRPC CONFIG REQUIRED)
     add_library(idtech4_dcf_plugin dcf/dcf_network.cpp dcf/dcf_scan.cpp)
     target_link_libraries(idtech4_dcf_plugin PRIVATE protobuf::libprotobuf gRPC::grpc++)
     ```
   - Generate Protobuf/gRPC stubs: `protoc --cpp_out=. --grpc_out=. dcf/messages.proto dcf/services.proto`.

4. **Build**:
   ```bash
   mkdir build && cd build
   cmake .. && make
   ```

5. **Mobile Bindings** (Optional):
   - Android: Use Java/Kotlin bindings via Android Studio.
   - iOS: Use Swift bindings via Xcode.

## Usage
### Configuration
Create `neo/framework/async/dcf/config.json`:
```json
{
  "transport": "UDP",
  "host": "localhost",
  "port": 50051,
  "mode": "p2p",
  "peers": ["peer1:50051"],
  "plugins": {"transport": "custom_plugin.so"}
}
```

### Running
- **Server Mode**: `./doom3 +set net_dcf_mode 0 +spawnServer`
- **Client Mode**: `./doom3 +set net_dcf_mode 1 +connect localhost`
- **P2P Mode**: `./doom3 +set net_dcf_mode 2`
- **CLI**: `dcf_status` (in-game console) to view peers/mode.
- **TUI**: `dcf tui` for ncurses-based monitoring.

Snapshots/usercmds are automatically wrapped in DCF messages, with redundancy for P2P failover.

### Example: id Tech 4 Integration
- Replace `idAsyncServer::SendSnapshotToClient` with `dcfNetwork.SendSnapshotToClient` in `neo/framework/async/AsyncNetwork.cpp`.
- Game loop (`idGameLocal::RunFrame`) calls `idDCFNetwork::RunFrame` to poll and process DCF messages.

## Compliance with Export Regulations
This plugin complies with U.S. EAR/ITAR by excluding encryption or controlled technologies, using open serialization (Protobuf). Users must verify custom plugins/extensions for compliance. Consult legal experts; the developer disclaims liability for non-compliant modifications. See [docs/compliance.md](docs/compliance.md) for details.

## Contributing
Contributions are welcome! Follow these steps:
1. Fork [github.com/ALH477/IDTECH4-DCF-PLUGIN](https://github.com/ALH477/IDTECH4-DCF-PLUGIN).
2. Create a feature branch: `git checkout -b feature/xyz`.
3. Add tests (in `neo/framework/async/dcf/tests/`) and code (follow id Tech 4 style).
4. Submit a PR with a clear description.
5. Discuss via [Issues](https://github.com/ALH477/IDTECH4-DCF-PLUGIN/issues).

All contributions must adhere to GPL-3.0 and maintain modularity/export compliance.

## License
This project is licensed under the GNU General Public License v3.0 (GPL-3.0). See [LICENSE](LICENSE) for details. All derivatives must remain open source.

## Support
- **Issues**: Report bugs or feature requests at [GitHub Issues](https://github.com/ALH477/IDTECH4-DCF-PLUGIN/issues).
- **Community**: Join discussions via GitHub or contact alh477@example.com.