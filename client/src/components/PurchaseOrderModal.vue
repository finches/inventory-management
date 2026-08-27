<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">
              {{
                mode === "view"
                  ? "Purchase Order Details"
                  : "Create Purchase Order"
              }}
            </h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path
                  d="M15 5L5 15M5 5L15 15"
                  stroke="currentColor"
                  stroke-width="2"
                  stroke-linecap="round"
                />
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <div class="shortage-header">
              <div class="shortage-title-section">
                <h4 class="item-name">{{ backlogItem.item_name }}</h4>
                <div class="item-sku">SKU: {{ backlogItem.item_sku }}</div>
              </div>
              <span class="priority-badge" :class="backlogItem.priority">
                {{ backlogItem.priority }} Priority
              </span>
            </div>

            <div class="info-grid">
              <div class="info-item">
                <div class="info-label">Order ID</div>
                <div class="info-value order-id">
                  {{ backlogItem.order_id }}
                </div>
              </div>
              <div class="info-item">
                <div class="info-label">Shortage</div>
                <div class="info-value">{{ shortage }} units</div>
              </div>
            </div>

            <div v-if="mode === 'view'" class="view-section">
              <div v-if="loadingPO" class="loading-po">
                Loading purchase order...
              </div>
              <div v-else-if="purchaseOrder" class="info-grid">
                <div class="info-item">
                  <div class="info-label">PO ID</div>
                  <div class="info-value order-id">{{ purchaseOrder.id }}</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Status</div>
                  <div class="info-value">
                    <span class="badge info">{{ purchaseOrder.status }}</span>
                  </div>
                </div>
                <div class="info-item">
                  <div class="info-label">Supplier</div>
                  <div class="info-value">
                    {{ purchaseOrder.supplier_name }}
                  </div>
                </div>
                <div class="info-item">
                  <div class="info-label">Quantity</div>
                  <div class="info-value">
                    {{ purchaseOrder.quantity }} units
                  </div>
                </div>
                <div class="info-item">
                  <div class="info-label">Unit Cost</div>
                  <div class="info-value">${{ purchaseOrder.unit_cost }}</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Expected Delivery</div>
                  <div class="info-value">
                    {{ formatDate(purchaseOrder.expected_delivery_date) }}
                  </div>
                </div>
              </div>
              <div v-else class="no-po">
                No purchase order found for this item.
              </div>
            </div>

            <form v-else class="po-form" @submit.prevent="submitOrder">
              <div class="form-row">
                <label class="form-label" for="po-supplier"
                  >Supplier Name</label
                >
                <input
                  id="po-supplier"
                  v-model="form.supplier_name"
                  type="text"
                  class="form-input"
                  required
                />
              </div>
              <div class="form-row">
                <label class="form-label" for="po-quantity">Quantity</label>
                <input
                  id="po-quantity"
                  v-model.number="form.quantity"
                  type="number"
                  min="1"
                  class="form-input"
                  required
                />
              </div>
              <div class="form-row">
                <label class="form-label" for="po-cost">Unit Cost</label>
                <input
                  id="po-cost"
                  v-model.number="form.unit_cost"
                  type="number"
                  min="0"
                  step="0.01"
                  class="form-input"
                  required
                />
              </div>
              <div class="form-row">
                <label class="form-label" for="po-date"
                  >Expected Delivery Date</label
                >
                <input
                  id="po-date"
                  v-model="form.expected_delivery_date"
                  type="date"
                  class="form-input"
                  required
                />
              </div>
              <div class="form-row">
                <label class="form-label" for="po-notes">Notes</label>
                <textarea
                  id="po-notes"
                  v-model="form.notes"
                  class="form-input"
                  rows="3"
                ></textarea>
              </div>
              <div v-if="submitError" class="submit-error">
                {{ submitError }}
              </div>
            </form>
          </div>

          <div class="modal-footer">
            <button class="btn-secondary" @click="close">Close</button>
            <button
              v-if="mode !== 'view'"
              class="btn-primary"
              :disabled="submitting"
              @click="submitOrder"
            >
              {{ submitting ? "Creating..." : "Create Purchase Order" }}
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup>
import { ref, computed, watch } from "vue";
import { api } from "../api";

