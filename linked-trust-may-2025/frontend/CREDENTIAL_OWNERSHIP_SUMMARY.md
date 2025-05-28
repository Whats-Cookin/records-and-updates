# Credential Ownership Implementation Summary

## What We Built

### 1. **Ownership Model**
- Users have URIs based on their auth method:
  - Google OAuth: `https://live.linkedtrust.us/userids/google/{googleId}`
  - MetaMask: `did:pkh:eip155:1:{address}`
  - Generic: `https://live.linkedtrust.us/users/{userId}`

### 2. **Claim Flow**
Users can claim credentials when:
- They arrive with a `claim_token` (from credential creation)
- They are the subject of the credential
- They have an `invite_token` (future: sent by issuer)

### 3. **Share Restrictions**
- Only users who have claimed a credential can share it on LinkedIn
- Export to PDF is always available (for viewing/reference)
- Unclaimed credentials show "Claim This Credential" button

### 4. **Issuer Actions**
Issuers can:
- Send credentials to recipients via email (with invitation link)
- See "Send Credential" button if they are the issuer
- Track if their issued credentials have been claimed

### 5. **UI Updates**
```
Unclaimed Credential (eligible user):
[Export PDF] [Claim This Credential] [Validate] [Request Validation]

Claimed Credential (owner):
[Export PDF] [Share] [Validate] [Request Validation]

Unclaimed Credential (issuer):
[Export PDF] [Send Credential] [Validate] [Request Validation]

Other Users:
[Export PDF] [Validate] [Request Validation]
```

## Code Changes

### 1. **CredentialCertificate Component**
- Added ownership detection logic
- Conditional button rendering based on user status
- Claim credential functionality
- Send credential modal for issuers

### 2. **useAuth Hook**
- Simple auth hook to get current user
- Extracts user info from JWT token
- Provides consistent user URI

### 3. **CredentialView Container**
- Passes currentUser to certificate component
- Passes credential URI for proper claim association

## API Requirements

### Backend Needs:
1. **Claim Token Validation**
   - Validate tokens when creating HAS claims
   - Single-use tokens that expire

2. **Offer Endpoint**
   - `POST /api/credentials/offer`
   - Send invitation emails with claim links

3. **User URI Consistency**
   - Ensure getUserUri() returns consistent format
   - Match the format used in frontend

## Integration Points

### 1. **linked-Creds-author → LinkedTrust**
- Redirect with claim_token after credential creation
- User can immediately claim their new credential

### 2. **Magic Links**
- `?claim_token=` - For credential creators
- `?invite_token=` - For invited recipients
- `?offer_token=` - For issuers to send

### 3. **Future Enhancements**
- Admin-generated invitation links
- Bulk credential distribution
- OAuth token passthrough to avoid re-login

## Security Model

1. **Only rightful owners can claim**
   - Must be authenticated
   - Must have valid token or be subject

2. **Only owners can share as theirs**
   - Prevents credential theft
   - LinkedIn posts are authentic

3. **Issuers maintain control**
   - Can send to intended recipients
   - Can track claim status

## Next Steps

1. **Test the flow end-to-end**
2. **Implement backend claim token validation**
3. **Add email sending for offers**
4. **Update linked-Creds-author to redirect with token**
5. **Add success/error notifications**

The implementation provides a secure, user-friendly way to handle credential ownership while maintaining the portability and openness of verifiable credentials.