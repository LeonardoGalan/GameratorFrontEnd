# Data Flow and API Integration

## Overview

The Gamerator frontend communicates with the backend via RESTful HTTP endpoints using the native Fetch API. All data fetching is component-driven with no centralized API layer or caching strategy. This document details every API integration point, data transformation, and flow pattern in the application.

---

## API Configuration

### Base URL Construction
```javascript
const BASE_URL = `http://${process.env.REACT_APP_HOSTNAME}:8080`
```

**Environment Variable**:
- `REACT_APP_HOSTNAME`: Backend server hostname (e.g., `localhost` or `gamerator-server.herokuapp.com`)

**Build-Time Injection**:
- Environment variables must have `REACT_APP_` prefix (Create React App requirement)
- Variables injected during `npm start` or `npm run build`
- No runtime configuration possible (static builds)

### Connection Details
- **Protocol**: HTTP (HTTPS in production via Heroku)
- **Port**: 8080
- **Content-Type**: application/json (implicit)
- **Authentication**: None (user email in URL path)
- **CORS**: Backend must allow frontend origin

---

## API Endpoints

### Summary Table

| Method | Endpoint | Purpose | Component | Request Body | Success Response |
|--------|----------|---------|-----------|--------------|------------------|
| POST | `/user/{email}` | Register user | FrontPage | None | 200 OK |
| GET | `/{gameType}/limit=24` | Fetch games | GameCardContainer | None | Game[] |
| GET | `/leaderboard` | Fetch top games | Leaderboard | None | LeaderboardData |
| PUT | `/user/{email}/{gameId}` | Check vote eligibility | GameCardPopup | None | boolean |
| PUT | `/{gameId}/{rating}` | Submit vote | GameCardPopup | None | 200 OK |

---

## Detailed API Specifications

### 1. User Registration

**Endpoint**: `POST /user/{email}`

**Purpose**: Register new user or acknowledge existing user

**Request**:
```javascript
fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/user/john.doe@gmail.com`, {
  method: "POST"
})
```

**Path Parameters**:
- `email` (string): User's Google account email address

**Request Body**: None

**Success Response**:
- **Status**: 200 OK
- **Body**: N/A (not consumed by frontend)

**Error Response**:
- Not explicitly handled by frontend
- Network errors caught by browser (CORS, DNS, etc.)

**Frontend Usage** (FrontPage.js:17-21):
```javascript
const responseGoogle = (response) => {
  const userEmail = response.profileObj.email

  fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/user/${userEmail}`, {
    method: "POST",
  })

  localStorage.setItem("userId", userEmail)
  localStorage.setItem("loginState", "true")
  navigate("/home")
}
```

**Flow Diagram**:
```
Google OAuth Success → Extract email → POST /user/{email} →
Store in localStorage → Navigate to /home
```

**Issues**:
- No error handling for failed registration
- No loading state during API call
- Fire-and-forget (doesn't wait for response)
- Could create race condition if backend is slow

---

### 2. Fetch Games by Category

**Endpoint**: `GET /{gameType}/limit=24`

**Purpose**: Retrieve games for a specific category with hardcoded limit

**Request**:
```javascript
fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/action/limit=24`)
```

**Path Parameters**:
- `gameType` (string): Category name ("action", "adventure", "indie", "shooter", "rpg")
- `limit` (number): Maximum games to return (hardcoded to 24)

**Request Body**: None

**Success Response**:
- **Status**: 200 OK
- **Content-Type**: application/json
- **Body**: Array of game objects

**Game Object Structure**:
```javascript
{
  id: number,                    // Unique game ID
  name: string,                  // Game title
  background_image: string,      // Main image URL
  description: string,           // HTML description
  description_raw: string,       // Plain text description
  developers: [                  // Developer array
    { id: number, name: string }
  ],
  genres: [                      // Genre array
    { id: number, name: string }
  ],
  released: string,              // Release date (YYYY-MM-DD)
  publishers: [                  // Publisher array
    { id: number, name: string }
  ],
  esrb_rating: {                 // ESRB object or null
    id: number,
    name: string                 // "Everyone", "Teen", etc.
  },
  metacritic: number,            // Metacritic score (0-100)
  screenshots: [                 // Screenshot array
    { id: number, image: string }
  ],
  website: string,               // Official website URL
  reddit_url: string,            // Reddit community URL
  ratings_count: number,         // Number of ratings
  tags: [                        // Tag array
    { id: number, name: string }
  ]
}
```

