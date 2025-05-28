# Credential Ownership - How It Works

## User Identification
- Google OAuth users: `https://live.linkedtrust.us/userids/google/{googleId}`
- MetaMask users: `did:pkh:eip155:1:{address}`

## Ownership Model
A user owns a credential when there's a claim:
```
{
  subject: "user-uri",
  claim: "HAS",
  object: "credential-uri"
}
```

## Button Logic

**Show "Claim This Credential"** when:
- No one has claimed it yet AND
- User has `claim_token` OR
- User is the credential subject OR
- User has `invite_token`

**Show "Share on LinkedIn"** when:
- User has a HAS claim for this credential

**Show "Send Credential"** when:
- No one has claimed it yet AND
- User is the issuer OR
- User has `offer_token`

## URL Parameters
- `?claim_token=xyz` - From credential creation flow
- `?invite_token=abc` - From email invitation
- `?offer_token=def` - For issuers to send

## Technical Notes
- Tokens should be validated server-side
- Claims use VERIFIED_LOGIN as howKnown
- Export PDF is always available (no ownership required)