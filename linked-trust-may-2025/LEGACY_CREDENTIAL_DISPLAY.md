# Legacy Credential Display Components

## Key Display Elements from Legacy Frontend

### 1. Badge Component (Badge.tsx)
Used to display claim types with distinct visual styling:

```tsx
// Badge colors and icons based on claim type:
- 'credential': Blue background (#cce6ff), Medal icon
- 'validated': Yellow background (#f8e8cc), ShieldCheck icon  
- Other: Green background (#c0efd7), CircleCheck icon

// Displays as a rounded pill with icon + label
```

### 2. Credential Popup/Details (CredentialDetails/index.tsx)
Dark themed card display for credentials:

```tsx
// Key features:
- Dark background (#1e1e2d) with white text
- Verified icon (green checkmark) next to subject
- Truncated issuer ID display
- Expandable description with "Read more/less" toggle
- "View Full Details" and "Share Credential" buttons
- Issued date formatting
```

### 3. Feed Display (feedOfClaim/index.tsx)
How credentials appear in the main feed:

```tsx
// Each claim card includes:
- Subject name with external link icon
- Badge component showing claim type
- Statement/description text
- Star ratings (if applicable)
- Action buttons: Validate, Evidence, Graph
- Metadata in dropdown menu (confidence, aspect, etc)
```

## Key Data Fields Used in Display

From the legacy display components, these fields were shown:

1. **Primary Display**:
   - `subject` - The credential subject/recipient
   - `name` - Credential name/title
   - `statement` - Description text
   - `effectiveDate` - Issued date
   - `issuerId` - Who issued it

2. **Secondary/Menu Items**:
   - `source_link` - Link to source
   - `how_known` - How the claim was verified
   - `confidence` - Confidence score
   - `stars` - Star rating (1-5)
   - `score` - Numeric score
   - `amt` - Amount/value
   - `aspect` - What aspect is being claimed

3. **Visual Elements**:
   - Badge/pill to show claim type
   - Verified checkmark for credentials
   - Star ratings visualization
   - Expandable text for long descriptions

## Achievement-Specific Considerations

While the legacy code doesn't have specific OpenBadges rendering, to support achievement display we would need:

1. **Badge Image**: Display the badge/achievement icon prominently
2. **Achievement Name**: The title of the achievement
3. **Criteria**: What was required to earn it
4. **Issuer Info**: Organization name and logo
5. **Evidence**: Links to work that earned the badge
6. **Skills/Competencies**: What skills it represents

## Recommendations for New Implementation

1. **Detect credential type** from context/type fields
2. **For OpenBadges/Achievements**:
   - Extract and display badge image from `credentialSubject.achievement.image`
   - Show achievement name prominently
   - Display criteria and description
   - Show issuer organization with logo if available

3. **Keep the visual styling**:
   - Dark card theme for credential details
   - Badge/pill component for credential type
   - Verified checkmark
   - Expandable descriptions
   - Action buttons for viewing/sharing

4. **Data mapping**:
   - Map OpenBadges fields to display fields
   - `achievement.name` → primary title
   - `achievement.description` → statement
   - `achievement.criteria` → additional info
   - `achievement.image` → visual badge
