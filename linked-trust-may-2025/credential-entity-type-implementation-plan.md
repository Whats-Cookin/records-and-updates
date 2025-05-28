# LinkedTrust Entity Type System Implementation Plan

## Overview

This document outlines the implementation plan for extending LinkedTrust to properly handle different entity types (Credentials, People, Organizations, etc.) throughout the data pipeline. The goal is to create a flexible system that can recognize, store, and display various entity types with their rich metadata while maintaining the simple LinkedClaim semantic triple structure.

## Core Architecture Principles

1. **Early Processing**: Extract and store entity-specific data as soon as claims arrive (API or frontend)
2. **View Tables**: Create entity-specific view tables that are derived from claims but not the source of truth
3. **Rich Display**: Enable the frontend to render entities based on their type and attached metadata

## Phase 1: Database Schema Updates

### 1.1 Add Entity Type View Tables

```sql
-- Generic entity metadata that can be attached to any node
CREATE TABLE EntityMetadata (
  id SERIAL PRIMARY KEY,
  nodeId INT REFERENCES Node(id),
  entityType EntityType NOT NULL,
  metadata JSONB NOT NULL,
  createdAt TIMESTAMP DEFAULT NOW(),
  updatedAt TIMESTAMP DEFAULT NOW()
);

-- Person-specific view
CREATE TABLE PersonEntity (
  id SERIAL PRIMARY KEY,
  nodeId INT REFERENCES Node(id) UNIQUE,
  firstName VARCHAR(255),
  lastName VARCHAR(255),
  email VARCHAR(255),
  profileUrl VARCHAR(500),
  avatarUrl VARCHAR(500),
  bio TEXT,
  metadata JSONB,
  createdAt TIMESTAMP DEFAULT NOW(),
  updatedAt TIMESTAMP DEFAULT NOW()
);

-- Organization-specific view
CREATE TABLE OrganizationEntity (
  id SERIAL PRIMARY KEY,
  nodeId INT REFERENCES Node(id) UNIQUE,
  legalName VARCHAR(255),
  displayName VARCHAR(255),
  website VARCHAR(500),
  logoUrl VARCHAR(500),
  description TEXT,
  metadata JSONB,
  createdAt TIMESTAMP DEFAULT NOW(),
  updatedAt TIMESTAMP DEFAULT NOW()
);

-- Extend existing Credential table to link with nodes
ALTER TABLE Credential ADD COLUMN nodeId INT REFERENCES Node(id);
ALTER TABLE Credential ADD COLUMN extractedMetadata JSONB;
```

### 1.2 Update Prisma Schema

```prisma
model EntityMetadata {
  id         Int        @id @default(autoincrement())
  node       Node       @relation(fields: [nodeId], references: [id])
  nodeId     Int
  entityType EntityType
  metadata   Json
  createdAt  DateTime   @default(now())
  updatedAt  DateTime   @updatedAt
}

model PersonEntity {
  id         Int      @id @default(autoincrement())
  node       Node     @relation(fields: [nodeId], references: [id])
  nodeId     Int      @unique
  firstName  String?
  lastName   String?
  email      String?
  profileUrl String?
  avatarUrl  String?
  bio        String?
  metadata   Json?
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt
}

model OrganizationEntity {
  id          Int      @id @default(autoincrement())
  node        Node     @relation(fields: [nodeId], references: [id])
  nodeId      Int      @unique
  legalName   String?
  displayName String?
  website     String?
  logoUrl     String?
  description String?
  metadata    Json?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}

// Update Node model
model Node {
  id             Int                 @id @default(autoincrement())
  nodeUri        String
  name           String
  entType        EntityType
  descrip        String
  image          String?
  thumbnail      String?
  edgesTo        Edge[]              @relation("edgeTo")
  edgesFrom      Edge[]              @relation("edgeFrom")
  
  // Relations to entity views
  entityMetadata EntityMetadata?
  personEntity   PersonEntity?
  orgEntity      OrganizationEntity?
  credential     Credential?
}

// Update Credential model
model Credential {
  id                String    @id
  nodeId            Int?      @unique
  node              Node?     @relation(fields: [nodeId], references: [id])
  context           Json?
  type              Json?
  issuer            Json?
  issuanceDate      DateTime?
  expirationDate    DateTime?
  credentialSubject Json?
  proof             Json?
  sameAs            Json?
  extractedMetadata Json?     // Processed/normalized data
  createdAt         DateTime  @default(now())
  updatedAt         DateTime  @updatedAt
}
```

## Phase 2: Backend API Processing

### 2.1 Entity Detection and Processing Service

Create `src/services/entityProcessor.service.ts`:

