# Gamerator Frontend - System Architecture Overview

## Executive Summary

The Gamerator frontend is a React-based Single Page Application (SPA) that provides a video game rating and discovery platform. Users authenticate via Google OAuth, browse games by category, and submit one-time ratings. The frontend integrates with a Node.js/Express backend API and deploys to Netlify for global CDN distribution.

---

## High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER BROWSER                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                  GAMERATOR SPA (React)                    │  │
│  │                                                            │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │  │
│  │  │   Routing   │  │    Auth     │  │    State    │      │  │
│  │  │ React Router│  │   Google    │  │   Context   │      │  │
│  │  │     v6      │  │    OAuth    │  │ localStorage│      │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘      │  │
│  │                                                            │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │           Component Layer                          │  │  │
│  │  │                                                      │  │  │
│  │  │  Navbar  │  GameCardContainer  │  GameCardPopup   │  │  │
│  │  │  Header  │  GameCard           │  Leaderboard     │  │  │
│  │  │  Footer  │  Category Pages     │  FrontPage       │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │                                                            │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │           API Communication Layer                  │  │  │
│  │  │           Fetch API (Native Browser)               │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ HTTPS
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    EXTERNAL SERVICES                             │
│                                                                  │
│  ┌──────────────────┐        ┌──────────────────────────────┐  │
│  │  Google OAuth    │        │    Gamerator Backend API     │  │
│  │  accounts.google │        │    (Heroku/Express)          │  │
│  │  .com            │        │    Port 8080                 │  │
│  └──────────────────┘        │                              │  │
│                              │  GET  /action/limit=24       │  │
│                              │  GET  /leaderboard           │  │
│                              │  POST /user/{email}          │  │
│                              │  PUT  /user/{email}/{gameId} │  │
│                              │  PUT  /{gameId}/{rating}     │  │
│                              └──────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    BACKEND DATA LAYER                            │
│                                                                  │
│  ┌──────────────────┐        ┌──────────────────────────────┐  │
│  │  PostgreSQL DB   │        │    RAWG.io API              │  │
│  │  (Heroku)        │        │    (Game Data Source)        │  │
│  │                  │        │    External                  │  │
│  │  - games         │        └──────────────────────────────┘  │
│  │  - users         │                                          │
│  │  - votes         │                                          │
│  └──────────────────┘                                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Technology Stack

### Core Framework
- **React 17.0.2**: Component-based UI library
- **React DOM 17.0.2**: React rendering engine for web
- **Create React App 5.0.0**: Zero-config build tooling

### Routing
- **React Router DOM v6.2.1**: Client-side routing with declarative navigation

### Authentication
- **react-google-login 5.2.2**: Google OAuth integration for user authentication

### UI Components
- **reactjs-popup 2.0.5**: Modal/popup component for game details
- **react-external-link 1.2.2**: Safe external link handling

### HTTP Communication
- **Fetch API**: Native browser API for HTTP requests
- **axios 0.25.0**: Installed but currently unused (future migration option)

### Styling
- **Plain CSS**: Component-scoped CSS files, no preprocessors
- **Flexbox**: Layout system for responsive design

### Testing
- **Jest**: Test runner (via react-scripts)
- **@testing-library/react 12.1.2**: React component testing utilities
- **@testing-library/jest-dom 5.16.1**: Custom Jest matchers

### Deployment
- **Netlify**: Primary hosting platform with CDN
- **gh-pages 3.2.3**: Alternative GitHub Pages deployment

### Build Tools
- **react-scripts 5.0.0**: Webpack, Babel, ESLint configuration abstraction
- **Webpack**: Module bundler (internal to react-scripts)
- **Babel**: JavaScript transpiler (internal to react-scripts)

---

## Key Architectural Principles

### 1. Single Page Application (SPA)
- All routing handled client-side via React Router
- No page reloads after initial load
- Netlify redirects all 404s to index.html for SPA support

