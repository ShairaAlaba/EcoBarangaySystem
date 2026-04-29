<script setup lang="ts">
import { ref, computed, onMounted } from "vue";
import { useCollectionsStore } from "@/stores/collectionsData";
import type { CollectionWithEmails } from "@/stores/collectionsData";
import InnerLayoutWrapper from "@/layouts/InnerLayoutWrapper.vue";

type Slot = "morning" | "afternoon" | "all";
type DayStats = { total: number; morning: number; afternoon: number };

const collectionsStore = useCollectionsStore();

const today = new Date();
today.setHours(0, 0, 0, 0);
const pad = (n: number) => String(n).padStart(2, "0");
const todayStr = `${today.getFullYear()}-${pad(today.getMonth() + 1)}-${pad(today.getDate())}`;

const allScheduled = ref<CollectionWithEmails[]>([]);
const loading = ref(false);
const selectedDate = ref(todayStr);
const searchQuery = ref("");
const slotFilter = ref<Slot>("all");

// Calendar nav
const calYear = ref(today.getFullYear());
const calMonth = ref(today.getMonth());

const calTitle = computed(() =>
  new Date(calYear.value, calMonth.value, 1).toLocaleString("default", {
    month: "long",
    year: "numeric",
  }),
);

const calDays = computed(() => {
  const year = calYear.value;
  const month = calMonth.value;
  const firstDow = new Date(year, month, 1).getDay();
  const daysInMonth = new Date(year, month + 1, 0).getDate();
  let startCol = (firstDow + 6) % 7;
  if (startCol > 4) startCol = 0;

  const days: (number | null)[] = [];
  for (let i = 0; i < startCol; i++) days.push(null);
  for (let d = 1; d <= daysInMonth; d++) {
    const dow = new Date(year, month, d).getDay();
    if (dow !== 0 && dow !== 6) days.push(d);
  }
  return days;
});

const prevMonth = () => {
  if (calMonth.value === 0) {
    calMonth.value = 11;
    calYear.value--;
  } else calMonth.value--;
};
const nextMonth = () => {
  if (calMonth.value === 11) {
    calMonth.value = 0;
    calYear.value++;
  } else calMonth.value++;
};

const getDateStr = (day: number) =>
  `${calYear.value}-${pad(calMonth.value + 1)}-${pad(day)}`;

const isPastDate = (day: number) => {
  const dateStr = getDateStr(day);
  return dateStr < todayStr;
};

// Schedule map and memoized stats
const scheduleMap = computed(() => {
  const map: Record<string, CollectionWithEmails[]> = {};
  for (const c of allScheduled.value) {
    if (c.scheduled_date) {
      (map[c.scheduled_date] ??= []).push(c);
    }
  }
  return map;
});

const statsMap = computed(() => {
  const map: Record<string, DayStats> = {};
  for (const date in scheduleMap.value) {
    const items = scheduleMap.value[date] || [];
    const morning = items.filter((c) => c.time_slot === "morning").length;
    const afternoon = items.filter((c) => c.time_slot === "afternoon").length;
    map[date] = { total: items.length, morning, afternoon };
  }
  return map;
});

const MORNING_MAX = 50;
const AFTERNOON_MAX = 50;

const getDayStats = (day: number) =>
  statsMap.value[getDateStr(day)] ?? { total: 0, morning: 0, afternoon: 0 };
const isFull = (day: number) => {
  const s = getDayStats(day);
  return s.morning >= MORNING_MAX && s.afternoon >= AFTERNOON_MAX;
};

const getDayClass = (day: number | null) => {
  if (!day) return "";
  const ds = getDateStr(day);
  if (selectedDate.value === ds) return "brd-day--selected";
  if (isPastDate(day)) return "brd-day--past";
  const stats = getDayStats(day);
  if (stats.total === 0) return "brd-day--empty-slot";
  if (isFull(day)) return "brd-day--full";
  return "brd-day--has-items";
};

const selectDay = (day: number | null) => {
  if (!day || isPastDate(day)) return;
  selectedDate.value = getDateStr(day);
  slotFilter.value = "all";
  searchQuery.value = "";
};

// Selected day residents and filters
const selectedDayItems = computed(() =>
  selectedDate.value ? scheduleMap.value[selectedDate.value] || [] : [],
);

