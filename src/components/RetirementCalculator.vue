<template>
    <div class="retirement-calculator">
      <h2>Retirement Forecast & Smart Recommendations</h2>
      <p>Build your complete retirement scenario and generate a professional report.</p>
  
      <form @submit.prevent="generateForecast">
        <label>
          Current Age:
          <input type="number" v-model.number="currentAge" required />
        </label>
  
        <label>
          Target Retirement Age:
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
          Expected Annual Return (%):
          <input type="number" v-model.number="annualReturn" required />
        </label>
  
        <label>
          Annual Spending in Retirement ($):
          <input type="number" v-model.number="retirementSpending" required />
        </label>
  
        <button type="submit">Generate Forecast</button>
      </form>
  
      <div v-if="forecastGenerated" class="forecast-results">
        <h3>Forecast Results:</h3>
        <p><strong>Projected Savings at Retirement:</strong> {{ formatCurrency(projectedSavings) }}</p>
        <p><strong>Estimated Years Your Funds Will Last:</strong> {{ estimatedYearsCovered }}</p>
        <p><strong>Recommendations:</strong></p>
        <ul>
          <li v-for="tip in recommendations" :key="tip">{{ tip }}</li>
        </ul>
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref } from 'vue'
  
  const currentAge = ref(30)
  const retirementAge = ref(65)
  const currentSavings = ref(50000)
  const monthlyContribution = ref(1000)
  const annualReturn = ref(7)
  const retirementSpending = ref(40000)
  
  const forecastGenerated = ref(false)
  const projectedSavings = ref(0)
  const estimatedYearsCovered = ref(0)
  const recommendations = ref([])
  
  const formatCurrency = (value) => {
    return `$${value.toLocaleString()}`
  }
  
  const generateForecast = () => {
// --- Existing variables ---
let yearsToRetirement = retirementAge.value - currentAge.value
let savings = currentSavings.value

// Simulate saving to retirement
for (let year = 0; year < yearsToRetirement; year++) {
  savings += monthlyContribution.value * 12
  savings *= 1 + annualReturn.value / 100
}

projectedSavings.value = Math.round(savings)

// Simulate retirement withdrawals
let remainingSavings = savings
let yearsCovered = 0
const maxYears = 100

while (remainingSavings > 0 && yearsCovered < maxYears) {
  remainingSavings -= retirementSpending.value
  remainingSavings *= 1 + annualReturn.value / 100
  yearsCovered++
}

estimatedYearsCovered.value = yearsCovered

// --- Improved recommendations ---
recommendations.value = []

const targetRetirementDuration = 30 // assume 30-year retirement target

if (yearsCovered >= targetRetirementDuration) {
  recommendations.value.push('You are on track for a comfortable retirement!')
} else {
  const shortfallYears = targetRetirementDuration - yearsCovered
  recommendations.value.push(`Your current plan covers about ${yearsCovered} years of retirement. You are short by approximately ${shortfallYears} years.`)

  // Estimate needed monthly contribution increase
  let requiredMonthly = monthlyContribution.value
  let testSavings = currentSavings.value

  // Increase contributions until target met
  while (testSavings > 0 && yearsToRetirement > 0) {
    testSavings = currentSavings.value
    for (let year = 0; year < yearsToRetirement; year++) {
      testSavings += requiredMonthly * 12
      testSavings *= 1 + annualReturn.value / 100
    }

    let testRemaining = testSavings
    let testYearsCovered = 0
    while (testRemaining > 0 && testYearsCovered < targetRetirementDuration) {
      testRemaining -= retirementSpending.value
      testRemaining *= 1 + annualReturn.value / 100
      testYearsCovered++
    }

    if (testYearsCovered >= targetRetirementDuration) break
    requiredMonthly += 10 // increase in $10 increments for faster calculation
  }

  const additionalContribution = requiredMonthly - monthlyContribution.value
  if (additionalContribution > 0) {
    recommendations.value.push(`To meet your goal, consider increasing your monthly contributions by approximately $${additionalContribution.toFixed(0)}.`)
  }

  // Estimate needed retirement age adjustment
  let requiredRetirementAge = retirementAge.value
  let extendedSavings = currentSavings.value
  let extendedYearsToRetirement = requiredRetirementAge - currentAge.value

  while (extendedYearsToRetirement < 100) {
    extendedSavings = currentSavings.value
    for (let year = 0; year < extendedYearsToRetirement; year++) {
      extendedSavings += monthlyContribution.value * 12
      extendedSavings *= 1 + annualReturn.value / 100
    }

    let testRemaining = extendedSavings
    let testYearsCovered = 0
    while (testRemaining > 0 && testYearsCovered < targetRetirementDuration) {
      testRemaining -= retirementSpending.value
      testRemaining *= 1 + annualReturn.value / 100
      testYearsCovered++
    }

    if (testYearsCovered >= targetRetirementDuration) break
    requiredRetirementAge++
    extendedYearsToRetirement++
  }

  if (requiredRetirementAge > retirementAge.value && requiredRetirementAge < 100) {
    recommendations.value.push(`Alternatively, delaying retirement to around age ${requiredRetirementAge} could help meet your goal.`)
  }
}

  
    forecastGenerated.value = true
  }
  </script>
  
  <style scoped>
  .retirement-calculator {
    max-width: 600px;
    margin: 0 auto;
    text-align: left;
  }
  
  form {
    display: flex;
    flex-direction: column;
    gap: 1rem;
    margin-top: 1rem;
  }
  
  input {
    padding: 0.5rem;
    font-size: 1rem;
  }
  
  button {
    padding: 0.75rem;
    background-color: #4caf50;
    color: white;
    border: none;
    cursor: pointer;
    border-radius: 6px;
    transition: background-color 0.3s;
  }
  
  button:hover {
    background-color: #45a049;
  }
  
  .forecast-results {
  margin-top: 2rem;
  background-color: #1e1e1e;
  padding: 1rem;
  border-radius: 6px;
  color: white;
  box-shadow: 0 4px 8px rgba(0,0,0,0.3);
  border: 1px solid #333;
 /* ensure text is white in dark mode */
}

.forecast-results h3 {
  margin-bottom: 1rem;
  color: white;
}

.forecast-results p,
.forecast-results ul,
.forecast-results li {
  color: white;
}

  </style>
  