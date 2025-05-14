<template>
  <div class="calculator-container">
    <!-- Toggle Forecast Display Modes -->
    <div class="toggle-label">
      <label>
        <input type="checkbox" v-model="showScenarioBands" /> Show Best/Worst Market Scenarios
      </label>
    </div>

    <!-- Header -->
    <div class="header">
      <h1>Retirement Forecast & Reports</h1>
    </div>

    <!-- Input Form -->
    <div class="form-stacked">
      <div class="input-grid">
        <div class="input-box">
          <label>Current Age:</label>
          <input type="number" v-model.number="currentAge" @input="validateInput('currentAge')" />
          <span v-if="errors.currentAge" class="error">{{ errors.currentAge }}</span>
        </div>
        <div class="input-box">
          <label>Retirement Age:</label>
          <input type="number" v-model.number="retirementAge" @input="validateInput('retirementAge')" />
          <span v-if="errors.retirementAge" class="error">{{ errors.retirementAge }}</span>
        </div>

        <div class="input-box">
          <label>Projected Death Age:</label>
          <input type="number" v-model.number="projectedDeathAge" @input="validateInput('projectedDeathAge')" />
          <span v-if="errors.projectedDeathAge" class="error">{{ errors.projectedDeathAge }}</span>
        </div>
        <div class="input-box">
          <label>Current Savings ($):</label>
          <input type="number" v-model.number="currentSavings" @input="validateInput('currentSavings')" />
          <span v-if="errors.currentSavings" class="error">{{ errors.currentSavings }}</span>
        </div>

        <div class="input-box">
          <label>Monthly Contribution ($):</label>
          <input type="number" v-model.number="monthlyContribution" @input="validateInput('monthlyContribution')" />
          <span v-if="errors.monthlyContribution" class="error">{{ errors.monthlyContribution }}</span>
        </div>
        <div class="input-box">
          <label>Annual Growth Rate (%):</label>
          <input type="number" v-model.number="growthRate" @input="validateInput('growthRate')" />
          <span v-if="errors.growthRate" class="error">{{ errors.growthRate }}</span>
        </div>

        <div class="input-box">
          <label>Withdrawal Type:</label>
          <select v-model="drawdownType">
            <option value="amount">Fixed Amount</option>
            <option value="percentage">Percentage of Balance</option>
          </select>
        </div>
        <div class="input-box">
          <label>Annual Withdrawal:</label>
          <input type="number" v-model.number="drawdownAmount" @input="validateInput('drawdownAmount')" />
          <span v-if="errors.drawdownAmount" class="error">{{ errors.drawdownAmount }}</span>
        </div>

        <div class="input-box checkbox-box">
          <input type="checkbox" id="inflationToggle" v-model="showInflationAdjusted" />
          <label for="inflationToggle">Apply Inflation Adjustment</label>
        </div>

        <div class="input-box" v-if="showInflationAdjusted">
          <label>Inflation Rate (%):</label>
          <input type="number" v-model.number="inflationRate" @input="validateInput('inflationRate')" />
          <span v-if="errors.inflationRate" class="error">{{ errors.inflationRate }}</span>
        </div>

        <div class="input-box checkbox-box">
          <input type="checkbox" id="socialSecurityToggle" v-model="includeSocialSecurity" />
          <label for="socialSecurityToggle">Include Social Security Income</label>
        </div>

        <div class="input-box full-width">
          <label>Estimated Federal Effective Tax Rate (%): <strong>{{ calculatedEffectiveTaxRate }}%</strong></label>
          <p class="tax-note small">This is automatically calculated based on projected drawdown income and federal brackets.</p>
        </div>
      </div>

      <div class="button-row">
        <button class="calculate-button" @click="generateProjection" :disabled="hasErrors">Run Simulation</button>
        <button class="reset-button" @click="resetCalculator">Reset</button>
      </div>
    </div>

    <!-- Summary and Chart -->
    <div class="chart-summary-container">
      <div class="summary">
        <h2>Median Scenario Summary</h2>
        <p><strong>Total Contributions:</strong> {{ formatCurrency(totalContributions) }}</p>
        <p><strong>Balance at Retirement:</strong> {{ formatCurrency(balanceAtRetirement) }}</p>
        <p><strong>End Balance at Death:</strong> {{ formatCurrency(finalForecastValue) }}</p>
      </div>

      <div class="chart-container">
        <canvas id="retirementChart"></canvas>
      </div>
    </div>

    <!-- Calculation Notes -->
    <div class="calculation-method">
      <p><strong>Calculation Notes:</strong></p>
      <p>Monthly contributions are compounded monthly.</p>
      <p v-if="showInflationAdjusted">Results shown in today's dollars based on user-defined inflation rate.</p>
      <p v-else>Results shown in future nominal dollars.</p>
    </div>
  </div>
