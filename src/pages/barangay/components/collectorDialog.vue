<script setup lang="ts">
import { ref, computed, watch } from "vue";
import { formatDate } from "../utils/pickupHelpers";

interface Collection {
  id: number;
  created_at: string;
  address: string;
  purok?: string;
  request_by: string;
  collector_assign: string | null;
  status: string;
  garbage_type: string;
  is_hazardous?: boolean;
  notes?: string;
  scheduled_date?: string;
  time_slot?: string;
}

interface Collector {
  id: string;
  username: string;
  email: string;
}

const props = defineProps<{
  modelValue: boolean;
  collection: Collection | null;
  collectors: Collector[];
  loading?: boolean;
}>();

const emit = defineEmits<{
  "update:modelValue": [value: boolean];
  assign: [collectorId: string];
}>();

const selectedCollectorId = ref<string | null>(null);

const isReassignment = computed(() => !!props.collection?.collector_assign);
const dialogTitle    = computed(() => isReassignment.value ? "Reassign Collector" : "Assign Collector");

const collectorOptions = computed(() =>
  props.collectors.map((c) => ({
    title: `${c.username} (${c.email})`,
    value: c.id,
  }))
);

const formatScheduledDate = (dateStr?: string) => {
  if (!dateStr) return null;
  const d = new Date(dateStr + "T00:00:00");
  return d.toLocaleDateString("en-PH", {
    weekday: "long", year: "numeric", month: "long", day: "numeric",
  });
};

const slotLabel = (slot?: string) => {
  if (!slot) return null;
  return slot === "morning" ? "Morning  (8:00 AM – 12:00 PM)" : "Afternoon  (1:00 PM – 5:00 PM)";
};

const slotColor = (slot?: string) =>
  slot === "morning" ? "orange-darken-2" : "blue-darken-2";

watch(() => props.modelValue, (isOpen) => {
  if (isOpen && props.collection) {
    selectedCollectorId.value = props.collection.collector_assign;
  } else if (!isOpen) {
    selectedCollectorId.value = null;
  }
});

const closeDialog = () => emit("update:modelValue", false);
const handleAssign = () => {
  if (selectedCollectorId.value) emit("assign", selectedCollectorId.value);
};
</script>

<template>
  <v-dialog :model-value="modelValue" max-width="520px" persistent>
    <v-card>
      <v-card-title class="text-h6 font-weight-bold bg-primary text-white py-3 px-4">
        <v-icon class="mr-2">{{ isReassignment ? "mdi-account-switch" : "mdi-account-plus" }}</v-icon>
        {{ dialogTitle }}
      </v-card-title>

      <v-card-text class="pt-4 px-4">
        <!-- Request Details -->
        <div v-if="collection">
          <div class="text-subtitle-2 font-weight-bold text-medium-emphasis mb-3">
            REQUEST DETAILS
          </div>

          <!-- ID -->
          <div class="detail-row">
            <v-icon size="16" color="primary">mdi-pound</v-icon>
            <span class="text-body-2">Request #{{ collection.id }}</span>
          </div>

          <!-- Purok -->
          <div v-if="collection.purok" class="detail-row">
            <v-icon size="16" color="primary">mdi-map-marker</v-icon>
            <span class="text-body-2">{{ collection.purok }}</span>
          </div>

          <!-- Address -->
          <div class="detail-row">
            <v-icon size="16" color="primary">mdi-home</v-icon>
            <span class="text-body-2">{{ collection.address }}</span>
          </div>

          <!-- E-Waste Type + Hazardous -->
          <div class="detail-row">
            <v-icon size="16" color="primary">mdi-recycle</v-icon>
            <span class="text-body-2 mr-2">{{ collection.garbage_type }}</span>
            <v-chip v-if="collection.is_hazardous" size="x-small" color="error" variant="tonal">
              <v-icon start size="10">mdi-alert</v-icon>Hazardous
            </v-chip>
          </div>

          <!-- ── SCHEDULE SECTION ── -->
          <v-divider class="my-3" />
          <div class="text-subtitle-2 font-weight-bold text-medium-emphasis mb-3">
            PICKUP SCHEDULE
          </div>

          <!-- Scheduled Date -->
          <div class="detail-row">
            <v-icon size="16" color="teal">mdi-calendar-check</v-icon>
            <span v-if="formatScheduledDate(collection.scheduled_date)" class="text-body-2 font-weight-medium">
              {{ formatScheduledDate(collection.scheduled_date) }}
            </span>
            <span v-else class="text-body-2 text-medium-emphasis">No date scheduled</span>
          </div>

          <!-- Time Slot -->
          <div class="detail-row">
            <v-icon size="16" color="teal">mdi-clock-outline</v-icon>
            <v-chip
              v-if="collection.time_slot"
              size="small"
              :color="slotColor(collection.time_slot)"
              variant="tonal"
            >
              {{ slotLabel(collection.time_slot) }}
            </v-chip>
            <span v-else class="text-body-2 text-medium-emphasis">No time slot selected</span>
          </div>

          <!-- Notes -->
          <div v-if="collection.notes" class="detail-row">
            <v-icon size="16" color="grey">mdi-note-text</v-icon>
            <span class="text-body-2 text-medium-emphasis">{{ collection.notes }}</span>
          </div>

          <!-- Date Requested -->
          <div class="detail-row">
            <v-icon size="16" color="grey">mdi-calendar-clock</v-icon>
            <span class="text-body-2 text-medium-emphasis">Requested: {{ formatDate(collection.created_at) }}</span>
          </div>
        </div>

        <v-divider class="my-4" />

        <!-- Assign Collector -->
        <v-select
          v-model="selectedCollectorId"
          :items="collectorOptions"
          label="Select Collector"
          variant="outlined"
          density="comfortable"
          prepend-inner-icon="mdi-account"
          :rules="[(v) => !!v || 'Please select a collector']"
          :hint="isReassignment
            ? 'Choose a new collector to reassign this pickup request'
            : 'Choose a collector to assign to this pickup request'"
          persistent-hint
        />

        <v-alert
          v-if="isReassignment"
          type="warning" variant="tonal" density="compact" class="mt-4"
        >
          <template #text>
            <span class="text-caption">
              This request is currently assigned. Selecting a new collector will reassign it.
            </span>
          </template>
        </v-alert>
      </v-card-text>

      <v-card-actions class="px-4 pb-4">
        <v-spacer />
        <v-btn variant="text" @click="closeDialog" :disabled="loading">Cancel</v-btn>
        <v-btn
          color="primary" variant="elevated"
          @click="handleAssign"
          :loading="loading"
          :disabled="!selectedCollectorId"
        >
          {{ isReassignment ? "Reassign Collector" : "Assign Collector" }}
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<style scoped>
.detail-row {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}
</style>