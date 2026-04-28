# JobFinder — Project Context & Memory

## Architecture

### Core Stack
- **Frontend**: Pure vanilla HTML + JavaScript (no build step)
- **Styling**: Tailwind CSS via CDN
- **Data Source**: Wikipedia S&P 500 constituent list (live fetch with 12-hour localStorage cache)
- **Hosting**: GitHub Pages (static)

### Design Philosophy
- **No Backend**: All data fetching, filtering, and caching happens client-side
- **Offline First**: Falls back gracefully to curated company database if Wikipedia is unreachable
- **Zero Auth**: No user accounts, no API keys, no authentication needed
- **Performance**: Lightweight, fast load, minimal dependencies

### Key Architectural Decisions

1. **Live S&P 500 Data Over Static DB**
   - Initial implementation used hardcoded 90-company database
   - Moved to Wikipedia scraping for 500+ real companies with GICS sectors
   - Reduces maintenance burden, always current
   - Fallback to curated DB ensures app works offline

2. **GICS Sector Mapping**
   - User discipline inputs (e.g., "data", "energy", "finance") map to GICS sector classifiers
   - Allows keyword-based search across all 500 companies without manual curation
   - Sub-industry boost keywords for relevance (e.g., "semiconductor" boosts AI/ML results)

3. **Dual Card Renderer**
   - `buildSP500Card()`: Renders Wikipedia companies (ticker, sector, sub-industry, state)
   - `buildCuratedCard()`: Renders fallback database (tags, notes, remote policy, size)
   - Both feed same selection flow: `selectCompany()` → pre-fills company field → triggers search

4. **LocalStorage Caching Strategy**
   - `jf_sp500_v3`: Cached S&P 500 list (12-hour TTL)
   - `jf_recent`: Saved searches with role, location, company (max 8 items)
   - `jf_dark`: Dark mode preference
   - Versioning (`v3`) prevents cache corruption on schema changes

---

## Current Status

### ✅ Completed Features

1. **Job Search Form**
   - Inputs: Role, Location, Company (optional)
   - Generates search links for 8 job boards: LinkedIn, Indeed, Glassdoor, ZipRecruiter, Monster, SimplyHired, CareerBuilder, Google Jobs
   - Each link pre-fills role + location + company into board's search syntax
   - "Search Jobs" button + Enter key support

2. **Company Explorer**
   - Type discipline → maps to GICS sectors → shows matching S&P 500 companies
   - Quick-filter chips: Data, Engineering, Finance, Energy, Healthcare, Consulting, Marketing, AI/ML, Product
   - Live fetch from Wikipedia with "Loading..." → "503 companies · live" status
   - Graceful fallback to 90-company curated database if offline
   - Click company → pre-fill and search OR highlight role field if empty

3. **Results Display**
   - 8 job board cards in grid layout
   - Each card shows: platform name, ticker color-coded, search URL preview, copy link button
   - Action buttons: primary "Search [Platform]", WhatsApp share (pre-filled message), Email share (mailto)
   - "Open All" button: launches all 8 boards in staggered tabs (200ms delay)

4. **UX Features**
   - Recent searches (localStorage): click to reload previous search
   - Dark mode: system preference + manual toggle
   - Responsive design: mobile-first, Tailwind grid
   - Toast notifications: "Link copied!", "Opening all platforms…"
   - Smooth scrolling to results section
   - Error states: missing role input highlights field in red, no results shows emoji message

5. **Search Link Generation**
   - Dynamic URL builders for each platform
   - Format: `board.url(role, location, company)`
   - Handles company filtering per platform (e.g., LinkedIn: "role company", Indeed: "role company")
   - URL encoding via `encodeURIComponent`

---

## Important Patterns

### 1. **State Management**
```javascript
let SP500 = [];              // Global company list
let sp500State = 'idle';     // 'idle' | 'loading' | 'loaded' | 'failed'
let explorerDebounce = null; // Debounce timer for live search
let lastDiscipline = '';     // Track current discipline query
```
- Use global state sparingly; prefer DOM queries for UI state
- Always clear debounce timeouts before setting new ones

### 2. **Async Data Loading**
```javascript
async function initSP500() {
  const CACHE = 'jf_sp500_v3';
  const TTL = 12 * 3600 * 1000;
  
  try {
    // Check cache first
    const cached = JSON.parse(localStorage.getItem(CACHE) || 'null');
    if (cached && Date.now() - cached.ts < TTL && cached.data.length > 400) {
      SP500 = cached.data;
      return;
    }
    // Fetch if stale/missing
    const resp = await fetch(WIKIPEDIA_URL);
    // Parse, validate, cache, set state
  } catch (e) {
    // Gracefully fall back
    sp500State = 'failed';
  }
}
```
- Always wrap async in try/catch
- Validate fetched data length before accepting
- Set `sp500State` to reflect fetch outcome
- Call at init time, not on user interaction

