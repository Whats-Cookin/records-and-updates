# Frontend Implementation Guide - Phase 1

## Current State Analysis

The frontend has:
1. ✅ Basic API service layer created
2. ✅ Some endpoints updated to new structure
3. ✅ Field name transformations (rating→score, amount→amt)
4. ✅ Entity system partially implemented
5. ❌ Authentication not properly integrated with new backend
6. ❌ Data transformation incomplete for nested structures
7. ❌ Many components still using old data expectations

## Immediate Tasks (Phase 1)

### 1. Fix Authentication Integration

The backend expects JWT tokens for protected endpoints. Current frontend has auth utilities but needs updating.

**Required changes:**

```typescript
// Update /src/api/index.ts to properly handle auth errors
import axios from '../axiosInstance' // This already handles auth headers

// Ensure these endpoints include auth:
// - POST /api/claims
// - POST /api/credentials  
// - POST /api/reports/claim/:claimId/validate
```

### 2. Complete Data Transformation Layer

The backend returns nested structures but frontend expects flat data:

```typescript
// Backend returns:
{
  "entries": [{
    "id": 123,
    "subject": {
      "uri": "https://example.com/user1",
      "name": "User 1",
      "type": "PERSON",
      "image": "..."
    },
    "claim": "ENDORSES",
    "object": {
      "uri": "https://example.com/product1", 
      "name": "Product 1",
      "type": "PRODUCT"
    },
    "statement": "Great product!",
    "stars": 5
  }]
}

// Frontend expects:
{
  "claims": [{
    "claim_id": 123,
    "subject": "https://example.com/user1",
    "subject_name": "User 1",
    "subject_type": "PERSON",
    "subject_image": "...",
    "claim": "ENDORSES",
    "object": "https://example.com/product1",
    "statement": "Great product!",
    "stars": 5
  }]
}
```

**Update transformation in /src/api/index.ts:**

```typescript
export const transformFeedEntry = (entry: any): Claim => {
  return {
    claim_id: entry.id,
    subject: entry.subject?.uri || entry.subject,
    subject_name: entry.subject?.name,
    subject_type: entry.subject?.type,
    subject_image: entry.subject?.image,
    claim: entry.claim,
    object: entry.object?.uri || entry.object,
    object_name: entry.object?.name,
    object_type: entry.object?.type,
    statement: entry.statement,
    effectiveDate: entry.effectiveDate,
    confidence: entry.confidence,
    sourceURI: entry.sourceURI,
    howKnown: entry.howKnown,
    stars: entry.stars,
    score: entry.score,
    amt: entry.amt,
    unit: entry.unit,
    aspect: entry.aspect,
    // Map old fields for compatibility
    id: entry.id,
    source_link: entry.sourceURI,
    how_known: entry.howKnown
  }
}

// Update getFeed to transform response:
export const getFeed = async (params?: any) => {
  const response = await axios.get<any>('/api/feed', { params })
  return {
    claims: response.data.entries.map(transformFeedEntry),
    nextPage: response.data.pagination?.page < response.data.pagination?.pages 
      ? String(response.data.pagination.page + 1) 
      : undefined
  }
}
```

### 3. Update Components to Handle New Data

**Priority components to update:**

1. **Feed Component** (`/src/containers/feedOfClaim/index.tsx`)
   - Update to use transformed data
   - Handle pagination properly
   - Display entity badges

2. **Create Claim** (`/src/components/NewClaim/index.tsx`)
   - Update form fields (score, amt, unit, aspect)
   - Ensure auth token is sent
   - Handle new response structure

3. **Claim Details** (`/src/containers/ClaimDetails/index.tsx`)
   - Use new report endpoint structure
   - Display validations properly
   - Show entity information

4. **Graph** (`/src/containers/Explore/index.tsx`)
   - Handle enhanced nodes with entity data
   - Update node rendering

### 4. Add Login/Signup Components

Since the backend requires auth, we need basic auth UI:

```typescript
// Create /src/components/Auth/Login.tsx
import React, { useState } from 'react'
import { handleAuth } from '../../utils/authUtils'

export const Login: React.FC = () => {
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    try {
      // This endpoint needs to be added to backend or use existing auth
      const response = await axios.post('/auth/login', { email, password })
      handleAuth(response.data.accessToken, response.data.refreshToken)
      window.location.href = '/'
    } catch (error) {
      console.error('Login failed:', error)
    }
  }
  
  return (
    <form onSubmit={handleSubmit}>
      <input 
        type="email" 
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
        required
      />
      <input
        type="password"
        value={password} 
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Password"
        required
      />
      <button type="submit">Login</button>
    </form>
  )
}
```

### 5. Test Core Functionality

After implementing above changes, test:

1. **Feed Loading**
   - Should display claims with entity information
   - Pagination should work
   - No authentication required

2. **Claim Creation**
   - Should require login
   - Should submit with new fields
   - Should handle success/error properly

3. **Graph Display**
   - Should show nodes with entity types
   - Should handle clicks and navigation

4. **Reports**
   - Should show claim details
   - Should display validations
   - Validation submission should require auth

## File-by-File Changes Needed

### `/src/api/index.ts`
- Add comprehensive transformation functions
- Update all endpoints to match new backend
- Ensure proper error handling

### `/src/containers/feedOfClaim/index.tsx`
- Update to use new getFeed response structure
- Handle entity badges display
- Fix pagination

### `/src/components/NewClaim/index.tsx`
- Add new fields (stars, score, amt, unit, aspect)
- Remove old fields (rating, amount)
- Ensure auth is checked before submission

### `/src/containers/ClaimDetails/index.tsx`
- Update to use new report structure
- Display validation summary
- Show entity information

### `/src/containers/Explore/index.tsx`
- Update graph data handling
- Style nodes based on entity type
- Fix node click handlers

### `/src/App.tsx`
- Add login route
- Add auth check for protected routes
- Add global error handling

## Testing Checklist

- [ ] Can load feed without authentication
- [ ] Can view claim details without authentication  
- [ ] Login flow works and stores token
- [ ] Can create claim when authenticated
- [ ] Graph displays with entity-enhanced nodes
- [ ] Reports show validation data
- [ ] Can submit validation when authenticated
- [ ] Pagination works in feed
- [ ] Entity badges display correctly
- [ ] Error messages display appropriately

## Next Phase Preview

Once Phase 1 is complete and core functionality works:

**Phase 2** will add:
- Credential submission UI
- Entity report pages
- Trending topics
- Advanced search

**Phase 3** will add:
- Enhanced graph interactions
- User profiles
- Trust scores
- Advanced filtering

## Notes

- Keep console.log statements during development for debugging
- Test with both authenticated and unauthenticated states
- Ensure backward compatibility during transition
- Document any API issues discovered