</template>











<script setup>
import { ref, computed, watch, onMounted, nextTick } from 'vue'
import { Chart, registerables } from 'chart.js'

Chart.register(...registerables)

const subTab = ref('forecast')

const currentAge = ref(30)
const retirementAge = ref(65)
const projectedDeathAge = ref(90)
const currentSavings = ref(50000)
const monthlyContribution = ref(1000)
const growthRate = ref(7)
const inflationRate = ref(2)
const showInflationAdjusted = ref(false)

const drawdownAmount = ref(120000)
const drawdownType = ref('amount')

const includeSocialSecurity = ref(false)
const socialSecurityIncome = ref(20000)

const showScenarioBands = ref(true)

const errors = ref({
  currentAge: '',
  retirementAge: '',
  projectedDeathAge: '',
  currentSavings: '',
  monthlyContribution: '',
  growthRate: '',
  inflationRate: '',
  drawdownAmount: '',
  socialSecurityIncome: ''
})

const totalContributions = ref(0)
const finalForecastValue = ref(0)
const balanceAtRetirement = ref(0)
const depletionWarning = ref(false)

const hasErrors = computed(() => Object.values(errors.value).some(e => e !== ''))

const formatCurrency = (value) => {
  if (isNaN(value)) return '—'
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
    maximumFractionDigits: 0,
  }).format(value)
}

const validateInput = (field) => {
  const value = {
    currentAge: currentAge.value,
    retirementAge: retirementAge.value,
    projectedDeathAge: projectedDeathAge.value,
    currentSavings: currentSavings.value,
    monthlyContribution: monthlyContribution.value,
    growthRate: growthRate.value,
    inflationRate: inflationRate.value,
    drawdownAmount: drawdownAmount.value,
    socialSecurityIncome: socialSecurityIncome.value
  }[field]

  if (value === '' || value === null || isNaN(value)) {
    errors.value[field] = 'Value required and must be a number.'
  } else if (value < 0) {
    errors.value[field] = 'Negative values are not allowed.'
  } else if (field === 'drawdownAmount' && value > 1000000) {
    errors.value[field] = 'Drawdown is too high.'
  } else if ((field === 'currentAge' || field === 'retirementAge' || field === 'projectedDeathAge') && value > 120) {
    errors.value[field] = 'Value is too high.'
  } else {
    errors.value[field] = ''
  }
}

const calculateEffectiveTaxRate = (grossIncome) => {
  const brackets = [
    { rate: 0.10, threshold: 11000 },
    { rate: 0.12, threshold: 44725 },
    { rate: 0.22, threshold: 95375 },
    { rate: 0.24, threshold: 182100 },
    { rate: 0.32, threshold: 231250 },
    { rate: 0.35, threshold: 578125 },
    { rate: 0.37, threshold: Infinity }
  ]

  let tax = 0
  let lastThreshold = 0

  for (const bracket of brackets) {
    if (grossIncome > bracket.threshold) {
      tax += (bracket.threshold - lastThreshold) * bracket.rate
      lastThreshold = bracket.threshold
    } else {
      tax += (grossIncome - lastThreshold) * bracket.rate
      break
    }
  }

  return Math.round((tax / grossIncome) * 100)
}

