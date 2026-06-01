<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>

    <!-- Success state -->
    <div v-else-if="submittedOrder" class="card success-panel">
      <div class="success-icon">✓</div>
      <h3>{{ t('restocking.successTitle') }}</h3>
      <p class="success-order-num">
        <strong>{{ submittedOrder.order_number }}</strong> {{ t('restocking.successBody') }}
      </p>
      <div class="success-details">
        <div class="success-detail">
          <span class="detail-label">{{ t('restocking.expectedDelivery') }}</span>
          <span class="detail-value">{{ formatDate(submittedOrder.expected_delivery) }}</span>
        </div>
        <div class="success-detail">
          <span class="detail-label">{{ t('restocking.totalValue') }}</span>
          <span class="detail-value"
            >{{ currencySymbol }}{{ submittedOrder.total_value.toLocaleString() }}</span
          >
        </div>
      </div>
      <div class="success-actions">
        <router-link to="/orders" class="btn-primary">{{ t('restocking.viewOrders') }}</router-link>
        <button class="btn-secondary" @click="resetForm">{{ t('restocking.placeAnother') }}</button>
      </div>
    </div>

    <!-- Active state -->
    <div v-else>
      <!-- Budget control -->
      <div class="card budget-card">
        <div class="budget-header">
          <span class="budget-label">{{ t('restocking.budgetLabel') }}</span>
          <span class="budget-amount">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
        </div>
        <input
          type="range"
          class="budget-slider"
          :min="0"
          :max="maxBudget"
          :step="100"
          v-model.number="budget"
        />
        <div class="slider-labels">
          <span>{{ currencySymbol }}0</span>
          <span>{{ currencySymbol }}{{ maxBudget.toLocaleString() }}</span>
        </div>

        <!-- Budget meter -->
        <div class="budget-meter">
          <div class="budget-meter-track">
            <div
              class="budget-meter-fill"
              :class="meterColorClass"
              :style="{ width: budgetUtilizationPercent + '%' }"
            ></div>
          </div>
          <div class="budget-meter-label">
            <span>
              {{ currencySymbol }}{{ totalSelectedCost.toLocaleString() }}
              <span class="budget-of"> / {{ currencySymbol }}{{ budget.toLocaleString() }}</span>
            </span>
            <span class="items-count" v-if="selectedItems.length > 0">
              {{ selectedItems.length }} item{{ selectedItems.length !== 1 ? 's' : '' }} selected
            </span>
          </div>
        </div>
      </div>

      <!-- Recommendations table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            {{ t('restocking.allItems', { count: allRecommendations.length }) }}
          </h3>
        </div>

        <div v-if="selectedItems.length === 0 && budget > 0" class="no-items-msg">
          {{ t('restocking.noItemsSelected') }}
        </div>
        <div v-else-if="budget === 0" class="no-items-msg">
          {{ t('restocking.noItemsSelected') }}
        </div>

        <div class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th class="col-name">{{ t('restocking.table.itemName') }}</th>
                <th class="col-sku">{{ t('restocking.table.sku') }}</th>
                <th class="col-trend">{{ t('restocking.table.trend') }}</th>
                <th class="col-qty">{{ t('restocking.table.recommendedQty') }}</th>
                <th class="col-cost">{{ t('restocking.table.unitCost') }}</th>
                <th class="col-subtotal">{{ t('restocking.table.subtotal') }}</th>
                <th class="col-lead">{{ t('restocking.table.leadTime') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in allRecommendations"
                :key="item.id"
                :class="{ 'row-dimmed': !isSelected(item) }"
              >
                <td class="col-name">{{ item.item_name }}</td>
                <td class="col-sku">
                  <code>{{ item.item_sku }}</code>
                </td>
                <td class="col-trend">
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-qty">{{ item.recommended_quantity.toLocaleString() }}</td>
                <td class="col-cost">{{ currencySymbol }}{{ item.unit_cost.toFixed(2) }}</td>
                <td class="col-subtotal">
                  <strong>{{ currencySymbol }}{{ item.estimated_cost.toLocaleString() }}</strong>
                  <span v-if="!isSelected(item) && budget > 0" class="over-budget-tag">
                    {{ t('restocking.overBudget') }}
                  </span>
                </td>
                <td class="col-lead">{{ item.lead_time_days }} {{ t('restocking.table.days') }}</td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="table-footer">
          <button
            class="btn-primary place-order-btn"
            :disabled="selectedItems.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? t('restocking.placing') : t('restocking.placeOrder') }}
            <span v-if="!submitting && selectedItems.length > 0">
              ({{ selectedItems.length }} item{{ selectedItems.length !== 1 ? 's' : '' }} ·
              {{ currencySymbol }}{{ totalSelectedCost.toLocaleString() }})
            </span>
          </button>
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
      const { t, currentCurrency, currentLocale } = useI18n()

      const currencySymbol = computed(() => (currentCurrency.value === 'JPY' ? '¥' : '$'))

      const loading = ref(true)
      const error = ref(null)
      const allRecommendations = ref([])
      const budget = ref(0)
      const submitting = ref(false)
      const submittedOrder = ref(null)

      const maxBudget = computed(() =>
        Math.round(allRecommendations.value.reduce((s, i) => s + i.estimated_cost, 0))
      )

      const selectedItems = computed(() => {
        let remaining = budget.value
        return allRecommendations.value.filter((item) => {
          if (item.estimated_cost <= remaining) {
            remaining -= item.estimated_cost
            return true
          }
          return false
        })
      })

      const totalSelectedCost = computed(() =>
        Math.round(selectedItems.value.reduce((s, i) => s + i.estimated_cost, 0))
      )

      const budgetUtilizationPercent = computed(() =>
        budget.value > 0 ? Math.min((totalSelectedCost.value / budget.value) * 100, 100) : 0
      )

      const meterColorClass = computed(() => {
        const pct = budgetUtilizationPercent.value
        if (pct >= 90) return 'fill-red'
        if (pct >= 70) return 'fill-orange'
        return 'fill-green'
      })

      const isSelected = (item) => selectedItems.value.some((s) => s.id === item.id)

      const loadData = async () => {
        try {
          loading.value = true
          error.value = null
          allRecommendations.value = await api.getRestockingRecommendations()
          // Default to 50% of max, rounded to nearest $100
          const total = allRecommendations.value.reduce((s, i) => s + i.estimated_cost, 0)
          budget.value = Math.round((total * 0.5) / 100) * 100
        } catch (err) {
          error.value = 'Failed to load restocking recommendations: ' + err.message
        } finally {
          loading.value = false
        }
      }

      const placeOrder = async () => {
        if (selectedItems.value.length === 0 || submitting.value) return
        submitting.value = true
        try {
          submittedOrder.value = await api.submitRestockingOrder({
            items: selectedItems.value.map((i) => ({
              sku: i.item_sku,
              name: i.item_name,
              quantity: i.recommended_quantity,
              unit_price: i.unit_cost,
              lead_time_days: i.lead_time_days,
            })),
            total_value: totalSelectedCost.value,
          })
        } catch (err) {
          error.value = 'Failed to submit restocking order: ' + err.message
        } finally {
          submitting.value = false
        }
      }

      const resetForm = () => {
        submittedOrder.value = null
        loadData()
      }

      const formatDate = (dateString) => {
        const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
        return new Date(dateString).toLocaleDateString(locale, {
          year: 'numeric',
          month: 'short',
          day: 'numeric',
        })
      }

      onMounted(loadData)

      return {
        t,
        loading,
        error,
        allRecommendations,
        budget,
        maxBudget,
        selectedItems,
        totalSelectedCost,
        budgetUtilizationPercent,
        meterColorClass,
        isSelected,
        submitting,
        submittedOrder,
        currencySymbol,
        placeOrder,
        resetForm,
        formatDate,
      }
    },
  }
