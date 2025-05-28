# LinkedTrust Ecosystem Integration Guide

## Overview

This guide outlines the integration strategies for connecting various credential and claim projects with the LinkedTrust ecosystem. LinkedTrust provides a unified API for storing, discovering, and linking credentials and claims through persistent URIs.

## Core Architecture Principles

1. **Addressability**: Every credential and claim gets a persistent URI
2. **Linkability**: Claims can reference other claims and credentials
3. **Schema Flexibility**: Support for multiple credential formats (OBV3, Blockcerts, etc.)
4. **Display Hints**: Metadata guides proper UI rendering without backend changes

## Integration Projects

### 1. linked-Creds-author → LinkedTrust

**Purpose**: Publish OpenBadge v3 credentials created in linked-Creds-author to LinkedTrust.

**Implementation**:

```typescript
// In linked-Creds-author project
import { LinkedTrustPublisher } from './utils/linkedTrustPublisher';

const publisher = new LinkedTrustPublisher({
  apiUrl: 'https://live.linkedtrust.us/api',
  apiKey: process.env.LINKEDTRUST_API_KEY
});

// After creating credential
const result = await publisher.publishCredential(signedVC, {
  tags: ['achievement', 'skill'],
  visibility: 'public'
});

console.log('Credential URI:', result.uri);
```

**Key Features**:
- Automatic schema detection for OpenBadges
- Display hints for proper rendering
- Returns persistent URI for the credential
- Creates HAS claim linking subject to credential

### 2. linked-claims-extractor → LinkedTrust

**Purpose**: Extract impact measurements from reports and publish as LinkedClaims.

**Implementation**:

```python
from impact_claims_client import EnhancedLinkedTrustClient

client = EnhancedLinkedTrustClient()

# Submit impact measurement
response = client.submit_impact_claim(
    subject="Organization Name",
    statement="Provided clean water access",
    amount=50000,
    unit="people",
    aspect="impact:social:water",
    source_uri="https://org.com/report.pdf"
)
```

**Key Features**:
- Uses `amt`/`unit` fields for quantifiable impacts
- Hierarchical aspect categorization
- Source URI for provenance tracking
- Future-ready for ImpactCredential format

### 3. web-of-trust-ui → LinkedTrust Display

**Purpose**: Display LinkedTrust credentials with proper rendering based on type and metadata.

**Implementation**:

```tsx
import { LinkedTrustCredentialDisplay } from '@web-of-trust/ui';

// Display credential by URI
<LinkedTrustCredentialDisplay 
  uri="https://linkedtrust.us/credentials/123"
  showRelatedClaims={true}
/>

// Or provide credential directly
<LinkedTrustCredentialDisplay 
  credential={credentialData}
  onShare={handleShare}
/>
```

**Key Features**:
- Automatic credential fetching by URI
- Type-specific rendering (OpenBadges, Blockcerts, etc.)
- Related claims display
- Share and export functionality

## API Endpoints

### Submit Credential
```
POST /api/credentials
{
  "credential": { /* W3C VC */ },
  "schema": "OpenBadges",
  "metadata": {
    "displayHints": { /* UI hints */ },
    "tags": ["skill", "achievement"],
    "visibility": "public"
  }
}
```

### Retrieve Credential
```
GET /api/credentials/{uri}

Response:
{
  "credential": { /* stored credential */ },
  "relatedClaims": [ /* claims about this credential */ ]
}
```

## Display Hints System

Display hints enable proper UI rendering without backend changes:

```json
{
  "displayHints": {
    "primaryDisplay": "achievement.name",
    "imageField": "achievement.image", 
    "badgeType": "achievement",
    "showSkills": true,
    "showCriteria": true
  }
}
```

## URI Structure

LinkedTrust generates persistent URIs:
- Credentials: `https://linkedtrust.us/credentials/{id}`
- Claims: `https://linkedtrust.us/claims/{id}`
- Fallback: `urn:credential:{hash}` if no ID provided

## Trust Graph Benefits

By publishing to LinkedTrust, your credentials and claims become part of a navigable trust graph:

1. **Discovery**: Find credentials by subject, issuer, or type
2. **Validation**: See corroborating or conflicting claims
3. **Provenance**: Track information back to sources
4. **Reputation**: Build issuer and source credibility over time

## Migration Path

### For Existing Credentials

1. Submit existing credentials with appropriate schema identifier
2. LinkedTrust extracts LinkedClaims automatically
3. Original credential structure preserved
4. Receive persistent URI for future references

### For New Implementations

1. Use LinkedTrust API directly for credential storage
2. Leverage display hints for UI flexibility
3. Reference other claims/credentials by URI
4. Build on the trust graph incrementally

## Best Practices

1. **Always Include Schema**: Helps with proper rendering and discovery
2. **Use Display Hints**: Ensures credentials display correctly everywhere
3. **Reference by URI**: Build the web of trust through links
4. **Include Metadata**: Tags and visibility settings improve discovery
5. **Sign Claims**: Cryptographic signatures enable trust verification

## Future Roadmap

### ImpactCredentials
- Specialized format for certified impact measurements
- Third-party verification support
- Aggregation and comparison tools

### Enhanced Discovery
- GraphQL API for complex queries
- Semantic search across claims
- Trust path visualization

### Decentralized Storage
- IPFS integration for content
- Blockchain anchoring for proofs
- Federated LinkedTrust instances

## Support

- Documentation: https://linkedtrust.org/docs
- API Reference: https://linkedtrust.org/api
- GitHub: https://github.com/linkedtrust
