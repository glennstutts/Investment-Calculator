<template>
  <div class="budget-planner">
    <h2 class="title">Budget Planner</h2>

    <!-- Income & Goal -->
    <div class="goal-row">
      <div class="input-group">
        <label for="income">Monthly Income ($):</label>
        <input type="number" id="income" v-model.number="income" class="black-input" />
      </div>
      <div class="input-group">
        <label for="goal">Savings Goal ($/mo):</label>
        <input type="number" id="goal" v-model.number="savingsGoal" class="black-input" />
      </div>
    </div>

    <!-- Expenses -->
    <div class="expenses">
      <h3>Expenses</h3>
      <div
        v-for="(expense, index) in expenses"
        :key="index"
        class="expense-row"
      >
        <input v-model="expense.category" :readonly="presetCategories.includes(expense.category)" class="black-input" />
        <input v-model.number="expense.amount" type="number" class="black-input" />
        <button @click="removeExpense(index)">❌</button>
      </div>
      <button @click="addExpense">+ Add Custom Category</button>
    </div>

    <div class="summary-and-chart" style="display: flex; justify-content: center; gap: 2rem; flex-wrap: wrap; align-items: flex-start;">
  <div class="summary" style="text-align: left;">
    <p><strong>Total Income:</strong> ${{ income.toFixed(2) }}</p>
    <p><strong>Total Expenses:</strong> ${{ totalExpenses.toFixed(2) }}</p>
    <p><strong>Balance:</strong> ${{ balance.toFixed(2) }}</p>
    <p><strong>Goal Met:</strong> {{ goalMet ? '✅' : '❌' }}</p>
  </div>
  <!-- Pie Chart -->
    <div class="chart-container" style="max-width: 550px; width: 100%;">
      <canvas ref="pieCanvas"></canvas>
    </div>
      </div>
  </div>
</template>

<script setup>
import Chart from 'chart.js/auto'
import { ref, computed, onMounted, watch } from 'vue'

const presetCategories = [
  'Rent/Mortgage', 'Utilities', 'Groceries', 'Transportation', 'Insurance',
  'Subscriptions', 'Debt Payments', 'Childcare', 'Entertainment', 'Dining Out', 'Travel', 'Miscellaneous'
]

const income = ref(0)
const savingsGoal = ref(0)
const expenses = ref([])
const pieCanvas = ref(null)

const totalExpenses = computed(() =>
  expenses.value.reduce((sum, e) => sum + (e.amount || 0), 0)
)
const balance = computed(() => income.value - totalExpenses.value)
const goalMet = computed(() => balance.value >= savingsGoal.value)

function addExpense() {
  expenses.value.push({ category: '', amount: 0 })
}
function removeExpense(index) {
  expenses.value.splice(index, 1)
}

onMounted(() => {
  income.value = 5000
  expenses.value = [
    { category: 'Rent/Mortgage', amount: 1500 },
    { category: 'Utilities', amount: 300 },
    { category: 'Groceries', amount: 600 },
    { category: 'Transportation', amount: 300 },
    { category: 'Insurance', amount: 400 },
    { category: 'Subscriptions', amount: 100 },
    { category: 'Debt Payments', amount: 350 },
    { category: 'Childcare', amount: 400 },
    { category: 'Entertainment', amount: 250 },
    { category: 'Dining Out', amount: 300 },
    { category: 'Travel', amount: 200 },
    { category: 'Miscellaneous', amount: 200 }
  ]

  const labels = expenses.value.map(e => e.category)
  const data = expenses.value.map(e => e.amount)

  const ctx = pieCanvas.value
  new Chart(ctx, {
    type: 'pie',
    data: {
      labels,
      datasets: [{
        data,
        backgroundColor: [
          '#4caf50', '#f44336', '#2196f3', '#ff9800', '#9c27b0', '#00bcd4',
          '#8bc34a', '#ffc107', '#e91e63', '#3f51b5', '#795548', '#009688', '#cddc39', '#607d8b'
        ]
      }]
    },
    options: {
      responsive: true,
      plugins: { legend: { position: 'right' } }
    }
  })
})


</script>

<style scoped>
.budget-planner {
  max-width: 800px;
  margin: 0 auto;
  text-align: center;
}

.title {
  margin-bottom: 1rem;
  font-size: 2rem;
}

.goal-row {
  display: flex;
  justify-content: center;
  gap: 1rem;
  flex-wrap: wrap;
  align-items: flex-end;
  margin-bottom: 1rem;
}

.input-group,
.expenses,
.summary {
  margin-bottom: 1rem;
}

.input-group input,
.expense-row input,
select {
  padding: 0.5rem;
  margin-top: 0.25rem;
  width: 100%;
  max-width: 300px;
  background: black;
  color: white;
  border: 1px solid #ccc;
}

.expense-row {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
}

.expense-row input {
  flex: 1;
}

.expense-row button,
button {
  padding: 0.5rem 1rem;
  background-color: #4caf50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

button:hover {
  background-color: #388e3c;
}

.summary p {
  margin: 0.3rem 0;
  font-size: 1rem;
}

.positive {
  color: #2e7d32;
}

.negative {
  color: #c62828;
}

.chart-container {
  max-width: 500px;
  margin: 1rem auto;
}
</style>
