# LinkedTrust Credential and Entity Integration Plan

## High-Level Overview

Enhance LinkedTrust to properly handle, store, and display different entity types (credentials, people, organizations, claims) by:

1. **Backend Processing**: Detect and process entity types when claims arrive
2. **Entity View Tables**: Store entity-specific data in dedicated view tables
3. **Enhanced Nodes**: Enrich nodes with entity type and view references
4. **Type-Aware Display**: Render entities based on their type

## Core Design Principles

- **No Generic Metadata**: Each entity type gets its own view table with specific fields
- **View Tables are Derived**: Claims remain the source of truth; views are projections
- **Direct References**: Nodes reference specific view tables by ID, not generic metadata
- **Early Processing**: Extract entity data as soon as claims arrive

## Phase 1: Database Schema

### 1.1 Entity View Tables

```sql
-- Credential view table
CREATE TABLE credential_view (
  id SERIAL PRIMARY KEY,
  node_id INTEGER REFERENCES nodes(id),
  claim_id INTEGER REFERENCES claims(id),
  credential_uri TEXT NOT NULL,
  canonical_uri TEXT UNIQUE,
  credential_type TEXT, -- 'OBV3', 'W3C_VC', 'CUSTOM'
  issuer TEXT,
  issuer_name TEXT,
  subject_uri TEXT,
  subject_name TEXT,
  achievement_name TEXT,
  achievement_description TEXT,
  issuance_date TIMESTAMP,
  expiration_date TIMESTAMP,
  image_url TEXT,
  criteria_narrative TEXT,
  evidence JSONB,
  raw_credential JSONB NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Person view table  
CREATE TABLE person_view (
  id SERIAL PRIMARY KEY,
  node_id INTEGER REFERENCES nodes(id) UNIQUE,
  uri TEXT NOT NULL UNIQUE,
  given_name TEXT,
  family_name TEXT,
  display_name TEXT,
  email TEXT,
  profile_image TEXT,
  bio TEXT,
  linkedin_url TEXT,
  github_url TEXT,
  website_url TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Organization view table
CREATE TABLE organization_view (
  id SERIAL PRIMARY KEY,
  node_id INTEGER REFERENCES nodes(id) UNIQUE,
  uri TEXT NOT NULL UNIQUE,
  legal_name TEXT,
  display_name TEXT,
  description TEXT,
  logo_url TEXT,
  website_url TEXT,
  industry TEXT,
  size_range TEXT,
  founded_date DATE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Update nodes table
ALTER TABLE nodes 
ADD COLUMN entity_type TEXT,
ADD COLUMN entity_view_id INTEGER,
ADD COLUMN entity_view_table TEXT;

-- Index for fast lookups
CREATE INDEX idx_nodes_entity_type ON nodes(entity_type);
CREATE INDEX idx_credential_view_canonical_uri ON credential_view(canonical_uri);
```

### 1.2 Prisma Schema Updates

```prisma
model CredentialView {
  id                    Int      @id @default(autoincrement())
  node                  Node?    @relation(fields: [nodeId], references: [id])
  nodeId                Int?     @unique
  claim                 Claim?   @relation(fields: [claimId], references: [id])
  claimId               Int?
  credentialUri         String   @map("credential_uri")
  canonicalUri          String?  @unique @map("canonical_uri")
  credentialType        String?  @map("credential_type")
  issuer                String?
  issuerName            String?  @map("issuer_name")
  subjectUri            String?  @map("subject_uri")
  subjectName           String?  @map("subject_name")
  achievementName       String?  @map("achievement_name")
  achievementDescription String? @map("achievement_description")
  issuanceDate          DateTime? @map("issuance_date")
  expirationDate        DateTime? @map("expiration_date")
  imageUrl              String?  @map("image_url")
  criteriaNarrative     String?  @map("criteria_narrative")
  evidence              Json?
  rawCredential         Json     @map("raw_credential")
  createdAt             DateTime @default(now()) @map("created_at")
  updatedAt             DateTime @updatedAt @map("updated_at")
  
  @@map("credential_view")
}

model PersonView {
  id           Int      @id @default(autoincrement())
  node         Node?    @relation(fields: [nodeId], references: [id])
  nodeId       Int?     @unique
  uri          String   @unique
  givenName    String?  @map("given_name")
  familyName   String?  @map("family_name")
  displayName  String?  @map("display_name")
  email        String?
  profileImage String?  @map("profile_image")
  bio          String?
  linkedinUrl  String?  @map("linkedin_url")
  githubUrl    String?  @map("github_url")
  websiteUrl   String?  @map("website_url")
  createdAt    DateTime @default(now()) @map("created_at")
  updatedAt    DateTime @updatedAt @map("updated_at")
  
  @@map("person_view")
}

// Update Node model
model Node {
  id              Int              @id @default(autoincrement())
  nodeUri         String
  name            String
  entType         EntityType
  descrip         String
  image           String?
  thumbnail       String?
  entityType      String?          @map("entity_type")
  entityViewId    Int?             @map("entity_view_id")
  entityViewTable String?          @map("entity_view_table")
  
  // Relations
  credentialView  CredentialView?
  personView      PersonView?
  organizationView OrganizationView?
  
  edgesTo         Edge[]           @relation("edgeTo")
  edgesFrom       Edge[]           @relation("edgeFrom")
}
```

