# Portfolio Analyzer

Portfolio Analyzer is a Streamlit web app that turns raw broker trade exports into a clear picture of your investments. You upload your yearly trade CSV files, and the app adjusts for stock splits, rebuilds your holdings day by day, values them using historical prices, and calculates the annualized return (XIRR) for every holding. It also shows the latest news for each stock and includes an AI assistant, powered by Groq and Llama 3.3, that answers questions about your trades in plain English.

## Overview

Brokers give you a raw log of trades, not your actual returns. Stock splits distort quantities and prices, trades happen on irregular dates, and a simple profit-over-cost number ignores when the money actually went in. This project answers the questions that matter: what was my portfolio worth on a given day, what is my annualized return on each holding, what is happening with the companies I own, and can I ask my data a question directly.

Everything runs locally in a single Streamlit app. Upload your files and the dashboard builds itself.

## Features

- Daily portfolio value, calculated from your positions and historical closing prices. Weekends and holidays are forward-filled.
- Automatic stock split adjustment using split history from Yahoo Finance.
- XIRR per holding, which accounts for the exact date of every cash flow.
- Per-holding view: pick any ticker and see its value over time.
- Historical USD, INR and SGD forex rates fetched from Yahoo Finance and merged with your trades.
- Latest news: the top 5 recent headlines per holding from the Finnhub API.
- AI portfolio assistant: a ReAct agent (Groq, Llama 3.3 70B) that writes and runs pandas code on your trades to answer questions.
- A Clear Data button to reset the session and upload a new set of files.

## Tech Stack

- Streamlit for the interface
- pandas and NumPy for data processing
- yfinance for prices, splits and forex rates
- Finnhub REST API for news
- Groq API with llama-3.3-70b-versatile for the AI assistant
- SciPy (Newton method) for XIRR, with a built-in bisection fallback

## How It Works

1. Every uploaded CSV is read, and only rows where DataDiscriminator is "Order" are kept.
2. Numeric fields are cleaned, dates are parsed, and columns are renamed to standard names.
3. Each ticker's split history is fetched from Yahoo Finance and earlier trades are adjusted.
4. A cash flow is computed for every trade.
5. Historical prices for all tickers and forex rates for USD, INR and SGD are downloaded.
6. The app walks through every calendar day, updates positions, and values them at that day's closing price.
7. XIRR is computed for each holding using the trade cash flows plus the current market value as the final inflow.
8. The latest headlines are fetched for each ticker.

## Input Format

Upload one or more trade CSV files. The app was designed for three yearly files (for example 2023, 2024 and 2025), but any number of files works and they are combined automatically.

The files are expected to have these columns, which is the layout used in broker activity statements such as Interactive Brokers:

```csv
Trades,Header,DataDiscriminator,Asset Category,Currency,Symbol,Date/Time,Quantity,T. Price,C. Price,Proceeds,Comm/Fee,Basis,Realized P/L,MTM P/L,Code
```

Things to know:

- Only rows with DataDiscriminator equal to "Order" are used.
- Symbol must be a valid Yahoo Finance ticker, otherwise prices and splits cannot be fetched.
- Sells should have a negative Quantity.
- The ticker C6L is skipped on purpose because no data is available for it on Yahoo Finance.

## Installation

You need Python 3.11 or newer, a free Finnhub API key (for news), and a free Groq API key (for the AI assistant).

```bash
git clone https://github.com/souvik-2003/AI_portfolio_Analyzer.git
cd AI_portfolio_Analyzer

python -m venv venv
source venv/bin/activate        # on Windows: venv\Scripts\activate

pip install -r requirements.txt
```

## Configuration

Create a file at `.streamlit/secrets.toml` in the project root with your keys:

```toml
FINHUB_API = "your_finnhub_api_key"
GROQ_API_KEY = "your_groq_api_key"
```

The Finnhub key name is spelled FINHUB_API (one n) in the code, so keep it exactly as written. Add `.streamlit/secrets.toml` to your `.gitignore` so your keys are never committed.

## Running the App

```bash
streamlit run stock_price_analyzer.py
```

Then open http://localhost:8501 in your browser.

## Usage

Upload your trade CSV files on the main page. The app cleans the data and adjusts for stock splits, then loads the dashboard. From top to bottom the dashboard shows the historical price table for all holdings, the daily portfolio value chart, an individual holding chart selected from a dropdown, the latest 10 rows of the daily portfolio table, the XIRR for each holding, and the latest news headlines.

The sidebar has short FAQs, the AI chat, and a Clear Data button that resets everything so you can upload new files.

## AI Portfolio Assistant

Once your data is loaded, you can type questions in the sidebar chat. Examples: which ticker did I trade the most, how much commission have I paid in total, what is my total invested amount per ticker, how many trades did I make each year.

The assistant works as a ReAct loop. The model receives the column names of your combined trades DataFrame and must reply with a Thought, an Action and an Action Input. The generated pandas code is run in a Python REPL tool against your data (available as df, with pd and np). The output is sent back to the model, and a second call turns it into a short, finance-friendly answer. The loop runs for at most 5 iterations, and the thought, code and observation for every answer can be opened in an expandable panel.

Security note: the assistant runs LLM-generated Python with exec() and no sandbox. Use it locally with your own data, and do not expose the app publicly without adding proper sandboxing.

## Methodology

Split adjustment: for every split with ratio r on date S, all trades of that ticker before S are rescaled.

```
Quantity          = Quantity * r
Transaction_Price = Transaction_Price / r
```

Cash flow per trade: buys are negative (money out) and sells are positive (money in).

```
Cash_Flow = -1 * Quantity * Transaction_Price
```

Portfolio value on a given day is the sum, over all holdings, of the quantity held multiplied by that day's closing price.

XIRR is the annualized rate r that solves the following equation, where CF are the trade cash flows, d is the date of each cash flow, and the last cash flow is the current holding valued at the latest closing price. It is solved with SciPy's Newton method, falling back to bisection if SciPy is not installed.

```
sum( CF_i / (1 + r) ^ ((d_i - d_0) / 365) ) = 0
```

## Project Structure

```
AI_portfolio_Analyzer/
    stock_price_analyzer.py    Streamlit app: UI, data pipeline, XIRR and AI agent
    requirements.txt           Python dependencies
    README.md
    .streamlit/secrets.toml    API keys (create locally, do not commit)
```

## Known Limitations

- Portfolio value and XIRR use each ticker's native price, and totals are labelled USD. Forex rates are fetched but not yet applied to convert holdings in mixed currencies.
- Commissions and fees are not included in the XIRR cash flows.
- The news request uses a fixed start date, and the Finnhub key is required even if you only want the charts.
- Yahoo Finance data is unofficial and may be missing or delayed for some tickers.

Planned improvements: apply forex conversion to portfolio value and XIRR, add a portfolio-level XIRR, include commissions in returns, cache yfinance calls, sandbox the AI assistant's code execution, and add SciPy to requirements.txt along with tests for the XIRR and split logic.

## Troubleshooting

- KeyError for FINHUB_API at the bottom of the dashboard: the key is missing from `.streamlit/secrets.toml`.
- The AI chat shows an LLM error: GROQ_API_KEY is missing or invalid.
- A holding is missing from the charts or XIRR table: the ticker was not found on Yahoo Finance or has no price history.
- XIRR shows N/A: there is not enough cash-flow history for that ticker.
- Nothing happens after upload: check that the CSV has the columns listed under Input Format.

## Contributing

Issues and pull requests are welcome. For larger changes, please open an issue first to discuss what you would like to change.
