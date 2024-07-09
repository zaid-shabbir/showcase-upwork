<script lang="ts">
    import { createEventDispatcher } from "svelte";
    import { tweened } from "svelte/motion";
    import { cubicOut } from "svelte/easing";
    import { asyncTxAction } from "./util/tx";
    import Form from "../common/Form.svelte";
    import { messages } from "../global";

    const msg = messages.space.FileUpload;
    export let currentPath: string;

    const progressValue = tweened(0, {
        duration: 400,
        easing: cubicOut,
    });

    const dispatch = createEventDispatcher();
    let file;

    const submit = asyncTxAction(async (tx) => {
        progressValue.set(0.01);
        await tx.putFile(currentPath, file, progressValue.set);
        setTimeout(() => dispatch("close"), 200);
    });
</script>

<span>
    <h3>{msg.title}</h3>
    <Form {submit} cancel={() => dispatch('close')}>
        {#if $progressValue == 0}
        <p>
            {@html msg.intro}
        </p>
        <div>
            <input
                type="file"
                accept=".pdf,.doc,.docx,.xls,.xlsx,.ppt,.pptx,.png,.jpg,.jpeg,.mp4,.mov,.avi,.wmv"
                on:change={(e) => {
                    file = e.target.files[0];
                }}
                required />
            {#if currentPath}
                <br /><br />
                <p>{msg.folderLabel(currentPath)}</p>
            {/if}
        </div>
        {/if}

        {#if $progressValue > 0}
        <div>
            <p>{msg.uploadInProgress}</p>
            <progress value={$progressValue} />
            <p>{Math.round($progressValue * 1000) / 10}%</p>
        </div>
    {/if}
    </Form>
</span>

<style>
    progress {
        display: block;
        width: 100%;
    }
</style>
