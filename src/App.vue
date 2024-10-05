<template>
  <div id="app">
    <section class="section">
      <div class="container">
        <h1 class="title">Wallet Activity</h1>
        <WalletHeatmap />
      </div>
    </section>
  </div>
</template>

<script>
import WalletHeatmap from './components/WalletHeatmap.vue'

export default {
  name: 'App',
  components: {
    WalletHeatmap
  }
}
</script>

<style>
/* Global styles if needed */
</style>
