# Implementation Plan: PSX Portfolio Platform

## Overview

This plan implements a multi-service PSX portfolio management platform with a React/TypeScript frontend, Node.js/TypeScript REST API backend, and Python services for price fetching, news scraping, and AI-driven suggestions. Implementation proceeds bottom-up: data models and shared types first, then backend services, then Python services, then frontend, and finally integration wiring.

## Tasks

- [ ] 1. Set up project structure and shared types
  - [ ] 1.1 Initialize monorepo structure with frontend, backend, and python-services directories
    - Create directory structure: `frontend/` (React/TypeScript), `backend/` (Node.js/TypeScript), `python-services/` (Python)
    - Create root `.npmrc` with `save-exact=true` and `package-lock=true` to enforce exact pinned versions and prevent auto-updates
    - Initialize `backend/package.json` with exact pinned dependencies (NO ^ or ~ ranges):
      - `"express": "5.1.0"`
      - `"mongoose": "8.9.5"`
      - `"axios": "1.14.0"` ⚠️ DO NOT use 1.14.1 (compromised supply chain attack)
      - `"typescript": "5.8.3"` (devDependency)
      - `"@types/express": "5.0.0"` (devDependency)
      - `"@types/node": "22.10.0"` (devDependency)
    - Initialize `frontend/package.json` with exact pinned dependencies (NO ^ or ~ ranges):
      - `"react": "19.1.0"`
      - `"react-dom": "19.1.0"`
      - `"react-router-dom": "7.6.2"`
      - `"axios": "1.14.0"` ⚠️ DO NOT use 1.14.1 (compromised)
      - `"typescript": "5.8.3"` (devDependency)
      - `"vite": "6.0.0"` (devDependency)
      - `"@vitejs/plugin-react": "4.3.4"` (devDependency)
    - Initialize `python-services/requirements.txt` with exact pinned versions (use `==` not `>=`):
      - `Flask==3.1.0`
      - `pymongo==4.10.1`
      - `beautifulsoup4==4.12.3`
      - `requests==2.32.3`
      - `openai==1.58.1`
    - Create `python-services/pip.conf` or document constraint: always use `--no-deps` and `--require-hashes` for production installs
    - Set up TypeScript configs for backend and frontend
    - Commit `package-lock.json` files — installs must use `npm ci` (not `npm install`) to respect lockfile exactly
    - _Requirements: 7.1, 7.2, 7.3, 7.4_
    - _Security: Pinned versions prevent supply chain attacks (axios 1.14.1 RAT incident, March 2026). No auto-updates._

  - [ ] 1.2 Define MongoDB data models and TypeScript interfaces for backend
    - Create `backend/src/models/Holding.ts` — Mongoose schema with fields: user_id, symbol, quantity, average_buying_price, created_at, updated_at, version (for optimistic concurrency)
    - Create `backend/src/models/Trade.ts` — Mongoose schema with fields: user_id, symbol, type (buy|sell), quantity, price_per_share, total_amount, timestamp
    - Create `backend/src/models/Price.ts` — Mongoose schema with fields: symbol, price, fetched_at; index on { symbol: 1, fetched_at: -1 }
    - Create `backend/src/models/Article.ts` — Mongoose schema with fields: title, source, publication_date, full_text, summary, stock_symbols, scraped_at; unique index on { source: 1, title: 1, publication_date: 1 }; TTL index on scraped_at (180 days)
    - Create `backend/src/models/Suggestion.ts` — Mongoose schema matching the design's Suggestions collection structure
    - _Requirements: 3.6, 4.5, 4.7, 4.9, 7.4_

  - [ ] 1.3 Define Python data models for python-services
    - Create `python-services/models/price.py` with Price dataclass/model
    - Create `python-services/models/article.py` with Article dataclass/model (title max 500 chars, full_text max 50000 chars, summary max 300 chars)
    - Create `python-services/models/suggestion.py` with Suggestion, BotAnalysis, FinalSuggestion dataclasses
    - Set up pymongo connection utility in `python-services/db/connection.py`
    - _Requirements: 4.7, 5.5, 7.3, 7.4_

