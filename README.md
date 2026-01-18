# 🦅 Valkyrie Trading Signal System

**Version:** 2.1 (Production Release)  
**Platform:** n8n (Self-Hosted)  
**Strategy:** Multi-Timeframe Mean Reversion with Momentum Confirmation

---

## ⚠️ CRITICAL DISCLAIMERS

**THIS IS NOT A TRADING BOT. THIS IS A SIGNAL GENERATOR.**

- **Does NOT execute trades automatically**
- **Does NOT connect to exchange trading APIs**
- **Does NOT manage your portfolio**
- **Does NOT guarantee profits or prevent losses**

**LEGAL NOTICE:**
- Trading cryptocurrencies involves substantial risk of loss
- Past performance does not guarantee future results
- This software is provided "AS IS" without warranties of any kind
- You are solely responsible for all trading decisions
- The creator assumes no liability for financial losses
- Must be 18+ to use this software
- Check your local regulations regarding automated trading tools

**By using this software, you acknowledge you have read and understood these risks.**

---

## 📊 What This System Does

Valkyrie monitors cryptocurrency markets and generates **manual trading signals** based on strict technical criteria:

1. **4-Hour Trend Analysis**: Identifies assets in confirmed uptrends
2. **Pullback Detection**: Waits for healthy corrections to moving averages
3. **15-Minute Confirmation**: Prevents buying exhausted moves (FOMO protection)
4. **Automatic Circuit Breaker**: Stops signaling during poor market conditions

**You still need to:**
- Review each signal manually
- Decide position sizing
- Execute trades yourself
- Manage exits and risk

---

## 🎯 Who This Is For

**✅ Good Fit:**
- Semi-technical traders comfortable with command line
- Currently trading Kraken USD pairs manually
- Want to remove emotional decision-making
- Willing to spend 4-6 hours on initial setup
- Have experience with Docker/Linux basics
- Looking for signal assistance, not full automation

**❌ Not a Good Fit:**
- Complete beginners to trading or Linux
- Expecting "get rich quick" results
- Want fully automated trading
- Trading on exchanges other than Kraken (currently)
- Day traders (this uses 4H timeframe)

---

## 🔧 System Requirements

### Technical Prerequisites
- Linux server or VPS (Ubuntu 20.04+ recommended)
- Docker & Docker Compose installed
- 2GB RAM minimum, 4GB recommended
- 10GB free disk space
- Basic command-line knowledge

### Required Accounts
- Kraken account (no API keys needed - read-only public data)
- Discord account (optional, for alerts)
- n8n instance (included in setup)

### Time Investment
- Initial Setup: 4-6 hours
- Daily Monitoring: 10-15 minutes
- Weekly Maintenance: 30 minutes

---

## 🚀 Quick Start Installation

### Step 1: Clone Repository

```bash
git clone https://github.com/yourusername/valkyrie-trading.git
cd valkyrie-trading
```

### Step 2: Launch with Docker Compose

```bash
# Start n8n and create log directory
docker-compose up -d

# Verify services are running
docker-compose ps
```

### Step 3: Initialize Log Files

```bash
# Create log directory with proper permissions
sudo mkdir -p /opt/valkyrie/logs
sudo chown -R 1000:1000 /opt/valkyrie/logs
sudo chmod -R 700 /opt/valkyrie/logs

# Initialize state file
echo '{"state":"ACTIVE","last_change_ts":0}' | sudo tee /opt/valkyrie/logs/ibdl_state.json

# Initialize event log
echo 'event_ts,event_type,signal_id,pair,entry_price,signal,mae_pct,mfe_pct,bars_to_ma50,hit_ma50,structure_hold,fail,state' | sudo tee /opt/valkyrie/logs/mr_events.csv
```

### Step 4: Import Workflows

1. Open n8n at `http://localhost:5678`
2. Go to **Workflows** → **Import from File**
3. Import `workflows/ibdl_scanner_mercenary.json`
4. Import `workflows/ibdl_outcome_manager.json`
5. Activate both workflows

### Step 5: Configure Alerts (Optional)

```bash
# Set Discord webhook in n8n environment
# Go to Settings → Environment Variables
# Add: DISCORD_WEBHOOK_URL = your_webhook_here
```

### Step 6: Verify Installation

```bash
# Check that state file is readable
cat /opt/valkyrie/logs/ibdl_state.json

# Monitor the event log
tail -f /opt/valkyrie/logs/mr_events.csv
```

---

## 📖 Strategy Explanation

### The Four Gates Every Signal Must Pass

**Gate 1: Trend Permission (4H Chart)**
- Price must be above 200-period SMA (uptrend confirmation)
- 20 SMA must be above 50 SMA (momentum alignment)
- **Why**: Never trade against the dominant trend