const calculatedEffectiveTaxRate = computed(() => {
  const gross = drawdownType.value === 'percentage'
    ? 0.04 * finalForecastValue.value
    : drawdownAmount.value
  return calculateEffectiveTaxRate(gross || 1)
})

let retirementChart = null

const renderChart = (labels, datasets) => {
  const canvas = document.getElementById('retirementChart')
  if (!canvas || !canvas.getContext) return

  if (retirementChart) retirementChart.destroy()

  const ctx = canvas.getContext('2d')
  if (!ctx) return

  retirementChart = new Chart(ctx, {
    type: 'line',
    data: { labels, datasets },
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
            label: (context) => `${context.dataset.label}: ${formatCurrency(context.parsed.y)}`,
            labelTextColor: (context) => context.dataset.borderColor,
            labelColor: (context) => ({
              borderColor: context.dataset.borderColor,
              backgroundColor: context.dataset.borderColor
            })
          }
        }
      },
      scales: {
        y: {
          ticks: {
            callback: (value) => formatCurrency(value)
          }
        }
      }
    }
  })
}

const generateProjection = () => {
  nextTick(() => {
    const totalYears = projectedDeathAge.value - currentAge.value
    const accumulationYears = retirementAge.value - currentAge.value
    const inflationFactor = 1 + (inflationRate.value / 100)
    const taxRate = calculatedEffectiveTaxRate.value / 100
    const getDrawdown = drawdownType.value === 'percentage'
      ? (val) => val * (drawdownAmount.value / 100)
      : () => drawdownAmount.value

    let medianScenario = []
    let bestScenario = []
    let worstScenario = []
    let balance = currentSavings.value
    let cumulativeInflation = 1

    for (let year = 1; year <= totalYears; year++) {
      const age = currentAge.value + year - 1

      if (age < retirementAge.value) {
        balance = (balance + monthlyContribution.value * 12) * (1 + growthRate.value / 100)
      } else {
        if (year === (retirementAge.value - currentAge.value + 1)) {
          balanceAtRetirement.value = balance
        }

        let withdrawal = getDrawdown(balance)
        if (includeSocialSecurity.value) {
          withdrawal -= socialSecurityIncome.value
          withdrawal = Math.max(0, withdrawal)
        }
        const taxedWithdrawal = withdrawal / (1 - taxRate)
        balance = (balance - taxedWithdrawal) * (1 + growthRate.value / 100)
        if (balance <= 0) {
          balance = 0
          depletionWarning.value = true
        }
      }

      cumulativeInflation *= inflationFactor
      const adjustedBalance = showInflationAdjusted.value ? balance / cumulativeInflation : balance

      medianScenario.push(Math.round(adjustedBalance))
    }

    bestScenario = medianScenario.map(v => Math.round(v * 1.15))
    worstScenario = medianScenario.map(v => Math.round(v * 0.85))

    totalContributions.value = currentSavings.value + monthlyContribution.value * 12 * accumulationYears
    finalForecastValue.value = medianScenario[medianScenario.length - 1]

    const labels = Array.from({ length: totalYears }, (_, i) => `${currentAge.value + i}`)

    const getGradient = (ctx, color) => {
      const gradient = ctx.createLinearGradient(0, 0, 0, 400)
      gradient.addColorStop(0, color)
      gradient.addColorStop(1, 'rgba(255, 255, 255, 0)')
      return gradient
    }

    const canvas = document.getElementById('retirementChart')
    if (!canvas || !canvas.getContext) return
    const ctx = canvas.getContext('2d')
    if (!ctx) return

    const datasets = []

    if (showScenarioBands.value) {
      datasets.push({
        label: 'Best Market Return',
        data: bestScenario,
        borderColor: 'rgba(255, 205, 86, 1)',
        backgroundColor: getGradient(ctx, 'rgba(255, 205, 86, 1)'),
        fill: true,
        tension: 0.4,
        pointRadius: 0
      })
    }

    datasets.push({
      label: 'Projected Return',
      data: medianScenario,
      borderColor: 'rgba(75, 192, 192, 1)',
      backgroundColor: getGradient(ctx, 'rgba(75, 192, 192, 1)'),
      fill: true,
      tension: 0.4,
      pointRadius: 0
    })

    if (showScenarioBands.value) {
      datasets.push({
        label: 'Worst Market Return',
        data: worstScenario,
        borderColor: 'rgba(255, 99, 132, 1)',
        backgroundColor: getGradient(ctx, 'rgba(255, 99, 132, 1)'),
        fill: true,
        tension: 0.4,
        pointRadius: 0
      })
    }

    renderChart(labels, datasets)
  })
}