- [ ] 2. Implement trade recording and portfolio logic (Backend)
  - [ ] 2.1 Implement TradeService with validation and holding updates
    - Create `backend/src/services/TradeService.ts`
    - Implement trade validation: quantity must be 1–1,000,000; price_per_share must be 0.01–99,999.99; ticker must be valid PSX-listed stock
    - Implement buy trade logic: create new holding or update existing via weighted average formula
    - Implement sell trade logic: decrease quantity (reject if exceeds current), preserve average price, remove holding if quantity reaches zero
    - Use MongoDB transactions for atomicity (update holding + insert trade in single transaction)
    - Use optimistic concurrency (version field) to prevent lost updates
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8_

  - [ ]* 2.2 Write property test: Buy trade weighted average (Property 5)
    - **Property 5: Buy trade correctly updates holding via weighted average**
    - Generate arbitrary old_quantity (1–100000), old_avg_price (0.01–99999.99), trade_quantity (1–1000000), trade_price (0.01–99999.99)
    - Assert: new_quantity = old_quantity + trade_quantity; new_avg = ((old_qty × old_avg) + (trade_qty × trade_price)) / new_qty
    - Use fast-check library with minimum 100 iterations
    - **Validates: Requirements 3.1, 3.4**

  - [ ]* 2.3 Write property test: Sell trade preserves average price (Property 6)
    - **Property 6: Sell trade decreases quantity while preserving average price**
    - Generate arbitrary holding (quantity 2–100000, avg_price 0.01–99999.99) and sell_quantity (1–holding.quantity)
    - Assert: new_quantity = old_quantity - sell_quantity; average_buying_price unchanged
    - Use fast-check library with minimum 100 iterations
    - **Validates: Requirements 3.2**

  - [ ]* 2.4 Write property test: Invalid trade rejection (Property 7)
    - **Property 7: Invalid trade inputs are rejected**
    - Generate trades with invalid quantities (<1 or >1,000,000), invalid prices (<0.01 or >99,999.99), and invalid ticker symbols
    - Assert: trade is rejected, no holding is modified
    - Use fast-check library with minimum 100 iterations
    - **Validates: Requirements 3.7, 3.8**

- [ ] 3. Implement portfolio overview service (Backend)
  - [ ] 3.1 Implement PortfolioService with calculations
    - Create `backend/src/services/PortfolioService.ts`
    - Implement `getPortfolioOverview()`: fetch all holdings for user, sort alphabetically by symbol
    - Calculate total_investment as sum of (quantity × average_buying_price) for all holdings
    - For each holding with a fetched price: calculate P/L = (current_price - avg_price) × quantity; P/L% = ((current_price - avg_price) / avg_price) × 100
    - Round all monetary and percentage values to 2 decimal places
    - Handle missing price data: return "N/A" for value/P&L when no price exists; flag stale prices (>24h old)
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8_

  - [ ]* 3.2 Write property test: Total investment calculation (Property 1)
    - **Property 1: Total investment equals sum of holding costs**
    - Generate arbitrary list of holdings (each with quantity ≥ 1, avg_price ≥ 0.01)
    - Assert: total_investment === sum(quantity × average_buying_price) for all holdings
    - Use fast-check library with minimum 100 iterations
    - **Validates: Requirements 1.1**

  - [ ]* 3.3 Write property test: Holdings sorted alphabetically (Property 2)
    - **Property 2: Holdings are sorted alphabetically by ticker symbol**
    - Generate arbitrary list of holdings with random symbols
    - Assert: returned holdings are in ascending alphabetical order by symbol
    - Use fast-check library with minimum 100 iterations
    - **Validates: Requirements 1.2**

  - [ ]* 3.4 Write property test: P/L calculations (Property 3)
    - **Property 3: Profit/loss calculations are correct**
    - Generate arbitrary holding (quantity, avg_price) and current_price
    - Assert: P/L = (current_price - avg_price) × quantity; P/L% = ((current_price - avg_price) / avg_price) × 100
    - Use fast-check library with minimum 100 iterations
    - **Validates: Requirements 1.3, 1.4**

  - [ ]* 3.5 Write property test: Rounding to 2 decimal places (Property 4)
    - **Property 4: Monetary and percentage values are rounded to 2 decimal places**
    - Generate arbitrary monetary and percentage values with many decimal places
    - Assert: formatted output has exactly 2 decimal places
    - Use fast-check library with minimum 100 iterations
    - **Validates: Requirements 1.8**

