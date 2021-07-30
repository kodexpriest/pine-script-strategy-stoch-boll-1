//@version=4
//
//  This strategy triggers below (above) Bollinger band, in an oversold (overbought) condition determined by stochastic index.
//  It only accept long (short) trades with prices over EMA 200.
//  Fully customizable. It uses ATR to set the stop loss and take profit parameters.
//  
//
strategy("Kodex's Strategy: 15' BB+Stoch", shorttitle="K's BB+STOCH", overlay=true, commission_value=0.075, pyramiding=0, currency="USD", initial_capital=100)


//▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒//
//Vars, Inputs and consts
ema_slow = ema(close,input(200, "Slow EMA"))
using_ema = input(true, "Use EMA restriction")
go_long = input(true, "Go Long")
go_short = input(true, "Go Short")

margin = atr(input(7,"ATR lenght")) * input(1.5, "ATR multiplicator (sl and tp mult)", step=0.1)
lookback = input(7, "How far to look back for high/lows")
tp_factor = input(2.5, "Take Profit/Stop Loss relation", step=0.1)

var long_tsl = 0.0
var short_tsl = 0.0
var long_tp = 0.0
var short_tp = 0.0

has_open_trade() => strategy.opentrades != 0
is_long = strategy.position_size>0 and has_open_trade()
is_short = strategy.position_size<0 and has_open_trade()


// From Date Inputs
fromDay = input(defval = 1, title = "From Day", minval = 1, maxval = 31)
fromMonth = input(defval = 7, title = "From Month", minval = 1, maxval = 12)
fromYear = input(defval = 2021, title = "From Year", minval = 2005)
 
// To Date Inputs
toDay = input(defval = 1, title = "To Day", minval = 1, maxval = 31)
toMonth = input(defval = 8, title = "To Month", minval = 1, maxval = 12)
toYear = input(defval = 2021, title = "To Year", minval = 2005)
 
// Calculate start/end date and time condition
startDate = timestamp(fromYear, fromMonth, fromDay, 00, 00)
finishDate = timestamp(toYear, toMonth, toDay, 00, 00)
time_cond = time >= startDate and time <= finishDate



//▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒//
//Stochastic
periodK = 14//input(14, title="STOCH %K Length", minval=1)
smoothK = 1//input(1, title="STOCH %K Smoothing", minval=1)
periodD = 3//input(3, title="STOCH %D Smoothing", minval=1)
k = sma(stoch(close, high, low, periodK), smoothK)
d = sma(k, periodD)

//Bollinger Band
length = 20//input(20, minval=1, title="BB Length")
src = close//input(close, title="BB Source")
mult = 2.0//input(2.0, minval=0.001, maxval=50, title="BB StdDev")
basis = sma(src, length)
dev = mult * stdev(src, length)
bb_upper = basis + dev
bb_lower = basis - dev


//▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒//
//Logic & trailing stop loss

long_trigger = (k<20 and hl2<bb_lower and (low>ema_slow or not using_ema))
short_trigger = (k>80 and hl2>bb_lower and (high<ema_slow or not using_ema))


long_signal = long_trigger and go_long and not has_open_trade()
short_signal = short_trigger and go_short and not has_open_trade()

long_sl = lowest(close, lookback) - margin
short_sl = highest(close, lookback) + margin

long_tsl := long_signal ? long_sl : max(long_sl, nz(long_tsl[1]))
short_tsl := short_signal ? short_sl : min(short_sl, nz(short_tsl[1],9999999))


if long_signal and not has_open_trade() and time_cond
    strategy.entry(id="Long", long=true, comment="Long", alert_message="Long signal")
    long_tp := close + (margin * nz(tp_factor,1))

if short_signal and not has_open_trade() and time_cond
    strategy.entry(id="Short", long=false, comment="Short", alert_message="Short signal")
    short_tp := close - (margin * nz(tp_factor,1))
    
if has_open_trade()
    if strategy.position_size>0
        if short_trigger
            strategy.exit(id="Exit Long", from_entry="Long", comment="[cancel]", alert_message="Incoming Short", stop=close)
        else if close <= long_tsl
            strategy.exit(id="Exit Long", from_entry="Long", comment="[SL]", alert_message="Exit by Long TSL", stop=close)
        else if close >= long_tp
            strategy.exit(id="Exit Long", from_entry="Long", comment="[TP]", alert_message="Exit by Long TP", stop=close)

        
    if strategy.position_size<0
        if long_trigger
            strategy.exit(id="Exit Short", from_entry="Short", comment="[cancel]", alert_message="Incoming Long", stop=close)
        else if close >= short_tsl
            strategy.exit(id="Exit Short", from_entry="Short", comment="[SL]", alert_message="Exit by Short TSL", stop=close)
        else if close <= short_tp
            strategy.exit(id="Exit Short", from_entry="Short", comment="[TP]", alert_message="Exit by Short TP", stop=close)




//▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒//
//Plots
color_bull = color.new(#26a69a,0)
color_bear = color.new(#ef5350,0)
strat_color = color.new(#000000,92)

//BB
p1 = plot(bb_upper, "Upper", color=color.rgb(41,98,255,90), display=display.all)
p2 = plot(bb_lower, "Lower", color=color.rgb(41,98,255,90), display=display.all)
fill(p1, p2, title = "Background", color=color.rgb(41,98,255,95))

//EMAs
plot(ema_slow, title="Slow EMA", color=strat_color, linewidth=4)

//TSL
plotshape(is_long?long_tsl[1]:na, title="long tsl", location=location.absolute, color=color.new(color.red,75), size=size.tiny, style=shape.triangleup, display=display.all)
plotshape(is_short?short_tsl[1]:na, title="short tsl", location=location.absolute, color=color.new(color.red,75), size=size.tiny, style=shape.triangledown, display=display.all)
