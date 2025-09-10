# SDK v0.50 to v0.53 Structured Upgrade Process

> **Critical Decision**: Skip v0.52 (never stable) → Migrate directly from v0.50 to v0.53  
> **Proven By**: Gaia v25 (Cosmos Hub) production deployment - June 2025

## Phase 0: Pre-Upgrade Assessment (Day -14)

### Version Requirements
```bash
# Current → Target
Cosmos SDK: v0.50.x → v0.53.4+
IBC-Go:     v8.x    → v10.3.0+
Go:         1.21+   → 1.23.8+
```

### Critical Assessment Checklist
```bash
# 1. Check for IBC Fee Middleware
grep -r "ibcfee\|29-fee" app/

# 2. Check for Capability Module  
grep -r "capabilitykeeper\|ScopedKeeper" app/

# 3. Export current state for testing
<binary> export --height <current> > state_backup.json
```

⚠️ **STOP HERE** if using IBC Fee Middleware - Must handle escrowed fees (see Phase 1.5)

---

## Phase 1: Code Migration (Day -7)

### Step 1.1: Update Dependencies
```bash
go get github.com/cosmos/cosmos-sdk@v0.53.4
go get github.com/cosmos/ibc-go/v10@v10.3.0
go get cosmossdk.io/x/upgrade@latest
go get cosmossdk.io/store@latest
go mod tidy
```

### Step 1.2: Remove Capability Module Completely

**app/app.go - Remove ALL of these:**
```diff
- import (
-   capabilitykeeper "github.com/cosmos/ibc-go/modules/capability/keeper"
-   capabilitytypes "github.com/cosmos/ibc-go/modules/capability/types"
- )

type App struct {
-   CapabilityKeeper      *capabilitykeeper.Keeper
-   ScopedIBCKeeper       capabilitykeeper.ScopedKeeper  
-   ScopedTransferKeeper  capabilitykeeper.ScopedKeeper
-   // Remove ALL scoped keepers
}

// In NewApp():
- keys := storetypes.NewKVStoreKeys(
-   capabilitytypes.StoreKey,  // REMOVE
    ...
- )
- scopedIBCKeeper := app.CapabilityKeeper.ScopeToModule(...)  // REMOVE ALL
```

### Step 1.3: Update ALL Keeper Constructors

**Pattern for ALL keepers:**
```go
// OLD
- keeper.NewKeeper(codec, keys[types.StoreKey], ...)

// NEW  
+ keeper.NewKeeper(codec, runtime.NewKVStoreService(keys[types.StoreKey]), ...)
```

**IBC Keeper Specific:**
```go
app.IBCKeeper = ibckeeper.NewKeeper(
    appCodec,
    runtime.NewKVStoreService(keys[ibcexported.StoreKey]),
    app.GetSubspace(ibcexported.ModuleName),
    // app.StakingKeeper, // REMOVED
    app.UpgradeKeeper,
    // scopedIBCKeeper,   // REMOVED
    authtypes.NewModuleAddress(govtypes.ModuleName).String(), // NEW: authority
)
```

**Transfer Keeper:**
```go
app.TransferKeeper = ibctransferkeeper.NewKeeper(
    appCodec,
    runtime.NewKVStoreService(keys[ibctransfertypes.StoreKey]),
    app.GetSubspace(ibctransfertypes.ModuleName),
    app.IBCKeeper.ChannelKeeper,
    app.IBCKeeper.ChannelKeeper, // ICS4Wrapper
    app.MsgServiceRouter(),      // NEW: replaces PortKeeper
    app.AccountKeeper,
    app.BankKeeper,
    // scopedTransferKeeper,      // REMOVED
    authtypes.NewModuleAddress(govtypes.ModuleName).String(),
)
```

### Step 1.4: Wire Light Client Modules (NEW REQUIREMENT)

