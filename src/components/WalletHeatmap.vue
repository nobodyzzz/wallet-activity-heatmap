<template>
    <div>
      <!-- Wallet Selection Dropdown -->
      <b-field label="Select Wallet">
        <b-select v-model="selectedWallet" @change="updateHeatmap" placeholder="Choose a wallet">
          <option v-for="wallet in wallets" :key="wallet" :value="wallet">{{ wallet }}</option>
        </b-select>
      </b-field>

      <!-- Calendar Heatmap -->
      <calendar-heatmap
        tooltip-unit="swaps"
        no-data-text="no swaps"
        :start-date="startDate"
        :end-date="endDate"
        :values="heatmapData"
        :color-scale="colorScale"
        :tooltip-enabled="true"
      />

      <!-- Divider -->
      <b-divider class="my-4"></b-divider>

      <!-- Average Buy-Sell Time Display -->
      <b-field label="Average Buy-Sell Time">
        <p>{{ averageBuySellTime }}</p>
      </b-field>
    </div>
  </template>

  <script>
  // Import named CalendarHeatmap from 'vue-calendar-heatmap'
  import { CalendarHeatmap } from 'vue-calendar-heatmap'
  import 'vue-calendar-heatmap/dist/vue-calendar-heatmap.css'
  import axios from 'axios'

  export default {
    name: 'WalletHeatmap',
    components: {
      // Register CalendarHeatmap with kebab-case
      'calendar-heatmap': CalendarHeatmap
    },
    data() {
      return {
        wallets: [],
        selectedWallet: '',
        allData: [],
        heatmapData: [],
        averageBuySellTime: 'N/A',
        startDate: '',
        endDate: '',
        colorScale: ['#ebedf0', '#c6e48b', '#7bc96f', '#239a3b', '#196127'],
      }
    },
    created() {
      this.fetchData()
    },
    methods: {
      async fetchData() {
        try {
          // Fetch the test JSON data from the public folder
          const response = await axios.get('/test_wallet_events_updated.json')
          const data = response.data
          this.allData = data

          // Extract unique wallet addresses
          this.wallets = [...new Set(data.map(item => item.wallet_address))]

          // Select the first wallet by default if available
          if (this.wallets.length) {
            this.selectedWallet = this.wallets[0]
          }

          // Determine the date range for the heatmap
          const dates = data.map(item => new Date(item.timestamp))
          const minDate = new Date(Math.min(...dates))
          const maxDate = new Date(Math.max(...dates))
          this.startDate = minDate.toISOString().split('T')[0]
          this.endDate = maxDate.toISOString().split('T')[0]

          // Initialize the heatmap with the first wallet
          this.updateHeatmap()
        } catch (error) {
          console.error('Error fetching data:', error)
        }
      },
      updateHeatmap() {
        if (!this.selectedWallet) {
          this.heatmapData = []
          this.averageBuySellTime = 'N/A'
          return
        }

        // Filter data for the selected wallet
        const walletData = this.allData.filter(item => item.wallet_address === this.selectedWallet)

        // Aggregate activity counts per day
        const activityMap = {}
        walletData.forEach(item => {
          const date = item.timestamp.split('T')[0]
          if (!activityMap[date]) {
            activityMap[date] = 0
          }
          // Increment count for each buy and sell
          if (item.invested_amount > 0) activityMap[date] += 1
          if (item.earned_amount > 0) activityMap[date] += 1
        })

        // Convert the aggregated data into the format required by CalendarHeatmap
        this.heatmapData = Object.keys(activityMap).map(date => ({
          date,
          count: activityMap[date]
        }))

        // Calculate the average buy-sell time
        this.calculateAverageBuySellTime(walletData)
      },
      calculateAverageBuySellTime(walletData) {
        const buys = walletData
          .filter(item => item.invested_amount > 0)
          .map(item => new Date(item.timestamp))
          .sort((a, b) => a - b)

        const sells = walletData
          .filter(item => item.earned_amount > 0)
          .map(item => new Date(item.timestamp))
          .sort((a, b) => a - b)

        if (buys.length === 0 || sells.length === 0) {
          this.averageBuySellTime = 'N/A'
          return
        }

        let totalDiff = 0
        let count = 0
        let sellIndex = 0

        buys.forEach(buyDate => {
          while (sellIndex < sells.length && sells[sellIndex] <= buyDate) {
            sellIndex++
          }
          if (sellIndex < sells.length) {
            const sellDate = sells[sellIndex]
            const diffDays = (sellDate - buyDate) / (1000 * 60 * 60 * 24)
            totalDiff += diffDays
            count++
          }
        })

        if (count === 0) {
          this.averageBuySellTime = 'N/A'
          return
        }

        const avgDays = (totalDiff / count).toFixed(2)
        this.averageBuySellTime = `${avgDays} days`
      },
      tooltipFormat(date, count) {
        console.log('date', date)
        console.log('count', count)
        // Find all data points for the specific date and wallet
        const dataPoints = this.allData.filter(item =>
          item.timestamp.split('T')[0] === date &&
          item.wallet_address === this.selectedWallet
        )
        if (dataPoints.length) {
          let buys = 0, sells = 0, spent = 0.0, earned = 0.0
          dataPoints.forEach(dp => {
            if (dp.invested_amount > 0) {
              buys += 1
              spent += dp.invested_amount
            }
            if (dp.earned_amount > 0) {
              sells += 1
              earned += dp.earned_amount
            }
          })
          return `${date}:
  Buys: ${buys},
  Sells: ${sells}
  Spent: ${spent.toFixed(2)},
  Earned: ${earned.toFixed(2)}`
        }
        return `${date}: No Activity`
      }
    }
  }
  </script>

  <style scoped>
  .my-4 {
    margin-top: 1rem;
    margin-bottom: 1rem;
  }
  </style>
