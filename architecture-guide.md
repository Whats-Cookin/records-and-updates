# LinkedTrust Architecture Guide

## Core Concepts

### The Fundamental Distinction: Issuer vs Source

This is the key insight that drives our trust graph architecture:

- **Issuer**: The entity who signs and stakes their reputation on a claim
- **Source**: Where the information originally comes from (the evidence basis)

### Examples of Issuer vs Source

#### 1. First-hand Experience
```json
{
  "subject": "https://hotel.com",
  "claim": "HAS_RATING",
  "issuerId": "did:person:alice",
  "sourceURI": "did:person:alice",  // Same as issuer for first-hand
  "howKnown": "FIRST_HAND",
  "confidence": 1.0,
  "stars": 5,
  "statement": "I stayed here last week"
}
```

#### 2. Second-hand Information
```json
{
  "subject": "https://restaurant.com",
  "claim": "HAS_RATING", 
  "issuerId": "did:person:bob",      // Bob is making the claim
  "sourceURI": "did:person:charlie",  // But Charlie told him
  "howKnown": "SECOND_HAND",
  "confidence": 0.7,                  // Bob's confidence in Charlie's report
  "stars": 2,
  "statement": "Charlie said the food was bad"
}
```

#### 3. Web Research/Spider
```json
{
  "subject": "https://company.com",
  "claim": "HAS_REVENUE",
  "issuerId": "https://spider.linkedtrust.us",  // Spider service signs
  "sourceURI": "https://forbes.com/article/123", // Where it found the data
  "howKnown": "WEB_DOCUMENT",
  "confidence": 0.8,
  "amount": 1000000,
  "unit": "USD",
  "statement": "Annual revenue per Forbes article"
}
```

### Why This Matters

The sourceURI becomes a node in the trust graph, enabling:
- **Provenance tracking**: Trace information back to its origin
- **Source reputation**: Evaluate quality of different sources
- **Trust chains**: See relationships like "Bob trusts Charlie who saw X"
- **Multi-hop credibility**: Claims can reference other claims as sources

## Architecture Principles

### 1. Claims as Immutable Events

Claims are raw events that capture:
- **Semantic triple**: `subject` + `claim` + `object` (optional)
- **Provenance**: `issuerId`, `sourceURI`, `howKnown`
- **Confidence**: Issuer's stake in the claim
- **Context**: `effectiveDate`, `statement`
- **Proof**: Cryptographic signature

Claims are NEVER modified after creation - they represent historical assertions.

### 2. URIs as Persistent Identifiers

- **Nodes are views** generated from claims
- **Node IDs are not stable** across regeneration
- **URIs are the permanent identifiers**
- All entity relationships should key off URIs, not database IDs

### 3. Entity Type System

Instead of cramming everything into the Claim model, we use a flexible entity system:

```sql
-- Link URIs to their entity types and data
CREATE TABLE uri_entities (
  id SERIAL PRIMARY KEY,
  uri TEXT NOT NULL UNIQUE,
  entity_type entity_type_enum NOT NULL,
  entity_table TEXT NOT NULL,
  entity_id TEXT NOT NULL,
  extracted_name TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);
```

This allows different URI types to have appropriate storage:
- Credentials → Credential table
- People → PersonEntity table  
- Organizations → OrganizationEntity table
- Impact measurements → ImpactEntity table

### 4. LinkedClaims as Pragmatic Pattern

LinkedClaims are NOT a new credential format. They are:
- **An extraction pattern** from existing credentials
- **A lightweight assertion format** for common claims
- **A way to make everything addressable** with URIs

We keep ratings and measurements in the LinkedClaim itself because:
1. They're the most common type of claim
2. It avoids credential ceremony for simple assertions
3. The sourceURI provides the necessary linkage

### 5. Credentials as Special Objects

When someone submits a credential:
1. Store it in the Credential table with its original structure
2. Generate a canonical URI if it doesn't have one
3. Create a LinkedClaim: `[subject] HAS [credential-uri]`
4. Extract additional LinkedClaims from the credential content
5. Register the credential URI in uri_entities

## Data Flow

### 1. Claim Creation
```
User/API → Create Claim → Store in Claims table → Trigger Pipeline
```

### 2. Entity Detection
```
New Claim → Detect Entity Types → Update uri_entities → Enrich Views
```

### 3. Node Generation
```
Pipeline → Read Claims → Check uri_entities → Generate Nodes/Edges
```

### 4. API Response
```
Request → Query Nodes → Join uri_entities → Return Enriched Data
```

## Key Design Decisions

### What Stays in Claims
- Core semantic triple
- Issuer and source URIs
- Confidence and howKnown
- Ratings and measurements (for pragmatic reasons)
- Cryptographic proof

### What Lives in Entity Tables
- Credential details (achievements, criteria, etc.)
- Person profiles (names, images, bios)
- Organization data
- Any rich structured data beyond basic claims

### What We're NOT Doing
- Not forcing everything to be a credential
- Not modifying claims after creation
- Not relying on node IDs for relationships
- Not creating a generic "metadata" dumping ground

## Migration Strategy

1. **Keep existing Claim table** - It's immutable history
2. **Add uri_entities table** - For type detection and enrichment
3. **Create entity-specific tables** - As needed for rich data
4. **Update pipeline** - To use entity information when generating nodes
5. **Enhance API** - To return entity data with nodes

## Future Considerations

- **Hypercerts**: Import as-is, extract LinkedClaims
- **Open Badges**: Store in Credential table, extract relationships
- **Verifiable Credentials**: Same pattern - store original, extract links
- **Custom claim types**: Add new entity tables as needed

The system remains extensible while keeping the core LinkedClaim pattern simple and focused on creating an addressable, navigable trust graph.
