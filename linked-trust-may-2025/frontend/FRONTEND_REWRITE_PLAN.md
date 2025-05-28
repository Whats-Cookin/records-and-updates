# Frontend Rewrite Plan for LinkedTrust

## Overview
The backend has been completely rewritten with new API endpoints and data structures. While some frontend updates have been made, a significant rewrite is needed to properly align with the new backend architecture.

## Key Backend Changes to Address

### 1. New API Endpoints
- **Claims**: 
  - POST `/api/claims` (requires auth)
  - GET `/api/claims/:id`
  - GET `/api/claims/subject/:uri`
  
- **Credentials**:
  - POST `/api/credentials` (requires auth)
  - GET `/api/credentials/:uri`
  
- **Graph**:
  - GET `/api/graph/:uri`
  - GET `/api/graph`
  - GET `/api/graph/node/:nodeId/neighbors`
  
- **Feed**:
  - GET `/api/feed`
  - GET `/api/feed/entity/:entityType`
  - GET `/api/feed/trending`
  
- **Report**:
  - GET `/api/reports/claim/:claimId`
  - POST `/api/reports/claim/:claimId/validate` (requires auth)
  - GET `/api/reports/entity/:uri`

### 2. Data Structure Changes
- Claims now have nested subject/object structures with entity data
- New fields: `stars`, `score` (was `rating`), `amt` (was `amount`), `unit`, `aspect`
- Enhanced nodes with `entityType` and `entityData`
- Credentials as first-class objects
- Validation claims for claim verification

### 3. Authentication
- JWT token required for creating claims, submitting credentials, and validations
- Token should be sent as `Authorization: Bearer <token>`

## Required Frontend Updates

### Phase 1: Core API Integration (Priority)

1. **Update API Service Layer** ✅ (Partially done)
   - Need to handle authentication properly
   - Add proper error handling and retries
   - Handle nested response structures

2. **Fix Data Transformations**
   - Backend returns nested structures, frontend expects flat
   - Need comprehensive transformation layer
   - Handle both old and new data formats during transition

3. **Update Authentication Flow**
   - Implement proper JWT token management
   - Add login/signup components
   - Store token securely (not localStorage in artifacts)
   - Add auth headers to protected endpoints

### Phase 2: New Features

1. **Credential Management**
   - Create credential submission form
   - Display credentials in user profile
   - Show credential-based claims in feed

2. **Enhanced Feed**
   - Show entity badges/types
   - Add filtering by entity type
   - Implement trending topics section
   - Add pagination support

3. **Validation System**
   - Create validation UI for claims
   - Show validation summary (agrees/disagrees/confirms/refutes)
   - Display validation confidence levels

4. **Entity Reports**
   - Create entity profile pages
   - Show all claims about an entity
   - Display trust metrics
   - Show entity relationships

### Phase 3: UI/UX Improvements

1. **Graph Visualization**
   - Style nodes based on entity type
   - Show entity data on hover
   - Add filtering by entity type
   - Improve layout algorithm for better readability

2. **Search and Discovery**
   - Add entity search
   - Implement claim search
   - Add advanced filters (date range, confidence, claim type)

3. **User Profile**
   - Show user's claims
   - Display credentials
   - Show trust score/reputation
   - Add claim history

## Technical Debt to Address

1. **Remove Legacy Code**
   - Clean up old API endpoint references
   - Remove unused Ceramic integration code ✅ (Done)
   - Update all components to use new types

2. **Type Safety**
   - Ensure all API responses are properly typed
   - Add runtime validation for API responses
   - Fix any remaining type errors

3. **Performance**
   - Implement proper caching strategy
   - Add loading states for all async operations
   - Optimize graph rendering for large datasets

## Implementation Priority

1. **Immediate** (Required for basic functionality):
   - Fix authentication flow
   - Update all API calls to new endpoints
   - Fix data transformation issues
   - Ensure feed loads properly

2. **Short-term** (Within 1 week):
   - Add credential submission
   - Implement claim validation UI
   - Add entity filtering to feed
   - Fix graph visualization

3. **Medium-term** (Within 2 weeks):
   - Create entity report pages
   - Add trending topics
   - Implement search functionality
   - Improve UI/UX

## Testing Strategy

1. **API Integration Tests**
   - Test all endpoints with mock data
   - Verify authentication flow
   - Test error handling

2. **Component Tests**
   - Test data transformations
   - Verify UI updates with new data
   - Test user interactions

3. **E2E Tests**
   - Full user flow from login to claim creation
   - Credential submission flow
   - Validation flow
   - Graph exploration

## Migration Strategy

1. Keep transformation layer for backward compatibility
2. Gradually update components to use new data structures
3. Add feature flags for new functionality
4. Maintain parallel support during transition

## Next Steps

1. Review and approve this plan
2. Set up proper development environment
3. Create feature branches for each phase
4. Begin with authentication implementation
5. Update API service layer comprehensively
