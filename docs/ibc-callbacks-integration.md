# IBC Callbacks Middleware Integration Guide (v10+)

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Installation](#installation)
4. [Integration Steps](#integration-steps)
5. [Configuration](#configuration)
6. [Contract Integration](#contract-integration)
7. [Testing](#testing)
8. [Production Deployment](#production-deployment)
9. [Troubleshooting](#troubleshooting)
10. [Migration from IBC Hooks](#migration-from-ibc-hooks)
11. [API Reference](#api-reference)

## Overview

IBC Callbacks is a native middleware in ibc-go v10+ that provides standardized callback functionality for IBC packet lifecycle events. It enables secondary applications (smart contracts, modules) to register callbacks that execute during packet processing, enabling complex cross-chain composability logic.

### Key Features
- **Native Integration**: Built into ibc-go v10+ core, no external dependencies
- **Standardized Interface**: Consistent callback logic across all IBC applications
- **Gas Management**: Configurable gas limits with proper revert handling
- **Packet Lifecycle Coverage**: Callbacks for send, receive, acknowledgment, and timeout
- **Better PFM Integration**: Designed to work seamlessly with packet forward middleware

### Architecture

Callbacks middleware operates as a layer in the IBC stack:
1. Intercepts packet lifecycle events (send, receive, ack, timeout)
2. Checks for registered callbacks in packet metadata
3. Executes callbacks with configurable gas limits
4. Handles failures gracefully without breaking packet flow

### When to Use Callbacks vs IBC Hooks

**Use IBC Callbacks (v10+) when:**
- You're on ibc-go v10 or later
- You need standardized callback interfaces
- You want seamless packet forward middleware integration
- You need callbacks for various IBC applications (not just transfers)
- You want better gas management and error handling

**Use IBC Hooks when:**
- You're on ibc-go v7-v9
- You specifically need CosmWasm contract execution via memo
- You have existing IBC Hooks integrations
- You need the specific sender address transformation pattern

**IMPORTANT**: Callbacks and IBC Hooks are mutually exclusive in the transfer stack — do not combine them for the same base app.

## Prerequisites

### Version Requirements
- **IBC-go**: v10.0.0+ (callbacks are built into core)
- **Cosmos SDK**: v0.50.0+
- **Go**: 1.23+

### Dependencies
```go
require (
    github.com/cosmos/ibc-go/v10 v10.1.1
    github.com/cosmos/cosmos-sdk v0.50.13
    // Callbacks are included in ibc-go v10, no separate import needed
)
```

## Installation

### Step 1: Import Required Packages

In your `app/app.go`:
```go
import (
    // IBC Callbacks
    ibccallbacks "github.com/cosmos/ibc-go/v10/modules/apps/callbacks"
    ibccallbackstypes "github.com/cosmos/ibc-go/v10/modules/apps/callbacks/types"

    // Packet Forward Middleware (optional)
    packetforward "github.com/cosmos/ibc-apps/middleware/packet-forward-middleware/v10/packetforward"
    packetforwardkeeper "github.com/cosmos/ibc-apps/middleware/packet-forward-middleware/v10/packetforward/keeper"
    packetforwardtypes "github.com/cosmos/ibc-apps/middleware/packet-forward-middleware/v10/packetforward/types"

    // CosmWasm (optional)
    wasm "github.com/CosmWasm/wasmd/x/wasm"
    wasmkeeper "github.com/CosmWasm/wasmd/x/wasm/keeper"
)
```

## Integration Steps

### Step 1: Define Contract Keeper Interface

If using with CosmWasm or custom contract execution:

```go
// ContractKeeper defines the interface for executing contracts
type ContractKeeper interface {
    ExecuteContract(
        ctx sdk.Context,
        contractAddr sdk.AccAddress,
        caller sdk.AccAddress,
        msg []byte,
        coins sdk.Coins,
    ) ([]byte, error)
    
    // For sudo calls (privileged execution)
    Sudo(
        ctx sdk.Context,
        contractAddress sdk.AccAddress,
        msg []byte,
    ) ([]byte, error)
}
```

### Step 2: Create WasmHandler for Callbacks

```go
// Create a WasmHandler that implements the ContractKeeper interface
wasmHandler := wasm.NewIBCHandler(
    app.WasmKeeper,
    app.IBCKeeper.ChannelKeeper,
    app.IBCKeeper.ChannelKeeper,
)
```

### Step 3: Configure Middleware Stack with Callbacks

Callbacks wiring differs across ibc-go minor versions. Use the pattern that matches your ibc-go.

Option A — ibc-go v10.1.x (constructor with setters)

```go
maxCallbackGas := uint64(10_000_000)
wasmHandler := wasm.NewIBCHandler(app.WasmKeeper, app.IBCKeeper.ChannelKeeper, app.IBCKeeper.ChannelKeeper)

// 1) Construct callbacks middleware
cb := ibccallbacks.NewIBCMiddleware(wasmHandler, maxCallbackGas)

// 2) Set underlying application and ICS4 wrapper
cb.SetUnderlyingApplication(transfer.NewIBCModule(app.TransferKeeper))
cb.SetICS4Wrapper(app.PacketForwardKeeper) // or ChannelKeeper if PFM is not used

// 3) Build stack (bottom→top): transfer -> callbacks -> PFM -> (rate-limit)
var transferStack porttypes.IBCModule = cb
transferStack = packetforward.NewIBCMiddleware(
    transferStack,
    app.PacketForwardKeeper,
    0,
    packetforwardkeeper.DefaultForwardTransferPacketTimeoutTimestamp,
)

// 4) Ensure TransferKeeper uses callbacks as ICS4Wrapper
app.TransferKeeper.WithICS4Wrapper(cb)

// 5) Route
ibcRouter := porttypes.NewRouter()
ibcRouter.AddRoute(ibctransfertypes.ModuleName, transferStack)
app.IBCKeeper.SetRouter(ibcRouter)
```

Option B — ibc-go v10.3.x and IBC v2 (constructor takes app/wrappers)

```go
// See ibc-go modules/apps/callbacks/v2/ibc_middleware.go for the v2 constructor.
// Gaia uses this pattern in v10.3.x.
```

### Step 4: Alternative Stack Without Packet Forward

If not using packet forward middleware:

```go
maxCallbackGas := uint64(10_000_000)

var transferStack porttypes.IBCModule
transferStack = transfer.NewIBCModule(app.TransferKeeper)

cb := ibccallbacks.NewIBCMiddleware(wasmHandler, maxCallbackGas)
cb.SetUnderlyingApplication(transfer.NewIBCModule(app.TransferKeeper))
cb.SetICS4Wrapper(app.IBCKeeper.ChannelKeeper)
app.TransferKeeper.WithICS4Wrapper(cb)
ibcRouter.AddRoute(ibctransfertypes.ModuleName, cb)
```

## Configuration

### Callback Registration Format

Callbacks are registered via ICS‑20 memo JSON using keys defined in ibc-go.

Source (send‑side) callback:

```json
{
  "src_callback": {
    "address": "cosmos1contractaddress",
    "gas_limit": "500000"
  }
}
```

Destination (recv‑side) callback:

```json
{
  "dest_callback": {
    "address": "cosmos1contractaddress",
    "gas_limit": "600000"
  }
}
```

See: `../ibc-go/modules/apps/callbacks/types/keys.go` and `../ibc-go/modules/apps/callbacks/types/callbacks_test.go`.

### Callback Types

1. **Source Callbacks**: Execute on the source chain when sending packets
2. **Destination Callbacks**: Execute on the destination chain when receiving packets
3. **Acknowledgment Callbacks**: Execute when acknowledgment is received
4. **Timeout Callbacks**: Execute when packet times out

### Gas Management

```go
const (
    // Maximum gas allowed for any callback
    MaxCallbackGas = uint64(10_000_000)
    
    // Default gas if not specified
    DefaultCallbackGas = uint64(1_000_000)
    
    // Minimum gas required
    MinCallbackGas = uint64(100_000)
)

// Configure in middleware initialization (v10.1.x pattern)
cb := ibccallbacks.NewIBCMiddleware(contractKeeper, MaxCallbackGas)
cb.SetUnderlyingApplication(transfer.NewIBCModule(app.TransferKeeper))
cb.SetICS4Wrapper(ics4Wrapper)
```

## Contract Integration

### CosmWasm Contract Implementation

For contracts to receive callbacks:

```rust
use cosmwasm_std::{
    entry_point, Binary, Deps, DepsMut, Env, MessageInfo, Response, StdResult,
};

// Callback message types
#[cw_serde]
pub enum CallbackMsg {
    // Called when packet is sent
    SourceCallback {
        packet: IbcPacket,
        acknowledgement: Option<Binary>,
    },
    // Called when packet is received
    DestinationCallback {
        packet: IbcPacket,
    },
    // Called on acknowledgment
    AckCallback {
        packet: IbcPacket,
        acknowledgement: Binary,
    },
    // Called on timeout
    TimeoutCallback {
        packet: IbcPacket,
    },
}

// Entry point for callbacks (typically via sudo)
#[entry_point]
pub fn sudo(
    deps: DepsMut,
    env: Env,
    msg: CallbackMsg,
) -> StdResult<Response> {
    match msg {
        CallbackMsg::SourceCallback { packet, acknowledgement } => {
            handle_source_callback(deps, env, packet, acknowledgement)
        }
        CallbackMsg::DestinationCallback { packet } => {
            handle_dest_callback(deps, env, packet)
        }
        CallbackMsg::AckCallback { packet, acknowledgement } => {
            handle_ack_callback(deps, env, packet, acknowledgement)
        }
        CallbackMsg::TimeoutCallback { packet } => {
            handle_timeout_callback(deps, env, packet)
        }
    }
}

fn handle_ack_callback(
    deps: DepsMut,
    _env: Env,
    packet: IbcPacket,
    acknowledgement: Binary,
) -> StdResult<Response> {
    // Process acknowledgment
    let ack: AcknowledgementMsg = from_json(&acknowledgement)?;
    
    if ack.is_success() {
        // Handle successful transfer
        // e.g., update state, emit events
    } else {
        // Handle failed transfer
        // e.g., revert state, refund user
    }
    
    Ok(Response::new()
        .add_attribute("action", "ack_callback")
        .add_attribute("packet_sequence", packet.sequence.to_string()))
}
```

### Sending Transfers with Callbacks

```rust
use cosmwasm_std::{CosmosMsg, IbcMsg, IbcTimeout};

pub fn execute_transfer_with_callbacks(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    recipient: String,
    channel_id: String,
    amount: Coin,
) -> Result<Response, ContractError> {
    // Prepare callback metadata
    let callbacks = json!({
        "src_callback": {
            "address": env.contract.address.to_string(),
            "gas_limit": "500000"
        }
    });
    
    // Create IBC transfer with callbacks
    let msg = IbcMsg::Transfer {
        channel_id,
        to_address: recipient,
        amount,
        timeout: IbcTimeout::with_timestamp(env.block.time.plus_seconds(300)),
        memo: Some(callbacks.to_string()),
    };
    
    Ok(Response::new()
        .add_message(msg)
        .add_attribute("action", "transfer_with_callbacks"))
}
```

### Native Module Implementation

For Cosmos SDK modules to handle callbacks:

```go
// Implement the ContractKeeper interface
type ModuleCallbackHandler struct {
    keeper Keeper
}

func (h ModuleCallbackHandler) ExecuteContract(
    ctx sdk.Context,
    contractAddr sdk.AccAddress,
    caller sdk.AccAddress,
    msg []byte,
    coins sdk.Coins,
) ([]byte, error) {
    // Parse callback message
    var callbackMsg CallbackMessage
    if err := json.Unmarshal(msg, &callbackMsg); err != nil {
        return nil, err
    }
    
    // Handle callback according to your app's contract schema
    // ibc-go callbacks invoke ContractKeeper methods; the payload shape is app-defined
}
```

## Testing

### Unit Tests

```go
func TestCallbacksMiddleware(t *testing.T) {
    // Setup test environment
    app := simapp.Setup(t, false)
    ctx := app.BaseApp.NewContext(false, tmproto.Header{})
    
    // Create transfer with callbacks
    memo := `{"src_callback":{"address":"cosmos1testcontract","gas_limit":"500000"}}`
    
    transferMsg := transfertypes.NewMsgTransfer(
        "transfer",
        "channel-0",
        sdk.NewCoin("uatom", sdk.NewInt(1000)),
        senderAddr.String(),
        recipientAddr.String(),
        clienttypes.NewHeight(1, 100),
        0,
        memo,
    )
    
    // Execute transfer
    res, err := app.TransferKeeper.Transfer(ctx, transferMsg)
    require.NoError(t, err)
    
    // Simulate acknowledgment
    packet := channeltypes.NewPacket(
        transferData,
        res.Sequence,
        "transfer",
        "channel-0",
        "transfer",
        "channel-0",
        clienttypes.NewHeight(1, 110),
        0,
    )
    
    ack := channeltypes.NewResultAcknowledgement([]byte("success"))
    err = app.CallbacksMiddleware.OnAcknowledgementPacket(
        ctx,
        packet,
        ack.Acknowledgement(),
        relayerAddr,
    )
    require.NoError(t, err)
    
    // Verify callback was executed
    // Check state changes, events, etc.
}
```

### Integration Tests

Working test scenarios live in ibc-go:
- `../ibc-go/modules/apps/callbacks/ibc_middleware_test.go`
- `../ibc-go/modules/apps/callbacks/types/callbacks_test.go`

### Manual Testing

```bash
# Deploy a test contract with callback support
wasmd tx wasm store callback_contract.wasm --from validator

# Instantiate
wasmd tx wasm instantiate 1 '{}' --from validator --label "callback-test"

# Send transfer with callbacks (source callback)
wasmd tx ibc-transfer transfer transfer channel-0 \
    cosmos1recipient 1000uatom \
    --from validator \
    --memo '{"src_callback":{"address":"cosmos1contract","gas_limit":"500000"}}'

# Query contract state to verify callback execution
wasmd query wasm contract-state smart cosmos1contract '{"get_callback_count":{}}'
```

## Production Deployment

### Pre-deployment Checklist

1. **Gas Limits**: Configure appropriate maximum gas for callbacks
2. **Contract Validation**: Ensure callback contracts are audited
3. **Error Handling**: Implement proper error recovery in callbacks
4. **Monitoring**: Set up metrics for callback execution
5. **Rate Limiting**: Consider rate limiting callback executions

### Recommended Configuration

```go
// Production-ready configuration
const (
    // Conservative gas limits
    ProductionMaxCallbackGas = uint64(5_000_000)
    
    // Timeout for callback execution
    CallbackTimeout = 5 * time.Second
)

// Initialize with production parameters
cbStack := ibccallbacks.NewIBCMiddleware(
    transferStack,
    ics4Wrapper,
    contractKeeper,
    ProductionMaxCallbackGas,
)

// Add monitoring wrapper
cbStack = monitoring.WrapCallbacks(cbStack, metricsCollector)
```

### Monitoring and Alerts

Key metrics to monitor:
- `ibc_callbacks_executed_total`: Counter of callback executions
- `ibc_callbacks_failed_total`: Counter of failed callbacks
- `ibc_callbacks_gas_used`: Histogram of gas consumption
- `ibc_callbacks_duration_seconds`: Histogram of execution time

Example Prometheus alerts:
```yaml
groups:
- name: ibc_callbacks
  rules:
  - alert: HighCallbackFailureRate
    expr: rate(ibc_callbacks_failed_total[5m]) > 0.1
    for: 5m
    annotations:
      summary: "High callback failure rate: {{ $value }}"
      
  - alert: CallbackGasExhaustion
    expr: ibc_callbacks_gas_used > 4500000
    for: 1m
    annotations:
      summary: "Callback approaching gas limit"
      
  - alert: CallbackTimeout
    expr: ibc_callbacks_duration_seconds > 4
    annotations:
      summary: "Callback execution taking too long"
```

## Troubleshooting

### Common Issues and Solutions

#### Issue: Callback not executing
**Diagnosis**: Check callback registration format
```bash
# Verify memo format
echo '{"src_callback":{"address":"cosmos1...","gas_limit":"500000"}}' | jq .

# Check if contract exists and has sudo endpoint
wasmd query wasm contract cosmos1contract
```

#### Issue: "insufficient gas" error
**Solution**: Increase gas limit or optimize callback code
```go
// Increase gas limit in memo
{"src_callback":{"address":"cosmos1contract","gas_limit":"1000000"}}
```

#### Issue: Callback fails silently
**Solution**: Check event logs for error details
```bash
# Query transaction events
wasmd query tx <tx_hash> --output json | jq '.logs[].events[] | select(.type=="ibc_callback")'
```

#### Issue: Incompatibility with packet forward
**Solution**: Ensure correct middleware ordering
```go
// Correct order:
// 1. Transfer (base)
// 2. Callbacks (wraps transfer)
// 3. Packet Forward (wraps callbacks)
// 4. Rate Limiting (optional, outermost)
```

### Debug Logging

Enable detailed logging:
```toml
# In app.toml
[log]
level = "debug"
filter = "callbacks:debug,x/ibc:debug,*:info"
```

## Migration from IBC Hooks

### Key Differences

| Feature | IBC Hooks | Callbacks |
|---------|-----------|-----------|
| IBC-go Version | v7-v9 | v10+ |
| Integration | External module | Native to ibc-go |
| Memo Format | `{"wasm":...}` | `{"callbacks":...}` |
| Gas Management | Basic | Advanced with limits |
| PFM Compatibility | Issues | Seamless |
| Contract Interface | Execute + Sudo | Primarily Sudo |

### Migration Steps

1. **Update Dependencies**:
```go
// Remove
- ibchooks "github.com/cosmos/ibc-apps/modules/ibc-hooks/v9"

// Use native callbacks in ibc-go v10
+ ibccallbacks "github.com/cosmos/ibc-go/v10/modules/apps/callbacks"
```

2. **Update Middleware Stack**:
```go
// Old (IBC Hooks)
- transferStack = ibchooks.NewIBCMiddleware(transferStack, &app.HooksICS4Wrapper)

// New (Callbacks)
+ cbStack := ibccallbacks.NewIBCMiddleware(
+     transferStack,
+     app.PacketForwardKeeper,
+     wasmHandler,
+     maxCallbackGas,
+ )
```

3. **Update Contract Code**:
```rust
// Old memo format
- {"wasm": {"contract": "...", "msg": {...}}}

// New callback format
+ {"src_callback": {"address": "...", "gas_limit": "500000"}}
```

4. **Update Contract Handlers**:
```rust
// Implement new callback message types
pub enum CallbackMsg {
    AckCallback { packet: IbcPacket, acknowledgement: Binary },
    TimeoutCallback { packet: IbcPacket },
    // ... other callback types
}
```

## API Reference

See ibc-go source for exact signatures and version variants:
- v10.1.x: `modules/apps/callbacks/ibc_middleware.go`
- v10.3.x (IBCv2): `modules/apps/callbacks/v2/ibc_middleware.go`

### Contract Keeper Interface

```go
type ContractKeeper interface {
    // Execute contract with callbacks
    ExecuteContract(
        ctx sdk.Context,
        contractAddr sdk.AccAddress,
        caller sdk.AccAddress,
        msg []byte,
        coins sdk.Coins,
    ) ([]byte, error)
    
    // Sudo for privileged execution
    Sudo(
        ctx sdk.Context,
        contractAddress sdk.AccAddress,
        msg []byte,
    ) ([]byte, error)
}
```

### Events

```go
// Event types (see ibc-go callbacks types/events.go)
const (
    EventTypeSourceCallback      = "ibc_src_callback"
    EventTypeDestinationCallback = "ibc_dest_callback"

    AttributeKeyCallbackType                    = "callback_type"   // acknowledgement | timeout | recv_packet
    AttributeKeyCallbackAddress                 = "callback_address"
    AttributeKeyCallbackResult                  = "callback_result" // success | failure
    AttributeKeyCallbackError                   = "callback_error"
    AttributeKeyCallbackGasLimit                = "callback_exec_gas_limit"
    AttributeKeyCallbackCommitGasLimit          = "callback_commit_gas_limit"
    AttributeKeyCallbackSourcePortID            = "packet_src_port"
    AttributeKeyCallbackSourceChannelID         = "packet_src_channel"
    AttributeKeyCallbackDestPortID              = "packet_dest_port"
    AttributeKeyCallbackDestChannelID           = "packet_dest_channel"
    AttributeKeyCallbackSequence                = "packet_sequence"
    AttributeKeyCallbackBaseApplicationVersion  = "callback_base_application_version"
)
```

## Production Examples

### Neutron Chain Configuration

From Neutron's production implementation:

```go
// Neutron uses callbacks with packet forward and rate limiting
var transferStack porttypes.IBCModule

// Base transfer with sudo capabilities
transferStack = transferSudo.NewIBCModule(
    app.TransferKeeper,
    contractmanager.NewSudoLimitWrapper(app.ContractManagerKeeper, &app.WasmKeeper),
)

```

## Additional Resources

- [IBC-go v10 Callbacks Documentation](https://ibc.cosmos.network/v10/ibc/apps/callbacks)
- [ADR-008: Callback to IBC Actors](https://github.com/cosmos/ibc-go/blob/main/docs/architecture/adr-008-app-caller-cbs.md)
- [IBC-go Repository](https://github.com/cosmos/ibc-go/tree/main/modules/apps/callbacks)
- [Migration Guide from v9 to v10](https://ibc.cosmos.network/v10/migrations/v9-to-v10)

## Version Compatibility Matrix

| Callbacks Version | IBC-go Version | Cosmos SDK | Status |
|------------------|----------------|------------|---------|
| N/A              | v7-v9          | v0.47.x    | Use IBC Hooks instead |
| Native           | v10.x          | v0.50.x    | Current |

## Support

For issues and questions:
- GitHub Issues: [ibc-go/issues](https://github.com/cosmos/ibc-go/issues)
- Discord: Cosmos Network Discord #ibc channel
- Documentation: [ibc.cosmos.network](https://ibc.cosmos.network)
