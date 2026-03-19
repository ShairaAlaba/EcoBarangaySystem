<script setup lang="ts">
import { ref, computed, onMounted } from "vue";
import { storeToRefs } from "pinia";
import { useAuthUserStore } from "@/stores/authUser";
import { useCollectionsStore } from "@/stores/collectionsData";
import type { CollectionWithEmails } from "@/stores/collectionsData";

const authStore = useAuthUserStore();
const collectionsStore = useCollectionsStore();
const { userData } = storeToRefs(authStore);

const myCollections = ref<CollectionWithEmails[]>([]);
const loading = ref(false);

// Calendar nav
const today = new Date();
today.setHours(0, 0, 0, 0);
const calYear = ref(today.getFullYear());
const calMonth = ref(today.getMonth());

const calTitle = computed(() =>
  new Date(calYear.value, calMonth.value, 1)
    .toLocaleString("default", { month: "long", year: "numeric" })
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
  if (calMonth.value === 0) { calMonth.value = 11; calYear.value--; }
  else calMonth.value--;
};
const nextMonth = () => {
  if (calMonth.value === 11) { calMonth.value = 0; calYear.value++; }
  else calMonth.value++;
};

const getDateStr = (day: number) =>
  `${calYear.value}-${String(calMonth.value + 1).padStart(2, "0")}-${String(day).padStart(2, "0")}`;

// Map: dateStr → list of my collections on that day
const myScheduleMap = computed(() => {
  const map: Record<string, CollectionWithEmails[]> = {};
  myCollections.value.forEach((c) => {
    if (c.scheduled_date) {
      if (!map[c.scheduled_date]) map[c.scheduled_date] = [];
      map[c.scheduled_date].push(c);
    }
  });
  return map;
});

const hasSchedule = (day: number) => !!myScheduleMap.value[getDateStr(day)]?.length;

const getDayClass = (day: number | null) => {
  if (!day) return "";
  const ds = getDateStr(day);
  const d = new Date(calYear.value, calMonth.value, day);
  if (selectedDate.value === ds) return "sch-day--selected";
  if (myScheduleMap.value[ds]?.length) return "sch-day--scheduled";
  if (d < today) return "sch-day--past";
  return "sch-day--normal";
};

const selectedDate = ref("");
const selectedItems = computed(() =>
  selectedDate.value ? (myScheduleMap.value[selectedDate.value] || []) : []
);
const selectedDateFormatted = computed(() => {
  if (!selectedDate.value) return "";
  const d = new Date(selectedDate.value + "T00:00:00");
  return d.toLocaleDateString("en-PH", { weekday: "long", year: "numeric", month: "long", day: "numeric" });
});

const selectDay = (day: number | null) => {
  if (!day) return;
  selectedDate.value = getDateStr(day);
};

const statusColor = (status: string) => {
  const map: Record<string, string> = {
    pending: "orange", in_progress: "blue", completed: "green", cancelled: "red",
  };
  return map[status] || "grey";
};

const slotLabel = (slot: string) =>
  slot === "morning" ? "Morning (8AM–12PM)" : slot === "afternoon" ? "Afternoon (1PM–5PM)" : slot;

const fetchMyCollections = async () => {
  if (!userData.value?.id) return;
  loading.value = true;
  try {
    const data = await collectionsStore.fetchCollectionsByRequestBy(userData.value.id);
    myCollections.value = (data as CollectionWithEmails[]).filter((c) => c.scheduled_date);
  } catch (e) {
    console.error(e);
  } finally {
    loading.value = false;
  }
};

// Count scheduled items this month
const thisMonthCount = computed(() => {
  const prefix = `${calYear.value}-${String(calMonth.value + 1).padStart(2, "0")}`;
  return myCollections.value.filter((c) => c.scheduled_date?.startsWith(prefix)).length;
});

onMounted(fetchMyCollections);
</script>

