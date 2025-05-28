# LinkedTrust Implementation Plan - Clean Slate

## Overview
Implement entity type system while dramatically simplifying the backend. Replace complex, messy code with clean, minimal implementation focused on core LinkedClaims concepts.

## Key Principles
1. **Claims are immutable events** - Simple semantic triples with provenance
2. **URIs are persistent identifiers** - All relationships key off URIs
3. **Source vs Issuer** - Both become nodes in the graph  
4. **Radical simplification** - Remove 70% of backend complexity

## Phase 1: Database Schema Updates

### 1.1 Add CREDENTIAL to EntityType
```sql
ALTER TYPE "EntityType" ADD VALUE 'CREDENTIAL';
```

### 1.2 Create URI to Entity Mapping Table
```sql
CREATE TABLE uri_entities (
  id SERIAL PRIMARY KEY,
  uri TEXT NOT NULL UNIQUE,
  entity_type "EntityType" NOT NULL,
  entity_table TEXT NOT NULL,
  entity_id TEXT NOT NULL,
  name TEXT,
  image TEXT,
  thumbnail TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  
  INDEX idx_uri_entities_type (entity_type),
  INDEX idx_uri_entities_lookup (entity_table, entity_id)
);
```

### 1.3 Minimal Credential Extension
```sql
-- Only what we need - no redundant node_uri
ALTER TABLE "Credential"
ADD COLUMN canonical_uri TEXT,
ADD COLUMN name TEXT,
ADD COLUMN credential_schema TEXT;

CREATE INDEX idx_credential_canonical_uri ON "Credential"(canonical_uri);
```

### 1.4 Drop ClaimData
```sql
DROP TABLE "ClaimData";
```

## Phase 2: Backend Replacement

### 2.1 New Clean Structure
```
src/
  api/
    claims.ts         // Claim CRUD
    credentials.ts    // Credential submission
    graph.ts         // Graph generation
    feed.ts          // Feed generation
    report.ts        // Claim report/validations
  services/
    entityDetector.ts // Simple URI → Entity type
    graphBuilder.ts   // Claims → Nodes/Edges
    pipelineTrigger.ts // Trigger Python pipeline
  lib/
    prisma.ts        // Single Prisma client
    auth.ts          // Simple auth helpers
  index.ts           // Express app setup
```

### 2.2 Core API Endpoints

