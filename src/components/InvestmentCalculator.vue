<template>
    <div class="calculator-container">
      <div class="header">
        <h1>Investment Growth Calculator</h1>
      </div>
  
      <div class="form-stacked">
        <!-- Inputs with validation -->
        <label for="initialInvestment">
          Initial Investment ($):
          <input id="initialInvestment" type="number" v-model.number="initialInvestment" @input="validateInput('initialInvestment')" />
          <small>e.g., enter numbers only. No symbols.</small>
          <span v-if="errors.initialInvestment" class="error">{{ errors.initialInvestment }}</span>
        </label>
  
        <label for="monthlyContribution">
          Monthly Contribution ($):
          <input id="monthlyContribution" type="number" v-model.number="monthlyContribution" @input="validateInput('monthlyContribution')" />
          <small>e.g., enter numbers only. No symbols.</small>
          <span v-if="errors.monthlyContribution" class="error">{{ errors.monthlyContribution }}</span>
        </label>
  
        <label for="growthRate">
          Annual Growth Rate (%):
          <input id="growthRate" type="number" v-model.number="growthRate" @input="validateInput('growthRate')" />
          <small>e.g., enter numbers only. No symbols.</small>
          <span v-if="errors.growthRate" class="error">{{ errors.growthRate }}</span>
        </label>
  
        <label for="years">
          Number of Years:
          <input id="years" type="number" v-model.number="years" @input="validateInput('years')" />
          <small>e.g., enter numbers only. No symbols.</small>
          <span v-if="errors.years" class="error">{{ errors.years }}</span>
        </label>
  
        <label for="compoundingFrequency">
            
          Compounding Frequency:
          
          <select id="compoundingFrequency" v-model="compoundingFrequency">
            <option value="yearly">Yearly</option>
            <option value="monthly">Monthly</option>
          </select>
          <small>e.g., select Yearly or Monthly</small>
        </label>
  
        <!-- Inflation Toggle -->
        <label>
          <input type="checkbox" v-model="inflationAdjustmentEnabled" /> Apply Inflation Adjustment
        </label>
  
        <label v-if="inflationAdjustmentEnabled" for="customInflationRate">
          Inflation Rate (%):
          <input id="customInflationRate" type="number" v-model.number="customInflationRate" @input="validateInput('customInflationRate')" />
          <small>e.g., enter numbers only. No symbols.</small>
          <span v-if="errors.customInflationRate" class="error">{{ errors.customInflationRate }}</span>
        </label>
  
        <div class="button-row">
          <button class="calculate-button" @click="generateProjection" :disabled="hasErrors">Calculate</button>
          <button class="reset-button" @click="resetCalculator">Reset</button>
        </div>
      </div>
  
      <div class="chart-summary-container">
        <div class="summary">
          <h2>Summary</h2>
          <p><strong>User Contributions:</strong> {{ formatCurrency(animatedTotalInvested) }}</p>
          <p><strong>Total Interest:</strong> {{ formatCurrency(animatedTotalInterest) }}</p>
          <p><strong>Total Value ({{ inflationAdjustmentEnabled ? 'Inflation-Adjusted' : 'Nominal' }}):</strong> {{ formatCurrency(animatedFinalAccountValue) }}</p>
        </div>
  
        <div class="chart-container">
          <canvas id="investmentChart"></canvas>
        </div>
      </div>
  
      <div class="calculation-method">
        <p><strong>Calculation Notes:</strong></p>
        <p>Monthly contributions are compounded based on selected frequency.</p>
        <p v-if="inflationAdjustmentEnabled">Inflation is applied using user-defined rate of {{ customInflationRate }}%.</p>
        <p v-else>No inflation adjustment is currently applied.</p>
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref, computed, onMounted, nextTick } from 'vue'
  import { Chart, registerables } from 'chart.js'
  Chart.register(...registerables)
  
  // States
  const initialInvestment = ref(1000)
  const monthlyContribution = ref(500)
  const growthRate = ref(7)
  const years = ref(30)
  const compoundingFrequency = ref('yearly')
  
  const inflationAdjustmentEnabled = ref(false)
  const customInflationRate = ref(2)
  
  const totalInvested = ref(0)
  const finalAccountValue = ref(0)
  const totalInterest = ref(0)
  
  const animatedTotalInvested = ref(0)
  const animatedTotalInterest = ref(0)
  const animatedFinalAccountValue = ref(0)
  
  let investmentChart = null
  
  const errors = ref({
    initialInvestment: '',
    monthlyContribution: '',
    growthRate: '',
    years: '',
    customInflationRate: '',
  })
  
  // Validation
  const validateInput = (field) => {
    const valueMap = {
      initialInvestment: initialInvestment.value,
      monthlyContribution: monthlyContribution.value,
      growthRate: growthRate.value,
      years: years.value,
      customInflationRate: customInflationRate.value,
    }
  
    const value = valueMap[field]
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
  
  // Formatter
  const formatCurrency = (value) => {
    if (isNaN(value)) return '—'
    return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 }).format(value)
  }
  
  // Animation helper
  const animateValue = (targetRef, newValue) => {
    const duration = 800
    const frameDuration = 1000 / 60
    const totalFrames = Math.round(duration / frameDuration)
    const startValue = targetRef.value
    const increment = (newValue - startValue) / totalFrames
    let frame = 0
  
    const counter = setInterval(() => {
      frame++
      targetRef.value = Math.round(startValue + increment * frame)
      if (frame === totalFrames) {
        targetRef.value = Math.round(newValue)
        clearInterval(counter)
      }
    }, frameDuration)
  }
  
  // Chart generation
  const generateProjection = () => {
    if (hasErrors.value) return
  
    const canvas = document.getElementById('investmentChart')
    if (!canvas) return
  
    if (investmentChart) {
      investmentChart.destroy()
      investmentChart = null
    }
  
    const ctx = canvas.getContext('2d')
  
    const lineColor = 'rgba(75, 192, 192, 1)'
    const fillColor = 'rgba(75, 192, 192, 0.2)'
    const investedLineColor = 'rgba(255, 99, 132, 1)'
    const investedFillColor = 'rgba(255, 99, 132, 0.2)'
  
    const labels = []
    const totalInvestedData = []
    const accountValueData = []
  
    let currentValue = initialInvestment.value
    let cumulativeContributions = initialInvestment.value
  
    const compoundingPeriodsPerYear = compoundingFrequency.value === 'monthly' ? 12 : 1
    const totalPeriods = years.value * compoundingPeriodsPerYear
    const periodicContribution = compoundingFrequency.value === 'monthly'
      ? monthlyContribution.value
      : monthlyContribution.value * 12
  
    for (let period = 1; period <= totalPeriods; period++) {
      currentValue = (currentValue + periodicContribution) * (1 + (growthRate.value / 100 / compoundingPeriodsPerYear))
      cumulativeContributions += periodicContribution
  
      let inflationFactor = 1
      if (inflationAdjustmentEnabled.value) {
        const yearIndex = Math.floor(period / compoundingPeriodsPerYear)
        inflationFactor = Math.pow(1 + customInflationRate.value / 100, yearIndex)
      }
  
      labels.push(compoundingFrequency.value === 'monthly' ? `Month ${period}` : `Year ${period}`)
      totalInvestedData.push(Math.round(cumulativeContributions / inflationFactor))
      accountValueData.push(Math.round(currentValue / inflationFactor))
    }
  
    investmentChart = new Chart(ctx, {
      type: 'line',
      data: {
        labels,
        datasets: [
          {
            label: 'User Contributions ($):',
            data: totalInvestedData,
            borderColor: investedLineColor,
            backgroundColor: investedFillColor,
            fill: true,
            tension: 0.4,
            pointRadius: 0,
          },
          {
            label: 'Total Value ($):',
            data: accountValueData,
            borderColor: lineColor,
            backgroundColor: fillColor,
            fill: true,
            tension: 0.4,
            pointRadius: 0,
          },
        ],
      },
      options: {
        responsive: true,
        maintainAspectRatio: false,
        plugins: {
          legend: { display: true },
          tooltip: {
            mode: 'nearest',
            intersect: false,
            position: 'nearest',
            callbacks: {
              label: (context) => `${context.dataset.label}: ${formatCurrency(context.parsed.y)}`,
            },
          },
        },
        interaction: {
          mode: 'nearest',
          axis: 'x',
          intersect: false,
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
  
    totalInvested.value = totalInvestedData[totalInvestedData.length - 1]
    finalAccountValue.value = accountValueData[accountValueData.length - 1]
    totalInterest.value = finalAccountValue.value - totalInvested.value
  
    animateValue(animatedTotalInvested, totalInvested.value)
    animateValue(animatedFinalAccountValue, finalAccountValue.value)
    animateValue(animatedTotalInterest, totalInterest.value)
  }
  
  const resetCalculator = () => {
    initialInvestment.value = 1000
    monthlyContribution.value = 500
    growthRate.value = 7
    years.value = 30
    compoundingFrequency.value = 'yearly'
    inflationAdjustmentEnabled.value = false
    customInflationRate.value = 2
  
    totalInvested.value = 0
    finalAccountValue.value = 0
    totalInterest.value = 0
  
    animatedTotalInvested.value = 0
    animatedFinalAccountValue.value = 0
    animatedTotalInterest.value = 0
  
    Object.keys(errors.value).forEach(field => errors.value[field] = '')
  
    if (investmentChart) {
      investmentChart.destroy()
      investmentChart = null
    }
  }
  
  onMounted(async () => {
    await nextTick()
    generateProjection()
  })
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
  