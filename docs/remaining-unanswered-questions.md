# Remaining Unanswered Questions for IBC Middleware Integration

This document consolidates the questions that remain unanswered, requiring either:

1. Deep operational experience with working deployments
2. Specific version migration experience
3. CosmWasm-specific knowledge
4. Infrastructure expertise

## Hook System Implementation Details

### IBC Hooks Specific (from v7/v8 questions)
1. **Hook interface proliferation**: How many combinations of Before/After/Override hooks are valid in practice?
2. **Wasm contract execution integration**: How does Go code interface with WebAssembly contracts at the FFI boundary?
3. **Capability parameter usage**: Why does IBC hooks middleware use `*capabilitytypes.Capability` parameters when other middleware don't?
4. **ICS4 vs IBC middleware dual-layer**: When is the dual-layer pattern required vs optional?

## Memory and Performance Optimization

### Memory Management (relevant to v8 chains)
1. **Memory leak prevention**: What are common memory leak patterns to avoid in keeper implementations?
2. **Large packet payload handling**: What are best practices for efficiently handling large IBC packet payloads?
3. **Garbage collection tuning**: What GC settings work best for high-throughput IBC middleware?

### Performance Characteristics (relevant to v10 chains)
1. **Keeper storage access patterns**: What are the performance characteristics and optimization strategies?
2. **JSON marshaling overhead**: Are there more efficient serialization alternatives for high-frequency operations?

## Migration and Upgrade Specifics

### Version-Specific Migration (relevant to v7/v8 chains)
1. **v7→v8 specific breaking changes**: What exact code changes are required beyond interface updates?
2. **Mixed version coexistence**: Can v8 and v10 middleware run simultaneously during migration?
3. **Deprecated feature handling**: How to handle deprecated features during version upgrades?
4. **State migration patterns**: How to handle keeper state migrations during chain upgrades?

## Infrastructure and Operations

### Docker and E2E Testing (from all versions)
1. **Docker dependencies for E2E**: What exact Docker images and versions are required?
2. **Interchain test framework setup**: What is the minimal Docker compose configuration?
3. **SimApp differences from production**: What are key differences between SimApp and production deployments?

### Debugging and Monitoring (relevant to v8 chains)
1. **Middleware stack debugging**: How to trace requests through multiple middleware layers?
2. **Production monitoring setup**: What metrics and alerts are essential for production?
3. **Distributed tracing support**: How to implement distributed tracing across IBC hops?

## Governance and Configuration

### Runtime Configuration (relevant to v10 chains)
1. **Hot-swapping middleware**: Can middleware be reconfigured without chain restart?
2. **Parameter validation logic**: How are configuration parameters validated at runtime?
3. **Governance parameter updates**: How do governance proposals update middleware parameters?

## Architecture Decisions

### Design Patterns (from v7 questions)
1. **Empty interface usage**: How is type safety maintained with `interface{}` in packet data?
2. **Struct copying vs pointers**: What are the performance trade-offs in middleware design?
3. **Error wrapping strategies**: What is the standard pattern for error propagation through middleware?

### Module vs Middleware (relevant to v8 chains)
1. **Classification criteria**: What determines if functionality should be a module vs middleware?
2. **Router registration differences**: How does module registration differ from middleware wrapping?

## CosmWasm Integration

### Wasm-Specific (re: ibc hooks)
1. **Contract execution flow**: How are CosmWasm contracts invoked from IBC hooks?
2. What specifically is allowed?  Is it permissionless? Are there whitelisted or blacklisted messages?
3. **Gas management for contracts**: How is gas allocated between middleware and contract execution?
4. **Contract upgrade impacts**: How do contract upgrades affect middleware behavior?
