# Web3 Signing Implementation Plan

## What We've Added

### 1. New Web3 Authentication Module (`/src/utils/web3Auth.ts`)
- Clean MetaMask integration without Ceramic
- Client-side signing with ethers.js
- DID creation from Ethereum addresses (did:ethr:)
- Both simple message signing and EIP-712 structured signing

### 2. Updated Login Component
- Removed Ceramic dependencies
- Uses new web3Auth utilities
- Optional backend authentication (falls back to client-only)
- Creates DIDs from Ethereum addresses

### 3. Updated Claim Creation
- Automatically signs claims if wallet is connected
- Falls back to server signing if no wallet
- Preserves all claim data with added proof

## How It Works

### For Users with MetaMask:
1. Connect wallet → get Ethereum address
2. Create DID: `did:ethr:0x123...` 
3. When creating claims:
   - Client signs the claim with their private key
   - Proof is attached to the claim
   - Backend can verify the signature
   - Claim shows as "signed by did:ethr:0x123..."

### For Users without MetaMask:
1. Use email/Google/GitHub login
2. Claims are signed by the backend
3. Works exactly as before

## Benefits
- **True decentralization**: Users control their signing keys
- **No external dependencies**: Just ethers.js
- **Backward compatible**: Works with or without wallet
- **Standard DIDs**: Uses did:ethr method
- **Verifiable**: Anyone can verify signatures on-chain

## Required Dependencies
Add to package.json:
```json
"ethers": "^6.9.0"
```

## Backend Considerations
The backend should:
1. Accept claims with `proof` objects
2. Optionally verify Ethereum signatures
3. Store the proof with the claim
4. Show issuer as the DID when displaying claims

## Next Steps
1. Run: `yarn add ethers`
2. Test MetaMask login
3. Test claim signing
4. Update backend to handle signed claims (optional)

## Alternative Libraries
If you prefer other approaches:
- **@spruceid/didkit-wasm**: Full DID toolkit
- **did-jwt**: For JWT-based claims
- **@veramo/core**: Full-featured DID framework
- **web3.js**: Alternative to ethers

The current implementation is minimal and clean, perfect for getting started with decentralized signing!