const filteredItems = computed(() => {
  let items = selectedDayItems.value;
  if (slotFilter.value !== "all")
    items = items.filter((c) => c.time_slot === slotFilter.value);
  const q = searchQuery.value.trim().toLowerCase();
  if (q) {
    items = items.filter(
      (c) =>
        c.address?.toLowerCase().includes(q) ||
        c.requester_email?.toLowerCase().includes(q) ||
        c.requester_name?.toLowerCase().includes(q) ||
        c.garbage_type?.toLowerCase().includes(q) ||
        c.purok?.toLowerCase().includes(q),
    );
  }
  return items;
});

const morningItems = computed(() =>
  selectedDayItems.value.filter((c) => c.time_slot === "morning"),
);
const afternoonItems = computed(() =>
  selectedDayItems.value.filter((c) => c.time_slot === "afternoon"),
);

const selectedDateFormatted = computed(() => {
  if (!selectedDate.value) return "";
  const d = new Date(selectedDate.value + "T00:00:00");
  return d.toLocaleDateString("en-PH", {
    weekday: "long",
    year: "numeric",
    month: "long",
    day: "numeric",
  });
});

const statusColor = (status: string) => {
  const map: Record<string, string> = {
    pending: "orange",
    in_progress: "blue",
    completed: "green",
    cancelled: "red",
  };
  return map[status] || "grey";
};

const slotChipColor = (slot: string) =>
  slot === "morning" ? "orange-darken-2" : "blue-darken-2";

// Month summary counts
const monthPrefix = computed(
  () => `${calYear.value}-${pad(calMonth.value + 1)}`,
);
const monthMorningTotal = computed(
  () =>
    allScheduled.value.filter(
      (c) =>
        c.scheduled_date?.startsWith(monthPrefix.value) &&
        c.time_slot === "morning",
    ).length,
);
const monthAfternoonTotal = computed(
  () =>
    allScheduled.value.filter(
      (c) =>
        c.scheduled_date?.startsWith(monthPrefix.value) &&
        c.time_slot === "afternoon",
    ).length,
);
const monthTotal = computed(
  () => monthMorningTotal.value + monthAfternoonTotal.value,
);

// Data fetch
const fetchData = async () => {
  loading.value = true;
  try {
    allScheduled.value = await collectionsStore.fetchScheduledCollections();
  } catch (e) {
    console.error(e);
  } finally {
    loading.value = false;
  }
};

onMounted(fetchData);
</script>

