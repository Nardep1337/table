//@version=5
indicator("Multi-TF Checklist (RSI + StochRSI + Momentum + SuperTrend + BBVol + CMF + Hurst)", overlay=true, max_lines_count=500, max_labels_count=500, dynamic_requests=true, calc_on_every_tick=true)

// === Inputs ===
rsiLength    = input.int(14, "RSI Length")
stochRSILen  = input.int(14, "StochRSI Stoch Length")
stochK       = input.int(3, "StochRSI K")
momLength    = input.int(10, "Momentum Length")
superPeriod  = input.int(10, "SuperTrend ATR Period")
superMult    = input.float(3.0, "SuperTrend Multiplier")
bbLength     = input.int(20, "BB Length (Volatility)")
bbMult       = input.float(2.0, "BB StdDev")
cmfLength    = input.int(20, "CMF Length")
hurstLen     = input.int(200, "Hurst Lookback", minval=50)
table_pos    = input.string("Top Right", "Table Position", options=["Top Right", "Top Left", "Bottom Right", "Bottom Left"])

// Signals on chart
showSignals  = input.bool(true, "Show Chart Signals")
minAgree     = input.int(3, "Min Timeframes Agree", minval=1, maxval=5)
useTF1       = input.bool(true, "Use 1M")
useTF5       = input.bool(true, "Use 5M")
useTF15      = input.bool(true, "Use 15M")
useTF60      = input.bool(true, "Use 1H")
useTF240     = input.bool(true, "Use 4H")

// === Functions ===
// StochRSI %K
f_stochrsi() =>
    rsi_val = ta.rsi(close, rsiLength)
    fast = ta.stoch(rsi_val, ta.highest(rsi_val, stochRSILen), ta.lowest(rsi_val, stochRSILen), stochRSILen)
    ta.sma(fast, stochK)

// SuperTrend direction (1 for up, -1 for down)
f_supertrend() =>
    atr = ta.atr(superPeriod)
    src = hl2
    up = src - superMult * atr
    dn = src + superMult * atr
    var float trendUp = na
    var float trendDown = na
    var int trend = 1
    trendUp := na(trendUp) ? up : close[1] > trendUp[1] ? math.max(up, trendUp[1]) : up
    trendDown := na(trendDown) ? dn : close[1] < trendDown[1] ? math.min(dn, trendDown[1]) : dn
    trend := close > trendDown[1] ? 1 : close < trendUp[1] ? -1 : trend[1]
    trend

// Bollinger Band Width in %
f_bbwidth() =>
    [upper, basis, lower] = ta.bb(close, bbLength, bbMult)
    (upper - lower) / basis * 100

// High Volatility (BBWidth > SMA(BBWidth))
f_highvol() =>
    width = f_bbwidth()
    sma_width = ta.sma(width, bbLength)
    width > sma_width

// Hurst
f_hurst(_len) =>
    _sumAbs = 0.0
    for j = 1 to _len
        _sumAbs += math.abs(close[j] - close[j-1])
    _disp = math.abs(close - close[_len])
    _fd   = (_disp > 0 and _len > 1 and _sumAbs > 0) ? (1.0 + (math.log(_sumAbs / _disp) / math.log(_len))) : na
    _hRaw = na(_fd) ? na : (2.0 - _fd)
    na(_hRaw) ? na : math.max(0.0, math.min(1.0, _hRaw))

// CMF
f_cmf(length) =>
    mfv = ((close - low) - (high - close)) / (high - low) * volume
    ta.sma(mfv, length) / ta.sma(volume, length)

// === Table ===
pos = table_pos == "Top Right" ? position.top_right : table_pos == "Top Left" ? position.top_left : table_pos == "Bottom Right" ? position.bottom_right : position.bottom_left
var table t = table.new(pos, 10, 6, border_width=1, frame_color=color.gray)

// Header
table.cell(t, 0, 0, "TF", text_color=color.white)
table.cell(t, 1, 0, "RSI", text_color=color.white)
table.cell(t, 2, 0, "StochRSI", text_color=color.white)
table.cell(t, 3, 0, "Momentum", text_color=color.white)
table.cell(t, 4, 0, "SuperTrend", text_color=color.white)
table.cell(t, 5, 0, "BBVol", text_color=color.white)
table.cell(t, 6, 0, "CMF", text_color=color.white)
table.cell(t, 7, 0, "Hurst", text_color=color.white)
table.cell(t, 8, 0, "Score", text_color=color.white)
table.cell(t, 9, 0, "Signal", text_color=color.white)

