<script setup>
import { ref, onMounted, computed, nextTick } from 'vue';
import axios from 'axios';
import { Line, Doughnut } from 'vue-chartjs';
import {
  Chart as ChartJS,
  Title,
  Tooltip,
  Legend,
  LineElement,
  LinearScale,
  PointElement,
  CategoryScale,
  ArcElement
} from 'chart.js';

// Register Chart.js components
ChartJS.register(Title, Tooltip, Legend, LineElement, LinearScale, PointElement, CategoryScale, ArcElement);

const showPinModal = ref(false);
const pinInput = ref('');
const pinError = ref('');
const pinInputRef = ref(null);
const showDriveModal = ref(false);
const pendingTrips = ref([]);
const showQueueDetails = ref(false);

const API_URL = 'https://skinny-kara-lynn-abhinavsharma-a4ea3b65.koyeb.app';
//const API_URL = 'http://192.168.1.5:2990';

// App State
const logs = ref([]);
// App State
const stats = ref({
  totalSpend: 0,
  totalLitres: 0,
  averageMileage: 0,
  predictedMonthlyExpense: 0,
  cityMileage: 0,    // Added to map backend properties
  highwayMileage: 0  // Added to map backend properties
});
const loading = ref(false);

// Station Shortcuts
const popularStations = ['IOCL', 'HPCL', 'BPCL'];

// Popular Indian cities for the quick dropdown list
const popularCities = ['Gurugram', 'New Delhi', 'Chandigarh', 'Mohali'];

// Form State
const form = ref({
  logDate: new Date().toISOString().split('T')[0],
  city: '',
  fuelStation: '',
  litres: null,
  odometerReading: null,
  amountSpent: null,
  ratePerLitre: null,
  fuelType: 'NORMAL',
  tripType: 'MIXED',
  cityPercentage: 50
});

const driveForm = ref({
  distanceKm: null,
  knownHighwayMileage: null,
  notes: '' // Reset default notes
});

const deletePendingTrip = async (id) => {
  try {
    await axios.delete(`${API_URL}/api/pending-trips/${id}`);
    await fetchPendingTrips();
    if (pendingTrips.value.length === 0) {
      showQueueDetails.value = false;
    }
  } catch (error) {
    console.error('Failed to delete pending trip:', error);
  }
};

const autoCalculatedRate = computed(() => {
  if (form.value.amountSpent && form.value.litres) {
    return (form.value.amountSpent / form.value.litres).toFixed(2);
  }
  return null;
});

const selectStation = (brand) => {
  form.value.fuelStation = brand;
};

// Chart Reactive Data Structures
const lineChartData = ref({ labels: [], datasets: [] });
const doughnutChartData = ref({ labels: [], datasets: [] });

const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: { labels: { color: '#94a3b8' } }
  },
  scales: {
    y: { grid: { color: '#334155' }, ticks: { color: '#94a3b8' } },
    x: { grid: { color: '#334155' }, ticks: { color: '#94a3b8' } }
  }
};

const doughnutOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: { position: 'bottom', labels: { color: '#94a3b8' } }
  }
};

const updateCharts = () => {
  if (logs.value.length === 0) return;

  // Chronological sort for the line trend (oldest to newest)
  const sortedLogs = [...logs.value].reverse();

  const labels = [];
  const overallPoints = [];
  const cityPoints = [];
  const highwayPoints = [];

  for (let i = 1; i < sortedLogs.length; i++) {
    const dist = sortedLogs[i].odometerReading - sortedLogs[i - 1].odometerReading;
    const eff = parseFloat((dist / sortedLogs[i].litres).toFixed(2));
    const currentLog = sortedLogs[i];

    labels.push(currentLog.logDate);
    overallPoints.push(eff);

    // FIX: Push matching indexes or null to maintain the array layout balance
    if (currentLog.tripType === 'CITY') {
      cityPoints.push(eff);
      highwayPoints.push(null);
    } else if (currentLog.tripType === 'HIGHWAY') {
      cityPoints.push(null);
      highwayPoints.push(eff);
    } else {
      // MIXED environments leave both scatter variants empty
      cityPoints.push(null);
      highwayPoints.push(null);
    }
  }

  lineChartData.value = {
    labels: labels.length ? labels : ['Need multiple entries'],
    datasets: [
      {
        label: 'Overall Trend (km/L)',
        data: overallPoints,
        borderColor: '#38bdf8',
        backgroundColor: 'rgba(56, 189, 248, 0.05)',
        tension: 0.3,
        pointRadius: 4,
        hitRadius: 10
      },
      {
        label: 'Pure City (km/L)',
        data: cityPoints, // Length matches labels perfectly now
        borderColor: '#f43f5e',
        backgroundColor: '#f43f5e',
        showLine: false,
        pointRadius: 6,
        pointStyle: 'circle'
      },
      {
        label: 'Pure Highway (km/L)',
        data: highwayPoints, // Length matches labels perfectly now
        borderColor: '#10b981',
        backgroundColor: '#10b981',
        showLine: false,
        pointRadius: 6,
        pointStyle: 'triangle'
      }
    ]
  };

  // Spending Split by Variant
  let normalSpend = 0;
  let xpSpend = 0;
  logs.value.forEach(log => {
    if (log.fuelType === 'NORMAL') normalSpend += log.amountSpent;
    if (log.fuelType === 'XP100') xpSpend += log.amountSpent;
  });

  doughnutChartData.value = {
    labels: ['Normal Petrol', 'Premium XP100'],
    datasets: [{
      data: [normalSpend, xpSpend],
      backgroundColor: ['#06b6d4', '#f59e0b'],
      borderWidth: 0
    }]
  };
};

