# Legacy Endpoints to Preserve for v4 API

## Authentication Endpoints

### 1. Login
- **Path**: `/auth/login`
- **Method**: `POST`
- **Request Body**:
  ```json
  {
    "email": "string",
    "password": "string"
  }
  ```
- **Response (200)**:
  ```json
  {
    "accessToken": "string",
    "refreshToken": "string"
  }
  ```
- **Error Response (401)**: Invalid email/password

### 2. Signup
- **Path**: `/auth/signup`
- **Method**: `POST`
- **Request Body**:
  ```json
  {
    "email": "string",
    "password": "string"
  }
  ```
- **Response (201)**: 
  ```json
  {
    "message": "string"
  }
  ```
- **Error Responses**:
  - 409: Email already exists
  - 500: Invalid email/password or internal server error

### 3. Refresh Token
- **Path**: `/auth/refresh_token`
- **Method**: `POST`
- **Request Body**:
  ```json
  {
    "refreshToken": "string"
  }
  ```
- **Response (200)**:
  ```json
  {
    "accessToken": "string",
    "refreshToken": "string"
  }
  ```
- **Error Response (500)**: Invalid body/invalid or expired token/internal server error

## Claim Endpoints

### 1. Create Claim (JSON)
- **Path**: `/api/claim`
- **Method**: `POST`
- **Security**: Bearer Auth (JWT)
- **Request Body**:
  ```json
  {
    "subject": "string",          // required
    "claim": "string",            // required
    "name": "string",             // required
    "object": "string | null",
    "statement": "string | null",
    "aspect": "string | null",
    "amt": "integer | null",
    "howKnown": "FIRST_HAND | SECOND_HAND | WEB_DOCUMENT | VERIFIED_LOGIN | BLOCKCHAIN | SIGNED_DOCUMENT | PHYSICAL_DOCUMENT | INTEGRATION | RESEARCH | OPINION | OTHER | null",
    "images": [
      {
        "url": "string",          // required
        "signature": "string",    // required
        "owner": "string",        // required
        "metadata": {
          "captian": "string",
          "description": "string"
        },
        "effectiveDate": "datetime | null",
        "digestMultibase": "string | null"
      }
    ],
    "sourceURI": "string | null",
    "effectiveDate": "datetime | null",
    "confidence": "integer | null",
    "claimAddress": "string | null",
    "stars": "integer | null"
  }
  ```
- **Response (201)**:
  ```json
  {
    "claim": {
      "id": "integer",
      "name": "string",
      "thumbnail": "string",
      "link": "string",
      "description": "string",
      "claim_id": "integer",
      "statement": "string",
      "stars": "integer",
      "score": "integer",
      "amt": "integer",
      "effective_date": "datetime",
      "how_known": "string",
      "aspect": "string",
      "confidence": "integer",
      "claim": "string",
      "basis": "string",
      "source_name": "string",
      "source_thumbnail": "string",
      "source_link": "string",
      "source_description": "string"
    },
    "claimData": {
      "id": "integer",
      "claimId": "integer",
      "name": "string"
    },
    "claimImages": [
      {
        "id": "integer",
        "claimId": "integer",
        "url": "string",
        "digetedMultibase": "string",
        "metadata": {
          "captian": "string",
          "description": "string"
        },
        "effectiveDate": "datetime",
        "createdDate": "datetime",
        "owner": "string",
        "signature": "string"
      }
    ]
  }
  ```
- **Error Responses**:
  - 401: Unauthorized
  - 500: Invalid body/internal server error

### 2. Create Claim (Multipart/Form-Data with Image Streams)
- **Path**: `/api/claim/v2`
- **Method**: `POST`
- **Security**: Bearer Auth (JWT)
- **Content Type**: `multipart/form-data`
- **Request Body**:
  - `images`: Array of binary image files
  - `dto`: JSON object with same structure as Create Claim (JSON) but:
    - No `images[].url`, `images[].signature`, `images[].owner` required
    - Images array in dto only contains metadata, effectiveDate, digestMultibase
- **Response**: Same as Create Claim (JSON)

### 3. Get Single Claim
- **Path**: `/api/claim/{id}`
- **Method**: `GET`
- **Path Parameter**: `id` (integer) - Claim ID
- **Response (200)**: Same structure as Create Claim response
- **Error Responses**:
  - 404: Not found
  - 500: Internal server error

## Migration Notes for v4

### Canonical v4 paths:
- Auth endpoints: Keep at `/auth/*` (no version prefix needed)
- Claim endpoints: 
  - Create Claim: `/api/v4/claim` (also available at `/api/claim` if no conflict)
  - Create Claim v2: `/api/v4/claim/v2` (also available at `/api/claim/v2` if no conflict)
  - Get Single Claim: `/api/v4/claim/{id}` (also available at `/api/claim/{id}` if no conflict)

### Important Details:
1. **Authentication**: Uses JWT Bearer tokens
2. **Security Scheme**: Bearer Auth format for protected endpoints
3. **Image Handling**: Two approaches supported:
   - v1: Images as URLs with signatures in JSON
   - v2: Images as multipart streams with metadata in separate JSON
4. **How Known Enum**: Specific values for claim provenance tracking
5. **Response Format**: Nested structure with claim, claimData, and claimImages

### Fields to Consider for LinkedClaims Compatibility:
- `subject`: Maps to LinkedClaim's subject (URI)
- `claim`: The claim type/predicate
- `object`: Optional object of the claim
- `statement`: Human-readable statement
- `sourceURI`: Evidence/source link
- `effectiveDate`: Date claim is effective
- `confidence`: Confidence score
- `claimAddress`: Could map to LinkedClaim's id (URI)
- Image signatures and digestMultibase for content integrity