## Phase 2: Backend Processing

### 2.1 Entity Type Detection Service

```typescript
// src/services/entityTypeDetector.ts
export class EntityTypeDetector {
  static detectFromClaim(claim: any): {
    entityType: string;
    confidence: number;
  } {
    // Check if it's a credential submission
    if (claim.credentialSubject && claim.proof && claim['@context']) {
      return { entityType: 'CREDENTIAL', confidence: 1.0 };
    }
    
    // Check claim patterns
    if (claim.claim === 'HAS' && claim.object) {
      const objectUri = claim.object.toLowerCase();
      if (objectUri.includes('credential') || objectUri.includes('badge')) {
        return { entityType: 'CREDENTIAL', confidence: 0.8 };
      }
    }
    
    // Check URI patterns
    if (claim.subject) {
      const patterns = [
        { pattern: /linkedin\.com\/in\//, type: 'PERSON' },
        { pattern: /github\.com\/(?!orgs\/)/, type: 'PERSON' },
        { pattern: /twitter\.com\//, type: 'PERSON' },
        { pattern: /\.(com|org|net|io)\/?$/, type: 'ORGANIZATION' },
        { pattern: /company|corp|inc|llc/i, type: 'ORGANIZATION' }
      ];
      
      for (const { pattern, type } of patterns) {
        if (pattern.test(claim.subject)) {
          return { entityType: type, confidence: 0.7 };
        }
      }
    }
    
    return { entityType: 'UNKNOWN', confidence: 0 };
  }
}
```

### 2.2 Credential Processing Service

```typescript
// src/services/credentialProcessor.ts
export class CredentialProcessor {
  static async processCredential(claim: any): Promise<CredentialView> {
    // Extract credential data based on type
    const credentialType = this.detectCredentialType(claim);
    let extractedData: any;
    
    switch (credentialType) {
      case 'OBV3':
        extractedData = this.extractOBV3Data(claim);
        break;
      case 'W3C_VC':
        extractedData = this.extractW3CVCData(claim);
        break;
      default:
        extractedData = this.extractGenericCredentialData(claim);
    }
    
    // Generate canonical URI if needed
    const canonicalUri = claim.id || this.generateCanonicalUri(claim);
    
    // Store in credential view
    const credentialView = await prisma.credentialView.create({
      data: {
        credentialUri: claim.id || canonicalUri,
        canonicalUri,
        credentialType,
        ...extractedData,
        rawCredential: claim,
        claimId: claim.claimId
      }
    });
    
    return credentialView;
  }
  
  static generateCanonicalUri(credential: any): string {
    const hash = crypto.createHash('sha256')
      .update(JSON.stringify(credential))
      .digest('hex');
    return `urn:linkedtrust:credential:${hash}`;
  }
  
  static extractOBV3Data(badge: any) {
    return {
      issuer: badge.issuer?.name || badge.issuer,
      issuerName: badge.issuer?.name,
      subjectName: badge.credentialSubject?.name,
      achievementName: badge.achievement?.name,
      achievementDescription: badge.achievement?.description,
      issuanceDate: badge.issuanceDate,
      imageUrl: badge.achievement?.image?.id || badge.image,
      criteriaNarrative: badge.achievement?.criteria?.narrative
    };
  }
}
```

