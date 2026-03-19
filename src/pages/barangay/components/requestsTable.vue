<script setup lang="ts">
import {
  statusColor,
  garbageTypeColor,
  formatDate,
  getCollectorName,
} from "../utils/pickupHelpers";
import type { Collection, Collector } from "@/stores/collectionsData";

const props = defineProps<{
  collections: Collection[];
  collectors: Collector[];
  loading: boolean;
}>();

const emit = defineEmits<{
  accept: [collection: Collection];
  reject: [collection: Collection];
  delete: [collection: Collection];
}>();

const handleAccept  = (c: Collection) => emit("accept", c);
const handleReject  = (c: Collection) => emit("reject", c);
const handleDelete  = (c: Collection) => emit("delete", c);

const formatScheduledDate = (dateStr?: string) => {
  if (!dateStr) return "—";
  const d = new Date(dateStr + "T00:00:00");
  return d.toLocaleDateString("en-PH", { year: "numeric", month: "short", day: "numeric" });
};

const slotLabel = (slot?: string) => {
  if (!slot) return "—";
  return slot === "morning" ? "Morning (8AM–12PM)" : "Afternoon (1PM–5PM)";
};

const slotColor = (slot?: string) => {
  if (!slot) return "grey";
  return slot === "morning" ? "orange-darken-2" : "blue-darken-2";
};
</script>

<template>
  <v-card elevation="0" class="rounded-lg table-card">
    <v-data-table
      :headers="[
        { title: 'ID',                key: 'id',               width: '70px' },
        { title: 'Purok',             key: 'purok',            width: '110px' },
        { title: 'Address',           key: 'address' },
        { title: 'E-Waste Type',      key: 'garbage_type',     width: '150px' },
        { title: 'Scheduled Date',    key: 'scheduled_date',   width: '150px' },
        { title: 'Time Slot',         key: 'time_slot',        width: '170px' },
        { title: 'Status',            key: 'status',           width: '130px' },
        { title: 'Assigned Collector',key: 'collector_assign', width: '170px' },
        { title: 'Requested On',      key: 'created_at',       width: '160px' },
        { title: 'Actions',           key: 'actions',          width: '130px', sortable: false },
      ]"
      :items="collections"
      :loading="loading"
      :items-per-page="10"
      class="elevation-0"
    >
      <!-- ID -->
      <template #item.id="{ item }">
        <span class="font-weight-medium">#{{ item.id }}</span>
      </template>

      <!-- Purok -->
      <template #item.purok="{ item }">
        <div class="d-flex align-center">
          <v-icon size="small" class="mr-1" color="primary">mdi-map-marker</v-icon>
          <span>{{ item.purok || '—' }}</span>
        </div>
      </template>

      <!-- Address -->
      <template #item.address="{ item }">
        <div class="d-flex align-center">
          <v-icon size="small" class="mr-1" color="primary">mdi-home</v-icon>
          <span>{{ item.address }}</span>
        </div>
      </template>

      <!-- Garbage Type -->
      <template #item.garbage_type="{ item }">
        <div class="d-flex align-center ga-1">
          <v-chip :color="garbageTypeColor(item.garbage_type)" size="small" variant="tonal">
            {{ item.garbage_type }}
          </v-chip>
          <v-icon v-if="item.is_hazardous" size="14" color="error" title="Hazardous">mdi-alert</v-icon>
        </div>
      </template>

      <!-- Scheduled Date -->
      <template #item.scheduled_date="{ item }">
        <div class="d-flex align-center">
          <v-icon size="small" class="mr-1" color="teal">mdi-calendar-check</v-icon>
          <span :class="item.scheduled_date ? 'font-weight-medium' : 'text-medium-emphasis'">
            {{ formatScheduledDate(item.scheduled_date) }}
          </span>
        </div>
      </template>

      <!-- Time Slot -->
      <template #item.time_slot="{ item }">
        <v-chip
          v-if="item.time_slot"
          :color="slotColor(item.time_slot)"
          size="small"
          variant="tonal"
          prepend-icon="mdi-clock-outline"
        >
          {{ slotLabel(item.time_slot) }}
        </v-chip>
        <span v-else class="text-medium-emphasis text-body-2">—</span>
      </template>

      <!-- Status -->
      <template #item.status="{ item }">
        <div class="d-flex flex-column ga-1">
          <v-chip :color="statusColor(item.status)" size="small" variant="flat">
            {{ item.status.replace("_", " ") }}
          </v-chip>
          <v-chip
            v-if="item.status === 'pending' && item.collector_assign"
            color="info" size="x-small" variant="tonal"
          >
            Awaiting Response
          </v-chip>
        </div>
      </template>

      <!-- Assigned Collector -->
      <template #item.collector_assign="{ item }">
        <div class="d-flex align-center">
          <v-icon size="small" class="mr-1" :color="item.collector_assign ? 'success' : 'grey'">
            mdi-account
          </v-icon>
          <span :class="item.collector_assign ? '' : 'text-medium-emphasis'">
            {{ getCollectorName(item.collector_assign, props.collectors) }}
          </span>
        </div>
      </template>

      <!-- Requested On -->
      <template #item.created_at="{ item }">
        <div class="text-body-2">{{ formatDate(item.created_at) }}</div>
      </template>

      <!-- Actions -->
      <template #item.actions="{ item }">
        <div class="d-flex ga-1">
          <v-tooltip
            :text="item.collector_assign ? 'Reassign Collector' : 'Accept & Assign Collector'"
            location="top"
          >
            <template #activator="{ props }">
              <v-btn
                v-bind="props"
                :icon="item.collector_assign ? 'mdi-account-switch' : 'mdi-check'"
                size="small" color="success" variant="tonal"
                @click="handleAccept(item)"
                :disabled="item.status === 'completed' || item.status === 'cancelled'"
              />
            </template>
          </v-tooltip>

          <v-tooltip text="Reject Request" location="top">
            <template #activator="{ props }">
              <v-btn
                v-bind="props"
                icon="mdi-close" size="small" color="warning" variant="tonal"
                @click="handleReject(item)"
                :disabled="item.status === 'completed' || item.status === 'cancelled'"
              />
            </template>
          </v-tooltip>

          <v-tooltip text="Delete Request" location="top">
            <template #activator="{ props }">
              <v-btn
                v-bind="props"
                icon="mdi-delete" size="small" color="error" variant="tonal"
                @click="handleDelete(item)"
              />
            </template>
          </v-tooltip>
        </div>
      </template>

      <template #loading>
        <v-skeleton-loader type="table-row@10" />
      </template>

      <template #no-data>
        <div class="text-center py-8">
          <v-icon size="64" color="grey-lighten-1">mdi-package-variant</v-icon>
          <p class="text-h6 mt-4 text-medium-emphasis">No pickup requests found</p>
          <p class="text-body-2 text-medium-emphasis">Try adjusting your filters</p>
        </div>
      </template>
    </v-data-table>
  </v-card>
</template>

<style scoped>
.rounded-lg { border-radius: 12px; }
.table-card { background-color: white !important; }
.table-card :deep(.v-table) { background-color: white !important; }
.v-theme--dark .table-card { background-color: rgb(var(--v-theme-surface)) !important; }
.v-theme--dark .table-card :deep(.v-table) { background-color: rgb(var(--v-theme-surface)) !important; }
</style>