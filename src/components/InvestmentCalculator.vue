<template>
    <div class="app">
      <div class="header">
        <h1>Investment Growth Calculator</h1>
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
  
      <!-- Form Layout -->
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
  
      <!-- Chart and Summary Side by Side -->
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
  
      <!-- Action Buttons -->
      <div class="button-group">
        <button @click="showReportModal = true">Download Report (CSV)</button>
        <button @click="downloadPDF">Export Report (PDF)</button>
      </div>
  
  <!-- Modal for report export -->
  <div v-if="showReportModal" class="modal-overlay">
    <div :class="['report-modal', darkMode ? 'dark' : '']">
      <!-- Modal content -->
      <h2>Export Report Options</h2>
  
          <label>
            Breakdown Frequency:
            <select v-model="exportFrequency">
              <option value="yearly">Yearly</option>
              <option value="monthly">Monthly</option>
            </select>
          </label>
  
          <label>
            Year Range:
            <select v-model="selectedReportOption">
              <option disabled value="">Please select</option>
              <option value="5">Next 5 years</option>
              <option value="10">Next 10 years</option>
              <option value="20">Next 20 years</option>
              <option value="25">Next 25 years</option>
              <option value="30">Next 30 years</option>
              <option value="40">Next 40 years</option>
              <option value="50">Next 50 years</option>
              <option value="custom">Custom date range</option>
            </select>
          </label>
  
          <div v-if="selectedReportOption === 'custom'" class="custom-range">
            <label>
              Start Year:
              <input type="number" v-model.number="customStartYear" min="1900" />
            </label>
            <label>
              End Year:
              <input type="number" v-model.number="customEndYear" :min="customStartYear" />
            </label>
          </div>
  
          <div class="modal-buttons">
            <button @click="generateCSVReport">Generate Report</button>
            <button @click="showReportModal = false">Cancel</button>
          </div>
        </div>
      </div>
    </div>
    
  </template>
  
  <script setup>
import { ref, watch, onMounted, onActivated, nextTick } from 'vue'
import { Chart, registerables } from 'chart.js'
import html2pdf from 'html2pdf.js'

Chart.register(...registerables)

// === State ===
const darkMode = ref(true)
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

const showReportModal = ref(false)
const selectedReportOption = ref('')
const customStartYear = ref(new Date().getFullYear())
const customEndYear = ref(new Date().getFullYear() + 10)
const exportFrequency = ref('yearly')

let investmentChart = null
const breakdownData = ref([])

// === Lifecycle Hooks ===
onMounted(async () => {
  document.body.classList.add('dark')
  await nextTick()
  generateProjection()
  window.addEventListener('keydown', handleKeyDown)
})

onActivated(async () => {
  await nextTick()
  generateProjection()
})

// === Handlers ===
const closeModal = () => {
  showReportModal.value = false
}

const handleKeyDown = (event) => {
  if (event.key === 'Escape' && showReportModal.value) {
    closeModal()
  }
}

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

// === Main Logic ===
const generateProjection = () => {
  const canvas = document.getElementById('investmentChart')
  if (!canvas) {
    console.warn('Canvas not found, skipping chart generation.')
    return
  }

  if (investmentChart) {
    investmentChart.destroy()
    investmentChart = null
  }

  const ctx = canvas.getContext('2d')

  const lineColor = darkMode.value ? 'rgba(75, 192, 192, 1)' : 'rgba(54, 162, 235, 1)'
  const fillColor = darkMode.value ? 'rgba(75, 192, 192, 0.2)' : 'rgba(54, 162, 235, 0.2)'
  const investedLineColor = darkMode.value ? 'rgba(255, 99, 132, 1)' : 'rgba(255, 99, 132, 1)'
  const investedFillColor = darkMode.value ? 'rgba(255, 99, 132, 0.2)' : 'rgba(255, 99, 132, 0.2)'

  const labels = []
  const totalInvestedData = []
  const accountValueData = []
  breakdownData.value = []

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

    const interestEarned = Math.round(currentValue - cumulativeContributions)
    const label = compoundingFrequency.value === 'monthly' ? `Month ${period}` : `Year ${period}`

    breakdownData.value.push({
      period: label,
      totalContributions: Math.round(cumulativeContributions),
      interestEarned,
      totalValue: Math.round(currentValue),
    })

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
      animation: {
        duration: 1500,
        easing: 'easeOutBounce',
        animateScale: true,
        animateRotate: true,
      },
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

  totalInvested.value = cumulativeContributions
  finalAccountValue.value = currentValue
  totalInterest.value = currentValue - cumulativeContributions
}

