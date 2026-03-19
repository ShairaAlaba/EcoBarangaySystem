<script setup lang="ts">
import { ref, computed, watch } from "vue";
import { useCollectionsStore } from "@/stores/collectionsData";
import { useAuthUserStore } from "@/stores/authUser";
import { useToast } from "vue-toastification";
import { PUROK_OPTIONS } from "@/utils/constants";
import type { CreateCollectionData } from "@/stores/collectionsData";
import {
  getGarbageTypes,
  validateCollectionRequest,
  getGarbageTypeIcon,
} from "@/utils/collectionHelpers";

const props = defineProps<{
  modelValue: boolean;
  initialData?: { purok?: string; notes?: string };
}>();

const emit = defineEmits<{
  "update:modelValue": [value: boolean];
  "collection-created": [collection: any];
}>();

const collectionsStore = useCollectionsStore();
const authStore = useAuthUserStore();
const toast = useToast();

const formRef    = ref();
const loading    = ref(false);
const submitError = ref("");
const slotCounts  = ref<Record<string, { morning: number; afternoon: number }>>({});
const loadingSlots = ref(false);

// ── Calendar ──────────────────────────────────────────────────────────────
const today = new Date();
today.setHours(0, 0, 0, 0);

const calYear  = ref(today.getFullYear());
const calMonth = ref(today.getMonth());

const calTitle = computed(() =>
  new Date(calYear.value, calMonth.value, 1)
    .toLocaleString("default", { month: "long", year: "numeric" })
);