<template>
  <InnerLayoutWrapper>
    <template #content>
      <v-container fluid class="pa-6">
        <!-- Header -->
        <v-row class="mb-4">
          <v-col cols="12">
            <div
              class="d-flex align-center justify-space-between flex-wrap ga-3"
            >
              <div>
                <h1 class="text-h4 font-weight-bold mb-1">
                  Collection Schedule
                </h1>
                <p class="text-body-2 text-medium-emphasis">
                  View all resident-scheduled pickups. Each day has max 50
                  morning + 50 afternoon slots.
                </p>
              </div>
              <v-btn
                color="primary"
                prepend-icon="mdi-refresh"
                variant="tonal"
                @click="fetchData"
                :loading="loading"
              >
                Refresh
              </v-btn>
            </div>
          </v-col>
        </v-row>

        <!-- Month Summary Cards -->
        <v-row class="mb-4">
          <v-col cols="12" sm="4">
            <v-card rounded="lg" color="primary" variant="tonal">
              <v-card-text class="py-3 px-4">
                <div class="text-caption text-medium-emphasis">
                  This Month Total
                </div>
                <div class="text-h5 font-weight-bold">{{ monthTotal }}</div>
                <div class="text-caption">scheduled pickups</div>
              </v-card-text>
            </v-card>
          </v-col>
          <v-col cols="12" sm="4">
            <v-card rounded="lg" color="orange" variant="tonal">
              <v-card-text class="py-3 px-4">
                <div class="text-caption text-medium-emphasis">
                  Morning Slots Used
                </div>
                <div class="text-h5 font-weight-bold">
                  {{ monthMorningTotal }}
                </div>
                <div class="text-caption">8:00 AM – 12:00 PM</div>
              </v-card-text>
            </v-card>
          </v-col>
          <v-col cols="12" sm="4">
            <v-card rounded="lg" color="blue" variant="tonal">
              <v-card-text class="py-3 px-4">
                <div class="text-caption text-medium-emphasis">
                  Afternoon Slots Used
                </div>
                <div class="text-h5 font-weight-bold">
                  {{ monthAfternoonTotal }}
                </div>
                <div class="text-caption">1:00 PM – 5:00 PM</div>
              </v-card-text>
            </v-card>
          </v-col>
        </v-row>

        <v-row>
          <!-- Left: Calendar -->
          <v-col cols="12" md="5">
            <v-card rounded="lg" elevation="3" class="h-100 schedule-card">
              <v-card-title
                class="bg-primary text-white py-3 px-4 d-flex align-center"
              >
                <v-icon class="mr-2">mdi-calendar-month</v-icon>
                Schedule Overview
              </v-card-title>

              <v-card-text class="pa-4">
                <div v-if="loading" class="d-flex justify-center pa-6">
                  <v-progress-circular indeterminate color="primary" />
                </div>

                <template v-else>
                  <!-- Month nav -->
                  <div class="d-flex align-center justify-space-between mb-3">
                    <v-btn
                      icon="mdi-chevron-left"
                      variant="text"
                      density="compact"
                      @click="prevMonth"
                    />
                    <span class="font-weight-bold text-body-1">{{
                      calTitle
                    }}</span>
                    <v-btn
                      icon="mdi-chevron-right"
                      variant="text"
                      density="compact"
                      @click="nextMonth"
                    />
                  </div>

                  <!-- Weekday headers -->
                  <div class="brd-grid brd-grid--header mb-1">
                    <div
                      v-for="h in ['Mon', 'Tue', 'Wed', 'Thu', 'Fri']"
                      :key="h"
                      class="brd-cell brd-cell--header"
                    >
                      {{ h }}
                    </div>
                  </div>

                  <!-- Days -->
                  <div class="brd-grid">
                    <div
                      v-for="(day, idx) in calDays"
                      :key="idx"
                      class="brd-cell"
                      :class="day ? getDayClass(day) : 'brd-day--null'"
                      :aria-disabled="day ? isPastDate(day) : undefined"
                      :tabindex="day && !isPastDate(day) ? 0 : -1"
                      @click="selectDay(day)"
                    >
                      <template v-if="day">
                        <span class="brd-num">{{ day }}</span>
                        <span
                          v-if="getDayStats(day).total > 0"
                          class="brd-count"
                        >
                          {{ getDayStats(day).total }}
                        </span>
                        <div class="brd-slot-dots">
                          <span
                            class="brd-dot"
                            :class="
                              getDayStats(day).morning >= MORNING_MAX
                                ? 'dot-full'
                                : 'dot-ok'
                            "
                            title="Morning"
                          />
                          <span
                            class="brd-dot"
                            :class="
                              getDayStats(day).afternoon >= AFTERNOON_MAX
                                ? 'dot-full'
                                : 'dot-ok'
                            "
                            title="Afternoon"
                          />
                        </div>
                      </template>
                    </div>
                  </div>

                  <!-- Legend -->
                  <v-divider class="my-3" />
                  <div class="d-flex flex-wrap ga-2">
                    <div class="d-flex align-center ga-1">
                      <div class="brd-legend brd-legend--items" />
                      <span class="text-caption">Has pickups</span>
                    </div>
                    <div class="d-flex align-center ga-1">
                      <div class="brd-legend brd-legend--full" />
                      <span class="text-caption">Fully booked</span>
                    </div>
                    <div class="d-flex align-center ga-1">
                      <div class="brd-legend brd-legend--selected" />
                      <span class="text-caption">Selected</span>
                    </div>
                    <div class="d-flex align-center ga-1">
                      <span class="dot-legend dot-ok" /><span
                        class="text-caption"
                        >Slot open</span
                      >
                    </div>
                    <div class="d-flex align-center ga-1">
                      <span class="dot-legend dot-full" /><span
                        class="text-caption"
                        >Slot full</span
                      >
                    </div>
                  </div>
                </template>
              </v-card-text>
            </v-card>
          </v-col>

          <!-- Right: Resident List -->
          <v-col cols="12" md="7">
            <v-card rounded="lg" elevation="3" class="h-100 schedule-card">
              <v-card-title
                class="bg-teal-darken-1 text-white py-3 px-4 d-flex align-center"
              >
                <v-icon class="mr-2">mdi-account-group</v-icon>
                <span>
                  {{
                    selectedDate
                      ? selectedDateFormatted
                      : "Select a date to view residents"
                  }}
                </span>
              </v-card-title>

              <v-card-text class="pa-0">
                <!-- No date selected -->
                <div
                  v-if="!selectedDate"
                  class="d-flex flex-column align-center justify-center pa-10 text-medium-emphasis"
                >
                  <v-icon size="48" class="mb-3" color="grey-lighten-1"
                    >mdi-calendar-cursor</v-icon
                  >
                  <p class="text-body-2">
                    Click on any date in the calendar to view scheduled
                    residents.
                  </p>
                </div>

                <template v-else>
                  <!-- Slot summary chips -->
                  <div
                    class="d-flex align-center ga-2 flex-wrap px-4 pt-3 pb-2"
                  >
                    <v-chip
                      prepend-icon="mdi-weather-sunny"
                      color="orange"
                      variant="tonal"
                      size="small"
                    >
                      Morning: {{ morningItems.length }} / {{ MORNING_MAX }}
                    </v-chip>
                    <v-chip
                      prepend-icon="mdi-weather-partly-cloudy"
                      color="blue"
                      variant="tonal"
                      size="small"
                    >
                      Afternoon: {{ afternoonItems.length }} /
                      {{ AFTERNOON_MAX }}
                    </v-chip>
                    <v-chip
                      prepend-icon="mdi-account-multiple"
                      color="primary"
                      variant="tonal"
                      size="small"
                    >
                      Total: {{ selectedDayItems.length }}
                    </v-chip>
                  </div>

                  <!-- Search + filter -->
                  <div class="px-4 pb-3 d-flex ga-2 flex-wrap">
                    <v-text-field
                      v-model="searchQuery"
                      prepend-inner-icon="mdi-magnify"
                      placeholder="Search resident, address, type..."
                      variant="outlined"
                      density="compact"
                      hide-details
                      clearable
                      style="min-width: 200px; flex: 1"
                    />
                    <v-btn-toggle
                      v-model="slotFilter"
                      density="compact"
                      mandatory
                      color="primary"
                      variant="outlined"
                    >
                      <v-btn value="all" size="small">All</v-btn>
                      <v-btn value="morning" size="small">
                        <v-icon start size="14">mdi-weather-sunny</v-icon>AM
                      </v-btn>
                      <v-btn value="afternoon" size="small">
                        <v-icon start size="14"
                          >mdi-weather-partly-cloudy</v-icon
                        >PM
                      </v-btn>
                    </v-btn-toggle>
                  </div>

                  <!-- No results -->
                  <div
                    v-if="filteredItems.length === 0"
                    class="text-center pa-8 text-medium-emphasis"
                  >
                    <v-icon size="36" class="mb-2" color="grey-lighten-1"
                      >mdi-calendar-blank</v-icon
                    >
                    <p class="text-body-2">
                      No pickups found for this selection.
                    </p>
                  </div>

                  <!-- Resident list -->
                  <v-list v-else class="resident-list" density="compact">
                    <template
                      v-for="(item, idx) in filteredItems"
                      :key="item.id"
                    >
                      <v-list-item class="py-2 px-4">
                        <template #prepend>
                          <v-avatar
                            :color="
                              item.time_slot === 'morning' ? 'orange' : 'blue'
                            "
                            variant="tonal"
                            size="36"
                            class="mr-3"
                          >
                            <v-icon size="18">
                              {{
                                item.time_slot === "morning"
                                  ? "mdi-weather-sunny"
                                  : "mdi-weather-partly-cloudy"
                              }}
                            </v-icon>
                          </v-avatar>
                        </template>

                        <v-list-item-title
                          class="font-weight-medium text-body-2"
                        >
                          {{ item.requester_name || "Unknown Resident" }}
                        </v-list-item-title>

                        <v-list-item-subtitle class="mt-1">
                          <div class="d-flex flex-wrap ga-1 align-center">
                            <v-chip
                              size="x-small"
                              :color="slotChipColor(item.time_slot || '')"
                              variant="tonal"
                            >
                              <v-icon start size="10">mdi-clock-outline</v-icon>
                              {{
                                item.time_slot === "morning"
                                  ? "8AM–12PM"
                                  : "1PM–5PM"
                              }}
                            </v-chip>
                            <v-chip
                              size="x-small"
                              color="primary"
                              variant="tonal"
                            >
                              {{ item.garbage_type }}
                            </v-chip>
                            <v-chip
                              v-if="item.purok"
                              size="x-small"
                              color="teal"
                              variant="tonal"
                            >
                              {{ item.purok }}
                            </v-chip>
                            <v-chip
                              size="x-small"
                              :color="statusColor(item.status)"
                              variant="tonal"
                            >
                              {{ item.status }}
                            </v-chip>
                            <v-icon
                              v-if="item.is_hazardous"
                              size="14"
                              color="error"
                              title="Hazardous"
                            >
                              mdi-alert
                            </v-icon>
                          </div>
                          <div
                            class="text-caption mt-1 text-truncate"
                            style="max-width: 280px"
                          >
                            {{ item.address }}
                          </div>
                        </v-list-item-subtitle>

                        <template #append>
                          <div class="text-caption text-right">
                            #{{ item.id }}
                          </div>
                        </template>
                      </v-list-item>
                      <v-divider v-if="idx < filteredItems.length - 1" inset />
                    </template>
                  </v-list>
                </template>
              </v-card-text>
            </v-card>
          </v-col>
        </v-row>
      </v-container>
    </template>
  </InnerLayoutWrapper>
