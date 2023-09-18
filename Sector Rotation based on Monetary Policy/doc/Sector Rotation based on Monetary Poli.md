# Sector Rotation based on Monetary Policy
A sector rotational strategy picks several sectors from the equity index with the intention to outperform the broad equity index. A key factor is selecting indicators that effectively identify when the portfolio should be shifted to more defensive or more aggressive sectors. Academic research suggests that changes in the Federal Reserve monetary policy provide one such indicator. Fed easing has usually favored cyclical stocks, while Fed tightening has favored defensive stocks. Furthermore, the average returns for every sector are higher when the Fed has an expansive policy stance.

## Fundamental reason
The Fed’s actions impact current and future business and economic conditions. Cyclical stocks are more sensible to FED policy and the basic discount rate. Therefore, it pays off to own cyclical stocks during an expansive state and own defensive stocks in a restrictive state.


name | value | 
---------|----------|
 Markets Traded | equities |
 Backtest period from source paper | 1973-2005 |
 Confidence in anomaly's validity | Strong |
 Indicative Performance | 15.82% |
 Notes to Confidence in Anomaly's Validity| |
 Notes to Indicative Performance| per annum, data from table 4 panel A, benchmark 12,04% – market portfolio | performance| |
 Period of Rebalancing | Yearly |
 Estimated Volatility | 15.5% |
 Notes to Period of Rebalancing | 
 Notes to Estimated Volatility | data from table 4 panel A, benchmark 15,50% – market portfolio performance |
 Number of Traded Instruments | 10 |
 Maximum Drawdown | |
 Notes to Number of Traded Instruments | |
Notes to Maximum drawdown | |
Complexity Evaluation | |
Sharpe Ratio | 0.76 | 
Notes to Complexity Evaluation | |
Region | United States |
Financial instruments | ETFs, funds, stocks |

## Simple trading strategy
Stocks in the US equity market are divided into ten sectors: Resources, Noncyclical Consumer Goods, Noncyclical Services, and Utilities such as defense sectors and Cyclical Consumer Goods, Cyclical Services, General Industrials, Information Technology, Financials, and Basic Industries as cyclical sectors. As this allocation doesn’t exactly fit a standard sector decomposition, it isn’t possible to use conventional ten sector ETFs for all sectors, and some sectors used in this strategy must be created directly from stocks or industry ETFs or funds. The portfolio is then made up of equally-weighted six cyclical sectors during periods of expansive monetary policy and equally weighted four noncyclical sectors during periods of restrictive monetary policy. The period of the restrictive policy starts after the FED begins to raise rates and lasts until it starts to cut rates when we enter a period of the expansive policy.

## Hedge for stocks during bear markets
No - Sector rotation strategies are the most often implemented in a long-only variant, which means that they are not suitable to hedge equity market risk. However, an amended strategy in which the shorts equity market (in addition to going long some sectors) can have a negative correlation to equities, but this must be decided by carefully implemented backtest.

## Source paper
Conover, Jensen, Johnson, Mercer: Sector Rotation and Monetary Conditions
https://quantpedia.com/www/Sector_Rotation_and_Monetary_Conditions.pdf
- Abstract
https://joi.pm-research.com/content/17/1/34

We investigate the efficacy of a sector rotation strategy that utilizes an easily observable signal based on monetary conditions. Using 33 years of data, we find that the rotation strategy earns consistent and economically significant excess returns while requiring only infrequent rebalancing. The strategy places greater emphasis on cyclical stocks during periods of Fed easing, and over-weights defensive stocks during periods of Fed tightening. Interestingly, the benefits from the rotation strategy accrue predominantly during periods of poor equity market performance, which is when performance enhancement is most valued by investors. Specifically, during restrictive monetary periods, returns to the strategy are nearly twice that of comparable investments, yet the strategy assumes less risk. Overall, our results suggest that investors should consider monetary conditions when determining their portfolio allocations.

