# Component Architecture

## Overview

The Gamerator frontend follows a hierarchical component structure with clear separation between layout, pages, and reusable components. All components are functional components using React Hooks, with no class-based components in the codebase.

---

## Component Hierarchy

```
App.js (Root Component)
│
├── BrowserRouter
│   │
│   ├── UserEmail.Provider (Context)
│   │   │
│   │   └── Routes
│   │       │
│   │       ├── Route "/" (FrontPage - Login)
│   │       │   └── FrontPage
│   │       │
│   │       ├── Route "/home" (Protected)
│   │       │   ├── Header
│   │       │   ├── Navbar (consumes UserEmail context)
│   │       │   ├── Home
│   │       │   └── Footer
│   │       │
│   │       ├── Route "/action" (Protected)
│   │       │   ├── Header
│   │       │   ├── Navbar
│   │       │   ├── Action
│   │       │   │   └── GameCardContainer (gameType="action")
│   │       │   │       └── GameCard[] (mapped)
│   │       │   │           └── GameCardPopup (per card)
│   │       │   └── Footer
│   │       │
│   │       ├── Route "/adventure" (Protected)
│   │       │   ├── Header
│   │       │   ├── Navbar
│   │       │   ├── Adventure
│   │       │   │   └── GameCardContainer (gameType="adventure")
│   │       │   └── Footer
│   │       │
│   │       ├── Route "/indie" (Protected)
│   │       │   ├── Header
│   │       │   ├── Navbar
│   │       │   ├── Indie
│   │       │   │   └── GameCardContainer (gameType="indie")
│   │       │   └── Footer
│   │       │
│   │       ├── Route "/shooter" (Protected)
│   │       │   ├── Header
│   │       │   ├── Navbar
│   │       │   ├── Shooter
│   │       │   │   └── GameCardContainer (gameType="shooter")
│   │       │   └── Footer
│   │       │
│   │       ├── Route "/rpg" (Protected)
│   │       │   ├── Header
│   │       │   ├── Navbar
│   │       │   ├── Rpg
│   │       │   │   └── GameCardContainer (gameType="rpg")
│   │       │   └── Footer
│   │       │
│   │       └── Route "/leaderboard" (Protected)
│   │           ├── Header
│   │           ├── Navbar
│   │           ├── Leaderboard
│   │           └── Footer
```

---

## Component Categories

### 1. Root Component
- **App.js**: Application root with routing configuration

### 2. Layout Components
- **Header**: Application header with logo/title
- **Navbar**: Main navigation with category links
- **Footer**: Creator credits and links

### 3. Page Components
- **FrontPage**: Login page (unauthenticated)
- **Home**: Dashboard/welcome page (authenticated)
- **Action, Adventure, Indie, Shooter, Rpg**: Category pages (wrappers)
- **Leaderboard**: Top-rated games page

### 4. Data Components
- **GameCardContainer**: Fetches and displays game grid
- **GameCard**: Individual game card (presentational)
- **GameCardPopup**: Game details modal with voting

### 5. Utility Components
- **SmallCard**: Empty/unused component (legacy?)

---

## Detailed Component Breakdown

### App.js
**Location**: `src/components/App.js`

**Purpose**: Application root component managing routing and global state

**Key Features**:
- React Router configuration
- UserEmail Context Provider
- Protected route logic
- Login state management via localStorage

**State**:
```javascript
const [login, setLogin] = useState(localStorage.getItem("loginState") || "false")
const [userId, setUserId] = useState(localStorage.getItem("userId") || "null")
```

**Context Provided**:
```javascript
<UserEmail.Provider value={{ userId, setUserId }}>
```

**Route Protection Pattern**:
```javascript
{login === "true"
  ? <><Header /><Navbar /><PageComponent /><Footer /></>
  : <Navigate to="/" replace={true} />
}
```

**Responsibilities**:
- Mount/unmount lifecycle management
- Global state initialization
- Route definition
- Authentication gating

---

### FrontPage
**Location**: `src/components/FrontPage.js`

**Purpose**: Google OAuth login page

**Key Features**:
- Google OAuth button integration
- Success callback handling
- User registration via backend API
- localStorage persistence

