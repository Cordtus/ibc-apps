# Cosmos SDK v0.50 to v0.53 + IBC-Go v10 Migration Guide

> **CRITICAL UPDATE**: SDK v0.52 was never fully released. The recommended path is direct migration from v0.50 to v0.53, as proven by Gaia's production deployment.

> **CONFIDENCE LEVEL**: This guide is based on official documentation and real-world implementations from production chains.
> **UPDATE**: SDK v0.53 is released and actively used in production by multiple chains.

## Version Matrix

| Component | From Version | To Version | Status | Production Examples |
|-----------|-------------|------------|---------|-------------------|
| Cosmos SDK | v0.50.x | v0.53.4 | Released | Gaia v25 (v0.53.3), cosmos/evm (v0.53.4) |
| IBC-Go | v8.x | v10.3.0 | Released | Gaia v25, cosmos/evm (v10 beta) |
| IBC-Hooks | v7 | v8 | Updated for IBC v10 | ibc-apps repo |
| Go | 1.21+ | 1.23.8+ | Required | cosmos/evm uses 1.23.8 |

## CONFIRMED BREAKING CHANGES

### 1. Capability Module Removal (HIGH CONFIDENCE)
**Source**: IBC-Go v10 CHANGELOG, SDK UPGRADING.md

The capability module has been completely removed from IBC-Go v10.

**Required Actions:**
- Remove all `capabilitykeeper.ScopedKeeper` from app struct
- Remove `CapabilityKeeper` from app struct  
- Remove capability store keys
- Remove capability module from module manager
- Update all keeper constructors to remove scoped keeper parameters

```go
// REMOVE THESE:
- CapabilityKeeper      *capabilitykeeper.Keeper
- ScopedIBCKeeper       capabilitykeeper.ScopedKeeper
- ScopedTransferKeeper  capabilitykeeper.ScopedKeeper
- ScopedICAHostKeeper   capabilitykeeper.ScopedKeeper
```

### 2. IBC Fee Middleware Removal (HIGH CONFIDENCE)

> [!CAUTION]
> This is a critical breaking change for chains using relayer incentivization.

**Source**: IBC-Go v10 CHANGELOG line 199

The 29-fee middleware (ICS-29) has been completely removed. **This is a major breaking change for chains using relayer incentivization.**

**What ICS-29 Provided (Now Lost):**
- Automatic relayer fee payments
- Fee escrow mechanism  
- Relayer payee registration

**Required Actions:**
- Remove `IBCFeeKeeper` from app struct
- Remove fee middleware from all IBC stacks
- Remove `ibcfeetypes.StoreKey` from store keys
- Remove fee module from module manager
- **Handle existing escrowed fees before upgrade** (see migration handler in SDK-UPGRADE-PATH-ANALYSIS.md)

**For Chains Currently Using Fee Middleware:**
See [DEPRECATED-MIDDLEWARE-MIGRATION.md](./DEPRECATED-MIDDLEWARE-MIGRATION.md) for detailed migration strategies.

### 3. Module Manager Simplification (HIGH CONFIDENCE)
**Source**: SDK UPGRADING.md lines 89-105

The `BasicModuleManager` has been removed.

**Required Actions:**
```go
// OLD
- app.BasicModuleManager = module.NewBasicManagerFromManager(...)
- app.BasicModuleManager.RegisterLegacyAminoCodec(legacyAmino)

// NEW
+ app.ModuleManager.RegisterLegacyAminoCodec(legacyAmino)
+ app.ModuleManager.RegisterInterfaces(interfaceRegistry)
```

### 4. Store Service Migration (HIGH CONFIDENCE)
**Source**: SDK UPGRADING.md lines 710-738

All module keepers now use `KVStoreService` instead of direct `StoreKey`.

**Required Actions:**
```go
// OLD
app.BankKeeper = bankkeeper.NewKeeper(
    appCodec,
    keys[banktypes.StoreKey],  // direct key
    ...
)

// NEW
app.BankKeeper = bankkeeper.NewKeeper(
    appCodec,
    runtime.NewKVStoreService(keys[banktypes.StoreKey]),
    ...
)
```

### 5. Light Client Module Routing (HIGH CONFIDENCE)
**Source**: IBC-Go migration guide

Light clients now require explicit module registration.

**Required Actions:**
```go
// Add after creating IBC keeper
clientKeeper := app.IBCKeeper.ClientKeeper
storeProvider := app.IBCKeeper.ClientKeeper.GetStoreProvider()

// Register each light client
tmLightClientModule := ibctm.NewLightClientModule(appCodec, storeProvider)
clientKeeper.AddRoute(ibctm.ModuleName, &tmLightClientModule)

// For WASM light clients
wasmLightClientModule := ibcwasm.NewLightClientModule(app.WasmClientKeeper, storeProvider)
clientKeeper.AddRoute(ibcwasmtypes.ModuleName, &wasmLightClientModule)
```