```typescript
interface EntityProcessor {
  detectEntityType(claim: any): EntityType;
  extractEntityData(claim: any, entityType: EntityType): any;
  storeEntityData(claim: any, entityData: any, entityType: EntityType): Promise<void>;
}

class EntityProcessorService implements EntityProcessor {
  async processClaim(claim: any): Promise<{
    entityType: EntityType;
    entityData: any;
    credentialId?: string;
  }> {
    // 1. Detect entity type from claim
    const entityType = this.detectEntityType(claim);
    
    // 2. Extract entity-specific data
    const entityData = this.extractEntityData(claim, entityType);
    
    // 3. Store in appropriate tables
    await this.storeEntityData(claim, entityData, entityType);
    
    return { entityType, entityData };
  }
  
  detectEntityType(claim: any): EntityType {
    // Check if it's a credential
    if (claim.proof && claim.credentialSubject) {
      return EntityType.CREDENTIAL;
    }
    
    // Check object URI patterns
    if (claim.object) {
      if (claim.object.includes('linkedin.com/in/')) return EntityType.PERSON;
      if (claim.object.includes('credly.com/badges/')) return EntityType.CREDENTIAL;
      // Add more patterns
    }
    
    // Check claim type
    if (claim.claim === 'HAS' && claim.object?.includes('credential')) {
      return EntityType.CREDENTIAL;
    }
    
    return EntityType.UNKNOWN;
  }
  
  async extractCredentialData(claim: any): Promise<any> {
    const credentialData = {
      type: claim.type || ['VerifiableCredential'],
      issuer: claim.issuer,
      issuanceDate: claim.issuanceDate,
      credentialSubject: claim.credentialSubject,
      // Extract more fields
    };
    
    // Store in Credential table
    const credential = await prisma.credential.create({
      data: {
        id: claim.id || generateCredentialId(claim),
        ...credentialData
      }
    });
    
    return credential;
  }
}
```

### 2.2 Update Claim Processing Flow

Modify existing claim endpoints to use entity processor:

```typescript
// In claim.controller.ts or similar
async createClaim(req: Request, res: Response) {
  const claimData = req.body;
  
  // 1. Process entity data BEFORE creating claim
  const entityProcessor = new EntityProcessorService();
  const { entityType, entityData, credentialId } = await entityProcessor.processClaim(claimData);
  
  // 2. Enhance claim with entity metadata
  if (credentialId) {
    claimData.credentialId = credentialId;
    claimData.objectType = 'CREDENTIAL';
  }
  
  // 3. Create claim as usual
  const claim = await claimService.createClaim(claimData);
  
  // 4. Trigger pipeline processing
  await pipelineService.processClaimToNodes(claim.id);
  
  res.json({ claim, entityType, entityData });
}
```

## Phase 3: Pipeline Microservice Updates

### 3.1 Enhanced Node Creation

Update `claims_to_nodes/pipe.py`:

```python
def get_or_create_node_with_entity(node_uri, raw_claim, entity_type=None):
    """
    Create node and link to entity data if available
    """
    # Normal node creation
    node = get_or_create_node(node_uri, raw_claim)
    
    # Check if entity data exists
    if entity_type == 'CREDENTIAL' and raw_claim.get('credentialId'):
        # Link credential to node
        update_query = """
            UPDATE credential 
            SET "nodeId" = %s 
            WHERE id = %s
        """
        execute_query(update_query, (node['id'], raw_claim['credentialId']))
        
    # Store entity metadata
    if entity_type and entity_type != 'UNKNOWN':
        metadata = extract_entity_metadata(raw_claim, entity_type)
        insert_entity_metadata(node['id'], entity_type, metadata)
    
    return node
```

### 3.2 Validation Target Resolution

```python
def resolve_validation_target(raw_claim):
    """
    Determine what a validation actually validates
    """
    subject_uri = raw_claim['subject']
    
    # Check if subject is a HAS claim
    subject_claim = get_claim_by_uri(subject_uri)
    
    if subject_claim and subject_claim['claim'] == 'HAS':
        # Validation targets the object (credential), not the HAS claim
        return subject_claim['object'], 'VALIDATES_CREDENTIAL'
    
    return subject_uri, 'VALIDATES_CLAIM'
```

## Phase 4: API Response Enhancement

### 4.1 Graph Response with Entity Data

Create response builder that includes entity data:

```typescript
class GraphResponseBuilder {
  async buildEnhancedGraph(nodes: Node[], edges: Edge[]): Promise<EnhancedGraph> {
    // Load entity data for all nodes
    const nodeIds = nodes.map(n => n.id);
    
    const entityMetadata = await prisma.entityMetadata.findMany({
      where: { nodeId: { in: nodeIds } }
    });
    
    const credentials = await prisma.credential.findMany({
      where: { nodeId: { in: nodeIds } }
    });
    
    const people = await prisma.personEntity.findMany({
      where: { nodeId: { in: nodeIds } }
    });
    
    // Build enhanced nodes
    const enhancedNodes = nodes.map(node => {
      const metadata = entityMetadata.find(em => em.nodeId === node.id);
      const credential = credentials.find(c => c.nodeId === node.id);
      const person = people.find(p => p.nodeId === node.id);
      
      return {
        ...node,
        entityData: {
          type: node.entType,
          metadata: metadata?.metadata,
          credential,
          person,
          // Display hints
          displayType: this.getDisplayType(node.entType),
          actions: this.getAvailableActions(node.entType)
        }
      };
    });
    
    return { nodes: enhancedNodes, edges };
  }
}
```

