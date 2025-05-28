# LinkedTrust Development Handoff Summary

## NEXT STEPS

* fix the search (totally broken, sorry)
* test the credential import and publishing, there should be a magic link flow if the draft prs are completed see the main readme in this repo
if any changes remember to discuss them with http://hub.cooperation.org/p/linkedtrust-core-development/
* try to use it for real for publishing credentials to it and then to linkedin
* try to issue team-issued credentials to people and send to them

## Recent Development Overview

This document summarizes the recent major development work on the LinkedTrust platform, focusing on the API migration, frontend updates, and graph visualization improvements completed in May 2025.

## 1. Backend API Standardization

### What Changed
- Migrated from legacy endpoints (`/api/claim/v2`, `/api/claims/v3`) to RESTful patterns
- Standardized field names across the platform:
  - `rating` → `score`
  - `amount` → `amt`
- Implemented proper entity system with typed nodes (PERSON, ORGANIZATION, CLAIM, etc.)

### Key Files Modified
- `/src/api/` - New centralized API routes
- `/src/services/pipelineTrigger.ts` - Enhanced entity detection
- `/src/api/report.ts` - Added image support for claims

### Current State
- All endpoints follow RESTful conventions
- Backward compatibility maintained during transition
- Entity types properly detected and stored

## 2. Frontend Architecture Overhaul

### Major Changes
- **Removed all Ceramic/ComposeDB dependencies** - Simplified the stack
- Created centralized API service layer (`/src/api/`)
- Implemented proper TypeScript types throughout
- Added entity badge system for visual type identification

### New Components Created
- `EntityBadge` - Visual indicators for entity types
- `IdentityManager` - DID management interface
- `CredentialCard` - Standardized credential display
- `EnhancedClaimCreator` - Improved claim creation UX

### API Integration
All API calls now go through `/src/api/index.ts` with proper types, eliminating scattered axios calls and ensuring consistency.

## 3. Graph Visualization Fixes

### Problems Solved
1. **Missing edges** - Fixed parsing to handle new GraphResponse structure
2. **Grey box backgrounds** - Removed, now transparent
3. **Node shapes** - Circles for entities, rectangles for claims
4. **Click-to-expand** - Restored functionality

### Visual Improvements
- Nodes with images display as circles with labels below
- Consistent color coding by entity/claim type
- Proper edge styling with colors indicating relationship type
- Removed excessive console logging

### Technical Details
- Updated Cytoscape configuration for proper rendering
- Fixed node/edge ID handling (all strings now)
- Improved label extraction for nodes without names

## 4. Report Page Enhancement

### Image Support
- Backend already provided image data from nodes
- Frontend now properly displays images for:
  - Main claim (full width)
  - Validations (smaller thumbnails)
  - Related claims (thumbnails)
- Added `image` field to Claim type interface

## 5. Authentication & Identity

### Current Setup
- Google OAuth integration
- Ethereum wallet connection
- Custom DID support (did:key, did:web, etc.)
- Identity manager for switching between DIDs

### User Flow
1. Login with Google or connect wallet
2. System generates appropriate DID
3. User can override with custom DID if needed
4. All claims signed with selected identity

## 6. Data Flow Architecture

```
User Action → Frontend API Service → Backend REST API → PostgreSQL
                                           ↓
                                    Pipeline Trigger
                                           ↓
                                    Entity Detection
                                           ↓
                                    Node/Edge Creation
                                           ↓
                                    Graph Updates
```

## 7. Development Environment

### Frontend (`trust_claim/`)
```bash
npm install
npm run dev  # Starts on localhost:5173
```

### Backend (`trust_claim_backend/`)
```bash
npm install
npm run dev  # Starts on localhost:8000
```

### Key Environment Variables
- `VITE_BACKEND_BASE_URL` - Backend API URL
- `GOOGLE_CLIENT_ID` - For OAuth
- `DATABASE_URL` - PostgreSQL connection

## 8. Testing Checklist

- [ ] Feed loads with entity badges
- [ ] Claim creation with score/amt fields
- [ ] Graph visualization with proper shapes
- [ ] Report page shows node images
- [ ] Click-to-expand in graph works
- [ ] Identity switching works
- [ ] OAuth login flow completes

## 9. Known Issues & Next Steps

### To Complete
- Entity filtering in feed UI
- Credential submission form
- Batch claim processing
- Performance optimization for large graphs

### Future Enhancements
- ActivityPub inbox implementation
- Progressive trust calculations
- Advanced search functionality
- Mobile responsiveness improvements

## 10. Important Notes

### API Compatibility
During transition, some endpoints support both old and new field names. These should be cleaned up once all clients are updated.

### Database Migrations
No manual migrations needed - Prisma handles schema updates automatically.

### Graph Limits
Currently limited to 30 nodes to prevent performance issues. This can be adjusted in `Explore/index.tsx`.

## Resources

- Architecture Guide: `/linked-trust/architecture-guide.md`
- API Documentation: Swagger UI at `/api-docs`
- LinkedClaims Spec: `/linked-trust/linked-claims-spec.md`

## Handoff Contacts

For questions about:
- Frontend architecture: Check `FRONTEND_UPDATE_PROGRESS.md`
- Graph visualization: See `GRAPH_FIX_SUMMARY.md`
- Backend changes: Review git history in `trust_claim_backend/src/api/`

---

*Last Updated: May 2025*
*Platform Version: LinkedTrust v2.0*
