!pip install git+https://github.com/rongardF/tvdatafeed tradingview-screener
import numpy as np
import pandas as pd
import time
from tvDatafeed import TvDatafeed, Interval
from tradingview_screener import get_all_symbols
import warnings

warnings.simplefilter(action='ignore')

# Standart Moving Average
def sma(series, length):
    return series.rolling(window=length).mean()

def Bankery(data, length=20, mult=1.0, atrLen=500):
    df = data.copy()

    # Calculate moving average
    ma = sma(data['close'], length)

    # Calculate ATR
    high_low = data['high'] - data['low']
    high_close = np.abs(data['high'] - data['close'].shift())
    low_close = np.abs(data['low'] - data['close'].shift())
    tr = high_low.combine(high_close, np.maximum).combine(low_close, np.maximum)
    atr = tr.rolling(window=atrLen).mean()
    atr_mult = atr * mult

    # Range detection
    count = (np.abs(data['close'] - ma) > atr_mult).rolling(window=length).sum()
    df['Range Detected'] = (count == 0) & (count.shift(1) != 0)

    return df

tv = TvDatafeed()
Hisseler = get_all_symbols(market='turkey')
Hisseler = [symbol.replace('BIST:', '') for symbol in Hisseler]
Hisseler = sorted(Hisseler)

# Periyotlar ve isimleri
intervals = {
    Interval.in_30_minute: '30 Dakikalık',
    Interval.in_2_hour: '2 Saatlik',
    Interval.in_4_hour: '4 Saatlik',
    Interval.in_daily: 'Günlük',
}

# Raporlama için kullanılacak başlıklar
Titles = ['Hisse Adı', 'Son Fiyat', 'Periyot']
df_signals = pd.DataFrame(columns=Titles)

def get_data_with_retry(tv, symbol, exchange, interval, n_bars, max_retries=5):
    retries = 0
    while retries < max_retries:
        try:
            data = tv.get_hist(symbol=symbol, exchange=exchange, interval=interval, n_bars=n_bars)
            return data
        except Exception as e:
            retries += 1
            print(f"Error fetching data for {symbol} at {interval}: {e}. Retrying {retries}/{max_retries}...")
            time.sleep(5)  # Wait for 5 seconds before retrying
    raise Exception(f"Failed to fetch data for {symbol} at {interval} after {max_retries} retries.")

for hisse in Hisseler:
    for interval, period_name in intervals.items():
        try:
            data = get_data_with_retry(tv, hisse, 'BIST', interval, 100)
            data = data.reset_index()
            Banker = Bankery(data)
            Banker.rename(columns={'open': 'Open', 'high': 'High', 'low': 'Low', 'close': 'Close', 'volume': 'Volume'}, inplace=True)
            Banker.set_index('datetime', inplace=True)
            Signals = Banker.tail(2)
            Signals = Signals.reset_index()

            Range_Detected = (Signals.loc[0, 'Range Detected'] == False) & (Signals.loc[1, 'Range Detected'] == True)
            Last_Price = Signals.loc[1, 'Close']

            if Range_Detected:
                L1 = [hisse, Last_Price, period_name]
                df_signals.loc[len(df_signals)] = L1
                print(L1)
        except Exception as e:
            print(f"Error processing {hisse} at {period_name}: {e}")

# Sort results by time periods (smallest to largest)
period_order = ['30 Dakikalık', '2 Saatlik', '4 Saatlik', 'Günlük']
df_signals['Periyot'] = pd.Categorical(df_signals['Periyot'], categories=period_order, ordered=True)
df_signals = df_signals.sort_values(by=['Periyot'])

# Display all signals
print(df_signals.to_string(index=False))

#!/usr/bin/env python3
from subprocess import Popen, PIPE

with Popen(r'C:\path\to\program.exe "arg 1" "arg 2"',
           stdout=PIPE, stderr=PIPE) as p:
    output, errors = p.communicate()
lines = output.decode('utf-8').splitlines()
