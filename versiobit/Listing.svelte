<script lang="ts">
    import { createEventDispatcher } from "svelte";
    import Menu from "../common/menu/Menu.svelte";
    import MenuDivider from "../common/menu/MenuDivider.svelte";
    import MenuOption from "../common/menu/MenuOption.svelte";
    import type { ListingContext } from "./util/loader";
    import { formateDate, formatSince } from "../common/time";
    import DeleteIcon from "./icons/DeleteIcon.svelte";
    import DownloadIcon from "./icons/DownloadIcon.svelte";
    import EditIcon from "./icons/EditIcon.svelte";
    import FileIcon from "./icons/FileIcon.svelte";
    import FolderIcon from "./icons/FolderIcon.svelte";
    import MoreIcon from "./icons/MoreIcon.svelte";
    import Modal from "../common/modal/Modal.svelte";
    import DeleteFile from "./DeleteFile.svelte";
    import DeleteDirectory from "./DeleteDirectory.svelte";
    import Rename from "./Rename.svelte";
    import History from "./History.svelte";
    import Proof from "./Proof.svelte"
    import { handleError } from "../error-handler";
    import LockIcon from "./icons/LockIcon.svelte";
    import HistoryIcon from "./icons/HistoryIcon.svelte";
    import { tx } from "./util/tx";
    import { messages } from "../global";
    import { getSpaceClient } from "./util/space-context";

    export let ctx: ListingContext;
    let showInaccessible: boolean = true;
    const dispatch = createEventDispatcher();
    const msg = messages.space.Listing;
    const spaceClient = getSpaceClient();

    let renameModal;
    let historyModal;
    let deleteFileModal;
    let deleteDirectoryModal;
    let proofModal;

    function navigateToDir(dir) {
        const absolutePath = ctx.pathOf(dir.name);
        dispatch("navigate", { path: absolutePath });
    }

    // menu
    let menuContainer;
    let menuPosition = { x: 0, y: 0 };
    let menuVisible = false;
    let selection: versiobit_client.versiobit.ocas.FileMetadata & versiobit_client.versiobit.ocas.DirectoryMetadata = null;
    let clickedProofFile: versiobit_client.versiobit.ocas.FileMetadata
    let selectionWriteAccess: boolean = false;
    $: {
        if (selection != null) {
            if (selection.files) {
                // directory
                selectionWriteAccess = spaceClient.authorizer.requestDirectoryWriteAccess(selection) !== "INSUFFICIENT_ACCESS";
            } else {
                // file
                selectionWriteAccess = spaceClient.authorizer.requestFileWriteAccess(selection, ctx.directory) !== "INSUFFICIENT_ACCESS";
            }
        }
    }

    function showMenu(e, item) {
        (async function () {
            selection = item;
            if (menuVisible) {
                menuVisible = false;
                await new Promise((res) => setTimeout(res, 100));
            }
            const rect = menuContainer.getBoundingClientRect();
            menuPosition = {
                x: e.clientX - rect.x - 130,
                y: e.clientY - rect.y + 10,
            };
            menuVisible = true;
        })().catch(handleError);
    }

    function closeMenu() {
        menuVisible = false;
    }

    async function historyAvailable() {
        let rootDir: versiobit_client.versiobit.ocas.DirectoryMetadata = await $tx.getRootDirectory();
        return rootDir.protectionLevel == "STRONG_VERSIONING";
    }

    let attestationDate = spaceClient.getLatestAttestationDate()
</script>

