<script setup>
import { ref, computed } from 'vue'

const FPL_2024 = { 1: 15060, 2: 20440, 3: 25820, 4: 31200 }

const currentStep = ref(1)
const totalSteps = 4

const hasMedicare = ref(null)
const isOver65 = ref(null)
const householdSize = ref(1)
const monthlyIncome = ref('')

const result = ref(null)

const progress = computed(() => (currentStep.value / totalSteps) * 100)

function nextStep() {
  if (currentStep.value < totalSteps) {
    currentStep.value++
  }
}

function prevStep() {
  if (currentStep.value > 1) {
    currentStep.value--
  }
}

function calculate() {
  // Check Medicare
  if (!hasMedicare.value) {
    result.value = { eligible: false, reason: 'Must be enrolled in Medicare to qualify for MSP.' }
    currentStep.value = 5
    return
  }

  // Check age
  if (!isOver65.value) {
    result.value = { eligible: false, reason: 'Must be 65 or older (or disabled) to qualify.' }
    currentStep.value = 5
    return
  }

  // Calculate income percentage
  const annualIncome = parseFloat(monthlyIncome.value) * 12
  const fpl = FPL_2024[householdSize.value]
  const percent = Math.round((annualIncome / fpl) * 100)

  if (percent <= 100) {
    result.value = { 
      eligible: true, 
      program: 'QMB',
      programName: 'Qualified Medicare Beneficiary',
      benefits: 'Medicare Part A & B premiums, deductibles, and coinsurance',
      percent 
    }
  } else if (percent <= 120) {
    result.value = { 
      eligible: true, 
      program: 'SLMB',
      programName: 'Specified Low-Income Medicare Beneficiary',
      benefits: 'Medicare Part B premium',
      percent 
    }
  } else if (percent <= 135) {
    result.value = { 
      eligible: true, 
      program: 'QI',
      programName: 'Qualifying Individual',
      benefits: 'Medicare Part B premium',
      percent 
    }
  } else {
    result.value = { 
      eligible: false, 
      reason: `Income at ${percent}% of Federal Poverty Level exceeds the 135% limit for MSP programs.`,
      percent 
    }
  }

  currentStep.value = 5
}

function reset() {
  currentStep.value = 1
  hasMedicare.value = null
  isOver65.value = null
  householdSize.value = 1
  monthlyIncome.value = ''
  result.value = null
}
</script>

<template>
  <div class="app">
    <!-- Header -->
    <header>
      <h1>MSP Eligibility Checker</h1>
      <p>Step-by-Step Eligibility Wizard</p>
      <span class="badge">Vendor Gamma — Form Wizard</span>
    </header>

    <main>
      <!-- Progress Bar -->
      <div class="progress-container" v-if="currentStep <= totalSteps">
        <div class="progress-bar">
          <div class="progress-fill" :style="{ width: progress + '%' }"></div>
        </div>
        <div class="progress-text">Step {{ currentStep }} of {{ totalSteps }}</div>
      </div>

      <!-- Step 1: Medicare -->
      <div class="card" v-if="currentStep === 1">
        <div class="step-icon">🏥</div>
        <h2>Medicare Enrollment</h2>
        <p class="step-description">Is the beneficiary currently enrolled in Medicare?</p>
        
        <div class="button-group">
          <button 
            :class="['choice-btn', { selected: hasMedicare === true }]"
            @click="hasMedicare = true"
          >
            Yes, enrolled in Medicare
          </button>
          <button 
            :class="['choice-btn', { selected: hasMedicare === false }]"
            @click="hasMedicare = false"
          >
            No, not enrolled
          </button>
        </div>

        <button 
          class="next-btn" 
          :disabled="hasMedicare === null"
          @click="nextStep"
        >
          Continue →
        </button>
      </div>

      <!-- Step 2: Age -->
      <div class="card" v-if="currentStep === 2">
        <div class="step-icon">🎂</div>
        <h2>Age Verification</h2>
        <p class="step-description">Is the beneficiary 65 years of age or older?</p>
        
        <div class="button-group">
          <button 
            :class="['choice-btn', { selected: isOver65 === true }]"
            @click="isOver65 = true"
          >
            Yes, 65 or older
          </button>
          <button 
            :class="['choice-btn', { selected: isOver65 === false }]"
            @click="isOver65 = false"
          >
            No, under 65
          </button>
        </div>

        <div class="nav-buttons">
          <button class="back-btn" @click="prevStep">← Back</button>
          <button 
            class="next-btn" 
            :disabled="isOver65 === null"
            @click="nextStep"
          >
            Continue →
          </button>
        </div>
      </div>

      <!-- Step 3: Household -->
      <div class="card" v-if="currentStep === 3">
        <div class="step-icon">👨‍👩‍👧</div>
        <h2>Household Size</h2>
        <p class="step-description">How many people are in the household?</p>
        
        <div class="select-group">
          <select v-model="householdSize">
            <option :value="1">1 person</option>
            <option :value="2">2 people</option>
            <option :value="3">3 people</option>
            <option :value="4">4 people</option>
          </select>
        </div>

        <div class="nav-buttons">
          <button class="back-btn" @click="prevStep">← Back</button>
          <button class="next-btn" @click="nextStep">Continue →</button>
        </div>
      </div>

      <!-- Step 4: Income -->
      <div class="card" v-if="currentStep === 4">
        <div class="step-icon">💰</div>
        <h2>Monthly Income</h2>
        <p class="step-description">What is the beneficiary's gross monthly income from all sources?</p>
        
        <div class="input-group">
          <span class="input-prefix">$</span>
          <input 
            type="number" 
            v-model="monthlyIncome" 
            placeholder="1200"
            min="0"
          />
        </div>

        <div class="nav-buttons">
          <button class="back-btn" @click="prevStep">← Back</button>
          <button 
            class="next-btn calculate" 
            :disabled="!monthlyIncome"
            @click="calculate"
          >
            Check Eligibility ✓
          </button>
        </div>
      </div>

      <!-- Result -->
      <div class="card result-card" v-if="currentStep === 5 && result">
        <div v-if="result.eligible" class="result eligible">
          <div class="result-icon">✓</div>
          <h2>Likely Eligible</h2>
          <div class="program-badge">{{ result.program }}</div>
          <p class="program-name">{{ result.programName }}</p>
          <p class="benefits"><strong>Benefits:</strong> {{ result.benefits }}</p>
          <p class="percent">Income at {{ result.percent }}% of Federal Poverty Level</p>
        </div>

        <div v-else class="result not-eligible">
          <div class="result-icon">✗</div>
          <h2>Not Eligible</h2>
          <p class="reason">{{ result.reason }}</p>
        </div>

        <button class="reset-btn" @click="reset">Start Over</button>
      </div>
    </main>

    <footer>
      <p><strong>Demo Only</strong> — Not for actual eligibility determinations</p>
      <p>Part of the <a href="https://github.com/mes-bakeoff-demo">MES Bake-Off Demo</a></p>
    </footer>
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

