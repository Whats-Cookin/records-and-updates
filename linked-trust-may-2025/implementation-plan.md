# LinkedTrust Implementation Plan

## Overview
Implement entity type system for LinkedTrust while preserving Claims as immutable events and treating URIs as persistent identifiers.

## Phase 1: Database Schema Updates

### 1.1 Add CREDENTIAL to Existing EntityType

```sql
-- Add missing entity type to existing enum
ALTER TYPE "EntityType" ADD VALUE 'CREDENTIAL';
```

### 1.2 Create URI to Entity Mapping Table

```sql
-- URI to entity mapping table
CREATE TABLE uri_entities (
  id SERIAL PRIMARY KEY,
  uri TEXT NOT NULL UNIQUE,
  entity_type "EntityType" NOT NULL,  -- Use existing enum
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

### 1.3 Extend Credential Table (Minimal)

```sql
-- Add minimal fields for linking and display
ALTER TABLE "Credential"
ADD COLUMN canonical_uri TEXT,
ADD COLUMN node_uri TEXT,
ADD COLUMN name TEXT,
ADD COLUMN credential_schema TEXT;

CREATE INDEX idx_credential_canonical_uri ON "Credential"(canonical_uri);
CREATE INDEX idx_credential_node_uri ON "Credential"(node_uri);
```

### 1.4 Create Person Entity Table

```sql
CREATE TABLE person_entities (
  id SERIAL PRIMARY KEY,
  uri TEXT NOT NULL UNIQUE,
  given_name TEXT,
  family_name TEXT,
  display_name TEXT,
  email TEXT,
  profile_image_url TEXT,
  bio TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

### 1.5 Drop Deprecated Tables

```sql
-- Remove ClaimData as it's being replaced by uri_entities
DROP TABLE "ClaimData";
```

## Phase 2: Backend Services (Day 1 Afternoon)

### 2.1 Entity Detection Service

Create `src/services/entityDetector.service.ts`:

```typescript
import { prisma } from "../db/prisma";

export class EntityDetectorService {
  /**
   * Process a claim to detect and register entity types
   */
  static async processClaimEntities(claim: any) {
    // Process subject URI
    if (claim.subject) {
      await this.detectAndRegisterEntity(claim.subject, claim);
    }
    
    // Process object URI
    if (claim.object) {
      await this.detectAndRegisterEntity(claim.object, claim);
    }
    
    // Process source URI (important - sources become nodes!)
    if (claim.sourceURI && claim.sourceURI !== claim.issuerId) {
      await this.detectAndRegisterEntity(claim.sourceURI, claim);
    }
  }
  
  /**
   * Detect entity type for a URI and register it
   */
  static async detectAndRegisterEntity(uri: string, context: any) {
    // Check if already registered
    const existing = await prisma.$queryRaw`
      SELECT * FROM uri_entities WHERE uri = ${uri}
    `;
    if (existing.length > 0) return existing[0];
    
    // Detect type
    const entityInfo = await this.detectEntityType(uri, context);
    
    // Register in uri_entities
    if (entityInfo.type !== 'UNKNOWN') {
      await prisma.$executeRaw`
        INSERT INTO uri_entities (uri, entity_type, entity_table, entity_id, name)
        VALUES (${uri}, ${entityInfo.type}::"EntityType", ${entityInfo.table}, ${entityInfo.id}, ${entityInfo.name})
        ON CONFLICT (uri) DO NOTHING
      `;
    }
    
    return entityInfo;
  }
  
  /**
   * Determine entity type from URI and context
   */
  static async detectEntityType(uri: string, context: any) {
    // Check Credential table
    if (context.claim === 'HAS' && context.object === uri) {
      const credential = await prisma.credential.findFirst({
        where: { 
          OR: [
            { id: uri },
            { canonical_uri: uri }
          ]
        }
      });
      if (credential) {
        return {
          type: 'CREDENTIAL',
          table: 'Credential',
          id: credential.id,
          name: credential.name || 'Credential'
        };
      }
    }
    
    // URI pattern detection
    if (uri.includes('linkedin.com/in/')) {
      return {
        type: 'PERSON',
        table: 'person_entities',
        id: uri,
        name: this.extractNameFromUri(uri)
      };
    }
    
    if (uri.includes('did:person:')) {
      return {
        type: 'PERSON',
        table: 'person_entities', 
        id: uri,
        name: uri
      };
    }
    
    // Check if it's a web document
    if (uri.startsWith('http://') || uri.startsWith('https://')) {
      return {
        type: 'DOCUMENT',
        table: 'external_source',
        id: uri,
        name: this.extractNameFromUri(uri)
      };
    }
    
    return {
      type: 'UNKNOWN',
      table: 'unknown',
      id: uri,
      name: uri
    };
  }
  
  static extractNameFromUri(uri: string): string {
    // Extract meaningful name from URI
    try {
      const url = new URL(uri);
      const pathParts = url.pathname.split('/').filter(p => p);
      if (pathParts.length > 0) {
        return pathParts[pathParts.length - 1].replace(/[-_]/g, ' ');
      }
      return url.hostname;
    } catch {
      return uri;
    }
  }
}
```

### 2.2 Credential Processing Service

Create `src/services/credentialProcessor.service.ts`:

```typescript
export class CredentialProcessorService {
  /**
   * Process credential submission
   */
  static async processCredential(credentialData: any, userId: string) {
    // 1. Generate canonical URI if needed
    const canonicalUri = credentialData.id || this.generateCanonicalUri(credentialData);
    
    // 2. Extract minimal display data
    const name = this.extractName(credentialData);
    const schema = this.detectCredentialSchema(credentialData);
    
    // 3. Store credential with minimal enrichment
    const credential = await prisma.credential.create({
      data: {
        id: canonicalUri,
        canonical_uri: canonicalUri,
        name,
        credential_schema: schema,
        context: credentialData.context || credentialData['@context'],
        type: credentialData.type,
        issuer: credentialData.issuer,
        issuanceDate: credentialData.issuanceDate,
        expirationDate: credentialData.expirationDate,
        credentialSubject: credentialData.credentialSubject,
        proof: credentialData.proof
      }
    });
    
    // 4. Register as entity
    await prisma.$executeRaw`
      INSERT INTO uri_entities (uri, entity_type, entity_table, entity_id, name, image)
      VALUES (
        ${canonicalUri}, 
        'CREDENTIAL'::"EntityType", 
        'Credential', 
        ${credential.id}, 
        ${name},
        ${this.extractImage(credentialData)}
      )
    `;
    
    // 5. Create LinkedClaim about having the credential
    const claim = await this.createCredentialClaim(credential, userId);
    
    return { credential, claim };
  }
  
  static extractName(credential: any): string {
    // Try various paths for name
    if (credential.name) return credential.name;
    if (credential.credentialSubject?.achievement?.[0]?.name) {
      return credential.credentialSubject.achievement[0].name;
    }
    if (credential.credentialSubject?.name) {
      return `${credential.credentialSubject.name}'s Credential`;
    }
    return 'Credential';
  }
  
  static extractImage(credential: any): string | null {
    // Try to find an image URL
    if (credential.image) return credential.image;
    if (credential.credentialSubject?.achievement?.[0]?.image?.id) {
      return credential.credentialSubject.achievement[0].image.id;
    }
    return null;
  }
  
  static detectCredentialSchema(credential: any): string {
    // Detect schema based on context and type
    const context = credential.context || credential['@context'];
    const type = Array.isArray(credential.type) ? credential.type : [credential.type];
    
    if (type.includes('OpenBadgeCredential')) return 'OBV3';
    if (type.includes('VerifiableCredential')) return 'W3C_VC';
    if (context?.includes('blockcerts')) return 'BLOCKCERTS';
    
    return 'UNKNOWN';
  }
  
  static async createCredentialClaim(credential: any, userId: string) {
    const claim = await prisma.claim.create({
      data: {
        subject: credential.credentialSubject?.id || userId,
        claim: 'HAS',
        object: credential.canonical_uri,
        statement: `Has credential: ${credential.name}`,
        issuerId: userId,
        issuerIdType: 'URL',
        sourceURI: credential.issuer?.id || credential.canonical_uri,
        howKnown: 'VERIFIED_LOGIN',
        confidence: 1.0,
        effectiveDate: credential.issuanceDate || new Date()
      }
    });
    
    // Process entities for this claim
    await EntityDetectorService.processClaimEntities(claim);
    
    // Trigger pipeline
    await this.triggerPipeline(claim.id);
    
    return claim;
  }
  
  static generateCanonicalUri(credential: any): string {
    const hash = crypto.createHash('sha256')
      .update(JSON.stringify(credential))
      .digest('hex')
      .substring(0, 16);
    return `urn:linkedtrust:credential:${hash}`;
  }
}
```

### 2.3 Enhanced Graph Service

Create `src/services/graphEnhancer.service.ts`:

```typescript
export class GraphEnhancerService {
  /**
   * Enhance nodes with entity data
   */
  static async enhanceNodes(nodes: any[]) {
    // Get all URIs
    const uris = nodes.map(n => n.nodeUri);
    
    // Fetch entity data
    const entities = await prisma.$queryRaw`
      SELECT * FROM uri_entities WHERE uri = ANY(${uris})
    `;
    
    // Create URI->entity map
    const entityMap = new Map(entities.map(e => [e.uri, e]));
    
    // Enhance each node
    const enhancedNodes = await Promise.all(
      nodes.map(async (node) => {
        const entity = entityMap.get(node.nodeUri);
        if (!entity) return node;
        
        // Fetch entity-specific data
        let entityData = null;
        if (entity.entity_type === 'CREDENTIAL') {
          entityData = await prisma.credential.findFirst({
            where: { 
              OR: [
                { id: entity.entity_id },
                { canonical_uri: node.nodeUri }
              ]
            }
          });
        } else if (entity.entity_type === 'PERSON') {
          entityData = await prisma.$queryRaw`
            SELECT * FROM person_entities WHERE uri = ${node.nodeUri}
          `;
        }
        
        return {
          ...node,
          entityType: entity.entity_type,
          entityData,
          displayName: entity.extracted_name || node.name,
          displayImage: entity.extracted_image || node.thumbnail
        };
      })
    );
    
    return enhancedNodes;
  }
}
```

## Phase 3: API Updates (Day 2 Morning)

### 3.1 Update Credential Endpoint

```typescript
// In api.controller.ts
export const createCredential = async (req: Request, res: Response, next: NextFunction) => {
  try {
    const userId = (req as ModifiedRequest).userId || req.body.issuerId;
    const credentialData = req.body;
    
    // Process credential with new service
    const { credential, claim } = await CredentialProcessorService.processCredential(
      credentialData, 
      userId
    );
    
    return res.status(201).json({ 
      message: "Credential created successfully!", 
      credential, 
      claim 
    });
  } catch (err) {
    passToExpressErrorHandler(err, next);
  }
};
```

### 3.2 Update Claim Creation

```typescript
// In api.controller.ts
export const claimPost = async (req: Request, res: Response, next: NextFunction) => {
  try {
    // ... existing claim creation ...
    
    // After creating claim, process entities
    await EntityDetectorService.processClaimEntities(claim);
    
    // ... rest of existing code ...
  } catch (err) {
    passToExpressErrorHandler(err, next);
  }
};
```

### 3.3 Update Graph Endpoints

```typescript
// In node.controller.ts
export const claimGraph = async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { claimId } = req.params;
    const result = await nodeDao.getClaimGraph(claimId);
    
    // Enhance nodes with entity data
    const enhancedNodes = await GraphEnhancerService.enhanceNodes(result.nodes);
    
    res.status(200).json({
      nodes: enhancedNodes,
      count: result.count
    });
  } catch (err) {
    passToExpressErrorHandler(err, next);
  }
};
```

## Phase 4: Pipeline Updates (Day 2 Afternoon)

### 4.1 Update Pipeline Node Creation

In `claims_to_nodes/pipe.py`:

```python
def get_or_create_node(node_uri, raw_claim, new_node=None):
    """Enhanced node creation that checks entity types"""
    
    # Normalize URI
    node_uri = normalize_uri(node_uri, raw_claim["issuerId"])
    
    # Check if we have entity info for this URI
    entity_info = get_entity_info(node_uri)
    
    # Check if node exists
    node = get_node_by_uri(node_uri)
    
    if node is None:
        if new_node is None:
            # Use entity info if available
            if entity_info:
                name = entity_info['extracted_name'] or extract_fallback_name(node_uri)
                ent_type = entity_info['entity_type']
                thumbnail = entity_info['extracted_image'] or ""
            else:
                name = raw_claim.get("subject") or extract_fallback_name(node_uri)
                ent_type = infer_entity_type(node_uri, raw_claim)
                thumbnail = ""
            
            node = {
                "nodeUri": node_uri,
                "name": name,
                "entType": ent_type,
                "thumbnail": thumbnail,
                "descrip": "",
            }
        else:
            node = new_node
            
        # Insert node
        node["id"] = insert_node(node)
    
    return node

