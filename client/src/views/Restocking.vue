<template>
  <div class="view-container">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <!-- Budget Card -->
    <div class="card budget-card">
      <div class="budget-header">
        <span class="budget-label">{{ t('restocking.budget') }}</span>
        <span class="budget-value">${{ budget.toLocaleString() }}</span>
      </div>
      <input
        type="range"
        min="0"
        max="500000"
        step="10000"
        v-model.number="budget"
        class="budget-slider"
      />
      <div class="budget-marks">
        <span>$0</span>
        <span>$125K</span>
        <span>$250K</span>
        <span>$375K</span>
        <span>$500K</span>
      </div>
    </div>

    <!-- Loading / Error states -->
    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>

    <!-- Recommendations card (always visible once data loaded) -->
    <div v-else class="card">
      <div class="card-header">
        <h3 class="card-title">{{ t('restocking.recommendations') }}</h3>
        <div class="summary-line" v-if="recommendedItems.length > 0">
          {{ recommendedItems.length }} {{ t('restocking.itemsSelected') }}
          &middot; {{ t('restocking.total') }}: ${{ totalCost.toLocaleString() }}
          &middot; {{ t('restocking.remaining') }}: ${{ remaining.toLocaleString() }}
        </div>
      </div>

      <!-- No-budget state -->
      <div v-if="budget === 0" class="empty-state">{{ t('restocking.noBudget') }}</div>

      <!-- All items excluded state -->
      <div
        v-else-if="recommendedItems.length === 0 && sortedItems.length > 0"
        class="empty-state"
      >{{ t('restocking.noItems') }}</div>

      <div v-else class="table-container">
        <table class="restock-table">
          <thead>
            <tr>
              <th>{{ t('restocking.table.itemName') }}</th>
              <th>{{ t('restocking.table.sku') }}</th>
              <th>{{ t('restocking.table.warehouse') }}</th>
              <th>{{ t('restocking.table.leadTime') }}</th>
              <th>{{ t('restocking.table.quantity') }}</th>
              <th>{{ t('restocking.table.unitCost') }}</th>
              <th>{{ t('restocking.table.totalCost') }}</th>
            </tr>
          </thead>
          <tbody>
            <!-- Recommended items (within budget) -->
            <tr
              v-for="item in recommendedItems"
              :key="item.item_sku"
              class="row-included"
            >
              <td class="item-name">{{ item.item_name }}</td>
              <td class="sku">{{ item.item_sku }}</td>
              <td>{{ item.warehouse }}</td>
              <td>{{ item.lead_time_days }} {{ t('restocking.days') }}</td>
              <td>{{ item.forecasted_demand.toLocaleString() }}</td>
              <td>${{ item.unit_cost.toLocaleString() }}</td>
              <td class="cost-cell">${{ item.item_total_cost.toLocaleString() }}</td>
            </tr>
            <!-- Excluded items (over budget) — grayed out -->
            <tr
              v-for="item in excludedItems"
              :key="item.item_sku"
              class="row-excluded"
            >
              <td class="item-name">{{ item.item_name }}</td>
              <td class="sku">{{ item.item_sku }}</td>
              <td>{{ item.warehouse }}</td>
              <td>{{ item.lead_time_days }} {{ t('restocking.days') }}</td>
              <td>{{ item.forecasted_demand.toLocaleString() }}</td>
              <td>${{ item.unit_cost.toLocaleString() }}</td>
              <td class="cost-cell">${{ item.item_total_cost.toLocaleString() }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Place Order button + success message -->
    <div class="action-bar">
      <div v-if="orderSuccess" class="success-message">{{ t('restocking.orderPlaced') }}</div>
      <button
        class="place-order-btn"
        :disabled="recommendedItems.length === 0 || placing"
        @click="placeOrder"
      >
        {{ placing ? t('common.loading') : t('restocking.placeOrder') }}
      </button>
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
    const { t } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const forecasts = ref([])
    const inventoryItems = ref([])
    const budget = ref(100000)
    const placing = ref(false)
    const orderSuccess = ref(false)

    // Enriched items: join forecast + inventory on SKU, derive computed fields
    const enrichedItems = computed(() => {
      const trendPriority = { increasing: 1, stable: 2, decreasing: 3 }
      return forecasts.value
        .map(f => {
          const inv = inventoryItems.value.find(i => i.sku === f.item_sku)
          if (!inv) return null
          const unit_cost = inv.unit_cost
          const warehouse = inv.warehouse
          const demand_gap = f.forecasted_demand - f.current_demand
          const item_total_cost = unit_cost * f.forecasted_demand
          const lead_time_days = warehouse === 'Tokyo' ? 7 : 14
          const trend_priority = trendPriority[f.trend] ?? 2
          return {
            ...f,
            unit_cost,
            warehouse,
            demand_gap,
            item_total_cost,
            lead_time_days,
            trend_priority
          }
        })
        .filter(Boolean)
    })

    // Sort: demand_gap DESC, trend_priority ASC, unit_cost ASC
    const sortedItems = computed(() =>
      [...enrichedItems.value].sort((a, b) => {
        if (b.demand_gap !== a.demand_gap) return b.demand_gap - a.demand_gap
        if (a.trend_priority !== b.trend_priority) return a.trend_priority - b.trend_priority
        return a.unit_cost - b.unit_cost
      })
    )

    // Greedy selection: include items while running total stays within budget
    const recommendedItems = computed(() => {
      let running = 0
      return sortedItems.value.filter(item => {
        if (running + item.item_total_cost <= budget.value) {
          running += item.item_total_cost
          return true
        }
        return false
      })
    })

    const excludedItems = computed(() => {
      const included = new Set(recommendedItems.value.map(i => i.item_sku))
      return sortedItems.value.filter(i => !included.has(i.item_sku))
    })

    const totalCost = computed(() =>
      Math.round(recommendedItems.value.reduce((sum, i) => sum + i.item_total_cost, 0))
    )

    const remaining = computed(() => budget.value - totalCost.value)

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const [forecastData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        forecasts.value = forecastData
        inventoryItems.value = inventoryData
      } catch (err) {
        error.value = 'Failed to load data'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0) return
      placing.value = true
      orderSuccess.value = false
      try {
        const items = recommendedItems.value.map(i => ({
          sku: i.item_sku,
          name: i.item_name,
          quantity: i.forecasted_demand,
          unit_price: i.unit_cost,
          lead_time_days: i.lead_time_days
        }))
        await api.createRestockingOrder({
          customer: 'Internal Restock',
          items,
          warehouse: null,
          category: null
        })
        orderSuccess.value = true
      } catch (err) {
        console.error('Failed to place order', err)
      } finally {
        placing.value = false
      }
    }

    onMounted(loadData)

    return {
      t,
      loading,
      error,
      budget,
      placing,
      orderSuccess,
      sortedItems,
      recommendedItems,
      excludedItems,
      totalCost,
      remaining,
      placeOrder
    }
  }
}
</script>

