<script setup>
import { ref, watch } from 'vue';
import { useReviewData, menuItemsOptions } from '@/composables';
import { lockDoucumentIntake } from '@/services/ClaimsIntake.service';

const {
    loading,
    items,
    selectedProducts,
    sources,
    selectedSource,
    selectedDate,
    selectedFrom,
    aiRecognitionOptions,
    selectedAIRecognition,
    pagination,
    onSort,
    displayFormatDate,
    fetchItems,
    onPageChange,
    onRowsChange
} = useReviewData();

const dialogVisible = ref(false);
const spamDialogVisible = ref(false);
const followUpDialogVisible = ref(false);
const trashDialogVisible = ref(false);

const emit = defineEmits([
    'update:items',
    'itemsDeleted',
    'itemsSpammed',
    'itemsFollowUp'
]);

const {
    menuRef,
    menuItems,
    toggleMenuItem,
    buttonLabel,
    confirmDeletion,
    confirmSpam,
    confirmFollowUp,
    deletionConfirmationMessage,
    spamConfirmationMessage,
    followUpConfirmationMessage,
    isMoveTrash,
    isMoveSpam,
    isMoveFollowUp
} = menuItemsOptions(
    selectedProducts,
    items,
    emit,
    dialogVisible,
    spamDialogVisible,
    followUpDialogVisible,
    trashDialogVisible,
    'pending'
);

watch(
    [
        dialogVisible,
        spamDialogVisible,
        followUpDialogVisible,
        trashDialogVisible
    ],
    (newVals, oldVals) => {
        const anyDialogClosed = newVals.some(
            (newVal, index) => !newVal && oldVals[index]
        );
        if (anyDialogClosed) {
            if (items.value.length === 0 && pagination.currentPage > 1) {
                pagination.currentPage--;
            }
            fetchItems();
        }
    }
);

const closeAndReload = () => {
    trashDialogVisible.value = false;
    dialogVisible.value = false;
    spamDialogVisible.value = false;
    followUpDialogVisible.value = false;

    fetchItems(true);
};
</script>