### 2. Component-Based Architecture
- Reusable, composable React components
- Separation of concerns: layout, data fetching, presentation
- Props-based data flow with Context API for shared state

### 3. Authentication-First Design
- All routes (except login) are protected
- Google OAuth as sole authentication method
- User email as primary identifier
- One-vote-per-user-per-game enforcement

### 4. API-Driven Data
- No local data storage beyond user session
- All game data fetched from backend API
- Real-time voting with immediate server updates
- Backend as single source of truth

### 5. Mobile-First Responsive Design
- Flexbox-based layouts adapt to all screen sizes
- Touch-friendly UI elements
- PWA manifest for installable web app

### 6. Performance Optimization
- Code splitting via React Router lazy loading (potential)
- Asset optimization via Create React App build process
- CDN distribution via Netlify
- Minimal JavaScript bundle (focused dependencies)

---

## System Boundaries

### What the Frontend Does
1. User authentication via Google OAuth
2. Protected route management
3. Game browsing by category
4. Game detail viewing with modal popups
5. User voting/rating submission (1-5 stars)
6. Leaderboard visualization
7. Navigation between categories
8. Session persistence (localStorage)

### What the Backend Does
1. User registration and management
2. Game data storage (seeded from RAWG.io)
3. Vote storage and validation
4. One-vote-per-user enforcement
5. Leaderboard calculation
6. Rating aggregation
7. Data persistence (PostgreSQL)

### External Dependencies
1. **Google OAuth**: Identity provider
2. **Backend API**: Data and business logic
3. **Netlify**: CDN and hosting
4. **RAWG.io**: Game metadata source (backend integration)

---

## Data Flow Overview

### Authentication Flow
```
User → Google OAuth → Frontend receives token →
POST /user/{email} to backend → localStorage stores userId →
Redirect to /home → Navbar shows user email
```

### Game Browsing Flow
```
User navigates to /action → Action component renders →
GameCardContainer fetches GET /action/limit=24 →
Backend returns game array → GameCards rendered in grid →
User clicks card → GameCardPopup shows details
```

### Voting Flow
```
User clicks star (1-5) in popup →
PUT /user/{email}/{gameId} checks if eligible →
If true: PUT /{gameId}/{rating} submits vote →
If false: Alert "Already voted"
```

### Leaderboard Flow
```
User navigates to /leaderboard → Leaderboard component →
GET /leaderboard fetches top games per category →
Display in multi-column layout with ratings
```

---

## Environment Configuration

### Environment Variables (`.env`)
```
REACT_APP_HOSTNAME=<backend-server-hostname>
REACT_APP_GOOGLEID=<google-oauth-client-id>
```

### Build-Time Configuration
- Environment variables injected during `npm run build`
- Different values for development vs production
- No runtime configuration (static builds)

### Deployment Configuration

