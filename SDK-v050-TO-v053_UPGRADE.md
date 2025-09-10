# Complete SDK v0.50 to v0.53 Upgrade Guide

## Executive Overview

This guide consolidates documentation for upgrading from Cosmos SDK v0.50 to v0.53, including IBC-Go v8 to v10 migration.

## Table of Contents

1. [Pre-Migration Assessment](#pre-migration-assessment)
2. [Breaking Changes Checklist](#breaking-changes-checklist)
3. [Step-by-Step Migration Process](#step-by-step-migration-process)
4. [Code Changes Required](#code-changes-required)
5. [Upgrade Handler Implementation](#upgrade-handler-implementation)
6. [Testing Strategy](#testing-strategy)
7. [Production Deployment](#production-deployment)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Reference Implementations](#reference-implementations)

---

## Pre-Migration Assessment

### Current Version Requirements

| Component | Your Version | Target Version | Required? |
|-----------|-------------|---------------|-----------|
| Cosmos SDK | v0.50.x | v0.53.4+ | Yes |
| IBC-Go | v8.x | v10.3.0+ | Yes |
| Go | 1.21+ | 1.23.8+ | Yes |
| CosmWasm | v0.45.x | v0.53.x | If using |
| IBC-Hooks | v7 | v8 | If using |

### Risk Assessment Checklist

- [ ] **IBC Fee Middleware Usage**: Are you using ICS-29 fee middleware?
  - If YES: [See critical migration steps](#ibc-fee-middleware-removal)
- [ ] **Capability Module Dependencies**: Do you have custom capability usage?
  - If YES: Must refactor to remove all capability references
- [ ] **Custom Modules**: Do you have custom modules using SDK interfaces?
  - If YES: Review interface changes in [SDK UPGRADING.md](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/UPGRADING.md)
- [ ] **Light Client Customizations**: Do you use custom light clients?
  - If YES: Must implement new routing pattern
- [ ] **Chain Halt Duration**: Can your chain handle 2-6 hour upgrade window?
  - If NO: Consider staged rollout strategy

---

## Breaking Changes Checklist

### Critical Removals (Will Break Compilation)

#### 1. Capability Module - COMPLETE REMOVAL
```go
// REMOVE ALL OF THESE:
 capabilitykeeper "github.com/cosmos/ibc-go/modules/capability/keeper"
 capabilitytypes "github.com/cosmos/ibc-go/modules/capability/types"
 CapabilityKeeper *capabilitykeeper.Keeper
 ScopedIBCKeeper capabilitykeeper.ScopedKeeper
 ScopedTransferKeeper capabilitykeeper.ScopedKeeper
 ScopedICAHostKeeper capabilitykeeper.ScopedKeeper
 ScopedICAControllerKeeper capabilitykeeper.ScopedKeeper
```

#### 2. IBC Fee Middleware - COMPLETE REMOVAL
```go
// REMOVE ALL OF THESE:
 ibcfee "github.com/cosmos/ibc-go/v8/modules/apps/29-fee"
 ibcfeekeeper "github.com/cosmos/ibc-go/v8/modules/apps/29-fee/keeper"
 ibcfeetypes "github.com/cosmos/ibc-go/v8/modules/apps/29-fee/types"
 IBCFeeKeeper ibcfeekeeper.Keeper
```

#### 3. BasicModuleManager - REMOVED
```go
// OLD
 app.BasicModuleManager = module.NewBasicManagerFromManager(...)
 app.BasicModuleManager.RegisterLegacyAminoCodec(legacyAmino)

// NEW
 app.ModuleManager.RegisterLegacyAminoCodec(legacyAmino)
 app.ModuleManager.RegisterInterfaces(interfaceRegistry)
```

### Required Changes (Must Update)

#### 1. Store Service Migration - ALL KEEPERS
```go
// OLD - Direct StoreKey
 keeper.NewKeeper(codec, keys[types.StoreKey], ...)

// NEW - KVStoreService
 keeper.NewKeeper(codec, runtime.NewKVStoreService(keys[types.StoreKey]), ...)
```

#### 2. Import Path Updates - SDK MODULES
```go
// OLD
 import "github.com/cosmos/cosmos-sdk/x/bank"
 import "github.com/cosmos/cosmos-sdk/x/staking"

// NEW
 import "cosmossdk.io/x/bank"
 import "cosmossdk.io/x/staking"
```

---

## Step-by-Step Migration Process

### Phase 1: Preparation (1-2 weeks before)

1. **Fork and Test Environment**
   ```bash
   # Export current state
   <binary> export --height <current-height> > state_export.json
   
   # Set up test environment
   git clone <your-repo> test-upgrade
   cd test-upgrade
   git checkout -b upgrade/v0.53
   ```

2. **Update Dependencies**
   ```bash
   # Update go.mod
   go get github.com/cosmos/cosmos-sdk@v0.53.4
   go get github.com/cosmos/ibc-go/v10@v10.3.0
   go get cosmossdk.io/x/upgrade@latest
   go get cosmossdk.io/store@latest
   go get cosmossdk.io/core@latest
   
   # If using IBC-Apps
   go get github.com/cosmos/ibc-apps/modules/ibc-hooks/v8@latest
   go get github.com/cosmos/ibc-apps/modules/rate-limiting/v10@latest
   ```

### Phase 2: Code Migration

#### Step 1: Remove Capability Module

**File: app/app.go**
```go
// REMOVE these imports
- import (
-   capabilitykeeper "github.com/cosmos/ibc-go/modules/capability/keeper"
-   capabilitytypes "github.com/cosmos/ibc-go/modules/capability/types"
- )

// REMOVE from App struct
type App struct {
-   CapabilityKeeper      *capabilitykeeper.Keeper
-   ScopedIBCKeeper       capabilitykeeper.ScopedKeeper
-   ScopedTransferKeeper  capabilitykeeper.ScopedKeeper
    // ... rest of keepers
}

// REMOVE from NewApp()
- app.CapabilityKeeper = capabilitykeeper.NewKeeper(
-   appCodec,
-   keys[capabilitytypes.StoreKey],
-   memKeys[capabilitytypes.MemStoreKey],
- )

// REMOVE from store keys
- keys := storetypes.NewKVStoreKeys(
-   capabilitytypes.StoreKey,
    // ... other keys
- )

// REMOVE scoped keeper initialization
- scopedIBCKeeper := app.CapabilityKeeper.ScopeToModule(ibcexported.ModuleName)
- scopedTransferKeeper := app.CapabilityKeeper.ScopeToModule(ibctransfertypes.ModuleName)
```

#### Step 2: Update All Keeper Constructors

**IBC Keeper**
```go
// OLD
app.IBCKeeper = ibckeeper.NewKeeper(
    appCodec,
    keys[ibcexported.StoreKey],
    app.GetSubspace(ibcexported.ModuleName),
    app.StakingKeeper,     //  REMOVED
    app.UpgradeKeeper,
    scopedIBCKeeper,       //  REMOVED
)

// NEW
app.IBCKeeper = ibckeeper.NewKeeper(
    appCodec,
    runtime.NewKVStoreService(keys[ibcexported.StoreKey]), //  KVStoreService
    app.GetSubspace(ibcexported.ModuleName),
    app.UpgradeKeeper,     //  No StakingKeeper
    authtypes.NewModuleAddress(govtypes.ModuleName).String(), //  Authority
)
```

**Transfer Keeper**
```go
// NEW
app.TransferKeeper = ibctransferkeeper.NewKeeper(
    appCodec,
    runtime.NewKVStoreService(keys[ibctransfertypes.StoreKey]),
    app.GetSubspace(ibctransfertypes.ModuleName),
    app.IBCKeeper.ChannelKeeper,
    app.IBCKeeper.ChannelKeeper, // ICS4Wrapper
    app.MsgServiceRouter(),      //  Replaces PortKeeper
    app.AccountKeeper,
    app.BankKeeper,
    authtypes.NewModuleAddress(govtypes.ModuleName).String(),
)
```

**ICA Host Keeper**
```go
// NEW
app.ICAHostKeeper = icahostkeeper.NewKeeper(
    appCodec,
    runtime.NewKVStoreService(keys[icahosttypes.StoreKey]),
    app.GetSubspace(icahosttypes.SubModuleName),
    app.IBCKeeper.ChannelKeeper,
    app.IBCKeeper.ChannelKeeper, // ICS4Wrapper
    app.AccountKeeper,
    app.MsgServiceRouter(),
    app.GRPCQueryRouter(),        //  Now in constructor
    authtypes.NewModuleAddress(govtypes.ModuleName).String(),
)
// Note: No WithQueryRouter() call needed anymore
```

#### Step 3: Wire Light Client Modules

**After creating IBC keeper:**
```go
// Get references
clientKeeper := app.IBCKeeper.ClientKeeper
storeProvider := app.IBCKeeper.ClientKeeper.GetStoreProvider()

// Register Tendermint light client
tmLightClientModule := ibctm.NewLightClientModule(appCodec, storeProvider)
clientKeeper.AddRoute(ibctm.ModuleName, &tmLightClientModule)

// If using WASM light clients
if app.WasmClientKeeper != nil {
    wasmLightClientModule := ibcwasm.NewLightClientModule(
        app.WasmClientKeeper, 
        storeProvider,
    )
    clientKeeper.AddRoute(ibcwasmtypes.ModuleName, &wasmLightClientModule)
}
```

#### Step 4: Update IBC Stack (Remove Fee Middleware)

**OLD Stack with Fee Middleware:**
```go
//  OLD - With fee middleware
var transferStack porttypes.IBCModule
transferStack = transfer.NewIBCModule(app.TransferKeeper)
transferStack = ibcfee.NewIBCMiddleware(transferStack, app.IBCFeeKeeper)
transferStack = router.NewIBCMiddleware(transferStack, app.RouterKeeper)
```

**NEW Stack Options:**

**Option A: With IBC Callbacks (Native)**
```go
//  NEW - With native callbacks
var transferStack porttypes.IBCModule
transferStack = transfer.NewIBCModule(app.TransferKeeper)
transferStack = ibccallbacks.NewIBCMiddleware(
    transferStack,
    app.IBCKeeper.ChannelKeeper,
    app.ContractKeeper,    // Your contract keeper
    maxCallbackGas,
)
```

**Option B: With IBC-Hooks (External)**
```go
//  NEW - With IBC-hooks
import ibchooks "github.com/cosmos/ibc-apps/modules/ibc-hooks/v8"

app.IBCHooksKeeper = ibchookskeeper.NewKeeper(
    runtime.NewKVStoreService(keys[ibchookstypes.StoreKey]),
)

var transferStack porttypes.IBCModule
transferStack = transfer.NewIBCModule(app.TransferKeeper)
transferStack = ibchooks.NewIBCMiddleware(
    transferStack,
    app.IBCHooksKeeper,
)
```

#### Step 5: Update Module Manager

```go
// REMOVE BasicModuleManager
- app.BasicModuleManager = module.NewBasicManagerFromManager(
-     app.ModuleManager,
-     map[string]module.AppModuleBasic{
-         genutiltypes.ModuleName: genutil.NewAppModuleBasic(genutiltypes.DefaultMessageValidator),
-     },
- )
- app.BasicModuleManager.RegisterLegacyAminoCodec(legacyAmino)
- app.BasicModuleManager.RegisterInterfaces(interfaceRegistry)

// NEW - Direct on ModuleManager
+ app.ModuleManager.RegisterLegacyAminoCodec(legacyAmino)
+ app.ModuleManager.RegisterInterfaces(interfaceRegistry)
```

---

## Upgrade Handler Implementation

### Minimal Upgrade Handler (Gaia v25 Pattern)

```go
package v25

import (
    "context"
    
    upgradetypes "cosmossdk.io/x/upgrade/types"
    sdk "github.com/cosmos/cosmos-sdk/types"
    "github.com/cosmos/cosmos-sdk/types/module"
)

const UpgradeName = "v25"

func CreateUpgradeHandler(
    mm *module.Manager,
    configurator module.Configurator,
    keepers *keepers.AppKeepers,
) upgradetypes.UpgradeHandler {
    return func(ctx context.Context, plan upgradetypes.Plan, vm module.VersionMap) (module.VersionMap, error) {
        sdkCtx := sdk.UnwrapSDKContext(ctx)
        sdkCtx.Logger().Info("Starting module migrations...")

        // Run module migrations
        vm, err := mm.RunMigrations(sdkCtx, configurator, vm)
        if err != nil {
            return vm, err
        }

        sdkCtx.Logger().Info("Upgrade complete")
        return vm, nil
    }
}
```

### Store Upgrades Configuration

```go
app.SetStoreLoader(upgradetypes.UpgradeStoreLoader(upgradeInfo.Height, &storetypes.StoreUpgrades{
    Added: []string{
        accountstypes.StoreKey,     // New in v0.53
        protocolpooltypes.StoreKey, // New in v0.53
    },
    Deleted: []string{
        capabilitytypes.StoreKey,   // Removed
        ibcfeetypes.StoreKey,       // Removed (if using fee middleware)
    },
}))
```

### Critical: Handle IBC Fee Middleware Removal

**If using fee middleware, add this BEFORE upgrade:**

```go
func HandleEscrowedFees(ctx sdk.Context, app *App) error {
    // Get all escrowed fees
    fees := app.IBCFeeKeeper.GetAllIdentifiedPacketFees(ctx)
    
    for _, fee := range fees {
        // Refund to original payer
        err := app.BankKeeper.SendCoinsFromModuleToAccount(
            ctx,
            ibcfeetypes.ModuleName,
            fee.RefundAddress,
            fee.Fee.Total(),
        )
        if err != nil {
            return fmt.Errorf("failed to refund fees: %w", err)
        }
    }
    
    // Clear all fee state
    app.IBCFeeKeeper.DeleteAllIdentifiedPacketFees(ctx)
    app.IBCFeeKeeper.DeleteAllFeeEnabledChannels(ctx)
    app.IBCFeeKeeper.DeleteAllRegisteredPayees(ctx)
    
    return nil
}
```

---

## Testing Strategy

### Local Testing Checklist

1. **State Export/Import Test**
   ```bash
   # Export at current version
   <old-binary> export --height <height> > genesis_export.json
   
   # Migrate genesis
   <new-binary> genesis migrate v0.53 genesis_export.json > genesis_migrated.json
   
   # Start with migrated state
   <new-binary> start --home ./test
   ```

2. **Upgrade Handler Test**
   ```bash
   # Use cosmovisor for testing
   cosmovisor run start --x-crisis-skip-assert-invariants
   ```

3. **IBC Functionality Tests**
   - [ ] Create new IBC transfer
   - [ ] Verify existing channels work
   - [ ] Test packet timeout handling
   - [ ] Confirm relayer operations

4. **Module-Specific Tests**
   - [ ] Governance proposals work
   - [ ] Staking operations function
   - [ ] Bank transfers succeed
   - [ ] Custom modules operational

### Integration Testing

```go
func TestUpgradeHandler(t *testing.T) {
    // Setup
    app := simapp.Setup(t, false)
    ctx := app.BaseApp.NewContext(false, tmproto.Header{})
    
    // Pre-upgrade state
    preUpgradeChecks(t, ctx, app)
    
    // Run upgrade
    _, err := app.UpgradeKeeper.ApplyUpgrade(ctx, upgradetypes.Plan{
        Name:   "v25",
        Height: ctx.BlockHeight(),
    })
    require.NoError(t, err)
    
    // Post-upgrade verification
    postUpgradeChecks(t, ctx, app)
}
```

---

## Production Deployment

### Pre-Deployment Checklist

1. **Two Weeks Before**
   - [ ] Test upgrade on mainnet fork
   - [ ] Coordinate with validators
   - [ ] Prepare rollback procedure
   - [ ] Update documentation

2. **One Week Before**
   - [ ] Deploy to testnet
   - [ ] Monitor testnet for issues
   - [ ] Prepare announcement

3. **Day Before**
   - [ ] Final validator coordination
   - [ ] Confirm upgrade height
   - [ ] Stage binary releases

### Upgrade Execution

1. **Set Upgrade Height**
   ```bash
   # Submit governance proposal
   <binary> tx gov submit-proposal software-upgrade v25 \
     --title "Upgrade to v25" \
     --description "SDK v0.53 upgrade" \
     --upgrade-height <height> \
     --deposit 10000000stake
   ```

2. **Cosmovisor Setup**
   ```bash
   # Place new binary
   cp <new-binary> $DAEMON_HOME/cosmovisor/upgrades/v25/bin/
   
   # Verify
   $DAEMON_HOME/cosmovisor/upgrades/v25/bin/<binary> version
   ```

3. **Monitor Upgrade**
   ```bash
   # Watch logs
   journalctl -u cosmovisor -f
   
   # Check consensus
   curl localhost:26657/consensus_state
   ```

---

## Troubleshooting Guide

### Common Issues and Solutions

#### Issue 1: Capability Module References
```
Error: undefined: capabilitykeeper.ScopedKeeper
```
**Solution**: Remove all capability imports and references

#### Issue 2: Store Key Not Found
```
Error: store key capabilitytypes.StoreKey not found
```
**Solution**: Add to store upgrades `Deleted` list

#### Issue 3: IBC Channel Errors
```
Error: channel capability not found
```
**Solution**: Ensure light clients are properly routed

#### Issue 4: Fee Middleware Panic
```
panic: fee middleware not initialized
```
**Solution**: Remove fee middleware from IBC stack

#### Issue 5: Import Cycle
```
Error: import cycle not allowed
```
**Solution**: Update to new cosmossdk.io import paths

### Emergency Rollback Procedure

```bash
# Stop chain
systemctl stop cosmovisor

# Restore previous binary
cp <old-binary> /usr/local/bin/<binary>

# Restore data from backup
cp -r backup/<date>/* $HOME/.<chain>/

# Restart with old version
<old-binary> start --x-crisis-skip-assert-invariants
```

---

## Reference Implementations

### Production Examples

1. **Gaia v25** (Cosmos Hub)
   - [Upgrade Handler](https://github.com/cosmos/gaia/blob/main/app/upgrades/v25/upgrades.go)
   - [App Changes](https://github.com/cosmos/gaia/blob/main/app/app.go)
   - Successfully upgraded June 2025

2. **cosmos/evm**
   - [App Structure](https://github.com/cosmos/evm/blob/main/app/app.go)
   - Started directly on SDK v0.53.4

### Key Code References

```go
// Gaia's IBC Stack (proven pattern)
var transferStack porttypes.IBCModule
transferStack = transfer.NewIBCModule(app.TransferKeeper)
transferStack = ibccallbacks.NewIBCMiddleware(
    transferStack,
    app.IBCKeeper.ChannelKeeper,
    app.ContractKeeper,
    maxCallbackGas,
)
transferStack = packetforward.NewIBCMiddleware(
    transferStack,
    app.PacketForwardKeeper,
)
transferStack = ratelimit.NewIBCMiddleware(
    app.RatelimitKeeper,
    transferStack,
)
```

---

## Migration Timeline Example

| Week | Phase | Tasks |
|------|-------|-------|
| -3 | Planning | Review breaking changes, assess impact |
| -2 | Development | Update code, remove deprecated modules |
| -1 | Testing | Local tests, testnet deployment |
| 0 | Deployment | Governance proposal submission |
| +1 | Upgrade | Chain halt, upgrade execution |
| +2 | Monitoring | Post-upgrade monitoring, issue resolution |

---

## Resources and Support

### Official Documentation
- [SDK v0.53 Release Notes](https://github.com/cosmos/cosmos-sdk/releases/tag/v0.53.0)
- [SDK UPGRADING.md](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/UPGRADING.md)
- [IBC-Go v10 Migration](https://ibc.cosmos.network/main/migrations/v8-to-v10)

### Community Resources
- [Cosmos Discord](https://discord.gg/cosmosnetwork) - #dev-help channel
- [IBC Discord](https://discord.gg/AzefAFd) - #ibc-dev channel
- [Cosmos Forum](https://forum.cosmos.network) - Migration discussions

### Getting Help
- Review Gaia v25 implementation for proven patterns
- Check cosmos/evm for fresh v0.53 implementation
- Ask in Discord with specific error messages
- Reference this guide's troubleshooting section

---

## Final Checklist

Before starting migration:
- [ ] Backed up current state
- [ ] Reviewed all breaking changes
- [ ] Tested on local fork
- [ ] Prepared rollback plan
- [ ] Coordinated with validators
- [ ] Updated monitoring tools
- [ ] Documented custom changes

After migration:
- [ ] All modules initialize
- [ ] IBC transfers work
- [ ] Governance functional
- [ ] Metrics reporting
- [ ] No panic errors in logs
- [ ] Validators achieving consensus

---

*Last Updated: January 2025*  
*Based on Gaia v25 production deployment and SDK v0.53.4*
