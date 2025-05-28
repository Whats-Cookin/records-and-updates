# LinkedTrust Implementation Plan - Simplified

## Overview
Implement entity type system for LinkedTrust while preserving Claims as immutable events and treating URIs as persistent identifiers.

## Key Principles
1. **Claims are immutable** - Never modify the Claim table
2. **URIs are persistent** - All relationships key off URIs, not node IDs
3. **Source vs Issuer** - Both become nodes in the graph
4. **Minimal changes** - Work with existing structures

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

### 1.3 Extend Credential Table
```sql
ALTER TABLE "Credential"
ADD COLUMN canonical_uri TEXT,
ADD COLUMN node_uri TEXT,
ADD COLUMN name TEXT,
ADD COLUMN credential_schema TEXT;

CREATE INDEX idx_credential_canonical_uri ON "Credential"(canonical_uri);
```

### 1.4 Drop ClaimData
```sql
DROP TABLE "ClaimData";
```

## Phase 2: Backend Services

### 2.1 Entity Detection Service
```typescript
// src/services/entityDetector.service.ts
export class EntityDetectorService {
  static async processClaimEntities(claim: any) {
    // Process subject, object, and source URIs
    if (claim.subject) await this.detectAndRegisterEntity(claim.subject, claim);
    if (claim.object) await this.detectAndRegisterEntity(claim.object, claim);
    
    // IMPORTANT: Sources become nodes too!
    if (claim.sourceURI && claim.sourceURI !== claim.issuerId) {
      await this.detectAndRegisterEntity(claim.sourceURI, claim);
    }
  }
  
  static async detectAndRegisterEntity(uri: string, context: any) {
    // Check if already registered
    const existing = await prisma.$queryRaw`
      SELECT * FROM uri_entities WHERE uri = ${uri}
    `;
    if (existing.length > 0) return;
    
    // Detect type and register
    const entityType = await this.detectEntityType(uri, context);
    if (entityType !== 'UNKNOWN') {
      await prisma.$executeRaw`
        INSERT INTO uri_entities (uri, entity_type, entity_table, entity_id, name)
        VALUES (${uri}, ${entityType}::"EntityType", ${this.getTableName(entityType)}, ${uri}, ${this.extractName(uri)})
        ON CONFLICT (uri) DO NOTHING
      `;
    }
  }
}
```

### 2.2 Credential Processing
```typescript
// Update createCredential endpoint
export const createCredential = async (req: Request, res: Response) => {
  const credentialData = req.body;
  const userId = (req as ModifiedRequest).userId;
  
  // Generate canonical URI
  const canonicalUri = credentialData.id || generateCanonicalUri(credentialData);
  
  // Store credential
  const credential = await prisma.credential.create({
    data: {
      id: canonicalUri,
      canonical_uri: canonicalUri,
      name: extractCredentialName(credentialData),
      credential_schema: detectCredentialSchema(credentialData),
      ...credentialData
    }
  });
  
  // Register as entity
  await prisma.$executeRaw`
    INSERT INTO uri_entities (uri, entity_type, entity_table, entity_id, name)
    VALUES (${canonicalUri}, 'CREDENTIAL'::"EntityType", 'Credential', ${canonicalUri}, ${credential.name})
  `;
  
  // Create claim about having credential
  const claim = await prisma.claim.create({
    data: {
      subject: credentialData.credentialSubject?.id || userId,
      claim: 'HAS',
      object: canonicalUri,
      statement: `Has credential: ${credential.name}`,
      issuerId: userId,
      sourceURI: credentialData.issuer?.id || canonicalUri,
      howKnown: 'VERIFIED_LOGIN',
      confidence: 1.0
    }
  });
  
  // Process entities for the claim
  await EntityDetectorService.processClaimEntities(claim);
  
  // Trigger pipeline
  await triggerPipeline(claim.id);
  
  return res.json({ credential, claim });
};
```

## Phase 3: Pipeline Updates

