# Backward Compatibility Implementation Plan

## Required Legacy Endpoints (from https://live.linkedtrust.us/api/docs/)

### Auth Endpoints
- `POST /auth/login` - Email/password login
- `POST /auth/refresh_token` - Refresh JWT token

### Claim Endpoints  
- `POST /api/claim` - Create a claim (v3 format)
- `GET /api/claim/:id` - Get claim by ID
- `GET /api/claim` - Get claims with filters

## Implementation Strategy

### 1. Add Auth Endpoints (Quick - already exist)
Just need to add these routes to index.ts:
```typescript
app.post('/auth/login', authApi.login);
app.post('/auth/refresh_token', authApi.refreshToken);
```

### 2. Create Legacy Claim Endpoints
Need to create a new file `src/api/legacyClaims.ts` that:
- Accepts the old v3 claim format
- Transforms to new format internally
- Returns response in old format

### 3. Route Structure
Keep new endpoints at:
- `/api/claims/*` (new v4 endpoints)
- `/api/feed/*`
- `/api/graph/*`
- `/api/reports/*`

Add legacy endpoints at:
- `/api/claim` (old v3 endpoints)
- `/auth/*` (auth endpoints)

## Estimated Work

### Minimal Implementation (2-3 hours)
1. Add auth routes to index.ts (5 minutes)
2. Create legacyClaims.ts with:
   - Transform v3 claim format to new format
   - Call existing claim creation logic
   - Transform response back to v3 format
3. Add legacy routes to index.ts

### Full Implementation (4-6 hours)
1. Above plus:
2. Add proper Swagger/OpenAPI documentation
3. Add tests for legacy endpoints
4. Ensure exact response format matching
5. Add deprecation notices

## Key Transformations Needed

### V3 Claim Format (input)
```json
{
  "subject": "string",
  "claim": "string", 
  "object": "string",
  "rating": 0.5,  // maps to score
  "amount": 100,  // maps to amt
  // other fields...
}
```

### V3 Response Format (output)
```json
{
  "claimId": 123,  // maps from id
  "subject": "string",  // returns as string, not entity
  "claim": "string",
  "object": "string",  // returns as string, not entity
  // raw fields without entity enrichment
}
```
