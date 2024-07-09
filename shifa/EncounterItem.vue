<template>
  <expansion-panel :title="title" :icon="icon" v-model="panel">
    <template #content>
      <v-sheet class="px-3 py-5 bordered">
        <auto-complete
          :placeholder="placeholder"
          :items="customEncounters"
          v-model="selectedItem"
          :loading="loading"
          item-title="name"
          item-value="id"
          v-model:search="searchKey"
          @update:search="searchUpdated"
          @keyup.enter="createEncounterItem"
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
              @click="createEncounterItem"
            />
          </template>
        </auto-complete>

        <div class="ml-5 mt-3">
          <div
            v-for="(item, index) in encounterItems"
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

    <template #actions>
      <slot name="actions"></slot>
    </template>
  </expansion-panel>
</template>

<script setup lang="ts">
import { ref, computed, WritableComputedRef, PropType } from 'vue';
import _ from 'lodash';

import { useEncounterStore, useCustomEncountersStore } from '@/store';
import {
  CustomEncounterType,
  ICustomEncounter,
  IEncounter,
  IEncounterItem,
  IFilterQuery,
} from '@/types';

const encounterStore = useEncounterStore();
const customEncountersStore = useCustomEncountersStore();

const props = defineProps({
  title: {
    type: String,
    default: '',
  },
  icon: {
    type: String,
    default: '',
  },
  placeholder: {
    type: String,
    default: '',
  },
  encounterType: {
    type: String as PropType<CustomEncounterType>,
    required: true,
  },
  encounterItems: {
    type: Array as PropType<IEncounterItem[]>,
    default: () => [],
  },
});

const emit = defineEmits(['updated', 'openCustomEncounterDialog']);

const loading = ref<boolean>(false);
const customEncounters = ref<ICustomEncounter[]>([]);
const panel = ref<number | undefined>();
const selectedItem = ref<number>();
const searchKey = ref<string>('');
const isMyFavorite = ref(true);

const currentEncounter: WritableComputedRef<IEncounter> = computed(() => {
  return encounterStore.currentEncounter;
});

const getItemText = (item: any) => {
  return item[props.encounterType];
};

const searchItems = _.debounce(async () => {
  const query: IFilterQuery = {
    page: 1,
    name: searchKey.value,
    favored: isMyFavorite.value ? true : undefined,
  };

  const res = await customEncountersStore.searchCustomEncounters(
    props.encounterType,
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

const createEncounterItem = () => {
  if (!currentEncounter.value.id || !selectedItem.value) return;

  loading.value = true;
  const payload: IEncounterItem = {
    encounter: currentEncounter.value.id,
    [props.encounterType]: selectedItem.value,
  };
  encounterStore
    .addEncounterItem(payload, props.encounterType)
    .then(() => {
      emit('updated');
      clearInput();
    })
    .finally(() => (loading.value = false));
};

const deleteEncounterItem = (encounterItem: IEncounterItem) => {
  loading.value = true;
  encounterStore
    .deleteEncounterItem(encounterItem, props.encounterType)
    .then(() => {
      emit('updated');
    })
    .finally(() => (loading.value = false));
};
</script>

<style scoped lang="scss">
.switch {
  height: 44px;
  display: flex;
  align-items: center;
}
</style>
