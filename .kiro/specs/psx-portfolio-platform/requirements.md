# Requirements Document

## Introduction

A PSX (Pakistan Stock Exchange) portfolio management platform that enables investors to track their stock holdings, monitor profit/loss, fetch live market prices on demand, and receive AI-driven daily investment suggestions. The platform combines a React frontend, Node.js backend, and Python services for scraping and AI analysis. A daily cron job scrapes financial newspapers and company reports, storing 6 months of data. Two AI bots (one bullish, one bearish) analyze the data and produce competing recommendations — the bot with stronger conviction drives the final suggestion.

## Glossary

- **Platform**: The PSX Portfolio Management web application comprising the React frontend, Node.js API server, and Python scraping/AI services
- **Portfolio**: A user's collection of stock holdings on PSX, including quantities, average buying prices, and current values
- **Holding**: A single stock position within a Portfolio, identified by ticker symbol, quantity, and average buying price
- **Trade**: A buy or sell transaction of a PSX-listed stock that modifies the user's Holdings
- **Price_Fetcher**: The Python service that scrapes current stock prices from PSX on user demand
- **News_Scraper**: The Python cron-based service that scrapes daily newspapers and company reports
- **Bullish_Bot**: The AI agent that analyzes data with an optimistic bias, looking for reasons to buy or hold
- **Bearish_Bot**: The AI agent that analyzes data with a pessimistic bias, looking for reasons to sell or avoid
- **Conviction_Score**: A numerical score (1-10) assigned by each AI bot indicating strength of its recommendation
- **Suggestion**: A daily AI-generated recommendation to buy, hold, or sell specific stocks
- **Report_Store**: The database collection storing 6 months of scraped news articles and company reports

## Requirements

### Requirement 1: Portfolio Overview Display

**User Story:** As an investor, I want to see my total investment size, all holdings, and average buying prices at a glance, so that I can quickly assess my portfolio state.

#### Acceptance Criteria

1. WHEN a user navigates to the portfolio page, THE Platform SHALL display the total investment amount as the sum of (quantity × average_buying_price) for all Holdings
2. WHEN a user navigates to the portfolio page, THE Platform SHALL display each Holding with its ticker symbol, quantity, average buying price, current value (current_price × quantity), and profit/loss amount, sorted by ticker symbol in alphabetical order
3. THE Platform SHALL calculate profit/loss for each Holding as (current_price - average_buying_price) × quantity
4. THE Platform SHALL calculate profit/loss percentage for each Holding as ((current_price - average_buying_price) / average_buying_price) × 100
5. IF current_price data has not been fetched for a Holding within the last 24 hours, THEN THE Platform SHALL display the last fetched price with a timestamp indicating when it was retrieved
6. IF no price has ever been fetched for a Holding, THEN THE Platform SHALL display the current value and profit/loss fields as "N/A" and indicate that a price fetch is required
7. WHEN a user navigates to the portfolio page and the Portfolio contains zero Holdings, THE Platform SHALL display a message indicating the portfolio is empty and prompt the user to record a trade
8. THE Platform SHALL display all monetary values rounded to 2 decimal places and all percentage values rounded to 2 decimal places

### Requirement 2: Manual Price Fetch

**User Story:** As an investor, I want to manually trigger a fetch of current stock prices, so that I can see up-to-date valuations without relying on automated polling.

#### Acceptance Criteria

1. WHEN a user clicks the "Refresh Prices" button, THE Price_Fetcher SHALL scrape current prices for all stocks in the user's Portfolio from PSX and complete the operation within 30 seconds
2. WHEN the Price_Fetcher completes a fetch, THE Platform SHALL store each successfully fetched price in the database with the timestamp of retrieval
3. WHEN the Price_Fetcher completes a fetch, THE Platform SHALL update all displayed Holding values and profit/loss calculations using the newly fetched prices
4. IF the Price_Fetcher fails to retrieve data for one or more stocks but succeeds for others, THEN THE Platform SHALL store and display the successfully fetched prices, display an error message identifying each failed stock by ticker symbol, and retain the previous price data for the failed stocks
5. WHILE the Price_Fetcher is retrieving data, THE Platform SHALL display a loading indicator and disable the "Refresh Prices" button to prevent concurrent fetch requests
6. IF the Price_Fetcher does not complete within 30 seconds, THEN THE Platform SHALL abort the fetch operation, display an error message indicating a timeout, retain all previous price data, and re-enable the "Refresh Prices" button
7. IF the user clicks the "Refresh Prices" button and the Portfolio contains no Holdings, THEN THE Platform SHALL display a message indicating there are no stocks to fetch prices for

