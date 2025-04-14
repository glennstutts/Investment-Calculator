<template>
    <div class="app">
      <div class="header">
        <h1>Financial Tools Dashboard</h1>
  
        <label
  class="switch"
  :class="{ dark: darkMode }"
  :title="darkMode ? 'Switch to Light' : 'Switch to Dark'"
  style="display: none;"
>
  <input type="checkbox" v-model="darkMode" />
  <span class="slider round">
    <span class="slider-icon">
      <span v-if="darkMode">🌙</span>
      <span v-else>☀️</span>
    </span>
    <span class="mode-label">{{ darkMode ? 'DARK' : 'LIGHT' }}</span>
  </span>
</label>

      </div>
  
      <div class="tab-buttons">
        <button :class="{ active: activeTab === 'investment' }" @click="switchTab('investment')">
          Investment Growth Calculator
        </button>
        <button :class="{ active: activeTab === 'retirement' }" @click="switchTab('retirement')">
          Retirement Forecast & Reports
        </button>
      </div>
  
      <!-- ✅ Important: assign the ref -->
      <component :is="activeComponent" ref="activeComponentRef" />
    </div>
  </template>
  
  <script setup>
  import { ref, watch, computed, nextTick } from 'vue'
  import InvestmentCalculator from './InvestmentCalculator.vue'
  import RetirementCalculator from './RetirementCalculator.vue'
  
  const darkMode = ref(true)
  watch(darkMode, (newVal) => {
    document.body.classList.toggle('dark', newVal)
  })
  
  const activeTab = ref('investment')
  const activeComponentRef = ref(null)
  
  const activeComponent = computed(() => {
    return activeTab.value === 'investment' ? InvestmentCalculator : RetirementCalculator
  })
  
  async function switchTab(tab) {
    activeTab.value = tab
    await nextTick()
  
    if (tab === 'retirement' && activeComponentRef.value?.generateProjection) {
      activeComponentRef.value.generateProjection()
    }
  }
  </script>

  
  <style scoped>
  .app {
    text-align: center;
    padding: 2rem;
  }
  
  .header {
    position: relative;
    margin-bottom: 1.5rem;
  }
  
  .header h1 {
    margin-bottom: 1rem;
    font-size: 2rem;
  }
  
  .switch {
    position: absolute;
    top: 0;
    right: 0;
    display: inline-block;
    width: 60px;
    height: 24px;
    z-index: 10;
  }
  
  .switch input {
    opacity: 0;
    position: absolute;
    width: 0;
    height: 0;
  }
  
  .slider {
    position: relative;
    background-color: #f0f0f0;
    border-radius: 50px;
    cursor: pointer;
    height: 100%;
    display: flex;
    align-items: center;
    transition: background-color 0.4s ease;
    box-shadow: inset 0 0 8px rgba(0, 0, 0, 0.1);
    padding: 2px;
  }
  
  .slider-icon {
    position: absolute;
    left: 2px;
    width: 20px;
    height: 20px;
    background: white;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 12px;
    transition: transform 0.4s ease, background-color 0.4s ease;
    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.2);
  }
  
  .switch.dark .slider-icon {
    transform: translateX(36px);
    background-color: #444;
    color: #fff;
  }
  
  .mode-label {
    flex: 1;
    text-align: right;
    font-size: 8px;
    font-weight: bold;
    color: #666;
    user-select: none;
    transition: color 0.4s ease, text-align 0.4s ease;
    padding: 0 6px;
  }
  
  .switch.dark .mode-label {
    text-align: left;
    color: #000;
  }
  
  .tab-buttons {
    margin-bottom: 2rem;
    display: flex;
    justify-content: center;
    gap: 1rem;
  }
  
  .tab-buttons button {
    padding: 0.7rem 1.2rem;
    background-color: #4caf50;
    color: white;
    border: none;
    cursor: pointer;
    border-radius: 4px;
    font-size: 1rem;
    transition: background-color 0.3s, transform 0.2s;
  }
  
  .tab-buttons button.active {
    background-color: #388e3c;
    border: 2px solid white;
  }
  
  .tab-buttons button:hover {
    background-color: #45a049;
    transform: translateY(-2px);
  }
  
  body.dark {
    background: #121212;
    color: #f0f0f0;
  }
  
  body.dark .tab-buttons button {
    background-color: #4caf50;
  }
  
  body.dark .tab-buttons button.active {
    background-color: #388e3c;
  }
  </style>
  