**OAuth Flow**:
```javascript
responseGoogle = (response) => {
  const userEmail = response.profileObj.email

  // Register user in backend
  fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/user/${userEmail}`, {
    method: "POST"
  })

  // Store in localStorage
  localStorage.setItem("userId", userEmail)
  localStorage.setItem("loginState", "true")

  // Navigate to home
  navigate("/home")
}
```

**UI Elements**:
- Gamerator logo
- Google login button (react-google-login)
- Welcome text

**Error Handling**:
- Console logs OAuth failures
- No user-facing error messages

**CSS**: `FrontPage.css`

---

### Header
**Location**: `src/components/Header.js`

**Purpose**: Simple header with app branding

**Structure**:
```javascript
<div className="header">
  <h2>GAMERATOR</h2>
</div>
```

**Styling**: `Header.css`
- Centered text
- Dark background (#1B1B1B)
- Orange text color (#FF8D4A)

**Responsibilities**:
- Display app name
- Consistent branding across pages

---

### Navbar
**Location**: `src/components/Navbar.js`

**Purpose**: Main navigation bar with category links and user info

**Context Consumption**:
```javascript
const UserContext = useContext(UserEmail)
const loggedinUserEmail = UserContext.userId
```

**Navigation Links**:
- Home
- Action Games
- Adventure Games
- Indie Games
- Shooter Games
- RPG Games
- Leaderboard
- Logout

**Logout Handler**:
```javascript
const logout = () => {
  localStorage.clear()
  navigate("/")
  window.location.reload()
}
```

**UI Features**:
- Displays logged-in user email
- Hover effects on links
- Active link styling (could be added)

**CSS**: `Navbar.css`
- Dark theme with gradient accents
- Flexbox layout
- Impact font for category names
- Hover transforms (scale up)

**Responsiveness**:
- Flexbox wrapping for mobile
- Touch-friendly link sizes

---

### Footer
**Location**: `src/components/Footer.js`

**Purpose**: Creator credits and external links

**Structure**:
```javascript
<footer>
  <div>Powered By API from RAWG.io</div>
  <div>
    <ExternalLink href="https://linkedin.com/in/matt-fitzgerald">
      Matthew Fitzgerald
    </ExternalLink>
    // ... other creators
  </div>
  <div>© 2022 Gamerator</div>
</footer>
```

**External Link Component**:
- Uses `react-external-link` for safe external navigation
- Opens in new tab
- Prevents `window.opener` vulnerabilities

**CSS**: `Footer.css`
- Centered text
- Dark background
- Link hover effects

---

### Home
**Location**: `src/components/Home.js`

**Purpose**: Welcome dashboard after login

**Content**:
- Welcome message
- Instructions for using the app
- Link to backend GitHub repo
- Explanation of features

**Structure**:
```javascript
<div className="home">
  <h1>Welcome to Gamerator!</h1>
  <p>Browse games by category...</p>
  <p>Vote for your favorite games...</p>
  <p>Check the leaderboard...</p>
