//@version=4
//
//  Fully customizable. It uses ATR to set the stop loss and take profit parameters and ADX to operate in trending markets.
//  
//
strategy("Kodex's Bot: 15' Mix", shorttitle="K's Bot: 15'", overlay=true, commission_value=0.024, pyramiding=0, currency="USD", initial_capital=100, process_orders_on_close=true)


//▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒//
//Vars, Inputs and consts
time_cond       = time > (timenow-input(5, "Days back to backtesting")*86400000)
show_bb         = input(false,"Show Bollinger Bands. Aesthetic.")


DEFAULT_SLOW    = 200
DEFAULT_BASE    = 55
DEFAULT_FAST    = 20
DEFAULT_ADX     = 28
DEFAULT_LEN     = 7

//Entry filters
go_long         = input(true, "Go Long", group="Entry filters", inline="go")
go_short        = input(true, "Go Short", group="Entry filters", inline="go")
go_adx          = input(true, "ADX looking for trends", group="Entry filters", inline="go")
go_sar          = input(true, "SAR filter", group="Entry filters", inline="go")
go_sar_cross    = input(true, "SAR cross trigger", group="Entry filters", inline="go")


//ADX
go_adx_val      = input(DEFAULT_ADX, "Limit",  inline="adx", group="ADX")
adxlen          = input(DEFAULT_LEN, "Smoothing", group="ADX", inline="adx" )
dilen           = input(DEFAULT_LEN, "length", group="ADX", inline="adx")


//Hullma
hullma(hullma_length) =>
    wma(2*wma(close, hullma_length/2)-wma(close, hullma_length), floor(sqrt(hullma_length)))

//MAs
input_ema_s = input(DEFAULT_SLOW, "Slow EMA", group="MAs", inline="hullma")
input_ema_b = input(DEFAULT_BASE, "Base Ema", group="MAs", inline="hullma")
input_ema_f = input(DEFAULT_FAST, "Fast EMA", group="MAs", inline="hullma")

ema_slow        = hullma(input_ema_s)
ema_base        = hullma(input_ema_b)
ema_fast        = hullma(input_ema_f)
ema_direction   = input(true, "Use Price direction based on EMAs?", tooltip="Evaluate Long entries when the slow ema slope is possitive and the fast Ema is above the slow Ema\n(Viceversa for shorts)", group="EMAs")


//TP/SLs
lookback        = input(DEFAULT_LEN, "How far to look back for high/lows and ATR (volatility) caluclation",group="TP/SL")
win             = atr(lookback) * input(1, "ATR multiplictor", tooltip="Distance from price for SL and TP caluclation", step=0.1, group="TP/SL")
tp_factor       = input(1.5, "Take Profit/Stop Loss relation", step=0.1, tooltip="example: setting ATR mult to '1.5' and TP/SL to '2' sets the SL=1.5 and TP=3 times atr from price.", group="TP/SL")
macd_tolerance  = input(10, "% MacD cross reversion tolerance",tooltip="After MacD crossing under (over) singal, strategy will wait for x% separation to trigger 'Cancel strategy' condition", group="TP/SL")


var long_tsl = 0.0
var short_tsl = 0.0
var long_tp = 0.0
var short_tp = 0.0

has_open_trade() => strategy.opentrades != 0
is_long = strategy.position_size>0 and has_open_trade()
is_short = strategy.position_size<0 and has_open_trade()


//Parabolic SAR
start = input(0.02,title="Start", group="Parabolic SAR", inline="sar")
increment = input(0.02)
maximum = input(0.2,title="MAX", group="Parabolic SAR", inline="sar")
sar = sar(start, increment, maximum)


//Stochastic
periodK = 14//input(14, title="STOCH %K Length", minval=1)
smoothK = 1//input(1, title="STOCH %K Smoothing", minval=1)
periodD = 3//input(3, title="STOCH %D Smoothing", minval=1)
k = sma(stoch(close, high, low, periodK), smoothK)
d = sma(k, periodD)


//Bollinger Band
length = input(20, minval=1, title="Length",group="Bollinger", inline="bb")
src = close//input(close, title="BB Source")
mult = input(2.0, minval=0.001, maxval=50, title="StdDev",group="Bollinger", inline="bb")
basis = sma(src, length)
dev = mult * stdev(src, length)
bb_upper = basis + dev
bb_lower = basis - dev


//MACD
macd_fast_ma = ema(close,input(DEFAULT_FAST,"fast EMA", group="MACD", inline="macd"))
macd_slow_ma = ema(close,input(DEFAULT_BASE,"base EMA", group="MACD", inline="macd"))
macd = macd_fast_ma - macd_slow_ma
macd_signal = ema(macd, input(DEFAULT_LEN,"signal length", group="MACD", inline="macd"))
macd_hist = macd - macd_signal
macd_fore_bars  = input(3,"Bars to calculate MACD forecast", group="MACD")
mcad_prediction = macd + change(macd,macd_fore_bars)


