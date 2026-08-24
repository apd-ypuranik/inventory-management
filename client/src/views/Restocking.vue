<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div class="card budget-card">
      <label class="budget-label" for="budget-slider">
        {{ t('restocking.budgetLabel') }}: <strong>{{ currencySymbol }}{{ budget.toLocaleString() }}</strong>
      </label>
      <input
        id="budget-slider"
        type="range"
        min="0"
        max="100000"
        step="1000"
        v-model.number="budget"
        @change="loadRecommendations"
        class="budget-slider"
      />
    </div>

    <div v-if="confirmation" class="confirmation-banner">
      <div class="confirmation-title">{{ t('restocking.confirmation.title') }}</div>
      <div class="confirmation-message">
        {{ t('restocking.confirmation.message', { orderNumber: confirmation.order_number }) }}
      </div>
      <div class="confirmation-meta">
        <span>{{ t('restocking.confirmation.leadTime', { days: confirmation.lead_time_days }) }}</span>
        <span>{{ t('restocking.confirmation.expectedDelivery', { date: formatDate(confirmation.expected_delivery) }) }}</span>
      </div>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-label">{{ t('restocking.summary.totalCost') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ totalCost.toLocaleString() }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">{{ t('restocking.summary.remainingBudget') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ remainingBudget.toLocaleString() }}</div>
        </div>
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.summary.itemsCount') }}</div>
          <div class="stat-value">{{ recommendations.length }}</div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.summary.itemsCount') }} ({{ recommendations.length }})</h3>
          <button
            class="place-order-btn"
            :disabled="recommendations.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? t('restocking.placingOrder') : t('restocking.placeOrder') }}
          </button>
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          {{ t('restocking.noRecommendations') }}
        </div>
        <div v-else class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th class="col-sku">{{ t('restocking.table.sku') }}</th>
                <th class="col-name">{{ t('restocking.table.itemName') }}</th>
                <th class="col-trend">{{ t('restocking.table.trend') }}</th>
                <th class="col-num">{{ t('restocking.table.currentStock') }}</th>
                <th class="col-num">{{ t('restocking.table.forecastedDemand') }}</th>
                <th class="col-num">{{ t('restocking.table.urgency') }}</th>
                <th class="col-num">{{ t('restocking.table.recommendedQty') }}</th>
                <th class="col-value">{{ t('restocking.table.unitCost') }}</th>
                <th class="col-value">{{ t('restocking.table.lineTotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.item_sku">
                <td class="col-sku"><strong>{{ item.item_sku }}</strong></td>
                <td class="col-name">{{ translateProductName(item.item_name) }}</td>
                <td class="col-trend">
                  <span :class="['badge', item.trend]">
                    {{ t(`trends.${item.trend}`) }}
                  </span>
                </td>
                <td class="col-num">{{ item.quantity_on_hand }}</td>
                <td class="col-num">{{ item.forecasted_demand }}</td>
                <td class="col-num">
                  <span :style="{ color: getUrgencyColor(item) }">
                    {{ item.urgency_score.toFixed(1) }}
                  </span>
                </td>
                <td class="col-num"><strong>{{ item.recommended_quantity }}</strong></td>
                <td class="col-value">{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td class="col-value"><strong>{{ currencySymbol }}{{ item.line_total.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
          </table>
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
    const { t, currentCurrency, currentLocale, translateProductName } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const budget = ref(10000)
    const recommendations = ref([])
    const totalCost = ref(0)
    const remainingBudget = ref(0)
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const confirmation = ref(null)

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        const data = await api.getRestockRecommendations(budget.value)
        recommendations.value = data.items
        totalCost.value = data.total_cost
        remainingBudget.value = data.remaining_budget
      } catch (err) {
        error.value = 'Failed to load restock recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      try {
        submitting.value = true
        error.value = null
        confirmation.value = null
        const payload = {
          budget: budget.value,
          items: recommendations.value.map(r => ({
            item_sku: r.item_sku,
            item_name: r.item_name,
            quantity: r.recommended_quantity,
            unit_cost: r.unit_cost
          }))
        }
        const order = await api.submitRestockOrder(payload)
        confirmation.value = order
        await loadRecommendations()
      } catch (err) {
        error.value = 'Failed to place restock order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    const getUrgencyColor = (item) => {
      if (item.urgency_score >= 500) return '#dc2626'
      if (item.urgency_score >= 100) return '#ea580c'
      return '#2563eb'
    }

    const formatDate = (dateString) => {
      const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      return date.toLocaleDateString(locale, {
        year: 'numeric',
        month: 'short',
        day: 'numeric'
      })
    }

    onMounted(loadRecommendations)

    return {
      t,
      budget,
      recommendations,
      totalCost,
      remainingBudget,
      loading,
      error,
      submitting,
      confirmation,
      currencySymbol,
      translateProductName,
      loadRecommendations,
      placeOrder,
      getUrgencyColor,
      formatDate
    }
  }
}
</script>

<style scoped>
.budget-card {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.budget-label {
  font-size: 0.938rem;
  color: #334155;
  font-weight: 500;
}

.budget-label strong {
  color: #0f172a;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
}

.confirmation-banner {
  background: #ecfdf5;
  border: 1px solid #a7f3d0;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.25rem;
}

.confirmation-title {
  font-weight: 700;
  color: #065f46;
  margin-bottom: 0.25rem;
}

.confirmation-message {
  color: #065f46;
  font-size: 0.938rem;
  margin-bottom: 0.5rem;
}

.confirmation-meta {
  display: flex;
  gap: 1.5rem;
  font-size: 0.813rem;
  color: #047857;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 0.5rem 1.25rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}

.empty-state {
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.restocking-table {
  table-layout: fixed;
  width: 100%;
}

.col-sku {
  width: 110px;
}

.col-name {
  width: 200px;
}

.col-trend {
  width: 110px;
}

.col-num {
  width: 110px;
}

.col-value {
  width: 120px;
}
</style>
