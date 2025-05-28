# LinkedTrust Project Summary

## Overview

The LinkedTrust project implements a decentralized trust system based on the LinkedClaims specification. This system enables creating, validating, and connecting claims across different domains to establish a web of trust. The core concept revolves around cryptographically signed claims that are both addressable (having their own URI) and about addressable subjects (also with URIs), allowing for connections between claims and building trust networks.

## Core Concept

LinkedClaims is a minimal standard for interoperable, machine-readable claims that can be linked together to form a web of trust. Each claim:
- Has a URI identifier
- Is about a subject with a URI
- Is cryptographically signed
- Can be chained to other claims (a claim can be about another claim)

This allows for building trust networks where entities can make verifiable statements about resources or other claims, enabling credibility assessment across domains.

## Architecture Components

```
┌─────────────────────┐     ┌──────────────────┐     ┌───────────────────────┐
│   Claims Creation   │────▶│ Claims Storage   │────▶│  Claims Processing    │
│   and Extraction    │     │ and Retrieval    │     │  and Graph Building   │
└─────────────────────┘     └──────────────────┘     └───────────────────────┘
        │                           │                           │
        ▼                           ▼                           ▼
┌─────────────────────┐     ┌──────────────────┐     ┌───────────────────────┐
│  linked-claims-     │     │ trust_claim_     │     │  trust-claim-data-    │
│  extractor          │     │ backend          │     │  pipeline             │
└─────────────────────┘     └──────────────────┘     └───────────────────────┘
                                    │
                                    ▼
                            ┌──────────────────┐
                            │    trust_claim   │
                            │    (Frontend)    │
                            └──────────────────┘
```

### 1. linked-claims-extractor

**Purpose**: Extracts structured claims from unstructured text sources, URLs, or PDFs.

**Key Components**:
- AI-powered extraction using language models (LLM)
- PDF parser for document analysis
- Claim extraction services

This component transforms unstructured information into structured LinkedClaims, enabling ingestion of claims from various sources.

### 2. trust_claim_backend

**Purpose**: Provides the API and database storage for claims, nodes, and edges.

**Key Concepts**:
- **Claim**: A signed set of structured data with assertions, often signed by the user's DID
- **Node**: An entity that a claim is about
- **Edge**: A representation of a claim that relates to a node or connects two nodes

The backend uses PostgreSQL with Prisma ORM for data storage and provides REST endpoints for the frontend.

### 3. trust-claim-data-pipeline

**Purpose**: Processes raw claims and transforms them into a graph of nodes and edges.

**Pipeline Steps**:
1. Spider and save raw data to be turned into claims
2. Clean and normalize data into an importable format
3. Import into signed claims (signed by the spider)
4. Dedupe, parse, and transform claims into nodes and edges

This component also publishes to Ceramic Network for decentralized storage.

### 4. trust_claim (Frontend)

**Purpose**: User interface for viewing, creating, and interacting with claims.

**Features**:
- Authentication with GitHub
- Claim creation and viewing
- Graph visualization of nodes and edges
- Integration with the backend API

## LinkedClaims Specification Requirements

### MUST Requirements
- Have a subject that is a valid URI
- Have its own identifier that is a well-formed URI
- Be cryptographically signed

### SHOULD Requirements
- Provide deterministic machine-readable content from its URI
- Include a date in the signed data
- Contain evidence (links to sources or attachments)
- Have a URI-addressable cryptographic signer

### MAY Requirements
- Have a narrative statement
- Have a subject that is itself a claim
- Be a W3C Verifiable Credential
- Provide mutation or revocation mechanism
- Provide an inbox for notifications
- Have separate published and effective dates
- Be public or access-controlled

## Use Cases

- Decentralized content moderation
- DAO governance and reputation
- Trust networks for online entities
- Validation of private claims with public evidence
- Cross-domain trust establishment

## Development and Deployment

The project has development and production environments:
- Development: https://dev.linkedtrust.us
- Production: https://live.linkedtrust.us

CI/CD is implemented with Jenkins, and deployment is handled through Docker and PM2.

## Key Directories and Files Map

- **/linked-trust/**
  - **/linked-claims-extractor/** - AI-powered claim extraction from text and PDFs
  - **/trust-claim-data-pipeline/** - Processing claims into nodes and edges
  - **/trust_claim/** - Frontend React application
  - **/trust_claim_backend/** - Backend Node.js API and database
  - **linked-claims-spec.md** - The LinkedClaims specification document

## Future Development Areas

1. ActivityPub compatible inbox for claim notifications
2. Improved deterministic content addressing
3. Integration with Progressive Trust framework
4. Enhanced evidence verification through hashlinked resources
5. Privacy-preserving claim mechanisms

## Conclusion

The LinkedTrust project implements a practical version of the LinkedClaims specification, creating a framework for decentralized trust assertions that can be linked, verified, and evaluated. By allowing claims to reference other claims and providing cryptographic verification, it enables building a web of trust across different domains and use cases.
