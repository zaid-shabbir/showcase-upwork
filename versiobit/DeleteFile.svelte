<script lang="ts">
    import { createEventDispatcher } from "svelte";
    import { asyncTxAction } from "./util/tx";
    import Form from "../common/Form.svelte";
    import { messages } from "../global";

    const dispatch = createEventDispatcher();
    const msg = messages.space.DeleteFile;
    export let path: string;

    const submit = asyncTxAction(async (tx) => {
        await tx.deleteFile(path);
        dispatch("close");
    });
</script>

<span>
    <h3>{msg.title}</h3>
    <Form {submit} cancel={() => dispatch('close')} submitTitle={msg.submitTitle}>
        <p>{msg.intro(path)}</p>
    </Form>
</span>

<style>
</style>