const fetchDashboardData = async () => {
  loading.value = true;
  try {
    const [logsRes, statsRes] = await Promise.all([
      axios.get(`${API_URL}/api/fuel-logs`),
      axios.get(`${API_URL}/api/fuel-logs/stats`)
    ]);
    logs.value = logsRes.data;
    stats.value = statsRes.data;
    updateCharts();
    await fetchPendingTrips();
  } catch (error) {
    console.error("Backend unreachable:", error);
  } finally {
    loading.value = false;
  }
};

const submitLog = async () => {
  let cachedPin = localStorage.getItem('pitstop_admin_pin');

  if (!cachedPin) {
    pinInput.value = '';
    pinError.value = '';
    showPinModal.value = true;

    nextTick(() => {
      if (pinInputRef.value) pinInputRef.value.focus();
    });
    return;
  }

  await executeSecureSave(cachedPin);
};

const confirmPinSubmit = async () => {
  if (!/^\d{4}$/.test(pinInput.value)) {
    pinError.value = "PIN must be exactly 4 digits.";
    return;
  }

  const enteredPin = pinInput.value;
  pinError.value = '';

  await executeSecureSave(enteredPin);
};

const cancelPinSubmit = () => {
  showPinModal.value = false;
  pinInput.value = '';
  pinError.value = '';
};

const executeSecureSave = async (authPin) => {
  try {
    if (!form.value.ratePerLitre && autoCalculatedRate.value) {
      form.value.ratePerLitre = parseFloat(autoCalculatedRate.value);
    }

    await axios.post(API_URL + '/api/fuel-logs', form.value, {
      headers: { 'X-Admin-Pin': authPin }
    });

    localStorage.setItem('pitstop_admin_pin', authPin);
    showPinModal.value = false;

    form.value = {
      logDate: new Date().toISOString().split('T')[0],
      city: '',
      fuelStation: '',
      litres: null,
      odometerReading: null,
      amountSpent: null,
      ratePerLitre: null,
      fuelType: 'NORMAL',
      tripType: 'MIXED',
      cityPercentage: 50
    };

    await fetchDashboardData();
  } catch (error) {
    if (error.response && error.response.status === 401) {
      localStorage.removeItem('pitstop_admin_pin');
      pinInput.value = '';
      pinError.value = "Invalid PIN. Access denied.";
      showPinModal.value = true;

      nextTick(() => {
        if (pinInputRef.value) pinInputRef.value.focus();
      });
    } else {
      alert("Failed to save entry due to a transport error.");
    }
  }
};

const showDropdown = ref(false);

const selectCity = (city) => {
  form.value.city = city;
  showDropdown.value = false;
};

const toggleDropdown = () => {
  showDropdown.value = !showDropdown.value;
};

const hideDropdownDelayed = () => {
  setTimeout(() => {
    showDropdown.value = false;
  }, 200);
};

// 1. Fetch Pending Trips from Backend
const fetchPendingTrips = async () => {
  try {
    const response = await axios.get(`${API_URL}/api/pending-trips`);
    pendingTrips.value = response.data;
  } catch (error) {
    console.error('Error fetching pending trips:', error);
  }
};

// 2. Submit Quick Drive Log
const submitDriveLog = async () => {
  if (!driveForm.value.distanceKm || !driveForm.value.knownHighwayMileage) return;

  try {
    await axios.post(`${API_URL}/api/pending-trips`, driveForm.value, {
      headers: { 'Content-Type': 'application/json' }
    });

    // Reset form
    driveForm.value = { distanceKm: null, knownHighwayMileage: null, notes: '' };
    showDriveModal.value = false;

    // Refresh pending queue
    await fetchPendingTrips();
  } catch (error) {
    console.error('Failed to log drive:', error);
  }
};

onMounted(fetchDashboardData);
</script>

