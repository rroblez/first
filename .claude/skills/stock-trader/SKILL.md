---
name: stock-trader
description: "Use this skill when setting up or running autonomous stock trading operations. The system continuously researches stocks, makes trading decisions without human input, executes trades on Alpaca paper trading, and sends SMS notifications via Twilio. Based on AI-Trader's autonomous agent pattern."
---
 
Stock Trading Skill
When to Use
Activate when:
Setting up autonomous trading operations
Configuring continuous background trading
Implementing stock screening and selection
Setting up scheduled trading jobs
Creating notification systems for autonomous trades
Architecture: Fully Autonomous System
Based on AI-Trader Pattern (refs/AI-Trader/)
┌─────────────────────────────────────────────────┐
│         CONTINUOUS BACKGROUND LOOP              │
│                                                 │
│  Every Market Day:                              │
│  ├── 9:00 AM: Pre-market screening             │
│  ├── 9:30 AM: Market open → Analyze & Trade    │
│  ├── 12:00 PM: Mid-day review                  │
│  ├── 4:00 PM: Market close → Daily summary     │
│  └── 5:00 PM: Post-market analysis             │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│         AUTONOMOUS STOCK SELECTION              │
│                                                 │
│  1. Screen Universe (NASDAQ 100)                │
│  2. Filter by Criteria:                         │
│     - Volume > 1M shares/day                    │
│     - Price movement > 2% (volatility)          │
│     - News events in last 24h                   │
│  3. Rank by Opportunity Score                   │
│  4. Select Top 5-10 for Deep Analysis          │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│         MULTI-MODEL ANALYSIS                    │
│         (Parallel Execution)                    │
│                                                 │
│  For each selected stock:                       │
│  ├── Run all 5 AI models                       │
│  ├── Each model uses TradingAgents framework   │
│  ├── Aggregate decisions                        │
│  └── Consensus → TRADE or PASS                  │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│         AUTOMATIC TRADE EXECUTION               │
│                                                 │
│  If consensus reached:                          │
│  ├── Risk check (position sizing)              │
│  ├── Submit order to Alpaca (paper trading)    │
│  ├── Log to database                           │
│  └── Send SMS notification                      │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│         SMS NOTIFICATIONS (Twilio)              │
│                                                 │
│  Real-time updates to user, kept short for SMS: │
│  ├── "Screening 100 stocks..."                  │
│  ├── "NVDA: 4/5 models -> BUY"                  │
│  ├── "Bought 10 NVDA @ $850.50"                 │
│  └── "Daily P&L: +$234.50 (+2.3%)"              │
└─────────────────────────────────────────────────┘
Implementation Components
1. Stock Screening & Selection
# src/autonomous/stock_screener.py
 
from typing import List, Dict
import yfinance as yf
from datetime import datetime, timedelta
 
