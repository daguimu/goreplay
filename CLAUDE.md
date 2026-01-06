# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GoReplay (Gor) is a network monitoring tool that captures and replays live HTTP traffic. It uses raw sockets and libpcap for packet capture, allowing it to record production traffic without acting as a proxy. The captured traffic can be replayed to staging/test environments, saved to files, or forwarded to various outputs (HTTP, Kafka, S3, etc.).

## Development Commands

### Building

```bash
# Build locally (requires Go 1.21+)
go build -o gor ./cmd/gor/

# Build with vendor dependencies
make build

# Build using Docker containers (recommended for cross-platform)
make build-env              # Build both AMD64 and ARM64 Docker environments
make release-bin-linux-amd64
make release-bin-linux-arm64
make release-bin-mac-amd64
make release-bin-mac-arm64
```

### Testing

```bash
# Run all tests in current package (uses Docker)
make test

# Run all tests including subdirectories
make test_all

# Run a single test by name
make testone TEST=TestEmitterFiltered

# Run with race detector
make race

# Generate coverage report
make cover

# Run benchmarks
make bench BENCHMARK=BenchmarkRAWInput
```

### Running

```bash
# Basic traffic capture and stdout output
sudo ./gor --input-raw :8000 --output-stdout

# Capture and replay to staging
sudo ./gor --input-raw :8000 --output-http http://staging.env

# Using Make targets for development
make run-arg ARGS="--input-raw :8080 --output-stdout"
```

Note: Raw packet capture requires sudo/root privileges due to libpcap requirements.

## Architecture

### Plugin System

The core architecture is built around a plugin-based input/output system:

- **Plugins** (`plugins.go`): All I/O is handled through plugins implementing `PluginReader` and/or `PluginWriter` interfaces
- **Inputs**: Raw packet capture, file, TCP, HTTP, Kafka, dummy (for testing)
- **Outputs**: HTTP, file, TCP, WebSocket, Kafka, S3, stdout, null
- **Emitter** (`emitter.go`): Orchestrates data flow from inputs to outputs using `CopyMulty()` which reads from one input and writes to multiple outputs

### Message Flow

1. **Input plugins** capture/read traffic and return `*Message` structs containing metadata and payload
2. **Emitter** reads messages via `PluginRead()` and processes them through:
   - HTTP modifier for filtering/rewriting requests
   - Optional middleware (external process for custom transformations)
   - Prettification (decoding gzip/chunked encoding)
3. **Output plugins** receive messages via `PluginWrite()` and handle delivery

### Packet Capture (input_raw.go)

The `RAWInput` plugin is the most complex component:

- Uses `internal/capture` package for low-level packet capture via libpcap or raw sockets
- Assembles TCP streams from individual packets (`internal/tcp`)
- Extracts HTTP messages from TCP payloads
- Supports VXLAN, VLAN, BPF filtering, and various capture engines

### Settings System

All configuration is centralized in `settings.go`:

- Uses Go's `flag` package for CLI argument parsing
- `AppSettings` struct holds all configuration
- Global `Settings` variable accessed throughout codebase
- Supports multi-value flags via `MultiOption` for specifying multiple inputs/outputs

### HTTP Modification

The `http_modifier.go` component handles request filtering and rewriting:

- URL/header/method filtering with regex support
- URL and header rewriting
- Request parameter manipulation
- Rate limiting based on header/parameter hashing

## Key Files

- `cmd/gor/gor.go` - Main entry point, initializes plugins and starts emitter
- `plugins.go` - Plugin registration and initialization
- `emitter.go` - Core message routing between inputs and outputs
- `settings.go` - Configuration management and CLI flags
- `input_raw.go` - Raw packet capture implementation
- `output_http.go` - HTTP replay with worker pool and dynamic scaling
- `internal/capture/` - Low-level packet capture abstraction
- `internal/tcp/` - TCP stream reassembly

## Testing Notes

- Tests use Docker containers by default (see Makefile)
- Many tests create temporary files and use parallel execution
- The `TestDummyInput` and `TestDummyOutput` are useful for testing data flow
- RAW input tests may require special network setup or root privileges

## Dependencies

- `libpcap` - Required for packet capture (installed on host or in Docker)
- `google/gopacket` - Go wrapper for libpcap
- Vendored dependencies in `/vendor` (created with `go mod vendor`)

## Common Patterns

- **Error handling**: Uses `Debug()` function with verbosity levels for debugging
- **Configuration**: All plugins receive config structs passed from `Settings`
- **Concurrency**: Heavy use of goroutines; emitter starts one per input
- **Interfaces**: Plugin behavior defined by `PluginReader`/`PluginWriter` interfaces
- **Reflection**: Plugin registration uses reflection to call constructors dynamically
