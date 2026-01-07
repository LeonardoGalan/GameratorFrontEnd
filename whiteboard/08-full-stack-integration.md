# Full-Stack Integration

## Overview

Gamerator is a full-stack application consisting of a React frontend and Node.js/Express backend. This document provides a comprehensive view of how both systems integrate, communicate, and work together to deliver a cohesive game rating platform.

---

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                      COMPLETE SYSTEM ARCHITECTURE                    │
└─────────────────────────────────────────────────────────────────────┘

                           End Users (Browsers)
                                   │
                                   │ HTTPS
                                   ↓
┌──────────────────────────────────────────────────────────────────────┐
│                        FRONTEND LAYER                                 │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Netlify CDN (Global Distribution)                             │ │
│  │  - HTTPS Termination                                           │ │
│  │  - Static Asset Hosting                                        │ │
│  │  - SPA Routing (Redirects)                                     │ │
│  └────────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  React SPA (Build Output)                                      │ │
│  │  - Client-side Routing (React Router v6)                       │ │
│  │  - Component Rendering                                         │ │
│  │  - State Management (Context + localStorage)                   │ │
│  │  - API Communication (Fetch API)                               │ │
│  └────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
                                   │
                                   │ HTTP (Port 8080)
                                   │ JSON Payload
                                   ↓
┌──────────────────────────────────────────────────────────────────────┐
│                        BACKEND LAYER                                  │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Heroku Platform                                               │ │
│  │  - HTTPS Termination                                           │ │
│  │  - Load Balancing                                              │ │
│  │  - Auto-scaling                                                │ │
│  └────────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Express.js Application                                        │ │
│  │  - REST API Endpoints                                          │ │
│  │  - CORS Middleware                                             │ │
│  │  - Request Validation                                          │ │
│  │  - Business Logic                                              │ │
│  └────────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Sequelize ORM                                                 │ │
│  │  - Model Definitions                                           │ │
│  │  - Query Building                                              │ │
│  │  - Migrations & Associations                                   │ │
│  └────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
                                   │
                                   │ PostgreSQL Protocol
                                   ↓
┌──────────────────────────────────────────────────────────────────────┐
│                        DATA LAYER                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  PostgreSQL Database (Heroku Postgres)                         │ │
│  │  Tables:                                                        │ │
│  │  - games (RAWG data + aggregated ratings)                      │ │
│  │  - users (email, timestamps)                                   │ │
│  │  - votes (user_id, game_id, rating, one-per-user)              │ │
│  └────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
                                   │
                                   │ (Initial Seed)
                                   ↓
┌──────────────────────────────────────────────────────────────────────┐
│                     EXTERNAL SERVICES                                 │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Google OAuth 2.0 (accounts.google.com)                        │ │
│  │  - User Authentication                                         │ │
│  │  - Email Verification                                          │ │
│  └────────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  RAWG.io API (rawg.io/api)                                     │ │
│  │  - Game Metadata (descriptions, images, ratings)               │ │
│  │  - Used for Initial Database Seeding (One-time)                │ │
│  └────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Repository Structure

### Frontend Repository
**GitHub**: `mattlol85/GameratorFrontEnd`

**Tech Stack**:
- React 17.0.2
- React Router v6.2.1
- react-google-login 5.2.2
- Create React App 5.0.0

**Deployment**:
- Netlify (primary)
- GitHub Pages (secondary)

**Key Files**:
```
gamerator/
├── src/components/
│   ├── App.js              # Root component, routing
│   ├── FrontPage.js        # Login page
│   ├── GameCardContainer.js # Fetches games from API
│   ├── GameCardPopup.js    # Voting interface
│   └── Leaderboard.js      # Top-rated games
├── public/
│   ├── index.html
│   └── _redirects          # Netlify SPA config
└── package.json
```

### Backend Repository
**GitHub**: `mattlol85/Gamerator_Server`

**Tech Stack**:
- Node.js / Express.js
- Sequelize ORM
- PostgreSQL
- Axios (for RAWG API)

**Deployment**:
- Heroku

