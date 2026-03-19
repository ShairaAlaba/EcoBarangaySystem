<script setup lang="ts">
import { ref, computed } from "vue";
import { storeToRefs } from "pinia";
import { useAuthUserStore } from "@/stores/authUser";
import InnerLayoutWrapper from "@/layouts/InnerLayoutWrapper.vue";
import DailyAnnouncement from "@/pages/residents/DailyAnnouncement.vue";
import AnnouncementsWidget from "@/pages/residents/AnnouncementsWidget.vue";
import CollectionRequestWidget from "@/pages/residents/CollectionRequestWidget.vue";
import MyCollectionsWidget from "@/pages/residents/MyCollectionsWidget.vue";
import MyScheduleWidget from "@/pages/residents/MyScheduleWidget.vue";
import CollectionSender from "@/pages/residents/CollectionSender.vue";

const authStore = useAuthUserStore();
const { userName, userRole } = storeToRefs(authStore);

const isResidentUser = computed(() => userRole.value === 3);
</script>

<template>
  <!-- Daily Announcement - Fixed Position at Top Center -->
  <DailyAnnouncement />

  <InnerLayoutWrapper>
    <template #content>
      <v-container fluid>
        <v-row justify="center">
          <v-col cols="12" xl="12">

            <!-- Collection Widgets Row - Only for residents -->
            <v-row v-if="isResidentUser" class="mb-6">
              <v-col cols="12" sm="6" md="6">
                <CollectionRequestWidget />
              </v-col>
              <v-col cols="12" sm="6" md="6">
                <MyCollectionsWidget />
              </v-col>
            </v-row>

            <!-- Announcements Widget -->
            <AnnouncementsWidget />
            <v-divider class="my-4"></v-divider>

            <!-- My Schedule Calendar - below announcements, compact, only for residents -->
            <v-row v-if="isResidentUser" class="mb-4">
              <v-col cols="12" sm="8" md="6" lg="5">
                <MyScheduleWidget />
              </v-col>
            </v-row>

          </v-col>
        </v-row>
      </v-container>
    </template>
  </InnerLayoutWrapper>

  <!-- Collection Sender - Fixed bottom right, only for residents -->
  <div v-if="isResidentUser" class="collection-sender-wrapper">
    <CollectionSender />
  </div>
</template>

<style scoped lang="scss">
.welcome-header {
  padding: 1rem 0;
}

.collection-sender-wrapper {
  :deep(.floating-action-button) {
    right: 24px;
    left: auto;
    bottom: 80px !important;
    border-radius: 50% !important;
    min-width: 64px !important;
    width: 64px !important;
    height: 64px !important;
    padding: 0 !important;
    z-index: 1001 !important;

    .v-btn__content {
      width: 100%;
      height: 100%;
      display: flex;
      align-items: center;
      justify-content: center;
    }
  }

  @media (max-width: 600px) {
    :deep(.floating-action-button) {
      right: 16px;
      left: auto;
      bottom: 72px !important;
      min-width: 56px !important;
      width: 56px !important;
      height: 56px !important;
    }
  }
}

@media (min-width: 600px) {
  .welcome-header {
    padding: 1.5rem 0;
  }
}
</style>