class AutonomousStockScreener:
    """
    Automatically select stocks for analysis
    Based on AI-Trader's autonomous decision-making
    """
 
    # NASDAQ 100 universe (from refs/AI-Trader/agent/base_agent/base_agent.py:44-56)
    STOCK_UNIVERSE = [
        "NVDA", "MSFT", "AAPL", "GOOG", "GOOGL", "AMZN", "META", "AVGO", "TSLA",
        "NFLX", "PLTR", "COST", "ASML", "AMD", "CSCO", "AZN", "TMUS", "MU",
        # ... (full list from AI-Trader)
    ]
 
    def __init__(self):
        self.criteria = {
            "min_volume": 1_000_000,      # Minimum 1M shares/day
            "min_price_change": 2.0,      # At least 2% movement
            "max_positions": 10,          # Max stocks to analyze
            "lookback_days": 5            # Look back 5 days
        }
 
    async def screen_stocks(self) -> List[Dict]:
        """
        Screen entire universe and select top candidates
        """
        print("🔍 Screening universe of stocks...")
 
        candidates = []
 
        for ticker in self.STOCK_UNIVERSE:
            try:
                # Get recent data
                stock = yf.Ticker(ticker)
                hist = stock.history(period="5d")
 
                if len(hist) < 2:
                    continue
 
                # Calculate metrics
                latest_price = hist['Close'].iloc[-1]
                prev_price = hist['Close'].iloc[-2]
                price_change_pct = ((latest_price - prev_price) / prev_price) * 100
                avg_volume = hist['Volume'].mean()
 
                # Apply filters
                if (avg_volume >= self.criteria["min_volume"] and
                    abs(price_change_pct) >= self.criteria["min_price_change"]):
 
                    # Check for recent news
                    news = stock.news[:3] if hasattr(stock, 'news') else []
                    has_news = len(news) > 0
 
                    # Calculate opportunity score
                    opportunity_score = (
                        abs(price_change_pct) * 0.4 +      # Price movement
                        (avg_volume / 10_000_000) * 0.3 +  # Volume factor
                        (10 if has_news else 0) * 0.3      # News catalyst
                    )
 
                    candidates.append({
                        "ticker": ticker,
                        "price": latest_price,
                        "price_change_pct": price_change_pct,
                        "volume": avg_volume,
                        "has_news": has_news,
                        "opportunity_score": opportunity_score
                    })
 
            except Exception as e:
                print(f"Error screening {ticker}: {e}")
                continue
 
        # Sort by opportunity score
        candidates.sort(key=lambda x: x["opportunity_score"], reverse=True)
 
        # Select top N
        selected = candidates[:self.criteria["max_positions"]]
 
        print(f"✅ Selected {len(selected)} stocks for analysis:")
        for s in selected:
            print(f"  • {s['ticker']}: {s['price_change_pct']:+.2f}% | "
                  f"Vol: {s['volume']/1e6:.1f}M | "
                  f"Score: {s['opportunity_score']:.2f}")
 
        return selected
2. Autonomous Trading Loop
# src/autonomous/trading_loop.py
 
import asyncio
from datetime import datetime, time
import pytz
from apscheduler.schedulers.asyncio import AsyncIOScheduler
 
