<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Recommendations</h2>
      <p>Review low-stock items and place restocking orders within your budget.</p>
    </div>

    <div v-if="loading" class="loading">Loading restocking data...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div v-if="successMessage" class="success-banner">
        {{ successMessage }}
      </div>

      <div class="stats-grid">
        <div class="stat-card danger">
          <div class="stat-label">Low Stock Items</div>
          <div class="stat-value">{{ sortedLowStockItems.length }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">Items Selected</div>
          <div class="stat-value">{{ selectedItems.length }}</div>
        </div>
        <div class="stat-card info">
          <div class="stat-label">Budget Used</div>
          <div class="stat-value">{{ formatCurrency(totalCost) }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">Budget Remaining</div>
          <div class="stat-value">{{ formatCurrency(budget - totalCost) }}</div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div class="budget-slider-container">
          <label class="budget-label" for="budget-slider">Restocking Budget</label>
          <div class="slider-row">
            <span class="slider-min">$0</span>
            <input
              id="budget-slider"
              type="range"
              class="budget-slider"
              :min="0"
              :max="100000"
              :step="1000"
              v-model.number="budget"
            />
            <span class="slider-max">$100,000</span>
          </div>
          <div class="budget-value">{{ formatCurrency(budget) }}</div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Low Stock Items ({{ sortedLowStockItems.length }})</h3>
        </div>
        <div v-if="sortedLowStockItems.length === 0" class="loading">
          No low-stock items found. All inventory levels are above reorder points.
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Current Stock</th>
                <th>Reorder Point</th>
                <th>Gap</th>
                <th>Restock Qty</th>
                <th>Forecasted Demand</th>
                <th>Trend</th>
                <th>Unit Cost</th>
                <th>Restock Cost</th>
                <th>Status</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in sortedLowStockItems"
                :key="item.sku"
                :class="{ 'row-over-budget': !isSelected(item) }"
              >
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td>{{ item.quantity_on_hand }}</td>
                <td>{{ item.reorder_point }}</td>
                <td><strong>{{ item.gap }}</strong></td>
                <td>{{ item.restock_qty }}</td>
                <td>
                  <span v-if="item.forecasted_demand !== null">{{ item.forecasted_demand }}</span>
                  <span v-else class="muted">N/A</span>
                </td>
                <td>
                  <span
                    v-if="item.trend"
                    :class="['badge', getTrendClass(item.trend)]"
                  >{{ item.trend }}</span>
                  <span v-else class="muted">N/A</span>
                </td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td><strong>{{ formatCurrency(item.restock_cost) }}</strong></td>
                <td>
                  <span v-if="isSelected(item)" class="badge success">Selected</span>
                  <span v-else class="badge danger">Over Budget</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div v-if="sortedLowStockItems.length > 0" class="card summary-bar">
        <div class="summary-details">
          <span class="summary-stat">
            <strong>Items selected:</strong> {{ selectedItems.length }} of {{ sortedLowStockItems.length }} low-stock items
          </span>
          <span class="summary-divider">|</span>
          <span class="summary-stat">
            <strong>Total cost:</strong> {{ formatCurrency(totalCost) }} of {{ formatCurrency(budget) }} budget used
          </span>
        </div>
        <button
          class="place-order-btn"
          :disabled="submitting || selectedItems.length === 0"
          @click="placeOrder"
        >
          {{ submitting ? 'Placing Order...' : 'Place Order' }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const inventoryItems = ref([])
    const demandForecasts = ref([])
    const budget = ref(25000)
    const submitting = ref(false)
    const successMessage = ref(null)
    let successTimer = null

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [inventory, forecasts] = await Promise.all([
          api.getInventory(),
          api.getDemandForecasts()
        ])
        inventoryItems.value = inventory
        demandForecasts.value = forecasts
      } catch (err) {
        error.value = 'Failed to load restocking data. Please try again.'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const sortedLowStockItems = computed(() => {
      const forecastMap = {}
      demandForecasts.value.forEach(f => {
        forecastMap[f.item_sku] = f
      })

      const lowStock = inventoryItems.value
        .filter(item => item.quantity_on_hand <= item.reorder_point)
        .map(item => {
          const gap = item.reorder_point - item.quantity_on_hand
          const restock_qty = gap
          const restock_cost = gap * item.unit_cost
          const forecast = forecastMap[item.sku] || null
          return {
            ...item,
            gap,
            restock_qty,
            restock_cost,
            forecasted_demand: forecast ? forecast.forecasted_demand : null,
            trend: forecast ? forecast.trend : null
          }
        })

      return lowStock.sort((a, b) => b.gap - a.gap)
    })

    const selectedItems = computed(() => {
      let remaining = budget.value
      const selected = []
      for (const item of sortedLowStockItems.value) {
        if (item.restock_cost <= remaining) {
          selected.push(item)
          remaining -= item.restock_cost
        }
      }
      return selected
    })

    const totalCost = computed(() => {
      return selectedItems.value.reduce((sum, item) => sum + item.restock_cost, 0)
    })

    const isSelected = (item) => {
      return selectedItems.value.some(s => s.sku === item.sku)
    }

    const getTrendClass = (trend) => {
      if (!trend) return ''
      const t = trend.toLowerCase()
      if (t === 'increasing') return 'increasing'
      if (t === 'decreasing') return 'decreasing'
      return 'stable'
    }

    const formatCurrency = (value) => {
      if (value == null) return '$0'
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 })
    }

    const placeOrder = async () => {
      if (selectedItems.value.length === 0 || submitting.value) return

      submitting.value = true
      error.value = null

      const items = selectedItems.value.map(item => ({
        sku: item.sku,
        name: item.name,
        quantity: item.restock_qty,
        unit_cost: item.unit_cost
      }))
      const totalValue = totalCost.value

      try {
        await api.submitRestockingOrder(items, totalValue)
        if (successTimer) clearTimeout(successTimer)
        successMessage.value = 'Restocking order placed successfully. Expected delivery in 14 days. View it in the Orders tab.'
        successTimer = setTimeout(() => {
          successMessage.value = null
        }, 8000)
      } catch (err) {
        error.value = 'Failed to place restocking order. Please try again.'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(() => loadData())

    return {
      loading,
      error,
      budget,
      submitting,
      successMessage,
      sortedLowStockItems,
      selectedItems,
      totalCost,
      isSelected,
      getTrendClass,
      formatCurrency,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}

.budget-slider-container {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #475569;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.slider-min,
.slider-max {
  font-size: 0.813rem;
  color: #94a3b8;
  white-space: nowrap;
}

.budget-slider {
  flex: 1;
  height: 6px;
  -webkit-appearance: none;
  appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.budget-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

.row-over-budget {
  opacity: 0.45;
}

.muted {
  color: #94a3b8;
  font-size: 0.813rem;
}

.summary-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1.5rem;
  flex-wrap: wrap;
}

.summary-details {
  display: flex;
  align-items: center;
  gap: 1rem;
  flex-wrap: wrap;
}

.summary-stat {
  font-size: 0.938rem;
  color: #334155;
}

.summary-divider {
  color: #cbd5e1;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 0.625rem 1.5rem;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
  white-space: nowrap;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}
</style>