</template>

<script lang="ts">
export default { name: "ScheduleCalendarView" };
</script>

<style scoped lang="scss">
/* Calendar grid – Mon to Fri only (5 columns) */
.brd-grid {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 5px;
  &--header .brd-cell--header {
    font-size: 10px;
    font-weight: 700;
    text-align: center;
    color: rgba(var(--v-theme-on-surface), 0.6);
    text-transform: uppercase;
  }
}

.brd-cell {
  aspect-ratio: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  border-radius: 8px;
  cursor: pointer;
  user-select: none;
  min-height: 46px;
  transition:
    background 0.12s,
    transform 0.1s;
  position: relative;
  color: rgb(var(--v-theme-on-surface));
}

.brd-num {
  font-size: 13px;
  font-weight: 700;
  line-height: 1;
  color: inherit;
}
.brd-count {
  font-size: 10px;
  font-weight: 600;
  color: rgb(var(--v-theme-primary));
  position: absolute;
  top: 3px;
  right: 5px;
}

.brd-slot-dots {
  display: flex;
  gap: 3px;
  margin-top: 3px;
}
.brd-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  &.dot-ok {
    background: rgb(var(--v-theme-success));
  }
  &.dot-full {
    background: rgb(var(--v-theme-error));
  }
}

.brd-day {
  &--null {
    cursor: default;
    background: transparent;
  }
  &--empty-slot {
    background: rgb(var(--v-theme-surface-variant));
    &:hover {
      background: rgba(var(--v-theme-on-surface), 0.08);
    }
  }
  &--past {
    background: rgba(var(--v-theme-on-surface), 0.04);
    color: rgba(var(--v-theme-on-surface), 0.35);
    cursor: not-allowed;
    opacity: 0.75;
    &:hover {
      background: rgba(var(--v-theme-on-surface), 0.04);
      transform: none;
    }
  }
  &--has-items {
    background: rgba(var(--v-theme-primary), 0.12);
    border: 1.5px solid rgba(var(--v-theme-primary), 0.5);
    &:hover {
      background: rgba(var(--v-theme-primary), 0.18);
      transform: scale(1.07);
    }
  }
  &--full {
    background: rgba(var(--v-theme-error), 0.12);
    border: 1.5px solid rgba(var(--v-theme-error), 0.5);
    cursor: default;
  }
  &--selected {
    background: rgb(var(--v-theme-primary)) !important;
    color: rgb(var(--v-theme-on-primary)) !important;
    .brd-num {
      color: rgb(var(--v-theme-on-primary));
    }
    .brd-count {
      color: rgba(var(--v-theme-on-primary), 0.82);
    }
  }
}

/* Legend */
.brd-legend {
  width: 14px;
  height: 14px;
  border-radius: 4px;
  display: inline-block;
  &--items {
    background: rgba(var(--v-theme-primary), 0.12);
    border: 1.5px solid rgba(var(--v-theme-primary), 0.5);
  }
  &--full {
    background: rgba(var(--v-theme-error), 0.12);
    border: 1.5px solid rgba(var(--v-theme-error), 0.5);
  }
  &--selected {
    background: rgb(var(--v-theme-primary));
  }
}

.dot-legend {
  display: inline-block;
  width: 8px;
  height: 8px;
  border-radius: 50%;
  &.dot-ok {
    background: #4caf50;
  }
  &.dot-full {
    background: #f44336;
  }
}

.resident-list {
  max-height: 500px;
  overflow-y: auto;
  background-color: white;
}

.v-theme--dark .resident-list {
  background-color: rgb(var(--v-theme-surface));
}

.schedule-card {
  background-color: white;
  border: 1px solid rgba(var(--v-theme-on-surface), 0.12);
}

.v-theme--dark .schedule-card {
  background-color: rgb(var(--v-theme-surface));
  border-color: rgba(var(--v-theme-on-surface), 0.14);
}
</style>