def get_entity_info(uri):
    """Query uri_entities table for entity information"""
    query = """
        SELECT entity_type, extracted_name, extracted_image 
        FROM uri_entities 
        WHERE uri = %s
    """
    result = execute_query(query, (uri,))
    return result[0] if result else None

def infer_entity_type(uri, raw_claim):
    """Fallback entity type inference"""
    # Check if it's a claim about a claim
    if uri.startswith(make_subject_uri_prefix()):
        return "CLAIM"
    
    # Check if it's a source that's also the issuer (first-hand)
    if raw_claim.get("sourceURI") == raw_claim.get("issuerId"):
        return "PERSON"  # Likely a person making first-hand claim
    
    # Default
    return "UNKNOWN"

def process_claim(raw_claim):
    """Enhanced claim processing"""
    # ... existing code ...
    
    # IMPORTANT: Process source as a node if it's not the issuer
    if raw_claim.get("sourceURI") and raw_claim["sourceURI"] != raw_claim.get("issuerId"):
        source_node = get_or_create_node(raw_claim["sourceURI"], raw_claim)
        # This creates the source as a node in the graph!
```

## Phase 5: Frontend Integration (Day 3)

### 5.1 Update API Types

```typescript
// types/api.types.ts
export interface EnhancedNode extends Node {
  entityType?: string;
  entityData?: {
    credential?: Credential;
    person?: PersonEntity;
    // ... other entity types
  };
  displayName?: string;
  displayImage?: string;
}
```

### 5.2 Create Entity Renderers

```typescript
// components/entities/CredentialCard.tsx
export const CredentialCard: React.FC<{ node: EnhancedNode }> = ({ node }) => {
  const credential = node.entityData?.credential;
  if (!credential) return <GenericNodeCard node={node} />;
  
  return (
    <Card>
      <CardMedia
        component="img"
        height="200"
        image={credential.achievement_image_url || '/default-badge.png'}
        alt={credential.achievement_name}
      />
      <CardContent>
        <Typography variant="h5">
          {credential.achievement_name || credential.display_name}
        </Typography>
        <Typography variant="subtitle1" color="text.secondary">
          Issued by {credential.issuer_name}
        </Typography>
        <Typography variant="body2">
          {credential.achievement_description}
        </Typography>
        <Box mt={2}>
          <Button 
            onClick={() => navigate(`/certificate/${node.id}`)}
          >
            View Certificate
          </Button>
        </Box>
      </CardContent>
    </Card>
  );
};
```

### 5.3 Update Graph Visualization

```typescript
// components/graph/NodeRenderer.tsx
export const getNodeColor = (node: EnhancedNode) => {
  switch (node.entityType) {
    case 'CREDENTIAL': return '#4CAF50';  // Green
    case 'PERSON': return '#2196F3';      // Blue
    case 'ORGANIZATION': return '#FF9800'; // Orange
    case 'CLAIM': return '#9C27B0';       // Purple
    case 'DOCUMENT': return '#795548';    // Brown
    default: return '#757575';            // Grey
  }
};

