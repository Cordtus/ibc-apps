# IBC Hooks Middleware Integration Guide

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Installation](#installation)
4. [Integration Steps](#integration-steps)
5. [Configuration](#configuration)
6. [CosmWasm Contract Integration](#cosmwasm-contract-integration)
7. [Testing](#testing)
8. [Production Deployment](#production-deployment)
9. [Troubleshooting](#troubleshooting)
10. [Security Considerations](#security-considerations)
11. [API Reference](#api-reference)

## Overview

IBC Hooks is an IBC middleware that enables ICS-20 token transfers to trigger smart contract executions. This powerful primitive enables cross-chain composability by allowing tokens to be sent along with instructions for their use on the destination chain.

### Key Features
- **Contract Execution on Transfer**: Automatically execute CosmWasm contracts when receiving IBC transfers
- **Memo-Based Routing**: Uses the ICS-20 memo field to specify contract calls
- **Ack Callbacks**: Contracts can register for acknowledgment callbacks on sent transfers
- **Timeout Handling**: Proper handling of timeout scenarios with callbacks
- **Sender Address Transformation**: Secure address derivation to prevent impersonation

### Architecture

IBC Hooks operates as middleware in the IBC stack:
1. Intercepts incoming ICS-20 transfers
2. Parses the memo field for contract execution instructions
3. Executes the specified contract with the transferred tokens
4. Provides callbacks for acknowledgments and timeouts

### When to Use IBC Hooks vs Callbacks

**Use IBC Hooks when:**
- You need to execute CosmWasm contracts on IBC transfers
- Your chain uses CosmWasm for smart contracts
- You want simple memo-based contract triggering
- You need acknowledgment callbacks for sent transfers

**Use IBC Callbacks (v10+) when:**
- You're on ibc-go v10 or later
- You need a more standardized callback interface
- You want better integration with packet forward middleware
- You need more sophisticated callback handling

## Prerequisites

### Version Requirements
- **IBC-go**: v7.0.0+ (v8.0.0+ recommended, v10.0.0+ for callbacks alternative)
- **Cosmos SDK**: v0.47.0+ (v0.50.0+ recommended)
- **CosmWasm**: wasmd v0.40.0+ (for contract execution)
- **Go**: 1.21+ (1.23+ recommended)

### Dependencies
```go
require (
    github.com/cosmos/ibc-apps/modules/ibc-hooks/v10 v10.0.0
    github.com/cosmos/ibc-go/v10 v10.1.1
    github.com/cosmos/cosmos-sdk v0.50.13
    github.com/CosmWasm/wasmd v0.40.0
)
```

## Installation

### Step 1: Add Module Dependency

Add to your `go.mod`:
```bash
go get github.com/cosmos/ibc-apps/modules/ibc-hooks/v10@latest
```

### Step 2: Import Required Packages

In your `app/app.go`:
```go
import (
    ibchooks "github.com/cosmos/ibc-apps/modules/ibc-hooks/v10"
    ibchookskeeper "github.com/cosmos/ibc-apps/modules/ibc-hooks/v10/keeper"
    ibchookstypes "github.com/cosmos/ibc-apps/modules/ibc-hooks/v10/types"
    
    // If using wasmd
    wasmkeeper "github.com/CosmWasm/wasmd/x/wasm/keeper"
)
```

## Integration Steps

### Step 1: Add Keeper to App Structure

```go
type App struct {
    *baseapp.BaseApp
    // ... other keepers
    
    TransferKeeper    ibctransferkeeper.Keeper
    WasmKeeper        wasm.Keeper
    IBCHooksKeeper    ibchookskeeper.Keeper    // Add this
    ContractKeeper    wasmtypes.ContractOpsKeeper // Add this
    Ics20WasmHooks    *ibchooks.WasmHooks        // Add this
    HooksICS4Wrapper  ibchooks.ICS4Middleware     // Add this
    
    // ... rest of app
}
```

### Step 2: Add Store Keys

```go
// In NewApp function
keys := storetypes.NewKVStoreKeys(
    // ... other store keys
    ibchookstypes.StoreKey,  // Add this
)
```

### Step 3: Initialize the Keepers

```go
// Initialize IBC Hooks Keeper (before transfer keeper)
app.IBCHooksKeeper = ibchookskeeper.NewKeeper(
    keys[ibchookstypes.StoreKey],
)

// Initialize Wasm Hooks handler
app.Ics20WasmHooks = ibchooks.NewWasmHooks(
    &app.IBCHooksKeeper, 
    nil,  // Contract keeper will be set later
    app.AccountKeeper.GetModuleAddress(govtypes.ModuleName).String(), // authority
)

// Initialize your Wasm Keeper
app.WasmKeeper = wasmkeeper.NewKeeper(
    appCodec,
    keys[wasm.StoreKey],
    app.AccountKeeper,
    app.BankKeeper,
    app.StakingKeeper,
    distrkeeper.NewQuerier(app.DistrKeeper),
    app.IBCKeeper.ChannelKeeper,
    &app.IBCKeeper.PortKeeper,
    scopedWasmKeeper,
    app.TransferKeeper,
    app.MsgServiceRouter(),
    app.GRPCQueryRouter(),
    wasmDir,
    wasmConfig,
    wasmCapabilities,
    authtypes.NewModuleAddress(govtypes.ModuleName).String(),
)

// Set up contract keeper for hooks
app.ContractKeeper = wasmkeeper.NewDefaultPermissionKeeper(app.WasmKeeper)
app.Ics20WasmHooks.ContractKeeper = app.ContractKeeper

// Create ICS4 Wrapper for hooks
app.HooksICS4Wrapper = ibchooks.NewICS4Middleware(
    app.IBCKeeper.ChannelKeeper,
    app.Ics20WasmHooks,
)
```

### Step 4: Configure Middleware Stack

**IMPORTANT**: IBC Hooks is generally incompatible with Callbacks middleware (v10+). Choose one or the other.

```go
// Build the transfer stack with IBC hooks
var transferStack porttypes.IBCModule

// 1. Base transfer module
transferStack = transfer.NewIBCModule(app.TransferKeeper)

// 2. IBC Hooks middleware (wraps transfer)
transferStack = ibchooks.NewIBCMiddleware(transferStack, &app.HooksICS4Wrapper)

// 3. Optional: Packet Forward Middleware (wraps hooks)
if app.PacketForwardEnabled {
    transferStack = packetforward.NewIBCMiddleware(
        transferStack,
        app.PacketForwardKeeper,
        0,
        packetforwardkeeper.DefaultForwardTransferPacketTimeoutTimestamp,
    )
}

// 4. Optional: Rate limiting as outermost layer
if app.RateLimitingEnabled {
    transferStack = ratelimit.NewIBCMiddleware(app.RatelimitKeeper, transferStack)
}

// 5. Update Transfer Keeper's ICS4Wrapper if using packet forward
if app.PacketForwardEnabled {
    app.TransferKeeper.WithICS4Wrapper(app.HooksICS4Wrapper)
}

// 6. Register with IBC router
ibcRouter := porttypes.NewRouter()
ibcRouter.AddRoute(ibctransfertypes.ModuleName, transferStack)
app.IBCKeeper.SetRouter(ibcRouter)
```

### Step 5: Register Module

```go
// In module manager
app.ModuleManager = module.NewManager(
    // ... other modules
    ibchooks.AppModuleBasic{},
)

// Set module order for Begin/EndBlock
app.ModuleManager.SetOrderBeginBlockers(
    // ... other modules
    ibchookstypes.ModuleName,
)

app.ModuleManager.SetOrderEndBlockers(
    // ... other modules  
    ibchookstypes.ModuleName,
)

// Add to InitGenesis order
genesisModuleOrder := []string{
    // ... other modules
    ibchookstypes.ModuleName,
}
```

## Configuration

### Memo Format for Contract Execution

The memo field in ICS-20 transfers must be formatted as JSON:

```json
{
    "wasm": {
        "contract": "cosmos1contractaddress",
        "msg": {
            "your_contract_method": {
                "param1": "value1",
                "param2": "value2"
            }
        }
    }
}
```

### Complete Transfer Example

```json
{
    "denom": "uatom",
    "amount": "1000000",
    "sender": "cosmos1sender...",
    "receiver": "cosmos1contractaddress",
    "memo": {
        "wasm": {
            "contract": "cosmos1contractaddress",
            "msg": {
                "swap": {
                    "output_denom": "uosmo",
                    "slippage_percentage": "1"
                }
            }
        }
    }
}
```

### Callback Registration

For acknowledgment callbacks, include in the memo:

```json
{
    "ibc_callback": "cosmos1contractaddress"
}
```

### Sender Address Derivation

The sender address is transformed to prevent impersonation:
```go
senderBech32 = Bech32(Hash("ibc-wasm-hook-intermediary" || channelID || originalSender))
```

## CosmWasm Contract Integration

### Receiving IBC Transfers with Hooks

Your contract should handle the execute message:

```rust
#[entry_point]
pub fn execute(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: ExecuteMsg,
) -> Result<Response, ContractError> {
    match msg {
        ExecuteMsg::Swap { output_denom, slippage_percentage } => {
            // info.sender will be the transformed IBC sender
            // info.funds will contain the transferred tokens
            execute_swap(deps, env, info, output_denom, slippage_percentage)
        }
        // ... other messages
    }
}
```

### Implementing Callback Handlers

For acknowledgment and timeout callbacks:

```rust
#[cw_serde]
pub enum SudoMsg {
    #[serde(rename = "ibc_lifecycle_complete")]
    IBCLifecycleComplete(IBCLifecycleComplete),
}

#[cw_serde]
pub enum IBCLifecycleComplete {
    #[serde(rename = "ibc_ack")]
    IBCAck {
        channel: String,
        sequence: u64,
        ack: String,
        success: bool,
    },
    #[serde(rename = "ibc_timeout")]
    IBCTimeout {
        channel: String,
        sequence: u64,
    },
}

#[entry_point]
pub fn sudo(deps: DepsMut, env: Env, msg: SudoMsg) -> Result<Response, ContractError> {
    match msg {
        SudoMsg::IBCLifecycleComplete(lifecycle) => match lifecycle {
            IBCLifecycleComplete::IBCAck { channel, sequence, ack, success } => {
                handle_ack(deps, env, channel, sequence, ack, success)
            }
            IBCLifecycleComplete::IBCTimeout { channel, sequence } => {
                handle_timeout(deps, env, channel, sequence)
            }
        }
    }
}
```

### Sending IBC Transfers with Callbacks

```rust
pub fn execute_transfer_with_callback(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    channel_id: String,
    to_address: String,
    amount: Coin,
) -> Result<Response, ContractError> {
    let memo = json!({
        "ibc_callback": env.contract.address.to_string()
    }).to_string();

    let transfer_msg = IbcMsg::Transfer {
        channel_id,
        to_address,
        amount,
        timeout: IbcTimeout::with_timestamp(env.block.time.plus_seconds(300)),
        memo: Some(memo),
    };

    Ok(Response::new()
        .add_message(transfer_msg)
        .add_attribute("action", "transfer_with_callback"))
}
```

## Testing

### Unit Tests

```go
func TestIBCHooksMiddleware(t *testing.T) {
    // Setup test app
    app := simapp.Setup(t, false)
    ctx := app.BaseApp.NewContext(false, tmproto.Header{})
    
    // Create transfer with wasm memo
    memo := fmt.Sprintf(`{
        "wasm": {
            "contract": "%s",
            "msg": {
                "test_hook": {}
            }
        }
    }`, contractAddr)
    
    transferMsg := transfertypes.NewMsgTransfer(
        "transfer",
        "channel-0",
        sdk.NewCoin("uatom", sdk.NewInt(1000)),
        senderAddr.String(),
        contractAddr.String(),
        clienttypes.NewHeight(1, 100),
        0,
        memo,
    )
    
    // Execute transfer
    _, err := app.TransferKeeper.Transfer(ctx, transferMsg)
    require.NoError(t, err)
    
    // Verify contract was called
    // ... verification logic
}
```

### Integration Tests

```bash
# Run unit tests
cd modules/ibc-hooks
make test-unit

# Test with local CosmWasm contracts
cd tests/unit/testdata/counter
cargo build --release --target wasm32-unknown-unknown
cargo test
```

### Manual Testing

```bash
# Deploy a test contract
wasmd tx wasm store counter.wasm --from validator --chain-id test

# Instantiate contract
wasmd tx wasm instantiate 1 '{}' --from validator --label "test"

# Send IBC transfer with hook
wasmd tx ibc-transfer transfer transfer channel-0 \
    cosmos1contractaddr 1000uatom \
    --from validator \
    --memo '{"wasm":{"contract":"cosmos1contractaddr","msg":{"increment":{}}}}'

# Query contract state to verify execution
wasmd query wasm contract-state smart cosmos1contractaddr '{"get_count":{}}'
```

## Production Deployment

### Pre-deployment Checklist

1. **Security Audit**: Ensure all contracts that will be called via hooks are audited
2. **Gas Limits**: Configure appropriate gas limits for contract execution
3. **Whitelist Contracts**: Consider implementing contract whitelist for hooks
4. **Rate Limiting**: Implement rate limiting on hook executions
5. **Monitoring**: Set up monitoring for hook execution failures

### Recommended Configuration

```go
const (
    // Maximum gas for hook execution
    MaxHookGas = uint64(1_000_000)
    
    // Timeout for hook execution
    HookTimeout = 10 * time.Second
)

// Initialize with safety parameters
app.Ics20WasmHooks = ibchooks.NewWasmHooks(
    &app.IBCHooksKeeper,
    app.ContractKeeper,
    app.AccountKeeper.GetModuleAddress(govtypes.ModuleName).String(),
).WithGasLimit(MaxHookGas)
```

### Monitoring and Alerts

Key metrics to monitor:
- `ibc_hooks_executed_total`: Counter of hook executions
- `ibc_hooks_failed_total`: Counter of failed hook executions
- `ibc_hooks_gas_used`: Histogram of gas usage
- `ibc_hooks_execution_time`: Histogram of execution times

Example Prometheus alerts:
```yaml
groups:
- name: ibc_hooks
  rules:
  - alert: HighHookFailureRate
    expr: rate(ibc_hooks_failed_total[5m]) > 0.1
    annotations:
      summary: "High IBC hooks failure rate"
      
  - alert: HookGasExhaustion
    expr: ibc_hooks_gas_used > 900000
    annotations:
      summary: "IBC hook approaching gas limit"
```

## Troubleshooting

### Common Issues and Solutions

#### Issue: Hook not executing
**Diagnosis**: Check memo format and contract address
```bash
# Verify memo is valid JSON
echo '{"wasm":{"contract":"cosmos1...","msg":{}}}' | jq .

# Check contract exists
wasmd query wasm contract cosmos1contractaddr
```

#### Issue: "unauthorized" error
**Solution**: Verify sender transformation is correct:
```go
expectedSender := DeriveIntermediateSender(channelID, originalSender)
```

#### Issue: Callback not received
**Solution**: Ensure contract implements sudo entry point:
```rust
#[entry_point]
pub fn sudo(deps: DepsMut, env: Env, msg: SudoMsg) -> Result<Response, ContractError> {
    // Implementation required
}
```

#### Issue: Gas exhaustion in hooks
**Solution**: Optimize contract code or increase gas limit:
```go
app.Ics20WasmHooks.WithGasLimit(2_000_000)
```

### Debug Logging

Enable debug logging:
```toml
# In app.toml
[log]
level = "debug"
filter = "ibc_hooks:debug,wasm:debug,*:info"
```

## Security Considerations

### Critical Security Notes

1. **Sender Address Trust**: The sender address is transformed and CANNOT be trusted as the original sender, especially when packet forward middleware is in use. There's a known vulnerability where PFM can allow sender spoofing.

2. **Reentrancy Protection**: Contracts called via hooks must implement reentrancy guards:
```rust
pub fn execute_hook(deps: DepsMut, env: Env, info: MessageInfo) -> Result<Response, ContractError> {
    // Check reentrancy guard
    if PROCESSING.load(deps.storage)? {
        return Err(ContractError::Reentrancy {});
    }
    PROCESSING.save(deps.storage, &true)?;
    
    // Process hook
    let result = process_hook_internal(deps, env, info)?;
    
    // Clear reentrancy guard
    PROCESSING.save(deps.storage, &false)?;
    Ok(result)
}
```

3. **Contract Validation**: Validate all inputs from IBC hooks:
```rust
// Validate amounts
if info.funds.is_empty() {
    return Err(ContractError::NoFunds {});
}

// Validate addresses
deps.api.addr_validate(&recipient)?;
```

4. **Gas Limits**: Always enforce gas limits to prevent DoS:
```go
const MaxHookGas = 1_000_000
```

### Best Practices

1. **Whitelist Contracts**: Only allow hooks to call pre-approved contracts
2. **Rate Limiting**: Implement rate limiting per sender/channel
3. **Audit Trail**: Log all hook executions for audit purposes
4. **Fail Safely**: Ensure failed hooks don't block IBC transfers
5. **Version Pinning**: Pin specific versions of contracts that can be called

## API Reference

### Keeper Methods

```go
// Core IBC Hooks functions
func (k Keeper) OnRecvPacket(ctx sdk.Context, packet channeltypes.Packet, relayer sdk.AccAddress) exported.Acknowledgement
func (k Keeper) OnAcknowledgementPacket(ctx sdk.Context, packet channeltypes.Packet, acknowledgement []byte, relayer sdk.AccAddress) error
func (k Keeper) OnTimeoutPacket(ctx sdk.Context, packet channeltypes.Packet, relayer sdk.AccAddress) error

// Callback management
func (k Keeper) StorePacketCallback(ctx sdk.Context, channel string, sequence uint64, contract string)
func (k Keeper) GetPacketCallback(ctx sdk.Context, channel string, sequence uint64) (string, bool)
func (k Keeper) DeletePacketCallback(ctx sdk.Context, channel string, sequence uint64)
```

### Wasm Hooks Methods

```go
// Hook execution
func (h WasmHooks) OnRecvPacketOverride(im IBCMiddleware, ctx sdk.Context, packet channeltypes.Packet, relayer sdk.AccAddress) exported.Acknowledgement

// Callback handling
func (h WasmHooks) OnAcknowledgementPacketOverride(im IBCMiddleware, ctx sdk.Context, packet channeltypes.Packet, acknowledgement []byte, relayer sdk.AccAddress) error
func (h WasmHooks) OnTimeoutPacketOverride(im IBCMiddleware, ctx sdk.Context, packet channeltypes.Packet, relayer sdk.AccAddress) error

// Helper functions
func DeriveIntermediateSender(channel, originalSender, bech32Prefix string) (string, error)
func IsValidMemo(memo string) (isValid bool, err error)
```

### Events

```go
// Emitted events
const (
    EventTypeHookExecuted = "ibc_hook_executed"
    EventTypeHookFailed = "ibc_hook_failed"
    EventTypeCallbackRegistered = "ibc_callback_registered"
    EventTypeCallbackExecuted = "ibc_callback_executed"
    
    AttributeKeyContract = "contract"
    AttributeKeyChannel = "channel"
    AttributeKeySequence = "sequence"
    AttributeKeySuccess = "success"
    AttributeKeyError = "error"
    AttributeKeyGasUsed = "gas_used"
)
```

## Additional Resources

- [IBC Hooks Module Repository](https://github.com/cosmos/ibc-apps/tree/main/modules/ibc-hooks)
- [Osmosis IBC Hooks (Original)](https://github.com/osmosis-labs/osmosis/tree/main/x/ibc-hooks)
- [CosmWasm Documentation](https://docs.cosmwasm.com)
- [IBC-go v10 Callbacks Alternative](https://ibc.cosmos.network/v10/ibc/apps/callbacks)

## Version Compatibility Matrix

| IBC Hooks Version | IBC-go Version | Cosmos SDK | CosmWasm |
|------------------|----------------|------------|----------|
| v7.x             | v7.x           | v0.47.x    | v0.40.x  |
| v8.x             | v8.x           | v0.47.x    | v0.40.x  |
| v10.x            | v10.x          | v0.50.x    | v0.40.x  |

## Migration Notes

### From IBC Hooks to Callbacks (v10)

If migrating to IBC Callbacks in v10:
1. Callbacks provides a more standardized interface
2. Better integration with packet forward middleware
3. Not dependent on CosmWasm
4. Requires different memo format

### Middleware Compatibility

- **Compatible with**: Transfer, Packet Forward, Rate Limiting
- **Incompatible with**: IBC Callbacks (choose one or the other)
- **Requires**: CosmWasm for contract execution

## Support

For issues and questions:
- GitHub Issues: [ibc-apps/issues](https://github.com/cosmos/ibc-apps/issues)
- Discord: Cosmos Network Discord #ibc channel
- Security Issues: security@interchain.io (for sensitive disclosures)