<template>
  <div class="dashboard-wrapper">
    <header class="app-header">
      <div class="brand">
        <span class="icon">⚡</span>
        <h1>PitStop Pro <span class="badge-live">Analytics Suite</span></h1>
      </div>
      <button @click="fetchDashboardData" :disabled="loading" class="refresh-btn">
        {{ loading ? 'Updating...' : '🔄 Sync Database' }}
      </button>
    </header>

    <!-- 1. High Contrast Cards Grid -->
    <div class="metrics-grid">
      <div class="kpi-card border-cyan">
        <span class="kpi-title">Avg Efficiency</span>
        <div class="kpi-value-row">
          <span class="value text-cyan">{{ stats.averageMileage ? stats.averageMileage.toFixed(2) : '0.00' }}</span>
          <span class="unit">km/L</span>
        </div>
        <!-- Environment Breakdown Rows -->
        <div class="kpi-sub-breakdown">
          <div class="sub-item">
            <span class="sub-label">🏙️ City:</span>
            <span class="sub-val text-rose">{{ stats.cityMileage ? stats.cityMileage.toFixed(1) : '0.0' }}
              <small>km/L</small></span>
          </div>
          <div class="sub-item">
            <span class="sub-label">🛣️ Hwy:</span>
            <span class="sub-val text-emerald">{{ stats.highwayMileage ? stats.highwayMileage.toFixed(1) : '0.0' }}
              <small>km/L</small></span>
          </div>
        </div>
      </div>

      <div class="kpi-card border-green">
        <span class="kpi-title">Gross Investment</span>
        <div class="kpi-value-row">
          <span class="value text-green">₹{{ stats.totalSpend ? stats.totalSpend.toLocaleString('en-IN') : '0' }}</span>
        </div>
        <!-- Dynamic Fuel Variant Cost Sub-Breakdown -->
        <div class="kpi-sub-breakdown">
          <div class="sub-item">
            <span class="sub-label">⚓ Normal:</span>
            <span class="sub-val text-cyan">
              ₹{{logs.reduce((acc, log) => log.fuelType === 'NORMAL' ? acc + log.amountSpent : acc,
                0).toLocaleString('en-IN')}}
            </span>
          </div>
          <div class="sub-item">
            <span class="sub-label">🔥 XP100:</span>
            <span class="sub-val text-amber">
              ₹{{logs.reduce((acc, log) => log.fuelType === 'XP100' ? acc + log.amountSpent : acc,
                0).toLocaleString('en-IN')}}
            </span>
          </div>
        </div>
      </div>

      <div class="kpi-card prediction-card">
        <span class="kpi-title text-orange">Predicted 30-Day Runrate</span>
        <div class="kpi-value-row">
          <span class="value text-orange">₹{{ stats.predictedMonthlyExpense ?
            stats.predictedMonthlyExpense.toLocaleString('en-IN', { maximumFractionDigits: 0 }) : '0' }}</span>
        </div>
      </div>

      <div class="kpi-card">
        <span class="kpi-title">Volume Pumped</span>
        <div class="kpi-value-row">
          <span class="value">{{ stats.totalLitres ? stats.totalLitres.toFixed(1) : '0.0' }}</span>
          <span class="unit">Liters</span>
        </div>
      </div>
    </div>

    <!-- 2. Interactive Graphical Visualizers Section -->
    <div class="charts-section" v-if="logs.length > 1">
      <div class="chart-holder main-line">
        <h3>Mileage Performance Log</h3>
        <div class="canvas-container">
          <Line :data="lineChartData" :options="chartOptions" />
        </div>
      </div>
      <div class="chart-holder donut-share">
        <h3>Expense Share By Variant</h3>
        <div class="canvas-container">
          <Doughnut :data="doughnutChartData" :options="doughnutOptions" />
        </div>
      </div>
    </div>

    <!-- 3. Working Operations Layout -->
    <div class="main-layout">
      <!-- Input Side Panel -->
      <aside class="panel-form">
        <div class="flex-justify-between align-center mb-2">
          <h2>Refuel Telemetry</h2>
          <button type="button" @click="showDriveModal = true" class="btn-secondary-action">
            🛣️ + Drive Log
          </button>
        </div>

        <!-- Interactive Pending Trips Queue Indicator Badge -->
        <div v-if="pendingTrips && pendingTrips.length > 0" @click="showQueueDetails = true"
          class="pending-queue-badge clickable-badge" title="Click to view queued drives">
          <span>📍 {{ pendingTrips.length }} drive(s) queued for next refuel 🔍</span>
        </div>

        <form @submit.prevent="submitLog" class="styled-form">
          <div class="input-row">
            <div class="input-block">
              <label>Transaction Date</label>
              <input type="date" v-model="form.logDate" required
                style="position: relative !important; width: 100% !important; max-width: 100% !important; box-sizing: border-box !important;" />
            </div>
            <div class="input-block">
            </div>
          </div>

          <div class="input-row">
            <div class="input-block city-select-wrapper">
              <label>City</label>
              <div class="hybrid-input-container">
                <input type="text" v-model="form.city" placeholder="Select or type city..." @focus="showDropdown = true"
                  @blur="hideDropdownDelayed" required />
                <span class="dropdown-arrow-indicator" @click="toggleDropdown">▼</span>

                <!-- Custom overlay dropdown menu -->
                <ul v-if="showDropdown" class="custom-dropdown-list">
                  <li v-for="city in popularCities" :key="city" @mousedown="selectCity(city)">
                    {{ city }}
                  </li>
                </ul>
              </div>
            </div>
            <div class="input-block">
              <label>Station Outlet</label>
              <input type="text" v-model="form.fuelStation" placeholder="e.g., Indian Oil" required />
              <div class="brand-chips">
                <span v-for="brand in popularStations" :key="brand" @click="selectStation(brand)" class="chip"
                  :class="{ active: form.fuelStation === brand }">
                  {{ brand.split(' ')[0] }}
                </span>
              </div>
            </div>
          </div>

          <div class="input-row">
            <div class="input-block">
              <label>Litres Injected</label>
              <input type="number" step="0.01" inputmode="decimal" v-model="form.litres" placeholder="0.00" required />
            </div>
            <div class="input-block">
              <label>Odometer (km)</label>
              <input type="number" inputmode="decimal" v-model="form.odometerReading" placeholder="Current km"
                required />
            </div>
          </div>

          <div class="input-row">
            <div class="input-block">
              <label>Total Paid (₹)</label>
              <input type="number" inputmode="decimal" v-model="form.amountSpent" placeholder="Total Rs." required />
            </div>
            <div class="input-block">
              <label>Rate / L (₹)</label>
              <input type="number" step="0.01" inputmode="decimal" v-model="form.ratePerLitre"
                :placeholder="autoCalculatedRate || 'Auto-computed'" />
            </div>
          </div>

          <div class="input-block">
            <label>Fuel Variant Type</label>
            <div class="radio-toggle">
              <label class="radio-label">
                <input type="radio" value="NORMAL" v-model="form.fuelType"> Normal
              </label>
              <label class="radio-label">
                <input type="radio" value="XP100" v-model="form.fuelType"> Premium XP100
              </label>
            </div>
          </div>

          <div class="input-block">
            <label>Driving Environment</label>
            <div class="radio-toggle">
              <label class="radio-label">
                <input type="radio" value="CITY" v-model="form.tripType" @change="form.cityPercentage = 100"> 🏙️ City
              </label>
              <label class="radio-label">
                <input type="radio" value="HIGHWAY" v-model="form.tripType" @change="form.cityPercentage = 0"> 🛣️
                Highway
              </label>
              <label class="radio-label">
                <input type="radio" value="MIXED" v-model="form.tripType" @change="form.cityPercentage = 50"> 🔄 Mixed
              </label>
            </div>
          </div>

          <!-- Expanded Mixed Settings Block -->
          <template v-if="form.tripType === 'MIXED'">
            <div class="input-block">
              <div class="flex-justify-between">
                <label>Usage Split</label>
                <span v-if="pendingTrips && pendingTrips.length > 0" class="text-cyan font-semibold"
                  style="font-size: 12px;">
                  ✨ Auto-calculating from {{ pendingTrips.length }} drive(s)
                </span>
                <span v-else class="text-cyan font-semibold" style="font-size: 12px;">
                  {{ form.cityPercentage }}% City / {{ 100 - form.cityPercentage }}% Highway
                </span>
              </div>
              <input type="range" min="10" max="90" step="5" v-model.number="form.cityPercentage" class="custom-slider"
                :disabled="pendingTrips && pendingTrips.length > 0" />
            </div>

            <div class="input-block">
              <div class="flex-justify-between">
                <label>Dash Highway Mileage <span
                    style="font-size: 11px; opacity: 0.6; font-weight: normal;">(Optional)</span></label>
                <span class="text-cyan font-semibold" style="font-size: 12px;" v-if="form.knownHighwayMileage">
                  {{ form.knownHighwayMileage }} km/L
                </span>
              </div>
              <input type="number" step="0.1" min="1" max="50" v-model.number="form.knownHighwayMileage"
                placeholder="e.g. 20.0" class="form-input" />
            </div>
          </template>

          <button type="submit" class="submit-action-btn">Save Details</button>
        </form>
      </aside>

      <!-- History Data List View -->
      <main class="panel-table">
        <h2>Historical Registry</h2>
        <div class="responsive-table-holder">
          <table class="data-table">
            <thead>
              <tr>
                <th>Date</th>
                <th>Location Details</th>
                <th>Odometer</th>
                <th>Litres</th>
                <th>Total Bill</th>
                <th>Variant / Run Runrate</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(log, index) in logs" :key="log.id" class="table-data-row">
                <td data-label="Date" class="font-date">{{ log.logDate }}</td>
                <td data-label="Location Details">
                  <div class="location-group">
                    <div class="station-txt">{{ log.fuelStation }}</div>
                    <div class="city-txt">{{ log.city }}</div>
                  </div>
                </td>
                <td data-label="Odometer">{{ log.odometerReading.toLocaleString('en-IN') }} km</td>
                <td data-label="Litres">{{ log.litres }} L</td>
                <td data-label="Total Bill" class="text-green font-semibold">₹{{ log.amountSpent.toLocaleString('en-IN')
                  }}</td>
                <td data-label="Variant / Run Runrate">
                  <div class="flex-badges">
                    <div style="display: flex; gap: 4px; align-items: center; flex-wrap: wrap;">
                      <span :class="['variant-tag', log.fuelType.toLowerCase()]">{{ log.fuelType }}</span>
                      <span :class="['environment-badge', (log.tripType || 'MIXED').toLowerCase()]">
                        {{ log.tripType === 'MIXED' ? `${log.cityPercentage || 50}% City` : log.tripType }}
                      </span>
                    </div>

                    <!-- Sequential Metrics Group -->
                    <template v-if="index < logs.length - 1">
                      <span class="trip-mileage-badge">
                        🍃 {{ ((logs[index].odometerReading - logs[index + 1].odometerReading) /
                          logs[index].litres).toFixed(1) }} km/L
                      </span>
                      <span class="trip-cpk-badge">
                        ₹{{ (logs[index].amountSpent / (logs[index].odometerReading - logs[index +
                          1].odometerReading)).toFixed(2) }}/km
                      </span>
                    </template>
                  </div>
                </td>
              </tr>
              <tr v-if="logs.length === 0" class="empty-state-row">
                <td colspan="6" class="empty-state">No records available inside MySQL.</td>
              </tr>
            </tbody>
          </table>
        </div>
      </main>
    </div>
  </div>

  <!-- 1. Quick Drive Logger Modal (With Notes Input) -->
  <div v-if="showDriveModal" class="pin-modal-overlay">
    <div class="pin-modal-card">
      <div class="pin-modal-header">
        <h3>🛣️ Quick Drive Logger</h3>
        <p>Record a trip readout right after driving</p>
      </div>

      <div class="pin-modal-body" style="display: flex; flex-direction: column; gap: 12px; text-align: left;">
        <div class="input-block">
          <label>Distance Driven (km)</label>
          <input v-model.number="driveForm.distanceKm" type="number" step="0.1" inputmode="decimal"
            placeholder="e.g. 214.5" class="pin-input-field"
            style="text-align: left; font-size: 16px; letter-spacing: normal;" />
        </div>

        <div class="input-block">
          <label>Dash Highway Mileage (km/L)</label>
          <input v-model.number="driveForm.knownHighwayMileage" type="number" step="0.1" inputmode="decimal"
            placeholder="e.g. 20.0" class="pin-input-field"
            style="text-align: left; font-size: 16px; letter-spacing: normal;" />
        </div>

        <!-- NEW: Optional Notes Field -->
        <div class="input-block">
          <label>Notes <span style="font-size: 11px; opacity: 0.6; font-weight: normal;">(Optional)</span></label>
          <input v-model="driveForm.notes" type="text" placeholder="e.g. Expressway run, moderate traffic"
            class="pin-input-field" style="text-align: left; font-size: 14px; letter-spacing: normal;" />
        </div>
      </div>

      <div class="pin-modal-actions">
        <button @click="showDriveModal = false" class="btn-cancel">Cancel</button>
        <button @click="submitDriveLog" class="btn-confirm">Queue Drive</button>
      </div>
    </div>
  </div>

  <!-- 2. NEW: Queued Drives Inspector Modal -->
  <div v-if="showQueueDetails" class="pin-modal-overlay">
    <div class="pin-modal-card" style="max-width: 480px; width: 90%;">
      <div class="pin-modal-header">
        <h3>📍 Queued Drives</h3>
        <p>Pending drives that will be calculated into your next refuel</p>
      </div>

      <div class="pin-modal-body" style="text-align: left; max-height: 300px; overflow-y: auto;">
        <div v-for="trip in pendingTrips" :key="trip.id"
          style="background: rgba(255,255,255,0.05); padding: 10px; border-radius: 8px; margin-bottom: 8px; border: 1px solid rgba(255,255,255,0.1); display: flex; justify-content: space-between; align-items: center;">
          <div>
            <div style="font-weight: 600; color: #38bdf8;">
              🛣️ {{ trip.distanceKm }} km @ {{ trip.knownHighwayMileage }} km/L
            </div>
            <div style="font-size: 12px; color: #94a3b8; margin-top: 2px;">
              🗓️ {{ trip.logDate || 'Today' }} <span v-if="trip.notes">• 📝 {{ trip.notes }}</span>
            </div>
          </div>
          <button @click="deletePendingTrip(trip.id)"
            style="background: #ef4444; border: none; color: white; padding: 4px 8px; border-radius: 4px; cursor: pointer; font-size: 12px;">
            🗑️
          </button>
        </div>
      </div>

      <div class="pin-modal-actions">
        <button @click="showQueueDetails = false" class="btn-confirm" style="width: 100%;">Close</button>
      </div>
    </div>
  </div>

  <!-- 3. Admin Authorization PIN Modal -->
  <div v-if="showPinModal" class="pin-modal-overlay">
    <div class="pin-modal-card">
      <div class="pin-modal-header">
        <h3>Admin Authorization</h3>
        <p>Enter your 4-digit PIN to save the details</p>
      </div>

      <div class="pin-modal-body">
        <input v-model="pinInput" type="password" inputmode="numeric" pattern="\d*" maxlength="4" placeholder="••••"
          class="pin-input-field" :class="{ 'input-shake-error': pinError }" @keyup.enter="confirmPinSubmit"
          ref="pinInputRef" />
        <div v-if="pinError" class="pin-error-text">❌ {{ pinError }}</div>
      </div>

      <div class="pin-modal-actions">
        <button @click="cancelPinSubmit" class="btn-cancel">Cancel</button>
        <button @click="confirmPinSubmit" class="btn-confirm">Verify & Save</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Core Layout Structure & Defensively Restricting Viewport Overflow */
