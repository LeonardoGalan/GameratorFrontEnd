# Deployment and Infrastructure

## Overview

The Gamerator frontend is deployed as a static Single Page Application (SPA) to Netlify's global Content Delivery Network (CDN). The deployment process is automated via Git integration, with support for multiple deployment targets including GitHub Pages. This document covers the complete deployment architecture, build process, and infrastructure configuration.

---

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    DEPLOYMENT PIPELINE                       │
└─────────────────────────────────────────────────────────────┘

Developer
    ↓
  git push to GitHub
    ↓
┌───────────────────────────────────────────┐
│         GitHub Repository                 │
│    github.com/mattlol85/                  │
│    GameratorFrontEnd                      │
└───────────────────────────────────────────┘
    ↓ (webhook)
┌───────────────────────────────────────────┐
│         Netlify Build System              │
│  1. Clone repository                      │
│  2. Install dependencies (npm install)    │
│  3. Run build command (npm run build)     │
│  4. Optimize assets                       │
│  5. Deploy to CDN                         │
└───────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────┐
│         Netlify Global CDN                │
│  - Edge servers worldwide                 │
│  - HTTPS enabled                          │
│  - Automatic redirects for SPA            │
│  - Serverless functions (if needed)       │
└───────────────────────────────────────────┘
    ↓