**Gate 2: Pullback Detection (4H Chart)**
- Price must have touched 20 or 50 SMA within last 48 hours
- **Why**: Only buy after healthy corrections, not at tops

**Gate 3: Reclaim Signal (4H Chart)**
- Current close must be above 20 SMA
- Previous bar must have been below/touching 20 SMA
- Volume must be 1.1x average
- RSI between 35-55 (not oversold or overbought)
- **Why**: Confirms the bounce is starting with conviction

**Gate 4: 15-Minute Sniper Filter**
- 15M RSI must be between 45-62
- **Why**: Prevents buying when short-term momentum is exhausted (>62) or dying (<45)

### The Circuit Breaker

The system tracks its own performance:
- **Win Condition**: Price hits 50 SMA within 12 bars (48 hours)
- **Loss Condition**: Price drops >3% before hitting target

**Automatic State Management:**
- If 12+ of last 20 signals fail → Enter **STAND_DOWN** mode
- In STAND_DOWN: Scanning continues but NO alerts sent
- Exits STAND_DOWN after 3 consecutive wins
- **Why**: Protects you during choppy/bearish market conditions

---

## 🎛️ Configuration

### Customizing the Watchlist

Edit the scanner workflow, find the "Build USD Universe" node:

```javascript
// These pairs are ALWAYS scanned regardless of volume
const WHITELIST = [
  'XMR/USD', 
  'SOL/USD', 
  'LTC/USD', 
  'XRP/USD', 
  'ETH/USD', 
  'BTC/USD'
];
```

**Recommendation**: Start with the default list, add your favorites after 2-4 weeks.

### Adjusting Risk Parameters

In "Compute 4H Signal" node:

```javascript
const RSI_MIN = 35;      // Lower = more aggressive entries
const RSI_MAX = 55;      // Higher = risk more FOMO buys
const VOL_MULT = 1.10;   // Volume multiplier (1.10 = 10% above avg)
```

In "Validate 15M" node:

```javascript
const rsiSafe = (currentRsi >= 45 && currentRsi <= 62);
// Tighten to 48-58 for more conservative entries
```

### Circuit Breaker Sensitivity

In outcome manager "Update State" node (Python section):

```python
# Current: 12 fails out of 20 = STAND_DOWN
if len(last20) >= 20 and fails >= 12: new = 'STAND_DOWN'

# More aggressive (stop after 8 fails):
# if len(last20) >= 20 and fails >= 8: new = 'STAND_DOWN'
```

---

## 📊 Monitoring & Logs

### Check Current System State

```bash
cat /opt/valkyrie/logs/ibdl_state.json
# Output: {"state":"ACTIVE","last_change_ts":1736889600}
```

### View Recent Signals

```bash
tail -n 20 /opt/valkyrie/logs/mr_events.csv
```

### Filter by Signal Type

```bash
# Show only new signals
grep "SIGNAL" /opt/valkyrie/logs/mr_events.csv | tail -n 10

# Show only outcomes
grep "OUTCOME" /opt/valkyrie/logs/mr_events.csv | tail -n 10
```

### Calculate Win Rate

```bash
# Quick win rate calculation
awk -F, '$2=="OUTCOME" {total++; if($12=="0") wins++} END {print "Win Rate:", (wins/total)*100"%"}' /opt/valkyrie/logs/mr_events.csv
```

---

## 🔍 Interpreting Signals

When you receive a Discord alert:

```
🚀 VALKYRIE SIGNAL (Confirmed)
Pair: SOL/USD
Price: 142.50
4H Trend: UP
15M RSI: 52.3
SignalId: SOL_USD_1736889123
```

**What to Check Before Trading:**

1. **Open TradingView** and verify the 4H chart looks clean
2. **Check 15M chart** - is there room to run or near resistance?
3. **Review order book** on Kraken - sufficient liquidity?
4. **Calculate position size** based on your risk tolerance
5. **Set stop loss** at -3% to match the system's loss definition

**Target Exit:**
- System considers it a "win" if price hits 50 SMA on 4H chart
- You can use this as a profit target or trail stops manually

---

## 🐛 Troubleshooting

### No Signals Generating

**Check 1: Are workflows active?**
```bash
# In n8n UI, verify both workflows show "Active" status
```

**Check 2: Is the system in STAND_DOWN?**
```bash
cat /opt/valkyrie/logs/ibdl_state.json
# If state is "STAND_DOWN", the circuit breaker is active
```

**Check 3: Check n8n logs**
```bash
docker-compose logs -f n8n
# Look for API errors or rate limiting
```

### Permission Errors