**Key Files**:
```
server/
├── models/
│   ├── Game.js             # Game model
│   ├── User.js             # User model
│   └── Vote.js             # Vote model
├── routes/
│   ├── games.js            # Game endpoints
│   ├── users.js            # User endpoints
│   └── votes.js            # Voting endpoints
├── seeders/
│   └── rawgSeeder.js       # RAWG data importer
└── server.js               # Express app entry
```

---

## API Contract

### Base URL Configuration

**Frontend**:
```javascript
const BASE_URL = `http://${process.env.REACT_APP_HOSTNAME}:8080`
```

**Environments**:
- **Development**: `http://localhost:8080`
- **Production**: `https://gamerator-server.herokuapp.com:8080`

**CORS Configuration** (Backend):
```javascript
// server.js
const cors = require('cors')

const corsOptions = {
  origin: [
    'http://localhost:3000',           // Local development
    'https://gamerator.netlify.app',   // Production frontend
    'https://mattlol85.github.io'      // GitHub Pages
  ],
  credentials: true
}

app.use(cors(corsOptions))
```

---

## API Endpoints

### 1. User Registration

**Frontend Call** (FrontPage.js):
```javascript
fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/user/${userEmail}`, {
  method: "POST"
})
```

**Backend Route**:
```javascript
// POST /user/:email
router.post('/user/:email', async (req, res) => {
  const { email } = req.params

  try {
    // findOrCreate: Returns existing user or creates new one
    const [user, created] = await User.findOrCreate({
      where: { email },
      defaults: { email }
    })

    res.status(200).json({
      user,
      created  // true if new user, false if existing
    })
  } catch (error) {
    res.status(500).json({ error: error.message })
  }
})
```

**Database Operation**:
```sql
-- If user doesn't exist
INSERT INTO users (email, created_at, updated_at)
VALUES ('john.doe@gmail.com', NOW(), NOW())
RETURNING *;

-- If user exists
SELECT * FROM users WHERE email = 'john.doe@gmail.com';
```

**Frontend Handling**:
- Fire-and-forget (doesn't await response)
- Assumes success
- No error handling

**Backend Response** (Not Consumed):
```json
{
  "user": {
    "id": 123,
    "email": "john.doe@gmail.com",
    "createdAt": "2024-01-15T10:30:00.000Z",
    "updatedAt": "2024-01-15T10:30:00.000Z"
  },
  "created": true
}
```

---

### 2. Fetch Games by Category

**Frontend Call** (GameCardContainer.js):
```javascript
fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/${gameType}/limit=24`)
  .then(res => res.json())
  .then(data => setGames(data))
```

**Backend Route**:
```javascript
// GET /:category/limit=:limit
router.get('/:category/limit=:limit', async (req, res) => {
  const { category, limit } = req.params

  try {
    const games = await Game.findAll({
      where: {
        genres: {
          [Op.contains]: [category]  // PostgreSQL JSONB array contains
        }
      },
      limit: parseInt(limit),
      order: [['metacritic', 'DESC']]  // Highest rated first
    })

    res.status(200).json(games)
  } catch (error) {
    res.status(500).json({ error: error.message })
  }
})
```

**Database Query**:
```sql
SELECT * FROM games
WHERE genres @> '["action"]'::jsonb
ORDER BY metacritic DESC
LIMIT 24;
```

**Response Format**:
```json
[
  {
    "id": 3498,
    "name": "Grand Theft Auto V",
    "background_image": "https://media.rawg.io/media/games/456/...",
    "description": "<p>HTML description...</p>",
    "description_raw": "Plain text description...",
    "developers": [
      { "id": 3524, "name": "Rockstar Games" }
    ],
    "genres": [
      { "id": 4, "name": "Action" },
      { "id": 3, "name": "Adventure" }
    ],
    "released": "2013-09-17",
    "publishers": [
      { "id": 2155, "name": "Rockstar Games" }
    ],
    "esrb_rating": {
      "id": 4,
      "name": "Mature"
    },
    "metacritic": 97,
    "screenshots": [
      { "id": 1, "image": "https://..." },
      { "id": 2, "image": "https://..." }
    ],
    "website": "https://www.rockstargames.com/V/",
    "reddit_url": "https://www.reddit.com/r/GrandTheftAutoV/",
    "ratings_count": 5000,
    "tags": [
      { "id": 31, "name": "Singleplayer" },
      { "id": 7, "name": "Multiplayer" }
    ],
    "our_rating": 4.5  // Aggregated from votes table
  },
  // ... 23 more games
]
```

