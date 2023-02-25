<template>
    <div class="open-trades">
      <h2>Open Trades</h2>
      <button class="start-btn" @click="onClick">Start Bot</button>
      <div class="trade-list">
        <div class="trade-card" v-for="(trade, index) in symbolsData" :key="index">
          <div class="trade-card-front" :class="{ 'profit': trade.profitLoss > 0, 'loss': trade.profitLoss < 0, 'breakeven': trade.profitLoss === 0 }">
            <h3>{{ trade.symbol }}</h3>
            <div class="key-metrics">
              <p>Buy Price: {{ trade.buyPrice }}</p>
              <p>Stop Loss: {{ trade.stopLoss }}</p>
              <p>Profit Target: {{ trade.profitTarget }}</p>
            </div>
            <div class="trade-status">
              <i class="fas fa-circle" :class="{ 'text-success': trade.profitLoss > 0, 'text-danger': trade.profitLoss < 0, 'text-warning': trade.profitLoss === 0 }"></i>
              <span>{{ trade.profitLoss | formatProfitLoss }}</span>
            </div>
          </div>
          <div class="trade-card-back">
            <h3>{{ trade.symbol }}</h3>
            <div class="key-metrics">
              <p>Buy Price: {{ trade.buyPrice }}</p>
              <p>Stop Loss: {{ trade.stopLoss }}</p>
              <p>Profit Target: {{ trade.profitTarget }}</p>
              <p>Units: {{ trade.units }}</p>
              <p v-if="trade.trailingStopLoss">Trailing Stop Loss: {{ trade.trailingStopLoss }}</p>
              <p v-if="trade.sellPrice">Sell Price: {{ trade.sellPrice }}</p>
              <p v-if="trade.profitLoss">Profit/Loss: {{ trade.profitLoss }}</p>
            </div>
            <div class="trade-status">
              <i class="fas fa-circle" :class="{ 'text-success': trade.profitLoss > 0, 'text-danger': trade.profitLoss < 0, 'text-warning': trade.profitLoss === 0 }"></i>
              <span>{{ trade.profitLoss | formatProfitLoss }}</span>
            </div>
          </div>
          <div class="chart-container">
      <VueApexCharts :options="chartOptions" :series="chartSeries" type="line" height="200" />
    </div>
        </div>
      </div>
    </div>
  </template>
  
  <script>
  import axios from 'axios';
 // import net from 'net';
 import VueApexCharts from 'vue-apexcharts';
  import ccxt from 'ccxt';
  import * as technicalindicators from 'technicalindicators'
 
  const exchange = new ccxt.pro.binance({
          apiKey: 'IDSqM9d55lwHh3HmMWJBsulrj5ay3s7UKIMsObWpu7NlRDyK4FdEMEps2QlzcAGq',
          secret: '6d3yVZI23hCxQxmrCOtEaHHYomYTgPwbaN4YNFMlhebUTVx4QFGbtRSYZx81tEQs',
          proxy:'http://localhost:8081/'
        });