### 2.3 Updated Claim Processing Flow

```typescript
// In claim controller
export async function createClaim(req: Request, res: Response) {
  const claimData = req.body;
  
  // 1. Detect entity type
  const { entityType, confidence } = EntityTypeDetector.detectFromClaim(claimData);
  
  // 2. Process entity-specific data BEFORE creating claim
  let entityViewId: number | null = null;
  let entityViewTable: string | null = null;
  
  if (entityType === 'CREDENTIAL' && confidence > 0.5) {
    const credentialView = await CredentialProcessor.processCredential(claimData);
    entityViewId = credentialView.id;
    entityViewTable = 'credential_view';
  } else if (entityType === 'PERSON') {
    const personView = await PersonProcessor.processPerson(claimData);
    entityViewId = personView.id;
    entityViewTable = 'person_view';
  }
  
  // 3. Create claim with entity references
  const claim = await prisma.claim.create({
    data: {
      ...claimData,
      entityType,
      entityViewId,
      entityViewTable
    }
  });
  
  // 4. Trigger pipeline processing
  await triggerPipelineProcessing(claim.id);
  
  res.json({ claim, entityType });
}
```

## Phase 3: Pipeline Updates

### 3.1 Enhanced Node Creation

```python
# In claims_to_nodes/pipe.py
def create_node_with_entity(node_uri, raw_claim):
    """Create node and link to entity view if available"""
    
    # Get entity info from claim
    entity_type = raw_claim.get('entityType')
    entity_view_id = raw_claim.get('entityViewId')
    entity_view_table = raw_claim.get('entityViewTable')
    
    # Create base node
    node_data = {
        "nodeUri": node_uri,
        "name": extract_name(raw_claim),
        "entType": map_to_node_entity_type(entity_type),
        "descrip": make_description(raw_claim),
        "entity_type": entity_type,
        "entity_view_id": entity_view_id,
        "entity_view_table": entity_view_table
    }
    
    node_id = insert_node(node_data)
    
    # Update entity view with node reference
    if entity_view_id and entity_view_table:
        update_entity_view_node_id(entity_view_table, entity_view_id, node_id)
    
    return {"id": node_id, **node_data}
```

### 3.2 Validation Target Resolution

```python
def process_validation_claim(raw_claim):
    """Route validations to the appropriate target"""
    subject_uri = raw_claim['subject']
    
    # Check if validating a HAS claim
    subject_claim = get_claim_by_uri(subject_uri)
    if subject_claim and subject_claim['claim'] == 'HAS':
        # Get the credential node directly
        credential_uri = subject_claim['object']
        target_node = get_node_by_uri(credential_uri)
        edge_label = 'VALIDATES_CREDENTIAL'
    else:
        # Normal claim validation
        target_node = get_node_by_uri(subject_uri)
        edge_label = 'VALIDATES_CLAIM'
    
    # Create validation edge
    validation_node = create_validation_node(raw_claim)
    create_edge(validation_node, target_node, edge_label, raw_claim['id'])
```

## Phase 4: API Response Enhancement

### 4.1 Graph Builder with Entity Data

```typescript
// src/services/graphBuilder.ts
export class GraphBuilder {
  static async buildEnhancedGraph(nodeIds: number[]): Promise<EnhancedGraph> {
    // Get base nodes
    const nodes = await prisma.node.findMany({
      where: { id: { in: nodeIds } },
      include: {
        credentialView: true,
        personView: true,
        organizationView: true
      }
    });
    
    // Build enhanced nodes with entity data
    const enhancedNodes = nodes.map(node => ({
      ...node,
      entityData: this.getEntityData(node),
      displayHints: this.getDisplayHints(node.entityType || node.entType)
    }));
    
    return { nodes: enhancedNodes };
  }
  
  static getEntityData(node: Node): any {
    // Return the specific view data based on entity type
    if (node.credentialView) return node.credentialView;
    if (node.personView) return node.personView;
    if (node.organizationView) return node.organizationView;
    return null;
  }
  
  static getDisplayHints(entityType: string) {
    const hints: Record<string, any> = {
      CREDENTIAL: {
        primaryAction: 'viewCertificate',
        secondaryActions: ['validate', 'share'],
        iconType: 'certificate',
        colorScheme: 'credential'
      },
      PERSON: {
        primaryAction: 'viewProfile',
        secondaryActions: ['message', 'endorse'],
        iconType: 'person',
        colorScheme: 'person'
      }
      // ... more types
    };
    
    return hints[entityType] || hints.DEFAULT;
  }
}
```