export const getNodeIcon = (node: EnhancedNode) => {
  switch (node.entityType) {
    case 'CREDENTIAL': return '🏆';
    case 'PERSON': return '👤';
    case 'ORGANIZATION': return '🏢';
    case 'CLAIM': return '💬';
    case 'DOCUMENT': return '📄';
    default: return '⚪';
  }
};
```

## Phase 6: Testing & Migration (Day 3-4)

### 6.1 Data Migration Script

```sql
-- Populate uri_entities from existing data

-- From Credentials
INSERT INTO uri_entities (uri, entity_type, entity_table, entity_id, name)
SELECT 
  COALESCE(canonical_uri, id),
  'CREDENTIAL'::"EntityType",
  'Credential',
  id,
  COALESCE(name, 'Credential')
FROM "Credential"
ON CONFLICT (uri) DO NOTHING;

-- From Claims that look like people (first-hand sources)
INSERT INTO uri_entities (uri, entity_type, entity_table, entity_id, name)
SELECT DISTINCT
  "sourceURI",
  'PERSON'::"EntityType",
  'person_entities',
  "sourceURI",
  SUBSTRING("sourceURI" FROM '[^/]+
```

### 6.2 Test Cases

1. **Credential Submission**
   - Submit credential via API
   - Verify credential stored with display fields
   - Verify claim created with correct subject/object
   - Verify uri_entities populated
   - Verify nodes show with entity data

2. **Source as Node**
   - Create claim with different source than issuer
   - Verify source becomes a node
   - Verify graph shows issuer -> claim -> source relationship

3. **Entity Type Detection**
   - Submit various URI patterns
   - Verify correct entity types detected
   - Verify enhanced nodes return entity data

## Timeline Summary

- **Day 1**: Database schema + Backend services
- **Day 2**: API updates + Pipeline integration  
- **Day 3**: Frontend integration + Testing
- **Day 4**: Migration + Deployment

## Critical Success Factors

1. **Don't break existing functionality** - All changes are additive
2. **Test source vs issuer** - Ensure both become nodes correctly
3. **Verify URI persistence** - Entity relationships survive node regeneration
4. **Monitor performance** - Entity joins shouldn't slow down API

## Rollback Plan

If issues arise:
1. Frontend can ignore entityType/entityData fields
2. Pipeline continues to work without entity detection
3. API endpoints function without enhancement
4. Only new tables need to be dropped

This plan implements the architecture while maintaining system stability and allowing incremental deployment.
)
FROM "Claim"
WHERE "sourceURI" = "issuerId"
  AND "sourceURI" IS NOT NULL
ON CONFLICT (uri) DO NOTHING;

-- From web sources
INSERT INTO uri_entities (uri, entity_type, entity_table, entity_id, name)
SELECT DISTINCT
  "sourceURI",
  'DOCUMENT'::"EntityType",
  'external_source',
  "sourceURI",
  SUBSTRING("sourceURI" FROM '[^/]+
```

### 6.2 Test Cases

1. **Credential Submission**
   - Submit credential via API
   - Verify credential stored with display fields
   - Verify claim created with correct subject/object
   - Verify uri_entities populated
   - Verify nodes show with entity data

2. **Source as Node**
   - Create claim with different source than issuer
   - Verify source becomes a node
   - Verify graph shows issuer -> claim -> source relationship

3. **Entity Type Detection**
   - Submit various URI patterns
   - Verify correct entity types detected
   - Verify enhanced nodes return entity data

## Timeline Summary

- **Day 1**: Database schema + Backend services
- **Day 2**: API updates + Pipeline integration  
- **Day 3**: Frontend integration + Testing
- **Day 4**: Migration + Deployment

## Critical Success Factors

1. **Don't break existing functionality** - All changes are additive
2. **Test source vs issuer** - Ensure both become nodes correctly
3. **Verify URI persistence** - Entity relationships survive node regeneration
4. **Monitor performance** - Entity joins shouldn't slow down API

## Rollback Plan

If issues arise:
1. Frontend can ignore entityType/entityData fields
2. Pipeline continues to work without entity detection
3. API endpoints function without enhancement
4. Only new tables need to be dropped

This plan implements the architecture while maintaining system stability and allowing incremental deployment.
)
FROM "Claim"
WHERE "sourceURI" != "issuerId"
  AND "sourceURI" LIKE 'http%'
