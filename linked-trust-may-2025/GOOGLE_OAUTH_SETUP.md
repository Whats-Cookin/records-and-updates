# Google OAuth Setup for LinkedTrust

## Frontend Setup

### 1. Environment Variables
Add to your `.env` file:
```
VITE_GOOGLE_CLIENT_ID=your_google_client_id_here
VITE_BACKEND_BASE_URL=https://dev.linkedtrust.us
```

### 2. Get Google Client ID
1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select existing
3. Enable Google+ API
4. Go to Credentials
5. Create OAuth 2.0 Client ID
6. Add authorized JavaScript origins:
   - http://localhost:5173
   - https://dev.linkedtrust.us
   - https://live.linkedtrust.us
7. Copy the Client ID

### 3. Frontend is Ready!
The frontend already has Google OAuth set up in `/src/containers/Login/index.tsx`

## Backend Setup

### 1. Install Dependencies
```bash
cd trust_claim_backend
npm install google-auth-library bcryptjs
npm install --save-dev @types/bcryptjs
```

### 2. Run Database Migration
```bash
npx prisma migrate dev --name add_google_auth
```

### 3. Environment Variables
Add to backend `.env`:
```
GOOGLE_CLIENT_ID=same_google_client_id_as_frontend
ACCESS_SECRET=your_jwt_access_secret
REFRESH_SECRET=your_jwt_refresh_secret
```

### 4. Backend Auth Routes Added
- POST `/auth/google` - Google OAuth login
- POST `/auth/login` - Email/password login
- POST `/auth/register` - Email/password registration
- POST `/auth/refresh_token` - Refresh JWT token
- POST `/auth/github` - GitHub OAuth (placeholder)
- POST `/auth/wallet` - Wallet authentication

## How It Works

1. User clicks Google login button in frontend
2. Google OAuth popup appears
3. User authorizes the app
4. Frontend receives Google ID token
5. Frontend sends token to backend `/auth/google`
6. Backend verifies token with Google
7. Backend creates/finds user in database
8. Backend returns JWT tokens
9. Frontend stores tokens and redirects to feed

## Testing

1. Start backend:
```bash
cd trust_claim_backend
npm run dev
```

2. Start frontend:
```bash
cd trust_claim
yarn dev
```

3. Navigate to http://localhost:5173/login
4. Click the Google login button
5. Authorize with your Google account
6. You should be redirected to the feed, authenticated!

## Troubleshooting

### "Google authentication failed"
- Check that GOOGLE_CLIENT_ID matches in frontend and backend
- Ensure Google+ API is enabled in Google Cloud Console
- Check browser console for specific errors

### Database errors
- Run `npx prisma generate` after schema changes
- Run `npx prisma migrate dev` to apply migrations
- Check that PostgreSQL is running

### CORS errors
- Backend already has CORS enabled for all origins
- If still issues, check browser network tab

## Next Steps

1. Add user profile page to show Google profile info
2. Implement GitHub OAuth (backend placeholder exists)
3. Add logout functionality
4. Show user name/avatar in navbar when logged in