**Data Enrichment** (Backend):
- Games seeded from RAWG.io initially
- `our_rating` calculated from votes table
- Updated via trigger/aggregation query

---

### 3. Fetch Leaderboard

**Frontend Call** (Leaderboard.js):
```javascript
fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/leaderboard`)
  .then(res => res.json())
  .then(data => setTopGames(data))
```

**Backend Route**:
```javascript
// GET /leaderboard
router.get('/leaderboard', async (req, res) => {
  try {
    const categories = ['action', 'adventure', 'indie', 'shooter', 'rpg']
    const leaderboard = {}

    for (const category of categories) {
      const topGames = await Game.findAll({
        where: {
          genres: {
            [Op.contains]: [category]
          }
        },
        attributes: [
          'name',
          'metacritic',
          'rawg_rating',
          [sequelize.fn('AVG', sequelize.col('votes.rating')), 'our_rating']
        ],
        include: [{
          model: Vote,
          attributes: [],
          required: false
        }],
        group: ['Game.id'],
        order: [[sequelize.literal('our_rating'), 'DESC']],
        limit: 10
      })

      leaderboard[category] = topGames
    }

    res.status(200).json(leaderboard)
  } catch (error) {
    res.status(500).json({ error: error.message })
  }
})
```

**Database Query** (Per Category):
```sql
SELECT
  games.name,
  games.metacritic,
  games.rawg_rating,
  AVG(votes.rating) AS our_rating
FROM games
LEFT JOIN votes ON games.id = votes.game_id
WHERE games.genres @> '["action"]'::jsonb
GROUP BY games.id
ORDER BY our_rating DESC
LIMIT 10;
```

**Response Format**:
```json
{
  "action": [
    {
      "name": "The Witcher 3: Wild Hunt",
      "metacritic": 92,
      "rawg_rating": 4.66,
      "our_rating": 4.8
    },
    // ... 9 more
  ],
  "adventure": [ ... ],
  "indie": [ ... ],
  "shooter": [ ... ],
  "rpg": [ ... ]
}
```

---

### 4. Check Vote Eligibility

**Frontend Call** (GameCardPopup.js):
```javascript
fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/user/${loggedinUserId}/${gameId}`, {
  method: "PUT"
})
  .then(res => res.json())
  .then(canVote => {
    if (canVote) {
      // Submit vote
    } else {
      alert("You have already voted for this game!")
    }
  })
```

**Backend Route**:
```javascript
// PUT /user/:email/:gameId
router.put('/user/:email/:gameId', async (req, res) => {
  const { email, gameId } = req.params

  try {
    // Find user by email
    const user = await User.findOne({ where: { email } })
    if (!user) {
      return res.status(404).json({ error: 'User not found' })
    }

    // Check if vote exists
    const existingVote = await Vote.findOne({
      where: {
        user_id: user.id,
        game_id: parseInt(gameId)
      }
    })

    // Return true if no vote found (can vote), false if vote exists
    res.status(200).json(!existingVote)
  } catch (error) {
    res.status(500).json({ error: error.message })
  }
})
```

**Database Query**:
```sql
-- Find user
SELECT * FROM users WHERE email = 'john.doe@gmail.com';

-- Check for existing vote
SELECT * FROM votes
WHERE user_id = 123 AND game_id = 3498;
```

**Response**:
```json
true   // Can vote
false  // Already voted
```

**Issue**: Using PUT for read operation (should be GET)

---

### 5. Submit Vote

