# mamba-autosignal-1min
auto signal most profitable strategy 





 //@version=6
strategy("Mamba FX Universal Pro", overlay=true,
     pyramiding=0, initial_capital=10000,
     process_orders_on_close=true)

// ========== SETTINGS ==========
zoneTF = input.timeframe("5", "S/R Timeframe")
pivotLen = input.int(3, "Pivot Strength", minval=1)
zonePct = input.float(0.05, "Zone Width (%)", minval=0.001, step=0.01)
rr = input.float(2.0, "Risk Reward", minval=1.0, step=0.5)
slLookback = input.int(5, "SL Swing Lookback", minval=2)
maxWait = input.int(10, "Max Bars After Tap", minval=1)
allowBuy = input.bool(true, "Enable BUY")
allowSell = input.bool(true, "Enable SELL")
showZones = input.bool(true, "Show S/R Levels")

// Run entries on 1-minute chart
isOneMinute = timeframe.isminutes and timeframe.multiplier == 1

// ========== CONFIRMED 5M PIVOTS ==========
ph = request.security(syminfo.tickerid, zoneTF,
     ta.pivothigh(high, pivotLen, pivotLen),
     lookahead=barmerge.lookahead_off)

pl = request.security(syminfo.tickerid, zoneTF,
     ta.pivotlow(low, pivotLen, pivotLen),
     lookahead=barmerge.lookahead_off)

var float resistance = na
var float support = na

if not na(ph)
    resistance := ph

if not na(pl)
    support := pl

// Zones
supTop = support * (1 + zonePct / 100)
supBot = support * (1 - zonePct / 100)
resTop = resistance * (1 + zonePct / 100)
resBot = resistance * (1 - zonePct / 100)

supportTap = not na(support) and low <= supTop and high >= supBot
resistanceTap = not na(resistance) and high >= resBot and low <= resTop

// ========== TAP -> BREAKOUT ==========
var bool buyArmed = false
var bool sellArmed = false
var float buyTrigger = na
var float sellTrigger = na
var float buySL = na
var float sellSL = na
var int buyTapBar = na
var int sellTapBar = na

// Arm only once per setup
if isOneMinute and supportTap and not buyArmed
    buyArmed := true
    buyTrigger := high
    buySL := ta.lowest(low, slLookback)
    buyTapBar := bar_index

if isOneMinute and resistanceTap and not sellArmed
    sellArmed := true
    sellTrigger := low
    sellSL := ta.highest(high, slLookback)
    sellTapBar := bar_index

// Expire old setups
if buyArmed and bar_index - buyTapBar > maxWait
    buyArmed := false

if sellArmed and bar_index - sellTapBar > maxWait
    sellArmed := false

// Breakout must happen AFTER the tap candle
buySignal = isOneMinute and allowBuy and buyArmed and
     bar_index > buyTapBar and close > buyTrigger and
     close > open

sellSignal = isOneMinute and allowSell and sellArmed and
     bar_index > sellTapBar and close < sellTrigger and
     close < open

// ========== ENTRIES + 1:2 TARGET ==========
if buySignal
    buyArmed := false
    if strategy.position_size == 0 and buySL < close
        risk = close - buySL
        tp = close + risk * rr
        strategy.entry("BUY", strategy.long)
        strategy.exit("BUY TP/SL", from_entry="BUY",
             stop=buySL, limit=tp)

if sellSignal
    sellArmed := false
    if strategy.position_size == 0 and sellSL > close
        risk = sellSL - close
        tp = close - risk * rr
        strategy.entry("SELL", strategy.short)
        strategy.exit("SELL TP/SL", from_entry="SELL",
             stop=sellSL, limit=tp)

// ========== CHART DISPLAY ==========
plot(showZones ? support : na, "5m Support",
     color=color.lime, linewidth=2, style=plot.style_stepline)

plot(showZones ? resistance : na, "5m Resistance",
     color=color.red, linewidth=2, style=plot.style_stepline)

p1 = plot(showZones ? supTop : na, "Support Zone Top",
     color=color.new(color.green, 75))
p2 = plot(showZones ? supBot : na, "Support Zone Bottom",
     color=color.new(color.green, 75))
fill(p1, p2, color=color.new(color.green, 88))

p3 = plot(showZones ? resTop : na, "Resistance Zone Top",
     color=color.new(color.red, 75))
p4 = plot(showZones ? resBot : na, "Resistance Zone Bottom",
     color=color.new(color.red, 75))
fill(p3, p4, color=color.new(color.red, 88))

plotshape(buySignal, title="BUY Signal", text="BUY",
     style=shape.labelup, location=location.belowbar,
     color=color.green, textcolor=color.white, size=size.small)

plotshape(sellSignal, title="SELL Signal", text="SELL",
     style=shape.labeldown, location=location.abovebar,
     color=color.red, textcolor=color.white, size=size.small)

// ========== ALERTS ==========
alertcondition(buySignal, title="Mamba BUY",
     message="Mamba BUY: {{ticker}} at {{close}}")

alertcondition(sellSignal, title="Mamba SELL",
     message="Mamba SELL: {{ticker}} at {{close}}")