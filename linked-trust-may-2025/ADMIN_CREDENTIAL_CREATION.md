# Admin Credential Creation & Enhanced Claims

## New Features Added

### 1. Admin Credential Creation Endpoint
`POST /api/credentials/admin/create`

Creates a credential that can be assigned to someone via email or link.

**Request Body:**
```json
{
  "recipientEmail": "alice@example.com",
  "recipientName": "Alice Smith",
  "achievementName": "Python Developer",
  "achievementDescription": "Proficient in Python development",
  "skills": ["Python", "Django", "REST APIs"],
  "criteria": "Completed Python bootcamp and built 3 projects",
  "validityPeriod": 365
}
```

**Response:**
```json
{
  "credential": { /* stored credential */ },
  "credentialUri": "urn:uuid:123",
  "inviteLink": "https://live.linkedtrust.us/credential/urn:uuid:123?invite_token=abc",
  "offerToken": "abc",
  "recipientEmail": "alice@example.com"
}
```

### 2. Credential Templates
`GET /api/credentials/templates`

Returns pre-defined templates:
- Skill Verification
- Course Completion
- Volunteer Recognition
- Employee Achievement

### 3. Enhanced Claim Creator UI

The `EnhancedClaimCreator` component supports creating different claim types:

#### Standard Claims:
- **Rated**: Rate/review with stars (1-5) and statement
- **Impact**: Quantified impact with amount/unit (e.g., "helped 1000 people")
- **Same As**: Link two URIs as equivalent
- **Endorses/Validates**: Endorse or validate another claim/credential
- **Custom**: Any custom predicate

#### Quick Credential Creation:
When selecting "Issue Credential", opens a dialog to create credentials with:
- Recipient info (name, optional email)
- Achievement details
- Skills list
- Criteria
- Validity period

## UI Components

### QuickCredentialDialog
Simple form for creating credentials without leaving the claim interface.

### EnhancedClaimCreator
Unified interface for creating all types of claims and credentials.

## Use Cases

### 1. Organization Issues Achievement
```
1. Admin opens claim creator
2. Selects "Issue Credential"
3. Fills in recipient and achievement details
4. Gets invitation link
5. Sends link to recipient
```

### 2. Quick Skill Verification
```
1. Select "Issue Credential" 
2. Choose "Skill Verification" template
3. Add specific skills
4. Generate invite link
```

### 3. Impact Reporting
```
1. Select "Impact" claim type
2. Enter amount and unit (e.g., "500 trees planted")
3. Add aspect (impact:environmental)
4. Submit claim
```

## Benefits

1. **Unified Interface**: Create claims and credentials from one place
2. **Templates**: Quick creation with pre-filled fields
3. **Flexible Assignment**: Email invites or shareable links
4. **No Wallet Required**: Recipients can claim with email/social login
5. **Trackable**: See if credentials have been claimed

## Next Steps

1. Add email sending for invitations
2. Add bulk credential creation
3. Add credential revocation
4. Add more templates (certifications, badges, etc.)
5. Add QR codes for in-person distribution