**Frontend Call** (GameCardPopup.js):
```javascript
fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/${gameId}/${rating}`, {
  method: "PUT"
})
```

**Backend Route**:
```javascript
// PUT /:gameId/:rating
router.put('/:gameId/:rating', async (req, res) => {
  const { gameId, rating } = req.params

  try {
    // Create vote record
    const vote = await Vote.create({
      game_id: parseInt(gameId),
      rating: parseInt(rating),
      // Note: user_id should be passed but isn't in current implementation
      // This is a bug - votes not properly attributed to users
    })

    // Update game's aggregated rating
    const avgRating = await Vote.findOne({
      where: { game_id: gameId },
      attributes: [
        [sequelize.fn('AVG', sequelize.col('rating')), 'avg_rating']
      ]
    })

    await Game.update(
      { our_rating: avgRating.dataValues.avg_rating },
      { where: { id: gameId } }
    )

    res.status(200).json({ success: true, vote })
  } catch (error) {
    res.status(500).json({ error: error.message })
  }
})
```

**Database Operations**:
```sql
-- Insert vote
INSERT INTO votes (game_id, rating, created_at, updated_at)
VALUES (3498, 5, NOW(), NOW());

-- Calculate average rating
SELECT AVG(rating) as avg_rating
FROM votes
WHERE game_id = 3498;

-- Update game
UPDATE games
SET our_rating = 4.5
WHERE id = 3498;
```

**Issue**: User ID not passed from frontend, breaking vote attribution and one-vote-per-user enforcement

---

## Data Models

### Frontend Data Structures

**User Context**:
```javascript
{
  userId: "john.doe@gmail.com",  // User's email
  setUserId: Function              // Update function
}
```

**Game Object** (from API):
```javascript
{
  id: number,
  name: string,
  background_image: string,
  description: string,
  description_raw: string,
  developers: Array<{id: number, name: string}>,
  genres: Array<{id: number, name: string}>,
  released: string,  // YYYY-MM-DD
  publishers: Array<{id: number, name: string}>,
  esrb_rating: {id: number, name: string} | null,
  metacritic: number,
  screenshots: Array<{id: number, image: string}>,
  website: string,
  reddit_url: string,
  ratings_count: number,
  tags: Array<{id: number, name: string}>,
  our_rating: number  // Calculated by backend
}
```

**Leaderboard Data**:
```javascript
{
  [category: string]: Array<{
    name: string,
    metacritic: number,
    rawg_rating: number,
    our_rating: number
  }>
}
```

### Backend Database Schema

**Users Table**:
```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

**Games Table**:
```sql
CREATE TABLE games (
  id INTEGER PRIMARY KEY,              -- RAWG game ID
  name VARCHAR(255) NOT NULL,
  background_image TEXT,
  description TEXT,
  description_raw TEXT,
  developers JSONB,                    -- [{id, name}, ...]
  genres JSONB,                        -- [{id, name}, ...]
  released DATE,
  publishers JSONB,
  esrb_rating JSONB,
  metacritic INTEGER,
  screenshots JSONB,
  website TEXT,
  reddit_url TEXT,
  ratings_count INTEGER,
  tags JSONB,
  rawg_rating DECIMAL(3,2),
  our_rating DECIMAL(3,2),             -- Aggregated from votes
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Index for category queries
CREATE INDEX idx_games_genres ON games USING GIN (genres);
```

**Votes Table**:
```sql
CREATE TABLE votes (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
  game_id INTEGER REFERENCES games(id) ON DELETE CASCADE,
  rating INTEGER CHECK (rating >= 1 AND rating <= 5),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  UNIQUE (user_id, game_id)            -- One vote per user per game
);

-- Indexes for vote queries
CREATE INDEX idx_votes_user_id ON votes(user_id);
CREATE INDEX idx_votes_game_id ON votes(game_id);
CREATE UNIQUE INDEX idx_votes_user_game ON votes(user_id, game_id);
```

---

## Authentication Flow (Full Stack)

