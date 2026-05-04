<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">{{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}</h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <div class="backlog-summary">
              <div class="summary-row">
                <span class="summary-label">Item</span>
                <span class="summary-value">{{ backlogItem.item_name }} ({{ backlogItem.item_sku }})</span>
              </div>
              <div class="summary-row">
                <span class="summary-label">Shortage</span>
                <span class="summary-value danger">{{ backlogItem.quantity_needed - backlogItem.quantity_available }} units short</span>
              </div>
            </div>

            <!-- Create mode -->
            <form v-if="mode === 'create'" @submit.prevent="submitPO" class="po-form">
              <div class="form-group">
                <label>Supplier Name *</label>
                <input v-model="form.supplier_name" type="text" required placeholder="Enter supplier name" />
              </div>
              <div class="form-row">
                <div class="form-group">
                  <label>Quantity *</label>
                  <input v-model.number="form.quantity" type="number" required min="1" />
                </div>
                <div class="form-group">
                  <label>Unit Cost (USD) *</label>
                  <input v-model.number="form.unit_cost" type="number" required min="0.01" step="0.01" />
                </div>
              </div>
              <div class="form-group">
                <label>Expected Delivery Date *</label>
                <input v-model="form.expected_delivery_date" type="date" required />
              </div>
              <div class="form-group">
                <label>Notes</label>
                <textarea v-model="form.notes" placeholder="Optional notes..." rows="3" />
              </div>
              <div v-if="form.quantity && form.unit_cost" class="total-cost">
                Total Cost: <strong>{{ formatTotal(form.quantity * form.unit_cost) }}</strong>
              </div>
              <div v-if="submitError" class="submit-error">{{ submitError }}</div>
              <div class="form-actions">
                <button type="button" class="btn-cancel" @click="close">Cancel</button>
                <button type="submit" class="btn-submit" :disabled="submitting">
                  {{ submitting ? 'Creating...' : 'Create Purchase Order' }}
                </button>
              </div>
            </form>

            <!-- View mode -->
            <div v-else class="po-details">
              <div class="detail-grid">
                <div class="detail-item">
                  <div class="detail-label">PO ID</div>
                  <div class="detail-value mono">{{ po.id }}</div>
                </div>
                <div class="detail-item">
                  <div class="detail-label">Supplier</div>
                  <div class="detail-value">{{ po.supplier_name }}</div>
                </div>
                <div class="detail-item">
                  <div class="detail-label">Quantity</div>
                  <div class="detail-value">{{ po.quantity }} units</div>
                </div>
                <div class="detail-item">
                  <div class="detail-label">Unit Cost</div>
                  <div class="detail-value">{{ formatTotal(po.unit_cost) }}</div>
                </div>
                <div class="detail-item">
                  <div class="detail-label">Total Cost</div>
                  <div class="detail-value"><strong>{{ formatTotal(po.quantity * po.unit_cost) }}</strong></div>
                </div>
                <div class="detail-item">
                  <div class="detail-label">Expected Delivery</div>
                  <div class="detail-value">{{ po.expected_delivery_date }}</div>
                </div>
                <div class="detail-item">
                  <div class="detail-label">Status</div>
                  <div class="detail-value"><span class="badge info">{{ po.status }}</span></div>
                </div>
                <div class="detail-item">
                  <div class="detail-label">Created</div>
                  <div class="detail-value">{{ po.created_date }}</div>
                </div>
              </div>
              <div v-if="po.notes" class="detail-notes">
                <div class="detail-label">Notes</div>
                <div class="detail-value">{{ po.notes }}</div>
              </div>
              <div class="form-actions">
                <button class="btn-cancel" @click="close">Close</button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script>
import { ref, computed, watch } from 'vue'
import { api } from '../api'

