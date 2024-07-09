<script lang="ts">
    import { fly } from "svelte/transition";
    import { waitForModalCompletion } from "../common/modal/modal-utils";
    import Modal from "../common/modal/Modal.svelte";
    import { handleError } from "../error-handler";
    import { createCtxStack, DirectoryNotFoundError, ListingContext, NoAccessError } from "./util/loader";
    import { tx } from "./util/tx";
    import AqfNoAccess from "../aqf/AqfNoAccess.svelte";
    import CreateDirectory from "./CreateDirectory.svelte";
    import FileUpload from "./FileUpload.svelte";
    import NewFileIcon from "./icons/NewFileIcon.svelte";
    import NewFolderIcon from "./icons/NewFolderIcon.svelte";
    import Listing from "./Listing.svelte";
    import MultiFactorAuthentication from "./MultiFactorAuthentication.svelte";
    import Navbar from "./Navbar.svelte";
    import { getBuildInfo, messages } from "../global";
    import { getSpaceClient } from "./util/space-context";

    const msg = messages.space.Dataroom;
    const spaceClient = getSpaceClient();
    const urlHashPrefix = "#dataroom:";

    let fileUploadModal;
    let createDirectoryModal;
    let mfaModal: Modal;
    let mfaModalOnClose;
    let dataroomDiv;

    // were we want to be
    let currentPath = getTargetPathFromUrl();

    // the stack of listings, starting with "" for root dir
    $: pathStack = currentPath.length == 0 ? [""] : `/${currentPath}`.split("/");
    let ctxStack: Array<ListingContext> = [];
    let currentCtx: ListingContext = null;
    let storageUsed: number;
    let ctxLoadError = null;
    $: {
        (async function() {
            navigate(currentPath, $tx);
        })().catch(handleError);
    }

    async function navigate(path, tx) {
        const dir = await tx.findDirectoryByPathOrNull(path);
        if (dir && spaceClient.authorizer.requestDirectoryReadAccess(dir) === "MFA_REQUIRED") {
            // upgrade to mfa session before navigating
            try {
                await waitForModalCompletion(mfaModal, (handler) => (mfaModalOnClose = handler));
            } catch (err) {
                // modal canceled - going to previously loaded dir
                console.log(err.toString());
                updateUrlPath(currentCtx ? currentCtx.path : "");
                return;
            }
        }

        try {
            const newCtxStack = await createCtxStack(pathStack, ctxStack, spaceClient, tx);
            if (newCtxStack) {
                ctxStack = newCtxStack;
                currentCtx = newCtxStack[newCtxStack.length - 1];
                storageUsed = Math.round(tx.getContentSizeTotal() / (1024 * 1024));
            }
            ctxLoadError = null;
        } catch (error) {
            ctxStack = [];
            currentCtx = null;
            if (error instanceof NoAccessError || error instanceof DirectoryNotFoundError) {
                ctxLoadError = error;
            } else {
                handleError(error);
            }
        }
    }

    function getTargetPathFromUrl() {
        let defaultPath = "";
        const hash = window.location.hash;
        if (hash.startsWith(urlHashPrefix)) {
            defaultPath = decodeURIComponent(hash.substring(urlHashPrefix.length));
        }
        return defaultPath;
    }

    function updateUrlPath(path) {
        const urlHash = urlHashPrefix + path;
        location.replace(urlHash);
    }

    window.addEventListener("popstate", function (e: PopStateEvent) {
        if (window.location.hash.startsWith(urlHashPrefix)) {
            currentPath = getTargetPathFromUrl();
        }
    });

    let isWriteAccess;
    $: {
        isWriteAccess = currentCtx != null && spaceClient.authorizer.requestDirectoryWriteAccess(currentCtx.directory) === "OK"
    }
</script>