```
┌─────────────────────────────────────────────────────────────────────┐
│              COMPLETE AUTHENTICATION FLOW                            │
└─────────────────────────────────────────────────────────────────────┘

1. User Visits Frontend
   ↓
2. React App Loads (Netlify)
   ↓
3. App.js Checks localStorage.loginState
   ↓ (if "false" or null)
4. Redirect to FrontPage (Login)
   ↓
5. User Clicks "Sign In with Google"
   ↓
6. Google OAuth Popup Opens
   ↓
7. User Authorizes Gamerator
   ↓
8. Google Returns OAuth Response
   ├─ accessToken (not stored)
   ├─ tokenId (not stored)
   └─ profileObj.email (stored in localStorage)
   ↓
9. Frontend: POST /user/{email} to Backend
   ↓
10. Backend: Find or Create User in PostgreSQL
    ├─ SELECT * FROM users WHERE email = ?
    └─ INSERT INTO users (email) VALUES (?) [if new]
    ↓
11. Backend: Return User Object
    ↓
12. Frontend: Store email in localStorage
    ├─ localStorage.setItem("userId", email)
    └─ localStorage.setItem("loginState", "true")
    ↓
13. Frontend: Navigate to /home
    ↓
14. Frontend: Display User Email in Navbar (from Context)
    ↓
15. User Authenticated ✓
```

**Security Notes**:
- OAuth token not sent to backend
- Only email stored (non-sensitive)
- Backend doesn't validate OAuth token
- One-vote enforcement relies on email uniqueness

---

## Voting Flow (Full Stack)

```
┌─────────────────────────────────────────────────────────────────────┐
│                   COMPLETE VOTING FLOW                               │
└─────────────────────────────────────────────────────────────────────┘

1. User Browses Games (Frontend)
   ├─ GET /{category}/limit=24
   ├─ Backend: SELECT * FROM games WHERE genres @> '["action"]'
   └─ Frontend: Render GameCards
   ↓
2. User Clicks "More Info" on Game
   ↓
3. GameCardPopup Opens (Modal)
   ↓
4. User Clicks Star Rating (1-5)
   ↓
5. Frontend: PUT /user/{email}/{gameId}
   ├─ Check if user can vote
   ↓
6. Backend: Check Vote Eligibility
   ├─ SELECT * FROM users WHERE email = ?
   ├─ SELECT * FROM votes WHERE user_id = ? AND game_id = ?
   └─ RETURN true (no vote) or false (vote exists)
   ↓
7. Frontend: Receive Response
   ↓
   ├─ If true (Can Vote):
   │  ├─ PUT /{gameId}/{rating}
   │  ├─ Backend: INSERT INTO votes (game_id, rating, user_id) VALUES (?, ?, ?)
   │  ├─ Backend: Calculate AVG(rating) for game
   │  ├─ Backend: UPDATE games SET our_rating = ? WHERE id = ?
   │  ├─ Backend: RETURN success
   │  └─ Frontend: Alert "Thank you for voting!"
   │
   └─ If false (Already Voted):
      └─ Frontend: Alert "You have already voted for this game!"
```

**Issues in Current Implementation**:
1. Vote submission doesn't pass user_id (bug)
2. One-vote enforcement not working properly
3. No optimistic UI update
4. Alert blocks UI thread

---

## Data Seeding Process

### Initial Game Data Population

**Source**: RAWG.io API

**Seeder Script** (Backend):
```javascript
// seeders/rawgSeeder.js
const axios = require('axios')
const { Game } = require('../models')

const RAWG_API_KEY = process.env.RAWG_API_KEY
const categories = ['action', 'adventure', 'indie', 'shooter', 'rpg']

async function seedGames() {
  for (const category of categories) {
    const response = await axios.get('https://api.rawg.io/api/games', {
      params: {
        key: RAWG_API_KEY,
        genres: category,
        page_size: 40,
        ordering: '-metacritic'
      }
    })

    for (const game of response.data.results) {
      await Game.upsert({
        id: game.id,
        name: game.name,
        background_image: game.background_image,
        description: game.description,
        description_raw: game.description_raw,
        developers: game.developers,
        genres: game.genres,
        released: game.released,
        publishers: game.publishers,
        esrb_rating: game.esrb_rating,
        metacritic: game.metacritic,
        screenshots: game.screenshots,
        website: game.website,
        reddit_url: game.reddit_url,
        ratings_count: game.ratings_count,
        tags: game.tags,
        rawg_rating: game.rating,
        our_rating: 0  // No votes yet
      })
    }
  }

  console.log('Seeding complete!')
}

seedGames()
```

