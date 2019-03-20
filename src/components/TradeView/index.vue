<template>
  <div id="trade-view">
  </div>
</template>

<script>
import { widget as TvWidget } from '../../../static/tradeview/charting_library/charting_library.min.js'
import socket from './datafeeds/socket.js'
import datafeeds from './datafeeds/datafees.js'
export default {
  data() {
    return {
      widget: null,
      socket: new socket(),
      datafeeds: new datafeeds(this),
      symbol: null,
      interval: null,
      cacheData: {},
      lastTime: null,
      getBarTimer: null,
      isLoading: true
    }
  },
  created() {
    // 开启websocket
    this.socket.doOpen()
    this.socket.on('open', () => {
      this.socket.send({ cmd: 'req', args: [`candle.M5.btcusdt}`, 1440, parseInt(Date.now() / 1000)] })
    })
    this.socket.on('message', this.onMessage)
  },
  methods: {
    // 初始化tradingview
    init(symbol = 'BTCUSDT', interval = 5) {
      if (!this.widget) {
        this.widget = new TvWidget({
          symbol: symbol,
          interval: interval,
          fullscreen: true,
          container_id: 'trade-view',
          datafeed: this.datafeeds,
          library_path: '/static/tradeview/charting_library/',
          disabled_features: ['header_symbol_search'],
          enabled_features: [],
          timezone: 'Asia/Shanghai',
          locale: 'zh',
          debug: false
        })
        this.symbol = symbol
        this.interval = interval
      }
    },
    // 发送websocket信息
    sendMessage(data) {
      if (this.socket.checkOpen()) {
        this.socket.send(data)
      } else {
        this.socket.on('open', () => {
          this.socket.send(data)
        })
      }
    },
    // 取消订阅
    unSubscribe(interval) {
      if (interval < 60) {
        this.sendMessage({ cmd: 'unsub', args: [`candle.M${interval}.${this.symbol.toLowerCase()}`, 1440, parseInt(Date.now() / 1000)] })
      } else if (interval >= 60) {
        this.sendMessage({ cmd: 'unsub', args: [`candle.H${interval / 60}.${this.symbol.toLowerCase()}`, 1440, parseInt(Date.now() / 1000)] })
      } else {
        this.sendMessage({ cmd: 'unsub', args: [`candle.D1.${this.symbol.toLowerCase()}`, 207, parseInt(Date.now() / 1000)] })
      }
    },
    // 订阅K线实时数据
    subscribe() {
      if (this.interval < 60) {
        this.sendMessage({ cmd: 'sub', args: [`candle.M${this.interval}.${this.symbol.toLowerCase()}`] })
      } else if (this.interval >= 60) {
        this.sendMessage({ cmd: 'sub', args: [`candle.H${this.interval / 60}.${this.symbol.toLowerCase()}`] })
      } else {
        this.sendMessage({ cmd: 'sub', args: [`candle.D1.${this.symbol.toLowerCase()}`] })
      }
    },
    // webscoket返回数据
    onMessage(data) {
      console.log('onMessage data: ' + data)
      // 返回数据存在历史数据(历史数据)
      if (data.data && data.data.length) {
        const list = []
        const ticker = `${this.symbol}-${this.interval}`
        data.data.forEach(function (element) {
          list.push({
            time: this.interval !== 'D' || this.interval !== '1D' ? element.id * 1000 : element.id,
            open: element.open,
            high: element.high,
            low: element.low,
            close: element.close,
            volume: element.quote_vol
          })
        }, this)
        this.cacheData[ticker] = list
        this.lastTime = list[list.length - 1].time
        this.subscribe()
        console.log(list)
      }
      // 返回数据类型为(订阅数据)
      if (data.type && data.type.indexOf(this.symbol.toLowerCase()) !== -1) {
        console.log(' >> sub:', data.type)
        this.datafeeds.barsUpdater.updateData()
        const ticker = `${this.symbol}-${this.interval}`
        const barsData = {
          time: data.id * 1000,
          open: data.open,
          high: data.high,
          low: data.low,
          close: data.close,
          volume: data.quote_vol
        }
        if (barsData.time >= this.lastTime && this.cacheData[ticker] && this.cacheData[ticker].length) {
          this.cacheData[ticker][this.cacheData[ticker].length - 1] = barsData
        }
      }
    },
    // 获取历史数据
    getBars(symbolInfo, resolution, rangeStartDate, rangeEndDate, onLoadedCallback) {
      console.log(' getBars >> :', new Date(rangeStartDate * 1000).toLocaleString() , new Date(rangeEndDate * 1000).toLocaleString())
      if (this.interval !== resolution) {
        this.unSubscribe(this.interval)
        this.interval = resolution
        if (resolution < 60) {
          this.sendMessage({ cmd: 'req', args: [`candle.M${this.interval}.${this.symbol.toLowerCase()}`, 1440, parseInt(Date.now() / 1000)] })
        } else if (resolution >= 60) {
          this.sendMessage({ cmd: 'req', args: [`candle.H${this.interval / 60}.${this.symbol.toLowerCase()}`, 1440, parseInt(Date.now() / 1000)] })
        } else {
          this.sendMessage({ cmd: 'req', args: [`candle.D1.${this.symbol.toLowerCase()}`, 800, parseInt(Date.now() / 1000)] })
        }
      }
      const ticker = `${this.symbol}-${this.interval}`
      if (this.cacheData[ticker] && this.cacheData[ticker].length) {
        this.isLoading = false
        const newBars = []
        this.cacheData[ticker].forEach(item => {
          if (item.time >= rangeStartDate * 1000 && item.time <= rangeEndDate * 1000) {
            newBars.push(item)
          }
        })
        onLoadedCallback(newBars)
      } else {
        const self = this
        this.getBarTimer = setTimeout(function () {
          self.getBars(symbolInfo, resolution, rangeStartDate, rangeEndDate, onLoadedCallback)
        }, 10)
      }
    }
  }
}
</script>

