# Comprehensive IBC Middleware Integration Guide

## Executive Summary

This guide provides production-tested logic for integrating IBC middleware across different IBC-go versions (v7, v8, v10). Based on analysis of five production chains (Gaia, Juno, Neutron, Osmosis, Noble) and the ibc-apps reference implementations.

## Table of Contents

1. [Middleware Overview](#middleware-overview)
2. [Production Implementation logic](#production-implementation-logic)
3. [Version-Specific Integration](#version-specific-integration)
4. [Critical Integration Points](#critical-integration-points)
5. [Testing and Validation](#testing-and-validation)
6. [Troubleshooting Guide](#troubleshooting-guide)
7. [Migration logic](#migration-logic)

## Middleware Overview

### Available Middleware Types

| Middleware | Purpose | IBC-go Versions | Production Chains |
|------------|---------|-----------------|-------------------|
| **Rate Limiting** | Prevents economic attacks via transfer quotas | v7, v8, v10 | Gaia (v10), Neutron (WASM), Osmosis (WASM) |
| **Packet Forward** | Enables multi-hop IBC transfers | v7, v8, v10 | All analyzed chains |
| **IBC Hooks** | Triggers contract execution on transfers | v7, v8, v9 | Juno, Neutron, Osmosis |
| **IBC Callbacks** | Standardized packet lifecycle callbacks | v10+ only | Gaia |

### Middleware Stack Architecture

```
IBC Core <-> Middleware Stack <-> Application Module

Packet Flow:
- SendPacket: App -> Middleware (bottom to top) -> IBC Core
- RecvPacket: IBC Core -> Middleware (top to bottom) -> App
```

## Production Implementation logic

### Pattern 1: Gaia (IBC-go v10 with Callbacks)

Reference: `../gaia/app/keepers/keepers.go`

```go
// Bottom → top: Transfer -> Callbacks -> Provider -> PFM -> RateLimit
maxCallbackGas := gaiaparams.MaxIBCCallbackGas
wasmStackIBCHandler := wasm.NewIBCHandler(appKeepers.WasmKeeper, appKeepers.IBCKeeper.ChannelKeeper, appKeepers.IBCKeeper.ChannelKeeper)

// Callbacks middleware (constructor shape varies by ibc-go minor version)
cb := ibccallbacks.NewIBCMiddleware(wasmStackIBCHandler, maxCallbackGas)
cb.SetUnderlyingApplication(transfer.NewIBCModule(appKeepers.TransferKeeper))
cb.SetICS4Wrapper(appKeepers.PFMRouterKeeper)

transferStack := porttypes.IBCModule(cb)
transferStack = icsprovider.NewIBCMiddleware(transferStack, appKeepers.ProviderKeeper)
transferStack = pfmrouter.NewIBCMiddleware(transferStack, appKeepers.PFMRouterKeeper, 0, pfmrouterkeeper.DefaultForwardTransferPacketTimeoutTimestamp)
transferStack = ratelimit.NewIBCMiddleware(appKeepers.RatelimitKeeper, transferStack)
appKeepers.TransferKeeper.WithICS4Wrapper(cb)
```

### Pattern 2: Juno (IBC-go v8 with Hooks)

Reference: `../juno/app/keepers/keepers.go`

```go
// Stack order: Transfer -> Hooks -> PFM -> Fee -> IBC Core

var transferStack porttypes.IBCModule
transferStack = transfer.NewIBCModule(appKeepers.TransferKeeper)

// IBC Hooks wraps transfer
transferStack = ibchooks.NewIBCMiddleware(transferStack, &appKeepers.HooksICS4Wrapper)

// Packet forward middleware
transferStack = packetforward.NewIBCMiddleware(
    transferStack,
    appKeepers.PacketForwardKeeper,
    0,
    packetforwardkeeper.DefaultForwardTransferPacketTimeoutTimestamp,
)

// Fee middleware must come after PFM
transferStack = ibcfee.NewIBCMiddleware(transferStack, appKeepers.IBCFeeKeeper)
```

### Pattern 3: Osmosis (Custom Rate Limiting with WASM)

Reference: `../osmosis/app/keepers/keepers.go`

```go
// Custom WASM-based rate limiting implementation

// Create packet forward first
packetForwardMiddleware := packetforward.NewIBCMiddleware(
    transfer.NewIBCModule(*appKeepers.TransferKeeper),
    appKeepers.PacketForwardKeeper,
    0,
    packetforwardkeeper.DefaultForwardTransferPacketTimeoutTimestamp,
)

// WASM-based rate limiting
rateLimitingTransferModule := ibcratelimit.NewIBCModule(
    packetForwardMiddleware, 
    appKeepers.RateLimitingICS4Wrapper,
)

// Hooks as outermost layer
hooksTransferModule := ibchooks.NewIBCMiddleware(
    &rateLimitingTransferModule, 
    &appKeepers.HooksICS4Wrapper,
)
appKeepers.TransferStack = &hooksTransferModule
```

### Pattern 4: Noble (Minimal Middleware)

Reference: `../noble/legacy.go`

```go
// Minimal stack with custom middleware

var transferStack porttypes.IBCModule
transferStack = transfer.NewIBCModule(app.TransferKeeper)

// Custom forwarding middleware
transferStack = forwarding.NewMiddleware(transferStack, app.AccountKeeper, app.ForwardingKeeper)

// Packet forward middleware
transferStack = pfm.NewIBCMiddleware(
    transferStack,
    app.PFMKeeper,
    0,
    pfmkeeper.DefaultForwardTransferPacketTimeoutTimestamp,
    pfmkeeper.DefaultRefundTransferPacketTimeoutTimestamp,
)

// Custom blocking middleware
transferStack = blockibc.NewIBCMiddleware(transferStack, app.FTFKeeper)
```

## Version-Specific Integration

### IBC-go v7 Integration

```go
// Key differences in v7:
// 1. No authority parameter in keepers
// 2. PortKeeper is not a pointer
// 3. No native callbacks support

app.TransferKeeper = ibctransferkeeper.NewKeeper(
    appCodec, 
    keys[ibctransfertypes.StoreKey],
    app.GetSubspace(ibctransfertypes.ModuleName),
    app.IBCKeeper.ChannelKeeper,
    app.IBCKeeper.PortKeeper, // Not a pointer in v7
    app.AccountKeeper,
    app.BankKeeper,
    scopedTransferKeeper,
    // No authority parameter in v7
)
```

### IBC-go v8 Integration

```go
// v8 adds authority parameter
app.TransferKeeper = ibctransferkeeper.NewKeeper(
    appCodec,
    keys[ibctransfertypes.StoreKey],
    app.GetSubspace(ibctransfertypes.ModuleName),
    app.IBCKeeper.ChannelKeeper,
    &app.IBCKeeper.PortKeeper, // Pointer in v8+
    app.AccountKeeper,
    app.BankKeeper,
    scopedTransferKeeper,
    authtypes.NewModuleAddress(govtypes.ModuleName).String(), // Authority added
)
```

### IBC-go v10 Integration

```go
// v10 adds callbacks middleware (use setter pattern)
cb := ibccallbacks.NewIBCMiddleware(wasmHandler, maxCallbackGas)
cb.SetUnderlyingApplication(transfer.NewIBCModule(app.TransferKeeper))
cb.SetICS4Wrapper(app.PacketForwardKeeper)
app.TransferKeeper.WithICS4Wrapper(cb)
```

## Critical Integration Points

### 1. Keeper Initialization Order

**CRITICAL**: Initialize in this order to avoid circular dependencies

```go
// 1. Core IBC keepers first
app.IBCKeeper = ibckeeper.NewKeeper(...)

// 2. Middleware keepers (PFM needs special handling)
app.PacketForwardKeeper = packetforwardkeeper.NewKeeper(
    appCodec,
    keys[packetforwardtypes.StoreKey],
    nil, // Transfer keeper set later to avoid circular dep
    app.IBCKeeper.ChannelKeeper,
    app.BankKeeper,
    app.IBCKeeper.ChannelKeeper,
    authority,
)

// 3. Transfer keeper
app.TransferKeeper = ibctransferkeeper.NewKeeper(...)

// 4. Set circular references after all keepers created
app.PacketForwardKeeper.SetTransferKeeper(app.TransferKeeper)
```

### 2. ICS4Wrapper Configuration

**ICS4Wrapper guidance** (v10):

```go
// Ensure transfer sends go through callbacks middleware
app.TransferKeeper.WithICS4Wrapper(cb)
```

### 3. Store Key Registration

```go
keys := storetypes.NewKVStoreKeys(
    // ... other keys
    ibctransfertypes.StoreKey,
    packetforwardtypes.StoreKey,  // PFM store
    ratelimittypes.StoreKey,       // Rate limiting store
    ibchookstypes.StoreKey,        // Hooks store
)
```

### 4. Module Registration

```go
app.ModuleManager = module.NewManager(
    // ... other modules
    transfer.NewAppModule(app.TransferKeeper),
    packetforward.NewAppModule(app.PacketForwardKeeper, app.GetSubspace(packetforwardtypes.ModuleName)),
    ratelimit.NewAppModule(appCodec, app.RatelimitKeeper),
    ibchooks.NewAppModule(app.AccountKeeper), // Hooks doesn't need keeper in app module
)
```

### 5. Begin/End Block Order

```go
// Begin block order (rate limiting needs BeginBlock)
app.ModuleManager.SetOrderBeginBlockers(
    // ... other modules
    ratelimittypes.ModuleName,
    // ...
)
```

## Testing and Validation

### Unit Test Pattern

```go
func TestMiddlewareStack(t *testing.T) {
    app := simapp.Setup(t, false)
    ctx := app.BaseApp.NewContext(false, tmproto.Header{})
    
    // Test packet flow through middleware
    packet := channeltypes.NewPacket(
        transferData,
        1,
        "transfer",
        "channel-0",
        "transfer",
        "channel-0",
        clienttypes.NewHeight(1, 100),
        0,
    )
    
    // Test middleware processing
    ack := app.TransferStack.OnRecvPacket(ctx, packet, relayer)
    require.NotNil(t, ack)
}
```

### Integration Test Commands

```bash
# Test packet forward middleware
cd middleware/packet-forward-middleware
make local-image
make ictest-forward

# Rate limiting has unit tests in this repo
cd modules/rate-limiting
go test ./...

# Test hooks with contracts
cd modules/ibc-hooks
make test-unit
```

## Troubleshooting Guide

### Issue: Circular Dependency Error

**Symptom**: Panic during app initialization
```
panic: circular dependency between PacketForwardKeeper and TransferKeeper
```

**Solution**: Initialize PFM keeper with nil transfer keeper, set after transfer keeper creation:
```go
app.PacketForwardKeeper = packetforwardkeeper.NewKeeper(
    // ...
    nil, // Set later with SetTransferKeeper
    // ...
)
// After transfer keeper creation:
app.PacketForwardKeeper.SetTransferKeeper(app.TransferKeeper)
```

### Issue: Middleware Not Processing Packets

**Symptom**: Packets bypass middleware logic

**Solution**: Verify middleware is registered in IBC router:
```go
ibcRouter := porttypes.NewRouter()
ibcRouter.AddRoute(ibctransfertypes.ModuleName, transferStack) // Use stack, not module
app.IBCKeeper.SetRouter(ibcRouter)
```

### Issue: ICS4Wrapper Nil Pointer

**Symptom**: Panic when sending packets through middleware

**Solution**: Ensure ICS4Wrapper is set correctly:
```go
// For callbacks (v10)
app.TransferKeeper.WithICS4Wrapper(cb)
```

### Issue: Incompatible Middleware Combination

**Symptom**: Conflicts between IBC Hooks and Callbacks

**Solution**: Choose one approach:
- Use IBC Hooks for v7-v9
- Use IBC Callbacks for v10+
- Never use both in same stack

## Migration logic

### Upgrading from v7 to v8

1. Add authority parameter to all keeper constructors:
```go
// Before (v7)
NewKeeper(codec, storeKey, paramSpace, ...)

// After (v8+)
NewKeeper(codec, storeKey, paramSpace, ..., authority)
```

2. Update PortKeeper references to pointers:
```go
// Before (v7)
app.IBCKeeper.PortKeeper

// After (v8+)
&app.IBCKeeper.PortKeeper
```

### Upgrading from v8 to v10

1. Migrate from IBC Hooks to Callbacks:
```go
// Remove IBC Hooks
- transferStack = ibchooks.NewIBCMiddleware(transferStack, &app.HooksICS4Wrapper)

// Add Callbacks
+ cbStack := ibccallbacks.NewIBCMiddleware(
+     transferStack,
+     app.PacketForwardKeeper,
+     wasmHandler,
+     maxCallbackGas,
+ )
+ app.TransferKeeper.WithICS4Wrapper(cbStack)
```

2. Remove fee middleware (deprecated in v10):
```go
- transferStack = ibcfee.NewIBCMiddleware(transferStack, app.IBCFeeKeeper)
```

3. Remove capability module references:
```go
- capabilitytypes.StoreKey
- capability.NewAppModule(appCodec, *app.CapabilityKeeper, false)
```

## Best Practices

1. **Stack Order**: Rate limiting outermost, PFM in middle, hooks/callbacks closest to transfer
2. **Error Handling**: Middleware should not panic; return error acknowledgments
3. **Gas Management**: Set appropriate gas limits for callbacks/hooks
4. **Testing**: Test each middleware layer independently before integration
5. **Monitoring**: Add metrics for middleware packet processing

## References

- **SDK Integration Guide**: See [sdk-integration-answers.md](./sdk-integration-answers.md) for detailed answers about SDK types, testing, and integration patterns

- Production implementations analyzed (local checkouts):
  - Gaia: `../gaia/app/keepers/keepers.go`
  - Juno: `../juno/app/keepers/keepers.go`
  - Neutron: `../neutron/app/app.go`
  - Osmosis: `../osmosis/app/keepers/keepers.go`
  - Noble: `../noble/legacy.go`

- IBC-go versions:
  - v7: Limited middleware support, no callbacks
  - v8: Full middleware support, IBC hooks
  - v10: Native callbacks, deprecated fee middleware

- Module repositories:
  - [ibc-apps](https://github.com/cosmos/ibc-apps): Reference implementations
  - [ibc-go](https://github.com/cosmos/ibc-go): Core IBC and callbacks
  - [cosmos-sdk](https://github.com/cosmos/cosmos-sdk): SDK types and testing utilities
