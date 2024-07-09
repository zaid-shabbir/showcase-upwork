<template>
  <v-card width="320" class="px-4 py-5" :class="{ 'colored-border': isActive }">
    <v-row no-gutters>
      <v-col cols="6">
        <div class="text-body-1 text-disabled">
          {{ $t('dashboard.encounter_date') }}
        </div>
        <div class="text-h6 text-secondary font-weight-medium">
          {{ encounterDate }}
        </div>
      </v-col>
      <v-col cols="6">
        <div class="text-body-1 text-disabled">
          {{ $t('dashboard.type') }}
        </div>
        <div class="text-h6 text-secondary font-weight-medium">
          {{ encounter.encounter_type && encounter.encounter_type.name }}
        </div>
      </v-col>
      <v-col cols="6" class="mt-2">
        <div class="text-body-1 text-disabled">
          {{ $t('encounters.encounter_priority') }}
        </div>
        <div class="text-h6 text-secondary font-weight-medium">
          {{
            encounter.encounter_priority && encounter.encounter_priority.name
          }}
        </div>
      </v-col>
      <v-col cols="6" class="mt-2">
        <div class="text-body-1 text-disabled">
          {{ $t('encounters.encounter_status') }}
        </div>
        <div class="text-h6 text-secondary font-weight-medium">
          {{ encounter.encounter_status && encounter.encounter_status.name }}
        </div>
      </v-col>
    </v-row>
  </v-card>
</template>

<script setup lang="ts">
import moment from 'moment-timezone';
import { computed, PropType, ComputedRef } from 'vue';

const props = defineProps({
  encounter: {
    type: Object as PropType<any>,
    default: () => {},
  },
  isActive: {
    type: Boolean,
    default: false,
  },
});

const encounterDate: ComputedRef<any> = computed(() => {
  return moment(props.encounter.encounter_date).format('YYYY-MM-DD');
});
</script>

<style scoped></style>