<style scoped>
.view-container {
  padding: 0;
}

/* Budget card */
.budget-card {
  padding: 1.5rem 1.75rem 1.25rem;
  margin-bottom: 1.25rem;
}

.budget-header {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  margin-bottom: 1.25rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-value {
  font-size: 1.75rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

/* Range slider */
.budget-slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
  accent-color: #2563eb;
  margin-bottom: 0.625rem;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #ffffff;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  transition: box-shadow 0.15s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  box-shadow: 0 2px 8px rgba(37, 99, 235, 0.5);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #ffffff;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.budget-marks {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
  padding: 0 2px;
}

/* Recommendations table */
.restock-table {
  width: 100%;
  border-collapse: collapse;
}

.restock-table thead {
  background: #f8fafc;
  border-top: 1px solid #e2e8f0;
  border-bottom: 1px solid #e2e8f0;
}

.restock-table th {
  text-align: left;
  padding: 10px 16px;
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  border-bottom: 1px solid #e2e8f0;
}

.restock-table td {
  padding: 12px 16px;
  border-bottom: 1px solid #f1f5f9;
  font-size: 0.875rem;
  color: #334155;
}

.restock-table tbody tr {
  transition: background-color 0.15s ease;
}

/* Included rows */
.row-included:hover {
  background: #f8fafc;
}

.row-included .item-name {
  font-weight: 600;
  color: #0f172a;
}

/* Excluded rows — grayed out */
.row-excluded {
  opacity: 0.35;
}

.row-excluded td {
  color: #94a3b8;
}

/* Cell modifiers */
.cost-cell {
  font-weight: 600;
}

.sku {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
  font-size: 0.75rem;
  color: #64748b;
}

/* Summary line in card header */
.summary-line {
  font-size: 0.8125rem;
  color: #64748b;
}

/* Empty state */
.empty-state {
  padding: 40px;
  text-align: center;
  color: #94a3b8;
  font-size: 0.9375rem;
}

/* Action bar */
.action-bar {
  display: flex;
  justify-content: flex-end;
  align-items: center;
  gap: 16px;
  margin-top: 24px;
}

.success-message {
  color: #16a34a;
  font-size: 0.875rem;
  font-weight: 500;
}

.place-order-btn {
  background: #2563eb;
  color: #ffffff;
  padding: 10px 28px;
  border: none;
  border-radius: 6px;
  font-size: 0.9375rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease, opacity 0.15s ease;
  font-family: inherit;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}
</style>
