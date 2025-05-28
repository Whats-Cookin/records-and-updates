# Bring Your Own DID (BYO-DID) Implementation

## What We Built

### 1. Enhanced Web3 Auth (`/src/utils/web3Auth.ts`)
- **Auto DID**: Creates `did:ethr:0x...` from MetaMask addresses
- **Custom DID**: Users can set any valid DID they own
- **Flexible Storage**: Remembers user's DID preference

### 2. Identity Manager UI (`/src/components/IdentityManager/`)
- Visual dialog to manage identity
- Shows current DID/address
- Allows setting custom DIDs
- Clear button to revert to Ethereum-based DID

### 3. Navbar Integration
- Shows identity chip in navbar when logged in
- Click to open identity manager
- Displays shortened DID for clean UI

## How It Works

### For MetaMask Users (Default Flow):
1. Connect MetaMask
2. System creates: `did:ethr:0x742d35cc...`
3. All claims signed with this DID

### For Users with Existing DIDs:
1. Click identity button in navbar
2. Enter their DID (e.g., `did:key:z6MkhaXg...`, `did:web:example.com`, `did:ion:...`)
3. System uses their DID for all future claims
4. MetaMask still signs, but claims show custom DID as issuer

### Supported DID Methods:
- ✅ `did:ethr` - Ethereum addresses (default)
- ✅ `did:key` - Self-contained key-based DIDs
- ✅ `did:web` - Domain-based DIDs
- ✅ `did:ion` - ION/Sidetree DIDs
- ✅ `did:pkh` - Protocol/blockchain accounts
- ✅ Any other valid DID format

## User Benefits

1. **Identity Portability**: Use the same DID across multiple platforms
2. **Privacy Options**: Use different DIDs for different contexts
3. **Integration Ready**: Works with existing DID systems
4. **No Lock-in**: Can change DIDs anytime

## Example Scenarios

### Academic Researcher
- Has `did:web:university.edu/researchers/jane`
- Uses this DID for all professional claims
- Maintains institutional identity

### Privacy-Conscious User
- Generates `did:key:z6MkhaXg...` offline
- Uses ephemeral DIDs for different contexts
- No blockchain trail

### DAO Member
- Has `did:dao:example/members/alice`
- All claims tied to DAO identity
- Portable across DAO tools

## Technical Details

### Storage
- Custom DID: `localStorage.userDid`
- Ethereum address: `localStorage.ethAddress`
- ID type: `localStorage.userIdType`

### Validation
- DIDs must start with `did:`
- Must have method and identifier
- Format: `did:method:specific-identifier`

### Signing Process
1. User creates claim
2. System checks for custom DID
3. Falls back to did:ethr if none set
4. MetaMask signs the claim
5. Proof includes the chosen DID

## Next Steps

1. **Backend Support**: Update backend to:
   - Accept any DID format as issuerId
   - Optionally resolve DIDs to verify keys
   - Show DID metadata in UI

2. **DID Resolution** (Optional):
   - Add DID resolver library
   - Verify signing keys match DID documents
   - Cache DID documents

3. **Enhanced UI**:
   - Show DID method icon
   - Quick DID switcher
   - Import DID from file/QR code

The system is now fully functional for BYO-DID while maintaining the simplicity of MetaMask-only signing!