**Frontend Usage** (GameCardContainer.js:9-14):
```javascript
useEffect(() => {
  fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/${gameType}/limit=24`)
    .then((res) => res.json())
    .then((data) => {
      setGames(data)
    })
}, [gameType])
```

**Flow Diagram**:
```
Component Mount/gameType Change → Fetch Games →
Parse JSON → Update games state → Render GameCards
```

**Data Transformations**:
```javascript
// Backend returns array of games
data = [{game1}, {game2}, ...]

// Frontend stores directly in state
setGames(data)  // No transformation

// Frontend extracts first developer
developer = game.developers?.[0]?.name || "Unknown"

// Frontend formats release date (no transformation, displays as-is)
releaseDate = game.released  // "2023-11-15"
```

**Issues**:
- No error handling for failed fetch
- No loading state (instant render with empty array)
- Hardcoded limit (not configurable)
- No pagination (always shows first 24)
- Re-fetches entire dataset on every navigation
- No caching (refetches even if user revisits category)

---

### 3. Fetch Leaderboard

**Endpoint**: `GET /leaderboard`

**Purpose**: Retrieve top-rated games for all categories

**Request**:
```javascript
fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/leaderboard`)
```

**Path Parameters**: None

**Request Body**: None

**Success Response**:
- **Status**: 200 OK
- **Content-Type**: application/json
- **Body**: Object with category keys

**Leaderboard Object Structure**:
```javascript
{
  action: [
    {
      name: string,           // Game name
      metacritic: number,     // Metacritic score
      rawg_rating: number,    // RAWG rating
      our_rating: number      // Custom rating (user votes)
    },
    // ... more games
  ],
  adventure: [...],
  indie: [...],
  shooter: [...],
  rpg: [...]
}
```

**Frontend Usage** (Leaderboard.js:6-11):
```javascript
useEffect(() => {
  fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/leaderboard`)
    .then((res) => res.json())
    .then((data) => {
      setTopGames(data)
    })
}, [])
```

**Flow Diagram**:
```
Component Mount → Fetch Leaderboard → Parse JSON →
Update topGames state → Render category sections
```

**Rendering Logic** (Leaderboard.js:16-26):
```javascript
{Object.keys(topGames).map(category => (
  <div key={category}>
    <h2>{category.toUpperCase()}</h2>
    <ul>
      {topGames[category].map(game => (
        <li key={game.name}>
          {game.name} - Metacritic: {game.metacritic},
          RAWG: {game.rawg_rating}, Our Rating: {game.our_rating}
        </li>
      ))}
    </ul>
  </div>
))}
```

**Data Transformations**:
```javascript
// Backend returns object with category keys
data = { action: [], adventure: [], ... }

// Frontend extracts keys
Object.keys(topGames)  // ["action", "adventure", ...]

// Frontend displays as uppercase
category.toUpperCase()  // "ACTION"
```

**Issues**:
- No error handling
- No loading state
- Assumes all categories present in response
- No handling for empty categories
- Ratings displayed as raw numbers (no formatting)

---

### 4. Check Vote Eligibility

**Endpoint**: `PUT /user/{email}/{gameId}`

**Purpose**: Check if user has already voted for a game

**Request**:
```javascript
fetch(`http://${...}:8080/user/john.doe@gmail.com/123`, {
  method: "PUT"
})
```

**Path Parameters**:
- `email` (string): User's email address
- `gameId` (number): Game ID to check

**Request Body**: None

**Success Response**:
- **Status**: 200 OK
- **Content-Type**: application/json
- **Body**: `true` (can vote) or `false` (already voted)

**Frontend Usage** (GameCardPopup.js:14-31):
```javascript
const handleRating = (rating) => {
  fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/user/${loggedinUserId}/${gameId}`, {
    method: "PUT",
  })
    .then((res) => res.json())
    .then((data) => {
      if (data) {
        // User can vote, proceed to submit
        fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/${gameId}/${rating}`, {
          method: "PUT",
        })
        alert("Thank you for voting!")
      } else {
        // User already voted
        alert("You have already voted for this game!")
      }
    })
}
```

**Flow Diagram**:
```
User Clicks Star → Check Eligibility →
If true: Submit Vote + Show success alert
If false: Show "already voted" alert
```

**Issues**:
- PUT method for read operation (should be GET)
- No error handling for network failures
- No loading state during check
- Alert blocks UI
- Could race if user clicks multiple stars quickly

---

### 5. Submit Vote

**Endpoint**: `PUT /{gameId}/{rating}`

**Purpose**: Record user's rating for a game

**Request**:
```javascript
fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/123/5`, {
  method: "PUT"
})
```

