<template>
    <div class="app">
      <div class="header">
        <h1>Retirement Simulator with Monte Carlo + Inflation Adjustment</h1>
        <label class="switch" :class="{ dark: darkMode }" :title="darkMode ? 'Switch to Light' : 'Switch to Dark'">
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
  
      <div class="form-stacked">
        <label>
          Current Age:
          <input type="number" v-model.number="currentAge" required />
        </label>
        <label>
          Retirement Age:
          <input type="number" v-model.number="retirementAge" required />
        </label>
        <label>
          Current Savings ($):
          <input type="number" v-model.number="currentSavings" required />
        </label>
        <label>
          Monthly Contribution ($):
          <input type="number" v-model.number="monthlyContribution" required />
        </label>
        <label>
          Annual Growth Rate (%):
          <input type="number" v-model.number="growthRate" required />
        </label>
        <label>
          Inflation Rate (%):
          <input type="number" v-model.number="inflationRate" required />
        </label>
        <label>
          Market Volatility (% std dev):
          <input type="number" v-model.number="volatility" required />
        </label>
  
        <div class="button-row">
          <button class="calculate-button" @click="generateProjection">Run Simulation</button>
          <button class="reset-button" @click="resetCalculator">Reset</button>
        </div>
      </div>
  
      <div class="chart-summary-container">
        <div class="summary">
          <h2>Median Scenario Summary</h2>
          <p><strong>Total Contributions:</strong> {{ formatCurrency(totalContributions) }}</p>
          <p><strong>End Balance:</strong> {{ formatCurrency(finalForecastValue) }}</p>
        </div>
  
        <div class="chart-container">
          <canvas id="retirementChart"></canvas>
        </div>
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref, watch, onMounted, onActivated, nextTick } from 'vue'
  import { Chart, registerables } from 'chart.js'
  
  Chart.register(...registerables)
  
  // === State ===
  const darkMode = ref(true)
  const currentAge = ref(30)
  const retirementAge = ref(65)
  const currentSavings = ref(50000)
  const monthlyContribution = ref(1000)
  const growthRate = ref(7)
  const inflationRate = ref(2)
  const volatility = ref(15)
  
  const totalContributions = ref(0)
  const finalForecastValue = ref(0)
  
  let retirementChart = null
  
  // === Lifecycle Hooks ===
  onMounted(async () => {
    document.body.classList.add('dark')
    await nextTick()
    generateProjection()
  })
  
  onActivated(async () => {
    await nextTick()
    generateProjection()
  })
  
  // === Watchers ===
  watch(darkMode, (newVal) => {
    if (newVal) {
      document.body.classList.add('dark')
    } else {
      document.body.classList.remove('dark')
    }
    generateProjection()
  })
  
  // === Helpers ===
  const formatCurrency = (value) => {
    if (isNaN(value)) return '—'
    return new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency: 'USD',
      maximumFractionDigits: 0,
    }).format(value)
  }
  
  const getGradient = (ctx, color) => {
    const gradient = ctx.createLinearGradient(0, 0, 0, 400)
    gradient.addColorStop(0, color)
    gradient.addColorStop(1, 'rgba(255, 255, 255, 0)')
    return gradient
  }
  
  const randomNormal = (mean, stdDev) => {
    let u = 0, v = 0
    while (u === 0) u = Math.random()
    while (v === 0) v = Math.random()
    return mean + stdDev * Math.sqrt(-2.0 * Math.log(u)) * Math.cos(2.0 * Math.PI * v)
  }
  
  // === Main Logic ===
  const generateProjection = () => {
    const canvas = document.getElementById('retirementChart')
    if (!canvas) {
      console.warn('Canvas not found, skipping chart generation.')
      return
    }
  
    if (retirementChart) {
      retirementChart.destroy()
      retirementChart = null
    }
  
    const scenarios = 100
    const years = retirementAge.value - currentAge.value
    const inflationFactor = 1 + inflationRate.value / 100
    const allScenarios = []
  
    for (let i = 0; i < scenarios; i++) {
      let yearlyBalances = []
      let currentValue = currentSavings.value
      let totalContributed = currentSavings.value
      let cumulativeInflation = 1
  
      for (let year = 1; year <= years; year++) {
        const randomReturn = randomNormal(growthRate.value / 100, volatility.value / 100)
        currentValue = (currentValue + monthlyContribution.value * 12) * (1 + randomReturn)
        totalContributed += monthlyContribution.value * 12
        cumulativeInflation *= inflationFactor
        yearlyBalances.push({
          year,
          balance: currentValue / cumulativeInflation
        })
      }
  
      allScenarios.push(yearlyBalances)
    }
  
    const medianScenario = []
    for (let year = 0; year < years; year++) {
      const balances = allScenarios.map(scenario => scenario[year].balance).sort((a, b) => a - b)
      const medianBalance = balances[Math.floor(balances.length / 2)]
      medianScenario.push(Math.round(medianBalance))
    }
  
    totalContributions.value = currentSavings.value + (monthlyContribution.value * 12 * years)
    finalForecastValue.value = medianScenario[medianScenario.length - 1]
  
    const ctx = canvas.getContext('2d')
    const lineColor = darkMode.value ? 'rgba(75, 192, 192, 1)' : 'rgba(54, 162, 235, 1)'
    const fillColor = darkMode.value ? 'rgba(75, 192, 192, 0.2)' : 'rgba(54, 162, 235, 0.2)'
  
    retirementChart = new Chart(ctx, {
      type: 'line',
      data: {
        labels: Array.from({ length: years }, (_, i) => `Year ${i + 1}`),
        datasets: [{
          label: 'Median Scenario ($)',
          data: medianScenario,
          borderColor: lineColor,
          backgroundColor: getGradient(ctx, fillColor),
          fill: true,
          tension: 0.4,
          pointRadius: 0,
        }],
      },
      options: {
  responsive: true,
  maintainAspectRatio: false,
  interaction: {
    mode: 'nearest',
    axis: 'x',
    intersect: false
  },
  plugins: {
    legend: { display: true },
    tooltip: {
      mode: 'nearest',
      intersect: false,
      position: 'nearest',
      callbacks: {
        label: (context) => `${context.dataset.label}: ${formatCurrency(context.parsed.y)}`
      }
    },
  },
  scales: {
    y: {
      ticks: {
        callback: (value) => formatCurrency(value),
      },
    },
  },
}

    })
  }
  
  // === Reset Calculator ===
  const resetCalculator = () => {
    currentAge.value = 30
    retirementAge.value = 65
    currentSavings.value = 50000
    monthlyContribution.value = 1000
    growthRate.value = 7
    inflationRate.value = 2
    volatility.value = 15
    totalContributions.value = 0
    finalForecastValue.value = 0
  
    if (retirementChart) {
      retirementChart.destroy()
      retirementChart = null
    }
  }
  </script>
  
  
   <style scoped>
    /* Toggle Switch Styling */
    .switch {
      position: absolute;
      top: 0.5rem; /* move closer to top */
      right: 0.5rem; /* move closer to right edge */
      display: inline-block;
      width: 60px; /* reduce width */
      height: 24px; /* reduce height */
      z-index: 10; /* ensure it's on top */
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
      padding: 2px; /* reduced padding */
    }
    
    .slider-icon {
      position: absolute;
      left: 2px;
      width: 20px; /* smaller */
      height: 20px; /* smaller */
      background: white;
      border-radius: 50%;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 12px; /* smaller icon */
      transition: transform 0.4s ease, background-color 0.4s ease;
      box-shadow: 0 2px 6px rgba(0, 0, 0, 0.2);
    }
    
    .switch.dark .slider-icon {
      transform: translateX(36px); /* adjust for smaller width */
      background-color: #444;
      color: #fff;
    }
    
    .mode-label {
      flex: 1;
      text-align: right; /* default to right for LIGHT mode */
      font-size: 8px;
      font-weight: bold;
      color: #666;
      user-select: none;
      transition: color 0.4s ease, text-align 0.4s ease;
      padding: 0 6px;
    }
    
    .switch.dark .mode-label {
      text-align: left; /* switch to left for DARK mode */
      color: #000;
    }
    
    .modal-overlay {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      background-color: rgba(0, 0, 0, 0.6); /* dimmed background */
      display: flex;
      align-items: center;
      justify-content: center;
      z-index: 1000;
    }
    
    
    .report-modal.dark {
      background-color: #1e1e1e;
      color: #f0f0f0;
      box-shadow: 0 4px 20px rgba(255, 255, 255, 0.1);
    }
    
    /* Keep your input/button styling here */
    .report-modal select,
    .report-modal input,
    .report-modal button {
      padding: 0.5rem;
      margin-top: 1rem;
      border-radius: 4px;
      width: 100%;
      transition: background-color 0.3s, color 0.3s, border-color 0.3s;
    }
    
    .report-modal.dark select,
    .report-modal.dark input,
    .report-modal.dark button {
      background-color: #333;
      color: #f0f0f0;
      border: 1px solid #555;
    }
    
    
    
    
    
    /* Main Layout */
    @keyframes fadeIn {
      from {
        opacity: 0;
      }
      to {
        opacity: 1;
      }
    }
    
    .app {
      width: 100%;
      max-width: 900px;
      margin: 2rem auto;
      font-family: Arial, sans-serif;
      padding: 2rem;
      border-radius: 12px;
      box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
      transition: background 0.3s, color 0.3s;
      position: relative;
      text-align: center;
      opacity: 0;
      animation: fadeIn 0.8s ease-out forwards;
      min-height: 100vh; /* Full screen height */
      box-sizing: border-box;
    }
    
    .header {
      position: relative;
      margin-bottom: 2rem;
    }
    
    .header h1 {
      text-align: center;
    }
    
    /* Form Styling - Stack Inputs */
    .form-stacked {
      display: flex;
      flex-direction: column;
      align-items: center;
      margin-bottom: 2rem;
      gap: 1rem;
    }
    
    .form-stacked label {
      display: flex;
      flex-direction: column;
      width: 50%;
      font-size: 0.95rem;
      font-weight: 500;
    }
    
    .form-stacked input,
    .form-stacked select {
      padding: 0.5rem;
      font-size: 0.9rem;
      border: 1px solid #ccc;
      border-radius: 4px;
      margin-top: 0.3rem;
      width: 100%;
      box-sizing: border-box;
    }
    
    /* Button Row */
    .button-row {
      display: flex;
      gap: 1rem;
      width: 50%;
    }
    
    .calculate-button,
    .reset-button {
      flex: 1;
      padding: 0.7rem;
      background-color: #4caf50;
      color: white;
      border: none;
      cursor: pointer;
      border-radius: 4px;
      font-size: 1rem;
    }
    
    .reset-button {
      background-color: #f44336;
    }
    
    .calculate-button:hover {
      background-color: #45a049;
    }
    
    .reset-button:hover {
      background-color: #d32f2f;
    }
    
    /* Chart and Summary Container */
    .chart-summary-container {
      display: flex;
      flex-wrap: wrap;
      gap: 2rem;
      margin-bottom: 2rem;
      align-items: flex-start;
      justify-content: space-between;
      width: 100%;
    }
    
    .summary {
      flex: 1 1 250px;
    }
    
    .summary h2 {
      margin-bottom: 1rem;
    }
    
    .summary p {
      margin: 0.5rem 0;
      font-size: 1rem;
    }
    
    .chart-container {
    width: 100%;
    max-width: 100%;
    height: auto;
    min-height: 300px; /* Ensure min height for stability */
  }
  
  canvas {
    display: block;
    width: 100% !important;
    height: auto !important;
  }
  
    
    /* Action Buttons */
    .button-group {
      display: flex;
      flex-wrap: wrap;
      gap: 1rem;
      justify-content: center;
      margin-bottom: 2rem;
    }
    
    .button-group button {
      padding: 0.7rem 1.2rem;
      background-color: #4caf50;
      color: white;
      border: none;
      cursor: pointer;
      border-radius: 4px;
      font-size: 1rem;
    }
    
    .button-group button:hover {
      background-color: #45a049;
    }
    
    /* Modal Styling */
    .modal-overlay {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      background: rgba(0, 0, 0, 0.4);
      display: flex;
      align-items: center;
      justify-content: center;
      z-index: 1000;
    }
    
    .modal {
      background: white;
      padding: 2rem;
      border-radius: 8px;
      width: 320px;
      box-shadow: 0 2px 12px rgba(0, 0, 0, 0.3);
    }
    
    .modal h2 {
      margin-bottom: 1rem;
    }
    
    .modal label {
      display: block;
      margin: 1rem 0 0.5rem;
    }
    
    .modal select,
    .modal input {
      width: 100%;
      padding: 0.6rem;
      margin-bottom: 1rem;
      border: 1px solid #ccc;
      border-radius: 4px;
    }
    
    .modal-buttons {
      display: flex;
      justify-content: space-between;
    }
    
    body.dark {
    background: #121212;
    color: #f0f0f0;
  }
  
  body.dark .app {
    background: #1e1e1e;
    color: #f0f0f0;
    box-shadow: none;
  }
  
  body.dark .form-stacked label,
  body.dark .summary,
  body.dark .header h1 {
    color: #f0f0f0;
  }
  
  body.dark input,
  body.dark select,
  body.dark button {
    background-color: #2c2c2c;
    color: #e0e0e0;
    border: 1px solid #444;
  }
  
  body.dark .report-modal {
    background-color: #1e1e1e;
    color: #f0f0f0;
  }
  
  body.dark th,
  body.dark td {
    background: #1e1e1e;
    color: #e0e0e0;
  }
  
  body.dark .chart-container {
    background: transparent;
  }
    </style>
  