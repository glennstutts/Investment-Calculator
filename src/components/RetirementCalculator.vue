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
    let yearsToRetirement = retirementAge.value - currentAge.value
    let savings = currentSavings.value
  
    for (let year = 0; year < yearsToRetirement; year++) {
      savings += monthlyContribution.value * 12
      savings *= 1 + annualReturn.value / 100
    }
  
    projectedSavings.value = Math.round(savings)
  
    // Estimate how many years the funds will last in retirement
    let remainingSavings = savings
    let yearsCovered = 0
  
    while (remainingSavings > 0) {
      remainingSavings -= retirementSpending.value
      remainingSavings *= 1 + annualReturn.value / 100
      yearsCovered++
    }
  
    estimatedYearsCovered.value = yearsCovered
  
    // Generate recommendations
    recommendations.value = []
  
    if (yearsCovered < 20) {
      recommendations.value.push('Consider increasing your monthly contributions.')
      recommendations.value.push('Consider delaying retirement by a few years.')
      recommendations.value.push('Review your expected annual return or adjust spending.')
    } else {
      recommendations.value.push('You are on track for a comfortable retirement!')
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
    background-color: #f9f9f9;
    padding: 1rem;
    border-radius: 6px;
  }
  
  .forecast-results h3 {
    margin-bottom: 1rem;
  }
  </style>
  