// === Watchers ===
watch(totalInvested, (newVal) => animateValue(animatedTotalInvested, newVal))
watch(finalAccountValue, (newVal) => animateValue(animatedFinalAccountValue, newVal))
watch(totalInterest, (newVal) => animateValue(animatedTotalInterest, newVal))
watch(darkMode, (newVal) => {
  if (newVal) {
    document.body.classList.add('dark')
  } else {
    document.body.classList.remove('dark')
  }
  generateProjection()
})

// === PDF Export ===
const downloadPDF = () => {
  const element = document.querySelector('.chart-summary-container')
  html2pdf().from(element).save('investment_report.pdf')
}

// === Reset Calculator ===
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

  breakdownData.value = []

  if (investmentChart) {
    investmentChart.destroy()
    investmentChart = null
  }
}

// === CSV Export ===
const generateCSVReport = () => {
  let startYear = new Date().getFullYear()
  let endYear = new Date().getFullYear()

  if (!selectedReportOption.value) {
    alert('Please select a range.')
    return
  }

  if (selectedReportOption.value === 'custom') {
    if (!customStartYear.value || !customEndYear.value || customEndYear.value < customStartYear.value) {
      alert('Please enter a valid custom year range.')
      return
    }
    startYear = customStartYear.value
    endYear = customEndYear.value
  } else {
    endYear = startYear + parseInt(selectedReportOption.value)
  }

  const reportRows = breakdownData.value.filter(row => {
    const isCorrectPeriod = exportFrequency.value === 'yearly'
      ? row.period.includes('Year')
      : row.period.includes('Month')

    if (!isCorrectPeriod) return false

    const match = row.period.match(/(?:Year|Month) (\d+)/)
    if (!match) return false

    const offset = parseInt(match[1]) - 1
    const periodYear = new Date().getFullYear() + Math.floor(offset / (exportFrequency.value === 'monthly' ? 12 : 1))

    return periodYear >= startYear && periodYear <= endYear
  })

  if (!reportRows.length) {
    alert('No data available for the selected range.')
    showReportModal.value = false
    return
  }

  let csvContent = 'data:text/csv;charset=utf-8,'
  csvContent += `Investment Report (${exportFrequency.value}) for ${startYear} - ${endYear}\n\n`
  csvContent += `Initial Investment:,${formatCurrency(initialInvestment.value)}\n`
  csvContent += `Monthly Contribution:,${formatCurrency(monthlyContribution.value)}\n`
  csvContent += `Annual Growth Rate:,${growthRate.value}%\n`
  csvContent += `Compounding Frequency:,${compoundingFrequency.value}\n\n`
  csvContent += 'Period,Total Contributions,Interest Earned,Total Value\n'

  reportRows.forEach(row => {
    csvContent += `${row.period},"${formatCurrency(row.totalContributions)}","${formatCurrency(row.interestEarned)}","${formatCurrency(row.totalValue)}"\n`
  })

  const encodedUri = encodeURI(csvContent)
  const link = document.createElement('a')
  link.setAttribute('href', encodedUri)
  link.setAttribute('download', `investment_report_${startYear}-${endYear}.csv`)
  document.body.appendChild(link)
  link.click()
  document.body.removeChild(link)

  showReportModal.value = false
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
  