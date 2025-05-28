# Step-by-Step Implementation Instructions

## Prerequisites
- Ensure backend is running at https://dev.linkedtrust.us
- Have yarn installed
- Clone the frontend repo and run `yarn install`

## Step 1: Update API Transformation Layer

### 1.1 Update `/src/api/index.ts`

Replace the current transformBackendClaim function with comprehensive transformations:

```typescript
// Add at the top of the file
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
    object_image: entry.object?.image,
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
    // Legacy field mappings
    id: entry.id,
    source_link: entry.sourceURI,
    how_known: entry.howKnown,
    issuerId: entry.subject?.uri || entry.subject,
    issuerIdType: 'URL',
    createdDate: entry.effectiveDate
  }
}

// Update the getFeed function
export const getFeed = async (params?: {
  limit?: number
  search?: string
  nextPage?: string
}) => {
  const response = await axios.get<any>('/api/feed', { 
    params: {
      ...params,
      page: params?.nextPage || 1,
      limit: params?.limit || 50
    },
    timeout: 60000 
  })
  
  return {
    claims: response.data.entries.map(transformFeedEntry),
    nextPage: response.data.pagination?.page < response.data.pagination?.pages 
      ? String(response.data.pagination.page + 1) 
      : undefined,
    pagination: response.data.pagination
  }
}

// Update getClaimsBySubject
export const getClaimsBySubject = async (uri: string) => {
  const response = await axios.get<any>(`/api/claims/subject/${encodeURIComponent(uri)}`)
  return {
    claims: response.data.claims.map(transformBackendClaim),
    pagination: response.data.pagination
  }
}

// Update getClaimReport
export const getClaimReport = async (claimId: string) => {
  const response = await axios.get<any>(`/api/reports/claim/${claimId}`)
  const { claim, validations, validationSummary, relatedClaims, issuerReputation } = response.data
  
  return {
    claim: transformBackendClaim({
      ...claim,
      subject: claim.subjectEntity || { uri: claim.subject },
      object: claim.objectEntity || (claim.object ? { uri: claim.object } : null)
    }),
    validations: validations.map((v: any) => ({
      id: v.id,
      isValid: v.claim === 'AGREES_WITH' || v.claim === 'CONFIRMS',
      confidence: v.confidence,
      statement: v.statement,
      issuerName: v.issuerId,
      createdAt: v.effectiveDate
    })),
    summary: {
      totalValidations: validationSummary.total,
      averageConfidence: (validationSummary.agrees + validationSummary.confirms) / 
                        (validationSummary.total || 1),
      consensusValid: (validationSummary.agrees + validationSummary.confirms) > 
                      (validationSummary.disagrees + validationSummary.refutes)
    },
    relatedClaims: relatedClaims.map(transformBackendClaim),
    issuerReputation
  }
}

// Fix validation submission
export const submitValidation = async (claimId: string, validation: any) => {
  const payload = {
    validationType: validation.isValid ? 'agree' : 'disagree',
    confidence: validation.confidence,
    statement: validation.statement,
    evidence: validation.evidence
  }
  
  return axios.post<any>(`/api/reports/claim/${claimId}/validate`, payload)
}
```

### 1.2 Update `/src/api/types.ts`

Add missing fields to the Claim interface:

```typescript
export interface Claim {
  // ... existing fields ...
  object_name?: string
  object_type?: string
  object_image?: string
  // Ensure these pagination fields exist
  pagination?: {
    page: number
    limit: number
    total: number
    pages: number
  }
}
```

## Step 2: Fix Feed Component

### 2.1 Update `/src/containers/feedOfClaim/index.tsx`

Find the useEffect that loads the feed and update:

```typescript
useEffect(() => {
  const fetchFeed = async () => {
    try {
      setLoading(true)
      const response = await getFeed({
        limit: 50,
        nextPage: page?.toString()
      })
      
      if (page === 1) {
        setEntries(response.claims)
      } else {
        setEntries(prev => [...prev, ...response.claims])
      }
      
      // Handle pagination
      if (response.nextPage) {
        setHasMore(true)
      } else {
        setHasMore(false)
      }
    } catch (error) {
      console.error('Failed to load feed:', error)
      setError('Failed to load feed')
    } finally {
      setLoading(false)
    }
  }
  
  fetchFeed()
}, [page])
```

## Step 3: Update Create Claim Form

### 3.1 Update `/src/components/NewClaim/index.tsx` or form component

Add new fields to the form:

```typescript
// Add to form state
const [stars, setStars] = useState<number | undefined>()
const [score, setScore] = useState<number | undefined>()
const [amt, setAmt] = useState<number | undefined>()
const [unit, setUnit] = useState<string>('')
const [aspect, setAspect] = useState<string>('')

// Update submit handler
const handleSubmit = async (e: React.FormEvent) => {
  e.preventDefault()
  
  try {
    const claimData = {
      subject,
      claim: claimType,
      object,
      statement,
      confidence,
      howKnown,
      sourceURI,
      // New fields
      stars,
      score,
      amt,
      unit,
      aspect
    }
    
    const response = await createClaim(claimData)
    // Handle success
  } catch (error) {
    // Handle error - check if 401 unauthorized
    if (error.response?.status === 401) {
      alert('Please login to create claims')
      // Redirect to login
    }
  }
}
```

