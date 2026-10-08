# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

Skybrush Server is the backend server for the Skybrush ecosystem, providing an abstraction layer for communicating with drones and managing drone shows. It uses an async Trio-based architecture with an extension system for modularity.

## Common Commands

### Setup and Installation
```bash
# Install dependencies and setup virtual environment
uv sync

# Run the server
uv run skybrushd

# Run with custom config
uv run skybrushd -c path/to/config.jsonc

# Run in debug mode
uv run skybrushd --debug

# Run gateway or proxy
uv run skybrush-gateway
uv run skybrush-proxy
```

### Development
```bash
# Run tests
uv run pytest

# Run specific test file
uv run pytest test/test_show_player.py

# Run tests with coverage
uv run pytest --cov

# Lint and format (via pre-commit hooks)
uv run ruff check --fix
uv run ruff format

# Generate documentation
uv run pdoc flockwave.server
```

### Versioning and Release
```bash
# Bump version (configured via tbump.toml)
tbump <new-version>
```

## Architecture

### Core Application Structure

**Main Application Object**: `SkybrushServer` (in `src/flockwave/server/app.py`)
- Inherits from `DaemonApp` (flockwave-app-framework)
- Uses Trio for async I/O throughout
- Lifecycle: `prepare()` → `run()` → shutdown

**Central Registries** (all in `src/flockwave/server/registries/`):
- `ObjectRegistry`: All objects known to the server (UAVs, beacons, etc.)
- `ClientRegistry`: Connected GCS clients
- `ConnectionRegistry`: Communication channels (TCP, UDP, MAVLink, etc.)
- `UAVDriverRegistry`: Registered UAV driver implementations
- `ChannelTypeRegistry`: Types of communication channels (Socket.IO, TCP, UDP, etc.)

**Message Hub** (`src/flockwave/server/message_hub.py`):
- Central messaging system for Flockwave protocol
- Handles request/response and pub/sub patterns
- Implements rate limiting for different message types
- Routes messages between clients and extensions

### Extension System

**Extension Base Classes** (`src/flockwave/server/ext/base.py`):
- `Extension`: Base for all extensions
- `UAVExtension`: Base for UAV provider extensions

**Built-in Extensions** (48 total in `src/flockwave/server/ext/`):
Core extensions include:
- `mavlink`: MAVLink drone communication (ArduPilot, PX4)
- `crazyflie`: Crazyflie drone support
- `show`: Drone show choreography execution
- `socketio`: Socket.IO client communication
- `http_server`: HTTP API and WebSocket server
- `rtk`: RTK GPS correction distribution
- `virtual_uavs`: Virtual drones for testing
- `weather`: Weather data providers
- `missions`: Mission planning and execution

Extensions are loaded via configuration and can be enabled/disabled dynamically.

### UAV Model and Drivers

**UAV Object** (`src/flockwave/server/model/uav.py`):
- Core abstraction representing a drone
- Backed by a `UAVDriver` that handles protocol-specific communication
- Provides unified interface for: position, battery, status, commands, parameters, etc.

**UAV Driver Pattern**:
Each UAV type (MAVLink, Crazyflie, virtual) implements `UAVDriver` interface to translate between the unified UAV model and protocol-specific details.

### Configuration

**Config Files** (`.jsonc` format in `etc/conf/`):
- `skybrush-outdoor.jsonc`: MAVLink outdoor drones
- `skybrush-indoor.jsonc`: Indoor motion capture setup
- `skybrush-virtual.jsonc`: Virtual drones for testing
- Configuration structure: `{"EXTENSIONS": {"extension_name": {config}}}`

**Loading Order**:
1. Environment variables from `.env` (if present)
2. Config file specified via `-c` or default locations
3. Command-line overrides (e.g., `--port`)

### Flockwave Protocol

The server implements the Flockwave protocol for GCS↔Server communication:
- **Messages**: Request/response pairs (e.g., `UAV-INF` request → response)
- **Notifications**: Server→client broadcasts (e.g., `UAV-STATUS` updates)
- Message types defined in `src/flockwave/server/model/messages.py`
- Command handlers in `src/flockwave/server/command_handlers/`

### Asynchronous Patterns

**Trio Usage**:
- All async code uses Trio (not asyncio)
- Nurseries for structured concurrency
- `trio_util.periodic()` for recurring tasks
- Memory channels for inter-task communication

**Command Execution** (`src/flockwave/server/commands.py`):
- `CommandExecutionManager` tracks long-running drone commands
- Supports progress reporting and cancellation
- Commands can be synchronous, awaitable, or async generators

## Package Structure

```
src/flockwave/
├── server/          # Main Skybrush server
│   ├── app.py       # SkybrushServer application class
│   ├── launcher.py  # CLI entry point
│   ├── ext/         # Extension modules (48 built-in extensions)
│   ├── registries/  # Central registries for objects, clients, connections
│   ├── model/       # Data models (UAV, Client, World, Messages, etc.)
│   ├── command_handlers/  # Flockwave message handlers
│   ├── show/        # Drone show execution subsystem
│   ├── utils/       # Utility functions
│   └── middleware/  # Request/response middleware
├── gateway/         # Skybrush Gateway application
└── proxy/           # Skybrush Proxy application
```

## Development Notes

### Dependencies

- Uses `uv` for dependency management (not pip/poetry)
- Public dependencies on PyPI and public Fury index
- Private dependencies on `collmot` index (not required for community version)
- When modifying `pyproject.toml`, remove `collmot`/`pro` extras if you don't have access to private index

### Testing

- Test framework: pytest with pytest-trio
- Test location: `test/` directory
- Run with `trio_mode = 1` (configured in pyproject.toml)
- Tests use `pytest-datadir` for test data files

### Code Style

- Linter/formatter: Ruff (replaces Black, isort, flake8)
- Pre-commit hooks configured for: ruff, trailing whitespace, EOF fixer, uv-lock
- Ignored rules: B905, C901, E402, E501 (see pyproject.toml)

### Port Management

- Base port: 5000 (configurable via `PORT` env var or `-p` flag)
- Service ports are relative to base: HTTP=5000, TCP=5001, UDP=5001, etc.
- Port registry in `src/flockwave/server/ports.py`
- Extensions register ports via `use_port()` context manager

### Logging

- Uses `flockwave-logger` package
- Log styles: `fancy` (default), `plain`, `json`
- Configure via `--log-style` flag
- Extension loggers: `from .logger import log; log = log.getChild("ext_name")`
