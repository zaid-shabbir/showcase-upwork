<script setup>
import { ref, computed, nextTick } from 'vue';
import lodash from 'lodash';

import BenefitGroupsPriceForm from '@/components/benefit-groups/forms/BenefitGroupsPriceForm.vue';
import BenefitGroupsPricingTable from '@/components/benefit-groups/tables/BenefitGroupsPricingTable.vue';

const props = defineProps({
    id: {
        type: String,
        required: true
    },
    isNew: {
        type: Boolean,
        default: false
    }
});

const discardDialog = ref(false);
const isEditing = ref(false);
const openDialog = ref(false);
const itemToUpdate = ref({});
const item = ref({});

const isNotChanged = computed(() => {
    return lodash.isEqual(item.value, itemToUpdate.value);
});

const createNewPrice = () => {
    resetItem();
    openDialog.value = true;
};

const resetItem = () => {
    item.value = {
        unit_term: null,
        regions: [],
        countries: [],
        net_price: null,
        sale_price: null,
        min_days: null,
        max_days: null,
        effective_date: null,
        end_date: null
    };
    itemToUpdate.value = {
        unit_term: null,
        regions: [],
        countries: [],
        net_price: null,
        sale_price: null,
        min_days: null,
        max_days: null,
        effective_date: null,
        end_date: null
    };
};

const editPrice = (data) => {
    item.value = lodash.cloneDeep(data);
    itemToUpdate.value = lodash.cloneDeep(data);
    openDialog.value = true;
};

const discardChanges = () => {
    isNotChanged.value
        ? (openDialog.value = false)
        : (discardDialog.value = true);
};

const saved = (addOther) => {
    openDialog.value = false;

    if (addOther) {
        nextTick(() => {
            createNewPrice();
        });
    }
};
</script>

<template>
    <div>
        <div class="flex justify-content-between align-items-center mb-3">
            <h5>
                {{
                    isNew
                        ? $t('benefit_groups.pricing_based_on_region_country')
                        : $t('common.pricing')
                }}
            </h5>
            <div v-if="!isNew">
                <Button
                    v-if="isEditing"
                    label="Cancel"
                    class="p-button-outlined mr-2"
                    @click="isEditing = false"
                />
                <Button
                    v-else
                    size="small"
                    text
                    class="px-2 py-1"
                    label="Edit"
                    icon="pi pi-pencil"
                    @click="isEditing = true"
                />
            </div>
        </div>
        <BenefitGroupsPricingTable
            :id="props.id"
            :is-editable="isEditing || isNew"
            @addNewPrice="createNewPrice"
            @edit-price="editPrice"
        />
        <BenefitGroupsPriceForm
            v-if="openDialog"
            v-model="itemToUpdate"
            :id="props.id"
            :is-open="openDialog"
            @close="discardChanges"
            @saved="saved"
        />
    </div>

    <Confirmation
        v-model="discardDialog"
        show-alert-icon
        :header="$t('common.discard_header')"
        :content="$t('common.discard_content')"
        confirm-button-class="p-button-danger"
        :confirmButtonText="$t('common.discard_cancel')"
        :cancelButtonText="$t('common.discard_continue')"
        @confirm="openDialog = false"
    />
</template>

<style lang="scss" scoped></style>