![Alt text](images/%E6%93%B7%E5%8F%96.PNG)
![Alt text](images/%E6%93%B7%E5%8F%962.PNG)
```
# QuantBook Analysis Tool 
# For more information see [https://www.quantconnect.com/docs/v2/our-platform/research/getting-started]
qb = QuantBook()
spy = qb.AddEquity("SPY")
history = qb.History(qb.Securities.Keys, 360, Resolution.Daily)

# Indicator Analysis
bbdf = qb.Indicator(BollingerBands(30, 2), spy.Symbol, 360, Resolution.Daily)
bbdf.drop('standarddeviation', axis=1).plot()
```
<matplotlib.axes._subplots.AxesSubplot at 0x7ff0d5d81128>
![Alt text](images/%E4%B8%8B%E8%BC%89.png)
```python
# https://quantpedia.com/strategies/sector-rotation-based-on-monetary-policy/
# 
# Stocks in the US equity market are divided into ten sectors: Resources, Noncyclical Consumer Goods, Noncyclical Services, and Utilities such as defense sectors and Cyclical Consumer Goods,
# Cyclical Services, General Industrials, Information Technology, Financials, and Basic Industries as cyclical sectors. As this allocation doesn’t exactly fit a standard sector decomposition, 
# it isn’t possible to use conventional ten sector ETFs for all sectors, and some sectors used in this strategy must be created directly from stocks or industry ETFs or funds. The portfolio is
# then made up of equally-weighted six cyclical sectors during periods of expansive monetary policy and equally weighted four noncyclical sectors during periods of restrictive monetary policy. 
# The period of the restrictive policy starts after the FED begins to raise rates and lasts until it starts to cut rates when we enter a period of the expansive policy.
#
# QC implementation:
#   - Combination of set FED policy change dates and external FED rates is used to distinguish monetary policy.

from AlgorithmImports import *
from pandas.tseries.offsets import BDay

class SectorRotation(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(1999, 1, 1)
        self.SetCash(100000)
        
        self.cyclical:List[str] = ["XLY", "XLK", "XLF","XLI"]
        self.defensive:List[str] = ["XLU", "XLP", "XLV", "XLB", "XLE"]
        
        for symbol in self.cyclical:
            self.AddEquity(symbol, Resolution.Daily)

        for symbol in self.defensive:
            self.AddEquity(symbol, Resolution.Daily)
        
        # changes in FED policy from restrictive to expansive
        dates_str:List[str] = ["24.08.1999", "03.01.2001", "30.06.2004", "18.09.2007", "14.12.2016", "31.7.2019"]
        self.dates:List[datetime.date] = [datetime.strptime(x, "%d.%m.%Y").date() for x in dates_str] # datetime type
        
        # import quandl federal rate data
        self.target_rate:Symbol = self.AddData(FederalTargetRange, 'DFEDTARU', Resolution.Daily).Symbol
        self.restrictive_flag:bool = True    # from start of the algorithm
        self.last_target_rate = None
        self.external_restrictive_flag = None
    
    def OnData(self, data:Slice) -> None:
        if self.target_rate in data and data[self.target_rate]:
            curr_target_rate:float = data[self.target_rate].Value
            curr_date:datetime.date = self.Time.date()
            restrictive_flag = None
            
            # switch to external data source, when self.dates ends
            if self.Time.date() > self.dates[-1]:
                if self.last_target_rate:
                    if curr_target_rate > self.last_target_rate and self.restrictive_flag:
                        restrictive_flag = True
                        self.dates.append(curr_date + BDay(1))
                    elif curr_target_rate < self.last_target_rate and not self.restrictive_flag:
                        restrictive_flag = False
                        self.dates.append(curr_date + BDay(1))

            if restrictive_flag is not None:
                self.external_restrictive_flag = restrictive_flag
                    
            self.last_target_rate = curr_target_rate
    
        if self.Time.date() in self.dates:
            # if external data is present, trade out of it
            if self.external_restrictive_flag is not None:
                restrictive_flag:bool = self.external_restrictive_flag

                # end of quandl target rate data
                ftr_last_update_date:Dict[Symbol, datetime.date] = FederalTargetRange.get_last_update_date()
                
                if self.Securities[self.target_rate].GetLastData() and self.target_rate in ftr_last_update_date and self.Time.date() >= ftr_last_update_date[self.target_rate]: 
                    self.Liquidate()
                    return

            else:
                restrictive_flag:bool = self.restrictive_flag

            if restrictive_flag:

                for symbol in self.cyclical:
                    self.Liquidate(symbol)
                for symbol in self.defensive:
                    self.SetHoldings(symbol, 1/len(self.defensive))

                self.restrictive_flag = False

            else:
                for symbol in self.defensive:
                    self.Liquidate(symbol)
                for symbol in self.cyclical:
                    self.SetHoldings(symbol, 1/len(self.cyclical))


                self.restrictive_flag = True

# source: https://fred.stlouisfed.org/series/DFEDTARU
class FederalTargetRange(PythonData):
    def GetSource(self, config, date, isLiveMode):
        return SubscriptionDataSource('data.quantpedia.com/backtesting_data/economic/DFEDTARU.csv', SubscriptionTransportMedium.RemoteFile, FileFormat.Csv)

    _last_update_date:Dict[Symbol, datetime.date] = {}

    @staticmethod
    def get_last_update_date() -> Dict[Symbol, datetime.date]:
       return FederalTargetRange._last_update_date

    def Reader(self, config, line, date, isLiveMode):
        data = FederalTargetRange()
        data.Symbol = config.Symbol

        if not line[0].isdigit(): return None
        split = line.split(';')
        
        # Parse the CSV file's columns into the custom data class
        data.Time = datetime.strptime(split[0], "%Y-%m-%d") + timedelta(days=1)
        data.Value = float(split[1])

        if config.Symbol not in FederalTargetRange._last_update_date:
            FederalTargetRange._last_update_date[config.Symbol] = datetime(1,1,1).date()
        if data.Time.date() > FederalTargetRange._last_update_date[config.Symbol]:
            FederalTargetRange._last_update_date[config.Symbol] = data.Time.date()
        
        return data

```

### Related picture
![Alt text](images/Sn%C3%ADmka-obrazovky-2020-10-26-o-18.22.46.png)