**Execution**:
```bash
# Run once during setup
node seeders/rawgSeeder.js
```

**Result**:
- ~200 games (40 per category with overlap)
- Initial `our_rating` = 0
- Updated as users vote

---

## Error Handling Gaps

### Frontend Missing Error Handling

**1. Network Failures**:
```javascript
// Current: No error handling
fetch(url)
  .then(res => res.json())
  .then(data => setGames(data))

// Recommended:
fetch(url)
  .then(res => {
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    return res.json()
  })
  .then(data => setGames(data))
  .catch(error => {
    console.error('Failed to fetch games:', error)
    setError(error.message)
  })
```

**2. CORS Errors**:
- No user-facing message
- User sees blank page
- Should show "Backend unavailable" message

**3. Timeout Errors**:
- Fetch has no timeout by default
- User waits indefinitely
- Should implement AbortController with timeout

### Backend Missing Error Handling

**1. Database Connection Errors**:
```javascript
// Should implement connection pool error handling
sequelize.authenticate()
  .catch(err => {
    console.error('Database connection failed:', err)
    process.exit(1)
  })
```

**2. Validation Errors**:
- No input validation (email format, rating range)
- Sequelize validation exists but errors not formatted for frontend

**3. Rate Limiting**:
- No protection against spam
- Should implement express-rate-limit

---

## Performance Optimization Opportunities

### Frontend

**1. API Response Caching**:
```javascript
// React Query implementation
const { data: games } = useQuery(
  ['games', gameType],
  () => fetch(url).then(res => res.json()),
  {
    staleTime: 5 * 60 * 1000,  // Cache for 5 minutes
    cacheTime: 10 * 60 * 1000
  }
)
```

**2. Code Splitting**:
```javascript
const Leaderboard = lazy(() => import('./Leaderboard'))
```
- Reduces initial bundle size
- Faster time to interactive

**3. Image Lazy Loading**:
```javascript
<img src={game.background_image} loading="lazy" alt={game.name} />
```

### Backend

**1. Database Query Optimization**:
```javascript
// Add indexes
CREATE INDEX idx_games_metacritic ON games(metacritic DESC);

// Use query caching
const games = await Game.findAll({
  where: { ... },
  cache: true,
  cacheTTL: 300  // 5 minutes
})
```

**2. Response Compression**:
```javascript
// server.js
const compression = require('compression')
app.use(compression())
```

**3. Database Connection Pooling**:
```javascript
// config/database.js
const sequelize = new Sequelize(DATABASE_URL, {
  pool: {
    max: 10,
    min: 2,
    acquire: 30000,
    idle: 10000
  }
})
```

---

## Security Vulnerabilities

### Current Issues

**1. Vote Fraud**:
- User ID not passed in vote submission
- One-vote enforcement broken
- Users can vote multiple times

**Fix**:
```javascript
// Frontend: GameCardPopup.js
fetch(`http://.../${gameId}/${rating}`, {
  method: "PUT",
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    userId: loggedinUserId,
    rating: rating
  })
})

// Backend: votes route
router.put('/:gameId', async (req, res) => {
  const { gameId } = req.params
  const { userId, rating } = req.body

  const user = await User.findOne({ where: { email: userId } })

  const [vote, created] = await Vote.findOrCreate({
    where: { user_id: user.id, game_id: gameId },
    defaults: { rating }
  })

  if (!created) {
    return res.status(409).json({ error: 'Already voted' })
  }

  // Update aggregated rating...
})
```

**2. No Input Validation**:
```javascript
// Backend: Validate email format
const validator = require('validator')

if (!validator.isEmail(email)) {
  return res.status(400).json({ error: 'Invalid email' })
}