class AutonomousTradingLoop:
    """
    Main autonomous trading loop
    Runs continuously in background
    """
 
    def __init__(self, sms_notifier, models_orchestrator, alpaca_client):
        self.notifier = sms_notifier
        self.orchestrator = models_orchestrator
        self.alpaca = alpaca_client
        self.screener = AutonomousStockScreener()
        self.scheduler = AsyncIOScheduler(timezone=pytz.timezone('US/Eastern'))
 
    def start(self):
        """
        Schedule autonomous trading jobs
        """
        # Pre-market screening (9:00 AM ET)
        self.scheduler.add_job(
            self.pre_market_screening,
            'cron',
            day_of_week='mon-fri',
            hour=9,
            minute=0
        )
 
        # Market open analysis & trading (9:30 AM ET)
        self.scheduler.add_job(
            self.market_open_trading,
            'cron',
            day_of_week='mon-fri',
            hour=9,
            minute=30
        )
 
        # Mid-day check (12:00 PM ET)
        self.scheduler.add_job(
            self.mid_day_review,
            'cron',
            day_of_week='mon-fri',
            hour=12,
            minute=0
        )
 
        # Market close analysis (4:00 PM ET)
        self.scheduler.add_job(
            self.market_close_summary,
            'cron',
            day_of_week='mon-fri',
            hour=16,
            minute=0
        )
 
        # Start scheduler
        self.scheduler.start()
        print("🤖 Autonomous trading loop started!")
 
    async def pre_market_screening(self):
        """
        9:00 AM: Screen stocks for today's trading
        """
        await self.notifier.send_sms(
            "Pre-Market Screening Started. Analyzing NASDAQ 100 stocks..."
        )
 
        # Screen stocks
        selected_stocks = await self.screener.screen_stocks()
 
        # Send results
        # SMS has no markdown and a ~160-char-per-segment cost, so keep this
        # terse rather than mirroring a Telegram-style message
        message = "Today's Watchlist: " + ", ".join(
            f"{stock['ticker']} ({stock['price_change_pct']:+.1f}%)"
            for stock in selected_stocks
        )
 
        await self.notifier.send_sms(message)
 
        # Store for market open
        self.daily_watchlist = selected_stocks
 
    async def market_open_trading(self):
        """
        9:30 AM: Analyze watchlist and execute trades
        """
        await self.notifier.send_sms(
            f"Market Open - analyzing {len(self.daily_watchlist)} stocks..."
        )
 
        for stock in self.daily_watchlist:
            await self.analyze_and_trade(stock)
 
        await self.notifier.send_sms("Market Open analysis complete")
 
    async def analyze_and_trade(self, stock: Dict):
        """
        Run multi-model analysis and execute trade if consensus
        """
        ticker = stock['ticker']
 
        # Notify start
        await self.notifier.send_sms(f"Analyzing {ticker}...")
 
        # Run all models in parallel
        model_results = await self.orchestrator.run_all_models(ticker)
 
        # Aggregate decisions
        aggregated = self.orchestrator.aggregate_decisions(model_results)
 
        # Check consensus
        if aggregated['high_consensus']:  # >70% agreement
            decision = aggregated['majority_decision']
 
            # Send analysis summary
            message = self._format_analysis_result(ticker, aggregated)
            await self.notifier.send_sms(message)
 
            # Execute trade if BUY/SELL (not HOLD)
            if decision in ['BUY', 'SELL']:
                await self._execute_autonomous_trade(
                    ticker,
                    decision,
                    aggregated
                )
        else:
            # Low consensus - skip trade
            await self.notifier.send_sms(
                f"Skipping {ticker} - low consensus "
                f"({aggregated['consensus_level']:.0%})"
            )
 
    async def _execute_autonomous_trade(
        self,
        ticker: str,
        action: str,
        analysis: Dict
    ):
        """
        Automatically execute trade on Alpaca paper trading
        """
        # Calculate position size (risk-based)
        portfolio = self.alpaca.get_account()
        portfolio_value = float(portfolio.portfolio_value)
 
        # Risk 2-5% per position based on consensus strength
        risk_pct = 0.02 + (analysis['consensus_level'] - 0.7) * 0.1
        position_value = portfolio_value * risk_pct
 
        # Get current price
        current_price = self.alpaca.get_latest_trade(ticker).price
        shares = int(position_value / current_price)
 
        if shares < 1:
            await self.notifier.send_sms(
                f"Position size too small for {ticker} - skipping"
            )
            return
 
        try:
            # Submit order to Alpaca
            if action == 'BUY':
                order = self.alpaca.submit_order(
                    symbol=ticker,
                    qty=shares,
                    side='buy',
                    type='market',
                    time_in_force='day'
                )
            else:  # SELL
                # Check if we have position to sell
                positions = self.alpaca.list_positions()
                position = next((p for p in positions if p.symbol == ticker), None)
 
                if not position:
                    await self.notifier.send_sms(
                        f"No position in {ticker} to sell"
                    )
                    return
 
                order = self.alpaca.submit_order(
                    symbol=ticker,
                    qty=shares,
                    side='sell',
                    type='market',
                    time_in_force='day'
                )
 
            # Success notification (one line - SMS, no markdown)
            await self.notifier.send_sms(
                f"Trade executed: {action} {shares} {ticker} @ ${current_price:.2f} "
                f"(${shares * current_price:.2f}, consensus {analysis['consensus_level']:.0%}, "
                f"order {order.id})"
            )
 
            # Log to database
            await self._log_trade(ticker, action, shares, current_price, analysis)
 
        except Exception as e:
            await self.notifier.send_sms(f"Trade failed for {ticker}: {str(e)}")
 
    async def mid_day_review(self):
        """
        12:00 PM: Check positions and send update
        """
        positions = self.alpaca.list_positions()
        account = self.alpaca.get_account()
 
        pl_today = float(account.equity) - float(account.last_equity)
        message = (
            f"Mid-Day Update: portfolio ${float(account.portfolio_value):.2f}, "
            f"P&L today ${pl_today:+.2f}. "
        )
 
        if positions:
            message += "Positions: " + ", ".join(
                f"{pos.symbol} {pos.qty}sh ({float(pos.unrealized_pl):+.2f})"
                for pos in positions
            )
        else:
            message += "No open positions"
 
        await self.notifier.send_sms(message)
 
    async def market_close_summary(self):
        """
        4:00 PM: Daily summary and performance report
        """
        account = self.alpaca.get_account()
        positions = self.alpaca.list_positions()
 
        # Calculate daily P&L
        portfolio_value = float(account.portfolio_value)
        daily_pl = float(account.equity) - float(account.last_equity)
        daily_pl_pct = (daily_pl / float(account.last_equity)) * 100
 
        # Get model leaderboard
        leaderboard = await self.orchestrator.get_leaderboard()
 
        message = (
            f"Market Close Summary: value ${portfolio_value:.2f}, "
            f"daily P&L ${daily_pl:+.2f} ({daily_pl_pct:+.2f}%). "
        )
 
        # Today's trades
        today_trades = await self._get_today_trades()
        message += f"Today's trades ({len(today_trades)}): " + ", ".join(
            f"{trade['action']} {trade['ticker']} {trade['shares']}sh @ ${trade['price']:.2f}"
            for trade in today_trades
        )
 
        message += f". Open positions ({len(positions)}): " + ", ".join(
            f"{pos.symbol} ${float(pos.unrealized_pl):+.2f}" for pos in positions
        )
 
        # Model leaderboard
        message += ". Top models: " + ", ".join(
            f"{model['model']} {model['total_return_pct']:+.2f}%"
            for model in leaderboard[:3]
        )
 
        await self.notifier.send_sms(message)
 
    def _format_analysis_result(self, ticker: str, aggregated: Dict) -> str:
        """Format analysis as a single SMS-friendly line (no markdown)"""
        votes = ", ".join(
            f"{r['model']}={r['decision']}"
            for r in aggregated['individual_decisions']
        )
        return (
            f"{ticker}: {aggregated['majority_decision']} "
            f"(agreement {aggregated['consensus_level']:.0%}, "
            f"confidence {aggregated['average_confidence']:.0%}). Votes: {votes}"
        )