**Add immediately after IBC keeper creation:**
```go
// Get references
clientKeeper := app.IBCKeeper.ClientKeeper
storeProvider := app.IBCKeeper.ClientKeeper.GetStoreProvider()

// Register Tendermint light client
tmLightClientModule := ibctm.NewLightClientModule(appCodec, storeProvider)
clientKeeper.AddRoute(ibctm.ModuleName, &tmLightClientModule)

// If using WASM light clients
if app.WasmClientKeeper != nil {
    wasmLightClientModule := ibcwasm.NewLightClientModule(app.WasmClientKeeper, storeProvider)
    clientKeeper.AddRoute(ibcwasmtypes.ModuleName, &wasmLightClientModule)
}
```

### Step 1.5: Handle IBC Fee Middleware Removal (IF APPLICABLE)

**Pre-upgrade handler (MUST RUN BEFORE UPGRADE):**
```go
func HandleFeeMiddlewareRemoval(ctx sdk.Context, app *App) error {
    // Refund all escrowed fees
    fees := app.IBCFeeKeeper.GetAllIdentifiedPacketFees(ctx)
    for _, fee := range fees {
        err := app.BankKeeper.SendCoinsFromModuleToAccount(
            ctx, ibcfeetypes.ModuleName, fee.RefundAddress, fee.Fee.Total(),
        )
        if err != nil {
            return err
        }
    }
    
    // Clear state
    app.IBCFeeKeeper.DeleteAllIdentifiedPacketFees(ctx)
    app.IBCFeeKeeper.DeleteAllFeeEnabledChannels(ctx)
    return nil
}
```

**Remove from IBC stack:**
```diff
- transferStack = ibcfee.NewIBCMiddleware(transferStack, app.IBCFeeKeeper)
```

### Step 1.6: Rebuild IBC Stack (Choose ONE option)

**Option A: With IBC Callbacks (Native in v10):**
```go
var transferStack porttypes.IBCModule
transferStack = transfer.NewIBCModule(app.TransferKeeper)
transferStack = ibccallbacks.NewIBCMiddleware(
    transferStack,
    app.IBCKeeper.ChannelKeeper,
    app.ContractKeeper, // your contract keeper
    maxCallbackGas,
)
// Add other middleware (packet-forward, rate-limit) here
```

**Option B: With IBC-Hooks (External):**
```go
import ibchooks "github.com/cosmos/ibc-apps/modules/ibc-hooks/v8"

var transferStack porttypes.IBCModule
transferStack = transfer.NewIBCModule(app.TransferKeeper)
transferStack = ibchooks.NewIBCMiddleware(transferStack, app.IBCHooksKeeper)
// Add other middleware here
```

### Step 1.7: Update Module Manager

```diff
- app.BasicModuleManager = module.NewBasicManagerFromManager(...)
- app.BasicModuleManager.RegisterLegacyAminoCodec(legacyAmino)

+ app.ModuleManager.RegisterLegacyAminoCodec(legacyAmino)
+ app.ModuleManager.RegisterInterfaces(interfaceRegistry)
```

### Step 1.8: Update Import Paths

```diff
- import "github.com/cosmos/cosmos-sdk/x/bank"
+ import "cosmossdk.io/x/bank"

- import "github.com/cosmos/cosmos-sdk/x/staking"  
+ import "cosmossdk.io/x/staking"
```

---

## Phase 2: Upgrade Handler (Day -3)

### Minimal Upgrade Handler (Gaia v25 Pattern)
```go
package v25

import (
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
        
        // If using fee middleware, handle escrow FIRST
        // err := HandleFeeMiddlewareRemoval(sdkCtx, keepers)
        
        // Run module migrations (this handles most changes)
        vm, err := mm.RunMigrations(sdkCtx, configurator, vm)
        return vm, err
    }
}
```