//         const intervalMap = {
//   1000: '1s',
//   60000: '1m',
//   180000: '3m',
//   300000: '5m',
//   1800000: '30m',
//   3600000: '1h',
//   21600000: '6h',
//   86400000: '1d',
// };
let tradingCapital = 50; // The total amount of capital that we have for trading
const riskPercentage = 0.2; // The maximum percentage of trading capital that can be risked per trade
const profitTargetPercentage = 0.2; // The profit target percentage for each trade
const stopLossPercentage = 0.03; // The stop loss percentage for each trade
//const interval = intervalMap[180000];
  export default {
    filters: {
    formatProfitLoss(profitLoss) {
      if (profitLoss > 0) {
        return `+${profitLoss.toFixed(2)}`;
      } else if (profitLoss < 0) {
        return profitLoss.toFixed(2);
      } else {
        return '0.00';
      }
    },
  },
    components: {
    VueApexCharts
  },
    name: 'HistoricalData',
    data() {
      return {
        symbolsData: [],
        scannedSymbols: [],
        loading: false,
        zoomLevel: 1,
      
      };
    },
    methods: {
        async checkSymbolsData() {
  try {
    const response = await axios.get('http://localhost:3000/api/trades');
    const trades = response.data;

    if (trades.length > 0) {
      this.symbolsData = trades;

      for (const trade of this.symbolsData) {
        const symbol = trade.symbol;
        const ticker = await exchange.fetchTicker(symbol);

        // Check if the current price has hit the profit target or stop loss
        if (ticker.last <= trade.stopLoss || ticker.last >= trade.profitTarget) {
          // Close the trade
          const sellOrder = await exchange.createMarketSellOrder(symbol, trade.units);
          // Update the trade details with the sell price
          trade.sellPrice = sellOrder.price;
          trade.profitLoss = (trade.sellPrice - trade.buyPrice) * trade.units;

          // Remove the trade from the list of open trades
          this.symbolsData.splice(this.symbolsData.indexOf(trade), 1);

          // Save the trade details to the database
          axios.post('/api/trades', trade);
        } else {
          // Check if the current price has crossed the trailing stop loss
          if (trade.trailingStopLoss && ticker.last <= trade.trailingStopLoss) {
            // Update the stop loss to the current price
            const updatedStopLoss = ticker.last - (ticker.last * stopLossPercentage);
            const updatedTrade = { ...trade, stopLoss: updatedStopLoss };

            // Replace the old trade details with the updated ones
            this.symbolsData[this.symbolsData.indexOf(trade)] = updatedTrade;
          } else {
            // Check if the current price has moved in favor of the trade
            const currentProfitLoss = (ticker.last - trade.buyPrice) * trade.units;
            const profitLossPercentage = (currentProfitLoss / (trade.buyPrice * trade.units)) * 100;

            if (profitLossPercentage > profitTargetPercentage) {
              // Update the profit target to the current price
              const updatedProfitTarget = ticker.last + (ticker.last * profitTargetPercentage);
              const updatedTrade = { ...trade, profitTarget: updatedProfitTarget };

              // Replace the old trade
              this.symbolsData[this.symbolsData.indexOf(trade)] = updatedTrade;
            } else {
              console.log(`Current price (${ticker.last}) is within acceptable range for symbol ${symbol}.`);
               // Check if the trade should be adjusted
            if (trade.type === 'buy') {
              await this.checkShouldAdjustBuy(trade, ticker);
            } else if (trade.type === 'sell') {
              await this.checkShouldAdjustSell(trade, ticker);
            } else {
              console.error(`Invalid trade type: ${trade.type}`);
            }
            }
          }
        
        }
      }
    }
  } catch (error) {
    console.error(error);
  }
},
async runBot() {
  this.loading = true;

  try {
    const symbols = await exchange.loadMarkets();
    const filteredSymbols = Object.keys(symbols).filter((symbol) => symbol.endsWith('BUSD'));
    const scannedSymbols = [];
    const balance = await exchange.fetchBalance();
    const symbolsData = [];
    let scanningFinished = false;
    let i = 0;

    while (!scanningFinished && i < filteredSymbols.length) {
      const symbol = filteredSymbols[i];

      if (!scannedSymbols.includes(symbol)) {
        // Calculate the necessary technical indicators for the symbol
        const ohlcv = await exchange.fetchOHLCV(symbol, '1m', undefined, 100);
        const candles = ohlcv.map(c => ({
            time: new Date(c[0]),
            open: c[1],
            high: c[2],
            low: c[3],
            close: c[4],
            volume: c[5],
        }));

const rsi = technicalindicators.RSI.calculate({ period: 14, values: candles.map((c) => c.close) });
const macd = technicalindicators.MACD.calculate({ values: candles.map((c) => c.close) });
const ema20 = technicalindicators.EMA.calculate({ period: 20, values: candles.map((c) => c.close) });
const ema50 = technicalindicators.EMA.calculate({ period: 50, values: candles.map((c) => c.close) });
const ema100 = technicalindicators.EMA.calculate({ period: 100, values: candles.map((c) => c.close) });
const latestCandle = candles[candles.length - 1];
const netVolatility = await this.getNetVolatility(symbol, latestCandle);

console.log('volume for'+symbol+' is '+netVolatility)
const ohlcv1 = await exchange.fetchOHLCV(symbol, '1m', undefined, 100);
const bearbull = ohlcv1 ? this.getBearBull(ohlcv1.slice(-14)) : 0;
const netBearBull = bearbull ? this.getNetBearBull(bearbull) : 0;      
const currentVolatility =  this.getVolatility(ohlcv1, latestCandle.close);
const Checkvolat =  this.Checkvolatility(symbol,latestCandle);
console.log(Checkvolat)
        // Check if the conditions for buying are met
        const shouldBuy = this.checkShouldBuy(symbol,rsi, macd, ema20, ema50, ema100, latestCandle, bearbull, netBearBull, riskPercentage, tradingCapital, currentVolatility);

        if (shouldBuy) {
          // Get the free balance of the asset we want to buy
          if (balance[symbol.slice(0, -4)] && balance[symbol.slice(0, -4)]['free']) {
            const freeBalance = balance[symbol.slice(0, -4)]['free'];

            // Calculate the amount to be risked for this trade
            const { price } = await exchange.fetchTicker(symbol);
            let amountToBeRisked;

            if (price > 0 && price < 1) {
              amountToBeRisked = freeBalance * 0.2; // 20% of free balance
            } else if (price >= 1 && price < 10) {
              amountToBeRisked = freeBalance * 0.1; // 10% of free balance
            } else {
              amountToBeRisked = freeBalance * 0.05; // 5% of free balance
            }

            // Calculate the trade parameters
            const units = amountToBeRisked / price;
const stopLoss = latestCandle[4] - (latestCandle[4] * stopLossPercentage);
const profitTarget = latestCandle[4] + (latestCandle[4] * profitTargetPercentage);

        // Add the trade to the list of open trades
        symbolsData.push({
          symbol,
          buyPrice: latestCandle[4],
          stopLoss,
          profitTarget,
          units,
          trailingStopLoss: undefined,
          sellPrice: undefined,
          profitLoss: undefined,
        });

        // Reduce the available trading capital by the amount to be risked
        tradingCapital -= amountToBeRisked;

        console.log(`Buying ${symbol} at ${latestCandle[4]}. Stop loss: ${stopLoss}. Profit target: ${profitTarget}. Units: ${units}.`);
      } else {
        console.log(`Insufficient balance to buy ${symbol}.`);
      }
    }

    scannedSymbols.push(symbol);
    this.scannedSymbols = scannedSymbols;
    console.log(scannedSymbols)
    if (scannedSymbols.length === filteredSymbols.length) {
      scanningFinished = true;
    }
  } else {
    console.log(`${symbol} has already been scanned.`);
  }

  i++;
}

console.log(symbolsData);

// Save the trade details to the database
await axios.post('http://localhost:3000/api/trades/bulk', { trades: symbolsData });

// Start checking the status of the trades

} catch (error) {
console.error(error);
}

this.loading = false;
},
zoomOut() {
      this.zoomLevel /= 2;
    },
    zoomIn() {
      this.zoomLevel *= 2;
    },
      async onClick() {
        this.runBot();
        const response = await axios.get('http://localhost:3000/api/trades');
    const trades = response.data;
        while (trades.length > 0) {
  await this.checkSymbolsData();
  await new Promise((resolve) => setTimeout(resolve, 5000));
}
      },
      async getNetVolatility(symbol, latestCandle) {
    try {
      const ohlcv = await exchange.fetchOHLCV(symbol, '3m', undefined, 100);
      const filteredData = ohlcv.filter((data) => data[0] > (Date.now() - (3 * 60 * 1000)));
      if (filteredData.length > 0) {
        const lastClose = filteredData[filteredData.length - 1][4];
        const netVolatility = (latestCandle[4] - lastClose) / lastClose;
        return netVolatility;
      }
      return 0;
    } catch (error) {
      console.error(error);
      return 0;
    }
},
      getBearBull(candles) {
        let bear = 0;
        let bull = 0;
        let sum = 0;
  
        for (let i = 0; i < candles.length; i++) {
          if (candles[i][1] > candles[i][4]) {
            bear += 1;
          } else {
            bull += 1;
          }
          sum += candles[i][2] - candles[i][3];
        }
  
        const bodySizeAverage = sum / candles.length;
        const bearBullRatio = bear / bull;
  
        return [bearBullRatio, bodySizeAverage];
      },
      getNetBearBull(bearBull) {
        const bearBullRatio = bearBull[0];
        const bodySizeAverage = bearBull
        const netBearBull = bearBullRatio / bodySizeAverage;
        return netBearBull;
      },
      getVolatility(ohlcv, latestClose) {
        const closes = ohlcv.map((c) => c[4]);
        const standardDeviation = technicalindicators.SD.calculate({ period: 14, values: closes });
        const volatility = (standardDeviation / latestClose) * 100;
        return volatility;
      },
      async checkShouldAdjustBuy(trade, ticker) {
  try {
    // Fetch the open position for the symbol
    const position = await exchange.fetchOpenOrders(trade.symbol).find(p => p.side === 'buy');

    if (position) {
      // Calculate the new stop loss andprofit target prices
const currentPrice = ticker.last;
const newStopLoss = currentPrice - (currentPrice * trade.stopLossPercentage / 100);
const newProfitTarget = currentPrice + (currentPrice * trade.profitTargetPercentage / 100);

  // Check if the stop loss or profit target prices need to be adjusted
  if (position.stopPrice !== newStopLoss || position.price !== newProfitTarget) {
    // Update the stop loss and profit target prices for the position
    await exchange.editOrder(position.id, trade.symbol, 'stop_loss_limit', 'sell', position.amount, newStopLoss, { stopPrice: newStopLoss });
    await exchange.editOrder(position.id, trade.symbol, 'limit', 'sell', position.amount, newProfitTarget, { timeInForce: 'GTC' });

    // Log the adjustment
    console.log(`Adjusted stop loss to ${newStopLoss} and profit target to ${newProfitTarget} for ${trade.symbol}.`);
  }
}
} catch (error) {
console.error(`Error checking if ${trade.symbol} should be adjusted: ${error}`);
}
},
async checkShouldAdjustSell(trade, ticker) {
try {
// Fetch the open position for the symbol
const position = await exchange.fetchOpenOrders(trade.symbol).find(p => p.side === 'sell');

if (position) {
  // Calculate the new stop loss and profit target prices
  const currentPrice = ticker.last;
  const newStopLoss = currentPrice + (currentPrice * trade.stopLossPercentage / 100);
  const newProfitTarget = currentPrice - (currentPrice * trade.profitTargetPercentage / 100);

  // Check if the stop loss or profit target prices need to be adjusted
  if (position.stopPrice !== newStopLoss || position.price !== newProfitTarget) {
    // Update the stop loss and profit target prices for the position
    await exchange.editOrder(position.id, trade.symbol, 'stop_loss_limit', 'buy', position.amount, newStopLoss, { stopPrice: newStopLoss });
    await exchange.editOrder(position.id, trade.symbol, 'limit', 'buy', position.amount, newProfitTarget, { timeInForce: 'GTC' });

    // Log the adjustment
    console.log(`Adjusted stop loss to ${newStopLoss} and profit target to ${newProfitTarget} for ${trade.symbol}.`);
  }
}
} catch (error) {
console.error(`Error checking if ${trade.symbol} should be adjusted: ${error}`);
}
},
      async  checkShouldAdjustBuy1(symbol, latestCandle, currentVolatility) {
  try {
    // Fetch the open positions for the symbol
    const positions = await exchange.fetchOpenOrders(symbol);
    const openPosition = positions.find(p => p.side === 'buy');

    if (openPosition) {
      // Calculate the new stop loss and take profit prices
      const newStopLossPrice = latestCandle.close - currentVolatility * 2;
      const newTakeProfitPrice = latestCandle.close + currentVolatility * 4;

      // Edit the stop loss and take profit orders for the open position
      await exchange.editOrder(openPosition.id, symbol, 'stop_loss_limit', 'sell', openPosition.amount, newStopLossPrice, { stopPrice: newStopLossPrice });
      await exchange.editOrder(openPosition.id, symbol, 'limit', 'sell', openPosition.amount, newTakeProfitPrice, { timeInForce: 'GTC' });

      // Log the adjustment
      console.log(`Adjusted stop loss to ${newStopLossPrice} and take profit to ${newTakeProfitPrice} for ${symbol}.`);
    } else {
      console.log(`No open position found for ${symbol}.`);
    }
  } catch (error) {
    console.error(`Error checking if ${symbol} should be adjusted: ${error}`);
  }
},
async  checkShouldAdjustSell1(symbol, latestCandle, currentVolatility) {
  try {
    // Fetch the open positions for the symbol
    const positions = await exchange.fetchOpenOrders(symbol);
    const openPosition = positions.find(p => p.side === 'sell');

    if (openPosition) {
      // Calculate the new stop loss and take profit prices
      const newStopLossPrice = latestCandle.close + currentVolatility * 2;
      const newTakeProfitPrice = latestCandle.close - currentVolatility * 4;

      // Edit the stop loss and take profit orders for the open position
      await exchange.editOrder(openPosition.id, symbol, 'stop_loss_limit', 'buy', openPosition.amount, newStopLossPrice, { stopPrice: newStopLossPrice });
      await exchange.editOrder(openPosition.id, symbol, 'limit', 'buy', openPosition.amount, newTakeProfitPrice, { timeInForce: 'GTC' });

      // Log the adjustment
      console.log(`Adjusted stop loss to ${newStopLossPrice} and take profit to ${newTakeProfitPrice} for ${symbol}.`);
    } else {
      console.log(`No open position found for ${symbol}.`);
    }
  } catch (error) {
    console.error(`Error checking if ${symbol} should be adjusted: ${error}`);
  }
}
,
      async  Checkvolatility(symbol,latestCandle){
    const volatility = latestCandle.high - latestCandle.low;
    console.log(`Volatility: ${volatility} ${symbol} and ${latestCandle.close * 0.01}`);
    let volt = "L'actif est suffisamment volatil pour le trading";
    if (volatility < latestCandle.close * 0.01) {
        console.log("L'actif n'est pas assez volatil pour le trading");
        volt = "L'actif n'est pas assez volatil pour le trading";
        return volt      
    }
    return volt
},
      checkShouldBuy(Symbol,rsi, macd, ema20, ema50, ema100, latestCandle, bearbull, netBearBull, riskPercentage, tradingCapital, currentVolatility) {
        // Check if RSI is below 30
        if (rsi[rsi.length - 1] < 30) {
          console.log(`RSI for ${Symbol} is ${rsi[rsi.length - 1]}.`);
  
          // Check if MACD histogram is increasing
          if (macd[macd.length - 1].histogram > macd[macd.length - 2].histogram) {
            console.log(`MACD histogram for ${Symbol} is increasing.`);
  
            // Check if the latest candle is above the 20, 50 and 100 EMA
            if (latestCandle[4] > ema20[ema20.length - 1] && latestCandle[4] > ema50[ema50.length - 1] && latestCandle[4] > ema100[ema100.length - 1]) {
              console.log(`Latest candle for ${Symbol} is above the 20, 50 and 100 EMA.`);
  
              // Check if the bear-bull ratio is in favor of bears
              if (bearbull[0] < 1) {
                console.log(`Bear-bull ratio for ${Symbol} is in favor of bears.`);
  
                // Check if the net bear-bull ratio is increasing
                if (netBearBull > 0 && netBearBull > 1) {
                  console.log(`Net bear-bull ratio for ${Symbol} is increasing.`);
  
                  // Check if the risk percentage is not higher than the limit
                  const price = latestCandle[4];
                  const potentialLoss = price * stopLossPercentage;
                  const potentialProfit = price * profitTargetPercentage;
                  const maxAmountToBeRisked = (tradingCapital * riskPercentage) / 100;
                  const potentialRisk = Math.min(potentialLoss, potentialProfit);
                  const potentialReward = Math.max(potentialLoss, potentialProfit);
                  const riskRewardRatio = potentialRisk / potentialReward;
  
                  if (maxAmountToBeRisked > potentialRisk && riskRewardRatio > 1) {
                    console.log(`Risk for ${latestCandle[Symbol]} is within acceptable limits. Risk: ${potentialRisk}. Reward: ${potentialReward}. Risk-reward ratio: ${riskRewardRatio}.`);
  
                    // Check if the volatility is within acceptable limits
                    if (currentVolatility < 1) {
                      console.log(`Volatility for ${latestCandle[Symbol]} is within acceptable limits.`);
  
                      return true;
                    } else {
                      console.log(`Volatility for ${latestCandle[Symbol]} is too high. Current volatility: ${currentVolatility}.`);
                    }
                  } else {
                 
                    console.log(`Risk for ${latestCandle[Symbol]} is too high. Max amount to be risked: ${maxAmountToBeRisked}. Potential risk: ${potentialRisk}. Potential reward: ${potentialReward}. Risk-reward ratio: ${riskRewardRatio}.`);
                  }
                } else {
                  console.log(`Net bear-bull ratio for ${latestCandle[Symbol]} is not increasing.`);
                }
              } else {
                console.log(`Bear-bull ratio for ${latestCandle[Symbol]} is not in favor of bears.`);
              }
            } else {
              console.log(`Latest candle for ${latestCandle[Symbol]} is not above the 20, 50 and 100 EMA.`);
            }
          } else {
            console.log(`MACD histogram for ${latestCandle[Symbol]} is not increasing.`);
          }
        } else {
          console.log(`RSI for ${latestCandle[Symbol]} is not below 30.`);
        }
  
        return false;
      },
    },
    computed: {
    chartSeries() {
      return this.symbolsData.map((trade) => {
        const data = trade.history.map((entry) => {
          return [new Date(entry.time).getTime(), entry.price];
        });
        return {
          name: trade.symbol,
          data: data.slice(-100 * this.zoomLevel),
        };
      });
    },
  chartOptions() {
  return {
    chart: {
      id: 'trade-chart',
      toolbar: {
        show: false,
      },
      zoom: {
        enabled: true,
        type: 'x',
        autoScaleYaxis: true,
      },
    },
    xaxis: {
      type: 'datetime',
      labels: {
        datetimeFormatter: {
          year: 'yyyy',
          month: 'MMM \'yy',
          day: 'dd MMM',
          hour: 'HH:mm',
        },
      },
    },
  };
},
  },
  };
  </script>
  <style>
