# SDK Integration Answers and References

This document provides confirmed answers to expert questions about IBC middleware integration, with references to source code in the cosmos-sdk and ibc-go repositories.

## SDK Type System

### sdk.Context vs context.Context
**Answer**: `sdk.Context` is a specialized wrapper around Go's standard `context.Context` that adds blockchain-specific functionality.

**Reference**: `cosmos-sdk/types/context.go:41-65`

```go
type Context struct {
    baseCtx              context.Context  // Standard Go context
    ms                   storetypes.MultiStore
    header               cmtproto.Header
    gasMeter             storetypes.GasMeter
    eventManager         EventManagerI
    // ... additional blockchain-specific fields
}
```

**Key differences**:
- `sdk.Context` contains blockchain state (block height, time, gas meters)
- Provides access to key-value stores
- Manages events and gas consumption
- Thread-safe and immutable

**Usage**: Access the underlying Go context with `ctx.Context()`

### sdk.AccAddress
**Answer**: `AccAddress` is simply a byte slice (`[]byte`) that represents an account address with Bech32 encoding.

**Reference**: `cosmos-sdk/types/address.go:170`

```go
type AccAddress []byte
```

**Creating addresses**:
```go
// From Bech32 string
addr, err := sdk.AccAddressFromBech32("cosmos1...")

// From hex (unsafe - use with caution)
addr := sdk.AccAddressFromHexUnsafe("DEADBEEF...")

// From public key
addr := sdk.AccAddress(pubKey.Address())
```

### sdk.Coin
**Answer**: `Coin` is a struct containing a denomination and amount.

**Reference**: `cosmos-sdk/types/coin.pb.go:33`

```go
type Coin struct {
    Denom  string
    Amount math.Int  // Big integer for precision
}
```

**Creating coins**:
```go
// Single coin
coin := sdk.NewCoin("uatom", math.NewInt(1000000))

// Multiple coins
coins := sdk.NewCoins(
    sdk.NewCoin("uatom", math.NewInt(1000000)),
    sdk.NewCoin("ustake", math.NewInt(500000)),
)
```

### ModuleCdc
**Answer**: `ModuleCdc` is a module-specific codec for marshaling/unmarshaling data.

**Reference**: `cosmos-sdk/codec/legacy/amino.go`

**Usage in IBC**:
```go
var data transfertypes.FungibleTokenPacketData
if err := transfertypes.ModuleCdc.UnmarshalJSON(packet.GetData(), &data); err != nil {
    // Handle non-transfer packets
}
```

## IBC Acknowledgements

### When to Return Each Acknowledgement Type

**References**: 
- `ibc-go/modules/core/04-channel/types/acknowledgement.go`
- `ibc-go/modules/apps/rate-limiting/ibc_middleware.go:68`

**Success acknowledgement**:
```go
// For successful packet processing
return channeltypes.NewResultAcknowledgement([]byte{byte(1)})
```

**Error acknowledgement**:
```go
// For errors that should be communicated to the sender
return channeltypes.NewErrorAcknowledgement(err)
```

**Nil acknowledgement**:
```go
// For async processing - prevents automatic acknowledgement writing
return nil  // Middleware must later call WriteAcknowledgement
```

**Decision flow**:
1. Packet processed successfully → `NewResultAcknowledgement`
2. Packet failed with recoverable error → `NewErrorAcknowledgement`
3. Packet needs async processing → return `nil`
4. Critical failure → panic (will be caught by keeper)

## Testing Setup

### Creating SimApp for Testing

**Reference**: `cosmos-sdk/simapp/test_helpers.go:36-48`

**Basic setup**:
```go
func setup(withGenesis bool, invCheckPeriod uint) (*SimApp, GenesisState) {
    db := coretesting.NewMemDB()
    
    appOptions := make(simtestutil.AppOptionsMap, 0)
    appOptions[flags.FlagHome] = DefaultNodeHome
    appOptions[server.FlagInvCheckPeriod] = invCheckPeriod
    
    app := NewSimApp(log.NewNopLogger(), db, nil, true, appOptions)
    if withGenesis {
        return app, app.DefaultGenesis()
    }
    return app, GenesisState{}
}
```

### Interchain Test Framework

**Reference**: `ibc-go/testing/` package

**Setup pattern**:
```go
// See ibc-apps/modules/*/e2e/ directories for examples
import "github.com/strangelove-ventures/interchaintest/v8"

// Docker-based multi-chain testing environment
```

## Thread Safety and Concurrency

