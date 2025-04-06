<template>
  <div :class="['app', darkMode ? 'dark' : '']">
    <div class="header">
      <h1>Investment Growth Calculator</h1>
      <button class="dark-mode-toggle" @click="darkMode = !darkMode">
        {{ darkMode ? 'Light Mode' : 'Dark Mode' }}
      </button>
    </div>

    <!-- Stacked Form Layout -->
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

      <button class="calculate-button" @click="generateProjection">Calculate</button>
    </div>

    <!-- Chart and Summary Side by Side -->
    <div class="chart-summary-container">
      <div class="summary">
        <h2>Summary</h2>
        <p><strong>Total Invested:</strong> {{ formatCurrency(animatedTotalInvested) }}</p>
        <p><strong>Total Interest:</strong> {{ formatCurrency(animatedTotalInterest) }}</p>
        <p><strong>Total Value:</strong> {{ formatCurrency(animatedFinalAccountValue) }}</p>
      </div>

      <div class="chart-container">
        <canvas id="investmentChart"></canvas>
      </div>
    </div>

    <!-- Action Buttons -->
    <div class="button-group">
      <button @click="resetCalculator">Reset</button>
      <button @click="downloadChart">Download Chart as Image</button>
      <button @click="showReportModal = true">Download Report (CSV)</button>
      <button @click="downloadPDF">Export Report (PDF)</button>
    </div>

    <!-- Modal for report export -->
    <div v-if="showReportModal" class="modal-overlay">
      <div class="modal">
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
import { ref, onMounted, watch } from 'vue';
import { Chart, registerables } from 'chart.js';
import html2pdf from 'html2pdf.js';

Chart.register(...registerables);

// State variables
const darkMode = ref(false);
const initialInvestment = ref(1000);
const monthlyContribution = ref(0);
const growthRate = ref(7);
const years = ref(30);
const compoundingFrequency = ref('yearly');

const totalInvested = ref(0);
const finalAccountValue = ref(0);
const totalInterest = ref(0);

const animatedTotalInvested = ref(0);
const animatedTotalInterest = ref(0);
const animatedFinalAccountValue = ref(0);

let investmentChart = null;
const breakdownData = ref([]);

const showReportModal = ref(false);
const selectedReportOption = ref('');
const customStartYear = ref(new Date().getFullYear());
const customEndYear = ref(new Date().getFullYear() + 10);
const exportFrequency = ref('yearly');

const formatCurrency = (value) => {
  if (isNaN(value)) return '—';
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
    maximumFractionDigits: 0,
  }).format(value);
};

const animateValue = (targetRef, newValue) => {
  const duration = 800;
  const frameDuration = 1000 / 60;
  const totalFrames = Math.round(duration / frameDuration);
  const startValue = targetRef.value;
  const increment = (newValue - startValue) / totalFrames;
  let frame = 0;

  const counter = setInterval(() => {
    frame++;
    targetRef.value = Math.round(startValue + increment * frame);
    if (frame === totalFrames) {
      targetRef.value = Math.round(newValue);
      clearInterval(counter);
    }
  }, frameDuration);
};