- [ ] 4. Implement price fetch service (Backend + Python)
  - [ ] 4.1 Implement Python Price_Fetcher service
    - Create `python-services/price_fetcher/app.py` — Flask app with `POST /fetch-prices` endpoint
    - Accept list of ticker symbols in request body
    - Implement PSX website scraping using BeautifulSoup to extract current prices
    - Return response: `{ prices: [{ symbol, price, timestamp }], errors: [{ symbol, reason }] }`
    - Handle per-symbol failures gracefully (continue fetching remaining symbols)
    - _Requirements: 2.1, 2.2, 2.4, 7.3, 7.5_

  - [ ] 4.2 Implement PriceFetchService in Node.js backend
    - Create `backend/src/services/PriceFetchService.ts`
    - Implement `refreshPrices(userId)`: get user's holdings, call Python Price_Fetcher via HTTP, store results
    - Set 30-second timeout on HTTP call to Price_Fetcher
    - Handle timeout: abort request, return timeout error
    - Handle partial success (207 Multi-Status): store successful prices, return per-symbol error list
    - Handle Price_Fetcher unreachable: return 503 error
    - Store fetched prices as append-only records (never update existing price records)
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 2.7, 7.5, 7.6_

- [ ] 5. Checkpoint - Backend core services
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. Implement News Scraper (Python)
  - [ ] 6.1 Implement News_Scraper service
    - Create `python-services/news_scraper/scraper.py`
    - Implement scraping logic for configured Pakistani newspaper sources using BeautifulSoup
    - Implement PSX announcements scraping
    - Extract article metadata: title (truncate to 500 chars), source, publication_date, full_text (truncate to 50,000 chars), generate summary (max 300 chars)
    - Implement stock symbol tagging: match PSX-listed ticker symbols against article title and body
    - Implement deduplication: check (source, title, publication_date) before insert
    - Implement retry logic: retry failed sources up to 3 times
    - Implement 6-month cleanup: delete articles with scraped_at older than 180 days (also handled by TTL index)
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 4.8, 4.9_

  - [ ]* 6.2 Write property test: Article symbol tagging (Property 8)
    - **Property 8: Article symbol tagging identifies all matching portfolio symbols**
    - Generate arbitrary article text containing random PSX symbols and a set of portfolio symbols
    - Assert: tagging function identifies exactly those portfolio symbols present in text, no false positives
    - Use Hypothesis library with minimum 100 iterations
    - **Validates: Requirements 4.4**

  - [ ]* 6.3 Write property test: Article field length constraints (Property 9)
    - **Property 9: Stored article fields respect maximum length constraints**
    - Generate articles with titles > 500 chars, texts > 50000 chars, summaries > 300 chars
    - Assert: after processing, title ≤ 500, full_text ≤ 50,000, summary ≤ 300
    - Use Hypothesis library with minimum 100 iterations
    - **Validates: Requirements 4.7**

  - [ ]* 6.4 Write property test: Article deduplication idempotence (Property 10)
    - **Property 10: Article storage is idempotent for duplicates**
    - Generate arbitrary article, store it N times (N > 1)
    - Assert: exactly one record exists with that (source, title, publication_date) tuple
    - Use Hypothesis library with minimum 100 iterations
    - **Validates: Requirements 4.9**

- [ ] 7. Implement AI Bot services (Python)
  - [ ] 7.1 Implement Bullish_Bot and Bearish_Bot analysis
    - Create `python-services/ai_bots/bullish_bot.py` — optimistic bias analysis
    - Create `python-services/ai_bots/bearish_bot.py` — pessimistic bias analysis
    - Both bots: read news/reports from DB for user's holdings, call LLM API with appropriate bias prompt
    - Produce per-stock recommendations: action (buy/hold/sell), conviction_score (1–10), reasoning (max 500 chars)
    - Analyze all current holdings first, then identify up to 5 additional stocks
    - _Requirements: 5.1, 5.2, 5.5, 5.6_

  - [ ] 7.2 Implement suggestion resolution logic
    - Create `python-services/ai_bots/resolver.py`
    - For each stock: select final suggestion from bot with higher conviction_score
    - Tie-breaking: current holdings default to "hold"; new recommendations are excluded
    - Cap new stock recommendations at 5
    - Store suggestions in DB with both bot analyses and final suggestion
    - Handle single-bot failure: store available analysis, mark final suggestion incomplete
    - Handle both-bot failure: log failure, retain previous day's suggestions, send admin alert
    - _Requirements: 5.3, 5.4, 5.7, 5.8, 5.9_

  - [ ]* 7.3 Write property test: Conviction score resolution (Property 11)
    - **Property 11: Final suggestion selects the bot with higher conviction score**
    - Generate arbitrary bullish and bearish analyses with conviction scores 1–10
    - Assert: final suggestion comes from higher-conviction bot; ties default to hold (holdings) or exclude (new)
    - Use Hypothesis library with minimum 100 iterations
    - **Validates: Requirements 5.3**

  - [ ]* 7.4 Write property test: All holdings receive suggestions (Property 12)
    - **Property 12: All current holdings receive suggestions**
    - Generate arbitrary portfolio with N holdings (N ≥ 1) and mock bot outputs
    - Assert: suggestion output contains a recommendation for every held stock
    - Use Hypothesis library with minimum 100 iterations
    - **Validates: Requirements 5.6**

  - [ ]* 7.5 Write property test: New recommendations capped at 5 (Property 13)
    - **Property 13: New stock recommendations are capped at 5**
    - Generate arbitrary bot outputs with varying numbers of new stock recommendations
    - Assert: final output contains at most 5 new stock recommendations
    - Use Hypothesis library with minimum 100 iterations
    - **Validates: Requirements 5.7**

