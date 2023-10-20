//@version=5
indicator("luxalgo support resistance signal MTF", overlay=true)

// User settings
src = input(close, title="Source")
timeframe = input("D", title="Higher Timeframe")
color_up = input(color.green, title="Color Up")
color_down = input(color.red, title="Color Down")
color_neutral = input(color.gray, title="Color Neutral")
width = input(2, title="Line Width", minval=1)
style = input("solid", title="Line Style", options=["solid", "dashed", "dotted"])
gap = input(0.01, title="Gap %", minval=0)
sensitivity = input(0.5, title="Sensitivity", minval=0, maxval=1)
freq = input("Once Per Bar Close", title="Alert Frequency", options=["Once Per Bar Close", "Once Per Bar"])
sound = input("Pop", title="Alert Sound", options=["Pop", "Ding", "Bell"])
text = input(true, title="Show Alert Text")

// Higher timeframe data
htf_src = request.security(syminfo.tickerid, timeframe, src)
htf_high = request.security(syminfo.tickerid, timeframe, high)
htf_low = request.security(syminfo.tickerid, timeframe, low)

// Support and resistance levels and zones
sup_res(src) =>
    var float sup = na
    var float res = na
    var float sup_zone = na
    var float res_zone = na
    float dir = change(src) > 0 ? 1 : -1
    float dir_prev = change(src[1]) > 0 ? 1 : -1
    if dir != dir_prev or barstate.islastconfirmedhistory
        sup := dir > 0 ? src[1] : sup[1]
        res := dir < 0 ? src[1] : res[1]
        sup_zone := dir > 0 ? src[1] * (1 - gap / 100) : sup_zone[1]
        res_zone := dir < 0 ? src[1] * (1 + gap / 100) : res_zone[1]
    [sup, res, sup_zone, res_zone]

[sup, res, sup_zone, res_zone] = sup_res(src)
[htf_sup, htf_res, htf_sup_zone, htf_res_zone] = sup_res(htf_src)

// Plot support and resistance levels and zones
plot(sup, color=color_up, linewidth=width, style=plot.style_linebr, title="Support")
plot(res, color=color_down, linewidth=width, style=plot.style_linebr, title="Resistance")
plot(sup_zone, color=color_up, linewidth=width, style=style, title="Support Zone")
plot(res_zone, color=color_down, linewidth=width, style=style, title="Resistance Zone")
plot(htf_sup, color=color_up, linewidth=width * 2, style=plot.style_linebr, title="HTF Support")
plot(htf_res, color=color_down, linewidth=width * 2, style=plot.style_linebr, title="HTF Resistance")
plot(htf_sup_zone, color=color_up, linewidth=width * 2, style=style, title="HTF Support Zone")
plot(htf_res_zone, color=color_down, linewidth=width * 2, style=style, title="HTF Resistance Zone")

// Price action signals
breakout(src, level) => src > level
test(src, level) => src < level and src[1] > level
retest(src, level) => src > level and src[1] < level
reject(src, level) => src < level and src[1] < level

long_signal = breakout(src, res) or retest(src, sup) or breakout(src, htf_res) or retest(src, htf_sup)
short_signal = breakout(src, sup) or retest(src, res) or breakout(src, htf_sup) or retest(src, htf_res)

// Plot signals
plotshape(long_signal ? src : na, style=shape.triangleup, location=location.belowbar,
    color=color.new(color.green), size=size.small,
    text=text ? "Long Signal" : "", textcolor=color.new(color.white), title="Long Signal")
plotshape(short_signal ? src : na , style=shape.triangledown , location=location.abovebar,
    color=color.new(color.red), size=size.small,
    text=text ? "Short Signal" : "", textcolor=color.new(color.white), title="Short Signal")

// Trade panel
target_level = long_signal ? max(htf_high[1], htf_res) : short_signal ? min(htf_low[1], htf_sup) : na
stop_level = long_signal ? min(lowest(low), sup) : short_signal ? max(highest(high), res) : na
risk_reward = abs(target_level - src) / abs(stop_level - src)
trade_dir = long_signal ? "BUY" : short_signal ? "SELL" : "NEUTRAL"
trade_color = long_signal ? color.green : short_signal ? color.red : color.gray

var label trade_label = na
if barstate.islastconfirmedhistory
    trade_label := label.new(x=bar_index + 1.5 , y=target_level , xloc=xloc.bar_index , yloc=yloc.price ,
        text="\n Direction: " + trade_dir + "\n Target: " + tostring(target_level) + "\n Stop: " + tostring(stop_level) + "\n R/R: " + tostring(risk_reward),
        style=label.style_label_left , color=trade_color , textcolor=color.white , size=size.normal)
else
    label.set_xy(trade_label , x=bar_index + 1.5 , y=target_level)
    label.set_text(trade_label , "\n Direction: " + trade_dir + "\n Target: " + tostring(target_level) + "\n Stop: " + tostring(stop_level) + "\n R/R: " + tostring(risk_reward))
    label.set_color(trade_label , trade_color)

// Alerts
alertcondition(long_signal or short_signal , title="Entry Signal", message="{{ticker}} - {{interval}}: Entry Signal - {{close}}")
alertcondition(breakout(src , sup) or breakout(src , res) or breakout(src , htf_sup) or breakout(src , htf_res),
    title="Breakout Alert", message="{{ticker}} - {{interval}}: Breakout Alert - {{close}}")
alertcondition(test(src , sup) or test(src , res) or test(src , htf_sup) or test(src , htf_res),
    title="Test Alert", message="{{ticker}} - {{interval

