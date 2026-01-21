# Privy Integration Guide

## Overview

Privy provides easy authentication for web3 apps, including social login with X (Twitter). We'll use it to:
1. Allow users to sign in with their X account
2. Access basic profile information for personalization
3. Enable the feed simulator feature

---

## Setup

### 1. Create Privy Account
1. Go to https://privy.io
2. Create an account
3. Create a new app
4. Note your App ID

### 2. Configure X (Twitter) OAuth
In Privy Dashboard:
1. Go to Login Methods
2. Enable "Twitter"
3. Follow the Twitter Developer setup flow
4. Set callback URL to `https://yourdomain.com/api/auth/callback`

### 3. Install Dependencies

```bash
npm install @privy-io/react-auth @privy-io/server-auth
```

### 4. Environment Variables

```env
# .env.local
NEXT_PUBLIC_PRIVY_APP_ID=your-app-id
PRIVY_APP_SECRET=your-app-secret
```

---

## Implementation

### Provider Setup

```tsx
// app/layout.tsx
import { PrivyProvider } from '@privy-io/react-auth';

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <PrivyProvider
          appId={process.env.NEXT_PUBLIC_PRIVY_APP_ID!}
          config={{
            loginMethods: ['twitter'],
            appearance: {
              theme: 'dark',
              accentColor: '#1DA1F2', // X blue
              logo: '/logo.png',
            },
            embeddedWallets: {
              createOnLogin: 'off', // We don't need wallets
            },
          }}
        >
          {children}
        </PrivyProvider>
      </body>
    </html>
  );
}
```

### Login Button Component

```tsx
// components/auth/LoginButton.tsx
'use client';

import { usePrivy } from '@privy-io/react-auth';
import { Button } from '@/components/ui/Button';

export function LoginButton() {
  const { login, ready, authenticated } = usePrivy();

  if (!ready) {
    return <Button disabled>Loading...</Button>;
  }

  if (authenticated) {
    return null; // User is already logged in
  }

  return (
    <Button onClick={login} variant="primary">
      <XIcon className="w-5 h-5 mr-2" />
      Connect with X
    </Button>
  );
}
```

### User Profile Hook

```tsx
// hooks/useXProfile.ts
'use client';

import { usePrivy } from '@privy-io/react-auth';
import { useMemo } from 'react';

interface XProfile {
  id: string;
  username: string;
  displayName: string;
  profileImageUrl: string;
}

export function useXProfile(): XProfile | null {
  const { user, authenticated } = usePrivy();

  return useMemo(() => {
    if (!authenticated || !user?.twitter) {
      return null;
    }

    const twitter = user.twitter;
    return {
      id: twitter.subject,
      username: twitter.username || '',
      displayName: twitter.name || twitter.username || '',
      profileImageUrl: twitter.profilePictureUrl || '',
    };
  }, [user, authenticated]);
}
```

### Protected Route Wrapper

```tsx
// components/auth/ProtectedRoute.tsx
'use client';

import { usePrivy } from '@privy-io/react-auth';
import { useRouter } from 'next/navigation';
import { useEffect } from 'react';

interface ProtectedRouteProps {
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

export function ProtectedRoute({ children, fallback }: ProtectedRouteProps) {
  const { ready, authenticated } = usePrivy();
  const router = useRouter();

  useEffect(() => {
    if (ready && !authenticated) {
      router.push('/login');
    }
  }, [ready, authenticated, router]);

  if (!ready) {
    return <div>Loading...</div>;
  }

  if (!authenticated) {
    return fallback || null;
  }

  return <>{children}</>;
}
```

### Logout Function

```tsx
// components/auth/LogoutButton.tsx
'use client';

import { usePrivy } from '@privy-io/react-auth';
import { Button } from '@/components/ui/Button';

export function LogoutButton() {
  const { logout, authenticated } = usePrivy();

  if (!authenticated) {
    return null;
  }

  return (
    <Button onClick={logout} variant="ghost">
      Sign Out
    </Button>
  );
}
```

---

## Using X Profile Data

### In the Simulator