## Phase 5: Frontend Display Integration

### 5.1 Entity-Aware Components

```typescript
// EntityRenderer.tsx
const EntityRenderer: React.FC<{ node: EnhancedNode }> = ({ node }) => {
  switch (node.entType) {
    case 'CREDENTIAL':
      return <CredentialView credential={node.entityData.credential} node={node} />;
    
    case 'PERSON':
      return <PersonProfile person={node.entityData.person} node={node} />;
    
    case 'ORGANIZATION':
      return <OrganizationView org={node.entityData.org} node={node} />;
    
    case 'CLAIM':
      return <ClaimView claim={node} />;
    
    default:
      return <GenericNodeView node={node} />;
  }
};
```

### 5.2 Credential Display with Certificate

```typescript
// CredentialView.tsx
const CredentialView: React.FC<{ credential: Credential; node: Node }> = ({ credential, node }) => {
  return (
    <Card>
      <CardHeader>
        <Typography variant="h5">{credential.type?.[1] || 'Credential'}</Typography>
        <Typography variant="subtitle1">Issued by {credential.issuer}</Typography>
      </CardHeader>
      <CardContent>
        {/* Rich credential display */}
        <CredentialDetails credential={credential} />
        
        {/* Actions */}
        <Box mt={2}>
          <Button onClick={() => generateCertificate(node.id)}>
            View Certificate
          </Button>
          <Button onClick={() => shareToLinkedIn(node.nodeUri)}>
            Share on LinkedIn
          </Button>
        </Box>
        
        {/* Related claims */}
        <RelatedClaims nodeId={node.id} />
      </CardContent>
    </Card>
  );
};
```

## Phase 6: Credential-Specific Features

### 6.1 Credential Resolution Service

```typescript
// For credentials without proper URIs
class CredentialResolver {
  static generateCanonicalUri(credential: any): string {
    if (credential.id && isValidUri(credential.id)) {
      return credential.id;
    }
    
    // Generate deterministic URN
    const urn = `urn:linkedtrust:credential:${hashCredential(credential)}`;
    
    // Return resolvable URL
    return `https://live.linkedtrust.us/credentials/${encodeURIComponent(urn)}`;
  }
  
  static async resolveCredential(urnOrId: string): Promise<Credential> {
    // Try to find by ID first
    let credential = await prisma.credential.findUnique({
      where: { id: urnOrId }
    });
    
    // Try by URN pattern
    if (!credential) {
      credential = await prisma.credential.findFirst({
        where: { 
          id: { contains: urnOrId }
        }
      });
    }
    
    return credential;
  }
}
```

### 6.2 LinkedIn Sharing URL Structure

```
https://linkedtrust.us/claims/{claimId}
  → Shows the LinkedClaim about having the credential
  → Displays rich credential view
  → Shows validations of the credential itself
  → Allows adding recommendations
```

## Implementation Timeline

### Week 1: Database and Backend Foundation
- Day 1-2: Database schema updates and migrations
- Day 3-4: Entity processor service implementation
- Day 5: Update claim creation endpoints

### Week 2: Pipeline and Data Flow
- Day 1-2: Pipeline updates for entity-aware processing
- Day 3-4: Validation target resolution
- Day 5: Testing and debugging

### Week 3: API and Frontend
- Day 1-2: Enhanced API responses with entity data
- Day 3-4: Frontend entity renderers
- Day 5: Credential-specific features

### Week 4: Polish and Extension
- Day 1-2: Certificate generation integration
- Day 3-4: LinkedIn sharing functionality
- Day 5: Documentation and cleanup

## Success Criteria

1. **Credentials display with rich formatting** when viewing claims
2. **Validations point to credentials directly**, not to HAS claims
3. **Entity types are detected automatically** from claim content
4. **Frontend renders different entity types** with appropriate UIs
5. **System is extensible** for new entity types without major refactoring

## Future Extensions

1. **Badge Display**: Special rendering for Open Badges
2. **Person Profiles**: Aggregate view of all claims about a person
3. **Organization Pages**: Company/org overview with all related claims
4. **Document Previews**: Rich previews for document-type nodes
5. **Event Timelines**: Special views for event-type entities

## Notes

- Entity view tables are **derived data**, not source of truth
- Claims remain the authoritative source
- Entity processing should be **idempotent** - can reprocess without issues
- Frontend should **gracefully degrade** if entity data is missing
- Keep the **LinkedClaim model simple** - complexity lives in the entity layer