End Users (Browsers)
```

---

## Deployment Targets

### Primary: Netlify

**URL**: Custom domain or `*.netlify.app`

**Configuration**: Netlify.toml
```toml
[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

**Build Settings**:
- **Build Command**: `npm run build`
- **Publish Directory**: `build/`
- **Node Version**: Latest LTS (inferred from environment)

**Environment Variables** (Set in Netlify Dashboard):
```
REACT_APP_HOSTNAME=gamerator-server.herokuapp.com
REACT_APP_GOOGLEID=<client-id>.apps.googleusercontent.com
```

**Deployment Triggers**:
- Auto-deploy on push to main branch
- Deploy previews for pull requests
- Manual deploy via Netlify CLI or dashboard

**Features**:
- ✅ Automatic HTTPS (Let's Encrypt)
- ✅ Global CDN (edge caching)
- ✅ Instant cache invalidation
- ✅ Atomic deploys (zero downtime)
- ✅ Deploy previews for branches
- ✅ Rollback to previous deploys
- ✅ Custom domain support
- ✅ Form handling (not used)
- ✅ Serverless functions (not used)

### Secondary: GitHub Pages

**URL**: `https://mattlol85.github.io`

**Configuration**: package.json
```json
{
  "homepage": "https://mattlol85.github.io",
  "scripts": {
    "predeploy": "npm run build",
    "deploy": "gh-pages -d build"
  }
}
```

**Deployment Process**:
```bash
npm run deploy
```

**What Happens**:
1. `predeploy` runs `npm run build` (creates build/)
2. `gh-pages` pushes build/ contents to gh-pages branch
3. GitHub Pages serves from gh-pages branch

**Limitations**:
- No automatic deploys (manual npm run deploy)
- No environment variables (must be hardcoded at build time)
- No deploy previews
- No custom redirects (SPA routing requires workaround)
- HTTPS on custom domain requires configuration

**SPA Routing Workaround** (Not Implemented):
Create `public/404.html` that redirects to `index.html`:
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <script>
      sessionStorage.redirect = location.href;
    </script>
    <meta http-equiv="refresh" content="0;URL='/'">
  </head>
</html>
```

---

## Build Process

### Development Build

**Command**: `npm start`

**What Happens**:
1. `react-scripts start` executes
2. Webpack dev server starts on http://localhost:3000
3. Hot Module Replacement (HMR) enabled
4. Source maps generated
5. No optimization (fast rebuilds)

**Output**: In-memory bundle (not written to disk)

**Features**:
- Live reloading on file changes
- Detailed error messages
- React DevTools integration
- Fast incremental builds

**Environment**:
- `NODE_ENV=development`
- Development-only warnings enabled
- Verbose logging

### Production Build

**Command**: `npm run build`

**What Happens**:
```
1. Clean build/ directory
2. Set NODE_ENV=production
3. Transpile JavaScript (Babel)
   - ES6+ → ES5 (for older browsers)
   - JSX → JavaScript
   - Remove PropTypes (if used)
4. Bundle modules (Webpack)
   - Resolve imports
   - Tree shaking (remove unused code)
   - Code splitting (if configured)
5. Optimize CSS
   - Minification
   - Autoprefixer (vendor prefixes)
   - Critical CSS extraction
6. Optimize images
   - Compression
   - Format conversion (if configured)
7. Generate index.html
   - Inject script/link tags
   - Hash filenames for cache busting
8. Create service worker (if enabled)
9. Generate build manifest
10. Write to build/ directory
```

**Output Directory Structure**:
```
build/
├── index.html                 (Entry point)
├── favicon.ico
├── manifest.json
├── robots.txt
├── logo192.png
├── logo512.png
├── static/
│   ├── css/
│   │   └── main.[hash].css   (Bundled CSS)
│   ├── js/
│   │   ├── main.[hash].js    (App bundle)
│   │   ├── 2.[hash].chunk.js (Vendor bundle - React, etc.)
│   │   └── runtime-main.[hash].js (Webpack runtime)
│   └── media/
│       └── [images].[hash].png
└── asset-manifest.json        (Build metadata)
```

**File Hashing**:
- Format: `main.a1b2c3d4.js`
- Purpose: Cache busting (new deploy = new hash)
- Enables long-term caching (1 year)

**Bundle Sizes** (Approximate):
- main.js: ~150KB (gzipped: ~50KB) - App code
- 2.chunk.js: ~250KB (gzipped: ~80KB) - React + dependencies
- runtime.js: ~5KB (gzipped: ~2KB) - Webpack runtime
- Total: ~400KB (gzipped: ~130KB)

**Optimization Features**:
- Minification (Terser)
- Dead code elimination
- Tree shaking
- Constant folding
- Gzip/Brotli compression (by CDN)

---

## Environment Management

### Environment Variable Configuration

**Naming Convention**:
- Must start with `REACT_APP_`
- Example: `REACT_APP_HOSTNAME`, not `HOSTNAME`

**Storage**: `.env` file (not committed to Git)
```bash
# .env (local development)
REACT_APP_HOSTNAME=localhost
REACT_APP_GOOGLEID=123456789-abcdefg.apps.googleusercontent.com
```

**Loading**:
- Loaded by `react-scripts` at build time
- Injected into code via `process.env.REACT_APP_*`
- Not available at runtime (baked into bundle)

**Usage in Code**:
```javascript
const hostname = process.env.REACT_APP_HOSTNAME
// Compiled to: const hostname = "localhost"
```

**Security Note**:
- All `REACT_APP_*` variables are public (visible in browser)
- Never store secrets/API keys in these variables
- Backend API keys should stay on backend only

### Multi-Environment Setup

**Not Currently Implemented**:

**Development (.env.development)**:
```bash
REACT_APP_HOSTNAME=localhost
REACT_APP_GOOGLEID=dev-client-id.apps.googleusercontent.com
```

**Production (.env.production)**:
```bash
REACT_APP_HOSTNAME=gamerator-server.herokuapp.com
REACT_APP_GOOGLEID=prod-client-id.apps.googleusercontent.com
```

**Staging (.env.staging)**: Custom setup
```bash
REACT_APP_HOSTNAME=gamerator-staging.herokuapp.com
REACT_APP_GOOGLEID=staging-client-id.apps.googleusercontent.com
```

**Loading Priority**:
1. `.env.production.local` (git-ignored)
2. `.env.production`
3. `.env.local` (git-ignored)
4. `.env`

---

## CDN and Caching Strategy

### Netlify CDN Architecture

**Edge Network**:
- 100+ global edge servers
- Automatic geographic routing
- Smart CDN (application-aware)

**Caching Behavior**:

**Static Assets** (CSS, JS, images):
```
Cache-Control: public, max-age=31536000, immutable
```
- Cached for 1 year
- Filename hash ensures cache invalidation on changes
- Immutable flag prevents revalidation

**index.html**:
```
Cache-Control: public, max-age=0, must-revalidate
```
- Not cached (always fetched from origin)
- Ensures users get latest version
- Fast origin response (CDN edge)

**Custom Headers** (Netlify Headers - Not Configured):
```
# netlify.toml
[[headers]]
  for = "/*.js"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"

[[headers]]
  for = "/*.css"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"
```

### Cache Invalidation

**Automatic**:
- New deploy → Cache automatically invalidated
- Atomic deploys ensure no mixed versions

**Manual** (if needed):
```bash
# Netlify CLI
netlify deploy --prod --trigger-deploy
```

**Cache Purging**:
- Not needed (hash-based cache busting)
- index.html never cached

---

## HTTPS and Security

### SSL/TLS Configuration

**Netlify**:
- Automatic HTTPS via Let's Encrypt
- Auto-renewal (no manual management)
- TLS 1.2+ only
- Modern cipher suites

**Certificate**:
- Wildcard certificate for *.netlify.app
- Custom domain: Automatic Let's Encrypt cert

**HSTS** (HTTP Strict Transport Security):
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```
- Forces HTTPS for 1 year
- Includes subdomains
- Preload-ready (can submit to browser HSTS preload list)

### Security Headers

**Current** (Netlify Defaults):
```
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
```

**Recommended Addition** (netlify.toml):
```toml
[[headers]]
  for = "/*"
  [headers.values]
    Content-Security-Policy = "default-src 'self'; script-src 'self' https://accounts.google.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self' http://localhost:8080 https://*.herokuapp.com;"
    Permissions-Policy = "geolocation=(), microphone=(), camera=()"
```

---

## Deployment Workflows

### Automatic Deployment (Netlify)

**Trigger**: Push to main branch

**Process**:
1. GitHub webhook notifies Netlify
2. Netlify clones repository
3. Runs `npm install`
4. Runs `npm run build`
5. Deploys build/ to CDN
6. Invalidates cache
7. Sends deployment notification (email/Slack)

**Deploy Preview** (Pull Requests):
1. Create PR on GitHub
2. Netlify builds PR branch
3. Deploys to unique URL: `deploy-preview-123--app.netlify.app`
4. Comment on PR with preview link
5. Update preview on every commit to PR
6. Delete preview after PR merge/close

**Branch Deploys**:
- Configure specific branches for deploy
- Each branch gets unique URL: `branch-name--app.netlify.app`
- Useful for staging environments

### Manual Deployment (GitHub Pages)

**Process**:
```bash
# 1. Build the app
npm run build

# 2. Deploy to GitHub Pages (automated by gh-pages)
npm run deploy
```

**What `gh-pages` does**:
```bash
1. Create/checkout gh-pages branch
2. Remove old files
3. Copy build/ contents to branch root
4. Commit changes
5. Force push to origin/gh-pages
6. GitHub Pages detects change and deploys
```

### Manual Deployment (Netlify CLI)

**Installation**:
```bash
npm install -g netlify-cli
netlify login
```

**Link Project**:
```bash
netlify link
```

**Deploy**:
```bash
# Deploy to draft URL
netlify deploy

# Deploy to production
netlify deploy --prod
```

---

## Monitoring and Logging

### Current State: No Monitoring

**Missing**:
- No error tracking (Sentry, Bugsnag)
- No performance monitoring (Google Analytics, Lighthouse CI)
- No uptime monitoring (Pingdom, UptimeRobot)
- No build failure notifications

### Recommended Monitoring

**1. Error Tracking (Sentry)**:
```javascript
// index.js
import * as Sentry from "@sentry/react"

Sentry.init({
  dsn: process.env.REACT_APP_SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 1.0,
})

ReactDOM.render(
  <Sentry.ErrorBoundary fallback={<ErrorFallback />}>
    <App />
  </Sentry.ErrorBoundary>,
  document.getElementById('root')
)
```

**2. Analytics (Google Analytics)**:
```javascript
// Analytics.js
import { useEffect } from 'react'
import { useLocation } from 'react-router-dom'

export function Analytics() {
  const location = useLocation()

  useEffect(() => {
    if (window.gtag) {
      window.gtag('config', 'GA_MEASUREMENT_ID', {
        page_path: location.pathname,
      })
    }
  }, [location])

  return null
}
```

**3. Performance Monitoring (Web Vitals)**:
```javascript
// reportWebVitals.js (already included in CRA)
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals'

function sendToAnalytics({ name, delta, value, id }) {
  // Send to analytics endpoint
  if (window.gtag) {
    window.gtag('event', name, {
      event_category: 'Web Vitals',
      value: Math.round(name === 'CLS' ? delta * 1000 : delta),
      event_label: id,
      non_interaction: true,
    })
  }
}

getCLS(sendToAnalytics)
getFID(sendToAnalytics)
getFCP(sendToAnalytics)
getLCP(sendToAnalytics)
getTTFB(sendToAnalytics)
```

**4. Uptime Monitoring**:
- Use UptimeRobot or Pingdom
- Ping homepage every 5 minutes
- Alert on downtime (email/SMS)

---

## Continuous Integration/Deployment (CI/CD)

### Current: Netlify Auto-Deploy

**Simplest CI/CD**:
- No separate CI service needed
- Build + Deploy in one step
- Works for current needs

### Recommended: GitHub Actions

**Workflow** (.github/workflows/deploy.yml):
```yaml
name: Deploy to Netlify

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - name: Setup Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '16'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test -- --coverage

      - name: Build
        run: npm run build
        env:
          REACT_APP_HOSTNAME: ${{ secrets.REACT_APP_HOSTNAME }}
          REACT_APP_GOOGLEID: ${{ secrets.REACT_APP_GOOGLEID }}

      - name: Deploy to Netlify
        uses: nwtgck/actions-netlify@v1.2
        with:
          publish-dir: './build'
          production-branch: main
          github-token: ${{ secrets.GITHUB_TOKEN }}
          deploy-message: "Deploy from GitHub Actions"
        env:
          NETLIFY_AUTH_TOKEN: ${{ secrets.NETLIFY_AUTH_TOKEN }}
          NETLIFY_SITE_ID: ${{ secrets.NETLIFY_SITE_ID }}
```

**Benefits**:
- Run tests before deploy
- Linting checks
- Bundle size analysis
- Parallel jobs (test + build)
- Custom notifications

---

## Rollback Strategy

### Netlify Rollbacks

**Via Dashboard**:
1. Navigate to Deploys section
2. Find previous successful deploy
3. Click "Publish deploy"
4. Previous version goes live instantly

**Via CLI**:
```bash
# List deploys
netlify deploy:list

# Rollback to specific deploy
netlify rollback --deploy-id <deploy-id>
```

**Atomic Deploys**:
- New deploy doesn't replace old until fully uploaded
- Rollback is instant (just pointer change)
- No downtime during rollback

### GitHub Pages Rollbacks

**Manual Process**:
```bash
# Revert to previous commit
git revert <commit-hash>

# Rebuild and redeploy
npm run deploy
```

**Slower**:
- Requires full rebuild
- GitHub Pages propagation delay (minutes)
- Not atomic

---

## Performance Optimization

### Build Optimizations

**Current** (via Create React App):
- ✅ Minification (Terser)
- ✅ Tree shaking
- ✅ Dead code elimination
- ✅ CSS optimization
- ✅ Image compression (basic)

**Not Configured**:
- ❌ Code splitting (lazy loading routes)
- ❌ Service worker (offline support)
- ❌ Preload/prefetch hints
- ❌ Critical CSS extraction
- ❌ WebP image conversion

**Recommended: Code Splitting**:
```javascript
// App.js
import { lazy, Suspense } from 'react'

const Home = lazy(() => import('./Home'))
const Action = lazy(() => import('./Action'))
const Leaderboard = lazy(() => import('./Leaderboard'))

<Suspense fallback={<LoadingSpinner />}>
  <Routes>
    <Route path="/home" element={<Home />} />
    <Route path="/action" element={<Action />} />
    <Route path="/leaderboard" element={<Leaderboard />} />
  </Routes>
</Suspense>
```

**Result**:
- Initial bundle: ~100KB (vs ~400KB)
- Route bundles load on-demand
- Faster time to interactive

### CDN Optimizations

**Current Netlify Features**:
- ✅ Brotli compression (better than gzip)
- ✅ HTTP/2 (multiplexing)
- ✅ Edge caching
- ✅ Smart CDN routing

**Potential Additions**:
- Image optimization via Netlify Image CDN
- Asset optimization plugin
- Build plugins for compression

---

## Cost Analysis

### Netlify Pricing

**Free Tier** (Current):
- 100GB bandwidth/month
- Unlimited sites
- HTTPS included
- Build minutes: 300/month
- Concurrent builds: 1

**Sufficient For**:
- Small to medium traffic
- Educational projects
- Portfolio sites

**Upgrade Triggers**:
- Traffic > 100GB/month (~1M page views)
- Need for more build minutes
- Team collaboration features
- Advanced analytics

**Pro Tier** ($19/month):
- 400GB bandwidth
- 1000 build minutes
- Background functions
- Analytics

### GitHub Pages Pricing

**Free**:
- Unlimited bandwidth (soft limit)
- Public repositories only
- No build minutes (use GitHub Actions if needed)

**Limitation**:
- 1GB repository size
- 100GB bandwidth/month (soft limit)
- 10 builds/hour

---

## Disaster Recovery

### Backup Strategy

**Git Repository**:
- Source of truth is GitHub
- Full history preserved
- Can rebuild from any commit

**Netlify**:
- Stores all previous deploys
- Can rollback anytime
- No manual backups needed

**Environment Variables**:
- Not backed up automatically
- Store in password manager or .env.production.backup (git-ignored)

### Recovery Scenarios

**Scenario 1: Netlify Account Compromised**
1. Lock Netlify account
2. Deploy to GitHub Pages as temporary measure:
   ```bash
   npm run deploy
   ```
3. Update DNS to point to GitHub Pages
4. Recover Netlify account or create new one

**Scenario 2: Bad Deploy**
1. Identify issue
2. Rollback via Netlify dashboard
3. Fix issue locally
4. Deploy fix

**Scenario 3: Lost Access to Repository**
1. Clone from any contributor's local copy
2. Create new repository
3. Reconfigure Netlify to point to new repo

---

## Infrastructure as Code

### Current: Manual Configuration

**Netlify**:
- Settings configured via dashboard
- Not version controlled

**Recommended: Netlify.toml**:
```toml
[build]
  command = "npm run build"
  publish = "build"

[build.environment]
  NODE_VERSION = "16"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200

[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-Content-Type-Options = "nosniff"
    Referrer-Policy = "strict-origin-when-cross-origin"

[[headers]]
  for = "/static/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"
```

**Benefits**:
- Version controlled
- Reproducible deployments
- Easy to review changes
- Infrastructure as code

---

## Testing Deployments

### Pre-Deploy Checklist

**Local Testing**:
- [ ] `npm test` passes
- [ ] `npm run build` succeeds
- [ ] Serve build locally: `npx serve -s build`
- [ ] Test on http://localhost:5000
- [ ] Check all routes work
- [ ] Check mobile responsiveness
- [ ] Test in multiple browsers

**Environment Variables**:
- [ ] All required env vars set in Netlify dashboard
- [ ] No secrets in code
- [ ] Production API endpoints configured

**Build Validation**:
- [ ] No console errors in production build
- [ ] Bundle size acceptable (< 500KB gzipped)
- [ ] Lighthouse score > 90

### Post-Deploy Validation

**Smoke Tests**:
- [ ] Homepage loads
- [ ] Login works
- [ ] Navigation between pages
- [ ] Game cards display
- [ ] Voting works
- [ ] Leaderboard loads

**Performance Checks**:
- [ ] Time to Interactive < 3s
- [ ] Largest Contentful Paint < 2.5s
- [ ] Cumulative Layout Shift < 0.1

**Cross-Browser**:
- [ ] Chrome (latest)
- [ ] Firefox (latest)
- [ ] Safari (latest)
- [ ] Edge (latest)
- [ ] Mobile browsers

---

## Troubleshooting Common Issues

### Issue 1: Build Fails on Netlify

**Symptoms**:
- Local build works
- Netlify build fails

**Common Causes**:
1. **Missing Environment Variables**:
   - Check Netlify dashboard → Site settings → Build & deploy → Environment
   - Add all `REACT_APP_*` variables

2. **Node Version Mismatch**:
   ```toml
   # netlify.toml
   [build.environment]
     NODE_VERSION = "16"
   ```

3. **Dependency Issues**:
   - Delete `package-lock.json` and `node_modules/`
   - Run `npm install` to regenerate
   - Commit new `package-lock.json`

### Issue 2: SPA Routes Return 404

**Symptoms**:
- Navigating via app works
- Direct URL access or refresh returns 404

**Solution**: Ensure redirect rule exists

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

### Issue 3: Environment Variables Not Working

**Symptoms**:
- `process.env.REACT_APP_HOSTNAME` is undefined

**Causes**:
1. **Variable name doesn't start with `REACT_APP_`**
   - Must use `REACT_APP_` prefix
   - Rename variable

2. **Not set in Netlify dashboard**
   - Add in Site settings → Build & deploy → Environment variables

3. **Build not triggered after adding variables**
   - Trigger rebuild: Deploys → Trigger deploy → Deploy site

### Issue 4: Google OAuth Fails After Deploy

**Symptoms**:
- OAuth works locally
- Fails in production

**Causes**:
1. **Authorized JavaScript origins not set**:
   - Google Cloud Console → Credentials
   - Add production URL (e.g., https://gamerator.netlify.app)

2. **Wrong REACT_APP_GOOGLEID**:
   - Check Netlify environment variable
   - Should be production Client ID, not dev

---

## Future Infrastructure Enhancements

### Phase 1: Improve Current Setup
1. Add Netlify.toml for infrastructure as code
2. Configure security headers
3. Set up deploy notifications (Slack, email)
4. Enable automatic dependency updates (Dependabot)

### Phase 2: CI/CD Pipeline
1. Implement GitHub Actions workflow
2. Add automated testing before deploy
3. Bundle size monitoring
4. Lighthouse CI integration

### Phase 3: Advanced Features
1. A/B testing with Netlify Split Testing
2. Serverless functions for contact forms
3. CDN optimization (image CDN, lazy loading)
4. Progressive Web App (PWA) with service worker

### Phase 4: Enterprise Features
1. Multi-region deployments
2. DDoS protection (Cloudflare)
3. WAF (Web Application Firewall)
4. Custom CDN configuration

---

## Conclusion

Gamerator's deployment infrastructure leverages Netlify's powerful platform to provide a production-ready, globally distributed SPA with minimal configuration. The automated Git-based workflow ensures rapid iteration while maintaining stability through atomic deploys and instant rollbacks.

**Current Strengths**:
- Automated deployments via Git
- Global CDN with HTTPS
- Deploy previews for pull requests
- Zero-downtime deployments
- Simple, maintainable setup

**Areas for Improvement**:
- Infrastructure as code (Netlify.toml)
- CI/CD pipeline with testing
- Monitoring and error tracking
- Performance optimizations (code splitting)
- Security headers

For the current project scale, the deployment setup is excellent. As the application grows, adopting the recommended enhancements will ensure scalability, reliability, and observability.