**Path Parameters**:
- `gameId` (number): Game ID to rate
- `rating` (number): Star rating (1-5)

**Request Body**: None

**Success Response**:
- **Status**: 200 OK
- **Body**: N/A (not consumed by frontend)

**Frontend Usage** (GameCardPopup.js:21-25):
```javascript
fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/${gameId}/${rating}`, {
  method: "PUT",
})

alert("Thank you for voting!")
```

**Flow Diagram**:
```
Eligibility Check Passes → Submit Vote →
Show "Thank you" alert
```

**Issues**:
- User email not included (backend infers from eligibility check?)
- No error handling
- Fire-and-forget (doesn't wait for response)
- Alert shows before response received (race condition)
- No visual feedback on success (just alert)
- Popup doesn't close after voting

---

## Data Flow Patterns

### Pattern 1: Authentication Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    AUTHENTICATION FLOW                       │
└─────────────────────────────────────────────────────────────┘

User Clicks "Sign In with Google"
          ↓
Google OAuth Dialog Opens
          ↓
User Authorizes App
          ↓
react-google-login Callback Fires
          ↓
Extract email from response.profileObj.email
          ↓
POST /user/{email}  ← Register user in backend
          ↓
Store in localStorage:
  - loginState = "true"
  - userId = email
          ↓
Navigate to /home
          ↓
App.js Re-reads localStorage
          ↓
Renders protected routes with Navbar
          ↓
Navbar reads userId from UserEmail Context
          ↓
Displays user email in nav bar
```

**Data Stores Updated**:
1. localStorage.loginState
2. localStorage.userId
3. Backend user table (via POST /user)

---

### Pattern 2: Game Browsing Flow

```
┌─────────────────────────────────────────────────────────────┐
│                   GAME BROWSING FLOW                         │
└─────────────────────────────────────────────────────────────┘

User Clicks "Action Games" Link
          ↓
React Router navigates to /action
          ↓
Action Component Renders
          ↓
Action renders GameCardContainer with gameType="action"
          ↓
GameCardContainer useEffect() Triggers
          ↓
GET /action/limit=24
          ↓
Backend queries PostgreSQL for action games
          ↓
Returns JSON array of 24 games
          ↓
.json() parses response
          ↓
setGames(data) updates component state
          ↓
Component re-renders with games data
          ↓
games.map() creates 24 GameCard components
          ↓
Each GameCard displays:
  - Game image
  - Game name
  - Developer name
  - "More Info" button
```

**Data Stores Updated**:
1. GameCardContainer.games state

---

### Pattern 3: Voting Flow

```
┌─────────────────────────────────────────────────────────────┐
│                      VOTING FLOW                             │
└─────────────────────────────────────────────────────────────┘

User Clicks "More Info" on Game Card
          ↓
reactjs-popup Opens Modal
          ↓
GameCardPopup Displays Game Details
          ↓
User Hovers Star (1-5)
          ↓
Star Image Changes (visual feedback)
          ↓
User Clicks Star (e.g., 5 stars)
          ↓
handleRating(5) Function Fires
          ↓
Read userId from UserEmail Context
          ↓
PUT /user/{userId}/{gameId}  ← Check eligibility
          ↓
Backend Queries votes table
          ↓
Backend Returns:
  - true: User hasn't voted
  - false: User already voted
          ↓
Frontend Receives Response
          ↓
IF true:
  └→ PUT /{gameId}/5  ← Submit vote
     └→ Backend inserts vote record
     └→ Backend updates game rating
     └→ Alert: "Thank you for voting!"
          ↓
IF false:
  └→ Alert: "You have already voted for this game!"
          ↓
User Dismisses Alert
          ↓
Popup Remains Open (no auto-close)
```

**Data Stores Updated**:
1. Backend votes table
2. Backend game rating aggregate

**Issues in Flow**:
- No optimistic UI update
- Alert blocks user interaction
- No visual feedback of recorded vote
- Rating state not updated locally
- User must close popup manually

---

### Pattern 4: Leaderboard Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    LEADERBOARD FLOW                          │
└─────────────────────────────────────────────────────────────┘

User Clicks "Leaderboard" Link
          ↓
React Router navigates to /leaderboard
          ↓