.dashboard-wrapper {
  max-width: 1400px;
  margin: 0 auto;
  padding: 12px;
  background-color: #0f172a;
  min-height: 100vh;
  color: #e2e8f0;
  font-family: system-ui, sans-serif;
  box-sizing: border-box;
  overflow-x: hidden;
  width: 100%;
}

/* Aggressive box-sizing layout reset */
.dashboard-wrapper *,
.dashboard-wrapper *::before,
.dashboard-wrapper *::after {
  box-sizing: border-box !important;
}

.app-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  border-bottom: 1px solid #334155;
  padding-bottom: 12px;
  flex-wrap: wrap;
  gap: 12px;
  width: 100%;
}

.brand h1 {
  font-size: 20px;
  font-weight: 800;
  color: #fff;
  margin: 0;
  display: flex;
  align-items: center;
  gap: 8px;
}

.badge-live {
  font-size: 10px;
  background: #0284c7;
  padding: 2px 6px;
  border-radius: 4px;
  font-weight: 600;
}

.refresh-btn {
  background: #1e293b;
  border: 1px solid #475569;
  color: #cbd5e1;
  padding: 8px 14px;
  border-radius: 8px;
  cursor: pointer;
  font-weight: 600;
  font-size: 12px;
  transition: all 0.2s;
}