### 3. **Filtering with Multiple Data Sources**
```javascript
function exploreCompanies() {
  const companies = sp500State === 'loaded' && SP500.length 
    ? filterSP500(q)      // Live data
    : filterFallback(q);  // Curated fallback
  renderExplorerCards(companies, q);
}
```
- Check data availability before deciding filter strategy
- Both filter functions return same shape: `{ name, ticker?, sector, ... }`
- Render function agnostic to data source

### 4. **Keyboard + Click Handling**
```html
<input onkeydown="if(event.key==='Enter') searchJobs()" />
<button onclick="searchJobs()">Search Jobs</button>
```
- All form inputs support Enter key for submission
- Prevent form tag hijacking—use onClick for buttons instead
- Debounce live input (300ms for "type discipline")

### 5. **URL Generation Per Platform**
```javascript
const JOB_BOARDS = [
  {
    name: 'LinkedIn',
    url: (role, loc, co) => {
      const kw = co ? `${role} ${co}` : role;
      return `https://www.linkedin.com/jobs/search/?keywords=${enc(kw)}&location=${enc(loc)}`;
    },
  },
  // ... others
];

// Usage: board.url(role, location, company)
```
- Each platform has custom URL format
- Accept `company` param even if not used (consistent signature)
- Always encode with `encodeURIComponent` (aliased as `enc`)
- Test URLs in browser before adding

### 6. **HTML Escaping**
```javascript
function escHtml(s) {
  return String(s).replace(/[&<>"']/g, c => ({
    '&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'
  }[c]));
}
```
- Use on all user input displayed in templates
- Prevents XSS from company names, roles, locations
- Wrap in `String()` to handle null/undefined

### 7. **Toast Notification Pattern**
```javascript
function showToast(msg = 'Link copied!') {
  const t = document.getElementById('toast');
  document.getElementById('toast-msg').textContent = msg;
  t.classList.add('show');
  setTimeout(() => t.classList.remove('show'), 2500);
}
```
- Single global toast element, reuse it
- Auto-hide after 2.5s
- CSS handles animation via `.show` class

### 8. **CSS for Interactivity**
```css
.card-hover { transition: transform 0.2s, box-shadow 0.2s; }
.card-hover:hover { transform: translateY(-2px); }
.btn-press { transition: transform 0.1s; }
.btn-press:active { transform: scale(0.97); }
```
- Hover lifts cards (2px), press scales button (97%)
- All transitions under 0.2s for snappy feel
- Use transform, not top/position (better performance)

---

## Known Issues & Solutions

### ✅ Solved Issues

#### 1. **Duplicate `const COMPANY_DB` Declaration**
**Problem**: Two `const COMPANY_DB = [...]` lines in same scope caused `SyntaxError: Identifier already declared`. Entire script silent-failed—no error shown in console, app just dead.

**Solution**: Removed duplicate line. When editing, always grep for variable names before adding.

**Prevention**: Use unique variable names, test locally before pushing.

---

#### 2. **Old Function Stubs Overriding New Code**
**Problem**: After implementing live S&P 500 fetching with new `exploreByChip()`, `filterSP500()`, `buildSP500Card()` functions, the old `quickExplore()`, `exploreCompanies()`, `buildCompanyCard()` stubs from v1 were still in the code. They shadowed the new functions, so clicking chips did nothing.

**Solution**: Removed all old function implementations. Kept only the new S&P 500-powered versions.

**Prevention**: When refactoring, search for all old function names and delete them. Use git diff to spot function duplicates.

---

#### 3. **CORS Blocking Wikipedia Fetch**
**Problem**: Initial Wikipedia fetch failed silently due to browser CORS restrictions on direct fetch.

**Solution**: Used `https://en.wikipedia.org/wiki/List_of_S%26P_500_companies` with standard fetch headers. Browser allows it. If it fails, gracefully fall back to curated DB and set `sp500State = 'failed'`.

**Prevention**: Always test fetch URLs in browser console first. Assume some sources may fail and have a fallback.

---

#### 4. **Company Card "Find Jobs" Button Not Responding**
**Problem**: After adding S&P 500 cards, clicking "Find Jobs at [Company]" didn't select the company.

**Solution**: Ensured `selectCompany()` function exists and is accessible globally (not nested). All onclick handlers call global functions only.

**Prevention**: Keep all event handlers and their target functions at window scope. Test onclick before deploy.

---

#### 5. **localStorage Corruption from Schema Changes**
**Problem**: When changing company data structure (added `ticker` field), old cached entries failed to render.

**Solution**: Versioned cache key: `jf_sp500_v1` → `v2` → `v3`. Each version is isolated, old cache ignored.