Leaderboard Component Mounts
          ↓
useEffect() Triggers
          ↓
GET /leaderboard
          ↓
Backend Aggregates Ratings:
  - Groups by category
  - Calculates average our_rating
  - Sorts by rating DESC
  - Takes top N per category
          ↓
Returns JSON Object:
{
  action: [{game1}, {game2}, ...],
  adventure: [...],
  ...
}
          ↓
.json() parses response
          ↓
setTopGames(data) updates state
          ↓
Component re-renders
          ↓
Object.keys(topGames) iterates categories
          ↓
Each category renders:
  - Category heading (uppercase)
  - List of top games
  - Metacritic, RAWG, and Our Rating scores
```

**Data Stores Updated**:
1. Leaderboard.topGames state

---

## State Management Integration

### Component State
```javascript
// GameCardContainer.js
const [games, setGames] = useState([])

// Fetch updates state
fetch(url)
  .then(res => res.json())
  .then(data => setGames(data))  // State update triggers re-render
```

### Context State
```javascript
// App.js (Provider)
const [userId, setUserId] = useState(localStorage.getItem("userId"))
<UserEmail.Provider value={{ userId, setUserId }}>

// GameCardPopup.js (Consumer)
const UserContext = useContext(UserEmail)
const loggedinUserId = UserContext.userId  // Used in API calls
```

### localStorage Persistence
```javascript
// FrontPage.js (Write)
localStorage.setItem("userId", userEmail)
localStorage.setItem("loginState", "true")

// App.js (Read on mount)
const [login, setLogin] = useState(localStorage.getItem("loginState"))
const [userId, setUserId] = useState(localStorage.getItem("userId"))
```

**Data Sync Issue**:
- localStorage and Context state can desync
- Context initialized from localStorage on mount
- Changes to Context don't update localStorage
- Logout clears localStorage but requires page reload

---

## Error Handling Strategies

### Current State: Minimal Error Handling

**No Try-Catch Blocks**:
```javascript
// All fetch calls lack error handling
fetch(url)
  .then(res => res.json())
  .then(data => setState(data))
// Missing: .catch(error => handleError(error))
```

**No Network Error Handling**:
- DNS failures: User sees blank page
- CORS errors: User sees blank page
- 404/500 errors: User sees blank page
- Timeout: User waits indefinitely

**No Loading States**:
- No spinners or skeletons
- Instant render with empty data
- User doesn't know if data is loading or failed

### Recommended Error Handling

**Pattern 1: Try-Catch with Async/Await**:
```javascript
useEffect(() => {
  const fetchGames = async () => {
    try {
      setLoading(true)
      setError(null)

      const res = await fetch(url)

      if (!res.ok) {
        throw new Error(`HTTP ${res.status}: ${res.statusText}`)
      }

      const data = await res.json()
      setGames(data)
    } catch (error) {
      console.error('Failed to fetch games:', error)
      setError(error.message)
    } finally {
      setLoading(false)
    }
  }

  fetchGames()
}, [gameType])
```

**Pattern 2: Retry Logic**:
```javascript
const fetchWithRetry = async (url, retries = 3) => {
  for (let i = 0; i < retries; i++) {
    try {
      const res = await fetch(url)
      if (res.ok) return res.json()
    } catch (error) {
      if (i === retries - 1) throw error
      await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)))
    }
  }
}
```

**Pattern 3: User-Friendly Error Messages**:
```javascript
{error && (
  <div className="error-message">
    <p>Failed to load games. Please try again.</p>
    <button onClick={refetch}>Retry</button>
  </div>
)}
```

---

## Loading States

### Current State: No Loading Indicators

**GameCardContainer Example**:
```javascript
const [games, setGames] = useState([])  // Starts as empty array

return (
  <div className="gameCardContainer">
    {games.map(game => <GameCard {...game} />)}  // Renders nothing initially
  </div>
)
```

**User Experience**:
- User sees empty page briefly
- No indication data is loading
- User doesn't know if app is working

### Recommended Loading States

**Pattern 1: Loading Boolean**:
```javascript
const [games, setGames] = useState([])
const [loading, setLoading] = useState(true)

useEffect(() => {
  setLoading(true)
  fetch(url)
    .then(res => res.json())
    .then(data => {
      setGames(data)
      setLoading(false)
    })
}, [gameType])

