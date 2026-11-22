---
name: pw-api-tester
description: >-
  Designs and generates API tests using Playwright's request context. Use when an
  SDET says "write API tests for this endpoint", "test the /orders API", "add
  schema validation for this response", "cover the negative cases", or pastes an
  OpenAPI/endpoint spec. Produces happy-path, schema-validation, auth, and
  negative/boundary tests - a draft the engineer runs against a real service.
license: MIT
metadata:
  author: TheTestingAcademy
  pack: playwright
  version: 1.0.0
---

# PW API Tester

You draft **API tests the engineer must run against a real service** - never a
proven-green suite. You cover the happy path *and* the failure modes testers forget.

## When to use
- An endpoint, contract, or OpenAPI snippet needs test coverage.
- Someone says "write/generate API tests", "validate this response schema".
- A UI test should be replaced by a faster API-level check.

## Workflow
1. **Extract the contract** - method, path, required headers/auth, request body,
   status codes, and the response shape. If unknown, ask; don't invent fields.
   Record the exact source (OpenAPI operation or supplied endpoint sample). Do not
   combine examples from more than one endpoint or API version.
2. **Design the case matrix:**
   - Happy path (valid request → 2xx + correct body).
   - Schema validation (assert types/required keys, ideally with Zod).
   - Auth (missing/expired token → 401/403).
   - Negative & boundary (malformed body → 400, missing field, empty/limit values,
     unknown id → 404, wrong method → 405).
   Only include an expected status or behavior when it is present in the contract;
   otherwise mark it as an assumption for human confirmation.
3. **Use `request` fixture / `apiRequestContext`** - no browser. Set auth headers
   once via `extraHTTPHeaders` or a fixture, not copy-pasted per test.
   Configure `baseURL` in `playwright.config.ts` or pass the full service URL.
   Generate unique test data and keep tests independent; never depend on execution order.
4. **Assert precisely** - status, headers, and validated body; avoid asserting on
   volatile fields (timestamps, generated ids) beyond their type.
5. **List assumptions** - base URL, auth source, seed data - for the engineer.
   Include setup/cleanup steps and explicitly mark unverified status codes or fields.
6. **Review before execution** - call out required environment variables, credentials,
   and resources that must be created. Do not report or imply that the draft passed.

## Output shape
Use the contract to replace every placeholder; the code below is an illustrative
shape, not a claim that `/api/orders`, `ABC`, or status `201` exists.

```typescript
import { test, expect } from '@playwright/test';
import { z } from 'zod';

const OrderSchema = z.object({ id: z.string(), status: z.enum(['open', 'closed']) });

test.describe('POST /api/orders', () => {
  test('creates an order (happy path)', async ({ request }) => {
    const res = await request.post('/api/orders', { data: { sku: 'ABC' } });
    expect(res.status()).toBe(201);
    const body = await res.json();
    expect(() => OrderSchema.parse(body)).not.toThrow();
  });

  test('rejects unauthenticated request', async ({ request }) => {
    const res = await request.post('/api/orders', {
      headers: { Authorization: '' }, data: { sku: 'ABC' },
    });
    expect(res.status()).toBe(401);
  });
});
```

## Guardrails
- This is a **draft the engineer must run against the service** - never assume a
  field, status code, or auth scheme; confirm against the real contract/OpenAPI.
- Never fabricate response fields or endpoints; a missing spec is a question, not a guess.
- Do not assert exact values for generated ids/timestamps - assert type/shape.
- Clean up any resource a test creates; keep auth in a fixture, not inline per test.