</div>
```

**CSS**: `Home.css`
- Centered content
- Dark background
- Orange accent headings

**Interactivity**: None (pure informational)

---

### Action, Adventure, Indie, Shooter, Rpg
**Locations**:
- `src/components/Action.js`
- `src/components/Adventure.js`
- `src/components/Indie.js`
- `src/components/Shooter.js`
- `src/components/Rpg.js`

**Purpose**: Category page wrappers

**Pattern** (identical across all):
```javascript
export default function Action() {
  return (
    <div className="games">
      <h1 style={{textAlign: 'center'}}>Action Games</h1>
      <GameCardContainer gameType="action" />
    </div>
  )
}
```

**Props Passed to GameCardContainer**:
- `gameType`: String matching backend category ("action", "adventure", etc.)

**CSS**: Category-specific CSS files
- Each has unique gradient background for header
- Action: Red gradient
- Adventure: Blue gradient
- Indie: Purple gradient
- Shooter: Orange gradient
- RPG: Green gradient

**Responsibilities**:
- Display category title
- Render game grid via GameCardContainer
- Apply category-specific styling

---

### GameCardContainer
**Location**: `src/components/GameCardContainer.js`

**Purpose**: Fetches and displays game grid for a category

**Props**:
```javascript
{ gameType }  // e.g., "action", "adventure"
```

**State**:
```javascript
const [games, setGames] = useState([])
```

**Data Fetching**:
```javascript
useEffect(() => {
  fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/${gameType}/limit=24`)
    .then(res => res.json())
    .then(data => setGames(data))
}, [gameType])
```

**Rendering**:
```javascript
<div className="gameCardContainer">
  {games.map(game => (
    <GameCard
      key={game.id}
      gameId={game.id}
      img={game.background_image}
      name={game.name}
      developer={game.developers?.[0]?.name || "Unknown"}
      // ... all game data passed as props
    />
  ))}
</div>
```

**CSS**: `GameCardContainer.css`
- Flexbox grid layout
- Responsive wrapping
- Gap spacing between cards
- Centered alignment

**Error Handling**: None (assumes successful fetch)

**Loading State**: None (no skeleton/spinner)

**Performance**:
- Fetches all 24 games at once
- No pagination
- No lazy loading
- Re-fetches on gameType change

---

### GameCard
**Location**: `src/components/GameCard.js`

**Purpose**: Individual game card (presentational component)

**Props** (extensive):
```javascript
{
  gameId, img, name, developer, description, descriptionRaw,
  genres, releaseDate, publisher, esrbRating, metacriticScore,
  screenshots, websiteLink, redditLink, ratingsCount, tags
}
```

**Structure**:
```javascript
<div className="gameCard">
  <img src={img} alt={name} />
  <div className="cardContent">
    <h3>{name}</h3>
    <p>{developer}</p>
  </div>
  <GameCardPopup
    trigger={<button className="popupButton">More Info</button>}
    // ... pass all props to popup
  />
</div>
```

**Interaction**:
- Displays game image, name, developer
- "More Info" button triggers popup
- Hover effects (scale, shadow)

**CSS**: `GameCard.css`
- Card layout with image on top
- Fixed dimensions (300x400px)
- Hover zoom effect (scale: 1.05)
- Box shadow on hover
- Rounded corners

**Responsibilities**:
- Present game summary
- Trigger detail popup
- Visual card design

---

### GameCardPopup
**Location**: `src/components/GameCardPopup.js`

**Purpose**: Modal showing full game details with voting

**Props**: Same as GameCard (all game data)

**Context Consumption**:
```javascript
const UserContext = useContext(UserEmail)
const loggedinUserId = UserContext.userId
```

**State**:
```javascript
const [rating, setRating] = useState(0)  // User's star selection (not used correctly)
```

**Popup Trigger**:
```javascript
<Popup
  trigger={<button>More Info</button>}
  modal
  nested
>
  {close => (
    // ... popup content
  )}
</Popup>
```

**Voting Logic**:
```javascript
const handleRating = (starRating) => {
  // Check if user can vote
  fetch(`http://${...}:8080/user/${loggedinUserId}/${gameId}`, {
    method: "PUT"
  })
  .then(res => res.json())
  .then(canVote => {
    if (canVote) {
      // Submit vote
      fetch(`http://${...}:8080/${gameId}/${starRating}`, {
        method: "PUT"
      })
      alert("Vote submitted!")
    } else {
      alert("You have already voted for this game")
    }
  })
}
```

**Content Sections**:
1. **Header**: Game name, close button
2. **Image**: Main game image
3. **Details**:
   - Description
   - Release date
   - Publisher
   - ESRB rating
   - Metacritic score
4. **Star Rating**: 5 clickable stars for voting
5. **Genres**: List of genre tags
6. **Screenshots**: Gallery of game screenshots
7. **External Links**: Website, Reddit

**CSS**: `GameCardPopup.css`
- Full-screen overlay
- Scrollable content
- Animated gradient background
- Star hover effects (glow)
- Screenshot grid layout

**Issues**:
- `rating` state not used effectively (should track user's selection)
- No loading state during vote submission
- No visual feedback for already-voted games
- Popup doesn't close automatically after voting

---

### Leaderboard
**Location**: `src/components/Leaderboard.js`

**Purpose**: Display top-rated games across all categories

**State**:
```javascript
const [topGames, setTopGames] = useState({})
```

**Data Fetching**:
```javascript
useEffect(() => {
  fetch(`http://${process.env.REACT_APP_HOSTNAME}:8080/leaderboard`)
    .then(res => res.json())
    .then(data => setTopGames(data))
}, [])
```

**Backend Response Structure**:
```javascript
{
  action: [{ name, metacritic, rawg_rating, our_rating }, ...],
  adventure: [...],
  indie: [...],
  shooter: [...],
  rpg: [...]
}
```

**Rendering**:
```javascript
<div className="leaderboard">
  <h1>Top Rated Games</h1>
  {Object.keys(topGames).map(category => (
    <div key={category} className="categorySection">
      <h2>{category.toUpperCase()}</h2>
      <ul>
        {topGames[category].map(game => (
          <li>
            {game.name} - Metacritic: {game.metacritic},
            RAWG: {game.rawg_rating},
            Our Rating: {game.our_rating}
          </li>
        ))}
      </ul>
    </div>
  ))}
