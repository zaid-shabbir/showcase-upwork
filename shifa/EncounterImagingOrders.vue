<template>
  <div>
    <expansion-panel
      :title="$t('encounters.imaging_orders')"
      icon="orders"
      v-model="panel"
    >
      <template #content>
        <v-sheet class="px-3 py-5 bordered">
          <div class="d-flex flex-wrap mb-2">
            <v-chip-group
              v-model="selectedModality"
              v-for="(modality, index) in modalities"
              :key="index"
              class="text-primary"
              selected-class="bg-primary text-white"
              @change="selectedScans = []"
            >
              <v-chip variant="outlined" size="large" :value="modality">
                {{ modality }}
              </v-chip>
            </v-chip-group>
          </div>

          <auto-complete
            :placeholder="$t('encounters.imaging_orders_plcaeholder')"
            :items="filteredScans"
            v-model="selectedScans"
            :loading="loading"
            multiple
            chips
            item-title="name"
            item-value="id"
            closable-chips
            class="text-primary"
          >
            <template #item="{ item }">
              <v-list-item-title
                class="text-h5 font-weight-bold ml-6 my-2"
                v-if="item.raw.header"
              >
                {{ item.raw.header }}
              </v-list-item-title>
              <v-divider v-else-if="item.raw.divider" />

              <v-list-item
                v-else
                class="py-0"
                @click="scanClicked(item.value)"
                :active="selectedScans.includes(item.value)"
              >
                <span class="d-flex align-center">
                  <checkbox
                    v-model="selectedScans"
                    :value="item.value"
                    style="display: contents"
                    @click.stop
                  />
                  <span>
                    {{ item.title }}
                  </span>
                </span>
              </v-list-item>
            </template>

            <template v-slot:chip="{ props, item }">
              <v-chip
                v-bind="props"
                :text="getSelectedScanTypeName(item.value)"
              ></v-chip>
            </template>
          </auto-complete>

          <!-- <text-area
            :label="$t('encounters.note')"
            v-model="imagingNotes"
            class="mt-3"
          /> -->

          <div class="text-right mt-2">
            <btn
              :primary="false"
              secondary
              :disabled="!selectedScans"
              @click="addImagingOrder"
              :loading="loading"
            >
              {{ $t('common.save') }}
            </btn>
          </div>

          <div
            v-if="currentEncounter.imaging_orders.length"
            class="text-h6 bold"
          >
            {{ $t('encounters.imaging_orders') }}
          </div>
          <v-card
            v-for="(imagingOrder, index) in currentEncounter.imaging_orders"
            :key="index"
            class="my-4 bg-water"
          >
            <v-card-title class="pa-4 pb-4 bg-primary">
              <span class="text-h6 bold">{{ imagingOrder.order_number }}</span>
              <btn
                size="x-small"
                icon-only
                :primary="false"
                variant="flat"
                icon="$delete"
                color="white"
                class="float-right mt-n1"
                :loading="loading"
                @click="deleteImagingOrder(imagingOrder)"
              />
              <btn
                size="x-small"
                icon="$print"
                icon-only
                variant="flat"
                class="float-right mt-n1 mx-1"
                :primary="false"
                color="white"
                @click="printImagingOrder(imagingOrder)"
              />
            </v-card-title>
            <div class="pa-4">
              <span v-for="(scan, index) in imagingOrder.scans" :key="index">
                • {{ scan.scan_type.modality }} :
                {{ scan.scan_type && scan.scan_type.name }}
                <v-divider
                  class="my-3"
                  v-if="index !== imagingOrder.scans.length - 1"
                />
                <span v-if="scan.notes" class="text-body-1 font-italic">
                  <v-divider class="my-2" />
                  {{ scan.notes }}
                </span>
              </span>
            </div>
          </v-card>
        </v-sheet>
      </template>
    </expansion-panel>
  </div>
</template>

<script setup lang="ts">
import {
  ref,
  computed,
  onMounted,
  ComputedRef,
  WritableComputedRef,
} from 'vue';
import lodash from 'lodash';

import {
  useAuthStore,
  usePatientStore,
  useFacilityStore,
  useMetaDataStore,
  useEncounterStore,
} from '@/store';

import {
  IUserInfo,
  IEncounter,
  IImagingOrderPayload,
  IScanType,
  IEncounterImagingOrder,
} from '@/types';

const authStore = useAuthStore();
const patientStore = usePatientStore();
const facilityStore = useFacilityStore();
const metaDataStore = useMetaDataStore();
const encounterStore = useEncounterStore();

const emit = defineEmits(['updated', 'print']);

const loading = ref<boolean>(false);
const imagingNotes = ref<string>('');
const metaDataItems = ref<IScanType[]>([]);
const panel = ref<number | undefined>();
const selectedModality = ref<string | null>(null);
const selectedScans = ref<number[]>([]);

const userInfo: ComputedRef<IUserInfo> = computed(() => {
  return authStore.userInfo;
});

const currentPatientId: ComputedRef<string | null> = computed(() => {
  return patientStore.currentPatientId;
});

const currentFcilityId: ComputedRef<string | null> = computed(() => {
  return facilityStore.currentFacilityId;
});

const currentEncounter: WritableComputedRef<IEncounter> = computed(() => {
  return encounterStore.currentEncounter;
});

const modalities: ComputedRef<string[]> = computed(() => {
  return [...new Set(metaDataItems.value.map((item) => item.modality))];
});

const getSelectedScanTypeName = (scanId: number) => {
  const scanType = metaDataItems.value.find((item) => item.id === scanId);
  return scanType?.name || scanId;
};

const filteredScans: ComputedRef<any[]> = computed(() => {
  let scans: any = [];
  if (!selectedModality.value) return scans;

  const temp = metaDataItems.value.filter(
    (item) => item.modality == selectedModality.value,
  );

  const groupedScans = lodash.groupBy(temp, 'body_part');
  const keys = Object.keys(groupedScans);

  keys.forEach((key, index) => {
    scans.push({ header: key });
    scans = [...scans, ...groupedScans[key]];
    if (index !== keys.length - 1) scans.push({ divider: true });
  });

  return scans;
});

onMounted(() => {
  getMetaDataItems();
});

const getMetaDataItems = () => {
  metaDataStore.getMetaData('scan_type').then((res) => {
    metaDataItems.value = res as any;
  });
};

const scanClicked = (scan: number) => {
  if (selectedScans.value.includes(scan))
    selectedScans.value = selectedScans.value.filter(
      (item: number) => item !== scan,
    );
  else selectedScans.value.push(scan);
};

const addImagingOrder = () => {
  const scans = selectedScans.value;
  if (!scans.length) return;
  const payload: IImagingOrderPayload = {
    practitioner: userInfo.value.practitioner_id!,
    encounter: currentEncounter.value.id,
    patient: currentPatientId.value as any,
    facility: currentFcilityId.value as any,
    scans: scans.map((scan: any) => {
      return {
        scan_type: scan,
        notes: imagingNotes.value,
      };
    }),
  };

  loading.value = true;
  encounterStore
    .addImagingOrder(payload)
    .then(() => {
      selectedScans.value = [];
      selectedModality.value = null;
      imagingNotes.value = '';
      emit('updated');
    })
    .finally(() => (loading.value = false));
};

const deleteImagingOrder = (payload: IEncounterImagingOrder) => {
  loading.value = true;
  encounterStore
    .deleteImagingOrder(payload)
    .then(() => {
      emit('updated');
    })
    .finally(() => (loading.value = false));
};

const printImagingOrder = (payload: IEncounterImagingOrder) => {
  emit('print', payload);
};
</script>

<style scoped></style>
