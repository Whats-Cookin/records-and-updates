# LinkedTrust Frontend - Clean Architecture Implementation

## What We've Done

We've aligned the frontend with the backend's proper data structure - NO MORE TRANSFORMATIONS!

### Key Changes Made Today:

1. **Removed all data transformations** - Frontend now uses backend data directly
2. **Updated types to match backend exactly**:
   - Claims have nested Entity objects for subject/object
   - Entities have: uri, name, type, image
   - No more flat structure with `subject_name`, `subject_type` etc.
3. **Fixed the feed component** to work with proper pagination
4. **Cleaned up legacy code** - removed old endpoint mappings

## Current Data Structure

### Backend Returns (and Frontend Uses):
```typescript
interface Entity {
  uri: string
  name?: string
  type?: string  // PERSON, ORGANIZATION, PRODUCT, etc.
  image?: string
}

interface Claim {
  id: number
  subject: Entity | string  // Can be full entity or just URI
  claim: string            // ENDORSES, TRUSTS, etc.
  object?: Entity | string 
  statement?: string
  effectiveDate: string
  confidence?: number
  sourceURI?: string
  howKnown?: string
  stars?: number    // 1-5 star rating
  score?: number    // 0-1 score (was 'rating')
  amt?: number      // amount (was 'amount')
  unit?: string
  aspect?: string
}
```

### Feed Response:
```typescript
{
  entries: Claim[]
  pagination: {
    page: number
    limit: number
    total: number
    pages: number
  }
}
```

## API Endpoints

All in `/src/api/index.ts`:

- `POST /api/claims` - Create claim (requires auth)
- `GET /api/claims/:id` - Get single claim
- `GET /api/feed` - Get feed with pagination
- `GET /api/graph/:uri` - Get graph for entity
- `GET /api/reports/claim/:id` - Get claim report
- `POST /api/reports/claim/:id/validate` - Submit validation (requires auth)
- `POST /api/credentials` - Submit credential (requires auth)

## What Works Now

✅ **Feed** - Displays claims with entity badges, proper pagination
✅ **Entity System** - Shows badges for PERSON, ORGANIZATION, PRODUCT, etc.
✅ **Graph** - Visualizes network with entity-aware nodes
✅ **Reports** - Shows claim details and validations
✅ **API Layer** - Clean, no transformations

## What Needs Implementation

### 1. Authentication UI
Currently using mock auth. Need proper login/signup components.

### 2. Create Claim Form
Update to use new fields: stars, score, amt, unit, aspect

### 3. Credential Submission
Create UI for submitting verifiable credentials

### 4. Entity Pages
Create pages showing all claims about an entity

### 5. Search & Filtering
Add entity type filtering, advanced search

## Quick Start

```bash
# Install and run
yarn install
yarn dev

# Environment
VITE_BACKEND_BASE_URL=https://dev.linkedtrust.us
```

## Code Structure

```
src/
├── api/
│   ├── index.ts      # All API calls - NO TRANSFORMATIONS
│   └── types.ts      # TypeScript types matching backend
├── containers/
│   ├── feedOfClaim/  # Main feed - UPDATED
│   ├── Explore/      # Graph visualization
│   └── ClaimDetails/ # Claim reports
└── components/
    ├── EntityBadge/  # Shows entity type badges
    └── NewClaim/     # Needs update for new fields
```

## Design Decisions

1. **Nested Data is Correct** - Entities are first-class objects, not flat fields
2. **No Transformations** - Frontend uses backend data as-is
3. **Entity-Aware** - Everything knows about entity types
4. **Clean API Layer** - Simple, direct calls to backend

## For Tomorrow's Team

The architecture is clean and aligned. Focus on:
1. Adding proper authentication UI
2. Updating claim creation form
3. Building credential submission
4. Creating entity report pages

The hard part (aligning data structures) is DONE. Now it's just building features on top of the clean foundation.

## Testing

```bash
# Run dev server
yarn dev

# Check these work:
- Load feed at http://localhost:5173
- Click on claims to see reports
- Navigate to graph view
- See entity badges on claims
```

## No More Legacy!

- ❌ No more `claim_id` - use `id`
- ❌ No more `source_link` - use `sourceURI`
- ❌ No more `how_known` - use `howKnown`
- ❌ No more transformations - use data as-is
- ✅ Clean, modern, entity-aware architecture
