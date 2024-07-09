<script setup>
import { ref, watch } from 'vue';
import DocumentInfo from '@/components/claim-intake-details/components/DocumentInfo.vue';
import History from '@/components/claim-intake-details/components/History.vue';
import Batch from '@/components/claim-intake-details/components/Batch.vue';

const props = defineProps({
    claimDetails: Object,
    activeIndexes: Array,
    expandAll: Function,
    collapseAll: Function,
    validateFlag: Boolean,
    saveFlag: Boolean,
    refreshHistory: Number
});
const refreshHistoryKey = ref(1);
const emit = defineEmits(['refresh', 'reload-claim-details']);

watch(
    () => props.refreshHistory,
    () => {
        refreshHistoryKey.value++;
    }
);

const refresh = () => {
    emit('refresh');
};

const updated = () => {
    refreshHistoryKey.value++;
};
</script>

<template>
    <TabView>
        <TabPanel>
            <template #header>
                <div class="flex align-items-center gap-2">
                    <i class="pi pi-file-edit" />
                    <span class="font-bold white-space-nowrap"
                        >Document Info</span
                    >
                </div>
            </template>
            <DocumentInfo
                :claim-details="claimDetails"
                :active-indexes="activeIndexes"
                :activeIndexes="activeIndexes"
                :expand-all="expandAll"
                :collapse-all="collapseAll"
                :validateFlag="validateFlag"
                :saveFlag="saveFlag"
                @updated="updated"
                @reload-claim-details="emit('reload-claim-details')"
            />
        </TabPanel>
        <TabPanel>
            <template #header>
                <div class="flex align-items-center gap-2">
                    <i class="pi pi-clock" />
                    <span class="font-bold white-space-nowrap">History</span>
                </div>
            </template>
            <History
                :claim-details="claimDetails"
                :active-indexes="activeIndexes"
                :expand-all="expandAll"
                :key="refreshHistoryKey"
                :collapse-all="collapseAll"
            />
        </TabPanel>
        <TabPanel>
            <template #header>
                <div class="flex align-items-center gap-2">
                    <i class="pi pi-th-large" />
                    <span class="font-bold white-space-nowrap">Batch</span>
                </div>
            </template>

            <hr />

            <Batch :claim-details="claimDetails" @refresh="refresh" />
        </TabPanel>
    </TabView>
</template>
