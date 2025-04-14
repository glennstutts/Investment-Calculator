<template>
    <div class="investment-calculator">
      <div class="header">
        <h1>Investment Growth Calculator</h1>
      </div>
  
      <div class="form-stacked">
        <label>
          Initial Investment ($):
          <input type="number" v-model.number="initialInvestment" />
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
          Number of Years:
          <input type="number" v-model.number="years" />
        </label>
        <label>
          Compounding Frequency:
          <select v-model="compoundingFrequency">
            <option value="yearly">Yearly</option>
            <option value="monthly">Monthly</option>
          </select>
        </label>
  
        <div class="button-row">
          <button class="calculate-button" @click="generateProjection">Calculate</button>
          <button class="reset-button" @click="resetCalculator">Reset</button>
        </div>
      </div>
  
      <div class="chart-summary-container">
        <div class="summary">
          <h2>Summary</h2>
          <p><strong>User Contributions:</strong> {{ formatCurrency(animatedTotalInvested) }}</p>
          <p><strong>Total Interest:</strong> {{ formatCurrency(animatedTotalInterest) }}</p>
          <p><strong>Total Value:</strong> {{ formatCurrency(animatedFinalAccountValue) }}</p>
        </div>
  
        <div class="chart-container">
          <canvas id="investmentChart"></canvas>
        </div>
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref, onMounted, nextTick } from 'vue'
  import { Chart, registerables } from 'chart.js'
  
  Chart.register(...registerables)
  
  // Props from parent
  const props = defineProps({
    darkMode: {
      type: Boolean,
      default: true,
    }
  })
  
  // State
  const initialInvestment = ref(1000)
  const monthlyContribution = ref(500)
  const growthRate = ref(7)
  const years = ref(30)
  const compoundingFrequency = ref('yearly')
  
  const totalInvested = ref(0)
  const finalAccountValue = ref(0)
  const totalInterest = ref(0)
  
  const animatedTotalInvested = ref(0)
  const animatedTotalInterest = ref(0)
  const animatedFinalAccountValue = ref(0)
  
  let investmentChart = null
  
  // Lifecycle
  onMounted(async () => {
    await nextTick()
    generateProjection()
  })
  
  // Helpers
  const formatCurrency = (value) => {
    if (isNaN(value)) return '—'
    return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 }).format(value)
  }
  
  const getGradient = (ctx, color) => {
    const gradient = ctx.createLinearGradient(0, 0, 0, 400)
    gradient.addColorStop(0, color)
    gradient.addColorStop(1, 'rgba(255, 255, 255, 0)')
    return gradient
  }
  
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
  
  // Chart
  const generateProjection = () => {
    const canvas = document.getElementById('investmentChart')
    if (!canvas) return
  
    if (investmentChart) {
      investmentChart.destroy()
    }
  
    const ctx = canvas.getContext('2d')
  
    const lineColor = props.darkMode ? 'rgba(75, 192, 192, 1)' : 'rgba(54, 162, 235, 1)'
    const fillColor = props.darkMode ? 'rgba(75, 192, 192, 0.2)' : 'rgba(54, 162, 235, 0.2)'
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
  
      const label = compoundingFrequency.value === 'monthly' ? `Month ${period}` : `Year ${period}`
      labels.push(label)
      totalInvestedData.push(Math.round(cumulativeContributions))
      accountValueData.push(Math.round(currentValue))
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
            backgroundColor: getGradient(ctx, fillColor),
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
              labelTextColor: (context) => context.dataset.borderColor,
              labelColor: (context) => ({
                borderColor: context.dataset.borderColor,
                backgroundColor: context.dataset.borderColor,
              }),
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
  
    animateValue(animatedTotalInvested, cumulativeContributions)
    animateValue(animatedFinalAccountValue, currentValue)
    animateValue(animatedTotalInterest, currentValue - cumulativeContributions)
  }
  
  // Reset
  const resetCalculator = () => {
    initialInvestment.value = 1000
    monthlyContribution.value = 0
    growthRate.value = 7
    years.value = 30
    compoundingFrequency.value = 'yearly'
  
    totalInvested.value = 0
    finalAccountValue.value = 0
    totalInterest.value = 0
  
    animatedTotalInvested.value = 0
    animatedFinalAccountValue.value = 0
    animatedTotalInterest.value = 0
  
    if (investmentChart) {
      investmentChart.destroy()
      investmentChart = null
    }
  }
  </script>
  
  <style scoped>
  .investment-calculator {
    width: 100%;
    max-width: 900px;
    margin: 1rem auto;
    font-family: Arial, sans-serif;
    padding: 1.2rem;
    text-align: center;
    box-sizing: border-box;
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
  