</div>
```

**CSS**: `Leaderboard.css`
- Multi-column layout (CSS Grid or Flexbox)
- Category-specific header colors
- List styling with ratings display

**Features**:
- Shows top N games per category (backend determines count)
- Displays 3 types of ratings: Metacritic, RAWG, custom
- No filtering or sorting options

**Missing Features**:
- No pagination
- No search
- No filtering by rating source
- No links to game details

---

## Component Communication Patterns

### Props Drilling
Most data flows via props:
```
App → Route → Page Component → GameCardContainer → GameCard → GameCardPopup
```

Example: Game data props passed through 3 levels.

### Context API
User email shared across components:
```javascript
// App.js (Provider)
<UserEmail.Provider value={{ userId, setUserId }}>

// Navbar.js (Consumer)
const UserContext = useContext(UserEmail)
const email = UserContext.userId

// GameCardPopup.js (Consumer)
const UserContext = useContext(UserEmail)
const email = UserContext.userId
```

### localStorage
Persistent state across sessions:
```javascript
// Write
localStorage.setItem("loginState", "true")
localStorage.setItem("userId", email)

// Read
const login = localStorage.getItem("loginState")
const userId = localStorage.getItem("userId")

// Clear
localStorage.clear()  // On logout
```

---

## Component State Management

### Local State (useState)
- **App**: login, userId
- **GameCardContainer**: games[]
- **Leaderboard**: topGames{}
- **GameCardPopup**: rating (unused effectively)

### Global State (Context)
- **UserEmail**: userId, setUserId

### Persistent State (localStorage)
- **loginState**: "true" | "false"
- **userId**: email string | "null"

---

## Component Styling Approach

### CSS File Organization
Each component has its own CSS file:
```
src/components/
├── App.js
├── styles/
│   ├── App.css
│   ├── FrontPage.css
│   ├── Header.css
│   ├── Navbar.css
│   ├── Footer.css
│   ├── Home.css
│   ├── Action.css
│   ├── Adventure.css
│   ├── Indie.css
│   ├── Shooter.css
│   ├── Rpg.css
│   ├── GameCard.css
│   ├── GameCardContainer.css
│   ├── GameCardPopup.css
│   └── Leaderboard.css
```

### Inline Styles
Rarely used, only for dynamic styles:
```javascript
<h1 style={{ textAlign: 'center' }}>Action Games</h1>
```

### CSS Conventions
- Class-based selectors (e.g., `.gameCard`)
- No BEM or CSS modules
- Global namespace (potential collisions)
- Flexbox for layouts
- Hover effects with transitions

---

## Component Reusability

### Highly Reusable
- **GameCard**: Pure presentational, no side effects
- **GameCardPopup**: Reusable modal (via reactjs-popup)
- **Header/Footer**: Static layout components

### Moderately Reusable
- **GameCardContainer**: Reusable but tightly coupled to backend API structure
- **Navbar**: Could be reused with different links

### Not Reusable
- **Category Pages** (Action, Adventure, etc.): Hardcoded titles and gameType
- **FrontPage**: Google OAuth specific
- **Leaderboard**: Specific to backend leaderboard endpoint

---

## Component Testing Strategy

### Current State
- No component tests written
- Testing library dependencies installed but unused

### Recommended Tests

**App.js**:
- Renders login page when not authenticated
- Renders home page when authenticated
- Redirects to login on logout

**FrontPage**:
- Renders Google login button
- Calls backend API on successful OAuth
- Navigates to /home after login

**GameCardContainer**:
- Fetches games on mount
- Renders correct number of GameCard components
- Passes props correctly to GameCard

**GameCard**:
- Renders game image, name, developer
- Opens popup on button click

**GameCardPopup**:
- Displays all game details
- Handles star rating clicks
- Shows alert on vote submission
- Prevents double voting

**Navbar**:
- Displays user email from context
- Navigates on link clicks
- Clears localStorage on logout

**Leaderboard**:
- Fetches leaderboard data on mount
- Renders all categories
- Displays ratings correctly

---

## Component Performance Considerations

### Current Performance
- **No Memoization**: Components re-render on any parent state change
- **No Code Splitting**: All components in single bundle
- **No Lazy Loading**: All images load immediately

### Optimization Opportunities

**React.memo**:
```javascript
export default React.memo(GameCard)
```
Prevents re-renders when props haven't changed.

**useMemo**:
```javascript
const sortedGames = useMemo(() =>
  games.sort((a, b) => b.rating - a.rating),
  [games]
)
```
Cache expensive computations.

**React.lazy + Suspense**:
```javascript
const Leaderboard = React.lazy(() => import('./Leaderboard'))

