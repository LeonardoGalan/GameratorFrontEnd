# Routing and Navigation

## Overview

Gamerator uses React Router v6 for client-side routing, implementing a Single Page Application (SPA) architecture. All navigation occurs without page reloads, with routes protected by authentication checks. The routing configuration is centralized in App.js and integrates tightly with authentication state management.

---

## React Router Configuration

### Setup (index.js)

```javascript
// src/index.js
import React from 'react'
import ReactDOM from 'react-dom'
import { BrowserRouter } from 'react-router-dom'
import App from './components/App'

ReactDOM.render(
  <React.StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </React.StrictMode>,
  document.getElementById('root')
)
```

**BrowserRouter**:
- Uses HTML5 History API (pushState, replaceState)
- Clean URLs without hash fragments (no /#/about)
- Requires server-side redirect configuration (Netlify handles this)

**Alternative (Not Used)**:
- **HashRouter**: Uses URL hash (#/) for routing, no server config needed
- **MemoryRouter**: Routes stored in memory, used for testing

---

## Route Definitions (App.js)

### Complete Route Map

```javascript
import { Routes, Route, Navigate } from 'react-router-dom'

<Routes>
  {/* Public Route: Login Page */}
  <Route
    path="/"
    element={
      login === "true"
        ? <Navigate to="/home" replace={true} />
        : <FrontPage />
    }
  />

  {/* Protected Route: Home/Dashboard */}
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

  {/* Protected Route: Action Games */}
  <Route
    path="/action"
    element={
      login === "true"
        ? (
            <>
              <Header />
              <Navbar />
              <Action />
              <Footer />
            </>
          )
        : <Navigate to="/" replace={true} />
    }
  />

  {/* Protected Route: Adventure Games */}
  <Route
    path="/adventure"
    element={
      login === "true"
        ? (
            <>
              <Header />
              <Navbar />
              <Adventure />
              <Footer />
            </>
          )
        : <Navigate to="/" replace={true} />
    }
  />

  {/* Protected Route: Indie Games */}
  <Route
    path="/indie"
    element={
      login === "true"
        ? (
            <>
              <Header />
              <Navbar />
              <Indie />
              <Footer />
            </>
          )
        : <Navigate to="/" replace={true} />
    }
  />

  {/* Protected Route: Shooter Games */}
  <Route
    path="/shooter"
    element={
      login === "true"
        ? (
            <>
              <Header />
              <Navbar />
              <Shooter />
              <Footer />
            </>
          )
        : <Navigate to="/" replace={true} />
    }
  />

  {/* Protected Route: RPG Games */}
  <Route
    path="/rpg"
    element={
      login === "true"
        ? (
            <>
              <Header />
              <Navbar />
              <Rpg />
              <Footer />
            </>
          )
        : <Navigate to="/" replace={true} />
    }
  />

  {/* Protected Route: Leaderboard */}
  <Route
    path="/leaderboard"
    element={
      login === "true"
        ? (
            <>
              <Header />
              <Navbar />
              <Leaderboard />
              <Footer />
            </>
          )
        : <Navigate to="/" replace={true} />
    }
  />
</Routes>
```

---

## Route Structure

### URL Hierarchy

```
/                       (Root - Login)
├── /home              (Dashboard)
├── /action            (Action Games Category)
├── /adventure         (Adventure Games Category)
├── /indie             (Indie Games Category)
├── /shooter           (Shooter Games Category)
├── /rpg               (RPG Games Category)
└── /leaderboard       (Top Rated Games)
```

**Characteristics**:
- Flat hierarchy (no nested routes)
- All routes at root level
- No dynamic segments (e.g., /games/:id)
- No query parameters used
- No route parameters

### Route Types

**1. Public Route**: `/`
- Accessible to unauthenticated users
- Shows FrontPage (Google OAuth login)
- Redirects to /home if already authenticated

**2. Protected Routes**: All others
- Require authentication (loginState === "true")
- Redirect to / if not authenticated
- Share common layout (Header, Navbar, Footer)

---

## Navigation Methods

### 1. Declarative Navigation (Link Component)

**Used in Navbar.js**:
```javascript
import { Link } from 'react-router-dom'

<Link to="/home" style={{ textDecoration: "none" }}>
  <li>Home</li>
</Link>

<Link to="/action" className="navLink">
  <li>Action</li>
</Link>

<Link to="/adventure" className="navLink">
  <li>Adventure</li>
</Link>

<Link to="/leaderboard" className="navLink">
  <li>Leaderboard</li>
</Link>
```

**Behavior**:
- Prevents default anchor behavior (no page reload)
- Updates browser history (back/forward buttons work)
- Triggers route matching and component rendering
- Styled like regular links

**Props**:
- `to`: Destination path (string)
- `replace`: Replace history entry instead of push (not used)
- `state`: Pass data to destination route (not used)
- `style`, `className`: Standard HTML attributes

**Accessibility**:
- Renders as `<a>` tag (semantic HTML)
- Keyboard accessible (Tab, Enter)
- Screen reader friendly

### 2. Programmatic Navigation (useNavigate Hook)

**Used in FrontPage.js**:
```javascript
import { useNavigate } from 'react-router-dom'

const navigate = useNavigate()

const responseGoogle = (response) => {
  // ... authentication logic
  navigate("/home")  // Navigate after login
}
```

**Used in Navbar.js**:
```javascript
const navigate = useNavigate()

const logout = () => {
  localStorage.clear()
  navigate("/")             // Navigate to login
  window.location.reload()  // Force reload
}
```

**When to Use**:
- After form submission (login, logout)
- Conditional navigation (redirect based on logic)
- Navigation in event handlers
- Replacing browser history

**API**:
```javascript
// Navigate to path
navigate("/home")

// Navigate with replace (no history entry)
navigate("/home", { replace: true })

// Navigate backward
navigate(-1)

// Navigate forward
navigate(1)

// Navigate with state
navigate("/profile", { state: { from: "home" } })
```

### 3. Redirect (Navigate Component)

**Used in Route Protection**:
```javascript
import { Navigate } from 'react-router-dom'

<Route
  path="/home"
  element={
    login === "true"
      ? <Home />
      : <Navigate to="/" replace={true} />
  }
/>
```

**Characteristics**:
- Declarative redirect
- Renders nothing (null)
- Immediately triggers navigation
- Used in conditional rendering

**Props**:
- `to`: Destination path (string or object)
- `replace`: Replace history entry (default: false)
- `state`: Pass data to destination

**Use Cases**:
- Route guards (authentication checks)
- Redirects after successful actions
- 404 fallback routes (not implemented)

---

## Route Protection Pattern

### Authentication Guard

```javascript
// App.js - Pattern repeated for all protected routes

const [login, setLogin] = useState(localStorage.getItem("loginState") || "false")

<Route
  path="/protected"
  element={
    login === "true"
      ? <ProtectedComponent />   // Render if authenticated
      : <Navigate to="/" replace={true} />  // Redirect if not
  }
/>
```

**Flow Diagram**:
```
User Requests /protected
        ↓
React Router Matches Route
        ↓
Evaluate Ternary Expression
        ↓
Check: login === "true" ?
        ↓
   ┌────┴────┐
   │         │
  YES       NO
   │         │
   ↓         ↓
Render    Navigate
Component  to="/"
```

**Issues with Current Pattern**:
1. **Repetition**: Same ternary logic in every route
2. **No Loading State**: Instant redirect (could flicker)
3. **Coupled to localStorage**: Not testable without mocking

### Improved Pattern: Protected Route Component

**Recommended Refactor**:
```javascript
// ProtectedRoute.js
import { Navigate } from 'react-router-dom'

function ProtectedRoute({ children }) {
  const login = localStorage.getItem("loginState")

  if (login !== "true") {
    return <Navigate to="/" replace={true} />
  }

  return (
    <>
      <Header />
      <Navbar />
      {children}
      <Footer />
    </>
  )
}

// App.js - Cleaner usage
<Route path="/home" element={<ProtectedRoute><Home /></ProtectedRoute>} />
<Route path="/action" element={<ProtectedRoute><Action /></ProtectedRoute>} />
<Route path="/adventure" element={<ProtectedRoute><Adventure /></ProtectedRoute>} />
```

**Benefits**:
- DRY (Don't Repeat Yourself)
- Single place to update protection logic
- Easier to test
- Can add loading state, error handling

---

## Layout Pattern

### Current Implementation: Repeated Layout

```javascript
// Every protected route duplicates layout
<Route
  path="/home"
  element={
    <>
      <Header />
      <Navbar />
      <Home />
      <Footer />
    </>
  }
/>
```

**Issues**:
- Layout code repeated 7 times
- Changing layout requires updating 7 routes
- Inconsistent if developer forgets to update one route

### Recommended: Outlet Pattern (React Router v6)

```javascript
// Layout.js
import { Outlet } from 'react-router-dom'

function Layout() {
  return (
    <>
      <Header />
      <Navbar />
      <Outlet />  {/* Renders matched child route */}
      <Footer />
    </>
  )
}

// App.js - Nested routes
<Routes>
  <Route path="/" element={<FrontPage />} />

  <Route element={<ProtectedRoute />}>
    <Route element={<Layout />}>
      <Route path="/home" element={<Home />} />
      <Route path="/action" element={<Action />} />
      <Route path="/adventure" element={<Adventure />} />
      <Route path="/indie" element={<Indie />} />
      <Route path="/shooter" element={<Shooter />} />
      <Route path="/rpg" element={<Rpg />} />
      <Route path="/leaderboard" element={<Leaderboard />} />
    </Route>
  </Route>
</Routes>
```

**Benefits**:
- Layout defined once
- Nested route hierarchy
- Easier to understand route structure
- Supports multiple layouts

---

## Navigation State Management

### Browser History API

React Router v6 uses HTML5 History API:

```javascript
// Push new entry (default)
navigate("/home")
// History: [/, /home]

// Replace current entry
navigate("/home", { replace: true })
// History: [/home]  (/ is replaced)

// Go back
navigate(-1)
// History: [/]

// Go forward
navigate(1)
// History: [/, /home]
```

**Browser Integration**:
- Back button: Goes to previous route
- Forward button: Goes to next route (if available)
- Refresh: Re-renders current route
- Bookmark: Saves current URL

### Passing State Between Routes

**Not Currently Used, But Available**:

```javascript
// Navigate with state
navigate("/profile", {
  state: { from: "/home", userId: 123 }
})

// Receive state in destination component
import { useLocation } from 'react-router-dom'

function Profile() {
  const location = useLocation()
  console.log(location.state)  // { from: "/home", userId: 123 }
}
```

**Use Cases**:
- Pass form data to confirmation page
- Track where user came from
- Pass temporary data (not in URL)

**Limitation**:
- State lost on page refresh
- Not shareable via URL

---

## URL Parameters and Query Strings

### Current State: Not Used

The application doesn't use:
- Route parameters: `/games/:id`
- Query strings: `/search?query=zelda`
- URL state management

### Potential Use Cases

**1. Game Detail Pages**:
```javascript
// Route definition
<Route path="/games/:gameId" element={<GameDetail />} />

// Navigation
<Link to={`/games/${game.id}`}>View Details</Link>

// Access parameter
import { useParams } from 'react-router-dom'

function GameDetail() {
  const { gameId } = useParams()
  // Fetch game data using gameId
}
```

**Benefit**: Shareable URLs for specific games

**2. Search and Filters**:
```javascript
// Route definition
<Route path="/search" element={<SearchResults />} />

// Navigation with query string
navigate("/search?query=zelda&genre=rpg")

// Access query parameters
import { useSearchParams } from 'react-router-dom'

function SearchResults() {
  const [searchParams, setSearchParams] = useSearchParams()
  const query = searchParams.get("query")  // "zelda"
  const genre = searchParams.get("genre")  // "rpg"

  // Update query parameters
  const handleFilter = (newGenre) => {
    setSearchParams({ query, genre: newGenre })
  }
}
```

**Benefit**: Shareable search results, back button works correctly

**3. Pagination**:
```javascript
// Navigation
<Link to="/action?page=2">Page 2</Link>

// Access page number
const page = searchParams.get("page") || 1
```

---

## Server-Side Routing Configuration

### Problem: SPA Routes Don't Exist on Server

When user navigates from / to /action:
- React Router handles navigation (no server request)
- URL changes to /action in browser

When user refreshes /action or directly visits /action:
- Browser requests /action from server
- Server doesn't have /action file (404 error)

### Solution: Redirect All Requests to index.html

**Netlify Configuration (Netlify.toml)**:
```toml
[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

**Alternative: public/_redirects**:
```
/* /index.html 200
```

**Behavior**:
- All 404 requests redirect to index.html
- React app loads
- React Router reads URL path
- Renders matching route

**Testing**:
```bash
# Direct navigation works
curl https://gamerator.netlify.app/action
# Returns index.html, React Router matches /action
```

---

## Navigation Performance

### Current State

**Pros**:
- Instant navigation (no page reloads)
- Shared components (Header, Navbar, Footer) don't re-render
- Browser back/forward very fast

**Cons**:
- All routes in single bundle (no code splitting)
- No loading states during data fetching
- No prefetching (could preload data on hover)

### Optimization Strategies

**1. Code Splitting with React.lazy()**:
```javascript
import { lazy, Suspense } from 'react'

const Home = lazy(() => import('./Home'))
const Action = lazy(() => import('./Action'))
const Leaderboard = lazy(() => import('./Leaderboard'))

<Routes>
  <Route
    path="/home"
    element={
      <Suspense fallback={<div>Loading...</div>}>
        <Home />
      </Suspense>
    }
  />
</Routes>
```

**Benefit**: Smaller initial bundle, faster first page load

**2. Prefetching on Hover**:
```javascript
const PrefetchLink = ({ to, children }) => {
  const handleMouseEnter = () => {
    // Prefetch data for destination route
    fetch(`/api${to}`)
  }

  return (
    <Link to={to} onMouseEnter={handleMouseEnter}>
      {children}
    </Link>
  )
}
```

**Benefit**: Data ready when user clicks

**3. Loading States**:
```javascript
function Action() {
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    // Fetch data
    setLoading(false)
  }, [])

  if (loading) return <LoadingSpinner />

  return <GameCardContainer gameType="action" />
}
```

---

## Navigation Events and Tracking

### Tracking Page Views

**Not Currently Implemented**:

```javascript
import { useEffect } from 'react'
import { useLocation } from 'react-router-dom'

function Analytics() {
  const location = useLocation()

  useEffect(() => {
    // Track page view
    if (window.gtag) {
      window.gtag('config', 'GA_MEASUREMENT_ID', {
        page_path: location.pathname + location.search,
      })
    }
  }, [location])

  return null
}

// App.js
<BrowserRouter>
  <Analytics />
  <App />
</BrowserRouter>
```

**Tracks**:
- Route changes
- User navigation patterns
- Most visited pages

### Scroll Management

**Current Behavior**:
- Scroll position preserved between navigations
- User scrolled down on /action, navigates to /adventure, stays scrolled

**Better UX: Scroll to Top on Navigation**:
```javascript
import { useEffect } from 'react'
import { useLocation } from 'react-router-dom'

function ScrollToTop() {
  const location = useLocation()

  useEffect(() => {
    window.scrollTo(0, 0)
  }, [location.pathname])

  return null
}

// App.js
<BrowserRouter>
  <ScrollToTop />
  <App />
</BrowserRouter>
```

**Alternative: Scroll Restoration**:
```javascript
// Restore scroll position when going back
<BrowserRouter>
  <ScrollRestoration />  // React Router experimental feature
  <App />
</BrowserRouter>
```

---

## Error Handling and 404 Pages

### Current State: No 404 Handling

**Problem**:
- User visits /nonexistent
- No route matches
- Renders nothing (blank page)

### Recommended: Catch-All Route

```javascript
<Routes>
  <Route path="/" element={<FrontPage />} />
  <Route path="/home" element={<Home />} />
  {/* ... other routes */}

  {/* Catch-all route (must be last) */}
  <Route path="*" element={<NotFound />} />
</Routes>

// NotFound.js
function NotFound() {
  return (
    <div className="not-found">
      <h1>404 - Page Not Found</h1>
      <p>The page you're looking for doesn't exist.</p>
      <Link to="/home">Go Home</Link>
    </div>
  )
}
```

---

## Navigation Testing

### Manual Testing Checklist

**Basic Navigation**:
- [ ] Clicking navbar links changes route
- [ ] URL updates in address bar
- [ ] Back button returns to previous page
- [ ] Forward button works after going back
- [ ] Refresh page loads correct route

**Authentication Integration**:
- [ ] Accessing protected route redirects to login when not authenticated
- [ ] After login, redirects to intended route
- [ ] Logout redirects to login page
- [ ] Accessing login page when authenticated redirects to home

**Direct URL Access**:
- [ ] Typing /action in address bar loads Action page (if authenticated)
- [ ] Typing /action in address bar redirects to login (if not authenticated)
- [ ] Refreshing /action stays on /action

### Automated Testing

**Unit Tests (Navigation Logic)**:
```javascript
import { render, screen } from '@testing-library/react'
import { MemoryRouter } from 'react-router-dom'
import App from './App'

test('renders home page at /home', () => {
  // Mock authenticated state
  localStorage.setItem('loginState', 'true')

  render(
    <MemoryRouter initialEntries={['/home']}>
      <App />
    </MemoryRouter>
  )

  expect(screen.getByText('Welcome to Gamerator!')).toBeInTheDocument()
})

test('redirects to login when accessing protected route unauthenticated', () => {
  localStorage.setItem('loginState', 'false')

  render(
    <MemoryRouter initialEntries={['/action']}>
      <App />
    </MemoryRouter>
  )

  // Should redirect to FrontPage
  expect(screen.getByText('Sign In with Google')).toBeInTheDocument()
})
```

**Integration Tests (User Flows)**:
```javascript
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { BrowserRouter } from 'react-router-dom'
import App from './App'

test('navigates between pages', async () => {
  const user = userEvent.setup()

  // Mock authenticated state
  localStorage.setItem('loginState', 'true')

  render(
    <BrowserRouter>
      <App />
    </BrowserRouter>
  )

  // Click Action link
  await user.click(screen.getByText('Action'))

  // Should navigate to /action
  expect(window.location.pathname).toBe('/action')
  expect(screen.getByText('Action Games')).toBeInTheDocument()

  // Click Leaderboard link
  await user.click(screen.getByText('Leaderboard'))

  // Should navigate to /leaderboard
  expect(window.location.pathname).toBe('/leaderboard')
  expect(screen.getByText('Top Rated Games')).toBeInTheDocument()
})
```

---

## Accessibility Considerations

### Current State

**Good Practices**:
- Uses semantic `<a>` tags (via Link component)
- Keyboard accessible (Tab, Enter to navigate)
- Focus management (default browser behavior)

**Missing**:
- No skip navigation link
- No ARIA landmarks
- No focus indication on custom styled links
- No route announcements for screen readers

### Recommended Improvements

**1. Skip Navigation Link**:
```javascript
// App.js
<a href="#main-content" className="skip-link">
  Skip to main content
</a>

<Header />
<Navbar />
<main id="main-content">
  <Outlet />
</main>
<Footer />
```

**CSS**:
```css
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  background: #000;
  color: #fff;
  padding: 8px;
  text-decoration: none;
}

.skip-link:focus {
  top: 0;
}
```

**2. Route Announcements**:
```javascript
import { useEffect } from 'react'
import { useLocation } from 'react-router-dom'

function RouteAnnouncer() {
  const location = useLocation()

  useEffect(() => {
    // Announce route change to screen readers
    const message = document.querySelector('[role="status"]')
    if (message) {
      message.textContent = `Navigated to ${location.pathname}`
    }
  }, [location])

  return (
    <div role="status" aria-live="polite" aria-atomic="true" className="sr-only">
      {/* Screen reader announcement */}
    </div>
  )
}
```

**3. Focus Management**:
```javascript
import { useEffect, useRef } from 'react'

function PageComponent() {
  const headingRef = useRef(null)

  useEffect(() => {
    // Focus heading on page load
    headingRef.current?.focus()
  }, [])

  return <h1 ref={headingRef} tabIndex={-1}>Page Title</h1>
}
```

---

## Future Routing Enhancements

### Phase 1: Refactoring
1. Extract ProtectedRoute component
2. Implement Layout with Outlet
3. Add 404 Not Found page
4. Scroll to top on navigation

### Phase 2: Advanced Features
1. Code splitting with React.lazy()
2. Loading states with Suspense
3. Route-based data fetching
4. Prefetching on hover

### Phase 3: Dynamic Routes
1. Game detail pages (/games/:id)
2. User profile pages (/users/:userId)
3. Search results (/search?query=zelda)
4. Pagination (/action?page=2)

### Phase 4: Nested Routes
1. Category pages with sub-routes
   - /action (overview)
   - /action/top-rated
   - /action/new-releases
2. Settings pages
   - /settings (overview)
   - /settings/profile
   - /settings/privacy

---

## Conclusion

Gamerator's routing architecture effectively demonstrates React Router v6 fundamentals in a Single Page Application. The flat route structure keeps navigation simple and predictable, while authentication-based protection ensures secure access to content.

**Current Strengths**:
- Clean URL structure
- Instant navigation (no page reloads)
- Browser history integration
- Working authentication guards

**Areas for Improvement**:
- Reduce route definition repetition with ProtectedRoute component
- Implement Layout pattern with Outlet
- Add 404 handling
- Code splitting for performance
- Better accessibility (route announcements, focus management)

For the current scope, the routing implementation is solid and maintainable. As the application grows, adopting nested routes, dynamic segments, and code splitting will improve both developer experience and user experience.
