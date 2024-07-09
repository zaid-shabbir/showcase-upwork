<template>
  <div>
    <expansion-panel
      :title="$t('encounters.lab_orders')"
      icon="laboratory-drug-file"
      v-model="panel"
    >
      <template #content>
        <v-sheet class="px-3 py-5 bordered">
          <div class="d-flex flex-wrap mb-2">
            <v-chip-group
              v-model="selectedTestProfile"
              v-for="testProfile in testProfiles"
              :key="testProfile.key"
              class="text-primary"
              selected-class="bg-primary text-white"
            >
              <v-chip variant="outlined" size="large" :value="testProfile.key">
                {{ testProfile.short_name }}
                <tooltip> {{ testProfile.name }} </tooltip>
              </v-chip>
            </v-chip-group>
          </div>

          <auto-complete
            :placeholder="$t('encounters.lab_orders_plcaeholder')"
            :items="filteredLabTests"
            v-model="selectedLabTests"
            :loading="loading"
            multiple
            chips
            item-title="lab_test.name"
            item-value="lab_test.id"
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
                @click="labTestClicked(item.value)"
                :active="selectedLabTests.includes(item.value)"
              >
                <span class="d-flex align-center">
                  <checkbox
                    v-model="selectedLabTests"
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
                :text="getSelectedLabTestName(item.value)"
              ></v-chip>
            </template>
          </auto-complete>

          <text-area
            :label="$t('encounters.note')"
            v-model="labOrderNotes"
            class="mt-3"
          />

          <div class="text-right mt-2">
            <btn
              :primary="false"
              secondary
              :disabled="!selectedLabTests.length"
              @click="addLabTest"
              :loading="loading"
            >
              {{ $t('common.save') }}
            </btn>
          </div>

          <div v-if="currentEncounter.lab_orders.length" class="text-h6 bold">
            {{ $t('encounters.lab_orders') }}
          </div>
          <v-card
            v-for="(labOrder, index) in currentEncounter.lab_orders"
            :key="index"
            class="my-3 bg-water"
          >
            <v-card-title class="pa-4 pb-4 bg-primary">
              <span class="text-h6 bold">{{ labOrder.order_number }}</span>
              <btn
                size="x-small"
                icon-only
                :primary="false"
                variant="flat"
                icon="$delete"
                color="white"
                class="float-right mt-n1"
                :loading="loading"
                @click="deleteLabTest(labOrder)"
              />
              <btn
                size="x-small"
                icon="$print"
                icon-only
                variant="flat"
                class="float-right mt-n1 mx-1"
                :primary="false"
                color="white"
                @click="printLabOrder(labOrder)"
              />
            </v-card-title>
            <div class="pa-4">
              <span
                v-for="(labOrderItem, index) in labOrder.lab_order_items"
                :key="index"
              >
                • {{ labOrderItem.lab_test.specimen_type }} :
                {{ labOrderItem.lab_test && labOrderItem.lab_test.name }}
                <v-divider
                  class="my-3"
                  v-if="index !== labOrder.lab_order_items.length - 1"
                />
              </span>
              <span v-if="labOrder.notes" class="text-body-1 font-italic">
                <v-divider class="my-2" />
                {{ labOrder.notes }}
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
  watch,
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
  ILabOrderPayload,
  IEncounterLabOrder,
  IKeyNames,
  ILabTestProfileItem,
} from '@/types';

const authStore = useAuthStore();
const patientStore = usePatientStore();
const facilityStore = useFacilityStore();
const metaDataStore = useMetaDataStore();
const encounterStore = useEncounterStore();

const emit = defineEmits(['updated', 'print']);

const loading = ref<boolean>(false);
const labOrderNotes = ref<string>('');
const testProfiles = ref<IKeyNames[]>([]);
const panel = ref<number | undefined>();
const selectedTestProfile = ref<number | null>(null);
const selectedLabTests = ref<number[]>([]);
const labTestsResult = ref<ILabTestProfileItem[]>([]);

const userInfo: ComputedRef<IUserInfo> = computed(() => {
  return authStore.userInfo;
});

const currentPatientId: ComputedRef<string | null> = computed(() => {
  return patientStore.currentPatientId;
});

const currentFacilityId: ComputedRef<string | null> = computed(() => {
  return facilityStore.currentFacilityId;
});

const currentEncounter: WritableComputedRef<IEncounter> = computed(() => {
  return encounterStore.currentEncounter;
});

const filteredLabTests: ComputedRef<any[]> = computed(() => {
  let labTests: any = [];

  const groupedLabTests = lodash.groupBy(labTestsResult.value, 'profile_name');
  const keys = Object.keys(groupedLabTests);

  keys.forEach((key, index) => {
    labTests.push({ header: key });
    labTests = [...labTests, ...groupedLabTests[key]];
    if (index !== keys.length - 1) labTests.push({ divider: true });
  });

  return labTests;
});

onMounted(() => {
  getMetaDataItems();
});

watch(selectedTestProfile, () => {
  handleProfileClick();
});

const getMetaDataItems = () => {
  metaDataStore.getMetaData<void, IKeyNames>('lab_test_profile').then((res) => {
    testProfiles.value = res;
  });
};

const getSelectedLabTestName = (labTestId: number) => {
  const labTest = labTestsResult.value.find(
    (item) => item?.lab_test.id === labTestId,
  );
  return labTest?.lab_test.name_short || labTestId;
};

const handleProfileClick = () => {
  selectedLabTests.value = [];
  labTestsResult.value = [];
  if (!selectedTestProfile.value) return;
  encounterStore
    .getlabTestProfileItems(selectedTestProfile.value)
    .then((res) => {
      labTestsResult.value = res;
      selectedLabTests.value = labTestsResult.value.map(
        (test) => test.lab_test.id,
      );
    });
};

const labTestClicked = (labTest: number) => {
  if (selectedLabTests.value.includes(labTest))
    selectedLabTests.value = selectedLabTests.value.filter(
      (item: number) => item !== labTest,
    );
  else selectedLabTests.value.push(labTest);
};

const addLabTest = () => {
  const labTests = selectedLabTests.value;
  if (!labTests.length) return;
  const payload: ILabOrderPayload = {
    practitioner: userInfo.value.practitioner_id!,
    encounter: currentEncounter.value.id,
    patient: currentPatientId.value as any,
    facility: currentFacilityId.value as any,
    notes: labOrderNotes.value,
    lab_test_profile: selectedTestProfile.value as number,
    lab_order_items: labTests.map((labTest: any) => {
      return {
        lab_test: labTest,
      };
    }),
  };

  loading.value = true;
  encounterStore
    .addLabOrder(payload)
    .then(() => {
      selectedLabTests.value = [];
      selectedTestProfile.value = null;
      labOrderNotes.value = '';
      emit('updated');
    })
    .finally(() => (loading.value = false));
};

const deleteLabTest = (payload: IEncounterLabOrder) => {
  loading.value = true;
  encounterStore
    .deleteLabOrder(payload)
    .then(() => {
      selectedLabTests.value = [];
      selectedTestProfile.value = null;
      labOrderNotes.value = '';
      emit('updated');
    })
    .finally(() => (loading.value = false));
};

const printLabOrder = (payload: IEncounterLabOrder) => {
  emit('print', payload);
};
</script>

<style scoped></style>