.start-btn {
  background-color: #4CAF50;
  color: white;
  padding: 10px 20px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.start-btn:hover {
  background-color: #3e8e41;
}

</style>
<style scoped>
.open-trades {

margin-top: 20px;
}

.trade-list {
display: flex;
flex-wrap: wrap;
justify-content: space-between;
}

.trade-card {
position: relative;
width: 300px;
height: 200px;
margin-bottom: 20px;
border: 1px solid #ccc;
border-radius: 5px;
cursor: pointer;
transition: transform 0.3s ease;
}

.trade-card:hover {
transform: scale(1.1);
}

.trade-card-front {
display: flex;
flex-direction: column;
align-items: center;
justify-content: center;
height: 100%;
padding: 20px;
}

.trade-card-back {
position: absolute;
top: 0;
left: 0;
width: 100%;
height: 100%;
transform: rotateY(180deg);
backface-visibility: hidden;
display: flex;
flex-direction: column;
align-items: center;
justify-content: center;
padding: 20px;
}

.trade-card-back p:not(:last-child) {
margin-bottom: 10px;
}

.profit {
background-color: #d8f7d9;
border-color: #50c450;
}

.loss {
background-color: #f6cfcf;
border-color: #f44336;
}

.breakeven {
background-color: #fff0c6;
border-color: #ffeb3b;
}

.trade-chart {
margin-top: 10px;
width: 100%;
height: 200px;
}

.chart-header {
display: flex;
justify-content: space-between;
align-items: center;
margin-bottom: 10px;
}

.chart-zoom {
display: flex;
gap: 10px;
}

@media (max-width: 768px) {
.trade-card {
width: 100%;
height: 250px;
}
}
</style> 
  