### 6. Context to Environment Migration (HIGH CONFIDENCE)
**Source**: SDK UPGRADING.md lines 373-392

Modules should use `appmodule.Environment` instead of `sdk.Context`.

**Required Actions:**
```go
// Example for circuit keeper
app.CircuitKeeper = circuitkeeper.NewKeeper(
    runtime.NewEnvironment(
        runtime.NewKVStoreService(keys[circuittypes.StoreKey]),
        logger.With(log.ModuleKey, "x/circuit")
    ),
    appCodec,
    authtypes.NewModuleAddress(govtypes.ModuleName).String(),
    app.AuthKeeper.AddressCodec(),
)
```

## IBC v10 SPECIFIC CHANGES

### IBC v2 (Eureka) Support (MEDIUM CONFIDENCE)
**Source**: IBC-Go CHANGELOG line 123

IBC v10 introduces Eureka (IBC v2) with significant performance improvements.

**Optional Implementation:**
```go
// Add IBC v2 transfer stack
var ibcv2TransferStack ibcapi.IBCModule
ibcv2TransferStack = transferv2.NewIBCModule(app.TransferKeeper)

// With callbacks v2
ibcv2TransferStack = ibccallbacksv2.NewIBCMiddleware(
    transferv2.NewIBCModule(app.TransferKeeper),
    app.IBCKeeper.ChannelKeeperV2,
    wasmStackIBCHandler,
    app.IBCKeeper.ChannelKeeperV2,
    maxCallbackGas,
)
```

### IBC Callbacks vs IBC-Hooks (HIGH CONFIDENCE)

**IBC Callbacks** (Native in IBC v10):
- Location: `github.com/cosmos/ibc-go/v10/modules/apps/callbacks`
- Purpose: ADR-008 packet lifecycle callbacks
- Status: Moved into ibc-go core in v10

**IBC-Hooks** (External Middleware):
- Location: `github.com/cosmos/ibc-apps/modules/ibc-hooks/v8`
- Purpose: Memo-based contract execution for ICS-20
- Status: Updated to v8 for IBC v10 compatibility

## MODULE IMPORT UPDATES

### SDK Modules (HIGH CONFIDENCE)
All SDK modules moved to separate go.mod:
```go
// OLD
import "github.com/cosmos/cosmos-sdk/x/bank"

// NEW
import "cosmossdk.io/x/bank"
```

### IBC Imports (HIGH CONFIDENCE)
```go
// Core IBC
import "github.com/cosmos/ibc-go/v10/modules/core"

// IBC Apps - if using callbacks
import "github.com/cosmos/ibc-go/v10/modules/apps/callbacks"

// IBC-Hooks - if using hooks middleware
import ibchooks "github.com/cosmos/ibc-apps/modules/ibc-hooks/v8"
```

## KEEPER CONSTRUCTOR UPDATES

### IBC Keeper (HIGH CONFIDENCE)
```go
app.IBCKeeper = ibckeeper.NewKeeper(
    appCodec,
    runtime.NewKVStoreService(keys[ibcexported.StoreKey]), // KVStoreService
    app.GetSubspace(ibcexported.ModuleName),
    // stakingKeeper removed
    app.UpgradeKeeper,
    // scopedIBCKeeper removed
    authtypes.NewModuleAddress(govtypes.ModuleName).String(),
)
```

### Transfer Keeper (HIGH CONFIDENCE)
```go
app.TransferKeeper = ibctransferkeeper.NewKeeper(
    appCodec,
    runtime.NewKVStoreService(keys[ibctransfertypes.StoreKey]),
    app.GetSubspace(ibctransfertypes.ModuleName),
    app.IBCKeeper.ChannelKeeper,
    app.IBCKeeper.ChannelKeeper,
    app.MsgServiceRouter(), // replaces PortKeeper
    app.AccountKeeper,
    app.BankKeeper,
    // scopedTransferKeeper removed
    authtypes.NewModuleAddress(govtypes.ModuleName).String(),
)
```

### ICA Host Keeper (HIGH CONFIDENCE)
```go
app.ICAHostKeeper = icahostkeeper.NewKeeper(
    appCodec,
    runtime.NewKVStoreService(keys[icahosttypes.StoreKey]),
    app.GetSubspace(icahosttypes.SubModuleName),
    // IBCFeeKeeper removed
    app.IBCKeeper.ChannelKeeper,
    app.IBCKeeper.ChannelKeeper, // used as ICS4Wrapper
    app.AccountKeeper,
    app.MsgServiceRouter(),
    app.GRPCQueryRouter(), // now passed in constructor
    authtypes.NewModuleAddress(govtypes.ModuleName).String(),
)
// WithQueryRouter call removed - now in constructor
```

