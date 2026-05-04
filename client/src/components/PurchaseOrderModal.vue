<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">
              {{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}
            </h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <!-- Item context header shown in both modes -->
            <div class="item-context">
              <div class="item-context-info">
                <h4 class="item-name">{{ translateProductName(backlogItem.item_name) }}</h4>
                <div class="item-sku">SKU: {{ backlogItem.item_sku }}</div>
              </div>
              <span class="priority-badge" :class="backlogItem.priority">
                {{ backlogItem.priority }} Priority
              </span>
            </div>

            <!-- CREATE MODE: form -->
            <template v-if="mode === 'create'">
              <div class="shortage-info">
                <div class="shortage-stat">
                  <span class="shortage-label">Quantity Needed</span>
                  <span class="shortage-value">{{ backlogItem.quantity_needed }}</span>
                </div>
                <div class="shortage-stat">
                  <span class="shortage-label">Available</span>
                  <span class="shortage-value">{{ backlogItem.quantity_available }}</span>
                </div>
                <div class="shortage-stat danger">
                  <span class="shortage-label">Shortage</span>
                  <span class="shortage-value">{{ shortage }}</span>
                </div>
              </div>

              <form class="po-form" @submit.prevent="submitForm">
                <div v-if="submitError" class="form-error">
                  {{ submitError }}
                </div>

                <div class="form-group">
                  <label class="form-label" for="supplier-name">Supplier Name <span class="required">*</span></label>
                  <input
                    id="supplier-name"
                    v-model="form.supplierName"
                    type="text"
                    class="form-input"
                    placeholder="Enter supplier name"
                    required
                    :disabled="submitting"
                  />
                </div>

                <div class="form-row">
                  <div class="form-group">
                    <label class="form-label" for="quantity">Quantity <span class="required">*</span></label>
                    <input
                      id="quantity"
                      v-model.number="form.quantity"
                      type="number"
                      class="form-input"
                      min="1"
                      required
                      :disabled="submitting"
                    />
                  </div>

                  <div class="form-group">
                    <label class="form-label" for="unit-cost">Unit Cost (USD) <span class="required">*</span></label>
                    <input
                      id="unit-cost"
                      v-model.number="form.unitCost"
                      type="number"
                      class="form-input"
                      min="0"
                      step="0.01"
                      placeholder="0.00"
                      required
                      :disabled="submitting"
                    />
                  </div>
                </div>

                <div class="form-group">
                  <label class="form-label" for="delivery-date">Expected Delivery Date <span class="required">*</span></label>
                  <input
                    id="delivery-date"
                    v-model="form.expectedDeliveryDate"
                    type="date"
                    class="form-input"
                    required
                    :disabled="submitting"
                  />
                </div>

                <div class="form-group">
                  <label class="form-label" for="notes">Notes</label>
                  <textarea
                    id="notes"
                    v-model="form.notes"
                    class="form-textarea"
                    placeholder="Optional notes for this purchase order"
                    rows="3"
                    :disabled="submitting"
                  ></textarea>
                </div>

                <!-- Total cost preview when both quantity and unit cost are filled -->
                <div v-if="totalCost > 0" class="cost-preview">
                  <span class="cost-label">Estimated Total Cost</span>
                  <span class="cost-value">{{ formatCurrency(totalCost) }}</span>
                </div>
              </form>
            </template>

            <!-- VIEW MODE: read-only PO details -->
            <template v-else>
              <template v-if="backlogItem.purchase_order">
                <div class="po-detail-grid">
                  <div class="po-detail-item">
                    <div class="info-label">PO Number</div>
                    <div class="info-value po-id">{{ backlogItem.purchase_order.id || backlogItem.purchase_order_id }}</div>
                  </div>

                  <div class="po-detail-item">
                    <div class="info-label">Supplier</div>
                    <div class="info-value">{{ backlogItem.purchase_order.supplier_name }}</div>
                  </div>

                  <div class="po-detail-item">
                    <div class="info-label">Quantity Ordered</div>
                    <div class="info-value">{{ backlogItem.purchase_order.quantity }} units</div>
                  </div>

                  <div class="po-detail-item">
                    <div class="info-label">Unit Cost</div>
                    <div class="info-value">{{ backlogItem.purchase_order.unit_cost != null ? formatCurrency(backlogItem.purchase_order.unit_cost) : 'N/A' }}</div>
                  </div>

                  <div class="po-detail-item">
                    <div class="info-label">Total Cost</div>
                    <div class="info-value">
                      {{
                        backlogItem.purchase_order.unit_cost != null && backlogItem.purchase_order.quantity != null
                          ? formatCurrency(backlogItem.purchase_order.unit_cost * backlogItem.purchase_order.quantity)
                          : 'N/A'
                      }}
                    </div>
                  </div>

                  <div class="po-detail-item">
                    <div class="info-label">Expected Delivery</div>
                    <div class="info-value">{{ formatDate(backlogItem.purchase_order.expected_delivery_date) }}</div>
                  </div>

                  <div class="po-detail-item">
                    <div class="info-label">Status</div>
                    <div class="info-value">
                      <span class="badge" :class="backlogItem.purchase_order.status || 'pending'">
                        {{ backlogItem.purchase_order.status || 'Pending' }}
                      </span>
                    </div>
                  </div>

                  <div v-if="backlogItem.purchase_order.created_at" class="po-detail-item">
                    <div class="info-label">Created</div>
                    <div class="info-value">{{ formatDate(backlogItem.purchase_order.created_at) }}</div>
                  </div>
                </div>

                <div v-if="backlogItem.purchase_order.notes" class="po-notes">
                  <div class="info-label">Notes</div>
                  <div class="notes-text">{{ backlogItem.purchase_order.notes }}</div>
                </div>
              </template>

              <div v-else class="no-po-message">
                No purchase order found for this backlog item.
              </div>
            </template>
          </div>

          <div class="modal-footer">
            <button class="btn-secondary" @click="close" :disabled="submitting">
              {{ mode === 'create' ? 'Cancel' : 'Close' }}
            </button>
            <button
              v-if="mode === 'create'"
              class="btn-primary"
              :disabled="submitting"
              @click="submitForm"
            >
              {{ submitting ? 'Creating...' : 'Create Purchase Order' }}
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

const { translateProductName } = useI18n()

const props = defineProps({
  isOpen: {
    type: Boolean,
    default: false
  },
  backlogItem: {
    type: Object,
    default: null
  },
  mode: {
    type: String,
    default: 'create'
  }
})

const emit = defineEmits(['close', 'po-created'])

// Form state
const form = ref({
  supplierName: '',
  quantity: 0,
  unitCost: null,
  expectedDeliveryDate: '',
  notes: ''
})

const submitting = ref(false)
const submitError = ref(null)

// Pre-fill quantity with shortage when backlogItem changes or modal opens
watch(
  () => [props.backlogItem, props.isOpen],
  ([item, open]) => {
    if (open && item && props.mode === 'create') {
      // Reset form, pre-filling quantity with the shortage amount
      const shortageAmount = Math.max(0, (item.quantity_needed || 0) - (item.quantity_available || 0))
      form.value = {
        supplierName: '',
        quantity: shortageAmount,
        unitCost: null,
        expectedDeliveryDate: '',
        notes: ''
      }
      submitError.value = null
    }
  },
  { immediate: true }
)

const shortage = computed(() => {
  if (!props.backlogItem) return 0
  return Math.max(0, props.backlogItem.quantity_needed - props.backlogItem.quantity_available)
})

const totalCost = computed(() => {
  if (!form.value.quantity || !form.value.unitCost) return 0
  return form.value.quantity * form.value.unitCost
})

const close = () => {
  if (!submitting.value) {
    emit('close')
  }
}

const submitForm = async () => {
  if (submitting.value) return

  submitting.value = true
  submitError.value = null

  try {
    const result = await api.createPurchaseOrder({
      backlog_item_id: props.backlogItem.id,
      supplier_name: form.value.supplierName,
      quantity: form.value.quantity,
      unit_cost: form.value.unitCost,
      expected_delivery_date: form.value.expectedDeliveryDate,
      notes: form.value.notes
    })
    emit('po-created', result)
  } catch (err) {
    // Show a meaningful error message without crashing the modal
    if (err.response?.data?.detail) {
      submitError.value = err.response.data.detail
    } else {
      submitError.value = 'Failed to create purchase order. Please try again.'
    }
    console.error('PO creation error:', err)
  } finally {
    submitting.value = false
  }
}

const formatDate = (dateString) => {
  if (!dateString) return 'N/A'
  const date = new Date(dateString)
  if (isNaN(date.getTime())) return 'N/A'
  return date.toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}

const formatCurrency = (value) => {
  if (value == null || isNaN(value)) return 'N/A'
  return value.toLocaleString('en-US', { style: 'currency', currency: 'USD' })
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  padding: 1rem;
}

.modal-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.15);
  max-width: 700px;
  width: 100%;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.close-button {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 0.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  transition: all 0.15s ease;
}

