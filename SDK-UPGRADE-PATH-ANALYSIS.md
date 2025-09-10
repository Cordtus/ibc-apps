# Cosmos SDK Upgrade Path Analysis: v0.50 to v0.53

## Executive Summary

Based on comprehensive research of Git history, releases, and production implementations:

**KEY FINDING**: SDK v0.51 never existed, and v0.52 was never fully released. The Cosmos team skipped v0.52 and went directly to v0.53.

## Version Timeline

### SDK Release History

| Version | Release Date | Status | Notes |
|---------|-------------|--------|-------|
| v0.50.0 | 2023-11-07 | Stable | Full release |
| v0.50.14 | 2025-07-08 | Stable | Latest v0.50 patch |
| v0.52.0-alpha.1 | 2024-08-15 | Pre-release | Never went stable |
| v0.52.0-beta.2 | 2024-10-09 | Pre-release | Last beta |
| v0.52.0-rc.2 | 2025-01-24 | Pre-release | Never became stable |
| **v0.53.0** | 2025-04-29 | **Stable** | First stable after v0.50 |
| v0.53.4 | 2025-07-25 | Stable | Latest version |

## Real-World Migration Evidence

### Gaia's Direct v0.50 → v0.53 Migration

Gaia (Cosmos Hub) successfully migrated directly from v0.50.13 to v0.53.3 in June 2025:

```bash
# Commit 558be632 (2025-06-03)
# "deps: Upgrade sdk 53"
- github.com/cosmos/cosmos-sdk v0.50.13
+ github.com/cosmos/cosmos-sdk v0.53.3
```

**Key Changes in Gaia's Migration:**
- Removed 2,725 lines (mostly old upgrade handlers)
- Added 982 lines (new v25 upgrade handler)
- Successfully removed all capability module references
- Migrated to IBC-Go v10

### cosmos/evm Implementation

The cosmos/evm chain started directly with SDK v0.53.4:
- No v0.52 in their history
- Uses IBC-Go v10.0.0-beta

## Migration Paths Analysis

### Path 1: v0.50 → v0.52 (beta) → v0.53 (NOT RECOMMENDED)

**Problems:**
- v0.52 never had a stable release
- Breaking changes were incomplete in v0.52
- Would require two major migrations

### Path 2: Direct v0.50 → v0.53 (RECOMMENDED)

**Evidence:**
- Gaia successfully did this in production
- cosmos/evm started fresh on v0.53
- Avoids unstable v0.52 code

**Advantages:**
- Single migration effort
- Proven in production
- All breaking changes handled at once

## Critical Breaking Changes: v0.50 → v0.53

### 1. Capability Module Removal
```diff
# Complete removal required:
- All ScopedKeeper references
- CapabilityKeeper
- capability store keys
- scopedIBCKeeper, scopedTransferKeeper, etc.
```

### 2. Store Service Migration
```diff
# All keepers must use KVStoreService:
- keys[types.StoreKey]
+ runtime.NewKVStoreService(keys[types.StoreKey])
```

### 3. IBC Migration (v8 → v10)
```diff
# Major changes:
- Remove IBC fee middleware (29-fee)
- Wire light client modules explicitly
- Add IBC callbacks or IBC-hooks
- Optional: Enable IBC v2 (Eureka)
```

### 4. Module Manager Changes
```diff
- BasicModuleManager removed
- Direct use of module.Manager
- RegisterLegacyAminoCodec on module manager
```

## IBC-Go v8 to v10 Migration Details

### Safe Removal of Deprecated Modules

1. **Capability Module**
   - Moved to IBC-Go in v8, removed in v10
   - Safe to remove entirely
   - No migration needed - just delete

2. **IBC Fee Middleware (29-fee)**
   - Completely removed in v10
   - No replacement needed
   - Remove from all stacks

### IBC v10 New Features

1. **IBC Callbacks (Native)**
   - Location: `github.com/cosmos/ibc-go/v10/modules/apps/callbacks`
   - Implements ADR-008
   - For packet lifecycle events

2. **IBC-Hooks (Middleware)**
   - Location: `github.com/cosmos/ibc-apps/modules/ibc-hooks/v8`
   - For memo-based contract execution
   - Updated to v8 for IBC v10

3. **IBC v2 / Eureka (Optional)**
   - More efficient packet handling
   - No channel dependencies
   - Optional - can be added later

## Upgrade Handler Requirements

### Actual v0.50 → v0.53 Handler (From Gaia v25)

```go
// Gaia's actual v25 upgrade handler - surprisingly minimal!
func CreateUpgradeHandler(
    mm *module.Manager,
    configurator module.Configurator,
    keepers *keepers.AppKeepers,
) upgradetypes.UpgradeHandler {
    return func(c context.Context, plan upgradetypes.Plan, vm module.VersionMap) (module.VersionMap, error) {
        ctx := sdk.UnwrapSDKContext(c)
        ctx.Logger().Info("Starting module migrations...")

        // Just run module migrations - that's it!
        vm, err := mm.RunMigrations(ctx, configurator, vm)
        if err != nil {
            return vm, errorsmod.Wrapf(err, "running module migrations")
        }

        ctx.Logger().Info("Upgrade v25 complete")
        return vm, nil
    }
}
```

**Note**: Gaia's handler doesn't explicitly call `MigrateAccountNumberUnsafe` - this may be handled automatically by module migrations or not required if x/accounts isn't used.

### Critical: Handling Escrowed Fees Before Upgrade

> [!CAUTION]
> If your chain uses IBC fee middleware, you MUST handle escrowed fees before upgrading. Failure to do so will result in permanently locked tokens.