<Suspense fallback={<div>Loading...</div>}>
  <Leaderboard />
</Suspense>
```
Load components on-demand.

**Image Optimization**:
- Use `loading="lazy"` on img tags
- Serve WebP format
- Implement blur-up placeholders

---

## Component Accessibility

### Current Accessibility
- Semantic HTML (header, footer, nav)
- Alt text on some images
- Button elements (not div onClick)

### Accessibility Gaps
- No ARIA labels
- No keyboard navigation for popups
- No focus management
- No screen reader announcements
- Star rating not keyboard accessible

### Recommended Improvements
```javascript
// Keyboard-accessible star rating
<button
  aria-label={`Rate ${name} ${star} stars`}
  onClick={() => handleRating(star)}
  onKeyPress={(e) => e.key === 'Enter' && handleRating(star)}
>
  <img src={starImage} alt="" role="presentation" />
</button>
```

---

## Component Error Boundaries

### Current State
No error boundaries implemented.

### Recommended Implementation
```javascript
// ErrorBoundary.js
class ErrorBoundary extends React.Component {
  state = { hasError: false }

  static getDerivedStateFromError(error) {
    return { hasError: true }
  }

  componentDidCatch(error, errorInfo) {
    console.error('Component error:', error, errorInfo)
  }

  render() {
    if (this.state.hasError) {
      return <h1>Something went wrong. Please refresh.</h1>
    }
    return this.props.children
  }
}

// Usage in App.js
<ErrorBoundary>
  <Routes>...</Routes>
</ErrorBoundary>
```

---

## Future Component Architecture

### Potential Improvements

1. **Component Library**:
   - Extract reusable Button, Card, Modal components
   - Create design system (colors, spacing, typography)

2. **State Management**:
   - Migrate to React Query for server state
   - Keep Context API for UI state (theme, user)

3. **Type Safety**:
   - Migrate to TypeScript
   - PropTypes for immediate type checking

4. **Code Organization**:
   ```
   src/
   ├── components/
   │   ├── layout/      (Header, Navbar, Footer)
   │   ├── pages/       (FrontPage, Home, Category pages)
   │   ├── features/    (GameCard, Leaderboard)
   │   └── common/      (Button, Modal, etc.)
   ├── hooks/           (useAuth, useFetch, useLocalStorage)
   ├── contexts/        (UserContext, ThemeContext)
   └── utils/           (api.js, constants.js)
   ```

5. **Testing**:
   - Unit tests for all components
   - Integration tests for user flows
   - E2E tests with Cypress/Playwright

---

## Conclusion

The component architecture follows React best practices with functional components and hooks. The structure is straightforward and easy to understand, making it ideal for learning and collaboration. While there's room for optimization and feature enhancements, the current architecture provides a solid foundation for a voting-based game discovery platform.

**Key Strengths**:
- Clear component hierarchy
- Consistent patterns across similar components
- Good separation of layout, pages, and data components

**Key Opportunities**:
- Add loading and error states
- Improve reusability with more generic components
- Implement performance optimizations
- Add comprehensive testing
- Enhance accessibility