.refresh-btn:hover {
  background: #334155;
  color: #fff;
}

/* KPI Cards Layout */
.metrics-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 10px;
  margin-bottom: 20px;
  width: 100%;
}

.kpi-card {
  background: #1e293b;
  border-radius: 12px;
  padding: 14px;
  border: 1px solid #334155;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  min-width: 0;
}

.border-cyan {
  border-left: 4px solid #06b6d4;
}

.border-green {
  border-left: 4px solid #10b981;
}

.prediction-card {
  border-left: 4px solid #f59e0b;
  background: #252e3e;
}

.kpi-title {
  display: block;
  font-size: 10px;
  color: #94a3b8;
  text-transform: uppercase;
  margin-bottom: 4px;
}

.kpi-card .value {
  font-size: 18px;
  font-weight: 700;
  color: #fff;
  word-break: break-word;
  overflow: hidden;
}

.text-cyan {
  color: #06b6d4 !important;
}

.text-green {
  color: #10b981 !important;
}

.text-orange {
  color: #f59e0b !important;
}

.kpi-card .unit {
  font-size: 11px;
  color: #64748b;
  margin-left: 2px;
}

/* Graphical Charts Section Layout */
.charts-section {
  display: flex;
  flex-direction: column;
  gap: 16px;
  margin-bottom: 20px;
  width: 100%;
}

