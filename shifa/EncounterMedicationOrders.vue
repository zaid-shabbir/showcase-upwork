<template>
  <div>
    <expansion-panel
      :title="$t('encounters.medication_orders')"
      icon="orders"
      v-model="panel"
    >
      <template #content>
        <v-sheet class="px-3 py-5 bordered">
          <auto-complete
            :placeholder="$t('encounters.medication_orders_placeholder')"
            :items="customEncounters"
            v-model="selectedItem"
            :loading="loading"
            item-title="name"
            item-value="id"
            v-model:search="searchKey"
            @update:search="searchUpdated"
            @keyup.enter="addMedicationOrder"
          >
            <template #append>
              <v-tooltip :text="$t('common.my_favorite')" location="top">
                <template v-slot:activator="{ props }">
                  <switch-btn
                    v-bind="props"
                    :indeterminate="false"
                    v-model="isMyFavorite"
                    class="mr-2 switch"
                  />
                </template>
              </v-tooltip>

              <btn
                icon="mdi-plus"
                icon-only
                :primary="false"
                size="small"
                variant="text"
                color="primary"
                @click="addMedicationOrder"
              />
            </template>
          </auto-complete>

          <!-- <text-area
            :label="$t('encounters.note')"
            v-model="medicationOrderNotes"
            class="mt-3"
          />

          <div class="text-right mt-2">
            <btn
              :primary="false"
              secondary
              :disabled="!selectedItem"
              @click="addMedicationOrder"
              :loading="loading"
            >
              {{ $t('common.save') }}
            </btn>
          </div> -->

          <div
            v-if="currentEncounter.medication_orders.length"
            class="mt-3 text-h6 bold"
          >
            {{ $t('encounters.medication_orders') }}
          </div>
          <v-card
            v-for="(
              medicationOrder, index
            ) in currentEncounter.medication_orders"
            :key="index"
            class="my-4 bg-water"
          >
            <v-card-title class="pa-4 pb-4 bg-primary">
              <span class="text-h6 bold">
                {{ medicationOrder.order_number }}
              </span>
              <btn
                size="x-small"
                icon-only
                :primary="false"
                variant="flat"
                icon="$delete"
                color="white"
                class="float-right mt-n1"
                :loading="loading"
                @click="deleteMedicationOrder(medicationOrder)"
              />
              <btn
                size="x-small"
                icon="$print"
                icon-only
                variant="flat"
                class="float-right mt-n1 mx-1"
                :primary="false"
                color="white"
                @click="printMedicationOrder(medicationOrder)"
              />
            </v-card-title>
            <div class="pa-4">
              <span
                v-for="(
                  medicationOrderItem, index
                ) in medicationOrder.medication_order_items"
                :key="index"
              >
                • {{ medicationOrderItem.medication.IQ_national_code }} :
                {{ medicationOrderItem.medication.name }}
                <v-divider
                  class="my-3"
                  v-if="
                    index !== medicationOrder.medication_order_items.length - 1
                  "
                />
              </span>
              <span
                v-if="medicationOrder.notes"
                class="text-body-1 font-italic"
              >
                <v-divider class="my-2" />
                {{ medicationOrder.notes }}
              </span>
            </div>
          </v-card>
        </v-sheet>
      </template>
      <template #actions>
        <slot name="actions"></slot>
      </template>
    </expansion-panel>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue';
import _ from 'lodash';

import {
  useAuthStore,
  usePatientStore,
  useFacilityStore,
  useEncounterStore,
  useCustomEncountersStore,
} from '@/store';

import {
  ICustomEncounter,
  CustomEncounterType,
  IMedicationOrderPayload,
  IFilterQuery,
  IEncounterMedicationOrder,
} from '@/types';

const authStore = useAuthStore();
const patientStore = usePatientStore();
const facilityStore = useFacilityStore();
const encounterStore = useEncounterStore();
const customEncountersStore = useCustomEncountersStore();

const emit = defineEmits(['updated', 'print']);

const loading = ref<boolean>(false);
const medicationOrderNotes = ref<string>('');
const customEncounters = ref<ICustomEncounter[]>([]);
const panel = ref<number | undefined>();
const selectedItem = ref<number>();
const searchKey = ref<string>('');
const isMyFavorite = ref(true);

const currentEncounter = computed(() => {
  return encounterStore.currentEncounter;
});

const searchItems = _.debounce(async () => {
  const query: IFilterQuery = {
    page: 1,
    name: searchKey.value,
    favored: isMyFavorite.value ? true : undefined,
  };

  const res = await customEncountersStore.searchCustomEncounters(
    CustomEncounterType.Medication,
    query,
  );
  customEncounters.value = res.results || [];
}, 600);

const searchUpdated = async (query: string) => {
  // vuetify bug, triggers update:search when toggling switch
  if (!query || query.length < 2 || ['true', 'false'].includes(query)) {
    return;
  }

  const currentItem = customEncounters.value.find(
    (item) => item.id === selectedItem.value,
  );
  if (query === currentItem?.name) {
    return;
  }

  searchItems();
};

const clearInput = () => {
  searchKey.value = '';
  customEncounters.value = [];
  selectedItem.value = undefined;
};

const addMedicationOrder = () => {
  if (!currentEncounter.value.id || !selectedItem.value) return;

  const payload: IMedicationOrderPayload = {
    practitioner: authStore.userInfo.practitioner_id!,
    encounter: currentEncounter.value.id,
    patient: patientStore.currentPatientId!,
    medication_order_items: [{ medication: selectedItem.value }],
  };

  loading.value = true;
  encounterStore
    .addMedicationOrder(facilityStore.currentFacilityId!, payload)
    .then(() => {
      emit('updated');
      clearInput();
    })
    .finally(() => (loading.value = false));
};

const deleteMedicationOrder = async (
  medicationOrder: IEncounterMedicationOrder,
) => {
  loading.value = true;
  encounterStore
    .deleteMedicationOrder(facilityStore.currentFacilityId!, medicationOrder.id)
    .then(() => {
      emit('updated');
    })
    .finally(() => (loading.value = false));
};

const printMedicationOrder = (medicationOrder: IEncounterMedicationOrder) => {
  emit('print', medicationOrder);
};
</script>

<style scoped lang="scss">
.switch {
  height: 44px;
  display: flex;
  align-items: center;
}
</style>
