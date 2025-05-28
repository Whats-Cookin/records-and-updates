# Comprehensive Cleanup Guide for LinkedTrust Frontend

## Overview
This guide identifies all legacy code, unused dependencies, and outdated patterns that should be removed.

## 1. Dependencies to Remove

### From package.json:
```bash
# These appear to be unused based on the new architecture:
yarn remove @react-oauth/google  # If not using Google OAuth
yarn remove ethers  # If not using Web3 anymore
yarn remove react-day-and-night-toggle  # Check if actually used
```

## 2. Files to Delete

### Ceramic/Web3 Related (Already cleaned per FRONTEND_UPDATE_PROGRESS.md ✅)
- ✅ All Ceramic integration code removed
- Check for any remaining Web3 references

### Legacy API Files
```bash
# Search and remove old API patterns
find src -name "*.ts" -o -name "*.tsx" | xargs grep -l "api/claim/v2"
find src -name "*.ts" -o -name "*.tsx" | xargs grep -l "api/claims/v3"
find src -name "*.ts" -o -name "*.tsx" | xargs grep -l "rating" | grep -v "score"
find src -name "*.ts" -o -name "*.tsx" | xargs grep -l "amount" | grep -v "amt"
```

### Unused Components
```bash
# Use ts-prune to find dead code
yarn find-deadcode

# Common suspects to check:
src/components/RedirectPage/  # Check if used
src/components/OverLayModal/  # Check if used
src/saved_from_dev/  # This entire directory can likely be deleted
```

## 3. Code Patterns to Remove

### Old Field Names
Search and replace throughout the codebase:
- `rating` → `score`
- `amount` → `amt`
- `claim_id` → `id` (standardize on one)
- `how_known` → `howKnown`

### Legacy Type Definitions
In `/src/api/types.ts`, remove:
```typescript
// Remove these legacy fields if no longer needed:
nodeUri?: string
entType?: string  
descrip?: string
edgesFrom?: any[]
edgesTo?: any[]
```

### Old API Endpoints
Remove any references to:
- `/api/claim/v2`
- `/api/claims/v3`
- `/api/claim/`
- `/api/node/`
- `/api/claim_graph/`

## 4. Cleanup Tasks by Directory

### /src/utils/
- `web3Auth.ts` - Delete if not using Web3
- `authUtils.ts` - Remove Ceramic-related code (already done ✅)
- Check for other unused utility files

### /src/hooks/
```bash
# Check which hooks are actually used
grep -r "useCreateClaim" src/ --include="*.tsx" --include="*.ts"
grep -r "useFeed" src/ --include="*.tsx" --include="*.ts"
# Delete unused hooks
```

### /src/containers/
- Remove any container components that reference old endpoints
- Update or remove components using old data structures

### /src/components/
- `EndNode/` and `StartNode/` - Check if these are used with new graph
- Remove any components with hardcoded old API calls

## 5. Database/Storage Cleanup

### localStorage
Add a migration function to clean old data:
```typescript
// Add to App.tsx or a startup utility
const cleanupLocalStorage = () => {
  const keysToRemove = [
    'ceramic_session',
    'did_session',
    'old_claims_cache',
    // Add other legacy keys
  ]
  
  keysToRemove.forEach(key => {
    localStorage.removeItem(key)
  })
  
  // Migrate old data if needed
  const oldToken = localStorage.getItem('authToken')
  if (oldToken && !localStorage.getItem('accessToken')) {
    localStorage.setItem('accessToken', oldToken)
    localStorage.removeItem('authToken')
  }
}

// Run on app startup
cleanupLocalStorage()
```

### Old Mock Data
- Delete `db.json` if not used
- Remove `/mocks/` directory if not needed
- Clean up any hardcoded test data

## 6. Build and Config Cleanup

### Build Files
```bash
# Clean build artifacts
rm -rf dist/
rm -rf build/
rm -rf node_modules/.cache/
```

