<template>
  <!-- <router-link to="/bridge-veth">bridge.veth link</router-link> -->
  <div class="p-2">
    Dark/Light <input type="checkbox" value="garden" class="toggle theme-controller" />
  </div>
  <div class="p-2">
    These are the VRSC mainnet raw verus trading (<a class="link link-info" target="_blank" href="https://raw.verus.trading">rvt</a>) tables of basket currencies parsed from getcurrency rpc (<a class="link link-info" target="_blank" href="https://github.com/BuildWithDreams/rvt">GitHub Repo</a>).
  </div>
  <div class="p-2">
    Staking => {{ stakingsupply["vrsc"] }}

  </div>
  <div class="p-2">Binance prices: BTCUSD: {{ parseFloat(binance_btcusd).toFixed(2) }} , ETHUSD: {{
    parseFloat(binance_ethusd).toFixed(2) }} , ETHBTC: {{
      parseFloat(binance_ethbtc).toFixed(6) }}</div>

  <div class="tabs tabs-border">
    <input type="radio" name="my_tabs_2" class="tab" aria-label="Mainnet" checked="checked" />
    <div class="tab-content border-base-300 bg-base-100 p-10">
      <div class="mb-4">
        <BasketSettings 
          :baskets="baskets" 
          :currentSettings="basketSettings"
          @settings-changed="handleSettingsChanged"
        />
      </div>
      
      <VerusBasket v-for="basket of filteredAndOrderedBaskets" :key="basket.currencyid" v-bind:fullyQualifiedName="basket.ticker" v-bind:currencyid="basket.currencyid" v-bind:webLink="basket.website"
        v-bind:chartLink="basket.chart" v-bind:recentTransfersLink="basket.recenttransfers"
        v-bind:marketNote="basket.marketnote" v-bind:explorerLink="basket.explorer" v-bind:supply="basket.supply"
        v-bind:bestHeight="basket.bestheight" v-bind:reserveCurrencies="basket.reservecurrencies"
        v-bind:currencyDictionary="currencyDictionary" v-bind:isExtrasOverride="false" v-bind:isPreconvert="basket.isPreconvert" />


    </div>
  </div>
</template>

<script>
import axios from 'axios';
import { ref } from 'vue';
import VerusBasket from './VerusBasket.vue'
import PriceInTbtc from './PriceInTbtc.vue'
import BasketSettings from './BasketSettings.vue'


