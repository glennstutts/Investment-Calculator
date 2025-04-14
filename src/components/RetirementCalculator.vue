<template>
    <div class="retirement-calculator">
      <div class="header">
        <h1>Retirement Forecast & Reports</h1>
      </div>
  
      <div class="form-stacked">
        <label>
          Current Age:
          <input type="number" v-model.number="currentAge" />
        </label>
        <label>
          Retirement Age:
          <input type="number" v-model.number="retirementAge" />
        </label>
        <label>
          Current Savings ($):
          <input type="number" v-model.number="currentSavings" />
        </label>
        <label>
          Monthly Contribution ($):
          <input type="number" v-model.number="monthlyContribution" />
        </label>
        <label>
          Annual Growth Rate (%):
          <input type="number" v-model.number="growthRate" />
        </label>
        <label>
          Inflation Rate (%):
          <input type="number" v-model.number="inflationRate" />
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
  import { ref, watch, onMounted, nextTick } from 'vue'
  import { Chart, registerables } from 'chart.js'
  
  Chart.register(...registerables)
  
  // State
  const currentAge = ref(30)
  const retirementAge = ref(65)
  const currentSavings = ref(50000)
  const monthlyContribution = ref(1000)
  const growthRate = ref(7)
  const inflationRate = ref(2)
  
  const totalContributions = ref(0)
  const finalForecastValue = ref(0)
  
  let retirementChart = null
  
  const formatCurrency = (value) => {
    if (isNaN(value)) return '—'
    return new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency: 'USD',
      maximumFractionDigits: 0,
    }).format(value)
  }
  
  const randomNormal = (mean, stdDev) => {
    let u = 0, v = 0
    while (u === 0) u = Math.random()
    while (v === 0) v = Math.random()
    return mean + stdDev * Math.sqrt(-2.0 * Math.log(u)) * Math.cos(2.0 * Math.PI * v)
  }
  
  const generateProjection = async () => {
    const canvas = document.getElementById('retirementChart')
    if (!canvas) return
  
    if (retirementChart) {
      retirementChart.destroy()
    }
  
    const ctx = canvas.getContext('2d')
  
    const scenarios = 100
    const years = retirementAge.value - currentAge.value
    const inflationFactor = 1 + inflationRate.value / 100
    const fixedVolatility = 0.15
  
    const allScenarios = []
  
    for (let i = 0; i < scenarios; i++) {
      let yearlyBalances = []
      let currentValue = currentSavings.value
      let totalContributed = currentSavings.value
      let cumulativeInflation = 1
  
      for (let year = 1; year <= years; year++) {
        const randomReturn = randomNormal(growthRate.value / 100, fixedVolatility)
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
    const upperBoundScenario = []
    const lowerBoundScenario = []
  
    for (let year = 0; year < years; year++) {
      const balances = allScenarios.map(s => s[year].balance).sort((a, b) => a - b)
      medianScenario.push(Math.round(balances[Math.floor(balances.length / 2)]))
      upperBoundScenario.push(Math.round(balances[Math.floor(balances.length * 0.75)]))
      lowerBoundScenario.push(Math.round(balances[Math.floor(balances.length * 0.25)]))
    }
  
    totalContributions.value = currentSavings.value + monthlyContribution.value * 12 * years
    finalForecastValue.value = medianScenario[medianScenario.length - 1]
  
    const getGradient = (ctx, color) => {
      const gradient = ctx.createLinearGradient(0, 0, 0, 400)
      gradient.addColorStop(0, color)
      gradient.addColorStop(1, 'rgba(255, 255, 255, 0)')
      return gradient
    }
  
    const upperLineColor = 'rgba(255, 205, 86, 1)' // Yellow
    const medianLineColor = 'rgba(75, 192, 192, 1)' // Teal
    const lowerLineColor = 'rgba(255, 99, 132, 1)' // Red
  
    retirementChart = new Chart(ctx, {
      type: 'line',
      data: {
        labels: Array.from({ length: years }, (_, i) => `Year ${i + 1}`),
        datasets: [
          {
            label: 'Best Market Return',
            data: upperBoundScenario,
            borderColor: upperLineColor,
            backgroundColor: getGradient(ctx, upperLineColor),
            fill: true,
            tension: 0.4,
            pointRadius: 0,
          },
          {
            label: 'Projected Return',
            data: medianScenario,
            borderColor: medianLineColor,
            backgroundColor: getGradient(ctx, medianLineColor),
            fill: true,
            tension: 0.4,
            pointRadius: 0,
          },
          {
            label: 'Worst Market Return',
            data: lowerBoundScenario,
            borderColor: lowerLineColor,
            backgroundColor: getGradient(ctx, lowerLineColor),
            fill: true,
            tension: 0.4,
            pointRadius: 0,
          },
        ],
      },
      options: {
        responsive: true,
        maintainAspectRatio: false,
        interaction: {
          mode: 'nearest',
          axis: 'x',
          intersect: false,
        },
        plugins: {
          legend: { display: true },
          tooltip: {
            mode: 'nearest',
            intersect: false,
            position: 'nearest',
            callbacks: {
              label: (context) => `${context.dataset.label}: ${formatCurrency(context.parsed.y)}`,
              labelTextColor: (context) => context.dataset.borderColor,
              labelColor: (context) => ({
                borderColor: context.dataset.borderColor,
                backgroundColor: context.dataset.borderColor,
              }),
            },
          },
        },
        scales: {
          y: {
            ticks: {
              callback: (value) => formatCurrency(value),
            },
          },
        },
      },
    })
  }
  
  const resetCalculator = () => {
    currentAge.value = 30
    retirementAge.value = 65
    currentSavings.value = 50000
    monthlyContribution.value = 1000
    growthRate.value = 7
    inflationRate.value = 2
    totalContributions.value = 0
    finalForecastValue.value = 0
  
    if (retirementChart) {
      retirementChart.destroy()
      retirementChart = null
    }
  }
  
  onMounted(async () => {
    await nextTick()
    generateProjection()
  })
  
  watch(
    [currentAge, retirementAge, currentSavings, monthlyContribution, growthRate, inflationRate],
    () => generateProjection()
  )
  </script>
  
  <style scoped>
  .retirement-calculator {
    width: 100%;
    max-width: 900px;
    margin: 1rem auto;
    font-family: Arial, sans-serif;
    padding: 1.2rem;
    box-sizing: border-box;
    text-align: center;
  }
  
  .header {
    margin-bottom: 2rem;
  }
  
  .header h1 {
    margin: 0;
    font-size: 1.8rem;
  }
  
  .form-stacked {
    display: flex;
    flex-direction: column;
    align-items: center;
    margin-bottom: 1.5rem;
    gap: 0.5rem;
    width: 60%;
    margin: 0 auto 1rem;
  }
  
  .form-stacked label {
    display: flex;
    flex-direction: column;
    width: 100%;
    font-size: 0.85rem;
  }
  
  .form-stacked input,
  .form-stacked select {
    padding: 0.3rem;
    font-size: 0.85rem;
    border: 1px solid #ccc;
    border-radius: 4px;
    margin-top: 0.3rem;
    width: 100%;
    box-sizing: border-box;
  }
  
  .button-row {
    display: flex;
    gap: 0.75rem;
    width: 100%;
  }
  
  .calculate-button,
  .reset-button {
    flex: 1;
    padding: 0.4rem;
    border: none;
    cursor: pointer;
    border-radius: 4px;
    font-size: 0.85rem;
  }
  
  .calculate-button {
    background-color: #4caf50;
    color: white;
  }
  
  .reset-button {
    background-color: #f44336;
    color: white;
  }
  
  .calculate-button:hover {
    background-color: #45a049;
  }
  
  .reset-button:hover {
    background-color: #d32f2f;
  }
  
  .chart-summary-container {
    display: flex;
    flex-wrap: wrap;
    gap: 1rem;
    margin-bottom: 2rem;
    align-items: center;
    justify-content: center;
    width: 100%;
    margin-top: 1rem;
  }
  
  .summary {
    flex: 1 1 250px;
  }
  
  .summary h2 {
    font-size: 1rem;
    margin-bottom: 0.5rem;
  }
  
  .summary p {
    font-size: 0.85rem;
    margin: 0.3rem 0;
  }
  
  .chart-container {
    width: 100%;
    max-width: 100%;
    height: auto;
    min-height: 250px;
  }
  
  canvas {
    display: block;
    width: 100% !important;
    height: auto !important;
  }
  
  @media (max-width: 768px) {
    .form-stacked {
      width: 90%;
    }
  }
  </style>
  