const resetCalculator = () => {
  currentAge.value = 30
  retirementAge.value = 65
  projectedDeathAge.value = 90
  currentSavings.value = 50000
  monthlyContribution.value = 1000
  growthRate.value = 7
  inflationRate.value = 2
  drawdownAmount.value = 40000
  drawdownType.value = 'amount'
  socialSecurityIncome.value = 20000
  includeSocialSecurity.value = false
  showInflationAdjusted.value = false
  showScenarioBands.value = true
  totalContributions.value = 0
  finalForecastValue.value = 0
  balanceAtRetirement.value = 0
  depletionWarning.value = false

  Object.keys(errors.value).forEach((field) => (errors.value[field] = ''))

  if (retirementChart) {
    retirementChart.destroy()
    retirementChart = null
  }
}

onMounted(() => {
  nextTick(() => generateProjection())
})

watch(
  [
    currentAge,
    retirementAge,
    projectedDeathAge,
    currentSavings,
    monthlyContribution,
    growthRate,
    inflationRate,
    drawdownAmount,
    drawdownType,
    showInflationAdjusted,
    socialSecurityIncome,
    includeSocialSecurity,
    showScenarioBands
  ],
  () => nextTick(() => generateProjection())
)
</script>






<style scoped>
.calculator-container {
  width: 100%;
  max-width: 900px;
  margin: 1rem auto;
  font-family: Arial, sans-serif;
  padding: 1.2rem;
  text-align: center;
  box-sizing: border-box;
  color: white;
}

.header {
  margin-bottom: 1.5rem;
}

.header h1 {
  margin: 0;
  font-size: 1.8rem;
}

.toggle-label {
  margin-bottom: 1rem;
  font-size: 0.9rem;
  display: flex;
  justify-content: center;
}

.form-stacked {
  width: 100%;
  max-width: 700px;
  margin: 0 auto 1.5rem;
}

.input-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 1rem;
}

.input-box {
  display: flex;
  flex-direction: column;
}

.input-box label {
  font-size: 0.85rem;
  margin-bottom: 0.3rem;
  color: white;
}

.input-box input,
.input-box select {
  padding: 0.4rem;
  font-size: 0.85rem;
  background-color: #111;
  color: white;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.checkbox-box {
  display: flex;
  align-items: center;
  font-size: 0.85rem;
}

.checkbox-box label {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  font-size: 0.85rem;
  color: white;
}

.full-width {
  grid-column: span 2;
}

.tax-note.small {
  font-size: 0.75rem;
  margin: 0.25rem 0;
}

.error {
  color: #ff4d4f;
  font-size: 0.75rem;
  margin-top: 0.2rem;
}

.button-row {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
  margin-top: 1.5rem;
}

.calculate-button,
.reset-button {
  flex: 1;
  padding: 0.5rem;
  font-size: 0.85rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
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
  justify-content: center;
  margin-top: 2rem;
  margin-bottom: 2rem;
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
  min-height: 250px;
}

canvas {
  display: block;
  width: 100% !important;
  height: auto !important;
}

.calculation-method {
  margin-top: -0.5rem;
  font-size: 0.85rem;
  line-height: 1.3;
}

.calculation-method p {
  margin: 0.2rem 0;
}

@media (max-width: 600px) {
  .input-grid {
    grid-template-columns: 1fr;
  }
  .full-width {
    grid-column: span 1;
  }
}
</style>





  
  