const generateProjection = () => {
  const labels = [];
  const totalInvestedData = [];
  const accountValueData = [];
  breakdownData.value = [];

  let currentValue = initialInvestment.value;
  let totalInvestedAmount = initialInvestment.value;
  let cumulativeContributions = initialInvestment.value;

  const compoundingPeriodsPerYear = compoundingFrequency.value === 'monthly' ? 12 : 1;
  const totalPeriods = years.value * compoundingPeriodsPerYear;
  const periodicContribution = compoundingFrequency.value === 'monthly'
    ? monthlyContribution.value
    : monthlyContribution.value * 12;

  for (let period = 1; period <= totalPeriods; period++) {
    currentValue = (currentValue + periodicContribution) * (1 + (growthRate.value / 100 / compoundingPeriodsPerYear));
    cumulativeContributions += periodicContribution;

    const interestEarned = Math.round(currentValue - cumulativeContributions);
    const label = compoundingFrequency.value === 'monthly' ? `Month ${period}` : `Year ${period}`;

    breakdownData.value.push({
      period: label,
      totalContributions: Math.round(cumulativeContributions),
      interestEarned,
      totalValue: Math.round(currentValue),
    });

    labels.push(label);
    totalInvestedData.push(Math.round(cumulativeContributions));
    accountValueData.push(Math.round(currentValue));
  }

  if (investmentChart) investmentChart.destroy();

  const ctx = document.getElementById('investmentChart').getContext('2d');
  investmentChart = new Chart(ctx, {
    type: 'line',
    data: {
      labels,
      datasets: [
        {
          label: 'Total Invested',
          data: totalInvestedData,
          borderColor: 'rgba(255, 99, 132, 1)',
          backgroundColor: 'rgba(255, 99, 132, 0.2)',
          fill: true,
          tension: 0.4,
        },
        {
          label: 'Account Value Over Time',
          data: accountValueData,
          borderColor: 'rgba(75, 192, 192, 1)',
          backgroundColor: getGradient(ctx),
          fill: true,
          tension: 0.4,
        },
      ],
    },
    options: {
      responsive: true,
      animation: { duration: 1500, easing: 'easeOutQuart' },
      plugins: {
        legend: { display: true },
        tooltip: {
          callbacks: {
            label: (context) => `${context.dataset.label}: ${formatCurrency(context.parsed.y)}`
          }
        }
      },
      scales: {
        y: {
          ticks: {
            callback: (value) => formatCurrency(value),
          },
        },
      },
    },
  });

  totalInvested.value = totalInvestedAmount;
  finalAccountValue.value = currentValue;
  totalInterest.value = currentValue - totalInvestedAmount;
};

const getGradient = (ctx) => {
  const gradient = ctx.createLinearGradient(0, 0, 0, 400);
  gradient.addColorStop(0, 'rgba(75, 192, 192, 0.5)');
  gradient.addColorStop(1, 'rgba(255, 255, 255, 0)');
  return gradient;
};

watch(totalInvested, (newVal) => animateValue(animatedTotalInvested, newVal));
watch(finalAccountValue, (newVal) => animateValue(animatedFinalAccountValue, newVal));
watch(totalInterest, (newVal) => animateValue(animatedTotalInterest, newVal));

const downloadChart = () => {
  const link = document.createElement('a');
  link.download = 'investment-chart.png';
  link.href = investmentChart.toBase64Image();
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
};

const downloadPDF = () => {
  const element = document.querySelector('.chart-summary-container');
  html2pdf().from(element).save('investment_report.pdf');
};

const resetCalculator = () => {
  initialInvestment.value = 1000;
  monthlyContribution.value = 0;
  growthRate.value = 7;
  years.value = 30;
  compoundingFrequency.value = 'yearly';

  totalInvested.value = 0;
  finalAccountValue.value = 0;
  totalInterest.value = 0;

  animatedTotalInvested.value = 0;
  animatedFinalAccountValue.value = 0;
  animatedTotalInterest.value = 0;

  breakdownData.value = [];

  if (investmentChart) {
    investmentChart.destroy();
    investmentChart = null;
  }
};

// 🚀 CSV Export function will go here ✅
// (I will include in the next message to avoid cutoff)
</script>