.chart-holder {
  background: #1e293b;
  border: 1px solid #334155;
  border-radius: 12px;
  padding: 16px;
  width: 100%;
  min-width: 0;
}

.chart-holder h3 {
  font-size: 14px;
  margin-top: 0;
  color: #94a3b8;
  margin-bottom: 12px;
}

.canvas-container {
  height: 220px;
  position: relative;
  width: 100%;
}

/* Core Structural Panels */
.main-layout {
  display: flex;
  flex-direction: column;
  gap: 20px;
  width: 100%;
}

.panel-form,
.panel-table {
  background: #1e293b;
  border: 1px solid #334155;
  border-radius: 12px;
  padding: 16px;
  width: 100%;
  max-width: 100%;
  min-width: 0;
  overflow: hidden;
}

.panel-form h2,
.panel-table h2 {
  font-size: 16px;
  margin-top: 0;
  color: #fff;
  margin-bottom: 16px;
}

.styled-form {
  display: flex;
  flex-direction: column;
  gap: 14px;
  width: 100%;
  min-width: 0;
}

.input-row {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
  width: 100%;
  min-width: 0;
}

.input-block {
  display: flex;
  flex-direction: column;
  gap: 6px;
  width: 100%;
  min-width: 0;
}

.input-block label {
  font-size: 12px;
  color: #94a3b8;
  font-weight: 600;
}

.input-block input,
.input-block select {
  padding: 10px;
  background: #0f172a;
  border: 1px solid #475569;
  border-radius: 6px;
  color: #fff;
  font-size: 14px;
  width: 100%;
  max-width: 100%;
}

/* FIXED: Nuclear option for the date row input elements */
.input-block input[type="date"],
.styled-form input[type="date"],
input[type="date"] {
  position: relative !important;
  cursor: pointer;
  padding: 10px 12px !important;
  background-color: #0f172a !important;
  color: #fff !important;
  border: 1px solid #475569 !important;
  border-radius: 6px !important;
  font-family: inherit !important;
  width: 100% !important;
  max-width: 100% !important;
  box-sizing: border-box !important;
  text-align: center;
  display: block !important;
}

/* FIXED: Stripping the invisible picker layout bounds so it cannot poke out of the gray panel */
.input-block input[type="date"]::-webkit-calendar-picker-indicator,
.styled-form input[type="date"]::-webkit-calendar-picker-indicator,
input[type="date"]::-webkit-calendar-picker-indicator {
  position: absolute !important;
  top: 0 !important;
  left: 0 !important;
  width: 100% !important;
  height: 100% !important;
  margin: 0 !important;
  padding: 0 !important;
  opacity: 0 !important;
  background: transparent !important;
  color: transparent !important;
  cursor: pointer !important;
}

.input-block input[type="date"]:focus {
  border-color: #38bdf8;
  box-shadow: 0 0 0 2px rgba(56, 189, 248, 0.2);
  outline: none;
}

.input-block input:focus {
  border-color: #38bdf8;
  outline: none;
}

.brand-chips {
  display: flex;
  gap: 6px;
  margin-top: 4px;
  flex-wrap: wrap;
}

.chip {
  font-size: 11px;
  background: #0f172a;
  border: 1px solid #475569;
  padding: 4px 8px;
  border-radius: 4px;
  cursor: pointer;
  color: #94a3b8;
  transition: all 0.2s;
}

.chip:hover {
  border-color: #38bdf8;
  color: #fff;
}

.chip.active {
  background: #38bdf8;
  border-color: #38bdf8;
  color: #0f172a;
  font-weight: 600;
}

