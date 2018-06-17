<template>
  <v-app>
    <v-toolbar app color='primary'>
      <v-toolbar-title v-text="title"></v-toolbar-title>
      <v-spacer></v-spacer>
      <v-btn icon @click.stop="updateBalance()">
        <v-icon>refresh</v-icon>
      </v-btn>
    </v-toolbar>
    <v-dialog v-model="progress" persistent width="200">
      <v-layout class="loading" row justify-center>
        <div class="progress-area">
          <v-progress-circular
            :size="100"
            :width="15"
            :rotate="-90"
            :value="percentage"
            color="primary"
          >
            {{ percentage }}
          </v-progress-circular>
        </div>
      </v-layout>
    </v-dialog>
    <v-content v-if="loaded">
      <v-layout row>
        <v-flex xs12 sm6 offset-sm3>
          <v-card>
            <v-list three-line subheader>
              <v-list-tile>
                <v-list-tile-content>
                  <v-list-tile-title>TOTAL: {{ total }} BTC</v-list-tile-title>
                </v-list-tile-content>
              </v-list-tile>
              <v-divider></v-divider>
              <v-list-tile
                v-for="item in items"
                :key="item.title"
                avatar
              >
                <v-list-tile-avatar tile>
                  <img :src="item.icon">
                </v-list-tile-avatar>
                <v-list-tile-content>
                  <v-list-tile-title v-if="item.currency === 'BTC'">{{ balances[item.currency] }} {{ item.currency }}</v-list-tile-title>
                  <v-list-tile-title v-else>{{ balances[item.currency] }} {{ item.currency }} ({{ fiats[item.currency] }} BTC)</v-list-tile-title>
                  <v-list-tile-sub-title>
                    <a :href="item.insight_url" v-text="item.address"></a>
                  </v-list-tile-sub-title>
                </v-list-tile-content>
              </v-list-tile>
            </v-list>
          </v-card>
        </v-flex>
      </v-layout>
    </v-content>
  </v-app>
</template>

<style>
.loading {
  background-color: rgba(255,0,0,0);
}

.progress-area {
  background-color: rgba(255,0,0,0);
}
.progress-area .progress-circular {
  position: absolute;
}
</style>

<script>
import axios from 'axios'
import BigNumber from 'bignumber.js'

const TICKER_URL = 'https://api.coinmarketcap.com/v2/ticker/'
const API = {
  BTC: 'https://btc.blockdozer.com/insight-api',
  LTC: 'https://insight.litecore.io/api',
  BCH: 'https://bch-insight.bitpay.com/api',
  MONA: 'https://mona.insight.monaco-ex.org/insight-api-monacoin'
}

export default {
  data () {
    return {
      loaded: false,
      percentage: 0,
      items: [
        {
          currency: 'BTC',
          icon: require('./assets/btc.png'),
          ticker_id: 1,
          address: '1L52VzaxC8gDngQhcjvNAk1ywRsL49bG1Y',
          insight_url: 'https://btc.blockdozer.com/address/1L52VzaxC8gDngQhcjvNAk1ywRsL49bG1Y'
        },
        {
          currency: 'LTC',
          icon: require('./assets/ltc.png'),
          ticker_id: 2,
          address: 'LiR9R445oCBBi6KgiLz9UePm62PmtSQ75T',
          insight_url: 'https://insight.litecore.io/address/LiR9R445oCBBi6KgiLz9UePm62PmtSQ75T'
        },
        {
          currency: 'BCH',
          icon: require('./assets/bch.png'),
          ticker_id: 1831,
          address: 'qzkt7dnjtwgg9wyxg0ep9wcsdnhtnrwssgtaj8rhzk',
          insight_url: 'https://bch-insight.bitpay.com/address/qzkt7dnjtwgg9wyxg0ep9wcsdnhtnrwssgtaj8rhzk'
        },
        {
          currency: 'MONA',
          icon: require('./assets/mona.png'),
          ticker_id: 213,
          address: 'MQQKjnTW9Ea4TXtxwuyX38PVeHaioLDWX4',
          insight_url: 'https://mona.insight.monaco-ex.org/insight/address/MQQKjnTW9Ea4TXtxwuyX38PVeHaioLDWX4'
        }
      ],
      balances: {
        BTC: 0,
        LTC: 0,
        BCH: 0,
        MONA: 0
      },
      fiatRates: {
        BTC: 0,
        LTC: 0,
        BCH: 0,
        MONA: 0
      },
      fiats: {
        BTC: 0,
        LTC: 0,
        BCH: 0,
        MONA: 0
      },
      total: 0,
      title: 'KOTO Unofficial Donation Results'
    }
  },
  computed: {
    progress: {
      get () {
        return !this.loaded
      }
    }
  },
  mounted: function () {
    this.updateBalance()
  },
  methods: {
    updateBalance: function () {
      this.percentage = 0
      this.loaded = false
      let fiatList = []
      Promise.all(this.items.map(item => this.getFiatRate(item.ticker_id))).then(results => {
        fiatList = results
        return Promise.all(this.items.map(item => this.getBalance(item.currency, item.address)))
      }).then(results => {
        results.forEach((result, i) => {
          let fiat = new BigNumber(fiatList[i])
          this.balances[this.items[i].currency] = result.balance
          this.fiatRates[this.items[i].currency] = fiatList[i]
          this.fiats[this.items[i].currency] = fiat.multipliedBy(result.balance).decimalPlaces(8).toNumber()
        })
        this.total = Object.values(this.fiats).map(fiat => new BigNumber(fiat)).reduce((acc, cur) => acc.plus(cur), new BigNumber(0)).decimalPlaces(8).toNumber()
        this.loaded = true
        this.percentage = 100
      })
    },
    getFiatRate: function (ticker_id) {
      let uri = `${TICKER_URL}${ticker_id}/?convert=BTC`
      return axios.get(uri).then(resp => {
        let quote = resp.data.data.quotes['BTC']
        if (quote === null) {
          return Promise.reject(new Error(`${ticker_id} does'nt supported`))
        } else {
          this.percentage = this.percentage + 12
          return Promise.resolve(quote.price)
        }
      }).catch(error => {
        console.error(error)
      })
    },
    getBalance: function (currency, addr) {
      let uri = `${API[currency]}/addr/${addr}`
      return axios.get(uri).then(resp => {
        this.percentage = this.percentage + 12
        return Promise.resolve(resp.data)
      }).catch(error => {
        console.error(error)
      })
    }
  }
}
</script>