<template>
    <div class="card">
        <h5 class="py-3">Needs Review</h5>
        <DataTable
            :value="items"
            @sort="onSort"
            key="uuid"
            :loading="loading"
            v-model:selection="selectedProducts"
            :paginator="true"
            :rows="pagination.perPage"
            :totalRecords="pagination.totalRecords"
            :currentPage="pagination.currentPage"
            @page="onPageChange"
            :rowsPerPageOptions="[5, 15, 30]"
            @rows-per-page-change="onRowsChange"
            :paginatorTemplate="`CurrentPageReport FirstPageLink PrevPageLink PageLinks NextPageLink LastPageLink RowsPerPageDropdown`"
            :currentPageReportTemplate="`Showing {first} to {last} of {totalRecords} items`"
            :lazy="true"
            responsiveLayout="scroll"
            :first="
                pagination.currentPage * pagination.perPage - pagination.perPage
            "
        >
            <template #header>
                <div class="flex justify-content-between">
                    <div class="flex gap-2">
                        <Button
                            :label="buttonLabel"
                            :disabled="selectedProducts.length === 0"
                            @click="toggleMenuItem"
                            class="bg-transparent text-primary"
                            icon="pi pi-chevron-down"
                            iconPos="right"
                        ></Button>
                        <Menu ref="menuRef" popup :model="menuItems"></Menu>

                        <Calendar
                            v-model="selectedDate"
                            showButtonBar
                            showIcon
                            placeholder="Select Date Received"
                            :pt="{
                                dropdownButton: {
                                    icon: { class: 'text-gray-500' },
                                    root: { class: 'bg-white border-gray-400' }
                                }
                            }"
                        >
                        </Calendar>
                        <Dropdown
                            optionLabel="label"
                            optionValue="value"
                            placeholder="Source"
                            :options="sources"
                            v-model="selectedSource"
                        />
                        <Dropdown
                            optionLabel="label"
                            optionValue="value"
                            placeholder="AI Recognition"
                            :options="aiRecognitionOptions"
                            v-model="selectedAIRecognition"
                        />
                    </div>
                    <Search
                        v-model="selectedFrom"
                        placeholder="Search Provider"
                    />
                </div>
            </template>
            <template #empty> No claims found.</template>
            <template #loading>
                <div class="loader"></div>
            </template>
            <Column selectionMode="multiple"></Column>
            <Column field="isLocked">
                <template #body="data">
                    <i v-if="data.data.isLocked" class="pi pi-lock"> </i>
                </template>
            </Column>
            <Column field="document.uuid" :sortable="true" header="Record ID">
                <template #body="data">
                    {{ data.data.uuid.slice(0, 8) }}
                </template>
            </Column>

            <Column
                field="document.receivedAt"
                :sortable="true"
                header="Received At"
            >
                <template #body="slotProps">
                    {{ displayFormatDate(slotProps.data.document.receivedAt) }}
                </template>
            </Column>
            <Column
                :sortable="true"
                field="document.source"
                header="Source"
                class="capitalize"
            ></Column>
            <Column
                :sortable="true"
                field="document.receivedFrom"
                header="Received From"
            ></Column>
            <Column :sortable="true" field="pages" header="Pgs">
                <template #body="{ data }">
                    {{ data.pages.length }}
                </template>
            </Column>
            <Column
                :sortable="true"
                field="unrecognized"
                header="AI Recognition"
            >
                <template #body="{ data }">
                    <Tag
                        class="uppercase font-semibold"
                        :value="
                            data.unrecognized === false
                                ? 'Recognized'
                                : 'Unrecognized'
                        "
                        :icon="
                            data.unrecognized === false
                                ? 'pi pi-fw pi-check-circle'
                                : 'pi pi-fw pi-flag'
                        "
                        :severity="
                            data.unrecognized === false ? 'success' : 'danger'
                        "
                    />
                </template>
            </Column>
            <Column :sortable="true" field="accuracy" header="Accuracy">
                <template #body="{ data }">
                    <ProgressBar
                        v-if="parseFloat((data.accuracy * 100).toFixed(1)) > 0"
                        :value="parseFloat((data.accuracy * 100).toFixed(1))"
                        :showValue="true"
                        :pt="{
                            value: {
                                class: {
                                    'bg-orange-500 px-4':
                                        parseFloat(
                                            (data.accuracy * 100).toFixed(1)
                                        ) <= 50,
                                    'bg-green-800':
                                        parseFloat(
                                            (data.accuracy * 100).toFixed(1)
                                        ) > 50
                                }
                            }
                        }"
                    ></ProgressBar>
                    <Tag
                        v-else
                        severity="danger"
                        class="w-full text-sm font-medium"
                        >0%
                    </Tag>
                </template>
            </Column>
            <Column field="Details" header="Details">
                <template #body="{ data }">
                    <div class="text-center">
                        <router-link
                            :to="{
                                name: 'Details',
                                params: { uuid: data.uuid }
                            }"
                        >
                            <Button
                                icon="pi pi-align-left"
                                aria-label="Search"
                                severity="primary"
                                class="px-4"
                            />
                        </router-link>
                    </div>
                </template>
            </Column>
        </DataTable>
        <Dialog v-model:visible="trashDialogVisible" :closable="false" modal>
            <template #header>
                <div
                    class="flex justify-content-between align-items-center w-full"
                >
                    <h4>Delete Intake Entries</h4>
                    <Button
                        icon="pi pi-times"
                        class="p-button-rounded bg-white border-none text-color-secondary hover:bg-blue-50"
                        @click="closeAndReload"
                    />
                </div>
            </template>
            <p v-html="deletionConfirmationMessage"></p>
            <template #footer>
                <Button
                    label="Cancel"
                    @click="closeAndReload"
                    class="bg-white border-0 font-bold text-primary"
                />
                <Button
                    label="Delete"
                    @click="confirmDeletion"
                    class="p-button-danger"
                    :loading="isMoveTrash"
                    :disabled="isMoveTrash"
                />
            </template>
        </Dialog>

        <Dialog v-model:visible="spamDialogVisible" :closable="false" modal>
            <template #header>
                <div
                    class="flex justify-content-between align-items-center w-full"
                >
                    <h4>Report Spam</h4>
                    <Button
                        icon="pi pi-times"
                        class="p-button-rounded bg-white border-none text-color-secondary hover:bg-blue-50"
                        @click="closeAndReload"
                    />
                </div>
            </template>
            <p v-html="spamConfirmationMessage"></p>
            <template #footer>
                <Button
                    label="Cancel"
                    @click="closeAndReload"
                    class="bg-white border-0 font-bold text-primary"
                />
                <Button
                    label="Mark as Spam"
                    @click="confirmSpam"
                    class="p-button-primary"
                    :loading="isMoveSpam"
                    :disabled="isMoveSpam"
                />
            </template>
        </Dialog>

        <Dialog v-model:visible="followUpDialogVisible" :closable="false" modal>
            <template #header>
                <div
                    class="flex justify-content-between align-items-center w-full"
                >
                    <h4>Move to Follow Up</h4>
                    <Button
                        icon="pi pi-times"
                        class="p-button-rounded bg-white border-none text-color-secondary hover:bg-blue-50"
                        @click="closeAndReload"
                    />
                </div>
            </template>
            <p v-html="followUpConfirmationMessage"></p>
            <template #footer>
                <Button
                    label="Cancel"
                    @click="closeAndReload"
                    class="bg-white border-0 font-bold text-primary"
                />
                <Button
                    label="Confirm"
                    @click="confirmFollowUp"
                    class="p-button-primary"
                    :loading="isMoveFollowUp"
                    :disabled="isMoveFollowUp"
                />
            </template>
        </Dialog>
    </div>
</template>
