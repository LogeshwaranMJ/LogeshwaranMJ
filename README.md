import pandas as pd
import talib
from zipline.api import order_target_percent, record, symbol, history, add_history, set_commission, commission
from zipline.algorithm import TradingAlgorithm
from zipline.finance import commission

class MomentumMeanReversion(TradingAlgorithm):
    def initialize(context):
        context.instruments = [symbol('NIFTY'), symbol('BANKNIFTY')]
        context.ema_short_period = 9
        context.ema_long_period = 21
        context.rsi_period = 14
        context.atr_period = 14
        context.rsi_overbought = 70
        context.rsi_oversold = 30
        context.trailing_stop_multiplier = 1.5
        
        add_history(50, '1m', 'price')
        add_history(50, '1m', 'high')
        add_history(50, '1m', 'low')
        
        set_commission(commission.PerShare(cost=0.001))
    
    def handle_data(context, data):
        for symbol in context.instruments:
            prices = history(50, '1m', 'price')[symbol]
            high = history(50, '1m', 'high')[symbol]
            low = history(50, '1m', 'low')[symbol]

            ema_short = talib.EMA(prices, timeperiod=context.ema_short_period)
            ema_long = talib.EMA(prices, timeperiod=context.ema_long_period)
            rsi = talib.RSI(prices, timeperiod=context.rsi_period)
            atr = talib.ATR(high, low, prices, timeperiod=context.atr_period)
            
            current_price = prices[-1]
            trailing_stop = atr[-1] * context.trailing_stop_multiplier

            # Momentum Strategy
            if ema_short[-1] > ema_long[-1] and rsi[-1] > 60:
                order_target_percent(symbol, 0.1)
            elif ema_short[-1] < ema_long[-1] and rsi[-1] < 40:
                order_target_percent(symbol, -0.1)

            # Mean-Reversion Strategy
            if rsi[-1] < context.rsi_oversold:
                order_target_percent(symbol, 0.1)
            elif rsi[-1] > context.rsi_overbought:
                order_target_percent(symbol, -0.1)

            # Trailing Stop-Loss
            if symbol in context.portfolio.positions:
                position = context.portfolio.positions[symbol]
                if position.amount > 0 and current_price < position.cost_basis - trailing_stop:
                    order_target_percent(symbol, 0)
                elif position.amount < 0 and current_price > position.cost_basis + trailing_stop:
                    order_target_percent(symbol, 0)

    def analyze(context, perf):
        perf.to_csv('strategy_performance.csv')

# Data preparation and backtesting execution
import pandas as pd
from zipline.utils.factory import create_simulation_parameters
from datetime import datetime

start = pd.Timestamp(datetime(2021, 1, 1))
end = pd.Timestamp(datetime(2021, 12, 31))

simulation_params = create_simulation_parameters(start=start, end=end)
data = ... # Load your data here

algo = MomentumMeanReversion(simulation_params=simulation_params)
results = algo.run(data)
algo.analyze(results)