header {
  background: var(--primary);
  color: white;
  padding: 1.5rem 1rem;
  text-align: center;
}

header h1 {
  font-size: 1.5rem;
  font-weight: 600;
}

header p {
  opacity: 0.9;
  font-size: 0.9rem;
}

.badge {
  display: inline-block;
  background: var(--secondary);
  color: var(--primary);
  padding: 0.25rem 0.75rem;
  border-radius: 1rem;
  font-size: 0.75rem;
  font-weight: 600;
  margin-top: 0.5rem;
}

main {
  flex: 1;
  max-width: 500px;
  margin: 0 auto;
  padding: 1.5rem 1rem;
  width: 100%;
}

.progress-container {
  margin-bottom: 1.5rem;
}

.progress-bar {
  height: 8px;
  background: #e0e0e0;
  border-radius: 4px;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  background: var(--secondary);
  transition: width 0.3s ease;
}

.progress-text {
  text-align: center;
  font-size: 0.85rem;
  color: var(--text-light);
  margin-top: 0.5rem;
}

.card {
  background: white;
  border-radius: 12px;
  padding: 2rem 1.5rem;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  text-align: center;
}

.step-icon {
  font-size: 3rem;
  margin-bottom: 1rem;
}

.card h2 {
  color: var(--primary);
  font-size: 1.25rem;
  margin-bottom: 0.5rem;
}

.step-description {
  color: var(--text-light);
  margin-bottom: 1.5rem;
}

.button-group {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  margin-bottom: 1.5rem;
}

.choice-btn {
  padding: 1rem;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  background: white;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.2s;
}

.choice-btn:hover {
  border-color: var(--primary);
}

.choice-btn.selected {
  border-color: var(--secondary);
  background: #f1f8e9;
}

.select-group select,
.input-group input {
  width: 100%;
  padding: 1rem;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  font-size: 1rem;
  margin-bottom: 1.5rem;
}

.input-group {
  position: relative;
}

.input-prefix {
  position: absolute;
  left: 1rem;
  top: 50%;
  transform: translateY(-75%);
  color: var(--text-light);
  font-size: 1rem;
}

.input-group input {
  padding-left: 2rem;
}

.nav-buttons {
  display: flex;
  gap: 1rem;
}

.back-btn {
  flex: 1;
  padding: 1rem;
  border: 2px solid var(--primary);
  border-radius: 8px;
  background: white;
  color: var(--primary);
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
}

.next-btn {
  flex: 2;
  padding: 1rem;
  border: none;
  border-radius: 8px;
  background: var(--primary);
  color: white;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
}

.next-btn:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.next-btn.calculate {
  background: var(--secondary);
  color: var(--primary);
}

.result-card {
  padding: 2rem;
}

.result-icon {
  width: 64px;
  height: 64px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 2rem;
  margin: 0 auto 1rem;
}

.result.eligible .result-icon {
  background: #e8f5e9;
  color: var(--success);
}

.result.not-eligible .result-icon {
  background: #ffebee;
  color: var(--error);
}

.result.eligible h2 {
  color: var(--success);
}

.result.not-eligible h2 {
  color: var(--error);
}

.program-badge {
  display: inline-block;
  background: var(--secondary);
  color: var(--primary);
  padding: 0.5rem 1.5rem;
  border-radius: 4px;
  font-size: 1.25rem;
  font-weight: 700;
  margin: 1rem 0;
}

.program-name {
  font-weight: 500;
  margin-bottom: 0.5rem;
}

.benefits, .percent, .reason {
  color: var(--text-light);
  margin-bottom: 0.5rem;
}

.reset-btn {
  margin-top: 1.5rem;
  padding: 1rem 2rem;
  border: none;
  border-radius: 8px;
  background: var(--primary);
  color: white;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
}

footer {
  text-align: center;
  padding: 1.5rem 1rem;
  color: var(--text-light);
  font-size: 0.85rem;
}

footer a {
  color: var(--primary);
}
</style>
