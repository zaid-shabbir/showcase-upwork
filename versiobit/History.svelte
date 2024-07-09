<script lang="ts">
    import { createEventDispatcher } from "svelte";
    import { handleError } from "../error-handler";
    import { tx } from "./util/tx";
    import { getSpaceClient } from "./util/space-context";

    const dispatch = createEventDispatcher();
    const spaceClient = getSpaceClient();
    export let path: string;
    export let type: string;

    let history: any[] = [];

    function typeToHumanReadable(type: String) {
        switch (type) {
            case "FILE_OPERATION":
                return "Datei";
            case "DIRECTORY_OPERATION":
                return "Verzeichnis";
            case "SUBDIRECTORY_OPERATION":
                return "Unterverzeichnis";
            default:
                console.log(type);
                return "UNKNOWN TYPE";
        }
    }

    function operationToHumanReadable(operation: String) {
        switch (operation) {
            case "CREATION":
                return "erstellt";
            case "DELETION":
                return "gelöscht";
            case "NEW_VERSION":
                return "geupdated";
            case "RENAME":
                return "umbenannt";
            default:
                return "UNKNOWN OPERATION";
        }
    }

    let result = {
        title: "",
        promise: null,
    };

    async function fetchAndFormatHistory() {
        let history;
        if (type === "file") {
            history = await $tx.getFileHistoryByPath(path);
        } else {
            history = await $tx.getDirectoryHistoryByPath(path);
        }
        return history;
    }

    function formatDirectoryEntry(entry: versiobit_client.versiobit.storage.history.DataOperation) {
        let path = entry.newPath ? entry.newPath : entry.oldPath;
        switch (entry.ops) {
            case "RENAME":
                return typeToHumanReadable(entry.type) + " " + entry.oldPath + " in " + entry.newPath + " "+ operationToHumanReadable(entry.ops) ;
            default:
                return typeToHumanReadable(entry.type) + " " + path + " " + operationToHumanReadable(entry.ops);
        }
    }
    result.promise = fetchAndFormatHistory();
</script>

<span>
    <h3>{result.title}</h3>

    {#await result.promise}
        <p>...lade historische Daten</p>
    {:then history}
        <ul>
            {#each history as entry, index}
                <li>
                    {#if entry.type == "FILE_OPERATION"}
                        <a href={spaceClient.getFileDownloadUrl(entry.fileMetadata)} target="_blank">{formatDirectoryEntry(entry)}</a>
                    {:else}
                        {formatDirectoryEntry(entry)}
                    {/if}
                </li>
            {/each}
        </ul>
    {:catch error}
        {handleError(error)}
    {/await}

    <button on:click={() => dispatch("close")}>Close</button>
</span>

<style>
    ul {
        list-style-type: square;
    }
</style>