**Prevention**: Always version localStorage keys. Increment version when data structure changes.

---

#### 6. **Recent Searches Not Showing Company Filter**
**Problem**: Clicking a recent search restored role + location but not company.

**Solution**: Updated `saveRecent()` to include company param:
```javascript
items.unshift({ role, location, company: company || '', ts: Date.now() });
```
Updated `loadRecent()` to fill all three fields:
```javascript
document.getElementById('job-company').value = company || '';
```

**Prevention**: When adding a new search filter, update both save and load functions.

---

### ⚠️ Known Limitations

1. **Wikipedia Fetch Depends on User's Network**
   - If user behind strict corporate firewall, Wikipedia fetch fails
   - App silently falls back to curated DB (no error message)
   - Status badge shows `⚠ Offline — using curated list`
   - **Acceptable**: Curated list is complete enough for MVP

2. **Job Board URLs May Change**
   - LinkedIn, Indeed, etc. may update their search URL structure
   - If they do, search links will break
   - **Mitigation**: Test URLs monthly; update board configs as needed

3. **No Search History Sync**
   - Recent searches stored only in localStorage (per device/browser)
   - Clearing browser data loses history
   - **By design**: No backend to sync

4. **Company Filtering Not Exact**
   - Company field is optional text input, not a dropdown from S&P 500 list
   - User can type "Gogle" and it's included verbatim in search URL
   - Job boards handle typos gracefully (usually return 0 results)
   - **Acceptable**: UX cost of autocomplete dropdown not worth it

---

## Requirements & Constraints

### Must-Haves ✅
- [x] Frontend-only, no backend
- [x] Generate search links for 8+ job boards
- [x] Support role, location, optional company
- [x] Dark mode
- [x] Recent searches via localStorage
- [x] Responsive mobile/tablet/desktop
- [x] No auth, no API keys, no sign-up

### Nice-to-Haves ✅
- [x] Company explorer by discipline
- [x] Live S&P 500 data fetch
- [x] WhatsApp + Email share
- [x] Copy link button
- [x] Open all platforms
- [x] Smooth animations
- [x] Graceful offline fallback

### Browser Compatibility
- Modern browsers (ES6+, CSS Grid, Fetch API, localStorage, async/await)
- Works: Chrome 90+, Firefox 88+, Safari 14+, Edge 90+
- Does NOT work: IE 11 (not a goal)

---

## File Structure
```
job-finder/
├── index.html              # Single-file app (all HTML, CSS, JS)
├── README.md               # Public docs
├── .git/                   # GitHub repo
├── .cursor/context.md      # This file
└── (no src/ or dist/ — single file delivery)
```

---

## Next Steps / Future Enhancements

1. **Company Autocomplete**
   - Dropdown of S&P 500 company names as user types in company field
   - Prevent typos, improve UX

2. **Salary Range Filter**
   - Add min/max salary inputs
   - Filter boards' results (if each board's URL supports it)

3. **Job Type Filter**
   - Full-time, part-time, contract checkboxes
   - Append to each board's search URL

4. **Advanced Analytics**
   - Track which boards get clicked most
   - Popular role/location combinations (anonymous, no tracking, just logs)

5. **Mobile App Version**
   - Wrap web app in Capacitor/React Native
   - Distribute on iOS/Android app stores

6. **Browser Extension**
   - Detect job posting, "Find Similar on JobFinder" button
   - Link back to app with pre-filled role

---

## How to Maintain This Project

### Adding a New Job Board
1. Add entry to `JOB_BOARDS` array with name, icon, color, badge color, and `url()` function
2. Test URL in browser with sample role/location/company
3. Verify encoding handles special characters

### Updating S&P 500 Companies
1. If Wikipedia structure changes, update `parseSP500Html()` to match new table selectors
2. Test parse locally: fetch Wikipedia, verify `SP500.length > 400`
3. Increment cache version: `jf_sp500_v3` → `jf_sp500_v4`

### Fixing Bugs
1. Check browser console for JS errors
2. Use browser DevTools to inspect localStorage, network tab
3. Test in dark mode and light mode
4. Test on mobile viewport (375px)
5. Commit with clear message: "Fix: brief description of issue and solution"

---

## Deployment

Currently hosted on **GitHub Pages** at: https://olamidedurodola.github.io/job-finder/

### To Deploy
```bash
cd job-finder
git add .
git commit -m "Update: description"
git push origin main
```

GitHub Actions auto-deploys to Pages. Site live in ~1 minute.

---

## Contact & Attribution

- **Author**: Olamide Durodola (with Claude assistance)
- **Built with**: Tailwind CSS, vanilla JS, Wikipedia S&P 500 data
- **License**: MIT (or as specified in repo)
- **GitHub**: https://github.com/Olamidedurodola/job-finder
