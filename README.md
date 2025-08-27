# randomkeygen.com
Random Key Generator
//+------------------------------------------------------------------+ //|                                                  TrendRiskEA.mq5 | //| A risk-managed trend EA template for MT5                         | //| NOTE: No strategy can guarantee profit. Use at your own risk.    | //+------------------------------------------------------------------+ #property copyright   "Public domain" #property link        "" #property version     "1.00" #property strict

//--- Inputs input group               "Risk Management"; input double   RiskPerTrade      = 1.0;     // % of balance risked per trade input double   MaxDrawdownStop   = 30.0;    // % equity drawdown to disable trading input double   ATR_Mult_SL       = 2.0;     // StopLoss = ATR * multiplier input double   ATR_Mult_TP       = 3.0;     // TakeProfit = ATR * multiplier input bool     UseTrailingATR    = true;    // Trail using ATR input double   ATR_Mult_Trail    = 1.5;     // ATR multiplier for trailing stop

input group               "Entries & Filters"; input int      FastMAPeriod      = 20; input int      SlowMAPeriod      = 50; input ENUM_MA_METHOD MaMethod    = MODE_EMA; input ENUM_APPLIED_PRICE MaPrice = PRICE_CLOSE; input int      ATRPeriod         = 14; input int      MaxSpreadPoints   = 30;      // Skip if spread wider than this input int      MinBarsBetween    = 5;       // Cooldown bars between trades input bool     TradeLong         = true; input bool     TradeShort        = true;

input group               "Session & Execution"; input bool     UseSessionFilter  = false; input int      SessionStartHour  = 7;       // broker time input int      SessionEndHour    = 20;      // broker time input ulong    Magic             = 20250827; input int      SlippagePoints    = 10;

//--- Globals MqlTick        tick; double         point; double         pip; int            digits; string         sym; datetime       lastTradeBarTime = 0;

//--- Indicator handles int            hFastMA = -1; int            hSlowMA = -1; int            hATR    = -1;

//+------------------------------------------------------------------+ //| Helper: Log                                                      | //+------------------------------------------------------------------+ void Log(string msg){ Print("[TrendRiskEA] ", msg); }

//+------------------------------------------------------------------+ //| OnInit                                                            | //+------------------------------------------------------------------+ int OnInit() { sym    = _Symbol; digits = (int)SymbolInfoInteger(sym, SYMBOL_DIGITS); point  = SymbolInfoDouble(sym, SYMBOL_POINT); pip    = (digits==3 || digits==5) ? 10*point : point;

// Create Indicators hFastMA = iMA(sym, PERIOD_CURRENT, FastMAPeriod, 0, MaMethod, MaPrice); hSlowMA = iMA(sym, PERIOD_CURRENT, SlowMAPeriod, 0, MaMethod, MaPrice); hATR    = iATR(sym, PERIOD_CURRENT, ATRPeriod);

if(hFastMA==INVALID_HANDLE || hSlowMA==INVALID_HANDLE || hATR==INVALID_HANDLE) { Log("Failed to create indicators"); return(INIT_FAILED); }

Log("Initialized on " + sym + ", digits=" + (string)digits); return(INIT_SUCCEEDED); }

//+------------------------------------------------------------------+ //| OnDeinit                                                          | //+------------------------------------------------------------------+ void OnDeinit(const int reason) { if(hFastMA!=-1) IndicatorRelease(hFastMA); if(hSlowMA!=-1) IndicatorRelease(hSlowMA); if(hATR!=-1)    IndicatorRelease(hATR); }

//+------------------------------------------------------------------+ //| Utility: Session/Spread filters                                   | //+------------------------------------------------------------------+ bool AllowedSession() { if(!UseSessionFilter) return(true); MqlDateTime t; TimeCurrent(t); if(SessionStartHour<=SessionEndHour) return(t.hour>=SessionStartHour && t.hour<SessionEndHour); // Overnight session (e.g., 22 -> 6) return(t.hour>=SessionStartHour || t.hour<SessionEndHour); }

bool AllowedSpread() { if(!SymbolInfoTick(sym, tick)) return(false); int spreadPts = (int)MathRound((tick.ask - tick.bid)/point); return(spreadPts <= MaxSpreadPoints); }