f_draw_row(tf_label, rsi, stoch, mom, supert, highvol, cmf, hurst, score, signal, row) =>
    table.cell(t, 0, row, tf_label, text_color=color.yellow)
    table.cell(t, 1, row, str.tostring(rsi, "#.0"), text_color = rsi < 40 ? color.green : rsi > 60 ? color.red : color.gray)
    table.cell(t, 2, row, str.tostring(stoch, "#.0"), text_color = stoch < 20 ? color.green : stoch > 80 ? color.red : color.gray)
    table.cell(t, 3, row, str.tostring(mom, "#.0"), text_color = mom > 100 ? color.green : mom < 100 ? color.red : color.gray)
    table.cell(t, 4, row, supert == 1 ? "Up" : "Down", text_color = supert == 1 ? color.green : color.red)
    table.cell(t, 5, row, highvol ? "High" : "Low", text_color = highvol ? color.green : color.gray)
    table.cell(t, 6, row, str.tostring(cmf, "#.##"), text_color = cmf > 0 ? color.green : cmf < 0 ? color.red : color.gray)
    table.cell(t, 7, row, na(hurst) ? "na" : str.tostring(hurst, "#.00"),text_color = na(hurst) ? color.gray : hurst > 0.55 ? color.green : hurst < 0.45 ? color.blue : color.orange)
    table.cell(t, 8, row, str.tostring(score, "#"), text_color = score > 0 ? color.green : score < 0 ? color.red : color.gray)
    table.cell(t, 9, row, signal,text_color = signal == "BUY" ? color.green : signal == "SELL" ? color.red : color.orange,bgcolor    = signal == "BUY" ? color.new(color.green, 85) : signal == "SELL" ? color.new(color.red, 85) : color.new(color.orange, 85))

// === MTF signals for chart plotting ===
// Compute per-timeframe signals as series so we can trigger shapes on consensus
var string _tf1 = "1"
var string _tf5 = "5"
var string _tf15 = "15"
var string _tf60 = "60"
var string _tf240 = "240"

rsi1     = request.security(syminfo.tickerid, _tf1, ta.rsi(close, rsiLength), barmerge.gaps_off, barmerge.lookahead_on)
stoch1   = request.security(syminfo.tickerid, _tf1, f_stochrsi(), barmerge.gaps_off, barmerge.lookahead_on)
mom1     = request.security(syminfo.tickerid, _tf1, close / close[momLength] * 100.0, barmerge.gaps_off, barmerge.lookahead_on)
super1   = request.security(syminfo.tickerid, _tf1, f_supertrend(), barmerge.gaps_off, barmerge.lookahead_on)
highvol1 = request.security(syminfo.tickerid, _tf1, f_highvol(), barmerge.gaps_off, barmerge.lookahead_on)
cmf1     = request.security(syminfo.tickerid, _tf1, f_cmf(cmfLength), barmerge.gaps_off, barmerge.lookahead_on)
hurst1   = request.security(syminfo.tickerid, _tf1, f_hurst(hurstLen), barmerge.gaps_off, barmerge.lookahead_on)
score1   = (rsi1 < 40 ? 1 : rsi1 > 60 ? -1 : 0) + (stoch1 < 20 ? 1 : stoch1 > 80 ? -1 : 0) + (mom1 > 100 ? 1 : mom1 < 100 ? -1 : 0) + (super1 == 1 ? 1 : -1) + (cmf1 > 0 ? 1 : cmf1 < 0 ? -1 : 0) + (hurst1 > 0.55 ? 1 : hurst1 < 0.45 ? -1 : 0)
signal1  = score1 > 2 and highvol1 ? "BUY" : score1 < -2 and highvol1 ? "SELL" : "WAIT"