```typescript
// api/claims.ts - Simple claim creation
export async function createClaim(req: Request, res: Response) {
  const { subject, claim, object, sourceURI, howKnown, confidence, statement } = req.body;
  const userId = req.user?.id || req.body.issuerId;
  
  // Create claim
  const newClaim = await prisma.claim.create({
    data: {
      subject,
      claim,
      object,
      sourceURI,
      howKnown,
      confidence,
      statement,
      issuerId: userId,
      issuerIdType: 'URL',
      effectiveDate: new Date()
    }
  });
  
  // Detect entities
  await EntityDetector.processClaimEntities(newClaim);
  
  // Trigger pipeline
  await PipelineTrigger.processClaim(newClaim.id);
  
  res.json({ claim: newClaim });
}

// api/credentials.ts - Credential submission
export async function submitCredential(req: Request, res: Response) {
  const credential = req.body;
  const userId = req.user?.id;
  
  // Store credential
  const canonicalUri = credential.id || generateHash(credential);
  const stored = await prisma.credential.create({
    data: {
      id: canonicalUri,
      canonical_uri: canonicalUri,
      name: extractName(credential),
      credential_schema: detectSchema(credential),
      context: credential.context,
      type: credential.type,
      issuer: credential.issuer,
      credentialSubject: credential.credentialSubject,
      proof: credential.proof
    }
  });
  
  // Register as entity
  await prisma.uri_entities.create({
    data: {
      uri: canonicalUri,
      entity_type: 'CREDENTIAL',
      entity_table: 'Credential',
      entity_id: canonicalUri,
      name: stored.name
    }
  });
  
  // Create HAS claim
  const claim = await prisma.claim.create({
    data: {
      subject: credential.credentialSubject?.id || userId,
      claim: 'HAS',
      object: canonicalUri,
      statement: `Has credential: ${stored.name}`,
      issuerId: userId,
      sourceURI: credential.issuer?.id || canonicalUri,
      howKnown: 'VERIFIED_LOGIN',
      confidence: 1.0
    }
  });
  
  await PipelineTrigger.processClaim(claim.id);
  
  res.json({ credential: stored, claim });
}

// api/graph.ts - Simple graph generation
export async function getGraph(req: Request, res: Response) {
  const { uri } = req.params;
  
  // Get all claims involving this URI
  const claims = await prisma.claim.findMany({
    where: {
      OR: [
        { subject: uri },
        { object: uri },
        { sourceURI: uri }
      ]
    }
  });
  
  // Get nodes for these claims
  const nodes = await prisma.node.findMany({
    where: {
      OR: [
        { nodeUri: uri },
        { edgesFrom: { some: { claim: { id: { in: claims.map(c => c.id) } } } } },
        { edgesTo: { some: { claim: { id: { in: claims.map(c => c.id) } } } } }
      ]
    },
    include: {
      edgesFrom: { include: { endNode: true, claim: true } },
      edgesTo: { include: { startNode: true, claim: true } }
    }
  });
  
  // Enhance with entity data
  const enhanced = await enhanceNodesWithEntities(nodes);
  
  res.json({ nodes: enhanced });
}

// api/feed.ts - Clean feed implementation
export async function getFeed(req: Request, res: Response) {
  const { page = 1, limit = 50 } = req.query;
  
  // Simple: recent claims with their edges
  const claims = await prisma.claim.findMany({
    where: {
      effectiveDate: { not: null },
      statement: { not: null }
    },
    orderBy: { effectiveDate: 'desc' },
    skip: (page - 1) * limit,
    take: limit,
    include: {
      edges: {
        include: {
          startNode: true,
          endNode: true
        }
      }
    }
  });
  
  // Transform to feed entries
  const entries = claims.map(claim => ({
    id: claim.id,
    subject: claim.edges[0]?.startNode,
    statement: claim.statement,
    claim: claim.claim,
    effectiveDate: claim.effectiveDate,
    confidence: claim.confidence,
    source: claim.sourceURI
  }));
  
  res.json({ entries, page, limit });
}

// api/report.ts - Claim report with validations
export async function getClaimReport(req: Request, res: Response) {
  const { claimId } = req.params;
  
  // Get the claim
  const claim = await prisma.claim.findUnique({
    where: { id: parseInt(claimId) },
    include: {
      edges: {
        include: {
          startNode: true,
          endNode: true
        }
      }
    }
  });
  
  if (!claim) return res.status(404).json({ error: 'Claim not found' });
  
  // Get validations (claims about this claim)
  const claimUri = `${process.env.BASE_URL}/claims/${claimId}`;
  const validations = await prisma.claim.findMany({
    where: { subject: claimUri },
    include: {
      edges: {
        include: {
          startNode: true,
          endNode: true
        }
      }
    }
  });
  
  // Get other claims about same subject
  const relatedClaims = await prisma.claim.findMany({
    where: { 
      subject: claim.subject,
      id: { not: claim.id }
    },
    include: {
      edges: {
        include: {
          startNode: true,
          endNode: true
        }
      }
    }
  });
  
  res.json({
    claim,
    validations,
    relatedClaims
  });
}
```

### 2.3 Simple Services

