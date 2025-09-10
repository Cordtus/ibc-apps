# Critical: Deprecated IBC Middleware Migration Guide

> [!WARNING]
> **IBC Fee Middleware (ICS-29) Completely Removed in IBC v10**
> 
> The removal of the IBC fee middleware in v10 is a **BREAKING CHANGE** that requires careful planning for chains currently using it.

## What Was ICS-29 Fee Middleware?

ICS-29 provided in-protocol incentivization for relayers by:
- **Escrowing fees**: Allowing any party to escrow tokens as relayer incentives
- **Fee distribution**: Automatically paying relayers for successful packet relay
- **Three fee types**:
  - `RecvFee`: Paid to forward relayer (MsgRecvPacket)
  - `AckFee`: Paid to reverse relayer (MsgAcknowledgement) 
  - `TimeoutFee`: Paid to timeout relayer (MsgTimeout)

## Impact of Removal

### Chains Currently Using Fee Middleware

If your chain has IBC fee middleware deployed:

1. **Immediate Breaking Changes**:
   - Upgrade to IBC v10 will fail if fee middleware references exist
   - Existing fee-enabled channels will break
   - Escrowed fees become inaccessible without migration

2. **Lost Functionality**:
   - No automatic relayer incentivization
   - No fee escrow mechanism
   - No payee registration system

3. **Data Migration Required**:
   - Must handle escrowed fees before upgrade
   - Need to refund or redistribute locked tokens
   - Channel metadata needs updating

## Migration Strategies

### Option 1: Remove Fee Middleware Completely (Recommended)

**When to use**: If relayer incentivization isn't critical to your chain

```go
// BEFORE (with fee middleware)
transferStack = transfer.NewIBCModule(app.TransferKeeper)
transferStack = ibcfee.NewIBCMiddleware(transferStack, app.IBCFeeKeeper)

// AFTER (without fee middleware)  
transferStack = transfer.NewIBCModule(app.TransferKeeper)
```

**Pre-Upgrade Handler Required**:
```go
func PreUpgradeHandler(ctx sdk.Context, app *App) error {
    // 1. Refund all escrowed fees
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
            return err
        }
    }
    
    // 2. Clear all fee-related state
    app.IBCFeeKeeper.DeleteAllIdentifiedPacketFees(ctx)
    app.IBCFeeKeeper.DeleteAllFeeEnabledChannels(ctx)
    app.IBCFeeKeeper.DeleteAllRegisteredPayees(ctx)
    
    return nil
}
```

### Option 2: Implement Custom Fee Solution

**When to use**: If relayer incentivization is critical

**Approach 1: Application-Level Fees**
```go
// Implement fee logic in your application module
type CustomFeeMiddleware struct {
    app          porttypes.IBCModule
    bankKeeper   BankKeeper
    feeCollector string
}

func (m CustomFeeMiddleware) OnRecvPacket(
    ctx sdk.Context,
    packet channeltypes.Packet,
    relayer sdk.AccAddress,
) exported.Acknowledgement {
    // Pay relayer from fee collector
    fee := m.calculateFee(packet)
    m.bankKeeper.SendCoins(ctx, m.feeCollector, relayer, fee)
    
    return m.app.OnRecvPacket(ctx, packet, relayer)
}
```

**Approach 2: Use IBC Callbacks for Fee Events**
```go
// Use IBC callbacks to trigger fee payments
func (k Keeper) OnRecvPacketCallback(
    ctx sdk.Context,
    packet channeltypes.Packet,
    ack exported.Acknowledgement,
    relayer sdk.AccAddress,
) error {
    // Custom fee logic here
    return k.payRelayerFee(ctx, packet, relayer)
}
```

### Option 3: Wait for Alternative Solutions

The Cosmos ecosystem is exploring alternatives:
- **Cross-chain queries**: Allow destination chains to pay relayers
- **Protocol-level solutions**: New standardized fee mechanisms
- **Relayer DAOs**: Community-funded relayer operations

## Channel Migration Requirements

### Fee-Enabled Channels

Channels created with fee middleware have specific version metadata:

```json
{
  "fee_version": "ics29-1",
  "app_version": "ics20-1"
}
```

**Migration Options**:

1. **Close and Recreate Channels**:
   - Safest but most disruptive
   - Ensures clean state