```go
func PreUpgradeHandler(ctx sdk.Context, app *App) error {
    // Refund all escrowed fees before removing fee middleware
    fees := app.IBCFeeKeeper.GetAllIdentifiedPacketFees(ctx)
    for _, fee := range fees {
        err := app.BankKeeper.SendCoinsFromModuleToAccount(
            ctx, 
            ibcfeetypes.ModuleName,
            fee.RefundAddress,
            fee.Fee.Total(),
        )
        if err != nil {
            return err
        }
    }
    
    // Clear fee state
    app.IBCFeeKeeper.DeleteAllIdentifiedPacketFees(ctx)
    app.IBCFeeKeeper.DeleteAllFeeEnabledChannels(ctx)
    
    return nil
}
```

### Store Upgrades

```go
storetypes.StoreUpgrades{
    Added: []string{
        // Required for v0.53
        accountstypes.StoreKey,     // x/accounts module
        protocolpooltypes.StoreKey, // x/protocolpool module
    },
    Deleted: []string{
        // Removed modules
        capabilitytypes.StoreKey,   // capability removed
        ibcfeetypes.StoreKey,       // fee middleware removed (HANDLE FEES FIRST!)
    },
}
```

## Risk Assessment

### Direct v0.50 → v0.53 Migration

**Risks:**
- Large number of breaking changes at once
- Limited documentation for direct path
- Must handle all v0.52 changes implicitly

**Mitigations:**
- Proven successful in Gaia v25
- Single migration reduces operational risk
- Can reference Gaia's implementation

### Recommended Testing Strategy

1. **Local Testing**
   - Fork mainnet state
   - Test upgrade handler
   - Verify all modules initialize

2. **Testnet Deployment**
   - Run on testnet for 1-2 weeks
   - Test IBC connections
   - Verify governance works

3. **Staged Mainnet**
   - Coordinate upgrade height
   - Have rollback plan ready
   - Monitor closely post-upgrade

## IBC v2 (Eureka) Integration

### Gaia's Implementation

```go
// Create IBCv2 Transfer Stack
var transferStackV2 ibcapi.IBCModule
transferStackV2 = transferv2.NewIBCModule(appKeepers.TransferKeeper)
transferStackV2 = ibccallbacksv2.NewIBCMiddleware(
    transferStackV2, 
    appKeepers.IBCKeeper.ChannelKeeperV2,
    wasmStackIBCHandler, 
    appKeepers.IBCKeeper.ChannelKeeperV2, 
    gaiaparams.MaxIBCCallbackGas,
)
transferStackV2 = ratelimitv2.NewIBCMiddleware(appKeepers.RatelimitKeeper, transferStackV2)

// Create IBCv2 Router & seal
ibcv2Router := ibcapi.NewRouter().
    AddRoute(ibctransfertypes.PortID, transferStackV2)
appKeepers.IBCKeeper.SetRouterV2(ibcv2Router)
```

**Key Points:**
- IBC v2 runs parallel to v1 - not a replacement
- Uses `ChannelKeeperV2` for v2 operations
- Callbacks v2 integrated for packet lifecycle events
- Rate limiting v2 available

## Complete Migration Checklist

### Pre-Migration Preparation
- [ ] Review all breaking changes in SDK v0.53
- [ ] Test upgrade handler on forked mainnet state
- [ ] Prepare rollback plan

### Code Changes Required

#### 1. Remove Deprecated Modules
- [ ] Remove all capability module references
- [ ] Remove IBC fee middleware (29-fee)
- [ ] Remove BasicModuleManager
- [ ] Remove all ScopedKeeper references

#### 2. Update All Keepers
- [ ] Convert all keepers to use `runtime.NewKVStoreService()`
- [ ] Remove stakingKeeper from IBC keeper
- [ ] Update ICA/Transfer keepers with new signatures

#### 3. Wire Light Clients
- [ ] Add Tendermint light client module
- [ ] Add WASM light client if needed
- [ ] Register with client keeper router

#### 4. Add New Modules
- [ ] Add x/accounts module (if using)
- [ ] Add x/protocolpool module
- [ ] Configure store keys

#### 5. IBC v10 Integration
- [ ] Choose callbacks or IBC-hooks
- [ ] Wire IBC stacks without fee middleware
- [ ] Optionally add IBC v2 transfer stack

#### 6. Create Upgrade Handler
- [ ] Minimal handler (just RunMigrations)
- [ ] Configure store upgrades
- [ ] Test on testnet first

## Conclusion

**Recommended Path: Direct v0.50 → v0.53**

Reasons:
1. v0.52 was never stable - skipping it is safer
2. Gaia proved this path works in production
3. Single migration is operationally simpler
4. All breaking changes are well-documented in v0.53

The key to success is:
- Following Gaia's proven patterns
- Thorough testing on testnet
- Having a rollback plan ready
- Monitoring closely post-upgrade

**Timeline Evidence:**
- SDK v0.52: Only reached RC stage (Jan 2025)
- SDK v0.53: Full release (April 2025)
- Gaia: Successfully migrated v0.50 → v0.53 (June 2025)
- cosmos/evm: Started directly on v0.53

## References

- [Gaia v25 Upgrade Commit](https://github.com/cosmos/gaia/commit/558be632)
- [SDK v0.53 Release](https://github.com/cosmos/cosmos-sdk/releases/tag/v0.53.0)
- [IBC-Go v10 Migration Guide](/Users/cordt/repos/ibc-go/docs/docs/05-migrations/13-v8_1-to-v10.md)
- [SDK UPGRADING.md](/Users/cordt/repos/cosmos-sdk/UPGRADING.md)