```typescript
// services/entityDetector.ts
export class EntityDetector {
  static async processClaimEntities(claim: Claim) {
    // Check each URI
    await this.detectEntity(claim.subject);
    if (claim.object) await this.detectEntity(claim.object);
    if (claim.sourceURI && claim.sourceURI !== claim.issuerId) {
      await this.detectEntity(claim.sourceURI);
    }
  }
  
  static async detectEntity(uri: string) {
    // Skip if already registered
    const existing = await prisma.uri_entities.findUnique({ where: { uri } });
    if (existing) return;
    
    // Simple detection
    let entityType: EntityType = 'UNKNOWN';
    let entityTable = 'unknown';
    
    // Check if it's a credential
    const credential = await prisma.credential.findFirst({
      where: { OR: [{ id: uri }, { canonical_uri: uri }] }
    });
    if (credential) {
      entityType = 'CREDENTIAL';
      entityTable = 'Credential';
    }
    // Simple patterns
    else if (uri.includes('did:person:') || uri.includes('linkedin.com/in/')) {
      entityType = 'PERSON';
      entityTable = 'person_entities';
    }
    else if (uri.startsWith('http')) {
      entityType = 'DOCUMENT';
      entityTable = 'documents';
    }
    
    // Register
    await prisma.uri_entities.create({
      data: {
        uri,
        entity_type: entityType,
        entity_table: entityTable,
        entity_id: uri,
        name: uri.split('/').pop() || uri
      }
    });
  }
}

// services/graphBuilder.ts  
export async function enhanceNodesWithEntities(nodes: Node[]) {
  const uris = nodes.map(n => n.nodeUri);
  
  // Get all entity info
  const entities = await prisma.uri_entities.findMany({
    where: { uri: { in: uris } }
  });
  
  // Create lookup map
  const entityMap = new Map(entities.map(e => [e.uri, e]));
  
  // Enhance nodes
  return Promise.all(nodes.map(async node => {
    const entity = entityMap.get(node.nodeUri);
    if (!entity) return node;
    
    // Get entity-specific data
    let entityData = null;
    if (entity.entity_type === 'CREDENTIAL') {
      entityData = await prisma.credential.findFirst({
        where: { canonical_uri: node.nodeUri }
      });
    }
    
    return {
      ...node,
      entityType: entity.entity_type,
      entityData,
      displayName: entity.name || node.name
    };
  }));
}
```

### 2.4 What We're Removing

**DELETE these files/folders:**
- `src/dao/` - Complex DAOs with raw SQL
- `src/controllers/` - Messy controllers  
- Multiple claim endpoints (keep just one)
- Complex feed versions (v1, v2, v3)
- Validation request tables (use claims)
- Complex report generation
- Image handling complexity

**KEEP:**
- Basic auth (simplify if needed)
- Prisma schema (it's good)
- Pipeline trigger concept

## Phase 3: Simplified Pipeline

Update Python pipeline to be simpler:

```python
# pipeline.py
def process_claim(claim_id):
    claim = get_claim(claim_id)
    
    # Create nodes for subject, object, source
    subject_node = create_node_from_uri(claim['subject'])
    object_node = create_node_from_uri(claim['object']) if claim['object'] else None
    
    # Important: source becomes a node too
    if claim['sourceURI'] and claim['sourceURI'] != claim['issuerId']:
        source_node = create_node_from_uri(claim['sourceURI'])
    
    # Create edges
    if object_node:
        create_edge(subject_node, object_node, claim['claim'], claim['id'])
    else:
        # Create claim node
        claim_node = create_claim_node(claim)
        create_edge(subject_node, claim_node, claim['claim'], claim['id'])

def create_node_from_uri(uri):
    # Check entity type
    entity = get_entity_info(uri)
    
    return {
        'nodeUri': uri,
        'name': entity['name'] if entity else uri.split('/')[-1],
        'entType': entity['entity_type'] if entity else 'UNKNOWN'
    }
```

## Phase 4: Frontend Updates

The frontend just needs to:
1. Handle enhanced nodes with entityType
2. Render credentials nicely when entityType = 'CREDENTIAL'
3. Show source → claim → subject relationships

## Implementation Order

1. **Day 1**: 
   - Database migrations
   - Start fresh backend structure
   - Basic claim/credential endpoints

2. **Day 2**:
   - Graph and feed endpoints  
   - Report endpoint
   - Entity detection service

3. **Day 3**:
   - Update pipeline
   - Test end-to-end
   - Frontend adjustments

## Benefits of This Approach

1. **70% less code** - Removing complex DAOs, redundant endpoints
2. **Clearer structure** - Separation of concerns
3. **Easier to maintain** - Simple Prisma queries instead of raw SQL
4. **Better performance** - Less complex queries
5. **True to LinkedClaims** - Focus on simple semantic triples

## What Success Looks Like

- Clean, minimal API
- Simple claim creation
- Credentials automatically become nodes
- Sources and issuers both visible in graph
- Fast, simple feed
- Clear claim reports with validations

Ready to start implementation when you are!