// Validate rating range
if (rating < 1 || rating > 5) {
  return res.status(400).json({ error: 'Rating must be 1-5' })
}
```

**3. No Rate Limiting**:
```javascript
// Backend: server.js
const rateLimit = require('express-rate-limit')

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100  // 100 requests per window
})

app.use('/api/', limiter)
```

**4. SQL Injection (Mitigated by Sequelize)**:
- Sequelize parameterizes queries automatically
- Still should validate inputs

**5. XSS (Mitigated by React)**:
- React escapes rendered content
- No `dangerouslySetInnerHTML` usage

---

## Deployment Dependencies

### Environment Variables

**Frontend** (.env):
```bash
REACT_APP_HOSTNAME=gamerator-server.herokuapp.com
REACT_APP_GOOGLEID=<client-id>.apps.googleusercontent.com
```

**Backend** (Heroku Config Vars):
```bash
DATABASE_URL=postgres://user:pass@host:5432/db
RAWG_API_KEY=<api-key>
PORT=8080
NODE_ENV=production
```

### Deployment Order

**1. Backend First**:
```bash
# Deploy backend to Heroku
cd Gamerator_Server
git push heroku main

# Run migrations
heroku run sequelize db:migrate

# Seed database
heroku run node seeders/rawgSeeder.js
```

**2. Frontend Second**:
```bash
# Set backend URL in Netlify
# Netlify dashboard → Environment variables
REACT_APP_HOSTNAME=gamerator-server.herokuapp.com

# Deploy frontend
cd GameratorFrontEnd
git push origin main
# Netlify auto-deploys
```

**Why This Order?**:
- Frontend depends on backend API
- Database must be seeded before frontend can display games
- Deploying frontend first results in API errors

---

## Testing Full-Stack Integration

### Manual Integration Tests

**1. End-to-End User Flow**:
```
1. Visit frontend URL
2. Click "Sign In with Google"
3. Authorize with Google account
4. Verify redirect to /home
5. Verify email displayed in navbar
6. Navigate to /action
7. Verify games load
8. Click "More Info" on a game
9. Click 5-star rating
10. Verify "Thank you for voting!" alert
11. Refresh page
12. Try voting again
13. Verify "Already voted" alert
14. Navigate to /leaderboard
15. Verify voted game appears with updated rating
```

**2. API Integration Tests**:
```bash
# Test backend endpoints directly
curl http://localhost:8080/action/limit=24
curl http://localhost:8080/leaderboard
curl -X POST http://localhost:8080/user/test@example.com
curl -X PUT http://localhost:8080/user/test@example.com/3498
curl -X PUT http://localhost:8080/3498/5
```

### Automated Integration Tests

**Recommended: Cypress E2E Tests**:
```javascript
// cypress/e2e/voting.cy.js
describe('Voting Flow', () => {
  beforeEach(() => {
    // Mock Google OAuth
    cy.intercept('POST', '**/user/**', { statusCode: 200 })
    cy.visit('/')
    cy.login()  // Custom command
  })

  it('allows user to vote for a game', () => {
    cy.visit('/action')
    cy.get('.gameCard').first().click()
    cy.get('[data-testid="star-5"]').click()
    cy.get('.alert').should('contain', 'Thank you for voting!')
  })

  it('prevents double voting', () => {
    cy.visit('/action')
    cy.get('.gameCard').first().click()
    cy.get('[data-testid="star-5"]').click()
    cy.wait(1000)
    cy.get('[data-testid="star-4"]').click()
    cy.get('.alert').should('contain', 'already voted')
  })

  it('updates leaderboard after vote', () => {
    // Vote for game
    cy.visit('/action')
    cy.get('.gameCard').first().as('game')
    cy.get('@game').find('.gameName').invoke('text').as('gameName')
    cy.get('@game').click()
    cy.get('[data-testid="star-5"]').click()

    // Check leaderboard
    cy.visit('/leaderboard')
    cy.get('@gameName').then(name => {
      cy.contains(name).should('be.visible')
    })
  })
})
```

---

## Monitoring Full-Stack Health

### Backend Health Check

**Recommended Endpoint**:
```javascript
// routes/health.js
router.get('/health', async (req, res) => {
  try {
    await sequelize.authenticate()
    res.status(200).json({
      status: 'healthy',
      database: 'connected',
      timestamp: new Date().toISOString()
    })
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      database: 'disconnected',
      error: error.message
    })
  }
})
```

**Frontend Monitoring**:
```javascript
// Check backend health on app load
useEffect(() => {
  fetch(`${API_URL}/health`)
    .then(res => res.json())
    .then(health => {
      if (health.status !== 'healthy') {
        console.warn('Backend unhealthy:', health)
      }
    })
    .catch(error => {
      console.error('Backend unreachable:', error)
      setBackendStatus('offline')
    })
}, [])
```

---

## Scalability Considerations

### Current Limitations

**Frontend**:
- Static assets scale infinitely (CDN)
- No server-side rendering (SEO limitations)
- No edge functions (all logic client-side)

**Backend**:
- Single Heroku dyno (limited concurrency)
- No horizontal scaling configured
- No caching layer (Redis)
- No load balancer
- Heroku Postgres has connection limits

### Scaling Strategy

**Phase 1: Vertical Scaling**
```bash
# Upgrade Heroku dyno
heroku dyno:resize standard-2x
```

**Phase 2: Horizontal Scaling**
```bash
# Add more dynos
heroku ps:scale web=3
```

**Phase 3: Database Optimization**
- Add Redis for query caching
- Implement connection pooling
- Database read replicas

**Phase 4: Microservices**
- Separate voting service
- Dedicated leaderboard service
- API gateway (Kong, AWS API Gateway)

---

## Disaster Recovery Plan

### Backup Strategy

**Database Backups** (Heroku Postgres):
```bash
# Automatic daily backups (Heroku handles)
# Manual backup
heroku pg:backups:capture --app gamerator-server