.radio-toggle {
  display: flex;
  gap: 16px;
  padding: 4px 0;
  flex-wrap: wrap;
}

.radio-label {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  font-size: 14px;
  cursor: pointer;
  color: #fff;
  white-space: nowrap;
}

.radio-label input[type="radio"] {
  width: auto;
  margin: 0;
  cursor: pointer;
}

.submit-action-btn {
  background: #0284c7;
  color: #fff;
  border: none;
  padding: 12px;
  font-weight: 700;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  margin-top: 4px;
  width: 100%;
  transition: background 0.2s;
}

.submit-action-btn:hover {
  background: #0369a1;
}

/* Custom premium track slider adjustments */
.flex-justify-between {
  display: flex;
  justify-content: space-between;
  width: 100%;
  align-items: center;
}

.font-semibold {
  font-weight: 600;
}

.custom-slider {
  -webkit-appearance: none;
  width: 100%;
  height: 6px;
  background: #0f172a;
  border: 1px solid #334155;
  border-radius: 4px;
  outline: none;
  margin: 8px 0;
}

.custom-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background: #38bdf8;
  cursor: pointer;
  box-shadow: 0 0 8px rgba(56, 189, 248, 0.5);
}

.environment-badge {
  padding: 2px 6px;
  border-radius: 4px;
  font-size: 10px;
  font-weight: 700;
  text-transform: uppercase;
}

.environment-badge.city {
  background: rgba(244, 63, 94, 0.15);
  color: #f43f5e;
  border: 1px solid rgba(244, 63, 94, 0.3);
}

.environment-badge.highway {
  background: rgba(16, 185, 129, 0.15);
  color: #10b981;
  border: 1px solid rgba(16, 185, 129, 0.3);
}

.environment-badge.mixed {
  background: rgba(6, 182, 212, 0.15);
  color: #06b6d4;
  border: 1px solid rgba(6, 182, 212, 0.3);
}

/* ---------------------------------------------------
   HISTORICAL REGISTRY MOBILE CARDS / DESKTOP TABLE
--------------------------------------------------- */

/* Container safety boundary */
.responsive-table-holder {
  width: 100%;
  overflow-x: hidden;
}

/* Base Mobile View: Transform table elements into block cards */
.data-table {
  width: 100%;
  border-collapse: collapse;
  display: block;
}

.data-table thead {
  display: none;
}

.data-table tbody {
  display: block;
  width: 100%;
}

/* Each row turns into a self-contained card block */
.data-table tr:not(.empty-state-row) {
  display: flex;
  flex-direction: column;
  background: #0f172a;
  border: 1px solid #334155;
  border-radius: 8px;
  padding: 12px;
  margin-bottom: 12px;
  width: 100%;
}

/* Normalize table cells into standard vertical blocks */
.data-table td {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 6px 0;
  border-bottom: 1px solid rgba(51, 65, 85, 0.4);
  width: 100%;
  font-size: 13px;
  text-align: right;
}

.data-table td:last-child {
  border-bottom: none;
}

/* Dynamically generate column headers via data attributes on mobile */
.data-table td::before {
  content: attr(data-label);
  font-weight: 600;
  color: #94a3b8;
  font-size: 12px;
  text-transform: uppercase;
  text-align: left;
}

.font-date {
  color: #38bdf8;
  font-weight: 600;
}

.station-txt {
  color: #fff;
  font-weight: 600;
  display: block;
  text-align: right;
}

.city-txt {
  font-size: 11px;
  color: #64748b;
  display: block;
  text-align: right;
  margin-top: 2px;
}

.flex-badges {
  display: flex;
  gap: 6px;
  align-items: center;
  flex-wrap: wrap;
}

.variant-tag {
  padding: 2px 6px;
  border-radius: 4px;
  font-size: 10px;
  font-weight: 700;
}

.variant-tag.normal {
  background: #475569;
  color: #cbd5e1;
}

.variant-tag.xp100 {
  background: #b45309;
  color: #fef3c7;
}

.trip-mileage-badge {
  font-size: 10px;
  color: #10b981;
  background: rgba(16, 185, 129, 0.1);
  padding: 2px 6px;
  border-radius: 4px;
  font-weight: 600;
}

.trip-cpk-badge {
  font-size: 10px;
  color: #38bdf8;
  background: rgba(56, 189, 248, 0.1);
  padding: 2px 6px;
  border-radius: 4px;
  font-weight: 600;
}

.empty-state {
  display: block;
  text-align: center;
  padding: 30px !important;
  color: #64748b;
  width: 100%;
}

/* ---------------------------------------------------
   MEDIA QUERIES FOR LARGER TABLETS & DESKTOPS
--------------------------------------------------- */
@media (min-width: 768px) {
  .dashboard-wrapper {
    padding: 24px;
  }

  .metrics-grid {
    grid-template-columns: repeat(4, 1fr);
  }

  .kpi-card {
    padding: 20px;
  }

  .kpi-card .value {
    font-size: 26px;
  }

  .charts-section {
    display: grid;
    grid-template-columns: 2fr 1fr;
    gap: 20px;
  }

  .data-table {
    display: table;
  }

  .data-table thead {
    display: table-header-group;
  }

  .data-table tbody {
    display: table-row-group;
  }

  .data-table tr:not(.empty-state-row) {
    display: table-row;
    background: transparent;
    border: none;
  }

  .data-table td {
    display: table-cell;
    padding: 12px 10px;
    border-bottom: 1px solid #334155;
    width: auto;
    text-align: left;
  }

  .data-table td::before {
    display: none;
  }

  .data-table th {
    background: #0f172a;
    padding: 10px;
    color: #94a3b8;
    border-bottom: 2px solid #334155;
  }

  .flex-badges {
    flex-direction: column;
    align-items: flex-start;
    gap: 4px;
  }
}