3. SMS Notification System (Twilio)
# src/autonomous/sms_notifier.py
 
import asyncio
import os
from twilio.rest import Client
 
class SMSNotifier:
    """
    Send real-time SMS notifications to user via Twilio.
    Twilio's Python SDK is synchronous, so calls are run in a thread to fit
    the rest of this codebase's async style.
    """
 
    def __init__(self):
        self.client = Client(
            os.getenv('TWILIO_ACCOUNT_SID'),
            os.getenv('TWILIO_AUTH_TOKEN')
        )
        self.from_number = os.getenv('TWILIO_FROM_NUMBER')
        self.to_number = os.getenv('TWILIO_TO_NUMBER')
 
    async def send_sms(self, message: str):
        """Send a plain-text SMS to the user. No markdown - SMS doesn't render it.
        Twilio splits messages over ~160 chars into multiple segments and bills
        per segment, so keep messages short rather than porting Telegram-style
        multi-line formatting."""
        try:
            await asyncio.to_thread(
                self.client.messages.create,
                body=message,
                from_=self.from_number,
                to=self.to_number
            )
        except Exception as e:
            print(f"Failed to send SMS: {e}")
 
    async def send_chart(self, ticker: str, chart_url: str):
        """Send a chart as MMS. Unlike Telegram, Twilio MMS needs a publicly
        reachable URL for the image (not a local file path) - upload the chart
        to storage you control first and pass that URL in."""
        try:
            await asyncio.to_thread(
                self.client.messages.create,
                body=f"Chart for {ticker}",
                from_=self.from_number,
                to=self.to_number,
                media_url=[chart_url]
            )
        except Exception as e:
            print(f"Failed to send chart MMS: {e}")
 
    async def send_daily_digest(self, digest: Dict):
        """Send a single-message daily digest, condensed for SMS"""
        message = (
            f"Daily Digest: value ${digest['portfolio_value']:.2f}, "
            f"P&L ${digest['daily_pl']:+.2f} ({digest['daily_pl_pct']:+.2f}%), "
            f"total return {digest['total_return_pct']:+.2f}%. "
        )
        message += f"Trades ({digest['num_trades']}): " + ", ".join(
            f"{t['action']} {t['ticker']} {t['shares']}sh @ ${t['price']:.2f}"
            for t in digest['trades']
        )
        message += ". Top models: " + ", ".join(
            f"{m['rank']}.{m['model']} {m['return_pct']:+.2f}%"
            for m in digest['model_rankings']
        )

        await self.send_sms(message)