- [ ] 8. Implement cron scheduling for Python services
  - [ ] 8.1 Set up cron job configuration for News_Scraper and AI Bots
    - Create `python-services/cron/scheduler.py` or crontab configuration
    - Schedule News_Scraper to run daily at configured time
    - Schedule AI Bots to run after News_Scraper completes
    - Implement retry logic: if service unreachable, retry once after 60 seconds
    - Log all execution results with timestamps
    - _Requirements: 4.1, 7.7_

- [ ] 9. Checkpoint - Python services complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 10. Implement backend REST API routes
  - [ ] 10.1 Implement portfolio and trade API endpoints
    - Create `backend/src/routes/portfolio.ts` — GET `/api/portfolio` (portfolio overview)
    - Create `backend/src/routes/trades.ts` — POST `/api/trades` (record trade), GET `/api/trades` (trade history)
    - Wire to PortfolioService and TradeService
    - Return proper error responses: 400 for validation errors, 500 for server errors
    - _Requirements: 1.1, 1.2, 1.7, 3.1, 3.2, 3.3, 3.6, 3.7, 3.8_

  - [ ] 10.2 Implement price refresh API endpoint
    - Create `backend/src/routes/prices.ts` — POST `/api/prices/refresh`
    - Wire to PriceFetchService
    - Return 200 on full success, 207 on partial success, 503 on service unavailable, 408 on timeout
    - Return empty-portfolio message if no holdings
    - _Requirements: 2.1, 2.4, 2.5, 2.6, 2.7, 7.5, 7.6_

  - [ ] 10.3 Implement news and suggestions API endpoints
    - Create `backend/src/routes/news.ts` — GET `/api/news` (paginated, filtered), GET `/api/news/:id` (article detail)
    - Create `backend/src/routes/suggestions.ts` — GET `/api/suggestions` (latest), GET `/api/suggestions/history`
    - Implement pagination (20 items/page), sorting (newest first), and AND-logic filtering (symbol, source, date range)
    - _Requirements: 6.1, 6.2, 6.4, 6.5, 6.6, 5.4, 5.7_

  - [ ]* 10.4 Write property test: News filtering logic (Property 14)
    - **Property 14: News filtering returns only articles matching all active filters**
    - Generate arbitrary articles and filter combinations (symbol, source, date range)
    - Assert: every result matches ALL active filters; results sorted newest-first; max 20 per page
    - Use fast-check library with minimum 100 iterations
    - **Validates: Requirements 6.1, 6.2, 6.4**

