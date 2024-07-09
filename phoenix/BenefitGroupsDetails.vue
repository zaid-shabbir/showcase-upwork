<script setup>
import { ref } from 'vue';
import lodash from 'lodash';
import { useHelpers } from '@/composables';
import { computed } from 'vue';
import { onMounted } from 'vue';
import { useBenefitStore } from '@/store';
import useEventsBus from '@/composables/event-bus';
import BenefitGroupsDetailsForm from '@/components/benefit-groups/forms/BenefitGroupsDetailsForm.vue';

const props = defineProps({
    data: {
        type: Object,
        required: true
    },
    isNew: {
        type: Boolean,
        default: false
    }
});

const emits = defineEmits(['reload']);

const helpers = useHelpers();
const benefitStore = useBenefitStore();
const { emit } = useEventsBus();

const busy = ref(false);
const loading = ref(false);
const isEditing = ref(false);
const discardDialog = ref(false);
const item = ref({});
const itemToUpdate = ref({});

onMounted(() => {
    setItem();
});

const isNotChanged = computed(() => {
    return lodash.isEqual(item.value, itemToUpdate.value);
});

const setItem = () => {
    if (props.data) item.value = lodash.cloneDeep(props.data);
};

const processData = (data) => {
    item.value = lodash.cloneDeep(data);
    itemToUpdate.value = lodash.cloneDeep(item.value);
};

const handleEdit = () => {
    itemToUpdate.value = lodash.cloneDeep(item.value);
    isEditing.value = true;
};

const discardChanges = () => {
    isNotChanged.value
        ? (isEditing.value = false)
        : (discardDialog.value = true);
};

const save = async () => {
    try {
        busy.value = true;
        const res = await benefitStore.updateBenefitGroup(
            itemToUpdate.value.id,
            {
                ...itemToUpdate.value,
                end_date:
                    itemToUpdate.value.end_date != 'Invalid date'
                        ? itemToUpdate.value.end_date
                        : null
            }
        );
        processData(res.data);
        isEditing.value = false;
        emits('reload');
        emit('reloadBenefitGroupDetails');
    } finally {
        busy.value = false;
    }
};
</script>
<template>
    <Loader v-if="loading" />
    <div v-else>
        <div class="flex justify-content-between align-items-center">
            <h5 class="mb-2">
                {{ $t('benefit_groups.benefit_group_details') }}
            </h5>
            <div v-if="isEditing">
                <Button
                    label="Cancel"
                    class="p-button-outlined mr-2"
                    @click="discardChanges"
                />
                <Button
                    label="Save"
                    :loading="busy"
                    :disabled="isNotChanged"
                    @click="save"
                />
            </div>
            <Button
                v-else
                size="small"
                text
                class="px-2 py-1"
                label="Edit"
                icon="pi pi-pencil"
                @click="handleEdit"
            />
        </div>

        <BenefitGroupsDetailsForm
            v-if="isEditing"
            :isNew="isNew"
            v-model="itemToUpdate"
        />

        <div v-else class="grid mt-1">
            <div class="sm:col-6 md:col-2 text-sm font-semibold py-1">
                {{ $t('common.name') }}:
            </div>
            <div class="sm:col-6 md:col-10 text-sm py-1 p-break-word">
                {{ helpers.getLocaleValue(item.name) }}
            </div>
            <div class="sm:col-6 md:col-2 text-sm font-semibold py-1">
                {{ $t('common.coverage') }}:
            </div>
            <div class="sm:col-6 md:col-10 text-sm py-1">
                {{ item.coverage ? item.coverage + ' %' : '-' }}
            </div>
            <div class="sm:col-6 md:col-2 text-sm font-semibold py-1">
                {{ $t('common.to_a_maximum_of') }}:
            </div>
            <div class="sm:col-6 md:col-10 text-sm py-1">
                {{
                    item.max_amount ? helpers.moneyFormat(item.max_amount) : '-'
                }}
            </div>
            <div class="sm:col-6 md:col-2 text-sm font-semibold py-1">
                {{ $t('common.effective_date') }}:
            </div>
            <div class="sm:col-6 md:col-10 text-sm py-1">
                {{ helpers.formatDate(item.effective_date) }}
            </div>
            <div class="sm:col-6 md:col-2 text-sm font-semibold py-1">
                {{ $t('common.end_date') }}:
            </div>
            <div class="sm:col-6 md:col-10 text-sm py-1">
                {{ helpers.formatDate(item.end_date) }}
            </div>
        </div>
    </div>
    <Confirmation
        v-model="discardDialog"
        show-alert-icon
        :header="$t('common.discard_header')"
        :content="$t('common.discard_content')"
        confirm-button-class="p-button-danger"
        :confirmButtonText="$t('common.discard_cancel')"
        :cancelButtonText="$t('common.discard_continue')"
        @confirm="isEditing = false"
    />
</template>