4. Main Autonomous Entry Point
# src/autonomous/main.py
 
import asyncio
from trading_loop import AutonomousTradingLoop
from sms_notifier import SMSNotifier
from src.orchestrator.model_manager import ModelOrchestrator
from alpaca.trading.client import TradingClient
import os
 
async def main():
    """
    Start autonomous trading system
    """
    print("🚀 Starting Autonomous Trading System...")
 
    # Initialize components
    sms = SMSNotifier()
    orchestrator = ModelOrchestrator()
 
    # Alpaca paper trading
    alpaca = TradingClient(
        api_key=os.getenv('ALPACA_API_KEY'),
        secret_key=os.getenv('ALPACA_SECRET_KEY'),
        paper=True  # PAPER TRADING ONLY
    )
 
    # Create trading loop
    trading_loop = AutonomousTradingLoop(
        sms_notifier=sms,
        models_orchestrator=orchestrator,
        alpaca_client=alpaca
    )
 
    # Send startup notification
    await sms.send_sms(
        "Autonomous Trading System Online. Will screen stocks, analyze with "
        "all AI models, and execute trades on consensus (Alpaca paper account "
        "only). You'll get an SMS for every action."
    )
 
    # Start the loop
    trading_loop.start()
 
    # Keep running
    print("✅ System running. Press Ctrl+C to stop.")
    try:
        while True:
            await asyncio.sleep(60)
    except KeyboardInterrupt:
        print("\n🛑 Shutting down...")
        await sms.send_sms("Trading System Stopped. Autonomous trading disabled.")
 
if __name__ == "__main__":
    asyncio.run(main())
Deployment for 24/7 Operation
Railway Configuration for Background Jobs
# Procfile (for Railway)
web: uvicorn src.main:app --host 0.0.0.0 --port $PORT
worker: python src/autonomous/main.py
Docker Setup
# Dockerfile
FROM python:3.11-slim
 
WORKDIR /app
 
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
 
COPY src/ ./src/
COPY .claude/ ./.claude/
 
# For autonomous mode
CMD ["python", "src/autonomous/main.py"]
Safety Features
1. Hard Limits (Always Enforced)
SAFETY_LIMITS = {
    "max_daily_trades": 20,           # Max 20 trades per day
    "max_position_size_pct": 15,      # Max 15% portfolio in one stock
    "max_daily_loss_pct": 5,          # Stop if lose >5% in one day
    "min_cash_reserve_pct": 20,       # Keep at least 20% cash
    "max_stock_price": 1000,          # No stocks >$1000 (avoid expensive)
    "min_volume": 1_000_000,          # Only liquid stocks
}
def check_safety_limits(trade: Dict, portfolio: Dict) -> bool:
    """
    Verify trade doesn't violate safety limits
    """
    # Check all limits...
    # Return False if any violated
    pass
2. Emergency Stop
SMS is one-way by default (we only send). Two-way commands (STOP/RESUME/STATUS
by text reply) require standing up a small webhook server that Twilio calls on
inbound SMS - this is more infrastructure than a Telegram bot needs, since
Telegram commands work out of the box. If two-way control matters, set up a
Twilio inbound-SMS webhook (e.g. a FastAPI route) per
https://www.twilio.com/docs/messaging/guides/webhook-request.
# Example inbound webhook handler (e.g. FastAPI route at /sms/inbound)
async def handle_inbound_sms(body: str):
    """Parse a keyword from an inbound SMS and act on it"""
    global TRADING_ENABLED
    command = body.strip().upper()

    if command == "STOP":
        TRADING_ENABLED = False
        return "EMERGENCY STOP ACTIVATED. Trading halted. Existing positions remain open. Text RESUME to restart."
    elif command == "RESUME":
        TRADING_ENABLED = True
        return "Trading resumed."
    elif command == "STATUS":
        return await get_status_summary()
    else:
        return "Unknown command. Try STOP, RESUME, or STATUS."