ON CONFLICT (uri) DO NOTHING;
```

### 6.2 Test Cases

1. **Credential Submission**
   - Submit credential via API
   - Verify credential stored with display fields
   - Verify claim created with correct subject/object
   - Verify uri_entities populated
   - Verify nodes show with entity data

2. **Source as Node**
   - Create claim with different source than issuer
   - Verify source becomes a node
   - Verify graph shows issuer -> claim -> source relationship

3. **Entity Type Detection**
   - Submit various URI patterns
   - Verify correct entity types detected
   - Verify enhanced nodes return entity data

## Timeline Summary

- **Day 1**: Database schema + Backend services
- **Day 2**: API updates + Pipeline integration  
- **Day 3**: Frontend integration + Testing
- **Day 4**: Migration + Deployment

## Critical Success Factors

1. **Don't break existing functionality** - All changes are additive
2. **Test source vs issuer** - Ensure both become nodes correctly
3. **Verify URI persistence** - Entity relationships survive node regeneration
4. **Monitor performance** - Entity joins shouldn't slow down API

## Rollback Plan

If issues arise:
1. Frontend can ignore entityType/entityData fields
2. Pipeline continues to work without entity detection
3. API endpoints function without enhancement
4. Only new tables need to be dropped

This plan implements the architecture while maintaining system stability and allowing incremental deployment.