## Phase 5: Frontend Implementation

### 5.1 Entity Type Components

```typescript
// components/entities/EntityRenderer.tsx
export const EntityRenderer: React.FC<{ node: EnhancedNode }> = ({ node }) => {
  const entityType = node.entityType || node.entType;
  
  switch (entityType) {
    case 'CREDENTIAL':
      return <CredentialCard node={node} credential={node.entityData} />;
    
    case 'PERSON':
      return <PersonCard node={node} person={node.entityData} />;
    
    case 'ORGANIZATION':
      return <OrganizationCard node={node} org={node.entityData} />;
    
    case 'CLAIM':
    default:
      return <ClaimCard node={node} />;
  }
};
```

### 5.2 Credential Display Component

```typescript
// components/entities/CredentialCard.tsx
export const CredentialCard: React.FC<Props> = ({ node, credential }) => {
  const handleViewCertificate = () => {
    // Navigate to certificate view with credential data
    navigate(`/certificate/${node.id}`, { 
      state: { credential, node } 
    });
  };
  
  const handleShareLinkedIn = () => {
    // Share URL points to the claim about having the credential
    const shareUrl = `https://linkedtrust.us/claims/${node.nodeUri}`;
    window.open(
      `https://www.linkedin.com/sharing/share-offsite/?url=${encodeURIComponent(shareUrl)}`,
      '_blank'
    );
  };
  
  return (
    <Card>
      <CardMedia
        component="img"
        height="140"
        image={credential.imageUrl || '/default-badge.png'}
        alt={credential.achievementName}
      />
      <CardContent>
        <Typography variant="h5">
          {credential.achievementName || 'Credential'}
        </Typography>
        <Typography variant="subtitle1" color="text.secondary">
          Issued by {credential.issuerName || credential.issuer}
        </Typography>
        <Typography variant="body2">
          {credential.achievementDescription}
        </Typography>
        
        <Box mt={2}>
          <Button onClick={handleViewCertificate}>
            View Certificate
          </Button>
          <Button onClick={handleShareLinkedIn}>
            Share on LinkedIn
          </Button>
        </Box>
      </CardContent>
    </Card>
  );
};
```

## Phase 6: Credential Resolution

### 6.1 Credential Resolver Endpoint

```typescript
// New endpoint for resolving credentials by URN
app.get('/api/credentials/:urnOrId', async (req, res) => {
  const { urnOrId } = req.params;
  
  // Try to find credential
  const credential = await prisma.credentialView.findFirst({
    where: {
      OR: [
        { canonicalUri: urnOrId },
        { credentialUri: urnOrId }
      ]
    },
    include: {
      node: true,
      claim: true
    }
  });
  
  if (!credential) {
    return res.status(404).json({ error: 'Credential not found' });
  }
  
  res.json({ credential });
});
```

## Implementation Timeline

### Week 1: Foundation
- Database schema and migrations
- Entity detection service
- Basic credential processing

### Week 2: Backend Integration
- Update claim processing flow
- Pipeline modifications
- Entity view population

### Week 3: API and Frontend
- Enhanced graph responses
- Entity type components
- Credential display

### Week 4: Features and Polish
- Certificate generation
- LinkedIn sharing
- Testing and documentation

## Key Benefits

1. **Clean Separation**: Each entity type has its own view table with specific fields
2. **No Ambiguity**: No generic metadata field that becomes a dumping ground
3. **Type Safety**: Frontend knows exactly what fields to expect for each entity type
4. **Performance**: Direct table joins instead of JSON queries
5. **Extensibility**: Easy to add new entity types with their own view tables
