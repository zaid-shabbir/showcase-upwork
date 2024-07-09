<template>
  <expansion-panel
    :title="customForm.name"
    icon="medical-notes-6"
    v-model="panel"
  >
    <template #content>
      <v-sheet class="px-3 py-5 bordered">
        <FormKit
          type="form"
          v-model="form.form_json_data"
          @submit="handleSubmit"
        >
          <FormKitSchema
            :schema="form.form_json_schema"
            :data="form.form_json_data"
          />
        </FormKit>

        <!-- <div class="text-right mt-2">
          <btn
            :primary="false"
            secondary
            @click="submit"
            :loading="loading"
          >
            {{ $t('common.save') }}
          </btn>
        </div> -->
      </v-sheet>
    </template>

    <template #actions>
      <slot name="actions"></slot>
    </template>
  </expansion-panel>
</template>

<script setup lang="ts">
import {
  ref,
  watch,
  computed,
  PropType,
  onMounted,
  WritableComputedRef,
} from 'vue';
import { FormKitSchema } from '@formkit/vue';

import { useEncounterStore } from '@/store';
import { IEncounter, ICustomForm, IEncounterCustomForm } from '@/types';

const encounterStore = useEncounterStore();

const props = defineProps({
  title: {
    type: String,
    default: '',
  },
  icon: {
    type: String,
    default: '',
  },
  customForm: {
    type: Object as PropType<ICustomForm>,
    required: true,
  },
});

const emit = defineEmits(['updated', 'openCustomEncounterDialog']);

const form = ref<IEncounterCustomForm>(<IEncounterCustomForm>{});
const panel = ref<number | undefined>();

const currentEncounter: WritableComputedRef<IEncounter> = computed(() => {
  return encounterStore.currentEncounter;
});

onMounted(() => {
  getCustomForm();
});

watch(currentEncounter, () => {
  getCustomForm();
});

const getCustomForm = async () => {
  if (!currentEncounter.value.id) return;
  form.value = await encounterStore.getCustomForm(props.customForm.id);
};

const handleSubmit = () => {
  encounterStore.saveCustomForm(props.customForm.id, form.value.form_json_data);
};
</script>

<style scoped lang="scss">
.switch {
  height: 44px;
  display: flex;
  align-items: center;
}
</style>