### 3.1 Entity-Aware Node Creation
```python
# In claims_to_nodes/pipe.py
def get_or_create_node(node_uri, raw_claim, new_node=None):
    # Check entity info
    entity_info = get_entity_info(node_uri)
    
    if entity_info:
        # Use entity data for node
        name = entity_info['name'] or extract_fallback_name(node_uri)
        ent_type = entity_info['entity_type']
    else:
        # Existing logic
        name = extract_fallback_name(node_uri)
        ent_type = infer_entity_type(node_uri)
    
    # Create/get node
    node = {
        "nodeUri": node_uri,
        "name": name,
        "entType": ent_type,
        "descrip": ""
    }
    
    return insert_or_get_node(node)

def get_entity_info(uri):
    query = "SELECT * FROM uri_entities WHERE uri = %s"
    result = execute_query(query, (uri,))
    return result[0] if result else None
```

### 3.2 Process Sources as Nodes
```python
def process_claim(raw_claim):
    # Existing subject/object processing...
    
    # IMPORTANT: Create source node if different from issuer
    source_uri = raw_claim.get("sourceURI")
    issuer_id = raw_claim.get("issuerId")
    
    if source_uri and source_uri != issuer_id:
        # Source becomes its own node
        source_node = get_or_create_node(source_uri, raw_claim)
        # This enables trust chain visualization!
```

## Phase 4: API Enhancement

### 4.1 Return Enhanced Nodes
```typescript
// In graph endpoints
const enhancedNodes = await Promise.all(
  nodes.map(async (node) => {
    const entity = await prisma.$queryRaw`
      SELECT * FROM uri_entities WHERE uri = ${node.nodeUri}
    `;
    
    let entityData = null;
    if (entity[0]?.entity_type === 'CREDENTIAL') {
      entityData = await prisma.credential.findFirst({
        where: { canonical_uri: node.nodeUri }
      });
    }
    
    return {
      ...node,
      entityType: entity[0]?.entity_type,
      entityData
    };
  })
);
```

## Phase 5: Data Migration

```sql
-- Populate uri_entities from existing data

-- Credentials (after adding canonical_uri)
INSERT INTO uri_entities (uri, entity_type, entity_table, entity_id, name)
SELECT 
  COALESCE(canonical_uri, id),
  'CREDENTIAL'::"EntityType",
  'Credential',
  id,
  COALESCE(name, 'Credential')
FROM "Credential"
ON CONFLICT (uri) DO NOTHING;

-- First-hand sources (issuer = source)
INSERT INTO uri_entities (uri, entity_type, entity_table, entity_id, name)
SELECT DISTINCT
  "sourceURI",
  'PERSON'::"EntityType",
  'person_entities',
  "sourceURI",
  SUBSTRING("sourceURI" FROM '[^/]+$')
FROM "Claim"
WHERE "sourceURI" = "issuerId"
  AND "sourceURI" IS NOT NULL
ON CONFLICT (uri) DO NOTHING;

-- Web documents
INSERT INTO uri_entities (uri, entity_type, entity_table, entity_id, name)
SELECT DISTINCT
  "sourceURI",
  'DOCUMENT'::"EntityType",
  'external_source',
  "sourceURI",
  SUBSTRING("sourceURI" FROM '[^/]+$')
FROM "Claim"
WHERE "sourceURI" != "issuerId"
  AND "sourceURI" LIKE 'http%'
ON CONFLICT (uri) DO NOTHING;
```

## Testing Focus

1. **Credential Flow**
   - Submit credential → Creates claim → Both subject and credential become nodes
   - Verify uri_entities populated correctly

2. **Source vs Issuer**
   - Create claim where source ≠ issuer
   - Verify BOTH become nodes
   - Verify can trace: issuer → claim → source

3. **URI Persistence**
   - Regenerate nodes
   - Verify entity data still attached via URIs

## Critical Points

1. **Don't modify Claims** - They're immutable history
2. **Sources are nodes** - Not just metadata
3. **URIs are the key** - Not node IDs which can change
4. **Keep it simple** - Minimal changes to existing structure

This simplified plan focuses on the core changes needed to support credentials and entity types while maintaining the system's integrity.
