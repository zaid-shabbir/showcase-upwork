<script setup>
import { ref } from 'vue';
import BenefitGroupsBenefitsTable from '@/components/benefit-groups/tables/BenefitGroupsBenefitsTable.vue';
import BenefitGroupsBenefitsForm from '@/components/benefit-groups/forms/BenefitGroupsBenefitsForm.vue';

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

const isEditing = ref(false);
const totalBenefitIncluded = ref(0);

const setTotalBenefitIncluded = (val) => {
    totalBenefitIncluded.value = val;
};
</script>

<template>
    <div>
        <div class="flex justify-content-between align-items-center mb-3">
            <h5>
                {{ $t('common.benefits') }}
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

        <BenefitGroupsBenefitsForm
            v-if="isEditing || isNew"
            :id="props.id"
            @setTotalBenefitIncluded="setTotalBenefitIncluded"
        />

        <h5 v-if="isNew && totalBenefitIncluded > 0" class="mb-3">
            {{ totalBenefitIncluded }}
            {{ $t('benefit_groups.benefits_included') }}
        </h5>

        <BenefitGroupsBenefitsTable
            @setTotalBenefitIncluded="setTotalBenefitIncluded"
            :is-editable="isEditing || isNew"
            :id="props.id"
        />
    </div>
</template>

<style lang="scss" scoped></style>
