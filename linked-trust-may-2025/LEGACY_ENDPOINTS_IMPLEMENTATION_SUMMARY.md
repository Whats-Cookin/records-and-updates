# Legacy Endpoints Implementation Summary

## What Was Done

### 1. Added Legacy Authentication Routes
Added the following auth endpoints to `src/index.ts`:
- `POST /auth/login` - Email/password login
- `POST /auth/signup` - User registration (maps to `authApi.register`)
- `POST /auth/refresh_token` - Token refresh

These routes use the existing `authApi.ts` implementation.

### 2. Added Legacy Claim Endpoints (v3 compatibility)
Added the following claim endpoints to `src/index.ts`:
- `POST /api/claim` - Create claim (v3 format)
- `POST /api/claim/v2` - Create claim with multipart/form-data images
- `GET /api/claim/:id` - Get single claim by ID
- `GET /api/claim` - Get claims with filters

These routes use the existing `legacyClaims.ts` implementation which:
- Transforms v3 claim format to new format
- Creates claims using the new system
- Transforms responses back to v3 format for compatibility

### 3. Added v4 Routes for New Endpoints
All new endpoints are now available at both paths:
- Original path (e.g., `/api/claims`) 
- v4 path (e.g., `/api/v4/claims`)

This allows:
- Existing integrations to continue working
- New integrations to use v4 paths explicitly
- Clear separation between legacy and modern APIs

### 4. Added Missing Dependencies
Updated `package.json` to include:
- `bcryptjs` - For password hashing
- `google-auth-library` - For Google OAuth
- `multer` - For multipart file uploads
- Related TypeScript types

### 5. Enhanced Legacy Claims for Image Upload
Extended `legacyClaims.ts` to support the multipart/form-data endpoint:
- Handles image file uploads
- Saves images to `uploads/` directory
- Generates signatures and metadata
- Returns v3-compatible response format

### 6. Added Static File Serving
Added middleware to serve uploaded images from `/uploads` path.

## Key Design Decisions

1. **No Database Migrations**: Following your preferences, no migrations were created. The system uses existing database structure.

2. **Backward Compatibility**: All legacy endpoints return data in exactly the same format as the original v3 API.

3. **Dual Routing**: New endpoints are available at both original and v4 paths to prevent conflicts.

4. **Transformation Layer**: Legacy endpoints transform between v3 and new formats transparently.

## What External Apps Can Expect

### Legacy Endpoints (Preserved)
- `/auth/*` - Authentication endpoints work exactly as before
- `/api/claim` - v3 claim endpoints work with original request/response formats
- `/api/claim/v2` - Multipart upload endpoint for claims with images

### New v4 Endpoints
- `/api/v4/claims/*` - Modern claim management
- `/api/v4/credentials/*` - Credential submission
- `/api/v4/graph/*` - Graph queries
- `/api/v4/feed/*` - Activity feeds
- `/api/v4/reports/*` - Reporting endpoints

## Next Steps

1. Run `npm install` to install new dependencies
2. Test legacy endpoints to ensure compatibility
3. Update frontend to use v4 endpoints
4. Create `uploads/` directory with appropriate permissions
5. Configure environment variables for auth (Google OAuth client ID, JWT secrets)

## Important Notes

- Image uploads are stored locally in `uploads/` directory
- In production, consider using S3 or similar for image storage
- The multipart endpoint implementation is basic - enhance security and validation as needed
- Legacy endpoints should be considered deprecated but will remain functional