export default {
  components: {
    VerusBasket,
    PriceInTbtc,
    BasketSettings
  },
  data() {
    return {
      chains: ['vrsc'],
      explorertx: "https://insight.verus.io/tx/",
      veruslatestblock: ref(),
      veruslongestchain: ref(),
      verusSyncOK: ref(false),
      verusBlocksRemaining: ref(0),


      pureTbtcVrsc: ref(),
      res: ref([]),
      binance_btcusd: ref(),
      binance_ethusd: ref(),
      binance_mkrusd: ref(),
      binance_ethbtc: ref(),
      binance_mkrbtc: ref(),
      stakingsupply: ref([]),
      vrsc_staking: ref(),
      varrr_staking: ref(),
      vdex_staking: ref(),
      chips_staking: ref(),
      currencyDictionary: [],
      baskets: ref([]),
      basketSettings: {}
    };
  },
  computed: {
    filteredAndOrderedBaskets() {
      // Apply settings: filter out hidden baskets and sort by order
      const settingsArray = Object.entries(this.basketSettings).map(([currencyid, setting]) => ({
        currencyid,
        ...setting
      }));

      // Create a map for quick lookup
      const settingsMap = new Map(settingsArray.map(s => [s.currencyid, s]));

      return this.baskets
        .filter(basket => {
          const setting = settingsMap.get(basket.currencyid);
          return setting ? setting.visible !== false : true; // Default to visible
        })
        .sort((a, b) => {
          const settingA = settingsMap.get(a.currencyid);
          const settingB = settingsMap.get(b.currencyid);
          const orderA = settingA?.order !== undefined ? settingA.order : 999;
          const orderB = settingB?.order !== undefined ? settingB.order : 999;
          return orderA - orderB;
        });
    }
  },
  methods: {
    isExtras() {
      console.log(import.meta.env.VITE_EXTRAS)
      return parseInt(import.meta.env.VITE_EXTRAS)
    },
    prettySupply(lp) {
      if (lp === this.BRIDGEVETH) {
        return this.responseBridgeVethBestCurrencyState.supply.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",")
      }
    },
    clear(lp) {
      if (lp === this.PURE) {
        this.operationsPure = []
        this.relativeOperationsPure = []
      }
      else if (lp === this.SWITCH) {
        console.log(this.SWITCH)
      }
      else if (lp === this.BRIDGEVETH) {
        this.operationsBridgeVethSend = []
        this.relativeOperationsBridgeVethSend = []
        this.operationsBridgeVethReceive = []
        this.relativeOperationsBridgeVethReceive = []
        this.receiveMessage = ''
      }
      else if (lp === this.BRIDGEVARRR) {
        this.operationsBridgeVarrr = []
        this.relativeOperationsBridgeVarrr = []
      }
    },
    getCurrencyTicker(lpcurrencies, currency) {
      return lpcurrencies.find(item => item.currencyid === currency.currencyid).ticker
    },
    getCellClassSwitch(currencyBase, currencyRel) {
      if (currencyBase.currencyid === currencyRel.currencyid) {
        return ''
      }

      if (currencyRel === this.SWITCH) {
        return this.operationsSwitch[currencyBase.currencyid] === 'add' ? 'light-red' : this.operationsSwitch[currencyBase.currencyid] === 'remove' ? 'light-green' : ''
      }

      // the row is the base currency, it devalues when 
      if (this.operationsSwitch[currencyBase.currencyid]) {
        if (this.relativeOperationsSwitch[currencyRel.currencyid] != '') {
          return this.relativeOperationsSwitch[currencyRel.currencyid] === 'add' ? 'light-green' : this.relativeOperationsSwitch[currencyRel.currencyid] === 'remove' ? 'light-red' : ''
        }
      }
      // rel currency e.g. dai price of asset
      if (this.operationsSwitch[currencyRel.currencyid]) {
        return this.operationsSwitch[currencyRel.currencyid] === 'add' ? 'light-green' : this.operationsSwitch[currencyRel.currencyid] === 'remove' ? 'light-red' : ''
      }
    },
    getStakingSupply0(chain) {
      const requestData = {
        method: 'post',
        url: 'https://rpc.' + chain + '.komodefi.com',
        headers: { 'Content-Type': 'application/json' },
        data: {
          method: 'getmininginfo',
          params: [],
          id: 999
        }
      }
      // console.log(requestData)
      this.sendRequestRPC(requestData)
        .then((response) => {
          this.stakingsupply[chain] = Math.trunc(response.data.result.stakingsupply)
          // console.log(this.stakingsupply[chain])
        })
        .catch(async (error) => {
          console.log(`RPC for ${chain} failed, falling back to local file: ${error.message}`);
          this.stakingsupply[chain] = 'N/A';
          try {
            const response = await fetch(`/files/${chain}.mininginfo.json`);
            if (!response.ok) {
              throw new Error(`Failed to fetch local data for ${chain}: ${response.statusText}`);
            }
            const data = await response.json();
            this.stakingsupply[chain] = Math.trunc(data.stakingsupply);
          } catch (fetchError) {
            console.error(`Failed to fetch or parse local staking data for ${chain}:`, fetchError);
          }
        })
    },
    async getStakingSupply(chain) {
      try {
        const response = await fetch(`/files/${chain}.mininginfo.json`);
        if (!response.ok) {
          throw new Error(`Failed to fetch local data for ${chain}: ${response.statusText}`);
        }
        const data = await response.json();
        this.stakingsupply[chain] = Math.trunc(data.stakingsupply);
      } catch (fetchError) {
        console.error(`Failed to fetch or parse local staking data for ${chain}:`, fetchError);
        if (chain === 'vrsc') {
          const requestData = {
            method: 'post',
            url: 'https://rpc.vrsc.syncproof.net',
            headers: { 'Content-Type': 'application/json' },
            data: {
              method: 'getmininginfo',
              params: [],
              id: 999
            }
          }
          this.sendRequestRPC(requestData)
            .then((response) => {
              this.stakingsupply[chain] = Math.trunc(response.data.result.stakingsupply)
            })
            .catch((error) => {
              console.log(`RPC for ${chain} failed: ${error.message}`);
              this.stakingsupply[chain] = 'N/A';
            })
        } else {
          this.stakingsupply[chain] = 'N/A';
        }
      }
    },
    getLatestBlock() {
      const requestData = {
        method: 'post',
        url: 'https://rpc.vrsc.syncproof.net',
        headers: { 'Content-Type': 'application/json' },
        data: {
          method: 'getinfo',
          params: [],
          id: 3
        }
      };
      this.sendRequestRPC(requestData)
        .then((response) => {
          // console.log(response.data.result.blocks)
          // console.log(this.verusSyncOK)
          // return response.data.result.blocks
          this.veruslatestblock = response.data.result.blocks
          this.veruslongestchain = response.data.result.longestchain
          this.verusBlocksRemaining = this.veruslongestchain - this.veruslatestblock
          if (this.veruslatestblock == this.veruslongestchain) {
            this.verusSyncOK = true
            // console.log(this.verusSyncOK)

          }
        })
        .catch((error) => {
          this.veruslatestblock = error
        })
    },
    async sendRequestRPC(requestData) {
      console.log(requestData)
      try {
        const response = await axios({
          method: requestData.method,
          url: requestData.url,
          headers: {
            Accept: 'application/json',
            ...(requestData.headers || {})
          },
          data: requestData.data,
          responseType: 'text'
        });

        // Some RPC endpoints return JSON without a Content-Type header.
        const parsedData = typeof response.data === 'string'
          ? JSON.parse(response.data)
          : response.data;

        if (!parsedData || typeof parsedData !== 'object') {
          throw new Error('RPC response is not a valid JSON object');
        }

        return {
          ...response,
          data: parsedData
        };
      } catch (error) {
        console.log(error)
        // Re-throw with context without leaking sensitive URLs
        const shortUrl = requestData.url.replace(/^https?:\/\//, '').split('/')[0];
        if (error instanceof SyntaxError) {
          throw new Error(`RPC call to ${shortUrl} returned non-JSON content`);
        }
        throw new Error(`RPC call to ${shortUrl} failed: ${error.message}`);
      }
    },
    async getCurrencyDetails(rpcUrl, currencyid, useRpc = true) {
      if (useRpc) {
        try {
          const requestData = {
            method: 'post',
            url: rpcUrl,
            headers: { 'Content-Type': 'application/json' },
            data: {
              method: 'getcurrency',
              params: [currencyid],
              id: 1
            }
          };
          const response = await this.sendRequestRPC(requestData)
          console.log(response)
          const result = response.data.result
          return {
            reservecurrencies: result.bestcurrencystate.reservecurrencies,
            bestheight: result.bestheight,
            supply: result.bestcurrencystate.supply
          }
        } catch (error) {
          console.error(`RPC getcurrency failed for ${currencyid}:`, error.message);
          return null;
        }
      }
      else {
        try {
          const response = await fetch(`/files/${currencyid}.json`);
          if (!response.ok) {
            throw new Error(`Failed to fetch local data for ${currencyid}: ${response.statusText}`);
          }
          const data = await response.json();
          const result = data;
          return {
            reservecurrencies: result.bestcurrencystate.reservecurrencies,
            bestheight: result.bestheight,
            supply: result.bestcurrencystate.supply,
          };
        } catch (error) {
          console.error(error);
          return null;
        }
      }
    },
    // getPureCurrency() {
    //   const requestData = {
    //     method: 'post',
    //     url: 'https://rpc.vrsc.komodefi.com',
    //     headers: { 'Content-Type': 'application/json' },
    //     data: {
    //       method: 'getcurrency',
    //       params: ['Pure'],
    //       id: 1
    //     }
    //   };
    //   this.sendRequestRPC(requestData)
    //     .then((response) => {
    //       this.responsePureBestCurrencyState = response.data.result.bestcurrencystate
    //       this.purereservecurrencies = response.data.result.bestcurrencystate.reservecurrencies
    //       this.purebestheight = response.data.result.bestheight
    //       this.puresupply = response.data.result.bestcurrencystate.supply
    //       const vrscReserves = this.purereservecurrencies.find(item => item.currencyid == "i5w5MuNik5NtLcYmNzcvaoixooEebB6MGV").reserves
    //       const tbtcReserves = this.purereservecurrencies.find(item => item.currencyid == "iS8TfRPfVpKo5FVfSUzfHBQxo9KuzpnqLU").reserves
    //       this.pureTbtcVrsc = parseFloat(vrscReserves / tbtcReserves)
    //     })
    //     .catch((error) => {
    //       this.currencies = error
    //     })
    // },

    getBinancePrices() {
      axios.get("https://api.binance.com/api/v3/ticker/price?symbol=BTCUSDT").then(res =>
        this.binance_btcusd = res.data.price
      )
      axios.get("https://api.binance.com/api/v3/ticker/price?symbol=ETHUSDT").then(res =>
        this.binance_ethusd = res.data.price
      )
      axios.get("https://api.binance.com/api/v3/ticker/price?symbol=MKRUSDT").then(res =>
        this.binance_mkrusd = res.data.price
      )
      axios.get("https://api.binance.com/api/v3/ticker/price?symbol=ETHBTC").then(res =>
        this.binance_ethbtc = res.data.price
      )
      axios.get("https://api.binance.com/api/v3/ticker/price?symbol=MKRBTC").then(res =>
        this.binance_mkrbtc = res.data.price
      )
    },
    sendAxiosRequest(method, url, headers, data) {
      return axios({
        method: method,
        url: url,
        headers: headers,
        data: data
      });
    },
    loadBasketSettings() {
      try {
        const saved = localStorage.getItem('verusBasketSettings');
        if (saved) {
          this.basketSettings = JSON.parse(saved);
          console.log('Loaded basket settings from localStorage');
        }
      } catch (error) {
        console.error('Error loading basket settings:', error);
      }
    },
    saveBasketSettings() {
      try {
        localStorage.setItem('verusBasketSettings', JSON.stringify(this.basketSettings));
        console.log('Saved basket settings to localStorage');
      } catch (error) {
        console.error('Error saving basket settings:', error);
      }
    },
    handleSettingsChanged(newSettings) {
      this.basketSettings = newSettings;
      this.saveBasketSettings();
    }

  },
  async created() {
    // Load saved basket settings from localStorage
    this.loadBasketSettings();
    
    try {
      const [response1, response2] = await Promise.all([
        fetch('/currencies.json'),
        fetch('/baskets.json')
      ]);

      const data1 = await response1.json()
      const data2 = await response2.json()

      this.currencyDictionary = data1
      this.baskets = data2
    } catch (error) {
      console.log(error)
    }

    try {
      // Fetch details for each basket and update the array
      const fetchPromises = this.baskets.map(async (basket, index) => {
        let useRpc = true;
        if(basket.rpc==''){
          useRpc = false
        }
        const details = await this.getCurrencyDetails(basket.rpc, basket.currencyid, useRpc);
        if (details) {
          console.log(`Updating basket ${basket.ticker} with:`, details);
          // Update the basket in place
          this.baskets[index] = {
            ...basket,
            reservecurrencies: details.reservecurrencies,
            supply: details.supply,
            bestheight: details.bestheight,
          }
        }
        else {
          console.warn(`No details fetch for ${basket.currencyid}`);
        }
      })

      // Wait for all fetches to complete
      await Promise.all(fetchPromises);
    } catch (error) {
      console.error('Error loading JSON files or fetching details:', error);
    }
  },
  async mounted() {
    this.getLatestBlock()
    // this.getPureCurrency()
    this.getBinancePrices()
    this.chains.forEach(chain => this.getStakingSupply(chain))
  }
};
</script>
<style></style>