### Keeper Thread Safety
**Answer**: Keepers are thread-safe through the underlying store's mutex protection.

**Reference**: `cosmos-sdk/baseapp/abci.go` (shows mutex usage)

**Key points**:
- Store operations are protected by mutexes
- Each transaction gets its own context
- Parallel transaction processing is safe
- No additional synchronization needed in middleware

### Context Passing Overhead
**Answer**: Minimal - `sdk.Context` is a lightweight struct passed by value.

**Reference**: `cosmos-sdk/types/context.go`

**Performance characteristics**:
- Struct copying is optimized by Go compiler
- Most fields are pointers or small values
- Gas tracking adds minimal overhead
- Event collection is efficient

## Middleware Ordering

### Correct Stack Order

**Reference**: `gaia/app/keepers/keepers.go` and production chain examples

**Principles**:
1. **Bottom-up construction**: Base app → middleware → top
2. **Inbound flow**: Top middleware → base app
3. **Outbound flow**: Base app → top middleware

**Example order** (from production chains):
```go
// 1. Base transfer module
transferStack := transfer.NewIBCModule(app.TransferKeeper)

// 2. Packet forward middleware (wraps transfer)
transferStack = packetforward.NewIBCMiddleware(
    transferStack,
    app.PacketForwardKeeper,
    ...
)

// 3. Rate limiting (wraps packet forward)
transferStack = ratelimit.NewIBCMiddleware(
    app.RatelimitKeeper,
    transferStack,
)

// 4. IBC hooks (if needed)
transferStack = ibchooks.NewIBCMiddleware(
    transferStack,
    app.HooksICS4Wrapper,
)
```

## Configuration and Parameters

### Runtime Configuration
**Reference**: `cosmos-sdk/server/util.go:52`

**Pattern**:
```go
// Via app options
appOpts := make(simtestutil.AppOptionsMap, 0)
appOpts["key"] = value

// Via Viper configuration
viper.Set("key", value)
```

### Governance Parameters
**Reference**: `cosmos-sdk/x/params/` module

**Pattern**: Parameters managed through governance proposals and param store.

## Metrics and Telemetry

### Adding Telemetry

**Reference**: `cosmos-sdk/telemetry/` package

**Pattern**:
```go
import "cosmossdk.io/telemetry"

// In middleware methods
defer telemetry.ModuleMeasureSince(
    moduleName, 
    time.Now(), 
    telemetry.MetricKeyBeginBlocker,
)

// Counter metrics
telemetry.IncrCounter(1.0, moduleName, "packets_processed")
```

## Module Dependencies

### Import Versions

**Current versions** (as of documentation):
```go
require (
    github.com/cosmos/cosmos-sdk v0.50.13
    github.com/cosmos/ibc-go/v10 v10.1.1
    github.com/CosmWasm/wasmd v0.40.0  // if using CosmWasm
)
```

### Version Compatibility Matrix

| IBC-go | Cosmos SDK | Go Version |
|--------|------------|------------|
| v7.x   | v0.47.x    | 1.20+      |
| v8.x   | v0.50.x    | 1.21+      |
| v10.x  | v0.50.x    | 1.23+      |

## References for Further Investigation

### Primary Sources
- **SDK Types**: `cosmos-sdk/types/` directory
- **Testing Helpers**: `cosmos-sdk/simapp/test_helpers.go`
- **IBC Types**: `ibc-go/modules/core/*/types/`
- **Production Examples**: 
  - `gaia/app/app.go`
  - `osmosis/app/keepers/keepers.go`
  - `noble/app.go`

### Documentation
- [Cosmos SDK Docs](https://docs.cosmos.network)
- [IBC Protocol Docs](https://ibc.cosmos.network)
- [CosmWasm Docs](https://docs.cosmwasm.com)

## Expert Questions Quick Reference

| Question Category | Where to Find Answers |
|------------------|----------------------|
| SDK Types | `cosmos-sdk/types/` |
| Testing Setup | `cosmos-sdk/simapp/` |
| IBC Interfaces | `ibc-go/modules/core/exported/` |
| Acknowledgements | `ibc-go/modules/core/04-channel/types/` |
| Keeper Patterns | Production chain `/app/keepers/` |
| Thread Safety | `cosmos-sdk/baseapp/abci.go` |
| Configuration | `cosmos-sdk/server/` |
| Telemetry | `cosmos-sdk/telemetry/` |
| Migrations | `cosmos-sdk/x/upgrade/` |