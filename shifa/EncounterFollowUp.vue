<template>
  <expansion-panel
    :title="$t('encounters.follow_up')"
    icon="calendar"
    v-model="panel"
    class="encounter-follow-up-container"
  >
    <template #content>
      <v-sheet class="px-3 py-5 bordered">
        <div class="d-flex flex-wrap">
          <v-chip-group
            v-model="selectedFollowup"
            v-for="(followUp, index) in followUps"
            :key="index"
            class="text-primary"
            selected-class="bg-primary text-white"
          >
            <v-chip variant="outlined" :value="followUp.code">
              {{
                selectedFollowup == followUp.code
                  ? followUp.code
                  : followUp.name
              }}
            </v-chip>
          </v-chip-group>
        </div>

        <date-picker
          v-if="selectedFollowup == 'custom'"
          v-model="customFollowupDate"
          :placeholder="$t('encounters.followup_date')"
          attr="followup_date"
          class="mt-2"
        />
        <text-area
          :placeholder="$t('encounters.follow_up_plcaeholder')"
          v-model="followUpNotes"
          class="mt-3"
        />
        <div class="text-right mt-2">
          <btn
            :primary="false"
            secondary
            :disabled="!selectedFollowup"
            @click="createEncounterFollowup"
            :loading="loading"
          >
            {{ $t('common.save') }}
          </btn>
        </div>

        <v-card
          v-if="currentEncounter.encounter_followup.length"
          class="pa-4 mt-4 mb-2 bg-water"
        >
          <div
            v-for="(followup, index) in currentEncounter.encounter_followup"
            :key="index"
          >
            {{ getFollowupDate(followup.followup_date) }}
            <br />
            <span class="text-body-1">
              {{ followup.followup_note }}
            </span>

            <btn
              size="x-small"
              icon-only
              :primary="false"
              variant="text"
              icon="$delete"
              color="primary"
              class="float-right mt-n7"
              :loading="loading"
              @click="deleteEncounterFollowup(followup)"
            >
            </btn>
            <v-divider
              v-if="index !== currentEncounter.encounter_followup.length - 1"
              class="my-2"
            />
          </div>
        </v-card>
      </v-sheet>
    </template>
  </expansion-panel>
</template>

<script setup lang="ts">
import {
  ref,
  onMounted,
  computed,
  ComputedRef,
  WritableComputedRef,
} from 'vue';

import { useI18n } from 'vue-i18n';
import moment from 'moment-timezone';

import { useEncounterStore } from '@/store';

import { ICodeName, IEncounter, IEncounterFollowup } from '@/types';

const encounterStore = useEncounterStore();

const emit = defineEmits(['updated']);

const { t } = useI18n();

const loading = ref<boolean>(false);
const panel = ref<number | undefined>();
const followUpNotes = ref<string>('');
const selectedFollowup = ref<string | null>(null);
const customFollowupDate = ref<Date | null>(null);

const followUps: ComputedRef<ICodeName[]> = computed(() => {
  return [
    {
      code: moment().add(7, 'day').format('YYYY-MM-DD'),
      name: t('encounters.one_week'),
    },
    {
      code: moment().add(14, 'day').format('YYYY-MM-DD'),
      name: t('encounters.two_weeks'),
    },
    {
      code: moment().add(1, 'month').format('YYYY-MM-DD'),
      name: t('encounters.one_month'),
    },
    {
      code: moment().add(3, 'month').format('YYYY-MM-DD'),
      name: t('encounters.three_months'),
    },
    {
      code: 'custom',
      name: t('encounters.custom_date'),
    },
  ];
});

const currentEncounter: WritableComputedRef<IEncounter> = computed(() => {
  return encounterStore.currentEncounter;
});

onMounted(() => {});

const getFollowupDate = (date: string) => {
  return moment(date).format('MMM DD yyy');
};

const createEncounterFollowup = () => {
  if (!currentEncounter.value.id || !selectedFollowup.value) return;

  if (selectedFollowup.value == 'custom' && !customFollowupDate.value) {
    return;
  }

  const payload: IEncounterFollowup = {
    encounter: currentEncounter.value.id,
    followup_note: followUpNotes.value,
    followup_date:
      selectedFollowup.value == 'custom'
        ? moment.utc(customFollowupDate.value).format('YYYY-MM-DD')
        : selectedFollowup.value,
  };

  loading.value = true;
  encounterStore
    .addEncounterItem(payload, 'followup')
    .then(() => {
      followUpNotes.value = '';
      selectedFollowup.value = null;
      customFollowupDate.value = null;
      emit('updated');
    })
    .finally(() => (loading.value = false));
};

const deleteEncounterFollowup = (encounterItem: IEncounterFollowup) => {
  loading.value = true;
  encounterStore
    .deleteEncounterItem(encounterItem, 'followup')
    .then(() => {
      emit('updated');
    })
    .finally(() => (loading.value = false));
};
</script>

<style scoped>
.encounter-follow-up-container {
  z-index: 1;
}
</style>