```tsx
// app/simulate/page.tsx
'use client';

import { useXProfile } from '@/hooks/useXProfile';
import { ProtectedRoute } from '@/components/auth/ProtectedRoute';

export default function SimulatePage() {
  return (
    <ProtectedRoute>
      <SimulatorContent />
    </ProtectedRoute>
  );
}

function SimulatorContent() {
  const profile = useXProfile();

  if (!profile) {
    return <div>Loading profile...</div>;
  }

  return (
    <div>
      <h1>Feed Simulator</h1>
      <div className="profile-card">
        <img src={profile.profileImageUrl} alt={profile.displayName} />
        <h2>{profile.displayName}</h2>
        <p>@{profile.username}</p>
      </div>
      <p>
        We'll simulate how the algorithm would rank posts for your account.
      </p>
      {/* Simulator content */}
    </div>
  );
}
```

---

## Server-Side Verification

For API routes that need authentication:

```tsx
// app/api/simulate/route.ts
import { PrivyClient } from '@privy-io/server-auth';
import { NextRequest, NextResponse } from 'next/server';

const privy = new PrivyClient(
  process.env.NEXT_PUBLIC_PRIVY_APP_ID!,
  process.env.PRIVY_APP_SECRET!
);

export async function POST(request: NextRequest) {
  // Get the auth token from the request
  const authToken = request.headers.get('Authorization')?.replace('Bearer ', '');

  if (!authToken) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  try {
    // Verify the token
    const claims = await privy.verifyAuthToken(authToken);
    const userId = claims.userId;

    // Get user's Twitter data
    const user = await privy.getUser(userId);
    const twitter = user.twitter;

    if (!twitter) {
      return NextResponse.json({ error: 'No Twitter account linked' }, { status: 400 });
    }

    // Run simulation...
    const results = await runSimulation(twitter);

    return NextResponse.json(results);
  } catch (error) {
    return NextResponse.json({ error: 'Invalid token' }, { status: 401 });
  }
}
```

---

## What Data We Access

### From X Profile (via Privy)
- User ID (anonymous identifier)
- Username (@handle)
- Display name
- Profile picture URL

### What We DON'T Access
- Tweets
- Followers/following
- DMs
- Email
- Any private data

### Privacy Statement

```tsx
// components/auth/PrivacyNotice.tsx
export function PrivacyNotice() {
  return (
    <div className="privacy-notice">
      <h3>What we access:</h3>
      <ul>
        <li>Your public profile (username, display name, picture)</li>
      </ul>

      <h3>What we DON'T access:</h3>
      <ul>
        <li>Your tweets or timeline</li>
        <li>Your followers or following</li>
        <li>Your DMs or private messages</li>
        <li>Your email address</li>
        <li>Ability to post on your behalf</li>
      </ul>

      <h3>How we use it:</h3>
      <ul>
        <li>Personalize the feed simulator experience</li>
        <li>Display your profile in the app</li>
      </ul>

      <h3>Data storage:</h3>
      <p>We don't store any of your data. Everything is session-only.</p>
    </div>
  );
}
```

---

## Error Handling

```tsx
// hooks/useAuth.ts
'use client';

import { usePrivy } from '@privy-io/react-auth';
import { useState, useCallback } from 'react';

export function useAuth() {
  const { login, logout, authenticated, ready, user } = usePrivy();
  const [error, setError] = useState<string | null>(null);

  const handleLogin = useCallback(async () => {
    setError(null);
    try {
      await login();
    } catch (err) {
      setError('Failed to connect with X. Please try again.');
      console.error('Login error:', err);
    }
  }, [login]);

  const handleLogout = useCallback(async () => {
    setError(null);
    try {
      await logout();
    } catch (err) {
      setError('Failed to sign out. Please try again.');
      console.error('Logout error:', err);
    }
  }, [logout]);

  return {
    login: handleLogin,
    logout: handleLogout,
    authenticated,
    ready,
    user,
    error,
    clearError: () => setError(null),
  };
}
```

---

## Testing

### Mock Privy for Development

```tsx
// lib/privy-mock.ts
export const mockUser = {
  id: 'test-user-id',
  twitter: {
    subject: '12345678',
    username: 'testuser',
    name: 'Test User',
    profilePictureUrl: 'https://pbs.twimg.com/profile_images/default.png',
  },
};

// Use in development
if (process.env.NODE_ENV === 'development') {
  // Mock Privy responses
}
```

---

## Checklist

- [ ] Create Privy account and app
- [ ] Configure Twitter OAuth in Privy dashboard
- [ ] Set up environment variables
- [ ] Implement PrivyProvider in layout
- [ ] Create login/logout components
- [ ] Implement useXProfile hook
- [ ] Create ProtectedRoute wrapper
- [ ] Set up server-side verification
- [ ] Add privacy notice
- [ ] Test authentication flow
- [ ] Handle errors gracefully