## Step 4: Add Basic Authentication UI

### 4.1 Create `/src/components/Auth/Login.tsx`

```typescript
import React, { useState } from 'react'
import { Box, TextField, Button, Paper, Typography } from '@mui/material'
import { handleAuth } from '../../utils/authUtils'
import axios from 'axios'

export const Login: React.FC = () => {
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [error, setError] = useState('')
  const [isSignup, setIsSignup] = useState(false)
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setError('')
    
    try {
      // For now, generate a mock JWT token
      // TODO: Replace with actual auth endpoint when available
      const mockToken = btoa(JSON.stringify({ 
        userId: email, 
        email, 
        exp: Date.now() + 86400000 
      }))
      
      handleAuth(mockToken, mockToken)
      window.location.href = '/'
    } catch (error) {
      setError('Login failed. Please try again.')
    }
  }
  
  return (
    <Box sx={{ maxWidth: 400, mx: 'auto', mt: 8 }}>
      <Paper sx={{ p: 4 }}>
        <Typography variant="h5" gutterBottom>
          {isSignup ? 'Sign Up' : 'Login'}
        </Typography>
        
        {error && (
          <Typography color="error" sx={{ mb: 2 }}>
            {error}
          </Typography>
        )}
        
        <form onSubmit={handleSubmit}>
          <TextField
            fullWidth
            label="Email"
            type="email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            margin="normal"
            required
          />
          
          <TextField
            fullWidth
            label="Password"
            type="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            margin="normal"
            required
          />
          
          <Button
            fullWidth
            variant="contained"
            type="submit"
            sx={{ mt: 3, mb: 2 }}
          >
            {isSignup ? 'Sign Up' : 'Login'}
          </Button>
          
          <Button
            fullWidth
            onClick={() => setIsSignup(!isSignup)}
          >
            {isSignup ? 'Already have an account? Login' : "Don't have an account? Sign up"}
          </Button>
        </form>
      </Paper>
    </Box>
  )
}
```

### 4.2 Add route in `/src/App.tsx`

```typescript
import { Login } from './components/Auth/Login'

// In your routes
<Route path="/login" element={<Login />} />
```

## Step 5: Update Graph Component

### 5.1 Update `/src/containers/Explore/index.tsx`

Update the graph data loading:

```typescript
useEffect(() => {
  const loadGraph = async () => {
    try {
      const response = await getGraph(selectedUri)
      
      // Transform nodes for cytoscape
      const cytoscapeNodes = response.data.nodes.map(node => ({
        data: {
          id: node.id,
          label: node.name || node.uri,
          uri: node.uri,
          entityType: node.entityType,
          ...node.entityData
        },
        classes: node.entityType?.toLowerCase() || 'default'
      }))
      
      // Transform edges
      const cytoscapeEdges = response.data.edges.map(edge => ({
        data: {
          id: edge.id,
          source: edge.source,
          target: edge.target,
          label: edge.label
        }
      }))
      
      // Update cytoscape
      cy.elements().remove()
      cy.add([...cytoscapeNodes, ...cytoscapeEdges])
      cy.layout({ name: 'cose' }).run()
    } catch (error) {
      console.error('Failed to load graph:', error)
    }
  }
  
  loadGraph()
}, [selectedUri])
```

## Step 6: Test Everything

### 6.1 Start the dev server
```bash
yarn dev
```

### 6.2 Test each feature
1. **Feed Loading**: Navigate to home page, should see claims
2. **Claim Details**: Click on a claim, should see details
3. **Create Claim**: Try to create a claim (should prompt for login)
4. **Login**: Use the login form (mock auth for now)
5. **Graph**: Navigate to explore, should see network visualization

## Common Issues and Fixes

### Issue 1: Feed not loading
- Check browser console for errors
- Verify backend is accessible
- Check CORS settings

### Issue 2: Authentication errors
- Clear localStorage
- Try logging in again
- Check auth headers in network tab

### Issue 3: Graph not rendering
- Ensure cytoscape is properly initialized
- Check for JavaScript errors
- Verify node/edge data format

## Next Steps

Once basic functionality is working:
1. Add proper authentication endpoint
2. Implement credential submission
3. Add entity filtering
4. Create user profile page
5. Add validation UI
6. Implement trending topics

## Debugging Tips

1. Keep browser DevTools open
2. Check Network tab for API calls
3. Use console.log liberally during development
4. Test with different data scenarios
5. Check for TypeScript errors in terminal

Remember: The goal is to get basic functionality working first, then enhance incrementally.