<!-- Scoped style will go in the next message as well for full clean-up -->
const generateCSVReport = () => {
  let startYear = new Date().getFullYear();
  let endYear = new Date().getFullYear();

  if (!selectedReportOption.value) {
    alert('Please select a range.');
    return;
  }

  if (selectedReportOption.value === 'custom') {
    if (!customStartYear.value || !customEndYear.value || customEndYear.value < customStartYear.value) {
      alert('Please enter a valid custom year range.');
      return;
    }
    startYear = customStartYear.value;
    endYear = customEndYear.value;
  } else {
    endYear = startYear + parseInt(selectedReportOption.value);
  }

  const reportRows = breakdownData.value.filter(row => {
    const isCorrectPeriod = exportFrequency.value === 'yearly' ? row.period.includes('Year') : row.period.includes('Month');
    if (!isCorrectPeriod) return false;

    const match = row.period.match(/(?:Year|Month) (\d+)/);
    if (!match) return false;

    const offset = parseInt(match[1]) - 1;
    const periodYear = new Date().getFullYear() + Math.floor(offset / (exportFrequency.value === 'monthly' ? 12 : 1));

    return periodYear >= startYear && periodYear <= endYear;
  });

  if (!reportRows.length) {
    alert('No data available for the selected range.');
    showReportModal.value = false;
    return;
  }

  let csvContent = 'data:text/csv;charset=utf-8,';
  csvContent += `Investment Report (${exportFrequency.value}) for ${startYear} - ${endYear}\n\n`;
  csvContent += `Initial Investment:,${formatCurrency(initialInvestment.value)}\n`;
  csvContent += `Monthly Contribution:,${formatCurrency(monthlyContribution.value)}\n`;
  csvContent += `Annual Growth Rate:,${growthRate.value}%\n`;
  csvContent += `Compounding Frequency:,${compoundingFrequency.value}\n\n`;
  csvContent += 'Period,Total Contributions,Interest Earned,Total Value\n';

  reportRows.forEach(row => {
    csvContent += `${row.period},"${formatCurrency(row.totalContributions)}","${formatCurrency(row.interestEarned)}","${formatCurrency(row.totalValue)}"\n`;
  });

  const encodedUri = encodeURI(csvContent);
  const link = document.createElement('a');
  link.setAttribute('href', encodedUri);
  link.setAttribute('download', `investment_report_${startYear}-${endYear}.csv`);
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);

  showReportModal.value = false;
};
<style scoped>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

.app {
  max-width: 900px;
  margin: 2rem auto;
  font-family: Arial, sans-serif;
  padding: 1rem;
  background: #ffffff;
  color: #333;
  transition: background 0.3s, color 0.3s;
}

.dark {
  background: #121212;
  color: #f0f0f0;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
}

.dark-mode-toggle {
  padding: 0.5rem 1rem;
  border: none;
  background-color: #4caf50;
  color: white;
  cursor: pointer;
  border-radius: 4px;
}

.dark-mode-toggle:hover {
  background-color: #45a049;
}

.form-stacked {
  display: flex;
  flex-direction: column;
  width: 100%;
  max-width: 400px;
  margin: 0 auto 2rem auto;
  gap: 1rem;
}

.form-stacked label {
  display: flex;
  flex-direction: column;
  width: 100%;
  font-size: 0.95rem;
  font-weight: 500;
}

.form-stacked input,
.form-stacked select {
  display: block;
  width: 100%;
  padding: 0.7rem;
  font-size: 1rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  margin-top: 0.4rem;
  box-sizing: border-box;
}

.calculate-button {
  width: 100%;
  padding: 0.8rem;
  background-color: #4caf50;
  color: white;
  border: none;
  cursor: pointer;
  border-radius: 4px;
  text-align: center;
  font-size: 1rem;
}

.calculate-button:hover {
  background-color: #45a049;
}

.chart-summary-container {
  display: flex;
  flex-wrap: wrap;
  gap: 2rem;
  margin-bottom: 2rem;
  align-items: flex-start;
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
  flex: 2 1 500px;
}

canvas {
  width: 100%;
  height: auto;
}

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

.modal-buttons button {
  flex: 1;
  padding: 0.7rem;
  margin: 0 0.5rem;
  border: none;
  background-color: #4caf50;
  color: white;
  cursor: pointer;
  border-radius: 4px;
}

.modal-buttons button:hover {
  background-color: #45a049;
}

@media (max-width: 768px) {
  .chart-summary-container {
    flex-direction: column;
  }

  .summary,
  .chart-container {
    width: 100%;
  }
}
</style>