rsi5     = request.security(syminfo.tickerid, _tf5, ta.rsi(close, rsiLength), barmerge.gaps_off, barmerge.lookahead_on)
stoch5   = request.security(syminfo.tickerid, _tf5, f_stochrsi(), barmerge.gaps_off, barmerge.lookahead_on)
mom5     = request.security(syminfo.tickerid, _tf5, close / close[momLength] * 100.0, barmerge.gaps_off, barmerge.lookahead_on)
super5   = request.security(syminfo.tickerid, _tf5, f_supertrend(), barmerge.gaps_off, barmerge.lookahead_on)
highvol5 = request.security(syminfo.tickerid, _tf5, f_highvol(), barmerge.gaps_off, barmerge.lookahead_on)
cmf5     = request.security(syminfo.tickerid, _tf5, f_cmf(cmfLength), barmerge.gaps_off, barmerge.lookahead_on)
hurst5   = request.security(syminfo.tickerid, _tf5, f_hurst(hurstLen), barmerge.gaps_off, barmerge.lookahead_on)
score5   = (rsi5 < 40 ? 1 : rsi5 > 60 ? -1 : 0) + (stoch5 < 20 ? 1 : stoch5 > 80 ? -1 : 0) + (mom5 > 100 ? 1 : mom5 < 100 ? -1 : 0) + (super5 == 1 ? 1 : -1) + (cmf5 > 0 ? 1 : cmf5 < 0 ? -1 : 0) + (hurst5 > 0.55 ? 1 : hurst5 < 0.45 ? -1 : 0)
signal5  = score5 > 2 and highvol5 ? "BUY" : score5 < -2 and highvol5 ? "SELL" : "WAIT"

rsi15     = request.security(syminfo.tickerid, _tf15, ta.rsi(close, rsiLength), barmerge.gaps_off, barmerge.lookahead_on)
stoch15   = request.security(syminfo.tickerid, _tf15, f_stochrsi(), barmerge.gaps_off, barmerge.lookahead_on)
mom15     = request.security(syminfo.tickerid, _tf15, close / close[momLength] * 100.0, barmerge.gaps_off, barmerge.lookahead_on)
super15   = request.security(syminfo.tickerid, _tf15, f_supertrend(), barmerge.gaps_off, barmerge.lookahead_on)
highvol15 = request.security(syminfo.tickerid, _tf15, f_highvol(), barmerge.gaps_off, barmerge.lookahead_on)
cmf15     = request.security(syminfo.tickerid, _tf15, f_cmf(cmfLength), barmerge.gaps_off, barmerge.lookahead_on)
hurst15   = request.security(syminfo.tickerid, _tf15, f_hurst(hurstLen), barmerge.gaps_off, barmerge.lookahead_on)
score15   = (rsi15 < 40 ? 1 : rsi15 > 60 ? -1 : 0) + (stoch15 < 20 ? 1 : stoch15 > 80 ? -1 : 0) + (mom15 > 100 ? 1 : mom15 < 100 ? -1 : 0) + (super15 == 1 ? 1 : -1) + (cmf15 > 0 ? 1 : cmf15 < 0 ? -1 : 0) + (hurst15 > 0.55 ? 1 : hurst15 < 0.45 ? -1 : 0)
signal15  = score15 > 2 and highvol15 ? "BUY" : score15 < -2 and highvol15 ? "SELL" : "WAIT"

rsi1h     = request.security(syminfo.tickerid, _tf60, ta.rsi(close, rsiLength), barmerge.gaps_off, barmerge.lookahead_on)
stoch1h   = request.security(syminfo.tickerid, _tf60, f_stochrsi(), barmerge.gaps_off, barmerge.lookahead_on)
mom1h     = request.security(syminfo.tickerid, _tf60, close / close[momLength] * 100.0, barmerge.gaps_off, barmerge.lookahead_on)
super1h   = request.security(syminfo.tickerid, _tf60, f_supertrend(), barmerge.gaps_off, barmerge.lookahead_on)
highvol1h = request.security(syminfo.tickerid, _tf60, f_highvol(), barmerge.gaps_off, barmerge.lookahead_on)
cmf1h     = request.security(syminfo.tickerid, _tf60, f_cmf(cmfLength), barmerge.gaps_off, barmerge.lookahead_on)
hurst1h   = request.security(syminfo.tickerid, _tf60, f_hurst(hurstLen), barmerge.gaps_off, barmerge.lookahead_on)
score1h   = (rsi1h < 40 ? 1 : rsi1h > 60 ? -1 : 0) + (stoch1h < 20 ? 1 : stoch1h > 80 ? -1 : 0) + (mom1h > 100 ? 1 : mom1h < 100 ? -1 : 0) + (super1h == 1 ? 1 : -1) + (cmf1h > 0 ? 1 : cmf1h < 0 ? -1 : 0) + (hurst1h > 0.55 ? 1 : hurst1h < 0.45 ? -1 : 0)
signal1h  = score1h > 2 and highvol1h ? "BUY" : score1h < -2 and highvol1h ? "SELL" : "WAIT"

