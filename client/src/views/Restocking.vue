<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Set your available budget to get AI-powered restocking recommendations based on demand forecasts.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget control card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>
        <div class="budget-control">
          <div class="budget-display">{{ currencySymbol }}{{ budget.toLocaleString() }}</div>
          <div class="slider-wrapper">
            <input
              type="range"
              class="budget-slider"
              :min="0"
              :max="maxBudget"
              :step="1000"
              v-model.number="budget"
            />
            <div class="slider-labels">
              <span>{{ currencySymbol }}0</span>
              <span>{{ currencySymbol }}{{ maxBudget.toLocaleString() }}</span>
            </div>
          </div>

          <!-- Utilization bar -->
          <div class="utilization-bar">
            <div
              class="utilization-fill"
              :style="{ width: utilizationPercent + '%' }"
            ></div>
          </div>

          <!-- Budget summary -->
          <div class="budget-summary">
            {{ recommendedItems.length }} item(s) recommended
            &middot; Total: {{ currencySymbol }}{{ totalCost.toLocaleString() }}
            &middot; Remaining: {{ currencySymbol }}{{ (budget - totalCost).toLocaleString() }}
          </div>
        </div>
      </div>

      <!-- Success banner -->
      <div v-if="orderPlaced" class="success-banner">
        Order placed successfully. Check the Orders tab to track delivery.
      </div>

      <!-- Recommendations table card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Restocking ({{ recommendedItems.length }})</h3>
          <button
            class="place-order-btn"
            :disabled="recommendedItems.length === 0 || orderPlaced"
            @click="placeOrder"
          >
            Place Order
          </button>
        </div>

        <div v-if="orderError" class="error" style="margin: 0 1.5rem 1rem;">{{ orderError }}</div>

        <div v-if="recommendedItems.length > 0" class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Trend</th>
                <th>Restock Qty</th>
                <th>Unit Cost</th>
                <th>Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.item_sku">
                <td>{{ item.item_sku }}</td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', trendBadgeClass(item.trend)]">{{ item.trend }}</span>
                </td>
                <td>{{ item.forecasted_demand.toLocaleString() }}</td>
                <td>{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td><strong>{{ currencySymbol }}{{ item.line_total.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <div v-if="recommendedItems.length === 0 && budget > 0" class="empty-state">
          No items fit within the current budget.
        </div>
        <div v-if="budget === 0" class="empty-state">
          Move the slider to set a budget and see recommendations.
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const loading = ref(true)
    const error = ref(null)
    const orderError = ref(null)
    const orderPlaced = ref(false)

    const forecasts = ref([])
    // Maps sku -> unit_cost for line cost calculations
    const skuCostMap = ref({})

    const budget = ref(0)

    // Sort forecasts: increasing > stable > decreasing, then by highest forecasted_demand
    const sortedForecasts = computed(() => {
      const trendOrder = { increasing: 0, stable: 1, decreasing: 2 }
      return [...forecasts.value].sort((a, b) => {
        const trendDiff = (trendOrder[a.trend] ?? 99) - (trendOrder[b.trend] ?? 99)
        if (trendDiff !== 0) return trendDiff
        return b.forecasted_demand - a.forecasted_demand
      })
    })

    // Max budget is the full cost to restock every forecast item
    const maxBudget = computed(() => {
      return sortedForecasts.value.reduce((sum, item) => {
        return sum + item.forecasted_demand * (skuCostMap.value[item.item_sku] ?? 0)
      }, 0)
    })

    // Greedy selection: walk sorted forecasts, include items that fit within budget
    const recommendedItems = computed(() => {
      if (budget.value === 0) return []
      const result = []
      let runningTotal = 0
      for (const item of sortedForecasts.value) {
        const unitCost = skuCostMap.value[item.item_sku]
        // Skip items with no price match
        if (unitCost == null) continue
        const lineCost = item.forecasted_demand * unitCost
        if (runningTotal + lineCost <= budget.value) {
          result.push({
            ...item,
            unit_cost: unitCost,
            line_total: lineCost
          })
          runningTotal += lineCost
        }
      }
      return result
    })

    const totalCost = computed(() => {
      return recommendedItems.value.reduce((sum, item) => sum + item.line_total, 0)
    })

    const utilizationPercent = computed(() => {
      if (budget.value === 0) return 0
      return Math.min(100, (totalCost.value / budget.value) * 100)
    })

    const trendBadgeClass = (trend) => {
      const map = { increasing: 'increasing', stable: 'stable', decreasing: 'decreasing' }
      return map[trend] || 'info'
    }

    const placeOrder = async () => {
      orderError.value = null
      try {
        await api.createRestockingOrder({
          items: recommendedItems.value.map(item => ({
            sku: item.item_sku,
            name: item.item_name,
            quantity: item.forecasted_demand,
            unit_price: item.unit_cost
          })),
          total_value: totalCost.value,
          budget: budget.value
        })
        orderPlaced.value = true
      } catch (err) {
        orderError.value = 'Failed to place order: ' + (err.message || 'Unknown error')
        console.error(err)
      }
    }

    onMounted(async () => {
      loading.value = true
      error.value = null
      try {
        const [forecastData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])
        forecasts.value = forecastData
        // Build sku -> unit_cost lookup from inventory
        const costMap = {}
        for (const item of inventoryData) {
          costMap[item.sku] = item.unit_cost
        }
        skuCostMap.value = costMap
      } catch (err) {
        error.value = 'Failed to load restocking data'
        console.error(err)
      } finally {
        loading.value = false
      }
    })

    return {
      loading,
      error,
      orderError,
      orderPlaced,
      budget,
      maxBudget,
      sortedForecasts,
      recommendedItems,
      totalCost,
      utilizationPercent,
      currencySymbol,
      trendBadgeClass,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-control {
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.budget-display {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
  line-height: 1;
}

.slider-wrapper {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
  height: 6px;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #64748b;
}

.utilization-bar {
  height: 8px;
  border-radius: 4px;
  background: #e2e8f0;
  overflow: hidden;
}

.utilization-fill {
  height: 100%;
  border-radius: 4px;
  background: #2563eb;
  transition: width 0.25s ease;
}

.budget-summary {
  font-size: 0.875rem;
  color: #64748b;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-weight: 600;
  font-size: 0.875rem;
  cursor: pointer;
  white-space: nowrap;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 1rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}

.empty-state {
  text-align: center;
  color: #64748b;
  padding: 2rem;
  font-size: 0.938rem;
}
</style>