bool CooldownOk() { MqlRates rates[]; int copied = CopyRates(sym, PERIOD_CURRENT, 0, 3, rates); if(copied<3) return(true); datetime currentBar = rates[0].time; // current forming bar if(lastTradeBarTime==0) return(true); // Ensure MinBarsBetween closed bars passed int barsPassed = (int)((currentBar - lastTradeBarTime) / (PeriodSeconds(PERIOD_CURRENT))); return(barsPassed >= MinBarsBetween); }

//+------------------------------------------------------------------+ //| Utility: Position handling                                        | //+------------------------------------------------------------------+ long GetOpenPositionType() { // returns POSITION_TYPE_BUY, POSITION_TYPE_SELL, or -1 if none for(int i=0;i<PositionsTotal();i++) { ulong ticket = PositionGetTicket(i); if(PositionSelectByTicket(ticket)) { if(PositionGetString(POSITION_SYMBOL)==sym && (ulong)PositionGetInteger(POSITION_MAGIC)==Magic) return((long)PositionGetInteger(POSITION_TYPE)); } } return(-1); }

bool ClosePosition(long posType) { if(!SymbolInfoTick(sym, tick)) return(false);

double volume=0.0; ulong position_ticket=0; double price=0.0;

for(int i=0;i<PositionsTotal();i++) { ulong ticket = PositionGetTicket(i); if(PositionSelectByTicket(ticket)) { if(PositionGetString(POSITION_SYMBOL)==sym && (ulong)PositionGetInteger(POSITION_MAGIC)==Magic && (long)PositionGetInteger(POSITION_TYPE)==posType) { position_ticket = ticket; volume = PositionGetDouble(POSITION_VOLUME); break; } } }

if(position_ticket==0) return(false);

if(posType==POSITION_TYPE_BUY) price = tick.bid; else price = tick.ask;

MqlTradeRequest req; MqlTradeResult res; ZeroMemory(req); ZeroMemory(res); req.action   = TRADE_ACTION_DEAL; req.symbol   = sym; req.magic    = Magic; req.deviation= SlippagePoints; req.type     = (posType==POSITION_TYPE_BUY) ? ORDER_TYPE_SELL : ORDER_TYPE_BUY; req.volume   = volume; req.price    = price;

bool ok = OrderSend(req, res); if(!ok) Log("ClosePosition failed: " + (string)res.retcode); return(ok); }

//+------------------------------------------------------------------+ //| Utility: Risk-based lot sizing                                    | //+------------------------------------------------------------------+ double GetATR() { double atr[]; if(CopyBuffer(hATR,0,0,2,atr)<2) return(0.0); return(atr[0]); }

double NormalizeVolume(double lots) { double minLot = SymbolInfoDouble(sym, SYMBOL_VOLUME_MIN); double maxLot = SymbolInfoDouble(sym, SYMBOL_VOLUME_MAX); double step   = SymbolInfoDouble(sym, SYMBOL_VOLUME_STEP); lots = MathMax(minLot, MathMin(maxLot, lots)); // round to step double steps = MathFloor((lots - minLot + 1e-12)/step + 0.5); return(NormalizeDouble(minLot + steps*step, 2)); }

double LotsByRisk(double stop_points) { if(stop_points<=0) return(SymbolInfoDouble(sym, SYMBOL_VOLUME_MIN));

double balance    = AccountInfoDouble(ACCOUNT_BALANCE); double risk_money = balance * (RiskPerTrade/100.0);

double tick_value = SymbolInfoDouble(sym, SYMBOL_TRADE_TICK_VALUE); double tick_size  = SymbolInfoDouble(sym, SYMBOL_TRADE_TICK_SIZE); double contract_size = SymbolInfoDouble(sym, SYMBOL_TRADE_CONTRACT_SIZE);

// Convert stop in points to money per 1 lot // money_per_lot = (stop_points * point / tick_size) * tick_value * (contract_size/contract_size) double money_per_lot = (stop_points * point / tick_size) * tick_value; if(money_per_lot<=0) return(SymbolInfoDouble(sym, SYMBOL_VOLUME_MIN));

double lots = risk_money / money_per_lot; return(NormalizeVolume(lots)); }

//+------------------------------------------------------------------+ //| Entry conditions                                                  | //+------------------------------------------------------------------+ bool GetMAValues(double &fast, double &slow) { double f[], s[]; if(CopyBuffer(hFastMA,0,0,3,f)<3) return(false); if(CopyBuffer(hSlowMA,0,0,3,s)<3) return(false); fast = f[0]; slow = s[0]; return(true); }

int CrossDirection() { // returns +1 for bullish cross (fast over slow), -1 for bearish, 0 otherwise double f[3], s[3]; if(CopyBuffer(hFastMA,0,0,3,f)<3) return(0); if(CopyBuffer(hSlowMA,0,0,3,s)<3) return(0); bool wasBelow = f[1] < s[1]; bool nowAbove = f[0] > s[0]; bool wasAbove = f[1] > s[1]; bool nowBelow = f[0] < s[0]; if(wasBelow && nowAbove) return(+1); if(wasAbove && nowBelow) return(-1); return(0); }

//+------------------------------------------------------------------+ //| Order placement                                                   | //+------------------------------------------------------------------+ bool OpenTrade(int direction) { if(!SymbolInfoTick(sym, tick)) return(false);

double atr = GetATR(); if(atr<=0) return(false);

double sl_points = (ATR_Mult_SL * atr)/point; double tp_points = (ATR_Mult_TP * atr)/point;

double lots = LotsByRisk(sl_points); if(lots <= 0) return(false);

double price, sl, tp; ENUM_ORDER_TYPE type;

if(direction>0) { type  = ORDER_TYPE_BUY; price = tick.ask; sl    = price - sl_pointspoint; tp    = price + tp_pointspoint; } else { type  = ORDER_TYPE_SELL; price = tick.bid; sl    = price + sl_pointspoint; tp    = price - tp_pointspoint; }

MqlTradeRequest req; MqlTradeResult res; ZeroMemory(req); ZeroMemory(res); req.action   = TRADE_ACTION_DEAL; req.symbol   = sym; req.magic    = Magic; req.type     = type; req.volume   = lots; req.deviation= SlippagePoints; req.price    = price; req.sl       = sl; req.tp       = tp;

bool ok = OrderSend(req, res); if(ok) { Log(StringFormat("Opened %s %.2f lots @ %.5f | SL %.5f TP %.5f", (direction>0?"BUY":"SELL"), lots, price, sl, tp)); MqlRates r[]; if(CopyRates(sym, PERIOD_CURRENT, 0, 2, r)>=2) lastTradeBarTime = r[1].time; // last closed bar } else { Log("OrderSend failed: retcode=" + (string)res.retcode); } return(ok); }

//+------------------------------------------------------------------+ //| Trailing stop                                                     | //+------------------------------------------------------------------+ void TrailPositions() { if(!UseTrailingATR) return; double atr = GetATR(); if(atr<=0) return;

for(int i=0;i<PositionsTotal();i++) { ulong ticket = PositionGetTicket(i); if(!PositionSelectByTicket(ticket)) continue; if(PositionGetString(POSITION_SYMBOL)!=sym) continue; if((ulong)PositionGetInteger(POSITION_MAGIC)!=Magic) continue;

long   type   = (long)PositionGetInteger(POSITION_TYPE);
  double price  = (type==POSITION_TYPE_BUY) ? tick.bid : tick.ask;
  double posSL  = PositionGetDouble(POSITION_SL);
  double open   = PositionGetDouble(POSITION_PRICE_OPEN);
  double trail  = ATR_Mult_Trail * atr;

  double newSL;
  if(type==POSITION_TYPE_BUY)
  {
     newSL = price - trail;
     // never reduce SL below open (optional break-even lock)
     newSL = MathMax(newSL, open);
     if(newSL > posSL + point)
     {
        MqlTradeRequest req; MqlTradeResult res; ZeroMemory(req); ZeroMemory(res);
        req.action = TRADE_ACTION_SLTP;
        req.symbol = sym;
        req.magic  = Magic;
        req.sl     = newSL;
        req.tp     = PositionGetDouble(POSITION_TP);
        req.position = ticket;
        if(!OrderSend(req,res)) Log("Trail BUY failed: " + (string)res.retcode);
     }
  }
  else if(type==POSITION_TYPE_SELL)
  {
     newSL = price + trail;
     newSL = MathMin(newSL, open);
     if(newSL < posSL - point || posSL==0.0)
     {
        MqlTradeRequest req; MqlTradeResult res; ZeroMemory(req); ZeroMemory(res);
        req.action = TRADE_ACTION_SLTP;
        req.symbol = sym;
        req.magic  = Magic;
        req.sl     = newSL;
        req.tp     = PositionGetDouble(POSITION_TP);
        req.position = ticket;
        if(!OrderSend(req,res)) Log("Trail SELL failed: " + (string)res.retcode);
     }
  }

} }

//+------------------------------------------------------------------+ //| Equity protection                                                 | //+------------------------------------------------------------------+ bool EquityOk() { double bal = AccountInfoDouble(ACCOUNT_BALANCE); double eq  = AccountInfoDouble(ACCOUNT_EQUITY); double ddPerc = 100.0 * (bal - eq) / MathMax(bal, 0.01); return(ddPerc <= MaxDrawdownStop); }

//+------------------------------------------------------------------+ //| OnTick                                                            | //+------------------------------------------------------------------+ void OnTick() { if(!SymbolInfoTick(sym, tick)) return; if(!AllowedSession()) return; if(!AllowedSpread())  return; if(!EquityOk())       return;

TrailPositions();

// Only one position at a time per symbol for this EA long existing = GetOpenPositionType();

// Entry signals on MA crossover int cross = CrossDirection();

// Cooldown if(!CooldownOk()) return;

if(cross>0 && TradeLong) { // If short is open, close it; else open long if(existing==POSITION_TYPE_SELL) ClosePosition(POSITION_TYPE_SELL); if(existing==-1) OpenTrade(+1); } else if(cross<0 && TradeShort) { if(existing==POSITION_TYPE_BUY) ClosePosition(POSITION_TYPE_BUY); if(existing==-1) OpenTrade(-1); } }

//+------------------------------------------------------------------+ //| Optimization tips (read as comments)                              | //| - Optimize FastMA, SlowMA, ATR multipliers, and session on each  | //|   symbol & timeframe.                                            | //| - Test on multiple years with variable spread and slippage.      | //| - Consider adding trend filter (e.g., higher TF MA alignment).   | //| - Add news filter via custom indicator if needed.                | //+------------------------------------------------------------------+