.close-button:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.modal-body {
  flex: 1;
  overflow-y: auto;
  padding: 2rem;
}

/* Item context strip at the top of the body */
.item-context {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  padding-bottom: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
  margin-bottom: 1.5rem;
}

.item-context-info {
  min-width: 0;
}

.item-name {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  margin: 0 0 0.25rem 0;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.item-sku {
  font-size: 0.875rem;
  color: #64748b;
  font-family: 'Monaco', 'Courier New', monospace;
}

.priority-badge {
  padding: 0.5rem 1rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
  flex-shrink: 0;
}

.priority-badge.high {
  background: #fecaca;
  color: #991b1b;
}

.priority-badge.medium {
  background: #fed7aa;
  color: #92400e;
}

.priority-badge.low {
  background: #ccfbf1;
  color: #115e59;
}

/* Shortage summary row */
.shortage-info {
  display: flex;
  gap: 1rem;
  margin-bottom: 1.75rem;
}

.shortage-stat {
  flex: 1;
  padding: 0.875rem 1rem;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.shortage-stat.danger {
  background: #fef2f2;
  border-color: #fecaca;
}

.shortage-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.shortage-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.shortage-stat.danger .shortage-value {
  color: #dc2626;
}

/* Form */
.po-form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.form-error {
  padding: 0.875rem 1rem;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  color: #dc2626;
  font-size: 0.875rem;
  font-weight: 500;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.form-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #374151;
}

.required {
  color: #dc2626;
}

.form-input {
  padding: 0.625rem 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.875rem;
  color: #0f172a;
  background: white;
  font-family: inherit;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  outline: none;
  width: 100%;
  box-sizing: border-box;
}

.form-input:focus {
  border-color: #0d9488;
  box-shadow: 0 0 0 3px rgba(13, 148, 136, 0.1);
}

.form-input:disabled {
  background: #f8fafc;
  color: #94a3b8;
  cursor: not-allowed;
}

.form-textarea {
  padding: 0.625rem 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.875rem;
  color: #0f172a;
  background: white;
  font-family: inherit;
  resize: vertical;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  outline: none;
  width: 100%;
  box-sizing: border-box;
  min-height: 80px;
}

.form-textarea:focus {
  border-color: #0d9488;
  box-shadow: 0 0 0 3px rgba(13, 148, 136, 0.1);
}

.form-textarea:disabled {
  background: #f8fafc;
  color: #94a3b8;
  cursor: not-allowed;
}

/* Cost preview */
.cost-preview {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0.875rem 1rem;
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  border-radius: 8px;
}

.cost-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #15803d;
}

.cost-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #15803d;
}