# Download backup
heroku pg:backups:download --app gamerator-server
```

**Code Repositories**:
- Frontend: GitHub (mattlol85/GameratorFrontEnd)
- Backend: GitHub (mattlol85/Gamerator_Server)
- Full git history preserved

### Recovery Procedures

**Scenario 1: Database Corruption**
```bash
# Restore from backup
heroku pg:backups:restore <backup-id> DATABASE_URL --app gamerator-server
```

**Scenario 2: Backend Down**
```bash
# Check dyno status
heroku ps --app gamerator-server

# Restart dynos
heroku restart --app gamerator-server

# View logs
heroku logs --tail --app gamerator-server
```

**Scenario 3: Frontend Down**
```bash
# Rollback Netlify deploy
netlify rollback

# Or redeploy from GitHub
git push origin main --force
```

---

## Future Enhancements

### Phase 1: Fix Critical Issues
1. Fix vote attribution (pass user_id)
2. Implement proper one-vote enforcement
3. Add error handling to all API calls
4. Add loading states

### Phase 2: Improve UX
1. Real-time vote updates (WebSockets)
2. Optimistic UI for voting
3. User profile (view vote history)
4. Search functionality

### Phase 3: Advanced Features
1. Social features (share games, comments)
2. Game recommendations (ML-based)
3. Email notifications (new games, trending)
4. Admin dashboard (manage games, moderate)

### Phase 4: Production Hardening
1. Comprehensive test coverage (E2E, integration, unit)
2. Performance monitoring (Sentry, DataDog)
3. Rate limiting and DDoS protection
4. Database optimization (indexes, caching)
5. CDN optimization (image CDN, lazy loading)

---

## Conclusion

Gamerator demonstrates a functional full-stack application architecture with clear separation between frontend and backend concerns. The React SPA frontend communicates with a RESTful Express backend, with PostgreSQL as the persistent data store.

**Current State**: Educational, functional, demonstrates core concepts

**Production-Ready Requirements**:
- Fix vote attribution bug
- Comprehensive error handling
- Security hardening (rate limiting, validation)
- Performance optimization (caching, code splitting)
- Monitoring and observability
- Automated testing (E2E, integration)

The architecture provides a solid foundation for future enhancements while maintaining simplicity and clarity—ideal for a learning project that showcases full-stack development skills.