### Config Files
- Review `.env` for outdated variables
- Update `vite.config.ts` to remove unused plugins
- Clean up `tsconfig.json` paths if needed

## 7. Type Safety Cleanup

### Fix anys
```bash
# Find all uses of 'any' type
grep -r ": any" src/ --include="*.ts" --include="*.tsx" | wc -l

# Priority files to fix:
src/api/index.ts
src/api/types.ts
src/utils/
```

### Remove @ts-ignore
```bash
# Find all @ts-ignore comments
grep -r "@ts-ignore" src/ --include="*.ts" --include="*.tsx"
```

## 8. CSS/Styling Cleanup

### Unused Styles
- Review `App.css` for unused classes
- Check component-specific CSS files
- Remove duplicate styling between CSS and styled-components

### Theme Cleanup
- Consolidate `Theme.ts` and `theme/` directory
- Remove unused color definitions
- Standardize on MUI theme or styled-components

## 9. Documentation Cleanup

### Remove Outdated Docs
```bash
# Files that may be outdated:
rm BUILD_FIX_SUMMARY.md  # Old build fixes
rm CERAMIC_CLEANUP_SUMMARY.md  # Already done
rm README-legacy.md  # Any legacy readmes
```

### Update Remaining Docs
- Update README.md with new architecture
- Remove references to old endpoints
- Update setup instructions

## 10. Final Cleanup Script

Create `/scripts/cleanup.sh`:
```bash
#!/bin/bash

echo "🧹 Starting LinkedTrust Frontend Cleanup..."

# Remove old dependencies
echo "📦 Cleaning dependencies..."
yarn remove @react-oauth/google ethers react-day-and-night-toggle

# Clean build artifacts
echo "🗑️  Removing build artifacts..."
rm -rf dist/ build/ node_modules/.cache/

# Remove saved_from_dev if exists
if [ -d "src/saved_from_dev" ]; then
  echo "🗑️  Removing saved_from_dev directory..."
  rm -rf src/saved_from_dev
fi

# Find and report legacy patterns
echo "🔍 Checking for legacy patterns..."
echo "Files with 'rating' (should be 'score'):"
grep -r "rating" src/ --include="*.ts" --include="*.tsx" | grep -v "score" | wc -l

echo "Files with 'amount' (should be 'amt'):"
grep -r "amount" src/ --include="*.ts" --include="*.tsx" | grep -v "amt" | wc -l

echo "Files with old endpoints:"
grep -r "api/claim/v2\|api/claims/v3" src/ --include="*.ts" --include="*.tsx" | wc -l

# Run dead code detection
echo "🔍 Finding dead code..."
yarn find-deadcode

# Clean and reinstall
echo "📦 Reinstalling dependencies..."
rm -rf node_modules yarn.lock
yarn install

echo "✅ Cleanup complete!"
```

Make it executable:
```bash
chmod +x scripts/cleanup.sh
./scripts/cleanup.sh
```

## 11. Verification Checklist

After cleanup, verify:
- [ ] No references to old API endpoints
- [ ] No references to Ceramic/DID sessions
- [ ] No unused dependencies in package.json
- [ ] No dead code (run `yarn find-deadcode`)
- [ ] All TypeScript errors resolved
- [ ] App builds successfully (`yarn build`)
- [ ] All tests pass (`yarn test`)
- [ ] No console errors in browser
- [ ] localStorage is clean

## 12. What to Keep

Do NOT remove:
- Working authentication code
- Graph visualization (cytoscape)
- Core UI components (Navbar, Modal, Form)
- MUI theme setup
- Router configuration
- Working utility functions

## Summary

This cleanup will:
1. Remove ~30% of the codebase (dead code)
2. Reduce bundle size significantly
3. Improve TypeScript coverage
4. Make the codebase easier to maintain
5. Remove confusion from mixed old/new patterns

Run the cleanup incrementally and test after each major deletion to ensure nothing breaks.
