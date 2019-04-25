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
                  <v-list-tile-title v-if="balances[item.currency] === -1">Insight API Error</v-list-tile-title>
                  <v-list-tile-title v-else-if="item.currency === 'BTC(segwit)'">{{ balances[item.currency] }} BTC</v-list-tile-title>
                  <v-list-tile-title v-else-if="item.currency === 'BTC'">{{ balances[item.currency] }} {{ item.currency }}</v-list-tile-title>
                  <v-list-tile-title v-else>{{ balances[item.currency] }} {{ item.currency }} ({{ fiats[item.currency] }} BTC)</v-list-tile-title>
                  <v-list-tile-sub-title>
                    <a :href="item.explorer_url" v-text="item.address"></a>
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

const TICKER_URL = 'https://api.coingecko.com/api/v3/simple/price'
const API = {
  'BTC(segwit)': [
    'https://chain.so/api/v2/address/BTC/'
  ],
  BTC: [
    'https://insight.bitpay.com/api',
    'https://btc.blockdozer.com/insight-api',
    'https://blockexplorer.com/api',
    'https://www.localbitcoinschain.com/api',
    'https://explorer.bitcoin.com/api/btc'
  ],
  'LTC(segwit)': [
    'https://chain.so/api/v2/address/LTC/'
  ],
  LTC: [
    'https://insight.litecore.io/api',
    'https://ltc-bitcore1.trezor.io/api'
  ],
  BCH: [
    'https://bch-insight.bitpay.com/api',
    'https://blockdozer.com/insight-api',
    'https://bitcoincash.blockexplorer.com/api',
    'https://explorer.bitcoin.com/api/bch'
  ],
  MONA: [
    'https://mona.insight.monaco-ex.org/insight-api-monacoin',
    'https://mona.insight.monacoin.ml/insight-api-monacoin'
  ],
  KOTO: [
    'https://insight.kotocoin.info/api'
  ],
  ZNY: [
    'https://zeny.insight.monaco-ex.org/api',
    'https://insight.bitzeny.jp/api',
    'https://zenyinsight.tomotomo9696.xyz/api'
  ]
}

