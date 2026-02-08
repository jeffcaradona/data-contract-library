# Usage Examples

## Basic Usage

### Small Payload Response

```javascript
import { SmallPayload, sendResponse } from './src/index.js';

router.get('/status', (req, res) => {
  const contract = SmallPayload({
    status: 'healthy',
    uptime: process.uptime(),
    version: '1.0.0'
  });
  
  sendResponse(res)(contract);
});

// Response:
// {
//   "status": "healthy",
//   "uptime": 12345.67,
//   "version": "1.0.0"
// }
```

---

### Paginated Response

```javascript
import { PaginatedPayload, sendResponse } from './src/index.js';

router.get('/users', async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const pageSize = parseInt(req.query.pageSize) || 10;
  
  const { users, total } = await getUsersPaginated(page, pageSize);
  
  const contract = PaginatedPayload(users, page, pageSize, total);
  sendResponse(res)(contract);
});

// Response:
// {
//   "items": [
//     { "id": 1, "name": "John" },
//     { "id": 2, "name": "Jane" }
//   ],
//   "pagination": {
//     "page": 1,
//     "pageSize": 10,
//     "total": 42,
//     "totalPages": 5,
//     "hasNext": true,
//     "hasPrevious": false,
//     "timestamp": "2026-02-08T10:30:00.000Z"
//   }
// }
```

---

### Streamed File Response

```javascript
import { StreamedPayload, sendResponse } from './src/index.js';
import fs from 'fs';

router.get('/export', async (req, res) => {
  const stream = await generateExcelReport(req.query.filters);
  const filename = `report-${Date.now()}.xlsx`;
  
  const contract = StreamedPayload(
    stream,
    filename,
    'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
  );
  
  sendResponse(res)(contract);
});

// Response:
// Headers:
//   Content-Disposition: attachment; filename="report-1739097600000.xlsx"
//   Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
// Body: <binary stream>
```

---

## Controller Pattern

### Separating Business Logic from Response Handling

```javascript
// controllers/dataController.js
import { SmallPayload, PaginatedPayload, StreamedPayload } from '../src/index.js';

class DataController {
  async getStats(filters) {
    const stats = await calculateStats(filters);
    return SmallPayload(stats);
  }
  
  async getRecords(page, pageSize) {
    const records = await db.query('SELECT * FROM records LIMIT ? OFFSET ?', 
      [pageSize, (page - 1) * pageSize]);
    const total = await db.queryOne('SELECT COUNT(*) as count FROM records');
    
    return PaginatedPayload(records, page, pageSize, total.count);
  }
  
  async exportData(filters) {
    const stream = createDataStream(filters);
    return StreamedPayload(stream, `export-${Date.now()}.csv`, 'text/csv');
  }
}

// routes/api.js
import { sendResponse } from '../src/index.js';
const dataController = new DataController();

router.get('/stats', async (req, res) => {
  const contract = await dataController.getStats(req.query.filters);
  sendResponse(res)(contract);
});

router.get('/records', async (req, res) => {
  const contract = await dataController.getRecords(
    parseInt(req.query.page) || 1,
    parseInt(req.query.pageSize) || 10
  );
  sendResponse(res)(contract);
});

router.get('/export', async (req, res) => {
  const contract = await dataController.exportData(req.query.filters);
  sendResponse(res)(contract);
});
```

---

## Error Handling Pattern

### Graceful Contract Validation

```javascript
import { sendResponse, validateContract, SmallPayload } from '../src/index.js';

router.get('/data', async (req, res) => {
  try {
    const contract = await someController.getData();
    
    // Manual validation (optional, sendResponse does this automatically)
    const validation = validateContract(contract);
    if (!validation.ok) {
      return res.status(400).json({ error: validation.error });
    }
    
    sendResponse(res)(contract);
  } catch (error) {
    console.error('Route error:', error);
    sendResponse(res)(SmallPayload({ error: 'Internal server error' }));
  }
});
```

---

## Middleware Pattern

### Global Contract Support

```javascript
import { contractMiddleware } from '../src/middleware.js';

app.use(contractMiddleware());

// Now use res.sendContract() anywhere
router.get('/users', async (req, res) => {
  const users = await getUsers();
  res.sendContract(SmallPayload(users));
});

router.get('/download', async (req, res) => {
  const stream = createFileStream();
  res.sendContract(StreamedPayload(stream, 'file.pdf', 'application/pdf'));
});
```

---

## Composition Pattern

### Combining Validators

```javascript
import { validateContract } from '../src/index.js';

const customValidation = (contract) => {
  const baseValidation = validateContract(contract);
  if (!baseValidation.ok) return baseValidation;
  
  // Additional business logic validation
  if (contract.type === 'paginated' && contract.metadata.page < 1) {
    return { ok: false, error: 'Page must be >= 1' };
  }
  
  return { ok: true, contract };
};

router.get('/data', async (req, res) => {
  const contract = await getData();
  const validation = customValidation(contract);
  
  if (!validation.ok) {
    return res.status(400).json({ error: validation.error });
  }
  
  sendResponse(res)(contract);
});
```

---

## Advanced: Conditional Response Types

### Same Endpoint, Different Contracts

```javascript
router.get('/report', async (req, res) => {
  const format = req.query.format || 'json';
  
  let contract;
  if (format === 'stream' || format === 'excel') {
    const stream = await generateExcelReport();
    contract = StreamedPayload(stream, 'report.xlsx');
  } else if (req.query.page) {
    const page = parseInt(req.query.page);
    const pageSize = parseInt(req.query.pageSize) || 50;
    const { data, total } = await getReportPaginated(page, pageSize);
    contract = PaginatedPayload(data, page, pageSize, total);
  } else {
    const data = await getReportSummary();
    contract = SmallPayload(data);
  }
  
  sendResponse(res)(contract);
});
```

---

## Testing Pattern

### Testing Controllers with Contracts

```javascript
import { validateContract } from '../src/index.js';
import { DataController } from '../controllers/dataController.js';

const dataController = new DataController();

describe('DataController', () => {
  test('getStats returns valid small payload contract', async () => {
    const contract = await dataController.getStats({});
    
    expect(contract.type).toBe('small');
    expect(validateContract(contract).ok).toBe(true);
    expect(contract.data).toHaveProperty('stats');
  });
  
  test('getRecords returns valid paginated contract', async () => {
    const contract = await dataController.getRecords(1, 10);
    
    expect(contract.type).toBe('paginated');
    expect(validateContract(contract).ok).toBe(true);
    expect(contract.metadata).toMatchObject({
      page: 1,
      pageSize: 10,
      total: expect.any(Number)
    });
  });
  
  test('exportData returns valid streamed contract', async () => {
    const contract = await dataController.exportData({});
    
    expect(contract.type).toBe('streamed');
    expect(validateContract(contract).ok).toBe(true);
    expect(contract.data.pipe).toBeDefined();
  });
});
```

---

## Memory Efficiency Comparison

### Traditional vs Contract Pattern

```javascript
// Traditional: Buffers entire dataset in memory
router.get('/export-old', async (req, res) => {
  const allData = await db.query('SELECT * FROM massive_table'); // ❌ OOM with large datasets
  const csv = convertToCSV(allData);
  res.send(csv);
});

// Contract Pattern: Streams data
import { StreamedPayload, sendResponse } from '../src/index.js';

router.get('/export-new', async (req, res) => {
  const stream = db.queryStream('SELECT * FROM massive_table'); // ✅ Constant memory
  const csvStream = stream.pipe(csvTransform());
  sendResponse(res)(StreamedPayload(csvStream, 'export.csv', 'text/csv'));
});
```