<template>
  <v-card elevation="4" rounded="lg" class="h-100">
    <v-card-title class="d-flex align-center bg-deep-purple-darken-1 text-white py-3 px-4">
      <v-icon class="mr-2">mdi-calendar-month</v-icon>
      My Pickup Schedule
      <v-spacer />
      <v-chip size="small" color="white" variant="tonal" v-if="!loading">
        {{ thisMonthCount }} this month
      </v-chip>
    </v-card-title>

    <v-card-text class="pa-3">
      <div v-if="loading" class="d-flex justify-center pa-4">
        <v-progress-circular indeterminate color="deep-purple" size="28" />
      </div>

      <template v-else>
        <!-- Month nav -->
        <div class="d-flex align-center justify-space-between mb-2">
          <v-btn icon="mdi-chevron-left" variant="text" density="compact" @click="prevMonth" />
          <span class="font-weight-bold text-body-2">{{ calTitle }}</span>
          <v-btn icon="mdi-chevron-right" variant="text" density="compact" @click="nextMonth" />
        </div>

        <!-- Header -->
        <div class="sch-grid sch-grid--header mb-1">
          <div v-for="h in ['Mon','Tue','Wed','Thu','Fri']" :key="h" class="sch-cell sch-cell--header">
            {{ h }}
          </div>
        </div>

        <!-- Days -->
        <div class="sch-grid">
          <div v-for="(day, idx) in calDays" :key="idx"
            class="sch-cell" :class="day ? getDayClass(day) : 'sch-day--empty'"
            @click="selectDay(day)">
            <template v-if="day">
              <span class="sch-num">{{ day }}</span>
              <div v-if="hasSchedule(day)" class="sch-dot" />
            </template>
          </div>
        </div>

        <!-- Legend -->
        <div class="d-flex flex-wrap ga-2 mt-2">
          <div class="d-flex align-center ga-1">
            <div class="legend-swatch swatch--scheduled" />
            <span class="text-caption">Your schedule</span>
          </div>
          <div class="d-flex align-center ga-1">
            <div class="legend-swatch swatch--selected" />
            <span class="text-caption">Selected</span>
          </div>
        </div>

        <!-- Selected day detail -->
        <v-expand-transition>
          <div v-if="selectedDate" class="mt-3">
            <v-divider class="mb-2" />
            <div class="text-caption font-weight-bold mb-2 text-medium-emphasis">
              {{ selectedDateFormatted }}
            </div>
            <div v-if="selectedItems.length === 0" class="text-caption text-medium-emphasis">
              No pickups scheduled on this day.
            </div>
            <v-list v-else density="compact" class="pa-0" rounded="lg">
              <v-list-item v-for="item in selectedItems" :key="item.id"
                class="mb-1" rounded="lg"
                :style="{ background: 'rgba(var(--v-theme-surface-variant), 0.5)' }">
                <template #prepend>
                  <v-icon color="deep-purple" size="18">mdi-recycle</v-icon>
                </template>
                <v-list-item-title class="text-caption font-weight-medium">
                  {{ item.garbage_type }}
                </v-list-item-title>
                <v-list-item-subtitle class="text-caption">
                  <v-chip size="x-small" :color="item.time_slot === 'morning' ? 'orange' : 'blue'"
                    variant="tonal" class="mr-1">
                    <v-icon start size="10">mdi-clock-outline</v-icon>
                    {{ slotLabel(item.time_slot || '') }}
                  </v-chip>
                  <v-chip size="x-small" :color="statusColor(item.status)" variant="tonal">
                    {{ item.status }}
                  </v-chip>
                </v-list-item-subtitle>
              </v-list-item>
            </v-list>
          </div>
        </v-expand-transition>
      </template>
    </v-card-text>
  </v-card>
</template>

<style scoped lang="scss">
.sch-grid {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 3px;
  &--header .sch-cell--header {
    font-size: 10px; font-weight: 700; text-align: center;
    color: rgba(0,0,0,0.45); text-transform: uppercase;
  }
}

.sch-cell {
  aspect-ratio: 1; display: flex; flex-direction: column;
  align-items: center; justify-content: center;
  border-radius: 6px; cursor: pointer; position: relative;
  min-height: 34px; user-select: none;
  transition: background 0.12s, transform 0.1s;
}

.sch-num { font-size: 12px; font-weight: 600; line-height: 1; }
.sch-dot {
  width: 5px; height: 5px; border-radius: 50%;
  background: #7b1fa2; margin-top: 2px;
}

.sch-day {
  &--empty    { cursor: default; background: transparent; }
  &--normal   { background: #f5f5f5; &:hover { background: #ede7f6; transform: scale(1.08); } }
  &--past     { background: #f5f5f5; opacity: 0.4; cursor: default; }
  &--scheduled { background: #ede7f6; border: 1.5px solid #7b1fa2; &:hover { background: #d1c4e9; } }
  &--selected  { background: #7b1fa2 !important; .sch-num { color: #fff; } .sch-dot { background: #fff; } }
}

.legend-swatch {
  width: 12px; height: 12px; border-radius: 3px; display: inline-block;
  &.swatch--scheduled { background: #ede7f6; border: 1.5px solid #7b1fa2; }
  &.swatch--selected  { background: #7b1fa2; }
}
</style>