rsi4h     = request.security(syminfo.tickerid, _tf240, ta.rsi(close, rsiLength), barmerge.gaps_off, barmerge.lookahead_on)
stoch4h   = request.security(syminfo.tickerid, _tf240, f_stochrsi(), barmerge.gaps_off, barmerge.lookahead_on)
mom4h     = request.security(syminfo.tickerid, _tf240, close / close[momLength] * 100.0, barmerge.gaps_off, barmerge.lookahead_on)
super4h   = request.security(syminfo.tickerid, _tf240, f_supertrend(), barmerge.gaps_off, barmerge.lookahead_on)
highvol4h = request.security(syminfo.tickerid, _tf240, f_highvol(), barmerge.gaps_off, barmerge.lookahead_on)
cmf4h     = request.security(syminfo.tickerid, _tf240, f_cmf(cmfLength), barmerge.gaps_off, barmerge.lookahead_on)
hurst4h   = request.security(syminfo.tickerid, _tf240, f_hurst(hurstLen), barmerge.gaps_off, barmerge.lookahead_on)
score4h   = (rsi4h < 40 ? 1 : rsi4h > 60 ? -1 : 0) + (stoch4h < 20 ? 1 : stoch4h > 80 ? -1 : 0) + (mom4h > 100 ? 1 : mom4h < 100 ? -1 : 0) + (super4h == 1 ? 1 : -1) + (cmf4h > 0 ? 1 : cmf4h < 0 ? -1 : 0) + (hurst4h > 0.55 ? 1 : hurst4h < 0.45 ? -1 : 0)
signal4h  = score4h > 2 and highvol4h ? "BUY" : score4h < -2 and highvol4h ? "SELL" : "WAIT"

buy1 = signal1 == "BUY"
sell1 = signal1 == "SELL"
buy5 = signal5 == "BUY"
sell5 = signal5 == "SELL"
buy15 = signal15 == "BUY"
sell15 = signal15 == "SELL"
buy1h = signal1h == "BUY"
sell1h = signal1h == "SELL"
buy4h = signal4h == "BUY"
sell4h = signal4h == "SELL"

agreeBuy = (useTF1 ? (buy1 ? 1 : 0) : 0) + (useTF5 ? (buy5 ? 1 : 0) : 0) + (useTF15 ? (buy15 ? 1 : 0) : 0) + (useTF60 ? (buy1h ? 1 : 0) : 0) + (useTF240 ? (buy4h ? 1 : 0) : 0)
agreeSell = (useTF1 ? (sell1 ? 1 : 0) : 0) + (useTF5 ? (sell5 ? 1 : 0) : 0) + (useTF15 ? (sell15 ? 1 : 0) : 0) + (useTF60 ? (sell1h ? 1 : 0) : 0) + (useTF240 ? (sell4h ? 1 : 0) : 0)

buyConsensus = showSignals and agreeBuy >= minAgree
sellConsensus = showSignals and agreeSell >= minAgree

buyTrig = buyConsensus and not buyConsensus[1]
sellTrig = sellConsensus and not sellConsensus[1]

plotshape(buyTrig, title="MTF BUY", location=location.belowbar, color=color.new(color.lime, 0), style=shape.triangleup, size=size.tiny, text="BUY " + str.tostring(agreeBuy))
plotshape(sellTrig, title="MTF SELL", location=location.abovebar, color=color.new(color.red, 0), style=shape.triangledown, size=size.tiny, text="SELL " + str.tostring(agreeSell))

if barstate.islast
    f_draw_row("1M", rsi1, stoch1, mom1, super1, highvol1, cmf1, hurst1, score1, signal1, 1)
    f_draw_row("5M", rsi5, stoch5, mom5, super5, highvol5, cmf5, hurst5, score5, signal5, 2)
    f_draw_row("15M", rsi15, stoch15, mom15, super15, highvol15, cmf15, hurst15, score15, signal15, 3)
    f_draw_row("1H", rsi1h, stoch1h, mom1h, super1h, highvol1h, cmf1h, hurst1h, score1h, signal1h, 4)
    f_draw_row("4H", rsi4h, stoch4h, mom4h, super4h, highvol4h, cmf4h, hurst4h, score4h, signal4h, 5)