```bash
# Fix log directory permissions
sudo chown -R 1000:1000 /opt/valkyrie/logs
sudo chmod -R 700 /opt/valkyrie/logs
```

### Discord Webhooks Not Working

1. Verify webhook URL in n8n environment variables
2. Test webhook manually:
```bash
curl -X POST "YOUR_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{"content":"Test message"}'
```

### Workflows Timing Out

```bash
# Increase timeout in workflow settings
# Go to workflow → Settings → Execution Timeout → 600
```

---

## 📈 Performance Tracking

### Manual Backtesting Process

1. Export historical signals:
```bash
grep "SIGNAL" /opt/valkyrie/logs/mr_events.csv > historical_signals.csv
```

2. For each signal, record in spreadsheet:
   - Entry price (from log)
   - Exit price (manual or from outcome)
   - Win/Loss
   - % Return

3. Calculate metrics:
   - Win Rate = Wins / Total Trades
   - Average Win = Sum(Winning %s) / Number of Wins
   - Average Loss = Sum(Losing %s) / Number of Losses
   - Expectancy = (Win% × AvgWin) - (Loss% × AvgLoss)

### Recommended Metrics to Track

- **Win Rate Target**: 55-65% (circuit breaker activates below 40%)
- **Risk/Reward**: Aim for 1.5:1 minimum (1.5% avg win vs 1% avg loss)
- **Max Consecutive Losses**: Should not exceed 5
- **Signals Per Week**: Expect 2-8 depending on market conditions

---

## 🔐 Security Best Practices

### Log File Protection

```bash
# Never use 777 permissions
# Correct permissions (owner-only):
sudo chmod 700 /opt/valkyrie/logs
sudo chmod 600 /opt/valkyrie/logs/*.csv
sudo chmod 600 /opt/valkyrie/logs/*.json
```

### n8n Security

```bash
# Set basic auth in docker-compose.yml
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=yourusername
N8N_BASIC_AUTH_PASSWORD=strongpassword
```

### API Rate Limiting

The scanner includes 1-second delays between API calls. If you see rate limit errors:

1. Reduce the TOP_N value (currently 40 pairs)
2. Increase wait time in "Wait 1s" node to 2 seconds
3. Check Kraken API status: https://status.kraken.com

---

## 🆘 Support & Community

### Getting Help

1. **Check the FAQ** (below) first
2. **Review n8n logs** for error messages
3. **Post in Discord** (link in purchase confirmation)
4. **Email support** (response within 48 hours)

### FAQ

**Q: Can I run this on Windows?**  
A: Not recommended. Use WSL2 or a Linux VPS.

**Q: Does this work with Binance/Coinbase?**  
A: Not yet. Kraken only in v2.1. Other exchanges in roadmap.

**Q: Can I backtest before going live?**  
A: Manual backtesting process documented above. Automated backtesting coming in v3.0.

**Q: How much capital do I need?**  
A: Minimum $500-1000 to properly size positions. System doesn't manage capital allocation.

**Q: What if I miss a signal?**  
A: Signals are logged. You can review missed signals and manually enter if setup still valid.

**Q: Can I run multiple strategies?**  
A: Yes, duplicate the workflows and modify parameters. Use different log directories.

---

## 📅 Changelog

### v2.1 (Current - Production Release)
- ✅ Fixed security vulnerabilities (chmod 777 → 700)
- ✅ Added Docker Compose for easy deployment
- ✅ Improved error handling and logging
- ✅ Added 15M RSI confirmation filter
- ✅ Enhanced documentation and setup guide

### v2.0 (Mercenary Edition)
- Multi-timeframe confirmation (4H + 15M)
- Automatic circuit breaker (STAND_DOWN state)
- Forced watchlist scanning
- File-based state management

---

## 🗺️ Roadmap

### v2.5 (Next 3 Months)
- [ ] PostgreSQL option (replace CSV)
- [ ] Web dashboard for viewing signals
- [ ] Telegram integration
- [ ] Binance support
- [ ] Automated backtesting module

### v3.0 (6-12 Months)
- [ ] Multiple strategy templates
- [ ] Position sizing calculator
- [ ] Mobile app for alerts
- [ ] Paper trading mode

---

## 📄 License

MIT License - See LICENSE file for details

**Commercial Use**: Permitted  
**Modification**: Permitted  
**Distribution**: Permitted with attribution  
**Warranty**: NONE - Use at your own risk

---

## 🙏 Credits

Created by [Your Name]  
Strategy based on mean reversion principles and multi-timeframe analysis  
Built with n8n workflow automation

---

**Remember**: This system generates signals. YOU make the trading decisions. Always use proper risk management, never risk more than you can afford to lose, and thoroughly test before committing significant capital.

**Good luck, and trade safely! 🦅**