- [ ] 11. Implement React frontend
  - [ ] 11.1 Set up React app with routing and shared components
    - Initialize React app with TypeScript (Vite or Create React App)
    - Set up React Router with routes: `/portfolio`, `/trades`, `/news`, `/news/:id`, `/suggestions`
    - Create shared layout components: Navbar, Toast notification system, Loading spinner
    - Set up axios instance with base URL configuration
    - _Requirements: 7.1_

  - [ ] 11.2 Implement Portfolio page
    - Create `PortfolioPage` component displaying total investment, holdings table
    - Create `HoldingsTable` component: sorted alphabetically by symbol, showing symbol, quantity, avg price, current value, P/L, P/L%
    - Display stale price indicator (>24h) and "N/A" for unfetched prices
    - Implement empty portfolio state with CTA to record first trade
    - Create `PricesRefresh` button component with loading state and 30s timeout handling
    - Display partial fetch errors per-symbol
    - Round all monetary/percentage values to 2 decimal places
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8, 2.1, 2.3, 2.4, 2.5, 2.6_

  - [ ] 11.3 Implement Trade page
    - Create `TradePage` with `TradeForm` component
    - Form fields: ticker symbol (autocomplete from PSX list), trade type (buy/sell), quantity, price per share
    - Client-side validation: quantity 1–1,000,000; price 0.01–99,999.99; valid PSX ticker
    - Display inline validation error messages
    - Show sell-exceeds-holdings error
    - Display trade history list
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.6, 3.7, 3.8_

  - [ ] 11.4 Implement News browsing page
    - Create `NewsPage` with paginated article list (20 per page)
    - Create `NewsFilter` component: filter by stock symbol, source, date range
    - Visually distinguish articles related to user's holdings (highlight indicator)
    - Sort by publication date newest-first
    - Show "no results" state with clear-filters button
    - Create `ArticleDetail` page: title, source, date, related symbols, full text
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6_

  - [ ] 11.5 Implement Suggestions page
    - Create `SuggestionsPage` displaying daily AI recommendations
    - Create `SuggestionCard` component showing: final action, both bot analyses (action, conviction, reasoning)
    - Separate sections: current holdings suggestions and new stock recommendations
    - Handle unavailable suggestions: show notification, display previous day's fallback
    - Handle partial bot failure: show available analysis, indicate missing perspective
    - _Requirements: 5.3, 5.4, 5.7, 5.8, 5.9_

- [ ] 12. Checkpoint - Frontend complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 13. Integration wiring and final setup
  - [ ] 13.1 Wire backend Express app entry point
    - Create `backend/src/app.ts` — Express app setup with CORS, JSON parsing, route mounting
    - Create `backend/src/server.ts` — Server startup with MongoDB connection
    - Mount all route modules under `/api` prefix
    - Add global error handling middleware
    - _Requirements: 7.2, 7.4_

  - [ ] 13.2 Wire frontend API integration
    - Create `frontend/src/api/` module with typed API client functions for all endpoints
    - Connect Portfolio page to GET `/api/portfolio` and POST `/api/prices/refresh`
    - Connect Trade page to POST `/api/trades` and GET `/api/trades`
    - Connect News page to GET `/api/news` and GET `/api/news/:id`
    - Connect Suggestions page to GET `/api/suggestions`
    - Implement error handling: toast notifications, retry options, data preservation on failure
    - _Requirements: 7.1, 7.2_

  - [ ]* 13.3 Write integration tests for cross-service flows
    - Test price fetch flow: API → Python Price_Fetcher → mock PSX → DB storage
    - Test trade recording with DB persistence and holding update
    - Test news endpoint with filtering, pagination, and sorting
    - Test timeout and error scenarios
    - _Requirements: 7.4, 7.5, 7.6_

- [ ] 14. Final checkpoint - All services integrated
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties from the design document
- Unit tests validate specific examples and edge cases
- TypeScript is used for frontend (React) and backend (Node.js); Python for scraping and AI services
- fast-check library is used for property tests in TypeScript; Hypothesis for Python property tests
- All property tests require minimum 100 iterations as specified in the design

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["1.2", "1.3"] },
    { "id": 2, "tasks": ["2.1", "3.1", "4.1"] },
    { "id": 3, "tasks": ["2.2", "2.3", "2.4", "3.2", "3.3", "3.4", "3.5", "4.2"] },
    { "id": 4, "tasks": ["6.1", "7.1", "8.1"] },
    { "id": 5, "tasks": ["6.2", "6.3", "6.4", "7.2"] },
    { "id": 6, "tasks": ["7.3", "7.4", "7.5"] },
    { "id": 7, "tasks": ["10.1", "10.2", "10.3"] },
    { "id": 8, "tasks": ["10.4", "11.1"] },
    { "id": 9, "tasks": ["11.2", "11.3", "11.4", "11.5"] },
    { "id": 10, "tasks": ["13.1", "13.2"] },
    { "id": 11, "tasks": ["13.3"] }
  ]
}
```