//ADX
dirmov(len) =>
	up = change(high)
	down = -change(low)
	plusDM = na(up) ? na : (up > down and up > 0 ? up : 0)
	minusDM = na(down) ? na : (down > up and down > 0 ? down : 0)
	truerange = rma(tr, len)
	plus = fixnan(100 * rma(plusDM, len) / truerange)
	minus = fixnan(100 * rma(minusDM, len) / truerange)
	[plus, minus]
adx_(dilen, adxlen) =>
	[plus, minus] = dirmov(dilen)
	sum = plus + minus
	adx = 100 * rma(abs(plus - minus) / (sum == 0 ? 1 : sum), adxlen)

adx = adx_(dilen, adxlen)



//▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒//
//Logic

f_long_sar = (not go_sar_cross or crossunder(sar,close)) and (not go_sar or sar<close)
f_shrt_sar = (not go_sar_cross or crossover(sar,close)) and (not go_sar or sar>close)

f_long_ema = ohlc4>ema_slow and ema_fast>ema_base and ema_base>ema_slow
f_shrt_ema = ohlc4<ema_slow and ema_fast<ema_base and ema_base<ema_slow

f_long_dir = not ema_direction or change(ema_slow)>0
f_shrt_dir = not ema_direction or change(ema_slow)<0

f_adx = (go_adx?go_adx_val<adx:true)

long_trigger = f_long_sar and f_long_ema and f_long_dir and f_adx
shrt_trigger = f_shrt_sar and f_shrt_ema and f_shrt_dir and f_adx

long_cancel = shrt_trigger
shrt_cancel = long_trigger



//▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒//
//TP/SL

long_signal = long_trigger and go_long and not has_open_trade() and time_cond
shrt_signal = shrt_trigger and go_short and not has_open_trade() and time_cond

long_sl = lowest(close, lookback) - win
short_sl = highest(close, lookback) + win

long_tsl := long_signal ? long_sl : max(long_sl, nz(long_tsl[1]))
short_tsl := shrt_signal ? short_sl : min(short_sl, nz(short_tsl[1],9999999))


if long_signal
    strategy.entry(id="Long", long=true, comment="Long", alert_message="Long signal")
    long_tp := close + (win * nz(tp_factor,1))

if shrt_signal
    strategy.entry(id="Short", long=false, comment="Short", alert_message="Short signal")
    short_tp := close - (win * nz(tp_factor,1))
    
if has_open_trade()
    if strategy.position_size>0
        if close >= long_tp
            strategy.exit(id="Exit Long", from_entry="Long", comment="[TP]", alert_message="Exit by Long TP", stop=close)
        else if close <= long_tsl
            strategy.exit(id="Exit Long", from_entry="Long", comment="[SL]", alert_message="Exit by Long TSL", stop=close)
        else if long_cancel
            strategy.exit(id="Exit Long", from_entry="Long", comment="[cancel]", alert_message="Incoming Short", stop=close)


        
    if strategy.position_size<0
        if close >= short_tsl
            strategy.exit(id="Exit Short", from_entry="Short", comment="[SL]", alert_message="Exit by Short TSL", stop=close)
        else if close <= short_tp
            strategy.exit(id="Exit Short", from_entry="Short", comment="[TP]", alert_message="Exit by Short TP", stop=close)
        else if shrt_cancel
            strategy.exit(id="Exit Short", from_entry="Short", comment="[Cancel]", alert_message="Incoming Long", stop=close)




//▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒//
//Plots
color_bull = color.new(#66bb6a,75)
color_bear = color.new(#ffa726,75)
strat_color = color.new(#3f3f3f,85)


    

//BB
p1 = plot(show_bb?bb_upper:na, "Upper", color=color.rgb(41,98,255,90), display=display.all)
p2 = plot(show_bb?bb_lower:na, "Lower", color=color.rgb(41,98,255,90), display=display.all)

fill(p1, p2, title = "Background", color=color.rgb(41,98,255,95))

//EMAs
if long_trigger
    strat_color := color_bear
else if shrt_trigger
    strat_color := color_bull
else
    strat_color := color.new(#e29ae4, 50)
    
plot(ema_slow, title="Slow EMA", color=strat_color, linewidth=5)
plot(ema_base, title="Slow EMA", color=color.new(#cf58d3,70), linewidth=2)
plot(ema_fast, title="Fast EMA", color=color.new(#cf58d3,20), linewidth=1)


label_long = "tri:"+tostring(long_trigger)+"\nsig:"+tostring(long_signal)
label_shrt = "tri:"+tostring(shrt_trigger)+"\nsig:"+tostring(shrt_signal)


//TSL
plotshape(is_long?long_tsl[1]:na, title="long tsl", location=location.absolute, color=color.new(color.red,75), size=size.tiny, style=shape.triangleup, display=display.all)
plotshape(is_short?short_tsl[1]:na, title="short tsl", location=location.absolute, color=color.new(color.red,75), size=size.tiny, style=shape.triangledown, display=display.all)


//SAR
plot(sar, "ParabolicSAR", style=plot.style_cross, color=#2962FF)
