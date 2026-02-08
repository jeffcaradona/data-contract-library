# API Documentation

## Contract Creators

### `SmallPayload(data)`

Creates a contract for small JSON responses.

**Parameters:**
- `data` (Object|Array): The response data

**Returns:**
- Contract object with `type: 'small'`

**Example:**
```javascript
import { SmallPayload } from './src/index.js';

const contract = SmallPayload({ status: 'ok', message: 'Operation completed' });
// {
//   type: 'small',
//   data: { status: 'ok', message: 'Operation completed' },
//   metadata: { timestamp: '2026-02-08T10:30:00.000Z' }
// }
```

---

### `PaginatedPayload(data, page, pageSize, total)`

Creates a contract for paginated data responses.

**Parameters:**
- `data` (Array): The items for the current page
- `page` (Number): Current page number (1-indexed)
- `pageSize` (Number): Number of items per page
- `total` (Number): Total number of items across all pages

**Returns:**
- Contract object with `type: 'paginated'`

**Example:**
```javascript
import { PaginatedPayload } from './src/index.js';

const contract = PaginatedPayload(
  [{ id: 1, name: 'Item 1' }, { id: 2, name: 'Item 2' }],
  1,
  10,
  42
);
// {
//   type: 'paginated',
//   data: [...items...],
//   metadata: {
//     page: 1,
//     pageSize: 10,
//     total: 42,
//     totalPages: 5,
//     hasNext: true,
//     hasPrevious: false,
//     timestamp: '2026-02-08T10:30:00.000Z'
//   }
// }
```

---

### `StreamedPayload(stream, filename, contentType?)`

Creates a contract for streamed responses (file downloads, large data).

**Parameters:**
- `stream` (ReadableStream): The stream to pipe to response
- `filename` (String): Filename for Content-Disposition header
- `contentType` (String, optional): MIME type (defaults to 'application/octet-stream')

**Returns:**
- Contract object with `type: 'streamed'`

**Example:**
```javascript
import fs from 'fs';
import { StreamedPayload } from './src/index.js';

const stream = fs.createReadStream('report.xlsx');
const contract = StreamedPayload(
  stream,
  'monthly-report.xlsx',
  'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
);
// {
//   type: 'streamed',
//   data: <ReadableStream>,
//   metadata: {
//     filename: 'monthly-report.xlsx',
//     contentType: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
//     timestamp: '2026-02-08T10:30:00.000Z'
//   }
// }
```

---

## Validators

### `validateContract(contract)`

Validates a contract against its type-specific rules.

**Parameters:**
- `contract` (Object): Contract object to validate

**Returns:**
- Result object: `{ ok: true, contract }` or `{ ok: false, error: 'message' }`

**Example:**
```javascript
import { validateContract } from './src/index.js';

const result = validateContract(contract);
if (!result.ok) {
  console.error('Validation failed:', result.error);
}
```

---

## Handlers

### `sendResponse(res)`

Higher-order function that creates a response sender for Express.

**Parameters:**
- `res` (Express.Response): Express response object

**Returns:**
- Function that accepts a contract and sends the appropriate response

**Example:**
```javascript
import { sendResponse } from './src/index.js';

router.get('/data', async (req, res) => {
  const contract = await controller.getData();
  sendResponse(res)(contract);
});
```

---

### `handleResponse(res)`

Lower-level handler that processes validated contracts.

**Parameters:**
- `res` (Express.Response): Express response object

**Returns:**
- Function that accepts a contract and executes the appropriate handler

**Example:**
```javascript
import { handleResponse, StreamedPayload } from './src/index.js';

const handler = handleResponse(res);
handler(StreamedPayload(stream, 'file.csv'));
```

---

## Middleware

### `contractMiddleware()`

Express middleware that adds contract sending capability to response object.

**Returns:**
- Express middleware function

**Example:**
```javascript
import { contractMiddleware, SmallPayload } from './src/index.js';

app.use(contractMiddleware());

// Now in routes:
router.get('/data', async (req, res) => {
  const contract = SmallPayload({ data: 'value' });
  res.sendContract(contract);
});
```

---

## Utilities

### `createMetadata(baseMetadata, customMetadata)`

Creates metadata object with timestamp and merges custom metadata.

**Parameters:**
- `baseMetadata` (Object): Base metadata required by contract type
- `customMetadata` (Object, optional): Additional metadata to merge

**Returns:**
- Merged metadata object with timestamp

---

### `isValidStream(stream)`

Checks if an object is a valid readable stream.

**Parameters:**
- `stream` (Any): Object to check

**Returns:**
- Boolean indicating if object is a valid stream

---

## Error Handling

All validation failures return result objects with `{ ok: false, error: 'message' }`.

Common error messages:
- `'Invalid response contract type'` - Unknown contract type
- `'Contract data must be an object or array'` - Small payload validation failed
- `'Paginated contract requires array data'` - Paginated payload validation failed
- `'Stream must have pipe method'` - Streamed payload validation failed
- `'Missing required pagination metadata'` - Pagination metadata incomplete

---

## Type Definitions

### Contract Object

```javascript
{
  type: 'small' | 'paginated' | 'streamed',
  data: any,
  metadata: {
    timestamp: string,  // ISO 8601 timestamp
    ...additionalMetadata
  }
}
```

### Validation Result

```javascript
{
  ok: boolean,
  contract?: Object,  // Present if ok === true
  error?: string      // Present if ok === false
}
```
