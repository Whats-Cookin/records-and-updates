# Credential User Experience Design

## Route Structure

```
/credential/:uri         - Main credential view (singular, shareable)
/c/:uri                 - Short URL that redirects to /credential/:uri
/credentials            - Browse all credentials
/profile/:id/credentials - User's credential collection
```

## User Actions from Credential Certificate View

### 1. **Primary Actions** (Always Visible)
- **Export to PDF** - Download for resume/portfolio
- **Share** 
  - Copy link
  - Share on LinkedIn with pre-filled message
  - Share via email
  - QR code (for conferences/networking)

### 2. **Navigation Actions**
- **View Issuer Profile** - Click issuer name → `/profile/{issuer}`
- **View Recipient Profile** - Click recipient name → `/profile/{recipient}`
- **Explore Similar** - "More {skill} credentials" → `/credentials?skill={skill}`

### 3. **Trust Actions**
- **Endorse** - Add your endorsement (if logged in)
- **Validate** - Claim you've verified this credential
- **Report Issue** - Flag if something seems wrong

### 4. **Social Proof**
- **View Endorsements** - See who has endorsed
- **View in Trust Graph** - Visualize connections
- **Check Verification Status** - See blockchain/signature proof

### 5. **Collection Actions** (if logged in)
- **Save to Collection** - Add to your credential bookmarks
- **Add to Resume** - Include in your LinkedTrust resume
- **Request Similar** - "I want this credential too"

## Enhanced Certificate View Layout

```
┌─────────────────────────────────────┐
│          [Badge/Logo]               │
│                                     │
│      Certificate of Achievement     │
│        VERIFIED CREDENTIAL          │
│                                     │
│         Jane Smith                  │
│    Python Development Expert        │
│                                     │
│  [Description of achievement...]    │
│                                     │
│  Skills: [Python] [Django] [APIs]   │
│                                     │
│  ✓ Issued by: Code Academy         │
│    January 15, 2024                 │
│                                     │
│  ┌─────────────────────────┐       │
│  │ Endorsed by 5 people    │       │
│  │ [+] [+] [+] [+] [+]     │       │
│  └─────────────────────────┘       │
│                                     │
│  [Export PDF] [Share]               │
└─────────────────────────────────────┘

        Additional Actions Bar:
┌─────────────────────────────────────┐
│ [👤 View Recipient] [🏢 View Issuer]│
│ [✓ Endorse] [🔍 Verify] [📊 Graph] │
│ [💾 Save] [🔗 Similar Credentials]  │
└─────────────────────────────────────┘
```

## User Flow Examples

### Flow 1: From Feed to Certificate
1. User browses main feed
2. Sees "Alice earned Python Developer certificate"
3. Clicks credential card
4. Views full certificate at `/credential/urn:credential:abc123`
5. Shares on LinkedIn
6. Explores similar Python credentials

### Flow 2: From Profile
1. User visits `/profile/alice`
2. Clicks "Credentials" tab
3. Sees grid of credential cards
4. Clicks one to view
5. Endorses the credential
6. Views issuer's other credentials

### Flow 3: Direct Share
1. Alice shares link: `linkedtrust.us/c/abc123`
2. Recipient opens link
3. Sees beautiful certificate
4. Can verify authenticity
5. Can view Alice's profile
6. Can explore similar credentials

## Implementation Updates Needed

### 1. Update CredentialCertificate Component
Add action buttons below the certificate:
- View Recipient Profile
- View Issuer Profile  
- Endorse This Credential
- View Trust Graph
- Find Similar

### 2. Add Endorsement UI
- Show endorser avatars/names
- Click to add endorsement
- Show endorsement count

### 3. Add Verification Badge
- Show "Verified" with checkmark
- Click for verification details
- Show blockchain proof if available

### 4. Social Sharing Enhancements
- Pre-filled LinkedIn message
- Twitter/X share option
- Generate QR code
- Email template

### 5. Navigation Context
- Breadcrumbs: Home > Credentials > Python Developer
- Related credentials sidebar
- "Back to profile" if came from profile

## Benefits of This Design

1. **Shareable** - Single URL for resume, social media
2. **Discoverable** - Multiple paths to find credentials  
3. **Verifiable** - Clear trust signals and verification
4. **Actionable** - Users can endorse, share, explore
5. **Connected** - Links to profiles, similar content
6. **Professional** - PDF export for real-world use

## Route Configuration

```typescript
// In App.tsx
<Routes>
  {/* Main credential route */}
  <Route path="/credential/:uri" element={<CredentialView />} />
  
  {/* Short URL redirect */}
  <Route path="/c/:uri" element={<Navigate to={`/credential/${params.uri}`} />} />
  
  {/* Browse credentials */}
  <Route path="/credentials" element={<CredentialBrowse />} />
  
  {/* User's credentials */}
  <Route path="/profile/:userId/credentials" element={<UserCredentials />} />
  
  {/* Legacy redirect */}
  <Route path="/credentials/:uri" element={<Navigate to={`/credential/${params.uri}`} />} />
</Routes>
```

This design makes credentials:
- Easy to share (short URLs)
- Easy to discover (multiple entry points)
- Valuable to display (beautiful certificate)
- Actionable (endorse, verify, explore)
- Connected to the trust graph