<div class="dl-container">
    {#if ctx.subDirs.length > 0 || ctx.subFiles.length > 0}
        <table>
            <thead>
                <tr>
                    <th class="dl-name">{msg.colName}</th>
                    <th class="dl-lastModified">{msg.colLastModified}</th>
                    <th class="dl-more">{msg.colActions}</th>
                </tr>
            </thead>
            <tbody>
                {#each ctx.subDirs as dir (dir.name)}
                    {#if showInaccessible || spaceClient.authorizer.requestDirectoryReadAccess(dir) === "OK"}
                    <tr>
                        <td class="dl-name">
                            <span class="dl-filetype-icon">
                                {#if spaceClient.authorizer.requestDirectoryReadAccess(dir) === "INSUFFICIENT_ACCESS"}
                                <span class="dl-folder-overlay-icon">
                                    <LockIcon />
                                </span>
                                {/if}
                                <FolderIcon />
                            </span>
                            <a href="#dataroom:{ctx.pathOf(dir.name)}" on:click|preventDefault|stopPropagation={(e) => navigateToDir(dir)}>{dir.name}</a>
                        </td>
                        <td class="dl-lastModified">
                            <div class="dl-tooltip">
                                {formatSince(dir.date)}
                                <span class="dl-tooltiptext">{formateDate(dir.date)}</span>
                            </div>
                        </td>
                        <td class="dl-more">
                            <a class="dl-actions" href={null} on:click|preventDefault|stopPropagation={(e) => showMenu(e, dir)}>
                                <MoreIcon />
                            </a>
                        </td>
                    </tr>
                    {/if}
                {/each}

                {#each ctx.subFiles as file (file.name)}
                    <tr>
                        <td class="dl-name">
                            <span class="dl-filetype-icon">
                                <FileIcon />
                            </span>
                            <a href={spaceClient.getFileDownloadUrl(file)} target="_blank">{file.name}</a>
                            {#await attestationDate}  
                                ...loading status
                            {:then latestAttestationDate} 
                                {#if file.date <= latestAttestationDate}
                                <b style="cursor: pointer;" on:click={() => {
                                    clickedProofFile = file
                                    proofModal.show()
                                    }}>&#10003;</b>
                                {:else}
                                X
                                {/if}
                            {/await}
                            
                        </td>
                        <td class="dl-lastModified">
                            <div class="dl-tooltip">
                                {formatSince(file.date)}
                                <span class="dl-tooltiptext">{formateDate(file.date)}</span>
                            </div>
                        </td>
                        <td class="dl-more">
                            <a class="dl-actions" href={null} on:click|preventDefault|stopPropagation={(e) => showMenu(e, file)}>
                                <MoreIcon />
                            </a>
                        </td>
                    </tr>
                {/each}
            </tbody>
        </table>
    {:else}
        <p>{msg.empty}</p>
    {/if}

    <div style="position:relative;" bind:this={menuContainer}>
        {#if menuVisible}
            <Menu {...menuPosition} on:click={closeMenu} on:clickoutside={closeMenu}>
                <MenuOption enabled={!selection.files} on:click={() => window.open(spaceClient.getFileDownloadUrl(selection))}>
                    <span class="dl-icon"><DownloadIcon /></span>
                    <span>{msg.ctxDownload}</span>
                </MenuOption>
                {#await historyAvailable() then includeHistory}
                    {#if includeHistory}
                <MenuOption on:click={() => historyModal.show()}>
                    <span class="dl-icon"><HistoryIcon /></span>
                    <span>Version History</span>
                </MenuOption>
                    {/if}
                {:catch error}
                    {handleError(error)}
                {/await}
                <!--MenuOption on:click={console.log}>
                    <span class="dl-icon"><ShareIcon /></span>
                    <span>{msg.ctxShare}</span>
                </MenuOption-->
                <MenuDivider />
                <MenuOption enabled={selectionWriteAccess} on:click={() => renameModal.show()}>
                    <span class="dl-icon"><EditIcon /></span>
                    <span>{msg.ctxRename}</span>
                </MenuOption>
                <MenuOption
                    enabled={selectionWriteAccess}
                    on:click={() => (selection.files ? deleteDirectoryModal.show() : deleteFileModal.show())}>
                    <span class="dl-icon"><DeleteIcon /></span>
                    <span>{msg.ctxDelete}</span>
                </MenuOption>
            </Menu>
        {/if}
    </div>

    <Modal bind:this={renameModal}>
        <Rename
            path={ctx.pathOf(selection.name)}
            type={selection.files ? 'directory' : 'file'}
            on:close={() => renameModal.hide()} />
    </Modal>
    <Modal bind:this={historyModal}>
        <History
            path={ctx.pathOf(selection.name)}
            type={selection.files ? 'directory' : 'file'}
            on:close={() => historyModal.hide()} />
    </Modal>
    <Modal bind:this={proofModal}>
        <Proof
            file={clickedProofFile}
            on:close={() => proofModal.hide()} />
    </Modal>
    <Modal bind:this={deleteFileModal}>
        <DeleteFile path={ctx.pathOf(selection.name)} on:close={() => deleteFileModal.hide()} />
    </Modal>
    <Modal bind:this={deleteDirectoryModal}>
        <DeleteDirectory path={ctx.pathOf(selection.name)} on:close={() => deleteDirectoryModal.hide()} />
    </Modal>
</div>

<style>
    table {
        width: 100%;
        table-layout: fixed;
    }
    tr {
        height: 2.6rem;
    }
    .dl-name {
        text-align: left;
        width: 55%;
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
    }
    .dl-name:hover {
        overflow: visible;
        text-overflow: unset;
    }
    .dl-filetype-icon {
        overflow: auto;
        vertical-align: middle;
        padding: 0.5rem 0.4rem 0rem 0;
        position: relative;
    }
    .dl-folder-overlay-icon {
        position: absolute;
        left: 10px;
        top: -4px;
        opacity: 0.7;
    }
    .dl-lastModified {
        text-align: right;
        width: 15%;
    }
    .dl-more {
        text-align: center;
        width: 30%;
    }
    .dl-actions {
        cursor: pointer;
    }
    .dl-icon {
        width: 18px;
        height: 18px;
    }

    /** tooltip */
    .dl-tooltip {
        position: relative;
        display: inline-block;
    }
    .dl-tooltip .dl-tooltiptext {
        visibility: hidden;
        width: 120px;
        background-color: black;
        color: #fff;
        text-align: center;
        border-radius: 6px;
        padding: 5px 0;
        opacity: 0.6;
        position: absolute;
        z-index: 1;
        top: -5px;
        left: 105%;
    }
    .dl-tooltip:hover .dl-tooltiptext {
        visibility: visible;
    }
</style>