const calDays = computed(() => {
  const year  = calYear.value;
  const month = calMonth.value;
  const daysInMonth = new Date(year, month + 1, 0).getDate();
  const firstDow = new Date(year, month, 1).getDay(); // 0=Sun
  let startCol = (firstDow + 6) % 7; // Mon=0…Sun=6
  if (startCol > 4) startCol = 0;    // if 1st is Sat/Sun, no leading gap

  const days: (number | null)[] = [];
  for (let i = 0; i < startCol; i++) days.push(null);
  for (let d = 1; d <= daysInMonth; d++) {
    const dow = new Date(year, month, d).getDay();
    if (dow !== 0 && dow !== 6) days.push(d); // skip Sat & Sun
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

const isPast = (day: number) =>
  new Date(calYear.value, calMonth.value, day) < today;

const MORNING_MAX   = 50;
const AFTERNOON_MAX = 50;

const getMorningCount   = (day: number) => slotCounts.value[getDateStr(day)]?.morning   ?? 0;
const getAfternoonCount = (day: number) => slotCounts.value[getDateStr(day)]?.afternoon ?? 0;
const isMorningFull     = (day: number) => getMorningCount(day)   >= MORNING_MAX;
const isAfternoonFull   = (day: number) => getAfternoonCount(day) >= AFTERNOON_MAX;
const isDayFull         = (day: number) => isMorningFull(day) && isAfternoonFull(day);

const getDayClass = (day: number | null) => {
  if (!day) return "cal-cell--empty";
  if (isPast(day))    return "cal-day--past";
  if (isDayFull(day)) return "cal-day--full";
  if (formData.value.scheduled_date === getDateStr(day)) return "cal-day--selected";
  return "cal-day--available";
};

const selectDay = (day: number | null) => {
  if (!day || isPast(day) || isDayFull(day)) return;
  formData.value.scheduled_date = getDateStr(day);
  formData.value.time_slot = "";
  submitError.value = "";
};

// ── Form Data ─────────────────────────────────────────────────────────────
const formData = ref({
  purok: "", address: "", garbage_type: "", notes: "",
  is_hazardous: false, scheduled_date: "", time_slot: "",
});

const garbageTypes          = getGarbageTypes();
const garbageTypesWithIcons = computed(() =>
  garbageTypes.map((t) => ({ text: t, value: t, icon: getGarbageTypeIcon(t) }))
);

const availableTimeSlots = computed(() => {
  if (!formData.value.scheduled_date) return [];
  const day = parseInt(formData.value.scheduled_date.split("-")[2]);
  return [
    {
      title: isMorningFull(day)
        ? "Morning (8:00 AM – 12:00 PM) — FULL"
        : `Morning (8:00 AM – 12:00 PM) — ${MORNING_MAX - getMorningCount(day)} slots left`,
      value: "morning",
      disabled: isMorningFull(day),
    },
    {
      title: isAfternoonFull(day)
        ? "Afternoon (1:00 PM – 5:00 PM) — FULL"
        : `Afternoon (1:00 PM – 5:00 PM) — ${AFTERNOON_MAX - getAfternoonCount(day)} slots left`,
      value: "afternoon",
      disabled: isAfternoonFull(day),
    },
  ];
});

const rules = { required: (v: string) => !!v || "This field is required" };

const internalDialog = computed({
  get: () => props.modelValue,
  set: (v) => emit("update:modelValue", v),
});

const selectedDateFormatted = computed(() => {
  if (!formData.value.scheduled_date) return "";
  const d = new Date(formData.value.scheduled_date + "T00:00:00");
  return d.toLocaleDateString("en-PH", {
    weekday: "long", year: "numeric", month: "long", day: "numeric",
  });
});

// ── Methods ───────────────────────────────────────────────────────────────
const closeDialog = () => { resetForm(); internalDialog.value = false; };

const resetForm = () => {
  formData.value = {
    purok: "", address: "", garbage_type: "", notes: "",
    is_hazardous: false, scheduled_date: "", time_slot: "",
  };
  submitError.value = "";
  formRef.value?.reset();
};

const loadSlots = async () => {
  loadingSlots.value = true;
  try {
    slotCounts.value = await collectionsStore.getSlotCountsByDate();
  } catch {
    slotCounts.value = {};
  }
  loadingSlots.value = false;
};

const submitRequest = async () => {
  submitError.value = "";

  // ── 1. Manual field checks (avoid optional-chain crash on validate()) ──
  if (!formData.value.purok) {
    submitError.value = "Please select a Purok.";
    return;
  }
  if (!formData.value.address?.trim()) {
    submitError.value = "Please enter a Pickup Address.";
    return;
  }
  if (!formData.value.garbage_type) {
    submitError.value = "Please select a Garbage Type.";
    return;
  }
  if (!formData.value.scheduled_date) {
    submitError.value = "Please select a pickup date on the calendar.";
    return;
  }
  if (!formData.value.time_slot) {
    submitError.value = "Please select a time slot (Morning or Afternoon).";
    return;
  }

  loading.value = true;
  try {
    // ── 2. Get authenticated user ──────────────────────────────────────
    const userResult = await authStore.getCurrentUser();
    if (userResult.error || !userResult.user) {
      submitError.value = "You are not logged in. Please refresh and try again.";
      return;
    }

    // ── 3. Build payload ───────────────────────────────────────────────
    const collectionData: CreateCollectionData = {
      address:          formData.value.address.trim(),
      purok:            formData.value.purok,
      request_by:       userResult.user.id,
      collector_assign: null,
      status:           "pending",
      garbage_type:     formData.value.garbage_type,
      notes:            formData.value.notes || undefined,
      is_hazardous:     formData.value.is_hazardous,
      scheduled_date:   formData.value.scheduled_date,
      time_slot:        formData.value.time_slot,
    };

    // ── 4. Submit ──────────────────────────────────────────────────────
    const result = await collectionsStore.createCollection(collectionData);

    if (result) {
      // Check if schedule was actually saved (columns may not exist yet)
      if (!result.scheduled_date && formData.value.scheduled_date) {
        toast.warning(
          "Request submitted, but schedule was NOT saved. " +
          "Add 'scheduled_date' (date) and 'time_slot' (text) columns to your Supabase 'collections' table."
        );
      } else {
        toast.success("Collection request submitted successfully!");
      }
      emit("collection-created", result);
      closeDialog();
    } else {
      submitError.value = collectionsStore.error || "Submission failed. Please try again.";
    }
  } catch (err: any) {
    console.error("submitRequest error:", err);
    submitError.value = err?.message || "An unexpected error occurred.";
  } finally {
    loading.value = false;
  }
};

watch(() => props.modelValue, (newVal) => {
  if (!newVal) {
    resetForm();
  } else {
    loadSlots();
    if (props.initialData) formData.value.purok = props.initialData.purok || "";
  }
});
</script>

<template>
  <v-dialog v-model="internalDialog" max-width="640px" persistent scrollable>
    <v-card>
      <v-card-title class="text-h5 pa-4 bg-primary">
        <v-icon class="mr-2" color="white">mdi-recycle</v-icon>
        <span class="text-white">Request Collection</span>
      </v-card-title>

      <v-divider />

      <v-card-text class="pa-6">
        <!-- Error banner -->
        <v-alert
          v-if="submitError"
          type="error"
          variant="tonal"
          density="compact"
          class="mb-4"
          closable
          @click:close="submitError = ''"
        >
          {{ submitError }}
        </v-alert>

        <v-form ref="formRef">
          <v-row>
            <!-- Purok -->
            <v-col cols="12">
              <v-select
                v-model="formData.purok"
                label="Purok *"
                :items="PUROK_OPTIONS"
                placeholder="Select Purok"
                outlined dense
              />
            </v-col>

            <!-- Address -->
            <v-col cols="12">
              <v-textarea
                v-model="formData.address"
                label="Pickup Address *"
                placeholder="Enter the complete address for garbage collection"
                rows="3" outlined dense
              />
            </v-col>

            <!-- Garbage Type -->
            <v-col cols="12">
              <v-select
                v-model="formData.garbage_type"
                label="Garbage Type *"
                :items="garbageTypesWithIcons"
                item-title="text"
                item-value="value"
                outlined dense
              >
                <template v-slot:selection="{ item }">
                  <v-icon :icon="item.raw.icon" size="small" class="mr-2" />
                  {{ item.raw.text }}
                </template>
                <template v-slot:item="{ props, item }">
                  <v-list-item v-bind="props" :prepend-icon="item.raw.icon" />
                </template>
              </v-select>
            </v-col>

            <!-- Hazardous -->
            <v-col cols="12" class="py-0">
              <v-checkbox
                v-model="formData.is_hazardous"
                label="Is this item hazardous? (e.g., leaking batteries, broken screens)"
                color="error" hide-details density="compact"
              />
            </v-col>

            <!-- Notes -->
            <v-col cols="12">
              <v-textarea
                v-model="formData.notes"
                label="Additional Notes (Optional)"
                placeholder="Any special instructions or details"
                rows="2" outlined dense
              />
            </v-col>

            <!-- ══ DATE SCHEDULING ═══════════════════════════════════════ -->
            <v-col cols="12">
              <v-divider class="mb-4" />
              <div class="d-flex align-center mb-2">
                <v-icon color="primary" class="mr-2">mdi-calendar-clock</v-icon>
                <span class="text-subtitle-1 font-weight-bold">Schedule Pickup Date *</span>
              </div>
              <p class="text-caption text-medium-emphasis mb-3">
                Choose a weekday. 50 morning + 50 afternoon slots per day.
                Weekends and past dates cannot be selected.
              </p>

              <!-- Legend -->
              <div class="d-flex flex-wrap ga-2 mb-3">
                <div class="d-flex align-center ga-1">
                  <div class="leg leg--available" /><span class="text-caption">Available</span>
                </div>
                <div class="d-flex align-center ga-1">
                  <div class="leg leg--selected" /><span class="text-caption">Selected</span>
                </div>
                <div class="d-flex align-center ga-1">
                  <div class="leg leg--full" /><span class="text-caption">Fully booked</span>
                </div>
                <div class="d-flex align-center ga-1">
                  <div class="leg leg--past" /><span class="text-caption">Past</span>
                </div>
                <div class="d-flex align-center ga-1">
                  <span class="sdot sdot--ok" /><span class="text-caption ml-1">Slot open</span>
                </div>
                <div class="d-flex align-center ga-1">
                  <span class="sdot sdot--full" /><span class="text-caption ml-1">Slot full</span>
                </div>
              </div>

              <!-- Loading -->
              <div v-if="loadingSlots" class="d-flex justify-center pa-4">
                <v-progress-circular indeterminate color="primary" size="28" />
              </div>

              <!-- Calendar -->
              <v-card v-else variant="outlined" rounded="lg" class="calendar-card">
                <div class="cal-header d-flex align-center justify-space-between px-3 py-2">
                  <v-btn icon="mdi-chevron-left" variant="text" density="compact" @click="prevMonth" />
                  <span class="font-weight-bold text-body-1">{{ calTitle }}</span>
                  <v-btn icon="mdi-chevron-right" variant="text" density="compact" @click="nextMonth" />
                </div>
                <div class="cal-grid cal-grid--hdr px-2 pb-1">
                  <div v-for="h in ['Mon','Tue','Wed','Thu','Fri']" :key="h"
                    class="cal-cell cal-cell--hdr">{{ h }}</div>
                </div>
                <div class="cal-grid px-2 pb-2">
                  <div
                    v-for="(day, idx) in calDays" :key="idx"
                    class="cal-cell" :class="getDayClass(day)"
                    @click="selectDay(day)"
                  >
                    <template v-if="day">
                      <span class="cal-num">{{ day }}</span>
                      <div class="cal-dots">
                        <span class="sdot" :class="isMorningFull(day) ? 'sdot--full' : 'sdot--ok'" title="Morning" />
                        <span class="sdot" :class="isAfternoonFull(day) ? 'sdot--full' : 'sdot--ok'" title="Afternoon" />
                      </div>
                    </template>
                  </div>
                </div>
              </v-card>

              <!-- Selected date display -->
              <v-alert
                v-if="formData.scheduled_date"
                type="info" variant="tonal" density="compact"
                class="mt-3" icon="mdi-calendar-check"
              >
                <strong>Selected:</strong> {{ selectedDateFormatted }}
              </v-alert>

              <!-- Time Slot -->
              <div v-if="formData.scheduled_date" class="mt-3">
                <v-select
                  v-model="formData.time_slot"
                  label="Time Slot *"
                  :items="availableTimeSlots"
                  item-title="title"
                  item-value="value"
                  item-disabled="disabled"
                  outlined dense
                  prepend-inner-icon="mdi-clock-outline"
                >
                  <template v-slot:item="{ props, item }">
                    <v-list-item v-bind="props" :disabled="item.raw.disabled" />
                  </template>
                </v-select>
              </div>
            </v-col>
            <!-- ══ END DATE SCHEDULING ═══════════════════════════════════ -->
          </v-row>
        </v-form>
      </v-card-text>

      <v-divider />

      <v-card-actions class="pa-4">
        <v-spacer />
        <v-btn color="grey" variant="text" @click="closeDialog" :disabled="loading">
          Cancel
        </v-btn>
        <v-btn color="primary" variant="flat" @click="submitRequest" :loading="loading">
          Submit Request
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<style scoped lang="scss">
.v-card-title { font-weight: 600; }
.v-card-text  { max-height: 580px; }
.v-row        { margin-top: 0; }

.leg {
  width: 12px; height: 12px; border-radius: 50%; display: inline-block;
  &--available { background: #4caf50; }
  &--selected  { background: #1976d2; }
  &--full      { background: #f44336; }
  &--past      { background: #bdbdbd; }
}
.sdot {
  display: inline-block; width: 6px; height: 6px; border-radius: 50%;
  &--ok   { background: #4caf50; }
  &--full { background: #f44336; }
}

.calendar-card { overflow: hidden; }
.cal-header    { background: rgba(var(--v-theme-primary), 0.07); }

.cal-grid {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 4px;
  &--hdr .cal-cell--hdr {
    font-size: 11px; font-weight: 700; text-align: center;
    color: rgba(0,0,0,.45); text-transform: uppercase;
  }
}

.cal-cell {
  aspect-ratio: 1; min-height: 42px;
  display: flex; flex-direction: column; align-items: center; justify-content: center;
  border-radius: 8px; cursor: pointer; user-select: none;
  transition: background .14s, transform .1s; padding: 2px;
  &--empty { cursor: default; background: transparent; }
}

.cal-day {
  &--past      { cursor: not-allowed; opacity: .35; background: #f5f5f5; }
  &--full      { cursor: not-allowed; background: #ffebee; }
  &--available {
    background: #e8f5e9;
    &:hover { background: #c8e6c9; transform: scale(1.07); }
  }
  &--selected  {
    background: rgb(var(--v-theme-primary)) !important; transform: scale(1.07);
    .cal-num { color: #fff !important; }
  }
}

.cal-num  { font-size: 13px; font-weight: 700; line-height: 1; }
.cal-dots { display: flex; gap: 3px; margin-top: 3px; }
</style>