const props = defineProps({
  isOpen: {
    type: Boolean,
    default: false,
  },
  backlogItem: {
    type: Object,
    default: null,
  },
  mode: {
    type: String,
    default: "create",
  },
});

const emit = defineEmits(["close", "po-created"]);

const purchaseOrder = ref(null);
const loadingPO = ref(false);
const submitting = ref(false);
const submitError = ref(null);

const defaultForm = () => ({
  supplier_name: "",
  quantity: props.backlogItem ? shortageFor(props.backlogItem) : 1,
  unit_cost: 0,
  expected_delivery_date: "",
  notes: "",
});

function shortageFor(item) {
  return Math.max(item.quantity_needed - item.quantity_available, 1);
}

const form = ref(defaultForm());

const shortage = computed(() => {
  if (!props.backlogItem) return 0;
  return (
    props.backlogItem.quantity_needed - props.backlogItem.quantity_available
  );
});

watch(
  () => [props.isOpen, props.mode, props.backlogItem],
  async ([isOpen, mode, backlogItem]) => {
    if (!isOpen || !backlogItem) return;
    submitError.value = null;
    if (mode === "view") {
      loadingPO.value = true;
      try {
        purchaseOrder.value = await api.getPurchaseOrderByBacklogItem(
          backlogItem.id,
        );
      } catch (err) {
        purchaseOrder.value = null;
        console.error("Failed to load purchase order:", err);
      } finally {
        loadingPO.value = false;
      }
    } else {
      form.value = defaultForm();
    }
  },
  { immediate: true },
);

const close = () => {
  emit("close");
};

const submitOrder = async () => {
  if (!props.backlogItem) return;
  submitting.value = true;
  submitError.value = null;
  try {
    const po = await api.createPurchaseOrder({
      backlog_item_id: props.backlogItem.id,
      supplier_name: form.value.supplier_name,
      quantity: form.value.quantity,
      unit_cost: form.value.unit_cost,
      expected_delivery_date: form.value.expected_delivery_date,
      notes: form.value.notes || null,
    });
    emit("po-created", po);
  } catch (err) {
    submitError.value = "Failed to create purchase order";
    console.error("Failed to create purchase order:", err);
  } finally {
    submitting.value = false;
  }
};

const formatDate = (dateString) => {
  if (!dateString) return "N/A";
  const date = new Date(dateString);
  if (isNaN(date.getTime())) return dateString;
  return date.toLocaleDateString("en-US", {
    year: "numeric",
    month: "long",
    day: "numeric",
  });
};
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

.shortage-header {
  display: flex;
  align-items: center;
  gap: 1.25rem;
  padding-bottom: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
  margin-bottom: 1.5rem;
}

.shortage-title-section {
  flex: 1;
  min-width: 0;
}

.item-name {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  margin: 0 0 0.5rem 0;
}

.item-sku {
  font-size: 0.875rem;
  color: #64748b;
  font-family: "Monaco", "Courier New", monospace;
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
  background: #dbeafe;
  color: #1e40af;
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1.5rem;
  margin-bottom: 1.5rem;
}

.info-item {
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

.info-value.order-id {
  font-family: "Monaco", "Courier New", monospace;
  color: #2563eb;
}

.view-section {
  border-top: 1px solid #e2e8f0;
  padding-top: 1.5rem;
}

.loading-po,
.no-po {
  color: #64748b;
  font-size: 0.938rem;
  text-align: center;
  padding: 1.5rem;
}

.po-form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  border-top: 1px solid #e2e8f0;
  padding-top: 1.5rem;
}

.form-row {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.form-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.form-input {
  padding: 0.625rem 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.938rem;
  color: #0f172a;
  font-family: inherit;
}

.form-input:focus {
  outline: none;
  border-color: #3b82f6;
}

.submit-error {
  color: #991b1b;
  font-size: 0.875rem;
}

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

.btn-secondary:hover {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.btn-primary {
  padding: 0.625rem 1.25rem;
  background: #3b82f6;
  border: 1px solid #3b82f6;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover:not(:disabled) {
  background: #2563eb;
  border-color: #2563eb;
}

.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.badge.info {
  background: #dbeafe;
  color: #1e40af;
  padding: 0.25rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
}

/* Modal transition animations */
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
