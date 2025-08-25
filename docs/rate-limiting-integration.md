# IBC Rate-Limiting Middleware Integration Guide

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Installation](#installation)
4. [Integration Steps](#integration-steps)
5. [Configuration](#configuration)
6. [Testing](#testing)
7. [Production Deployment](#production-deployment)
8. [Troubleshooting](#troubleshooting)
9. [API Reference](#api-reference)

## Overview

The IBC Rate-Limiting middleware provides critical protection against economic attacks by enforcing governance-configurable quotas on IBC token transfers. This middleware acts as a circuit breaker, limiting the financial damage from security incidents by capping the rate of token flows over specified time periods.

### Key Features
- **Bidirectional Flow Control**: Separate limits for inbound and outbound transfers
- **Net Flow Tracking**: Tracks net flow (outflow - inflow) to prevent DoS attacks
- **Time-Window Based**: Fixed UTC-aligned time windows for quota resets
- **Governance Configurable**: All parameters adjustable via governance proposals
- **Blacklist/Whitelist Support**: Fine-grained control over specific assets

### Architecture

The rate-limiting middleware operates by:
1. Intercepting all IBC transfer packets
2. Checking against configured quotas
3. Tracking net flows per (channel, denom) pair
4. Reverting state on failed/timeout packets

## Prerequisites

### Version Requirements
- **IBC-go**: v7.0.0+ (v8.0.0+ for latest features, v10.0.0+ for current stack)
- **Cosmos SDK**: v0.47.0+ (v0.50.0+ recommended)
- **Go**: 1.21+ (1.23+ recommended)

### Dependencies
```go
require (
    github.com/cosmos/ibc-apps/modules/rate-limiting/v10 v10.0.0
    github.com/cosmos/ibc-go/v10 v10.1.1
    github.com/cosmos/cosmos-sdk v0.50.13
)
```

## Installation

### Step 1: Add Module Dependency

Add to your `go.mod`:
```bash
go get github.com/cosmos/ibc-apps/modules/rate-limiting/v10@latest
```

### Step 2: Import Required Packages

In your `app/app.go`:
```go
import (
    ratelimit "github.com/cosmos/ibc-apps/modules/rate-limiting/v10"
    ratelimitkeeper "github.com/cosmos/ibc-apps/modules/rate-limiting/v10/keeper"
    ratelimittypes "github.com/cosmos/ibc-apps/modules/rate-limiting/v10/types"
)
```

## Integration Steps

### Step 1: Add Keeper to App Structure

```go
type App struct {
    *baseapp.BaseApp
    // ... other keepers
    
    TransferKeeper   ibctransferkeeper.Keeper
    RateLimitKeeper  ratelimitkeeper.Keeper  // Add this
    
    // ... rest of app
}
```

### Step 2: Add Store Keys

```go
// In NewApp function
keys := storetypes.NewKVStoreKeys(
    // ... other store keys
    ratelimittypes.StoreKey,  // Add this
)
```

### Step 3: Initialize the Keeper

```go
// After initializing IBC and Transfer keepers
app.RateLimitKeeper = ratelimitkeeper.NewKeeper(
    appCodec,
    keys[ratelimittypes.StoreKey],
    app.GetSubspace(ratelimittypes.ModuleName),
    authtypes.NewModuleAddress(govtypes.ModuleName).String(), // authority
    app.BankKeeper,
    app.IBCKeeper.ChannelKeeper,
    app.IBCKeeper.ChannelKeeper, // ICS4Wrapper
)
```

### Step 4: Configure Middleware Stack

```go
// Build the transfer stack with rate limiting as the outermost layer
var transferStack porttypes.IBCModule

// 1. Base transfer module
transferStack = transfer.NewIBCModule(app.TransferKeeper)

// 2. Add other middleware (callbacks, packet forward, etc.)
// ... other middleware wrapping

// 3. Rate limiting as outermost layer (security first)
transferStack = ratelimit.NewIBCMiddleware(
    app.RateLimitKeeper,
    transferStack,
)

// 4. Register with IBC router
ibcRouter := porttypes.NewRouter()
ibcRouter.AddRoute(ibctransfertypes.ModuleName, transferStack)
app.IBCKeeper.SetRouter(ibcRouter)
```

### Step 5: Register Module

```go
// In module manager
app.ModuleManager = module.NewManager(
    // ... other modules
    ratelimit.NewAppModule(app.RateLimitKeeper),
)

// Set module order for Begin/EndBlock
app.ModuleManager.SetOrderBeginBlockers(
    // ... other modules
    ratelimittypes.ModuleName,
)

app.ModuleManager.SetOrderEndBlockers(
    // ... other modules  
    ratelimittypes.ModuleName,
)

// Add to InitGenesis order
genesisModuleOrder := []string{
    // ... other modules
    ratelimittypes.ModuleName,
}
```

### Step 6: Add to Params Subspace

```go
// In init() function of app.go
func init() {
    // ... other param registrations
    paramsKeeper.Subspace(ratelimittypes.ModuleName)
}
```

## Configuration

### Rate Limit Parameters

Rate limits are configured per (channel, denom) pair with the following structure:

```go
type RateLimit struct {
    Path *Path `json:"path"`
    Quota *Quota `json:"quota"`
    Flow *Flow `json:"flow"`
}

type Path struct {
    Denom string `json:"denom"`
    ChannelId string `json:"channel_id"`
}

type Quota struct {
    MaxPercentSend math.Int `json:"max_percent_send"`
    MaxPercentRecv math.Int `json:"max_percent_recv"`
    DurationHours uint64 `json:"duration_hours"`
}
```

### Example Configuration via Governance

```json
{
  "title": "Add ATOM Rate Limit",
  "description": "Add 30% daily rate limit for ATOM on channel-0",
  "add_rate_limit": {
    "denom": "uatom",
    "channel_id": "channel-0",
    "max_percent_send": "30",
    "max_percent_recv": "30",
    "duration_hours": 24
  }
}
```

### Blacklist/Whitelist Configuration

```go
// Blacklist specific denoms
app.RateLimitKeeper.BlacklistDenom(ctx, "malicious-token")

// Whitelist trusted channels
app.RateLimitKeeper.WhitelistChannel(ctx, "channel-0")
```

## Testing

### Unit Tests

```go
func TestRateLimitMiddleware(t *testing.T) {
    // Setup test app
    app := simapp.Setup(t, false)
    ctx := app.BaseApp.NewContext(false, tmproto.Header{})
    
    // Add rate limit
    err := app.RateLimitKeeper.AddRateLimit(ctx, types.RateLimit{
        Path: &types.Path{
            Denom:     "uatom",
            ChannelId: "channel-0",
        },
        Quota: &types.Quota{
            MaxPercentSend: math.NewInt(10),
            MaxPercentRecv: math.NewInt(10),
            DurationHours:  1,
        },
    })
    require.NoError(t, err)
    
    // Test packet exceeding limit
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
    
    err = app.RateLimitKeeper.ReceiveRateLimitedPacket(ctx, packet)
    require.Error(t, err)
    require.Contains(t, err.Error(), "rate limit exceeded")
}
```

### Integration Tests

```bash
# Run integration tests
cd modules/rate-limiting
make test-unit

# Run e2e tests with Docker
make local-image
make ictest-ratelimit
```

### Manual Testing Commands

```bash
# Query all rate limits
simd query ratelimit list-rate-limits

# Query specific rate limit
simd query ratelimit rate-limit channel-0 uatom

# Query current flow
simd query ratelimit flow channel-0 uatom

# Add rate limit via governance
simd tx gov submit-proposal add-rate-limit proposal.json --from validator

# Update rate limit
simd tx gov submit-proposal update-rate-limit update.json --from validator

# Remove rate limit
simd tx gov submit-proposal remove-rate-limit remove.json --from validator
```

## Production Deployment

### Recommended Initial Configuration

```go
// Conservative initial limits (10% daily)
initialRateLimits := []types.RateLimit{
    {
        Path: &types.Path{
            Denom:     "uatom",
            ChannelId: "channel-0",
        },
        Quota: &types.Quota{
            MaxPercentSend: math.NewInt(10),
            MaxPercentRecv: math.NewInt(10),
            DurationHours:  24,
        },
    },
}
```

### Monitoring and Alerts

Key metrics to monitor:
- `rate_limit_exceeded_total`: Counter of rate limit violations
- `current_flow_percentage`: Gauge of current flow vs quota
- `rate_limit_quota_usage`: Histogram of quota usage

Example Prometheus alerts:
```yaml
groups:
- name: rate_limiting
  rules:
  - alert: RateLimitApproaching
    expr: rate_limit_quota_usage > 0.8
    for: 5m
    annotations:
      summary: "Rate limit approaching for {{ $labels.channel }}/{{ $labels.denom }}"
      
  - alert: RateLimitExceeded
    expr: increase(rate_limit_exceeded_total[5m]) > 0
    annotations:
      summary: "Rate limit exceeded for {{ $labels.channel }}/{{ $labels.denom }}"
```

### Emergency Response

In case of an active exploit:

1. **Immediate Response**: Tighten rate limits via emergency governance
```bash
# Emergency proposal with expedited voting
simd tx gov submit-proposal emergency-rate-limit.json --from validator --expedited
```

2. **Blacklist Affected Channels**:
```bash
simd tx ratelimit blacklist-channel channel-X --from authority
```

3. **Monitor and Adjust**:
- Watch for transaction logic
- Adjust limits based on legitimate traffic needs
- Consider full channel pause if necessary

## Troubleshooting

### Common Issues and Solutions

#### Issue: Rate limit not enforcing
**Solution**: Verify middleware ordering - rate limiting must be outermost layer:
```go
// Correct order (rate limiting wraps everything)
transferStack = ratelimit.NewIBCMiddleware(app.RateLimitKeeper, transferStack)
```

#### Issue: Legitimate transactions blocked
**Solution**: Adjust quotas or add to whitelist:
```bash
# Increase quota
simd tx gov submit-proposal increase-quota.json

# Or whitelist specific paths
simd tx ratelimit whitelist-path channel-0 uatom --from authority
```

#### Issue: State inconsistency after timeout
**Solution**: Ensure proper acknowledgement handling:
```go
func (k Keeper) TimeoutRateLimitedPacket(ctx sdk.Context, packet channeltypes.Packet) error {
    // Revert the outflow on timeout
    if err := k.RevertSentPacket(ctx, packet); err != nil {
        return err
    }
    return nil
}
```

### Debug Logging

Enable debug logging for troubleshooting:
```toml
# In app.toml
[log]
level = "debug"
filter = "ratelimit:debug,*:info"
```

## API Reference

### Keeper Methods

```go
// Core rate limiting functions
func (k Keeper) SendPacket(...) (sequence uint64, err error)
func (k Keeper) ReceiveRateLimitedPacket(ctx sdk.Context, packet channeltypes.Packet) error
func (k Keeper) AcknowledgeRateLimitedPacket(ctx sdk.Context, packet channeltypes.Packet, ack []byte) error
func (k Keeper) TimeoutRateLimitedPacket(ctx sdk.Context, packet channeltypes.Packet) error

// Configuration
func (k Keeper) AddRateLimit(ctx sdk.Context, rateLimit types.RateLimit) error
func (k Keeper) UpdateRateLimit(ctx sdk.Context, rateLimit types.RateLimit) error
func (k Keeper) RemoveRateLimit(ctx sdk.Context, denom, channelID string) error

// Queries
func (k Keeper) GetRateLimit(ctx sdk.Context, denom, channelID string) (types.RateLimit, bool)
func (k Keeper) GetAllRateLimits(ctx sdk.Context) []types.RateLimit
func (k Keeper) GetFlow(ctx sdk.Context, denom, channelID string) (types.Flow, bool)
```

### Message Types

```go
// Governance proposals
type MsgAddRateLimit struct {
    Authority string
    Denom string
    ChannelId string
    MaxPercentSend math.Int
    MaxPercentRecv math.Int
    DurationHours uint64
}

type MsgUpdateRateLimit struct {
    Authority string
    Denom string
    ChannelId string
    MaxPercentSend math.Int
    MaxPercentRecv math.Int
    DurationHours uint64
}

type MsgRemoveRateLimit struct {
    Authority string
    Denom string
    ChannelId string
}

type MsgResetRateLimit struct {
    Authority string
    Denom string
    ChannelId string
}
```

### Events

```go
// Emitted events
const (
    EventTypeRateLimitExceeded = "rate_limit_exceeded"
    EventTypeRateLimitUpdated = "rate_limit_updated"
    EventTypeFlowUpdated = "flow_updated"
    
    AttributeKeyDenom = "denom"
    AttributeKeyChannelID = "channel_id"
    AttributeKeyDirection = "direction"
    AttributeKeyAmount = "amount"
    AttributeKeyFlow = "flow"
    AttributeKeyQuota = "quota"
)
```

## Additional Resources

- [IBC-go v10 Documentation](https://ibc.cosmos.network/v10)
- [Rate Limiting Module Repository](https://github.com/cosmos/ibc-apps/tree/main/modules/rate-limiting)
- [Example Implementation (Stride)](https://github.com/Stride-Labs/stride/tree/main/x/ratelimit)
- [Osmosis Rate Limiting (Wasm-based)](https://github.com/osmosis-labs/osmosis/tree/main/x/ibc-rate-limit)

## Security Considerations

1. **Always place rate limiting as the outermost middleware layer** to ensure packets are checked before any processing
2. **Set conservative initial limits** and adjust based on actual usage logic
3. **Monitor for bypass attempts** through transaction batching or channel hopping
4. **Maintain an incident response plan** for rapid limit adjustments
5. **Consider different limits for different asset risk profiles**
6. **Implement comprehensive monitoring and alerting**

## Migration Notes

### From v7 to v8
- Add authority parameter to keeper constructor
- Update import paths to v8

### From v8 to v10
- No breaking changes in rate limiting module
- Update to work with new IBC-go v10 stack logic
- Consider integration with callbacks middleware if using

## Support

For issues and questions:
- GitHub Issues: [ibc-apps/issues](https://github.com/cosmos/ibc-apps/issues)
- Discord: Cosmos Network Discord #ibc channel
- Security Issues: security@interchain.io (for sensitive disclosures)