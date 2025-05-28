# linked-Creds-author to LinkedTrust Integration

## Overview

This integration allows users who create credentials in linked-Creds-author to be redirected to LinkedTrust where they can claim ownership of their credential.

## Flow

1. User creates credential in linked-Creds-author
2. Credential is published to LinkedTrust API
3. User is redirected to LinkedTrust with a claim token
4. User logs in to LinkedTrust (if not already)
5. User can claim the credential as theirs
6. Once claimed, they can share on LinkedIn

## Implementation in linked-Creds-author

### 1. Update the LinkedTrustPublisher

```typescript
// In linkedTrustPublisher.ts
export class LinkedTrustPublisher {
  async publishCredential(credential: any, metadata?: any): Promise<PublishResult> {
    try {
      // ... existing code ...
      
      const response = await axios.post(
        `${this.apiUrl}/credentials`,
        payload,
        { headers }
      );

      // Generate claim token (could be returned by API)
      const claimToken = response.data.claimToken || this.generateClaimToken();
      
      return {
        success: true,
        uri: response.data.uri,
        claim: response.data.claim,
        credential: response.data.credential,
        claimToken, // Add this
        redirectUrl: `https://live.linkedtrust.us/credential/${encodeURIComponent(response.data.uri)}?claim_token=${claimToken}`
      };
    } catch (error: any) {
      // ... error handling ...
    }
  }

  private generateClaimToken(): string {
    // Generate a random token
    return Math.random().toString(36).substring(2) + Date.now().toString(36);
  }
}
```

### 2. Update Credential Creation Flow

```typescript
// In your credential creation component
const handleCreateAndPublish = async () => {
  // Create and sign credential
  const signedVC = await createCredential(formData);
  
  // Publish to LinkedTrust
  const publisher = new LinkedTrustPublisher();
  const result = await publisher.publishCredential(signedVC);
  
  if (result.success && result.redirectUrl) {
    // Show success message
    setShowSuccess(true);
    setRedirectUrl(result.redirectUrl);
    
    // Redirect after 2 seconds
    setTimeout(() => {
      window.location.href = result.redirectUrl;
    }, 2000);
  }
};
```

### 3. Success Message Component

```tsx
{showSuccess && (
  <Box sx={{ 
    p: 3, 
    bgcolor: 'success.light', 
    borderRadius: 2,
    mb: 2 
  }}>
    <Typography variant="h6" gutterBottom>
      Credential Created Successfully!
    </Typography>
    <Typography variant="body1" gutterBottom>
      Your credential has been published to LinkedTrust.
    </Typography>
    <Typography variant="body2" color="text.secondary">
      Redirecting you to claim your credential...
    </Typography>
    <Link href={redirectUrl} sx={{ mt: 1, display: 'block' }}>
      Click here if not redirected automatically
    </Link>
  </Box>
)}
```

## Backend Support Needed

### 1. Claim Token Validation

The LinkedTrust backend should validate claim tokens when processing claims:

```typescript
// In claims API
export async function createClaim(req: AuthRequest, res: Response) {
  const { claimToken, ...claimData } = req.body;
  
  if (claimToken) {
    // Validate token matches credential
    const isValid = await validateClaimToken(claimToken, claimData.object);
    if (!isValid) {
      return res.status(403).json({ error: 'Invalid claim token' });
    }
  }
  
  // Process claim...
}
```

### 2. Credential Creation Response

When creating credentials, return a claim token:

```typescript
// In credentials API
export async function submitCredential(req: AuthRequest, res: Response) {
  // ... existing credential creation ...
  
  // Generate claim token for creator
  const claimToken = generateClaimToken();
  await storeClaimToken(claimToken, credential.id, req.user.id);
  
  res.json({ 
    credential: stored, 
    claim,
    uri: canonicalUri,
    claimToken // Add this
  });
}
```

## User Experience

### Success State in linked-Creds-author:
```
┌──────────────────────────────────┐
│  ✅ Credential Created!          │
│                                  │
│  Your Python Developer           │
│  credential has been published.  │
│                                  │
│  Redirecting to LinkedTrust...   │
│                                  │
│  [Click here if not redirected]  │
└──────────────────────────────────┘
```

### Landing in LinkedTrust:
```
┌──────────────────────────────────┐
│  Welcome! Please login to claim  │
│  your credential.                │
│                                  │
│  [Login with Google]             │
│  [Login with MetaMask]           │
└──────────────────────────────────┘
```

### After Login:
```
┌──────────────────────────────────┐
│     Python Developer             │
│   [Claim This Credential]        │
│                                  │
│  This is your credential.        │
│  Claim it to add to your        │
│  profile and share on LinkedIn.  │
└──────────────────────────────────┘
```

## Security Considerations

1. **Token Expiry**: Claim tokens should expire after 24 hours
2. **Single Use**: Each token should only be usable once
3. **Issuer Validation**: Only the credential creator should get a claim token
4. **HTTPS Required**: All redirects must use HTTPS

## Future Enhancements

1. **OAuth Passthrough**: Pass auth token to avoid re-login
2. **Embedded Claiming**: Claim via iframe without redirect
3. **Bulk Claims**: Claim multiple credentials at once
4. **Mobile App Links**: Deep link to mobile app