export default {
  name: 'PurchaseOrderModal',
  props: {
    isOpen: { type: Boolean, required: true },
    backlogItem: { type: Object, default: null },
    mode: { type: String, default: 'create' }
  },
  emits: ['close', 'po-created'],
  setup(props, { emit }) {
    const submitting = ref(false)
    const submitError = ref(null)

    const form = ref({
      supplier_name: '',
      quantity: '',
      unit_cost: '',
      expected_delivery_date: '',
      notes: ''
    })

    const po = computed(() => props.backlogItem?.purchase_order || {})

    watch(() => props.isOpen, (open) => {
      if (open) {
        form.value = { supplier_name: '', quantity: '', unit_cost: '', expected_delivery_date: '', notes: '' }
        submitError.value = null
      }
    })

    const formatTotal = (value) => {
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD' })
    }

    const close = () => emit('close')

    const submitPO = async () => {
      if (!props.backlogItem) return
      submitting.value = true
      submitError.value = null
      try {
        const payload = {
          backlog_item_id: props.backlogItem.id,
          supplier_name: form.value.supplier_name,
          quantity: form.value.quantity,
          unit_cost: form.value.unit_cost,
          expected_delivery_date: form.value.expected_delivery_date,
          notes: form.value.notes || null
        }
        const result = await api.createPurchaseOrder(payload)
        emit('po-created', result)
      } catch (err) {
        submitError.value = 'Failed to create purchase order. Please try again.'
      } finally {
        submitting.value = false
      }
    }

    return { form, po, submitting, submitError, formatTotal, close, submitPO }
  }
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(15, 23, 42, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  padding: 1rem;
}

.modal-container {
  background: white;
  border-radius: 12px;
  width: 100%;
  max-width: 520px;
  max-height: 90vh;
  overflow-y: auto;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.15);
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1.25rem 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.close-button {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 0.25rem;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.15s;
}

.close-button:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.modal-body {
  padding: 1.5rem;
}

.backlog-summary {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 1rem;
  margin-bottom: 1.25rem;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.summary-row {
  display: flex;
  justify-content: space-between;
  font-size: 0.875rem;
}

.summary-label {
  color: #64748b;
  font-weight: 500;
}

.summary-value {
  color: #0f172a;
  font-weight: 600;
}

.summary-value.danger {
  color: #dc2626;
}

.po-form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.form-group label {
  font-size: 0.813rem;
  font-weight: 600;
  color: #475569;
}

.form-group input,
.form-group textarea {
  padding: 0.625rem 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.875rem;
  color: #0f172a;
  transition: border-color 0.15s;
  font-family: inherit;
}

.form-group input:focus,
.form-group textarea:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.form-group textarea {
  resize: vertical;
}

.total-cost {
  font-size: 0.875rem;
  color: #475569;
  padding: 0.75rem;
  background: #eff6ff;
  border-radius: 6px;
  border: 1px solid #bfdbfe;
}

.submit-error {
  font-size: 0.875rem;
  color: #dc2626;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 6px;
  padding: 0.75rem;
}

.form-actions {
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
  margin-top: 0.5rem;
}

.btn-cancel {
  padding: 0.625rem 1.25rem;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  background: white;
  color: #475569;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.15s;
}

.btn-cancel:hover {
  background: #f8fafc;
}

.btn-submit {
  padding: 0.625rem 1.25rem;
  border: none;
  border-radius: 6px;
  background: #3b82f6;
  color: white;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}

.btn-submit:hover:not(:disabled) {
  background: #2563eb;
}

.btn-submit:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.detail-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
  margin-bottom: 1rem;
}

.detail-item {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.detail-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.detail-value {
  font-size: 0.875rem;
  color: #0f172a;
}

.detail-value.mono {
  font-family: monospace;
  font-size: 0.813rem;
}

.detail-notes {
  margin-top: 1rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.badge {
  display: inline-block;
  padding: 0.25rem 0.625rem;
  border-radius: 4px;
  font-size: 0.75rem;
  font-weight: 600;
}

.badge.info {
  background: #dbeafe;
  color: #1e40af;
}

.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}
</style>