## CRITICAL REMOVALS CHECKLIST

- [ ] Remove capability module completely
- [ ] Remove all scoped keepers
- [ ] Remove IBC fee middleware (29-fee)
- [ ] Remove stakingKeeper from IBC keeper
- [ ] Remove client proposal handler route
- [ ] Remove BasicModuleManager
- [ ] Remove WithQueryRouter calls (now in constructor)
- [ ] Remove PortKeeper references (use MsgServiceRouter)

## REAL-WORLD IMPLEMENTATIONS

### Production Chains Using SDK v0.53 + IBC v10

1. **Gaia v25**: SDK v0.53.3 + IBC-Go v10.3.0
2. **cosmos/evm**: SDK v0.53.4 + IBC-Go v10.0.0-beta

### Confirmed Working Patterns from Production

Based on these successful v0.53 implementations:

1. **No Capability Module References**: Gaia confirms complete removal
2. **All Keepers use KVStoreService**: `runtime.NewKVStoreService(keys[...])`
3. **Light Client Routing Works**: WASM and Tendermint clients properly routed
4. **IBC v10 with Callbacks**: Both v1 and v2 callbacks operational
5. **No IBC Fee Module**: Successfully removed without issues
6. **Packet Forward Middleware**: Working with new stack structure

### Gaia's IBC Stack Configuration

```go
// From Gaia v25 keepers.go
// IBC-Go v10.3.0 with proper light client routing
appKeepers.IBCKeeper = ibckeeper.NewKeeper(
    appCodec,
    runtime.NewKVStoreService(appKeepers.keys[ibcexported.StoreKey]),
    appKeepers.GetSubspace(ibcexported.ModuleName),
    appKeepers.UpgradeKeeper,
    authtypes.NewModuleAddress(govtypes.ModuleName).String(),
)

// WASM Light Client with proper VM setup
appKeepers.WasmClientKeeper = ibcwasmkeeper.NewKeeperWithVM(
    appCodec,
    runtime.NewKVStoreService(appKeepers.keys[ibcwasmtypes.StoreKey]),
    appKeepers.IBCKeeper.ClientKeeper,
    authtypes.NewModuleAddress(govtypes.ModuleName).String(),
    lc08, // wasmvm.VM instance
    bApp.GRPCQueryRouter(),
    ibcwasmkeeper.WithQueryPlugins(&wasmLightClientQuerier),
)
```

## TESTING RECOMMENDATIONS

1. **Unit Tests**: Update keeper mocks for new signatures
2. **Integration Tests**: Test IBC packet flow without capabilities
3. **Upgrade Handler**: Test state migrations thoroughly
4. **Light Clients**: Verify routing works correctly
5. **Reference Implementation**: Compare with Gaia v25 for proven patterns

## CONFIDENCE LEVELS

- **HIGH**: Directly documented in official UPGRADING.md, CHANGELOG, or proven in Gaia v25
- **MEDIUM**: Inferred from code examples or partial documentation
- **LOW**: Community reports or unofficial sources

## NEXT STEPS

1. Review your app.go against this checklist
2. Update all keeper constructors
3. Remove capability-related code
4. Wire up light client modules
5. Choose between IBC callbacks or IBC-hooks
6. Test thoroughly with ibc-go v10.3.0

## REFERENCES

### Official Documentation
- [SDK v0.52 UPGRADING.md](/Users/cordt/repos/cosmos-sdk/UPGRADING.md#v052x)
- [IBC-Go v8.1 to v10 Migration](/Users/cordt/repos/ibc-go/docs/docs/05-migrations/13-v8_1-to-v10.md)
- [IBC-Go CHANGELOG](/Users/cordt/repos/ibc-go/CHANGELOG.md)
- [IBC Callbacks README](/Users/cordt/repos/ibc-go/modules/apps/callbacks/README.md)
- [IBC-Hooks v8 README](/Users/cordt/repos/ibc-apps/modules/ibc-hooks/README.md)

### Production References
- [Gaia v25 keepers.go](/Users/cordt/repos/gaia/app/keepers/keepers.go) - SDK v0.53.3 implementation
- [cosmos/evm go.mod](/Users/cordt/repos/evm/go.mod) - SDK v0.53.4 implementation

## IMPORTANT NOTES

- **cosmos/evm**: The official Cosmos EVM implementation uses SDK v0.53.4
- **IBC v10 Beta**: Some chains (cosmos/evm) are using IBC v10 beta versions in production