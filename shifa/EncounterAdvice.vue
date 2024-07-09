<template>
  <expansion-panel
    :title="$t('encounters.advice')"
    icon="medical-personnel-doctor-1"
    v-model="panel"
  >
    <template #content>
      <v-sheet class="px-3 py-5 bordered">
        <combo-box
          :placeholder="$t('encounters.advice_plcaeholder')"
          :items="metaDataItems"
          v-model="selectedItem"
          append-icon="mdi-plus"
          :loading="loading"
          @click:append="createEncounterItem"
          @keyup.enter="createEncounterItem"
        >
        </combo-box>

        <div class="ml-5 mt-3">
          <div
            v-for="(item, index) in currentEncounter.encounter_advice"
            :key="index"
            class="d-flex justify-space-between py-1"
          >
            <span class="text-subtitle-1"> • {{ getItemText(item) }} </span>
            <btn
              size="x-small"
              icon-only
              :primary="false"
              variant="text"
              icon="$delete"
              :loading="loading"
              @click="deleteEncounterItem(item)"
            >
            </btn>
          </div>
        </div>
      </v-sheet>
    </template>
  </expansion-panel>
</template>

<script setup lang="ts">
import { ref, computed, WritableComputedRef, onMounted } from 'vue';
import { useMetaDataStore, useEncounterStore } from '@/store';
import { IEncounter, IEncounterItem, IMetaDataICDCode } from '@/types';

const metaDataStore = useMetaDataStore();
const encounterStore = useEncounterStore();

const emit = defineEmits(['updated']);

const loading = ref<boolean>(false);
const metaDataItems = ref<string[]>([]);
const panel = ref<number | undefined>();
const selectedItem = ref<string>();

const currentEncounter: WritableComputedRef<IEncounter> = computed(() => {
  return encounterStore.currentEncounter;
});

onMounted(() => {
  loadMetaData();
});

const loadMetaData = async () => {
  const res = await metaDataStore.getMetaData('advice', {});
  metaDataItems.value = (res || []).map((item: any) => item.advice);
};

const getItemText = (item: any) => {
  return item.advice;
};

const createEncounterItem = () => {
  if (!currentEncounter.value.id || !selectedItem.value) return;

  loading.value = true;
  const payload: IEncounterItem = {
    encounter: currentEncounter.value.id,
    advice: selectedItem.value,
  };
  encounterStore
    .addEncounterItem(payload, 'advice')
    .then(() => {
      selectedItem.value = '';
      loadMetaData();
      emit('updated');
    })
    .finally(() => (loading.value = false));
};

const deleteEncounterItem = (encounterItem: IEncounterItem) => {
  loading.value = true;
  encounterStore
    .deleteEncounterItem(encounterItem, 'advice')
    .then(() => {
      emit('updated');
    })
    .finally(() => (loading.value = false));
};
</script>

<style scoped lang="scss"></style>
