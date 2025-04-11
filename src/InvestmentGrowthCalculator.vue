<template>
  <div class="app">
    <h1>Financial Tools Dashboard</h1>

    <!-- Tab Navigation -->
    <div class="tab-buttons">
      <button :class="{ active: activeTab === 'growth' }" @click="activeTab = 'growth'">
        Investment Growth Calculator
      </button>
      <button :class="{ active: activeTab === 'retirement' }" @click="activeTab = 'retirement'">
        Retirement Forecast & Reports
      </button>
    </div>

    <!-- Tab Content -->
    <div v-if="activeTab === 'growth'">
      <InvestmentCalculator />
    </div>

    <div v-if="activeTab === 'retirement'">
      <RetirementCalculator />
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

// Import components
import InvestmentCalculator from './components/InvestmentCalculator.vue'
import RetirementCalculator from './components/RetirementCalculator.vue'

// Tab state
const activeTab = ref('growth')
</script>

<style scoped>
.app {
  text-align: center;
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
}

.tab-buttons {
  display: flex;
  justify-content: center;
  margin-bottom: 2rem;
}

.tab-buttons button {
  margin: 0 0.5rem;
  padding: 0.75rem 1.5rem;
  background-color: #4caf50;
  color: white;
  border: none;
  cursor: pointer;
  border-radius: 6px;
  transition: background-color 0.3s;
}

.tab-buttons button:hover {
  background-color: #45a049;
}

.tab-buttons button.active {
  background-color: #388e3c;
}
</style>