export default {
  data () {
    return {
      loaded: false,
      percentage: 0,
      items: [
        {
          currency: 'BTC(segwit)',
          icon: require('./assets/btc.png'),
          ticker_id: 'bitcoin',
          address: 'bc1qx6ns2h2qplyyw4uap45hyae09jlseyrun2yjs9',
          explorer_url: 'https://chain.so/address/BTC/bc1qx6ns2h2qplyyw4uap45hyae09jlseyrun2yjs9'
        },
        {
          currency: 'BTC',
          icon: require('./assets/btc.png'),
          ticker_id: 'bitcoin',
          address: '1L52VzaxC8gDngQhcjvNAk1ywRsL49bG1Y',
          explorer_url: 'https://btc.blockdozer.com/address/1L52VzaxC8gDngQhcjvNAk1ywRsL49bG1Y'
        },
        {
          currency: 'LTC(segwit)',
          icon: require('./assets/ltc.png'),
          ticker_id: 'litecoin',
          address: 'ltc1qf9yws768xg0ghrxu69ud597u99e6vfkml9qsuw',
          explorer_url: 'https://chain.so/address/LTC/ltc1qf9yws768xg0ghrxu69ud597u99e6vfkml9qsuw'
        },
        {
          currency: 'LTC',
          icon: require('./assets/ltc.png'),
          ticker_id: 'litecoin',
          address: 'LiR9R445oCBBi6KgiLz9UePm62PmtSQ75T',
          explorer_url: 'https://insight.litecore.io/address/LiR9R445oCBBi6KgiLz9UePm62PmtSQ75T'
        },
        {
          currency: 'BCH',
          icon: require('./assets/bch.png'),
          ticker_id: 'bitcoin-cash',
          address: 'qzkt7dnjtwgg9wyxg0ep9wcsdnhtnrwssgtaj8rhzk',
          explorer_url: 'https://bch-insight.bitpay.com/#/address/qzkt7dnjtwgg9wyxg0ep9wcsdnhtnrwssgtaj8rhzk'
        },
        {
          currency: 'MONA',
          icon: require('./assets/mona.png'),
          ticker_id: 'monacoin',
          address: 'MQQKjnTW9Ea4TXtxwuyX38PVeHaioLDWX4',
          explorer_url: 'https://mona.insight.monaco-ex.org/insight/address/MQQKjnTW9Ea4TXtxwuyX38PVeHaioLDWX4'
        },
        {
          currency: 'KOTO',
          icon: require('./assets/koto.png'),
          ticker_id: 'koto',
          address: 'k1Hq88YVhnLo4FqrCCzoJBGam9tQGUCQ1Wi',
          explorer_url: 'https://insight.kotocoin.info/address/k1Hq88YVhnLo4FqrCCzoJBGam9tQGUCQ1Wi'
        },
        {
          currency: 'ZNY',
          icon: require('./assets/zny.png'),
          ticker_id: 'bitzeny',
          address: 'ZdD3J465cy47RWyRz6S9za6wybFTrJ6FSi',
          explorer_url: 'https://zenyinsight.tomotomo9696.xyz/address/ZdD3J465cy47RWyRz6S9za6wybFTrJ6FSi'
        }
      ],
      balances: {
        'BTC(segwit)': 0,
        BTC: 0,
        'LTC(segwit)': 0,
        LTC: 0,
        BCH: 0,
        MONA: 0,
        ZNY: 0
      },
      fiatRates: {
        'BTC(segwit)': 0,
        BTC: 0,
        'LTC(segwit)': 0,
        LTC: 0,
        BCH: 0,
        MONA: 0,
        ZNY: 0
      },
      fiats: {
        'BTC(segwit)': 0,
        BTC: 0,
        'LTC(segwit)': 0,
        LTC: 0,
        BCH: 0,
        MONA: 0,
        ZNY: 0
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
    updateBalance: async function () {
      this.percentage = 0
      this.loaded = false
      let list = await Promise.all(this.items.map(async item => {
        const rate = await this.getFiatRate(item.ticker_id)
        const balance = await this.getBalance(item.currency, item.address)
        return [rate, balance]
      }))
      list.forEach((pair, i) => {
        let fiat = new BigNumber(pair[0])
        this.balances[this.items[i].currency] = pair[1]
        this.fiatRates[this.items[i].currency] = pair[0]
        if (pair[1] > -1) {
          this.fiats[this.items[i].currency] = fiat.multipliedBy(pair[1]).decimalPlaces(8).toNumber()
        } else {
          this.fiats[this.items[i].currency] = 0
        }
      })
      this.total = Object.values(this.fiats).map(fiat => new BigNumber(fiat)).reduce((acc, cur) => acc.plus(cur), new BigNumber(0)).decimalPlaces(8).toNumber()
      this.loaded = true
      this.percentage = 100
    },
    getFiatRate: async function (ticker_id) {
      let uri = `${TICKER_URL}?ids=${ticker_id}&vs_currencies=btc`
      let resp = await axios.get(uri)
      let price = resp.data[ticker_id]
      if (price === null || price === undefined) {
        return new Error(`${ticker_id} does'nt supported`)
      } else {
        this.percentage = this.percentage + 7
        return price.btc
      }
    },
    getBalance: async function (currency, addr, n = 0) {
      let base_uri = API[currency][n]
      if (base_uri === undefined) {
        this.percentage = this.percentage + 7
        return -1
      }
      if (base_uri.match(/chain.so/)) {
        let uri = `${base_uri}/${addr}`
        try {
          let resp = await axios.get(uri, {timeout: 3000})
          this.percentage = this.percentage + 7
          return (new BigNumber(resp.data.data.balance)).toNumber()
        } catch (error) {
          console.error(e)
          return this.getBalance(currency, addr, n + 1)
        }
      } else {
        let uri = `${API[currency][n]}/addr/${addr}`
        try {
          let resp = await axios.get(uri, {timeout: 3000})
          this.percentage = this.percentage + 7
          return (new BigNumber(resp.data.balance)).toNumber()
        } catch(error) {
          console.error(error)
          return this.getBalance(currency, addr, n + 1)
        }
      }
    }
  }
}
</script>