### Store Upgrades
```go
app.SetStoreLoader(upgradetypes.UpgradeStoreLoader(upgradeInfo.Height, &storetypes.StoreUpgrades{
    Added: []string{
        accountstypes.StoreKey,     // New in v0.53
        protocolpooltypes.StoreKey, // New in v0.53
    },
    Deleted: []string{
        capabilitytypes.StoreKey,   // Capability removed
        // ibcfeetypes.StoreKey,     // Only if using fee middleware
    },
}))
```

---

## Phase 3: Testing (Day -2)

### Local Testing Commands
```bash
# 1. Build new binary
make build

# 2. Test state migration
./build/<binary> genesis migrate v0.53 state_backup.json > genesis_migrated.json

# 3. Start test node
./build/<binary> start --home ./test --x-crisis-skip-assert-invariants

# 4. Verify IBC functionality
./build/<binary> tx ibc-transfer transfer transfer channel-0 <address> 100stake
```

### Critical Test Points
- [ ] Chain starts without panic
- [ ] IBC transfers work
- [ ] Governance proposals submit
- [ ] All custom modules initialize
- [ ] No capability-related errors

---

## Phase 4: Deployment (Day 0)

### Governance Proposal
```bash
<binary> tx gov submit-proposal software-upgrade v25 \
  --title "Upgrade to SDK v0.53" \
  --description "Direct upgrade from v0.50 to v0.53" \
  --upgrade-height <height> \
  --deposit 10000000stake
```

### Cosmovisor Setup
```bash
# Place binary
mkdir -p $DAEMON_HOME/cosmovisor/upgrades/v25/bin
cp build/<binary> $DAEMON_HOME/cosmovisor/upgrades/v25/bin/

# Verify
$DAEMON_HOME/cosmovisor/upgrades/v25/bin/<binary> version
```

### Monitor Upgrade
```bash
# Watch logs
journalctl -u cosmovisor -f | grep -E "ERROR|PANIC|upgrade"

# Check consensus  
curl localhost:26657/consensus_state | jq '.result.round_state.height'
```

---

## Quick Troubleshooting

| Error | Solution |
|-------|----------|
| `undefined: capabilitykeeper.ScopedKeeper` | Remove all capability imports |
| `store key not found: capability` | Add to `Deleted` in store upgrades |
| `cannot use keys[...] as KVStoreService` | Wrap with `runtime.NewKVStoreService()` |
| `fee middleware not found` | Remove from IBC stack completely |
| `channel capability not found` | Ensure light clients are routed |
| `import cycle not allowed` | Update to cosmossdk.io/x/ paths |

### Emergency Rollback
```bash
# Stop immediately
systemctl stop cosmovisor

# Restore from backup
cp -r backup/<date>/* $HOME/.<chain>/
cp backup/<binary> /usr/local/bin/

# Restart old version
<old-binary> start --x-crisis-skip-assert-invariants
```

---

## Critical Success Factors

### ✅ Must Complete
1. Remove ALL capability module references
2. Remove ALL scoped keepers
3. Update ALL keepers to use KVStoreService
4. Wire light client modules
5. Handle fee middleware removal (if applicable)

### ⚠️ Common Mistakes
1. Forgetting to wire light clients → IBC breaks
2. Missing a keeper KVStoreService update → Panic at startup
3. Not handling escrowed fees → Tokens locked forever
4. Wrong import paths → Compilation errors

### 📊 Success Metrics
- Chain resumes in <6 hours
- >95% validator participation post-upgrade
- IBC transfers functional immediately
- Zero panic errors in first 24 hours

---

## Resources

**Production Examples:**
- [Gaia v25](https://github.com/cosmos/gaia/tree/v25) - Successful v0.50→v0.53
- [cosmos/evm](https://github.com/cosmos/evm) - Fresh v0.53 implementation

**Support:**
- Discord: [Cosmos #dev-help](https://discord.gg/cosmosnetwork)
- Discord: [IBC #ibc-dev](https://discord.gg/AzefAFd)

---

*Timeline: 2 weeks preparation, 2-6 hour upgrade window*  
*Based on: Gaia v25 production deployment (June 2025)*