SMS Commands for Monitoring (requires the inbound webhook above)
STATUS   - Current portfolio and positions
TODAY    - Today's trading activity
MODELS   - Model leaderboard
STOP     - Emergency stop (halt all trading)
RESUME   - Resume autonomous trading
LIMITS   - View current safety limits
WATCHLIST - See today's selected stocks
Testing Autonomous Mode
1. Dry Run (No Real Trades)
DRY_RUN_MODE = True  # Set in .env
# All trade executions log but don't submit
if not DRY_RUN_MODE:
    order = alpaca.submit_order(...)
else:
    print(f"[DRY RUN] Would execute: {action} {shares} {ticker}")
2. Start Small
# First week: Only 1-2 stocks, small positions
AUTONOMOUS_CONFIG = {
    "max_stocks_per_day": 2,
    "position_size_pct": 0.02,  # Only 2% per position
    "require_high_consensus": True  # >80% agreement
}
Code References
AI-Trader BaseAgent: refs/AI-Trader/agent/base_agent/base_agent.py
AI-Trader Stock Universe: Lines 44-56 in base_agent.py
TradingAgents Graph: refs/TradingAgents/tradingagents/graph/trading_graph.py
Scheduling: Use APScheduler (similar to AI-Trader patterns)
Best Practices
Start with dry run - Test without real trades first
Monitor closely first week - Watch your SMS notifications
Use paper trading - NEVER use real money initially
Small positions - Start with 2-5% risk per trade
High consensus required - Only trade when >70% models agree
Keep cash reserve - Always maintain 20%+ cash
Daily limits - Cap number of trades per day
Emergency stop - Available via the inbound-SMS webhook (STOP keyword), if set up
Expected SMS Notifications (one text per line below, not multi-line messages)
Morning (9:00 AM)
"Pre-Market Screening Started. Analyzing NASDAQ 100 stocks..."
"Today's Watchlist: NVDA (+3.2%), AAPL (-2.1%)"
Market Open (9:30 AM)
"Market Open - analyzing 5 stocks..."
"Analyzing NVDA..."
"NVDA: BUY (agreement 80%, confidence 75%). Votes: Claude=BUY, GPT-5=BUY, Gemini=BUY, DeepSeek=BUY, Qwen=HOLD"
"Trade executed: BUY 10 NVDA @ $850.50 ($8,505.00, consensus 80%, order <id>)"
Throughout Day
"Skipping AAPL - low consensus (60%)"
"Mid-Day Update: portfolio $10,234.50, P&L today +$234.50. Positions: NVDA 10sh (+150.00)"
Market Close (4:00 PM)
"Market Close Summary: value $10,345.00, daily P&L +$345.00 (+3.45%). Today's trades (2): BUY NVDA 10sh @ $850.50, SELL AAPL 5sh @ $175.20. Open positions (3): NVDA +$175.00, MSFT -$25.00, GOOGL +$50.00. Top models: Claude +4.2%, GPT-5 +3.8%, Gemini +3.1%"
Usage Example
Setup (One Time):
# Deploy to Railway
railway up
 
# Set environment variables
AUTONOMOUS_MODE=true
DRY_RUN_MODE=false  # Set true for testing
TWILIO_ACCOUNT_SID=your_account_sid
TWILIO_AUTH_TOKEN=your_auth_token
TWILIO_FROM_NUMBER=+1XXXXXXXXXX
TWILIO_TO_NUMBER=+1XXXXXXXXXX
ALPACA_API_KEY=paper_key
ALPACA_SECRET_KEY=paper_secret
It Runs Automatically:
Every trading day, system wakes up at 9:00 AM
Screens stocks, analyzes, trades
You get an SMS for everything
No human input required
You Can Monitor:
Text the Twilio number (requires the inbound-SMS webhook from step 2 above):
STATUS  -> See current state
STOP    -> Halt trading immediately
RESUME  -> Resume if stopped