### Requirement 3: Trade Recording

**User Story:** As an investor, I want to record buy and sell trades, so that my holdings and average buying prices are automatically updated.

#### Acceptance Criteria

1. WHEN a user submits a buy trade, THE Platform SHALL increase the Holding quantity by the traded quantity and recalculate the average buying price as ((old_qty × old_avg_price) + (new_qty × trade_price)) / (old_qty + new_qty)
2. WHEN a user submits a sell trade, THE Platform SHALL decrease the Holding quantity by the traded quantity while maintaining the existing average buying price
3. IF a user submits a sell trade with quantity exceeding the current Holding quantity, THEN THE Platform SHALL reject the trade and display an error message indicating the sell quantity exceeds available holdings
4. WHEN a user submits a buy trade for a stock not currently in the Portfolio, THE Platform SHALL create a new Holding with the traded quantity and price as the average buying price
5. WHEN a sell trade reduces a Holding quantity to zero, THE Platform SHALL remove the Holding from the active Portfolio and move it to trade history
6. THE Platform SHALL store each Trade with: ticker symbol, trade type (buy/sell), quantity, price per share, total amount, and trade submission timestamp
7. IF a user submits a trade with a quantity less than 1 share or a price per share less than 0.01, THEN THE Platform SHALL reject the trade and display an error message indicating the invalid field and its acceptable range (quantity: 1 to 1,000,000 shares; price: 0.01 to 99,999.99 per share)
8. IF a user submits a trade with a ticker symbol that is not a valid PSX-listed stock, THEN THE Platform SHALL reject the trade and display an error message indicating the ticker symbol is not recognized

### Requirement 4: Daily News and Report Scraping

**User Story:** As an investor, I want the system to automatically scrape financial newspapers and company reports daily, so that I have current market intelligence without manual effort.

#### Acceptance Criteria

1. THE News_Scraper SHALL execute once daily via a cron job at a configured time
2. WHEN the News_Scraper executes, THE News_Scraper SHALL scrape financial news from configured Pakistani newspaper sources
3. WHEN the News_Scraper executes, THE News_Scraper SHALL scrape company reports and announcements from PSX and configured sources
4. WHEN the News_Scraper stores a scraped article, THE News_Scraper SHALL tag the article with any stock symbols from the user's current Holdings that appear in the article title or body, and SHALL sort tagged articles before untagged articles when presenting results
5. THE Report_Store SHALL retain scraped articles and reports for 6 months from their scrape date
6. WHEN a scraped article is older than 6 months, THE Report_Store SHALL delete it during the daily cleanup cycle
7. THE News_Scraper SHALL extract and store for each article: title (maximum 500 characters), source name, publication date, full text (maximum 50,000 characters), stock symbols identified by matching PSX-listed ticker symbols found in the article title or body, and a plain-text summary of no more than 300 characters
8. IF the News_Scraper fails to scrape a source after 3 retry attempts, THEN THE News_Scraper SHALL log the failure with source name and error reason, and continue scraping remaining sources
9. IF the News_Scraper encounters an article with the same source, title, and publication date as an existing stored article, THEN THE News_Scraper SHALL skip the duplicate article without storing it

### Requirement 5: AI-Driven Daily Suggestions

**User Story:** As an investor, I want to receive daily AI-generated suggestions about whether to hold, buy, or sell stocks, so that I can make informed investment decisions.

#### Acceptance Criteria

