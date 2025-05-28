# LinkedTrust Frontend Cleanup Plan

## Current Status
✅ All API endpoints updated to new backend
✅ TypeScript build errors fixed
✅ Entity system implemented
✅ Core functionality working

## Recommended Cleanups (Priority Order)

### 1. Remove Dead Code
- [ ] Remove all Ceramic/ComposeDB code if not being used
  - `/src/composedb/` directory
  - References in useCreateClaim.ts
- [ ] Remove duplicate type definitions
  - Consolidate Claim types (multiple definitions exist)
  - Remove unused interfaces in feedOfClaim/types.ts
- [ ] Remove .DS_Store files and add to .gitignore

### 2. Type System Cleanup
- [ ] Remove the legacy fields from API types once confirmed not needed
- [ ] Create proper type guards instead of using `any` casts
- [ ] Fix all TypeScript strict mode issues
- [ ] Remove duplicate LocalClaim interfaces

### 3. API Layer Improvements
- [ ] Add proper error handling to all API calls
- [ ] Implement retry logic for failed requests
- [ ] Add request/response interceptors for common transformations
- [ ] Create a proper error boundary for API failures

### 4. Component Modernization
- [ ] Convert class components to functional components (if any remain)
- [ ] Replace deprecated MUI components
- [ ] Implement proper loading states (not just spinners)
- [ ] Add skeleton screens for better UX

### 5. Performance Optimizations
- [ ] Implement proper memoization for expensive computations
- [ ] Add React.lazy() for code splitting on routes
- [ ] Optimize graph rendering (virtualization for large graphs)
- [ ] Implement proper caching strategy

### 6. Code Organization
- [ ] Move all API-related types to /src/api/types/
- [ ] Create proper feature folders with colocated components
- [ ] Extract common UI patterns to reusable components
- [ ] Standardize file naming (some use PascalCase, some camelCase)

### 7. State Management
- [ ] Consider adding Zustand or Redux Toolkit for global state
- [ ] Remove prop drilling (lots of props passed through multiple levels)
- [ ] Implement proper data fetching patterns (React Query or SWR)

### 8. UI/UX Improvements
- [ ] Add proper empty states for all lists
- [ ] Implement better error messages (user-friendly)
- [ ] Add loading skeletons instead of just spinners
- [ ] Fix responsive design issues (some components break on mobile)
- [ ] Add dark mode support properly (currently incomplete)

### 9. Testing Infrastructure
- [ ] Add unit tests for API service layer
- [ ] Add integration tests for critical flows
- [ ] Set up E2E tests with Playwright or Cypress
- [ ] Add visual regression tests

### 10. Developer Experience
- [ ] Add proper ESLint rules
- [ ] Set up Prettier with consistent formatting
- [ ] Add pre-commit hooks with Husky
- [ ] Create proper documentation for new developers
- [ ] Add Storybook for component development

## Quick Wins (Do These First)
1. Delete unused Ceramic code
2. Fix TypeScript strict issues
3. Add .DS_Store to .gitignore
4. Consolidate duplicate types
5. Add proper error boundaries

## Technical Debt Items
1. **Feed Component** - 600+ lines, needs breaking up
2. **Graph Utils** - Complex parsing logic needs refactoring
3. **API Types** - Mixed legacy and new fields
4. **Error Handling** - Inconsistent across components
5. **Authentication** - Auth utils could be cleaner

## Files Needing Major Refactoring
- `/src/containers/feedOfClaim/index.tsx` - Too large, doing too much
- `/src/containers/Explore/index.tsx` - Complex graph logic mixed with UI
- `/src/components/ClaimReport/index.tsx` - Nested components, poor separation
- `/src/types/` - Multiple conflicting type definitions

## New Features to Consider
- [ ] Real-time updates with WebSockets
- [ ] Offline support with service workers
- [ ] Progressive Web App capabilities
- [ ] Better search with filters
- [ ] Bulk operations on claims
- [ ] Export functionality (CSV, JSON)
- [ ] Advanced graph visualization options

## Migration Risks
- Some components still use old field names internally
- Mixed use of API types and local types
- Inconsistent null handling
- Some components may break with strict TypeScript

## For Next Developer
1. Start with "Quick Wins" section
2. Focus on one feature area at a time
3. Don't try to refactor everything at once
4. Keep backwards compatibility during transition
5. Test thoroughly after each change

## Environment Setup Notes
- Node version: Use .nvmrc file
- Yarn for package management
- Vite for build tooling
- TypeScript strict mode should be enabled
- Dev server: http://localhost:5173

## Key Architecture Decisions Made
1. Centralized API service layer in /src/api/
2. Entity type system for better data modeling
3. Field name mappings handled in API layer
4. TypeScript types aligned with backend

Remember: The app is functional now, but needs cleanup for long-term maintainability.