**Netlify.toml**:
```toml
[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

**public/_redirects**:
```
/* /index.html 200
```

Both ensure SPA routing works correctly on Netlify.

---

## Security Considerations

### Authentication
- Google OAuth token validation
- User email stored in localStorage (not sensitive data)
- No passwords or credentials stored client-side

### API Communication
- CORS configured on backend to allow frontend origin
- No API keys exposed (backend handles RAWG.io API key)
- HTTP-only communication (HTTPS in production)

### Vote Integrity
- One-vote-per-user enforced by backend
- Frontend checks voting eligibility before submission
- User email tied to vote records

### XSS Prevention
- React automatically escapes rendered content
- No `dangerouslySetInnerHTML` usage
- External links use react-external-link for safety

---

## Performance Characteristics

### Bundle Size
- Minimal dependencies (< 500KB gzipped)
- React + React Router + OAuth library as core
- No heavy state management libraries

### Loading Strategy
- Single initial bundle load
- Lazy loading potential for routes (not yet implemented)
- Image optimization via Create React App

### Caching Strategy
- Netlify CDN caching for static assets
- No client-side data caching (always fresh from backend)
- Service worker potential via PWA manifest (not configured)

---

## Scalability Considerations

### Current Architecture
- Static frontend, scales via CDN
- Backend API is bottleneck
- No client-side rate limiting

### Future Enhancements
1. Implement React.lazy() for route-based code splitting
2. Add service worker for offline support
3. Implement client-side caching with stale-while-revalidate
4. Add loading skeletons for better perceived performance
5. Migrate to React 18 for concurrent rendering
6. Consider React Query for server state management
7. Implement pagination for game categories (currently 24 limit)

---

## Development Workflow

### Local Development
```bash
npm start           # Starts dev server on http://localhost:3000
npm test            # Runs Jest test suite
npm run build       # Creates production bundle
```

### Deployment
```bash
# Netlify (automatic via git push)
git push origin main → Netlify auto-deploys

# GitHub Pages (manual)
npm run deploy      # Builds and pushes to gh-pages branch
```

---

## Monitoring and Observability

### Current State
- No error tracking (e.g., Sentry)
- No analytics (e.g., Google Analytics)
- Console.log for debugging (manual)

### Recommended Additions
1. Error boundary components for graceful error handling
2. Sentry or similar for error tracking
3. Google Analytics for user behavior insights
4. Performance monitoring (Web Vitals already included)

---

## Architecture Decisions

### Why React Router v6?
- Declarative routing matches React philosophy
- Client-side routing for SPA experience
- Easy protected route implementation with `Navigate` component

### Why Google OAuth Only?
- Simplifies authentication logic
- No password management burden
- Most users have Google accounts
- Reduces vote fraud (verified email addresses)

### Why Context API Instead of Redux?
- Simple state requirements (just user email)
- Avoids Redux boilerplate
- Context + localStorage sufficient for this app
- No complex async state management needed

### Why Fetch Instead of Axios?
- Native browser API (no extra dependency)
- Sufficient for simple GET/PUT/POST requests
- Axios installed but unused (future migration path)

### Why Netlify Instead of Heroku?
- Specialized for static frontend hosting
- Free CDN included
- Git-based deployment workflow
- Faster global distribution than Heroku

---

## Known Limitations

1. **No Pagination**: Categories show only 24 games (hardcoded limit)
2. **No Search**: Users can't search for specific games
3. **No Filtering**: Can't filter by release date, rating, etc.
4. **No User Profile**: No view of user's voting history
5. **No Real-time Updates**: Ratings don't update without page refresh
6. **No Error Recovery**: Network failures show alerts, no retry logic
7. **No Loading States**: No skeleton screens or spinners during data fetches

---

## Integration Points

### With Backend API
- Base URL: `http://${REACT_APP_HOSTNAME}:8080`
- Authentication: None (open API, user email in request path)
- Content-Type: application/json
- Error Handling: Basic try/catch with alerts

### With Google OAuth
- Client ID: `REACT_APP_GOOGLEID`
- Scopes: profile, email
- Callback: Handled by react-google-login library
- Token Storage: Not stored (only email extracted)

### With Netlify
- Build Command: `npm run build`
- Publish Directory: `build/`
- Environment Variables: Set in Netlify dashboard
- Redirects: Configured via Netlify.toml

---

## Conclusion

The Gamerator frontend is a well-structured, educational React application demonstrating core SPA concepts. It leverages modern React patterns (hooks, functional components, Context API) while maintaining simplicity. The architecture prioritizes developer experience and ease of understanding over complex optimizations, making it ideal for a team learning full-stack development.

**Primary Strengths**:
- Clear separation of concerns
- Simple, predictable data flow
- Minimal dependencies
- Easy to deploy and maintain

**Areas for Growth**:
- State management for complex features
- Performance optimizations
- User experience enhancements (loading states, error recovery)
- Advanced React patterns (code splitting, suspense)
