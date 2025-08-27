# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository contains IBC (Inter-Blockchain Communication) applications and middleware for Cosmos SDK blockchains. It serves as a central location for IBC modules and middleware that can be integrated into Cosmos SDK chains.

## Architecture

The repository is organized into two main categories:

### Modules

- **async-icq**: Interchain Queries enable blockchains to query the state of an account on another chain
- **ibc-hooks**: IBC middleware that enables ICS-20 token transfers to initiate contract calls
- **rate-limiting**: Module for rate limiting ingress/egress of packets

### Middleware

- **packet-forward-middleware**: Middleware for forwarding IBC packets between chains

### CosmWasm Contracts

- **polytone**: CosmWasm contracts for cross-chain account abstraction (located in `cosmwasm/polytone/`)

## Development Commands

### Go Modules (rate-limiting, async-icq, ibc-hooks, packet-forward-middleware)

Each module has its own Makefile with these common commands:

```bash
# Build
make build           # Build the module
make build-linux     # Build for Linux

# Testing
make test-unit       # Run unit tests
make test           # Run all tests
make test-race      # Run tests with race detector

# Linting
make lint           # Run golangci-lint
make lint-fix       # Auto-fix linting issues

# Protobuf
make proto-all      # Format, lint, and generate protobuf files
make proto-gen      # Generate protobuf files
make proto-format   # Format proto files
make proto-lint     # Lint proto files

# E2E Tests (from module directory)
make local-image    # Build Docker image for testing
make ictest-*       # Run specific interchain tests
```

### CosmWasm Contracts (polytone)

From `cosmwasm/polytone/` directory, use `just` commands:

```bash
just build          # Build contracts
just test           # Run tests
just optimize       # Optimize contracts for deployment
just simtest        # Run simulation tests
just integrationtest # Run integration tests
just schema         # Generate JSON schemas
```

## Testing

### Unit Tests

Each module can be tested independently:

```bash
cd modules/rate-limiting && make test-unit
cd modules/async-icq && make test-unit
cd middleware/packet-forward-middleware && make test-unit
```

### E2E Interchain Tests

E2E tests use Docker images and simulate multi-chain environments:

```bash
# Rate limiting
cd modules/rate-limiting && make ictest-ratelimit

# Packet forward middleware
cd middleware/packet-forward-middleware && make ictest-forward
```

## Code Standards

- Go version: 1.23.6
- IBC-Go version: v10.1.1
- Cosmos SDK version: v0.50.13
- Linter: golangci-lint with custom configuration (see `.golangci.yml`)
- Import ordering: standard library, external deps, cosmossdk.io, cosmos-sdk, cometbft, ibc-go

## Module-Specific Notes

### Rate Limiting Module

- Located in `modules/rate-limiting/`
- Implements packet flow control with quotas
- Has BeginBlock handler for epoch management
- Supports whitelisting and blacklisting of channels

### Async ICQ Module

- Located in `modules/async-icq/`
- Allows querying other chains' state without ICA
- Includes demo app in `testing/demo-simapp/`
- Supports migrations (currently v2)

### IBC Hooks Module

- Located in `modules/ibc-hooks/`
- Wraps ICS-20 transfers with contract execution
- Includes test contracts in `tests/unit/testdata/`
- Provides wasm hooks functionality

### Packet Forward Middleware

- Located in `middleware/packet-forward-middleware/`
- Enables multi-hop packet forwarding
- Has comprehensive e2e test suite
- Supports non-refundable packets

## Protobuf Generation

All modules use Docker-based protobuf generation:

```bash
# From module directory
make proto-all
```

Proto files are located in `proto/` subdirectories within each module.

## Workspace Setup

For multi-module development, copy `go.work.example` to `go.work` and modify as needed:

```bash
cp go.work.example go.work
```

## Common Gotchas

1. Each module has its own go.mod - ensure you're in the correct directory when running commands
2. E2E tests require Docker and can take significant time (15-60 minutes)
3. Some modules depend on specific IBC-Go versions - check go.mod before upgrading
4. Proto generation requires Docker with specific builder images
5. When working with rate limiting, ensure proper BeginBlock interface implementation
