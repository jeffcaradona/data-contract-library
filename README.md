# Functional Response Contract Library

A lightweight Node.js library demonstrating functional programming paradigms for enforcing response contracts across heterogeneous payload types.

## Overview

This library provides composable utilities to create and enforce response contracts that work seamlessly across:

- **Small payloads**: JSON responses with metadata
- **Paginated payloads**: Data sets with pagination metadata
- **Streamed payloads**: File downloads and streaming content

## Core Concepts

### Discriminated Unions

Response types are encoded as discriminated unions, making invalid states unrepresentable:

```javascript
const contract = SmallPayload({ status: 'ok' });
// contract.type === 'small'
// contract.data === { status: 'ok' }
```

### Higher-Order Functions

Handlers are composed using closures to enforce contracts before execution:

```javascript
const sendResponse = (res) => (contract) => {
  validateContract(contract);
  handleResponse(res)(contract);
};
```

### Pure Functions

All contract creators and validators are pure functions with no side effects:

```javascript
const PaginatedPayload = (data, page, pageSize, total) => ({
  type: 'paginated',
  data,
  metadata: { page, pageSize, total, timestamp: new Date().toISOString() }
});
```

## Benefits

1. **Type Safety**: Discriminated unions prevent invalid contract combinations
2. **Composability**: Validators and handlers can be mixed and reused
3. **Predictability**: Single responsibility for each payload type
4. **Testability**: Pure functions are trivial to test
5. **Memory Efficiency**: Streaming contracts avoid buffering large payloads

## Status

**In Planning Phase** - Documentation and architecture design in progress.

## License

MIT