/* View mode: PO detail grid */
.po-detail-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1.5rem;
  margin-bottom: 1.5rem;
}

.po-detail-item {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.info-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.info-value {
  font-size: 0.938rem;
  color: #0f172a;
  font-weight: 500;
}

.info-value.po-id {
  font-family: 'Monaco', 'Courier New', monospace;
  color: #0d9488;
}

.badge {
  display: inline-block;
  padding: 0.25rem 0.625rem;
  border-radius: 4px;
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: capitalize;
}

.badge.pending {
  background: #fef3c7;
  color: #92400e;
}

.badge.approved {
  background: #ccfbf1;
  color: #115e59;
}

.badge.fulfilled,
.badge.completed {
  background: #dcfce7;
  color: #15803d;
}

.badge.cancelled {
  background: #f1f5f9;
  color: #64748b;
}

/* PO notes section */
.po-notes {
  padding-top: 1.25rem;
  border-top: 1px solid #e2e8f0;
}

.notes-text {
  margin-top: 0.5rem;
  font-size: 0.938rem;
  color: #334155;
  line-height: 1.6;
  white-space: pre-wrap;
}

/* Empty state */
.no-po-message {
  text-align: center;
  padding: 3rem 1rem;
  color: #64748b;
  font-size: 0.938rem;
}

/* Footer */
.modal-footer {
  padding: 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

.btn-secondary {
  padding: 0.625rem 1.25rem;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: #334155;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-secondary:hover:not(:disabled) {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.btn-secondary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.btn-primary {
  padding: 0.625rem 1.25rem;
  background: #0d9488;
  border: 1px solid #0d9488;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover:not(:disabled) {
  background: #0f766e;
  border-color: #0f766e;
}

.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* Modal transition animations — matches BacklogDetailModal */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-active .modal-container,
.modal-leave-active .modal-container {
  transition: transform 0.2s ease;
}

.modal-enter-from .modal-container,
.modal-leave-to .modal-container {
  transform: scale(0.95);
}
</style>