</script>

<style scoped>
  /* Budget card */
  .budget-card {
    margin-bottom: 1.25rem;
  }

  .budget-header {
    display: flex;
    justify-content: space-between;
    align-items: baseline;
    margin-bottom: 0.75rem;
  }

  .budget-label {
    font-size: 0.875rem;
    font-weight: 600;
    color: #475569;
    text-transform: uppercase;
    letter-spacing: 0.04em;
    font-size: 0.75rem;
  }

  .budget-amount {
    font-size: 1.75rem;
    font-weight: 700;
    color: #0f172a;
    letter-spacing: -0.02em;
  }

  .budget-slider {
    width: 100%;
    height: 6px;
    accent-color: #2563eb;
    cursor: pointer;
    margin-bottom: 0.375rem;
  }

  .slider-labels {
    display: flex;
    justify-content: space-between;
    font-size: 0.75rem;
    color: #94a3b8;
    margin-bottom: 1rem;
  }

  /* Budget meter */
  .budget-meter {
    margin-top: 0.75rem;
  }

  .budget-meter-track {
    background: #e2e8f0;
    border-radius: 4px;
    height: 8px;
    overflow: hidden;
    margin-bottom: 0.4rem;
  }

  .budget-meter-fill {
    height: 100%;
    border-radius: 4px;
    transition:
      width 0.25s ease,
      background-color 0.25s ease;
  }

  .fill-green {
    background: #10b981;
  }
  .fill-orange {
    background: #f59e0b;
  }
  .fill-red {
    background: #ef4444;
  }

  .budget-meter-label {
    display: flex;
    justify-content: space-between;
    font-size: 0.813rem;
    color: #475569;
  }

  .budget-of {
    color: #94a3b8;
  }

  .items-count {
    font-weight: 600;
    color: #2563eb;
  }

  /* Table */
  .restocking-table {
    table-layout: fixed;
    width: 100%;
  }

  .col-name {
    width: 220px;
  }
  .col-sku {
    width: 100px;
  }
  .col-trend {
    width: 110px;
  }
  .col-qty {
    width: 90px;
  }
  .col-cost {
    width: 90px;
  }
  .col-subtotal {
    width: 160px;
  }
  .col-lead {
    width: 90px;
  }

  .row-dimmed {
    opacity: 0.35;
  }

  code {
    font-family: 'Cascadia Code', Consolas, monospace;
    font-size: 0.8rem;
    background: #f1f5f9;
    padding: 0.1rem 0.35rem;
    border-radius: 3px;
    color: #334155;
  }

  .over-budget-tag {
    display: block;
    font-size: 0.65rem;
    color: #ef4444;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.04em;
    margin-top: 0.1rem;
  }

  .no-items-msg {
    text-align: center;
    padding: 1.5rem;
    color: #94a3b8;
    font-size: 0.875rem;
  }

  /* Footer & buttons */
  .table-footer {
    padding-top: 1rem;
    border-top: 1px solid #e2e8f0;
    margin-top: 0.5rem;
    display: flex;
    justify-content: flex-end;
  }

  .btn-primary {
    display: inline-block;
    background: #2563eb;
    color: white;
    border: none;
    border-radius: 8px;
    padding: 0.625rem 1.5rem;
    font-size: 0.9rem;
    font-weight: 600;
    cursor: pointer;
    transition: background 0.15s ease;
    text-decoration: none;
  }

  .btn-primary:hover:not(:disabled) {
    background: #1d4ed8;
  }

  .btn-primary:disabled {
    background: #93c5fd;
    cursor: not-allowed;
  }

  .place-order-btn {
    padding: 0.75rem 2rem;
    font-size: 0.938rem;
  }

  .btn-secondary {
    display: inline-block;
    background: white;
    color: #0f172a;
    border: 1px solid #e2e8f0;
    border-radius: 8px;
    padding: 0.625rem 1.5rem;
    font-size: 0.9rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.15s ease;
    text-decoration: none;
  }

  .btn-secondary:hover {
    background: #f8fafc;
    border-color: #cbd5e1;
  }

  /* Success panel */
  .success-panel {
    text-align: center;
    padding: 3rem 2rem;
  }

  .success-icon {
    width: 60px;
    height: 60px;
    background: #d1fae5;
    color: #059669;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 1.75rem;
    font-weight: 700;
    margin: 0 auto 1.25rem;
  }

  .success-panel h3 {
    font-size: 1.375rem;
    font-weight: 700;
    color: #0f172a;
    margin-bottom: 0.5rem;
  }

  .success-order-num {
    color: #64748b;
    font-size: 0.938rem;
    margin-bottom: 1.5rem;
  }

  .success-details {
    display: flex;
    justify-content: center;
    gap: 3rem;
    margin-bottom: 2rem;
  }

  .success-detail {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
  }

  .detail-label {
    font-size: 0.75rem;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: #94a3b8;
  }

  .detail-value {
    font-size: 1.125rem;
    font-weight: 700;
    color: #0f172a;
  }

  .success-actions {
    display: flex;
    gap: 1rem;
    justify-content: center;
  }
</style>
