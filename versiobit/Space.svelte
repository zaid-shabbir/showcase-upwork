<script lang="ts">
    import Spinner from "../common/Spinner.svelte";
    import { handleError } from "../error-handler";
    import { initTx, tx } from "./util/tx";
    import Dataroom from "./Dataroom.svelte";
    import { setSpaceClient } from "./util/space-context";
    import { appContext } from "../global";

    export let spaceId: number;

    const spaceClient = appContext.createSpaceClient(spaceId);
    setSpaceClient(spaceClient);

    let loaded = false;
    async function init() {
        // initTx will create a SpaceTransaction, which needs a session, and will then load metadata
        // all context is coming from the spaceClient
        await initTx(spaceClient);
        loaded = true;
    }
    init().catch(handleError);
</script>

<span>
    {#if loaded}
        <Dataroom />
    {:else}
        <Spinner />
    {/if}
</span>
