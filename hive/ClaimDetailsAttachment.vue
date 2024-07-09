<script setup>
import { ref, defineProps, onMounted, computed } from 'vue';
import {
    softDeletePage,
    restorePageDocument
} from '@/services/ClaimsIntake.service';
import { useCapitalizeWords } from './composables/fieldFormats';
import { useToast } from 'primevue/usetoast';
import { useConfirm } from 'primevue/useconfirm';
import { useNeedsReviewStore } from '@/store';
import Loader from '@/components/common/Loader.vue';

const props = defineProps({
    claimAttachmentMessage: { type: String, default: '' },
    pages: { type: Array, default: () => [] },
    claimDetails: { type: Object, default: () => {} },
    archivedPages: { type: Array, default: () => [] }
});

const emit = defineEmits(['reload-attachment']);
const toast = useToast();
const confirm = useConfirm();
const reviewStore = useNeedsReviewStore();
const deleteInProgress = ref(false);
const restoreInProgress = ref(false);
const selectedPage = ref(null);
const currentPageIndex = ref(null);
const currentRestoredPageIndex = ref(null);
const zoomLevel = ref(90);

const start = ref(10);
const end = ref(200);
const step = ref(10);
const percent = computed(() => {
    const items = [];
    for (let i = start.value; i <= end.value; i += step.value) {
        items.push({ name: `${i}%`, value: i });
    }
    return items;
});

const uniqueThumbnailCount = computed(() => {
    return new Set(props.pages.map((page) => page.thumbnailLink)).size;
});

const { formatText } = useCapitalizeWords();

onMounted(() => {
    if (props.pages.length > 0) {
        selectPage(props.pages[0], 0);
    } else if (props.archivedPages.length > 0) {
        selectRestoredPage(props.archivedPages[0], 0);
    }
});

function selectPage(page, index) {
    selectedPage.value = page;
    currentPageIndex.value = index;
    currentRestoredPageIndex.value = null;
    reviewStore.updateSelectedPage(selectedPage.value);
}

function selectRestoredPage(archivedPage, index) {
    selectedPage.value = archivedPage;
    currentRestoredPageIndex.value = index;
    currentPageIndex.value = null;
    reviewStore.updateSelectedPage(selectedPage.value);
}

function moveToNextPage() {
    if (
        currentPageIndex.value !== null &&
        currentPageIndex.value < props.pages.length - 1
    ) {
        currentPageIndex.value++;
        selectedPage.value = props.pages[currentPageIndex.value];
    } else if (
        currentRestoredPageIndex.value !== null &&
        currentRestoredPageIndex.value < props.archivedPages.length - 1
    ) {
        currentRestoredPageIndex.value++;
        selectedPage.value =
            props.archivedPages[currentRestoredPageIndex.value];
    }
}

function moveToPreviousPage() {
    if (currentPageIndex.value !== null && currentPageIndex.value > 0) {
        currentPageIndex.value--;
        selectedPage.value = props.pages[currentPageIndex.value];
    } else if (
        currentRestoredPageIndex.value !== null &&
        currentRestoredPageIndex.value > 0
    ) {
        currentRestoredPageIndex.value--;
        selectedPage.value =
            props.archivedPages[currentRestoredPageIndex.value];
    }
}

function increaseZoom() {
    zoomLevel.value = Math.min(zoomLevel.value + 10, 200);
}

function decreaseZoom() {
    zoomLevel.value = Math.max(zoomLevel.value - 10, 10);
}

async function deletePage() {
    if (!selectedPage.value || currentPageIndex.value === null) return;
    const pageIndexToDelete = currentPageIndex.value;
    confirm.require({
        message: `Are you sure that you want to delete the ${
            selectedPage.value.pageType
        } on page ${currentPageIndex.value + 1} ?`,
        header: 'Delete Page',
        rejectClass: 'p-button-primary p-button-outlined',
        rejectLabel: 'Cancel',
        acceptClass: 'p-button-danger',
        acceptLabel: 'Delete',
        accept: async () => {
            try {
                const pageUUID = props.pages[currentPageIndex.value].uuid;
                deleteInProgress.value = true;
                await softDeletePage(pageUUID);
                emit('reload-attachment');
                deleteInProgress.value = false;
                props.pages.splice(currentPageIndex.value, 1);

                if (pageIndexToDelete >= props.pages.length) {
                    currentPageIndex.value = props.pages.length - 1;
                } else {
                    currentPageIndex.value = pageIndexToDelete;
                }

                if (props.pages.length > 0) {
                    selectedPage.value = props.pages[currentPageIndex.value];
                } else {
                    selectedPage.value = null;
                }

                toast.add({
                    severity: 'success',
                    summary: 'Page Deleted',
                    detail: `Page ${pageIndexToDelete + 1} has been deleted.`,
                    life: 3000
                });
            } catch (error) {
                toast.add({
                    severity: 'error',
                    summary: 'Error',
                    detail: 'Error deleting page.'
                });
            }
        },
        reject: () => {}
    });
}

