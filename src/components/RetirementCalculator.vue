<template>
    <div class="calculator-container">
      <div class="header">
        <h1>Retirement Forecast & Reports</h1>
      </div>
  
      <div class="form-stacked">
        <label for="currentAge">
          Current Age:
          <input
            id="currentAge"
            type="number"
            v-model.number="currentAge"
            @input="validateInput('currentAge')"
            aria-describedby="currentAgeHelp"
            title="Your current age."
          />
          <small id="currentAgeHelp">e.g., enter numbers only. No symbols.</small>
          <span v-if="errors.currentAge" class="error">{{ errors.currentAge }}</span>
        </label>
  
        <label for="retirementAge">
          Retirement Age:
          <input
            id="retirementAge"
            type="number"
            v-model.number="retirementAge"
            @input="validateInput('retirementAge')"
            aria-describedby="retirementAgeHelp"
            title="Your target retirement age."
          />
          <small id="retirementAgeHelp">e.g., enter numbers only. No symbols.</small>
          <span v-if="errors.retirementAge" class="error">{{ errors.retirementAge }}</span>
        </label>
  
        <label for="currentSavings">
          Current Savings ($):
          <input
            id="currentSavings"
            type="number"
            v-model.number="currentSavings"
            @input="validateInput('currentSavings')"
            aria-describedby="currentSavingsHelp"
            title="Your current savings towards retirement."
          />
          <small id="currentSavingsHelp">e.g., enter numbers only. No symbols.</small>
          <span v-if="errors.currentSavings" class="error">{{ errors.currentSavings }}</span>
        </label>
  
        <label for="monthlyContribution">
          Monthly Contribution ($):
          <input
            id="monthlyContribution"
            type="number"
            v-model.number="monthlyContribution"
            @input="validateInput('monthlyContribution')"
            aria-describedby="monthlyContributionHelp"
            title="Amount you plan to contribute monthly."
          />
          <small id="monthlyContributionHelp">e.g., enter numbers only. No symbols.</small>
          <span v-if="errors.monthlyContribution" class="error">{{ errors.monthlyContribution }}</span>
        </label>
  
        <label for="growthRate">
          Annual Growth Rate (%):
          <input
            id="growthRate"
            type="number"
            v-model.number="growthRate"
            @input="validateInput('growthRate')"
            aria-describedby="growthRateHelp"
            title="Expected annual return rate."
          />
          <small id="growthRateHelp">e.g., enter numbers only. No symbols.</small>
          <span v-if="errors.growthRate" class="error">{{ errors.growthRate }}</span>
        </label>
  
        <label for="showInflationAdjusted" class="toggle-label">
          <input
            id="showInflationAdjusted"
            type="checkbox"
            v-model="showInflationAdjusted"
            aria-label="Toggle inflation-adjusted results"
          />
          Apply Inflation Adjustment
        </label>
  
        <!-- Show inflation input only if toggle is enabled -->
        <label v-if="showInflationAdjusted" for="inflationRate">
          Inflation Rate (%):
          <input
            id="inflationRate"
            type="number"
            v-model.number="inflationRate"
            @input="validateInput('inflationRate')"
            aria-describedby="inflationRateHelp"
            title="Expected annual inflation rate."
          />
          <small id="inflationRateHelp">e.g., enter numbers only. No symbols.</small>
          <span v-if="errors.inflationRate" class="error">{{ errors.inflationRate }}</span>
        </label>
  
        <div class="button-row">
          <button
            class="calculate-button"
            @click="generateProjection"
            :disabled="hasErrors"
          >
            Run Simulation
          </button>
          <button class="reset-button" @click="resetCalculator">
            Reset
          </button>
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
  
      <div class="calculation-method">
        <p><strong>Calculation Notes:</strong></p>
        <p>Monthly contributions are compounded monthly.</p>
        <p v-if="showInflationAdjusted">Results shown in today's dollars based on user-defined inflation rate.</p>
        <p v-else>Results shown in future nominal dollars.</p>
      </div>
    </div>
  </template>
  
  
  <script setup>
  import { ref, watch, onMounted, nextTick, computed } from 'vue'
  import { Chart, registerables } from 'chart.js'
  
  Chart.register(...registerables)
  
  // State
  const currentAge = ref(30)
  const retirementAge = ref(65)
  const currentSavings = ref(50000)
  const monthlyContribution = ref(1000)
  const growthRate = ref(7)
  const inflationRate = ref(2)
  const showInflationAdjusted = ref(false)
  
  const totalContributions = ref(0)
  const finalForecastValue = ref(0)
  
  const errors = ref({
    currentAge: '',
    retirementAge: '',
    currentSavings: '',
    monthlyContribution: '',
    growthRate: '',
    inflationRate: '',
  })
  
  const validateInput = (field) => {
    const value = {
      currentAge: currentAge.value,
      retirementAge: retirementAge.value,
      currentSavings: currentSavings.value,
      monthlyContribution: monthlyContribution.value,
      growthRate: growthRate.value,
      inflationRate: inflationRate.value,
    }[field]
  
    if (value === '' || value === null || isNaN(value)) {
      errors.value[field] = 'Value required and must be a number.'
    } else if (value < 0) {
      errors.value[field] = 'Negative values are not allowed.'
    } else if (value > 10000000) {
      errors.value[field] = 'Value is too high.'
    } else {
      errors.value[field] = ''
    }
  }
  
  const hasErrors = computed(() => {
    return Object.values(errors.value).some(error => error !== '')
  })
  
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
  
  let retirementChart = null
  
  const generateProjection = async () => {
    if (hasErrors.value) return
  
    const canvas = document.getElementById('retirementChart')
    if (!canvas) return
  
    if (retirementChart) {
      retirementChart.destroy()
    }
  
    const ctx = canvas.getContext('2d')
    const scenarios = 100
    const years = retirementAge.value - currentAge.value
    const inflationFactor = 1 + (inflationRate.value / 100)
    const fixedVolatility = 0.15
  
    const allScenarios = []
  
    for (let i = 0; i < scenarios; i++) {
      let yearlyBalances = []
      let currentValue = currentSavings.value
      let cumulativeInflation = 1
  
      for (let year = 1; year <= years; year++) {
        const randomReturn = randomNormal(growthRate.value / 100, fixedVolatility)
        currentValue = (currentValue + monthlyContribution.value * 12) * (1 + randomReturn)
        cumulativeInflation *= inflationFactor
  
        yearlyBalances.push({
          year,
          balance: showInflationAdjusted.value ? currentValue / cumulativeInflation : currentValue,
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
  
    retirementChart = new Chart(ctx, {
      type: 'line',
      data: {
        labels: Array.from({ length: years }, (_, i) => `Year ${i + 1}`),
        datasets: [
          {
            label: 'Best Market Return',
            data: upperBoundScenario,
            borderColor: 'rgba(255, 205, 86, 1)',
            backgroundColor: getGradient(ctx, 'rgba(255, 205, 86, 1)'),
            fill: true,
            tension: 0.4,
            pointRadius: 0,
          },
          {
            label: 'Projected Return',
            data: medianScenario,
            borderColor: 'rgba(75, 192, 192, 1)',
            backgroundColor: getGradient(ctx, 'rgba(75, 192, 192, 1)'),
            fill: true,
            tension: 0.4,
            pointRadius: 0,
          },
          {
            label: 'Worst Market Return',
            data: lowerBoundScenario,
            borderColor: 'rgba(255, 99, 132, 1)',
            backgroundColor: getGradient(ctx, 'rgba(255, 99, 132, 1)'),
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
    showInflationAdjusted.value = false
    totalContributions.value = 0
    finalForecastValue.value = 0
  
    Object.keys(errors.value).forEach(field => errors.value[field] = '')
  
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
    [
      currentAge,
      retirementAge,
      currentSavings,
      monthlyContribution,
      growthRate,
      inflationRate,
      showInflationAdjusted
    ],
    () => generateProjection()
  )
  </script>
  

<style scoped>
/* Shared Container */
.calculator-container {
  width: 100%;
  max-width: 900px;
  margin: 1rem auto;
  font-family: Arial, sans-serif;
  padding: 1.2rem;
  text-align: center;
  box-sizing: border-box;
}

/* Header */
.header {
  margin-bottom: 2rem;
}

.header h1 {
  margin: 0;
  font-size: 1.8rem;
}

/* Form Styling */
.form-stacked {
  display: flex;
  flex-direction: column;
  align-items: center;
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

/* Buttons Row */
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

/* Chart Summary Container */
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

/* Chart Container */
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

/* Calculation Notes */
.calculation-method {
  margin-top: -0.5rem;
  text-align: center;
  font-size: 0.85rem;
  line-height: 1.3;
}

.calculation-method p {
  margin: 0.2rem 0;
}

/* Responsive */
@media (max-width: 768px) {
  .form-stacked {
    width: 90%;
  }
}
</style>

  
  