<div class="dl-dataroom" bind:this={dataroomDiv}>
    <div class="dl-navbar">
        <Navbar {currentPath} on:navigate={(e) => updateUrlPath(e.detail.path)} />
    </div>

    {#if isWriteAccess}
        <div>
            <div class="dl-menubar-button" on:click={() => createDirectoryModal.show()}>
                <span class="dl-menubar-icon">
                    <NewFolderIcon />
                </span>
                {msg.addFolder}
            </div>

            <div class="dl-menubar-button" on:click={() => fileUploadModal.show()}>
                <span class="dl-menubar-icon">
                    <NewFileIcon />
                </span>
                {msg.addFile}
            </div>
        </div>

        <Modal bind:this={createDirectoryModal}>
            <CreateDirectory
                {currentPath}
                owner={currentCtx.directory.owner}
                on:close={createDirectoryModal.hide()} />
        </Modal>

        <Modal bind:this={fileUploadModal}>
            <FileUpload {currentPath} on:close={fileUploadModal.hide()} />
        </Modal>
    {/if}

    <div class="dl-stack" class:dl-stack-hide="{ctxLoadError != null}">
        {#each ctxStack as ctx, i (ctx.path)}
            <div
                class="dl-stackItem"
                class:dl-item_covered={i < pathStack.length - 1}
                class:dl-item_active={i == pathStack.length - 1}
                class:dl-item_obsolete={i > pathStack.length - 1}>
                <div in:fly={{ x: 400, duration: 600 }} out:fly={{ x: 400, duration: 300 }}>
                    <Listing {ctx} on:navigate={(e) => updateUrlPath(e.detail.path)} />
                </div>
            </div>
        {/each}
    </div>

    {#if ctxLoadError != null}
        <div class="dl-error">
            {#if ctxLoadError instanceof DirectoryNotFoundError}
                <p>{msg.directoryNotFound.message}</p>
            {/if}

            {#if ctxLoadError instanceof NoAccessError}
                <AqfNoAccess error={ctxLoadError}></AqfNoAccess>
            {/if}
        </div>
    {/if}

    <div class="dl-footer">
        <span>
            {#if currentCtx && spaceClient.authorizer.requestDirectoryWriteAccess(currentCtx.directory) === "OK"}
            {msg.usageSummary(storageUsed, 5000)}
            {/if}
            &nbsp;
        </span>
        <span class="dl-buildinfo">
            <span on:click|preventDefault|stopPropagation={(e) => alert(getBuildInfo())}>Build info</span>
        </span>
    </div>

    <Modal bind:this={mfaModal}>
        <MultiFactorAuthentication on:close={mfaModalOnClose} />
    </Modal>
</div>

<style>
    .dl-dataroom {
        border: 1px solid #dddd;
        border-radius: 0.5rem;
        padding: 1rem;
        max-width: 90rem;
        min-height: 20rem;
    }
    .dl-navbar {
        margin: 0.2rem 0rem 2rem 0rem;
    }

    /** menubar */
    .dl-menubar-button {
        cursor: pointer;
        display: inline-block;
        line-height: 1;
        margin: 0.3rem 1rem 0.3rem 0rem;
        padding: 0.8rem;
        background: rgb(241, 241, 241);
        border-radius: 0.5rem;
    }
    .dl-menubar-button:hover {
        background: rgb(230, 230, 230);
    }
    .dl-menubar-icon {
        overflow: auto;
        vertical-align: middle;
        padding: 0.5rem 0.4rem 0rem 0;
    }

    /** listing stack */
    .dl-stack {
        position: relative;
        margin: 1rem 0rem 2rem 0rem;
        min-height: 20rem;
    }
    .dl-stack-hide {
        display: none;
    }
    .dl-stackItem {
        top: 0;
        width: 100%;
    }
    .dl-stackItem.dl-item_covered {
        position: absolute;
        visibility: hidden;
        opacity: 0;
        transition: visibility 0.3s, opacity 0.3s linear;
    }
    .dl-stackItem.dl-item_active {
        position: relative;
    }
    .dl-stackItem.dl-item_obsolete {
        position: absolute;
    }

    /** footer */
    .dl-footer {
        color: rgb(181, 181, 181);
    }
    .dl-buildinfo {
        float: right;
        color: #fafcfb;
        cursor: default;
    }

    .dl-error p {
        margin-top: 4rem;
        margin-bottom: 2rem;
    }
</style>