async function restorePage() {
    if (!selectedPage.value || currentRestoredPageIndex.value === null) return;
    confirm.require({
        message: `Are you sure that you want to restore the ${
            selectedPage.value.pageType
        } on page ${currentRestoredPageIndex.value + 1} ?`,
        header: 'Restore Page',
        rejectClass: 'p-button-primary p-button-outlined',
        rejectLabel: 'Cancel',
        acceptClass: 'p-button-primary',
        acceptLabel: 'Confirm',
        accept: async () => {
            try {
                const pageUUID =
                    props.archivedPages[currentRestoredPageIndex.value].uuid;
                restoreInProgress.value = true;
                await restorePageDocument(pageUUID);
                emit('reload-attachment');
                restoreInProgress.value = false;

                const restoredPage = props.archivedPages.splice(
                    currentRestoredPageIndex.value,
                    1
                )[0];
                props.pages.push(restoredPage);
                currentPageIndex.value = props.pages.length - 1;
                selectedPage.value = props.pages[currentPageIndex.value];
                currentRestoredPageIndex.value = null;

                toast.add({
                    severity: 'success',
                    summary: 'Page Restored',
                    detail: 'Page has been restored.',
                    life: 3000
                });
            } catch (error) {
                toast.add({
                    severity: 'error',
                    summary: 'Error',
                    detail: 'Error restoring page.'
                });
            }
        },
        reject: () => {}
    });
}
</script>