2. **Channel Upgrade** (if available):
   - Modify channel version during upgrade
   - Remove fee_version from metadata

## Testing Migration

### Critical Test Scenarios

1. **Escrowed Fee Recovery**:
   ```go
   // Test that all fees are refunded
   escrowedBefore := app.IBCFeeKeeper.GetAllIdentifiedPacketFees(ctx)
   require.NotEmpty(t, escrowedBefore)
   
   // Run migration
   err := PreUpgradeHandler(ctx, app)
   require.NoError(t, err)
   
   // Verify cleanup
   escrowedAfter := app.IBCFeeKeeper.GetAllIdentifiedPacketFees(ctx)
   require.Empty(t, escrowedAfter)
   ```

2. **Channel Functionality**:
   - Test IBC transfers still work
   - Verify no fee deductions occur
   - Ensure packets flow normally

3. **State Verification**:
   - Check all fee stores are removed
   - Verify module account is empty
   - Confirm no orphaned data

## Risk Assessment

### High Risk Scenarios

1. **Active Escrowed Fees**:
   - Risk: Token loss if not handled
   - Mitigation: Mandatory pre-upgrade refund

2. **High-Volume Relaying**:
   - Risk: Relayers stop operating without incentives
   - Mitigation: Establish off-chain agreements

3. **Cross-Chain Dependencies**:
   - Risk: Connected chains expect fee support
   - Mitigation: Coordinate upgrades

### Medium Risk Scenarios

1. **Channel Compatibility**:
   - Risk: Version mismatch with counterparties
   - Mitigation: Coordinate channel upgrades

2. **Relayer Configuration**:
   - Risk: Relayers configured for fees fail
   - Mitigation: Update relayer configs pre-upgrade

## Alternative Incentivization Models

### Without ICS-29

1. **Direct Agreements**:
   - Chains pay relayers directly
   - Off-chain service agreements
   - Foundation/DAO funding

2. **Transaction Fee Sharing**:
   - Share portion of tx fees with relayers
   - Implement in application logic
   - Use gov proposals for rates

3. **Token Incentives**:
   - Mint tokens for relayers
   - Stake-based rewards
   - Liquidity mining programs

## Checklist for Migration

### Pre-Upgrade
- Identify all fee-enabled channels
- Calculate total escrowed fees
- Notify relayers of changes
- Test fee refund logic
- Prepare alternative incentive model
- Coordinate with counterparty chains

### During Upgrade
- Execute fee refund handler
- Remove fee middleware from stack
- Delete fee module stores
- Update channel metadata

### Post-Upgrade
- Verify transfers still work
- Confirm no fees are collected
- Monitor relayer activity
- Implement alternative incentives

## Emergency Procedures

If upgrade fails due to fee middleware:

1. **Rollback Plan**:
   ```bash
   # Stop chain immediately
   systemctl stop gaiad
   
   # Restore from backup
   gaiad rollback
   
   # Apply hotfix to handle fees
   ```

2. **Emergency Fee Recovery**:
   ```go
   // Emergency handler to recover locked funds
   func EmergencyFeeRecovery(ctx sdk.Context, app *App) {
       // Force refund all fees to community pool
       total := app.IBCFeeKeeper.GetAllEscrowedFees(ctx)
       app.DistrKeeper.FundCommunityPool(ctx, total, app.AccountKeeper.GetModuleAddress(ibcfeetypes.ModuleName))
   }
   ```

## Conclusion

The removal of IBC fee middleware is a significant breaking change that requires careful planning. Chains must:

1. **Handle existing escrowed fees** before upgrading
2. **Remove all fee middleware references** from the codebase
3. **Implement alternative relayer incentives** if needed
4. **Coordinate with connected chains** to ensure compatibility

Failure to properly migrate will result in:
- Failed upgrade
- Locked tokens
- Broken IBC channels
- Loss of relayer support

## Resources

- [IBC v10 Migration Guide](./SDK-v052-IBC-v10-MIGRATION-GUIDE.md)
- [Original ICS-29 Spec](https://github.com/cosmos/ibc/tree/master/spec/app/ics-029-fee-payment)
- [Gaia v25 Implementation](https://github.com/cosmos/gaia) - Example of successful removal