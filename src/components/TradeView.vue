<template>
    <div class="trade-view" id="trade-view" />
</template>

<script>
import { widget } from '../assets/js/charting_library.min';
import socket from '../assets/js/datafeeds/socket';
import datafeeds from '../assets/js/datafeeds/datafees';

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
      isLoading: true,
      containerId: {
        default: 'tv_chart_container',
        type: String,
      }
    }
  },
  created() {
    this.socket.doOpen()
    this.socket.on('open', () => {
      //unsub
      //{"args":["candle.M5.BTC_USDT"],"cmd":"sub","bindGroup":"sub","id":"A9827374-AFCE-4CFB-9E5A-2B40849AB9E3"}
      //this.socket.send({ cmd: 'req', args: [`candle.M5.BTC_USDT}`, 1440, parseInt(Date.now() / 1000)] })
      //this.socket.send({"args":["candle.M5.BTC_USDT"],"cmd":"sub","bindGroup":"sub","id":"A9827374-AFCE-4CFB-9E5A-2B40849AB9E3"})
      this.socket.send({"args":["candle.M5.BTC_USDT",1441,1555315107],"cmd":"req","id":"A9827374-AFCE-4CFB-9E5A-2B40849AB9E3"})
    })
    this.socket.on('message', this.onMessage)
  },
  methods: {
    init(symbol = 'BTC_USDT', interval = 5) {
      if (!this.widget) {
        this.widget = new widget({
          symbol: symbol,
          interval: interval,
          fullscreen: true,
          container_id: 'trade-view',
          datafeed: this.datafeeds,
          library_path: '/static/charting_library/',
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
    sendMessage(data) {
      if (this.socket.checkOpen()) {
        this.socket.send(data)
      } else {
        this.socket.on('open', () => {
          this.socket.send(data)
        })
      }
    },
    unSubscribe(interval) {
      // if (interval < 60) {
      //   this.sendMessage({ cmd: 'unsub', args: [`candle.M${interval}.${this.symbol.toUpperCase()}`, 1440, parseInt(Date.now() / 1000)] })
      // } else if (interval >= 60) {
      //   this.sendMessage({ cmd: 'unsub', args: [`candle.H${interval / 60}.${this.symbol.toUpperCase()}`, 1440, parseInt(Date.now() / 1000)] })
      // } else {
      //   this.sendMessage({ cmd: 'unsub', args: [`candle.D1.${this.symbol.toUpperCase()}`, 207, parseInt(Date.now() / 1000)] })
      // }
    },
    subscribe() {
      if (this.interval < 60) {
        this.sendMessage({ cmd: 'sub', args: [`candle.M${this.interval}.${this.symbol.toUpperCase()}`] })
      } else if (this.interval >= 60) {
        this.sendMessage({ cmd: 'sub', args: [`candle.H${this.interval / 60}.${this.symbol.toUpperCase()}`] })
      } else {
        this.sendMessage({ cmd: 'sub', args: [`candle.D1.${this.symbol.toUpperCase()}`] })
      }
    },
    onMessage(data) {
      // console.log(data)
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
      }
      if (data.type && data.type.indexOf(this.symbol.toUpperCase()) !== -1) {
        // console.log(' >> sub:', data.type)
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
    getBars(symbolInfo, resolution, rangeStartDate, rangeEndDate, onLoadedCallback) {
      // console.log(' >> :', rangeStartDate, rangeEndDate)
      if (this.interval !== resolution) {
        this.unSubscribe(this.interval)
        this.interval = resolution
        if (resolution < 60) {
          this.sendMessage({ cmd: 'req', args: [`candle.M${this.interval}.${this.symbol.toUpperCase()}`, 1440, parseInt(Date.now() / 1000)] })
        } else if (resolution >= 60) {
          this.sendMessage({ cmd: 'req', args: [`candle.H${this.interval / 60}.${this.symbol.toUpperCase()}`, 1440, parseInt(Date.now() / 1000)] })
        } else {
          this.sendMessage({ cmd: 'req', args: [`candle.D1.${this.symbol.toUpperCase()}`, 800, parseInt(Date.now() / 1000)] })
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

<style scoped>
    #trade-view {
        height: calc(100vh - 80px);
    }
</style>

