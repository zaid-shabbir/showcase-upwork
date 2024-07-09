<script setup>
import { ref } from 'vue';
import BenefitGroupsDocumentsTable from '@/components/benefit-groups/tables/BenefitGroupsDocumentsTable.vue';
import BenefitGroupsUploadForm from '@/components/benefit-groups/forms/BenefitGroupsUploadForm.vue';

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
const isUploadFiles = ref(false);

const cancelUploadFiles = () => {
    isUploadFiles.value = false;
};
</script>

<template>
    <div>
        <BenefitGroupsUploadForm
            :id="props.id"
            v-if="isUploadFiles"
            @cancel="cancelUploadFiles"
        />
        <div v-else>
            <Card class="mt-5">
                <template #content>
                    <div
                        class="flex justify-content-between align-items-center mb-3"
                    >
                        <h5>
                            {{ $t('common.documents_and_files') }}
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

                    <BenefitGroupsDocumentsTable
                        :id="props.id"
                        @upload-files="isUploadFiles = true"
                        :is-editable="isEditing || isNew"
                    />
                </template>
            </Card>
        </div>
    </div>
</template>

<style lang="scss" scoped></style>