return (
  <div>
    {loading ? (
      <div className="spinner">Loading games...</div>
    ) : (
      <div className="gameCardContainer">
        {games.map(game => <GameCard {...game} />)}
      </div>
    )}
  </div>
)
```

**Pattern 2: Skeleton Screens**:
```javascript
{loading ? (
  <div className="gameCardContainer">
    {Array(24).fill(0).map((_, i) => <GameCardSkeleton key={i} />)}
  </div>
) : (
  <div className="gameCardContainer">
    {games.map(game => <GameCard {...game} />)}
  </div>
)}
```

**Pattern 3: Optimistic UI for Voting**:
```javascript
const [voted, setVoted] = useState(false)
const [optimisticRating, setOptimisticRating] = useState(null)

const handleRating = async (rating) => {
  // Optimistic update
  setVoted(true)
  setOptimisticRating(rating)

  try {
    const eligible = await checkEligibility()
    if (eligible) {
      await submitVote(rating)
    } else {
      // Revert optimistic update
      setVoted(false)
      setOptimisticRating(null)
      alert("Already voted")
    }
  } catch (error) {
    // Revert on error
    setVoted(false)
    setOptimisticRating(null)
    alert("Failed to submit vote")
  }
}
```

---

## Caching Strategies

### Current State: No Caching

**Issues**:
- Every navigation to /action refetches same 24 games
- Leaderboard refetches on every visit
- Wasted bandwidth and backend load
- Slower user experience

### Recommended Caching

**Pattern 1: Component-Level Cache**:
```javascript
const gameCache = {}

useEffect(() => {
  if (gameCache[gameType]) {
    setGames(gameCache[gameType])
    return
  }

  fetch(url)
    .then(res => res.json())
    .then(data => {
      gameCache[gameType] = data
      setGames(data)
    })
}, [gameType])
```

**Pattern 2: React Query (Recommended)**:
```javascript
import { useQuery } from 'react-query'

const { data: games, isLoading, error } = useQuery(
  ['games', gameType],
  () => fetch(url).then(res => res.json()),
  {
    staleTime: 5 * 60 * 1000,  // 5 minutes
    cacheTime: 10 * 60 * 1000, // 10 minutes
  }
)
```

**Pattern 3: localStorage Cache**:
```javascript
useEffect(() => {
  const cached = localStorage.getItem(`games_${gameType}`)
  const cacheTime = localStorage.getItem(`games_${gameType}_time`)

  if (cached && Date.now() - cacheTime < 5 * 60 * 1000) {
    setGames(JSON.parse(cached))
    return
  }

  fetch(url)
    .then(res => res.json())
    .then(data => {
      localStorage.setItem(`games_${gameType}`, JSON.stringify(data))
      localStorage.setItem(`games_${gameType}_time`, Date.now())
      setGames(data)
    })
}, [gameType])
```

---

## API Client Abstraction

### Current State: Direct Fetch Calls

**Issues**:
- Repeated URL construction
- Duplicated error handling
- Hard to mock for testing
- No central configuration

### Recommended: API Service Layer

**api.js**:
```javascript
const BASE_URL = `http://${process.env.REACT_APP_HOSTNAME}:8080`

class ApiError extends Error {
  constructor(status, message) {
    super(message)
    this.status = status
  }
}

const apiClient = {
  async get(endpoint) {
    const res = await fetch(`${BASE_URL}${endpoint}`)
    if (!res.ok) throw new ApiError(res.status, res.statusText)
    return res.json()
  },

  async put(endpoint) {
    const res = await fetch(`${BASE_URL}${endpoint}`, { method: 'PUT' })
    if (!res.ok) throw new ApiError(res.status, res.statusText)
    return res.json()
  },

  async post(endpoint) {
    const res = await fetch(`${BASE_URL}${endpoint}`, { method: 'POST' })
    if (!res.ok) throw new ApiError(res.status, res.statusText)
    return res.json()
  },
}

export const gameApi = {
  registerUser: (email) =>
    apiClient.post(`/user/${email}`),

  getGames: (gameType, limit = 24) =>
    apiClient.get(`/${gameType}/limit=${limit}`),

  getLeaderboard: () =>
    apiClient.get('/leaderboard'),

  checkVoteEligibility: (email, gameId) =>
    apiClient.put(`/user/${email}/${gameId}`),

  submitVote: (gameId, rating) =>
    apiClient.put(`/${gameId}/${rating}`),
}
```

**Usage**:
```javascript
import { gameApi } from '../utils/api'

