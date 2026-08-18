# AI Stock Market News → Discord (Multi-Channel Sentiment Routing)

An n8n workflow that pulls live stock market news from the Marketaux API, runs each
article through an AI agent for sentiment/impact analysis, and auto-posts a formatted
alert to the matching Discord channel — Bullish, Bearish, Neutral, or High Impact.

## What it does

1. Runs on a schedule (daily at 6 AM)
2. Fetches the latest news for a watchlist of stock symbols (e.g. `AAPL, TSLA, MSFT, NVDA`) from the Marketaux News API
3. Splits the batch response into individual articles
4. A Code node cleans and extracts each article's entities, symbols, sentiment scores, and highlights into a compact text block for the AI
5. An AI Agent (Google Gemini) analyzes each article and returns structured output: `route` (BULLISH / BEARISH / NEUTRAL / HIGH_IMPACT), `impact`, `confidence`, `summary`, `why_it_matters`, `key_factors`, `potential_market_effect`, `time_horizon`, and more — validated against a strict JSON schema via a Structured Output Parser
6. A Switch node routes each article based on `route`
7. Each branch posts a formatted, emoji-coded message to its own Discord channel

The AI is explicitly instructed **not** to give BUY/SELL/HOLD recommendations or financial
advice — it classifies sentiment and impact only, and every Discord message ends with an
"informational analysis, not investment advice" disclaimer.

## Architecture

```
Schedule Trigger (daily, 6 AM)
        │
HTTP Request — Get Market Data (Marketaux News API)
        │
Split Out — Data (one item per article)
        │
Code — Clean/Extract News (parses entities, symbols, sentiment, highlights)
        │
AI Agent (Google Gemini) ──uses──> Structured Output Parser (JSON schema)
        │
Switch — Route (BULLISH / BEARISH / NEUTRAL / HIGH_IMPACT)
        │
        ├──> Discord — Bullish channel
        ├──> Discord — Bearish channel
        ├──> Discord — Neutral channel
        └──> Discord — High Impact channel
```

> Add a screenshot of the actual n8n canvas here: `screenshots/canvas.png`
> Add a screenshot of a sample Discord message here: `screenshots/discord-message.png`

## Stack

- **n8n** — orchestration
- **Marketaux API** — real-time stock market news + entity/sentiment data
- **Google Gemini** (via n8n's LangChain Agent node) — news analysis and classification
- **n8n Structured Output Parser** — enforces a strict JSON schema on the AI's response
- **Discord API (OAuth2)** — multi-channel message delivery

## AI classification logic

The agent classifies each article into exactly one route:

| Route | Meaning |
|---|---|
| `BULLISH` | Generally positive developments for the company/stock |
| `BEARISH` | Generally negative developments for the company/stock |
| `NEUTRAL` | Mixed, unclear, or unlikely to materially affect the stock |
| `HIGH_IMPACT` | Flagged separately based on the `impact` field regardless of sentiment direction |

It also returns `impact` (LOW/MEDIUM/HIGH), a 0–100 `confidence` score, and `time_horizon`
(SHORT_TERM/MEDIUM_TERM/LONG_TERM) — full schema in the "Structured Output Parser" node.

## Setup

1. Import `workflow.json` into your n8n instance
2. Get a free or paid API key from [marketaux.com](https://www.marketaux.com/) and set it in the "Get Market Data" HTTP Request node's `api_token` query parameter
3. Update the `symbols` query parameter with your own stock watchlist
4. Create a Discord server (or use an existing one) with 4 channels for Bullish / Bearish / Neutral / High Impact news
5. Set up Discord OAuth2 credentials in n8n and connect them on all 4 Discord nodes
6. Replace these placeholders in `workflow.json` with your own values:
   - `YOUR_DISCORD_GUILD_ID` → your Discord server ID
   - `YOUR_BULLISH_CHANNEL_ID`, `YOUR_BEARISH_CHANNEL_ID`, `YOUR_NEUTRAL_CHANNEL_ID`, `YOUR_HIGH_IMPACT_CHANNEL_ID` → your channel IDs
   - `YOUR_MARKETAUX_API_KEY` → your Marketaux API key
7. Connect a Google Gemini API credential to the "Google Gemini Chat Model" node
8. Activate the workflow

## Files

- `workflow.json` — sanitized n8n export (API key, credential IDs, Discord server/channel IDs all replaced with placeholders)
- `.env.example` — reference list of values you'll need to configure
- `screenshots/` — canvas view and sample Discord output (add your own)

## Notes

This is a sanitized, portfolio version of a working automation. The original export had a
live Marketaux API key hardcoded in plain text plus real Discord server/channel IDs — all
of that has been removed and replaced with placeholders. No real API keys or Discord
identifiers are included in this repo.
