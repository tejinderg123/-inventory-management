<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Budget Card -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budget') }}</h3>
        </div>
        <div class="budget-body">
          <div class="budget-display">
            {{ formatMoney(budget) }}
          </div>
          <input
            type="range"
            class="budget-slider"
            v-model.number="budget"
            :min="BUDGET_MIN"
            :max="BUDGET_MAX"
            :step="BUDGET_STEP"
          />
          <div class="budget-range-labels">
            <span>{{ formatMoney(BUDGET_MIN) }}</span>
            <span>{{ formatMoney(BUDGET_MAX) }}</span>
          </div>
          <div class="budget-bar-container">
            <div
              class="budget-bar-fill"
              :style="{ width: budgetPercent + '%', background: budgetPercent > 90 ? '#ef4444' : '#2563eb' }"
            ></div>
          </div>
          <div class="budget-bar-label">
            <span :style="{ color: budgetPercent > 90 ? '#ef4444' : '#475569' }">
              {{ formatMoney(totalSelected) }} {{ t('restocking.budgetUsed') }}
            </span>
            <span class="budget-items-count">
              {{ selectedSkus.size }} {{ t('restocking.itemsSelected') }}
            </span>
          </div>
        </div>
      </div>

      <!-- Success Banner -->
      <div v-if="orderPlaced" class="success-banner">
        <div class="success-content">
          <svg width="20" height="20" viewBox="0 0 20 20" fill="currentColor">
            <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z" clip-rule="evenodd" />
          </svg>
          <div>
            <strong>{{ lastOrderNumber }}</strong> — {{ t('restocking.orderSuccess') }}
          </div>
        </div>
        <button class="dismiss-btn" @click="orderPlaced = false">{{ t('restocking.dismiss') }}</button>
      </div>

      <!-- Recommendations Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }}</h3>
          <span class="card-subtitle">Increasing demand · sorted by demand gap</span>
        </div>

        <div v-if="recommendations.length === 0" class="no-data">
          {{ t('restocking.noRecommendations') }}
        </div>
        <div v-else class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th class="col-check">{{ t('restocking.table.select') }}</th>
                <th class="col-sku">{{ t('restocking.table.sku') }}</th>
                <th class="col-name">{{ t('restocking.table.itemName') }}</th>
                <th class="col-gap">{{ t('restocking.table.demandGap') }}</th>
                <th class="col-cost">{{ t('restocking.table.unitCost') }}</th>
                <th class="col-qty">{{ t('restocking.table.qtyToOrder') }}</th>
                <th class="col-total">{{ t('restocking.table.lineTotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.sku"
                :class="{ 'row-selected': selectedSkus.has(item.sku), 'row-over-budget': !selectedSkus.has(item.sku) && wouldExceedBudget(item) }"
                @click="toggleItem(item.sku)"
                class="recommendation-row"
              >
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="selectedSkus.has(item.sku)"
                    @change.stop="toggleItem(item.sku)"
                    @click.stop
                  />
                </td>
                <td class="col-sku"><strong>{{ item.sku }}</strong></td>
                <td class="col-name">{{ item.name }}</td>
                <td class="col-gap">
                  <span class="gap-badge">+{{ item.gap }} units</span>
                </td>
                <td class="col-cost">{{ formatMoney(item.unit_cost) }}</td>
                <td class="col-qty">{{ item.gap }}</td>
                <td class="col-total"><strong>{{ formatMoney(item.line_cost) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <div v-if="orderError" class="error" style="margin: 1rem 1.25rem 0;">{{ orderError }}</div>

        <div class="place-order-section">
          <div class="order-summary" v-if="selectedSkus.size > 0">
            <span>{{ selectedSkus.size }} item(s) · Total: <strong>{{ formatMoney(totalSelected) }}</strong></span>
          </div>
          <button
            class="place-order-btn"
            :disabled="!canPlaceOrder || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Submitting...' : t('restocking.placeOrder') }}
          </button>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const allForecasts = ref([])
    const inventoryItems = ref([])

    const BUDGET_MIN = 0
    const BUDGET_MAX = 1000000
    const BUDGET_STEP = 5000

    const budget = ref(100000)
    // new Set each time — Vue 3 doesn't track Set mutations, so always reassign
    const selectedSkus = ref(new Set())
    const orderPlaced = ref(false)
    const orderError = ref(null)
    const lastOrderNumber = ref('')
    const submitting = ref(false)

    const formatMoney = (value) => {
      if (currentCurrency.value === 'JPY') {
        return '¥' + Math.round(value).toLocaleString()
      }
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 })
    }

    const recommendations = computed(() => {
      const inventoryMap = new Map(inventoryItems.value.map(i => [i.sku, i]))
      return allForecasts.value
        .filter(f => f.trend === 'increasing')
        .map(f => {
          const inv = inventoryMap.get(f.item_sku)
          if (!inv) return null
          const gap = f.forecasted_demand - f.current_demand
          return {
            id: f.id,
            sku: f.item_sku,
            name: f.item_name,
            current_demand: f.current_demand,
            forecasted_demand: f.forecasted_demand,
            gap,
            unit_cost: inv.unit_cost,
            line_cost: gap * inv.unit_cost
          }
        })
        .filter(Boolean)
        .sort((a, b) => b.gap - a.gap)
    })

    const autoSelect = () => {
      const next = new Set()
      let running = 0
      for (const item of recommendations.value) {
        if (running + item.line_cost <= budget.value) {
          next.add(item.sku)
          running += item.line_cost
        }
      }
      selectedSkus.value = next
    }

    const totalSelected = computed(() => {
      return recommendations.value
        .filter(r => selectedSkus.value.has(r.sku))
        .reduce((sum, r) => sum + r.line_cost, 0)
    })

    const budgetPercent = computed(() => {
      if (budget.value === 0) return 0
      return Math.min(100, (totalSelected.value / budget.value) * 100)
    })

    const toggleItem = (sku) => {
      const next = new Set(selectedSkus.value)
      if (next.has(sku)) {
        next.delete(sku)
      } else {
        next.add(sku)
      }
      selectedSkus.value = next
    }

    const wouldExceedBudget = (item) => {
      return totalSelected.value + item.line_cost > budget.value
    }

    const canPlaceOrder = computed(() => {
      return selectedSkus.value.size > 0 && totalSelected.value <= budget.value && !submitting.value
    })

    const placeOrder = async () => {
      orderError.value = null
      submitting.value = true
      const items = recommendations.value
        .filter(r => selectedSkus.value.has(r.sku))
        .map(r => ({ sku: r.sku, name: r.name, quantity: r.gap, unit_price: r.unit_cost }))
      try {
        const result = await api.createOrder({
          customer: 'Internal Restock',
          items,
          is_restocking: true
        })
        lastOrderNumber.value = result.order_number
        orderPlaced.value = true
        selectedSkus.value = new Set()
        autoSelect()
      } catch (err) {
        orderError.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        allForecasts.value = forecasts
        inventoryItems.value = inventory
        autoSelect()
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    watch(budget, autoSelect)

    onMounted(loadData)

    return {
      t,
      loading,
      error,
      budget,
      BUDGET_MIN,
      BUDGET_MAX,
      BUDGET_STEP,
      recommendations,
      selectedSkus,
      totalSelected,
      budgetPercent,
      formatMoney,
      toggleItem,
      wouldExceedBudget,
      canPlaceOrder,
      placeOrder,
      submitting,
      orderPlaced,
      orderError,
      lastOrderNumber
    }
  }
}
</script>

<style scoped>
.page-header {
  margin-bottom: 1.5rem;
}
.page-header h2 {
  font-size: 1.875rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  margin-bottom: 0.375rem;
}
.page-header p {
  color: #64748b;
  font-size: 0.938rem;
}

/* ── Budget Card ── */
.budget-card {
  margin-bottom: 1.25rem;
}
.budget-body {
  padding: 0.5rem 0.25rem;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}
.budget-display {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.03em;
  text-align: center;
}
.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}
.budget-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
}
.budget-bar-container {
  width: 100%;
  height: 8px;
  background: #f1f5f9;
  border-radius: 4px;
  overflow: hidden;
}
.budget-bar-fill {
  height: 100%;
  border-radius: 4px;
  transition: width 0.3s ease, background 0.3s ease;
  min-width: 2px;
}
.budget-bar-label {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
}
.budget-items-count {
  color: #64748b;
}

/* ── Success Banner ── */
.success-banner {
  background: #d1fae5;
  border: 1px solid #34d399;
  border-radius: 8px;
  padding: 0.875rem 1.25rem;
  margin-bottom: 1.25rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}
.success-content {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  color: #065f46;
  font-size: 0.875rem;
}
.dismiss-btn {
  background: none;
  border: 1px solid #34d399;
  color: #065f46;
  border-radius: 6px;
  padding: 0.375rem 0.875rem;
  font-size: 0.813rem;
  font-weight: 500;
  cursor: pointer;
  white-space: nowrap;
}
.dismiss-btn:hover {
  background: #a7f3d0;
}

/* ── Recommendations Table ── */
.card-subtitle {
  font-size: 0.813rem;
  color: #94a3b8;
}
.restocking-table {
  table-layout: fixed;
  width: 100%;
}
.col-check  { width: 48px; }
.col-sku    { width: 110px; }
.col-name   { width: auto; }
.col-gap    { width: 130px; }
.col-cost   { width: 110px; }
.col-qty    { width: 110px; }
.col-total  { width: 120px; }

.recommendation-row {
  cursor: pointer;
  transition: background 0.15s;
}
.recommendation-row:hover {
  background: #f8fafc;
}
.row-selected td {
  background: #eff6ff;
}
.row-over-budget {
  opacity: 0.45;
}

.gap-badge {
  display: inline-block;
  background: #d1fae5;
  color: #065f46;
  font-size: 0.75rem;
  font-weight: 600;
  padding: 0.2rem 0.5rem;
  border-radius: 4px;
}

/* ── Place Order ── */
.place-order-section {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 1.25rem;
  padding: 1rem 0 0.25rem;
  border-top: 1px solid #f1f5f9;
  margin-top: 0.5rem;
}
.order-summary {
  font-size: 0.875rem;
  color: #475569;
}
.place-order-btn {
  padding: 0.75rem 1.75rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s, transform 0.1s;
}
.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
  transform: translateY(-1px);
}
.place-order-btn:disabled {
  background: #cbd5e1;
  color: #94a3b8;
  cursor: not-allowed;
  transform: none;
}

.no-data {
  padding: 2.5rem;
  text-align: center;
  color: #94a3b8;
  font-size: 0.875rem;
}
</style>