@media (min-width: 1024px) {
  .dashboard-wrapper {
    padding: 30px;
  }

  .kpi-card .value {
    font-size: 34px;
  }

  .main-layout {
    display: grid;
    grid-template-columns: 400px 1fr;
    gap: 30px;
  }
}

/* Container rules establishing anchor bounds */
.city-select-wrapper {
  position: relative;
}

.hybrid-input-container {
  position: relative;
  width: 100%;
}

.hybrid-input-container input {
  padding-right: 32px !important;
}

.dropdown-arrow-indicator {
  position: absolute;
  right: 12px;
  top: 50%;
  transform: translateY(-50%);
  font-size: 10px;
  color: #64748b;
  cursor: pointer;
  pointer-events: auto;
  user-select: none;
}

/* High-contrast floating choice window */
.custom-dropdown-list {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  background: #0f172a;
  border: 1px solid #475569;
  border-radius: 6px;
  margin-top: 4px;
  padding: 0;
  list-style: none;
  max-height: 160px;
  overflow-y: auto;
  z-index: 50;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.5);
}

.custom-dropdown-list li {
  padding: 10px 12px;
  color: #fff;
  cursor: pointer;
  font-size: 14px;
  border-bottom: 1px solid #1e293b;
}

.custom-dropdown-list li:last-child {
  border-bottom: none;
}

.custom-dropdown-list li:hover {
  background: #38bdf8;
  color: #0f172a;
  font-weight: 600;
}

/* ---------------------------------------------------
   CUSTOM THEME PIN MODAL & ERROR STATES
--------------------------------------------------- */
.pin-modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(15, 23, 42, 0.85);
  backdrop-filter: blur(4px);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 9999;
}

.pin-modal-card {
  background: #0f172a;
  border: 1px solid #1e293b;
  border-radius: 12px;
  width: 90%;
  max-width: 360px;
  padding: 24px;
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.5);
  text-align: center;
}

.pin-modal-header h3 {
  color: #fff;
  margin: 0 0 6px 0;
  font-size: 18px;
}

.pin-modal-header p {
  color: #64748b;
  font-size: 13px;
  margin: 0 0 20px 0;
}

.pin-input-field {
  width: 140px;
  background: #1e293b;
  border: 2px solid #334155;
  color: #fff;
  font-size: 24px;
  text-align: center;
  letter-spacing: 8px;
  padding: 8px;
  border-radius: 8px;
  outline: none;
  transition: border-color 0.2s;
}

.pin-input-field:focus {
  border-color: #38bdf8;
}

/* Standardized Custom Modal Warning Alert Panel Layout */
.pin-error-text {
  color: #f43f5e;
  font-size: 13px;
  margin-top: 12px;
  font-weight: 600;
  background: rgba(244, 63, 94, 0.1);
  padding: 6px 12px;
  border-radius: 6px;
  display: inline-block;
}

/* Structural validation failure animation trigger hook */
.input-shake-error {
  border-color: #f43f5e !important;
  animation: modal-shake 0.3s ease-in-out;
}

@keyframes modal-shake {

  0%,
  100% {
    transform: translateX(0);
  }

  20%,
  60% {
    transform: translateX(-6px);
  }

  40%,
  80% {
    transform: translateX(6px);
  }
}

.pin-modal-actions {
  display: flex;
  gap: 12px;
  margin-top: 24px;
}

.btn-cancel,
.btn-confirm {
  flex: 1;
  padding: 10px;
  border-radius: 6px;
  font-weight: 600;
  font-size: 14px;
  cursor: pointer;
  border: none;
  transition: background 0.2s;
}

.btn-cancel {
  background: #334155;
  color: #94a3b8;
}

.btn-cancel:hover {
  background: #475569;
}

.btn-confirm {
  background: #38bdf8;
  color: #0f172a;
}

.btn-confirm:hover {
  background: #7dd3fc;
}

.kpi-sub-breakdown {
  display: flex;
  justify-content: space-between;
  margin-top: 10px;
  padding-top: 8px;
  border-top: 1px solid rgba(51, 65, 85, 0.5);
  gap: 8px;
}

.sub-item {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
}

.sub-label {
  font-size: 10px;
  color: #64748b;
  font-weight: 600;
  text-transform: uppercase;
}

.sub-val {
  font-size: 13px;
  font-weight: 700;
  margin-top: 2px;
}

.sub-val small {
  font-size: 10px;
  font-weight: 500;
  color: #64748b;
}

/* Color overrides matching line highlights */
.text-rose {
  color: #f43f5e;
}

.text-emerald {
  color: #10b981;
}

.text-cyan {
  color: #06b6d4 !important;
}

.text-amber {
  color: #f59e0b !important;
}
</style>