<template>
    <div v-if="pages.length || archivedPages.length">
        <Loader
            v-if="deleteInProgress || restoreInProgress"
            class="loader-component"
        />
        <div class="w-full flex flex-row">
            <div
                class="flex flex-column gap-4 bg-black-alpha-80 text-white p-4 text-center w-19rem"
            >
                <div
                    v-for="(page, index) in pages"
                    :key="page.uuid"
                    @click="selectPage(page, index)"
                    class="cursor-pointer"
                >
                    <div
                        v-if="
                            claimDetails.status !== 'submitted' &&
                            pages.length > 1
                        "
                    >
                        <div v-if="currentPageIndex === index">
                            <a href="#" @click="deletePage">
                                <div class="icon-button icon-delete-button">
                                    <i class="pi pi-trash"></i>
                                </div>
                            </a>
                        </div>
                    </div>
                    <div
                        v-if="page.thumbnailLink"
                        class="mb-3 flex flex-column align-items-center"
                    >
                        <img
                            :src="page.thumbnailLink"
                            :alt="'Page ' + (index + 1)"
                            class="thumbnail-size"
                            :class="{ bordered: currentPageIndex === index }"
                        />
                        <div class="mt-2 text-center">
                            {{ formatText(page.pageType) }}
                        </div>
                    </div>
                    <div v-else class="flex flex-column">
                        <img
                            src="@/assets/images/no-claims-image.svg"
                            :alt="'Page ' + (index + 1)"
                            :class="{ bordered: currentPageIndex === index }"
                            class="h-11rem"
                        />
                        <div class="mt-2 text-center">No Image</div>
                    </div>
                </div>
                <div v-if="archivedPages.length > 0">
                    <Accordion>
                        <AccordionTab
                            header="Trash"
                            :pt="{
                                content: {
                                    class: 'bg-transparent border-0 border-transparent text-white w-full '
                                }
                            }"
                        >
                            <div
                                v-for="(archivedPage, index) in archivedPages"
                                :key="archivedPage.uuid"
                                class="cursor-pointer"
                                @click="selectRestoredPage(archivedPage, index)"
                            >
                                <div
                                    v-if="archivedPage.thumbnailLink"
                                    class="pa-4"
                                >
                                    <div
                                        v-if="
                                            claimDetails.status !== 'submitted'
                                        "
                                    >
                                        <div
                                            v-if="
                                                currentRestoredPageIndex ===
                                                index
                                            "
                                        >
                                            <a href="#" @click="restorePage">
                                                <div
                                                    class="icon-button icon-plus-button"
                                                >
                                                    <i class="pi pi-plus"></i>
                                                </div>
                                            </a>
                                        </div>
                                    </div>
                                    <img
                                        :src="archivedPage.thumbnailLink"
                                        :alt="'Page ' + (index + 1)"
                                        class="thumbnail-size"
                                        :class="{
                                            bordered:
                                                currentRestoredPageIndex ===
                                                index
                                        }"
                                    />
                                    <div class="mt-2 text-center">
                                        {{ formatText(archivedPage.pageType) }}
                                    </div>
                                </div>
                            </div>
                        </AccordionTab>
                    </Accordion>
                </div>
            </div>

            <div
                v-if="selectedPage"
                class="w-full min-h-screen text-center bg-black-alpha-60"
            >
                <div
                    class="flex flex-column md:flex-row justify-content-between align-items-baseline w-full justify-between items-center px-5 py-4 text-xl"
                >
                    <div>
                        <Button
                            type="button"
                            icon="pi pi-chevron-up"
                            class="p-button-text p-button-plain text-white border-1 border-white-alpha-30 px-4 md:mr-2"
                            @click="moveToPreviousPage"
                        ></Button>
                        <Button
                            type="button"
                            icon="pi pi-chevron-down"
                            class="p-button-text p-button-plain text-white border-1 border-white-alpha-30 px-4 md:mr-3"
                            @click="moveToNextPage"
                        ></Button>
                    </div>

                    <div class="text-white">
                        <span
                            >Page
                            <InputField
                                variant="text"
                                class="w-2 bg-transparent text-white mx-2"
                                :value="
                                    currentPageIndex !== null
                                        ? currentPageIndex + 1
                                        : currentRestoredPageIndex !== null
                                        ? currentRestoredPageIndex + 1
                                        : ''
                                "
                                readonly
                            ></InputField>
                            of {{ uniqueThumbnailCount }}</span
                        >
                    </div>

                    <div>
                        <Button
                            type="button"
                            icon="pi pi-minus"
                            class="p-button-text p-button-plain text-white border-1 border-white-alpha-30 px-4 md:mr-2"
                            @click="decreaseZoom"
                        ></Button>

                        <Dropdown
                            v-model="zoomLevel"
                            :options="percent"
                            optionLabel="name"
                            optionValue="value"
                            placeholder="100%"
                            class="w-full md:w-14rem md:mr-2 bg-transparent text-white px-4 border-white-alpha-30 h-3rem"
                            :pt="{
                                dropdownIcon: {
                                    class: 'text-white'
                                },
                                input: {
                                    class: 'text-white'
                                }
                            }"
                        />

                        <Button
                            type="button"
                            icon="pi pi-plus"
                            class="p-button-text p-button-plain text-white border-1 border-white-alpha-30 px-4 md:mr-2"
                            @click="increaseZoom"
                        ></Button>
                    </div>
                </div>
                <div
                    class="overflow-hidden"
                    v-if="selectedPage && selectedPage.thumbnailLink !== ''"
                >
                    <img
                        :src="selectedPage.thumbnailLink"
                        :alt="
                            'Page ' +
                            (currentPageIndex !== null
                                ? currentPageIndex + 1
                                : currentRestoredPageIndex + 1)
                        "
                        class="inline-block"
                        :style="{ width: zoomLevel + '%' }"
                    />
                </div>
                <div
                    v-else
                    class="flex justify-content-center align-items-center h-screen"
                >
                    <img
                        src="@/assets/images/no-claims-image-full.svg"
                        alt="No Image"
                        class="thumbnail-size"
                    />
                </div>
            </div>
        </div>
    </div>
    <div v-else>No pages available.</div>
</template>

<style scoped lang="scss">
.thumbnail-size {
    display: flex;
    justify-content: center;
    height: 200px;
    width: 145px;
}

.bordered {
    border-radius: 5px;
    border: 5px solid red;
    overflow: hidden;
}

.icon-button {
    display: flex;
    justify-content: center;
    align-items: center;
    position: relative;
    z-index: 1;
    background-color: red;
    color: white;
    border-radius: 50%;
    cursor: pointer;
    width: 40px;
    height: 40px;
    font-size: 1.5rem;

    &:hover {
        background-color: lightcoral;
    }
}

.icon-delete-button {
    top: 30px;
    left: 3px;
}

.icon-plus-button {
    top: 30px;
    left: -15px;
}
</style>
