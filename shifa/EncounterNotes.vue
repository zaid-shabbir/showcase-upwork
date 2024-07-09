<template>
  <expansion-panel
    :title="$t('encounters.notes')"
    icon="medical-notes"
    v-model="panel"
  >
    <template #content>
      <v-sheet class="px-3 py-5 bordered">
        <Editor
          v-model="notes"
          :placeholder="$t('encounters.notes_plcaeholder')"
        />
        <div class="text-right mt-2">
          <btn
            :primary="false"
            secondary
            @click="updateEncounterNotes"
            :loading="loading"
          >
            {{ $t('common.save') }}
          </btn>
        </div>
      </v-sheet>
    </template>
  </expansion-panel>
</template>

<script setup lang="ts">
import { ref, watch, computed, WritableComputedRef } from 'vue';
import { useEncounterStore } from '@/store';
import { IEncounter } from '@/types';

const encounterStore = useEncounterStore();

const emit = defineEmits(['updated']);

const loading = ref<boolean>(false);
const notes = ref<string>('');
const panel = ref<number | undefined>();

const currentEncounter: WritableComputedRef<IEncounter> = computed(() => {
  return encounterStore.currentEncounter;
});

watch(currentEncounter, () => {
  notes.value = currentEncounter.value.notes;
});

const updateEncounterNotes = () => {
  loading.value = true;
  encounterStore
    .updatePatientEncounter(currentEncounter.value.id, { notes: notes.value })
    .then(() => {
      emit('updated');
    })
    .finally(() => (loading.value = false));
};
</script>

<style scoped></style>