useEffect(() => {
  gameApi.getGames(gameType)
    .then(games => setGames(games))
    .catch(error => setError(error))
}, [gameType])
```

---

## Data Validation

### Current State: No Validation

**Issues**:
- Assumes backend always returns valid data
- No null checks before rendering
- Array methods called on potentially undefined values

**Example Vulnerabilities**:
```javascript
// GameCard.js - Could crash if developers is undefined
developer={game.developers?.[0]?.name || "Unknown"}

// Leaderboard.js - Could crash if topGames is null
Object.keys(topGames).map(...)
```

### Recommended: Runtime Validation

**Pattern 1: PropTypes**:
```javascript
import PropTypes from 'prop-types'

GameCard.propTypes = {
  gameId: PropTypes.number.isRequired,
  name: PropTypes.string.isRequired,
  img: PropTypes.string,
  developer: PropTypes.string,
  // ... all props
}
```

**Pattern 2: Zod Schema Validation**:
```javascript
import { z } from 'zod'

const GameSchema = z.object({
  id: z.number(),
  name: z.string(),
  background_image: z.string().url().optional(),
  developers: z.array(z.object({
    id: z.number(),
    name: z.string(),
  })).optional(),
  // ... all fields
})

// Validate API response
const data = await apiClient.get('/action/limit=24')
const games = data.map(game => GameSchema.parse(game))
```

---

## WebSocket Potential (Future)

### Current Limitation: Polling Required

**Problem**:
- User votes are not reflected in real-time
- Leaderboard requires manual refresh
- Other users' votes don't show up

### Recommended: WebSocket Integration

**Server-Sent Events for Leaderboard**:
```javascript
useEffect(() => {
  const eventSource = new EventSource(`${BASE_URL}/leaderboard/stream`)

  eventSource.onmessage = (event) => {
    const updatedLeaderboard = JSON.parse(event.data)
    setTopGames(updatedLeaderboard)
  }

  return () => eventSource.close()
}, [])
```

**WebSocket for Vote Updates**:
```javascript
useEffect(() => {
  const ws = new WebSocket(`ws://${hostname}:8080/votes`)

  ws.onmessage = (event) => {
    const { gameId, newRating } = JSON.parse(event.data)
    // Update local game rating
    setGames(games => games.map(game =>
      game.id === gameId ? { ...game, our_rating: newRating } : game
    ))
  }

  return () => ws.close()
}, [])
```

---

## Performance Optimization

### Current Issues
- Fetch entire 24 games on every navigation
- No debouncing or throttling
- No request cancellation
- All images load at once

### Recommended Optimizations

**1. Request Cancellation (AbortController)**:
```javascript
useEffect(() => {
  const controller = new AbortController()

  fetch(url, { signal: controller.signal })
    .then(res => res.json())
    .then(data => setGames(data))
    .catch(error => {
      if (error.name === 'AbortError') return  // Cancelled, ignore
      setError(error)
    })

  return () => controller.abort()  // Cancel on unmount/gameType change
}, [gameType])
```

**2. Pagination**:
```javascript
const [page, setPage] = useState(1)
const GAMES_PER_PAGE = 12

fetch(`${url}?page=${page}&limit=${GAMES_PER_PAGE}`)

// Infinite scroll or "Load More" button
const loadMore = () => setPage(page + 1)
```

**3. Lazy Image Loading**:
```javascript
<img
  src={game.background_image}
  alt={game.name}
  loading="lazy"  // Native lazy loading
/>
```

**4. Request Deduplication**:
```javascript
const inFlightRequests = {}

const fetchGames = (gameType) => {
  if (inFlightRequests[gameType]) {
    return inFlightRequests[gameType]  // Return existing promise
  }

  const promise = fetch(url).then(res => res.json())
  inFlightRequests[gameType] = promise

  promise.finally(() => {
    delete inFlightRequests[gameType]
  })

  return promise
}
```

---

## Conclusion

The current data flow architecture is straightforward and functional but lacks production-ready features like error handling, loading states, and caching. The direct use of Fetch API makes the code simple but repetitive.

**Key Strengths**:
- Simple, easy to understand
- No over-engineering
- Clear data flow

**Key Opportunities**:
- Centralized API client
- Error handling and retry logic
- Loading states and skeletons
- Client-side caching
- Data validation
- Optimistic UI updates
- Real-time capabilities (WebSockets)

For a production application, adopting React Query or SWR would address most of these issues with minimal code changes. For the current educational context, the existing approach effectively demonstrates fundamental API integration patterns.
