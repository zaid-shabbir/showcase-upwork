<template>
  <form-dialog
    v-model="isShowDialog"
    class="dialog-form"
    :show-footer="false"
    v-bind:="$attrs"
    width="600"
    @submit="submit"
    @opened="onOpened"
  >
    <template v-slot:header>
      <div class="text-primary">
        {{ $t(`${path}title`) }}
      </div>
    </template>
    <label for="email">
      {{ $t(`${path}email_copy`) }}
    </label>
    <text-field
      id="email"
      v-model="email"
      :label="$t(`${path}enter_email`)"
      attr="email"
      class="mt-1 mb-5"
      :rules="[validOptionalEmail]"
    />

    <checkbox
      :label="$t(`${path}encounter_notes`)"
      value="encounter-note"
      v-model="selectedOptions"
    />
    <checkbox
      :label="$t(`${path}lab_order`)"
      value="lab-order"
      v-model="selectedOptions"
    />
    <checkbox
      :label="$t(`${path}imaging_order`)"
      value="imaging-order"
      v-model="selectedOptions"
    />
    <checkbox
      :label="$t(`${path}medication_order`)"
      value="medication-order"
      v-model="selectedOptions"
    />

    <PrintDialog v-model="isShowPrintDialog" :html="printableHtml" />
  </form-dialog>
</template>

<script setup lang="ts">
import { ref, computed, WritableComputedRef, PropType } from 'vue';
import { useI18n } from 'vue-i18n';
import { IEncounter } from '@/types';
import { useEncounterStore } from '@/store';
import { validOptionalEmail } from '@/utils/validations';
import PrintDialog from '@/components/common/PrintDialog.vue';

const { t } = useI18n();
const encounterStore = useEncounterStore();

const emit = defineEmits(['update:modelValue']);

const props = defineProps({
  modelValue: {
    type: Boolean,
    default: false,
  },
  encounter: {
    type: Object as PropType<IEncounter>,
    required: true,
  },
});

const path = ref('encounters.print_dialog.');
const email = ref('');
const isShowPrintDialog = ref<boolean>(false);
const printableHtml = ref<string>('');
const selectedOptions = ref<string[]>([
  'encounter-note',
  'lab-order',
  'imaging-order',
  'medication-order',
]);

const isShowDialog: WritableComputedRef<boolean> = computed({
  get() {
    return props.modelValue;
  },
  set(value) {
    emit('update:modelValue', value);
  },
});

const onOpened = () => {
  email.value = '';
  selectedOptions.value = [
    'encounter-note',
    'lab-order',
    'imaging-order',
    'medication-order',
  ];
};

const submit = async () => {
  const res = await encounterStore.getEncounterDocumentHtml('encounter-note');
  printableHtml.value = res;
  isShowPrintDialog.value = true;
};
</script>

<style scoped></style>