1. WHEN the daily analysis cycle runs (after News_Scraper completes), THE Bullish_Bot SHALL analyze the scraped news, reports, and market data with an optimistic bias and produce a recommendation with a Conviction_Score (1-10) for each stock in the user's Holdings and for up to 5 additional PSX-listed stocks identified as potential opportunities
2. WHEN the daily analysis cycle runs, THE Bearish_Bot SHALL analyze the same data with a pessimistic bias and produce a recommendation with a Conviction_Score (1-10) for each stock in the user's Holdings and for up to 5 additional PSX-listed stocks identified as potential risks or avoidances
3. WHEN both bots have completed analysis, THE Platform SHALL select the final Suggestion from the bot with the higher Conviction_Score for each stock; IF both bots produce equal Conviction_Scores, THEN THE Platform SHALL default to a "hold" recommendation for current Holdings and exclude the stock from new recommendations
4. THE Platform SHALL display both the Bullish_Bot and Bearish_Bot analyses alongside the final Suggestion, showing each bot's action, Conviction_Score, and reasoning summary so the user can see both perspectives
5. THE Platform SHALL generate Suggestions that include: action (buy/hold/sell), target stock symbol, reasoning summary (maximum 500 characters), and the Conviction_Scores from both bots
6. WHEN generating suggestions, THE Bullish_Bot and Bearish_Bot SHALL analyze news and reports related to the user's current Holdings before analyzing non-held stocks, and SHALL produce recommendations for all current Holdings regardless of available data volume
7. THE Platform SHALL present Suggestions in two categories: recommendations for current Holdings and recommendations for new stocks to consider (maximum 5 new stocks per daily cycle)
8. IF either the Bullish_Bot or the Bearish_Bot fails to complete analysis, THEN THE Platform SHALL display the available bot's analysis without a final Suggestion selection, and indicate that the opposing perspective is unavailable
9. IF both the Bullish_Bot and the Bearish_Bot fail to complete analysis, THEN THE Platform SHALL display a notification informing the user that daily suggestions are unavailable and retain the previous day's Suggestions as the latest visible recommendations

### Requirement 6: News and Reports Browsing

**User Story:** As an investor, I want to browse stored news articles and company reports, so that I can conduct my own research alongside the AI suggestions.

#### Acceptance Criteria

1. WHEN a user navigates to the news section, THE Platform SHALL display articles sorted by publication date (newest first) with a maximum of 20 articles per page and pagination controls to access older articles
2. THE Platform SHALL allow filtering articles by stock symbol, source, and date range, applying all selected filters as a combined (AND) condition to narrow results
3. THE Platform SHALL visually distinguish articles related to the user's current Holdings from other articles by displaying a persistent visual indicator adjacent to the article title
4. WHEN a user selects a specific Holding, THE Platform SHALL display all related news articles and reports for that company, paginated at 20 items per page and sorted by publication date (newest first)
5. WHEN a user selects an article from the list, THE Platform SHALL display the article detail view including: title, source, publication date, related stock symbols, and full text content
6. IF the applied filters return no matching articles, THEN THE Platform SHALL display a message indicating no articles match the current filter criteria and provide an option to clear all filters

### Requirement 7: Technology Stack and Architecture

**User Story:** As a developer, I want the platform built with React, Node.js, and Python, so that we leverage appropriate tools for each layer.

#### Acceptance Criteria

1. THE Platform SHALL use React for the frontend user interface
2. THE Platform SHALL use Node.js for the backend API server handling authentication, portfolio CRUD, and trade operations, exposed as a REST API consumed by the React frontend
3. THE Platform SHALL use Python for the Price_Fetcher service, News_Scraper service, and AI bot analysis
4. THE Platform SHALL store all persistent data (Holdings, Trades, prices, news) in a database accessible by both the Node.js API and Python services
5. WHEN the Node.js API receives a price fetch request, THE Platform SHALL invoke the Python Price_Fetcher service and return results to the frontend within 30 seconds
6. IF the Node.js API fails to receive a response from the Python Price_Fetcher service within 30 seconds, THEN THE Platform SHALL return an error response to the frontend indicating the price service is unavailable and preserve any previously stored price data
7. IF the News_Scraper or AI bot Python services are unreachable during their scheduled execution, THEN THE Platform SHALL log the failure with a timestamp and retry the operation once after 60 seconds before marking the cycle as failed
